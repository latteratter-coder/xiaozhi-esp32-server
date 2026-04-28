# Spring Boot 入门与项目架构原理

> 面向首次接触 Spring Boot 的开发者，以 manager-api 项目为实例，讲解从核心概念到项目架构再到深度机制的完整学习路径。
>
> **相关文档：**
> - 源码目录逐文件速查 → 见 `11-manager-api源码目录功能详解(小白入门).md`
> - OAuth2 认证完整流程 → 见 `10-OAuth2用户Token认证完整流程.md`

---

## 目录

- [第一部分：Spring Boot 核心概念](#第一部分spring-boot-核心概念)
  - [1. IoC 容器与依赖注入](#1-ioc-容器与依赖注入)
    - [1.1 @Bean vs @Component](#11-bean-vs-component--两种注册-bean-的方式)
    - [1.2 "在类上打标签"是什么意思](#12-在类上打标签是什么意思)
    - [1.3 Bean 的单例特性与普通对象的区别](#13-bean-的单例特性与普通对象的区别)
    - [1.4 Tomcat、DispatcherServlet 与 Controller 的关系](#14-tomcatdispatcherservlet-与-controller-的关系)
    - [1.5 Dao 与 XML 的映射机制](#15-dao-与-xml-的映射机制)
  - [2. 自动配置与启动原理](#2-自动配置与启动原理)
  - [3. 配置文件与 Profile](#3-配置文件与-profile)
  - [4. 统一响应与全局异常处理](#4-统一响应与全局异常处理)
- [第二部分：项目架构全景](#第二部分项目架构全景)
  - [5. 系统全局架构](#5-系统全局架构)
  - [6. manager-api 内部架构](#6-manager-api-内部架构)
  - [7. 一次完整请求的生命周期](#7-一次完整请求的生命周期)
  - [8. 安全认证三通道](#8-安全认证三通道)
  - [9. 业务模块依赖关系](#9-业务模块依赖关系)
- [第三部分：骨架与增强件 — 理解项目本质](#第三部分骨架与增强件--理解项目本质)
  - [10. 最小骨架 vs 增强件](#10-最小骨架-vs-增强件)
  - [11. 增强件是怎么"套"上去的](#11-增强件是怎么套上去的)
- [第四部分：深度机制详解](#第四部分深度机制详解)
  - [12. AOP 面向切面编程](#12-aop-面向切面编程)
  - [13. RenExceptionHandler 全局异常处理原理](#13-renexceptionhandler-全局异常处理原理)
  - [14. XssFilter — XSS 攻击防护](#14-xssfilter--xss-攻击防护)
- [附录 A：注解速查手册](#附录-a注解速查手册)

---

# 第一部分：Spring Boot 核心概念

> 在看项目代码之前，先掌握这四个核心概念。这是理解一切的基础。

## 1. IoC 容器与依赖注入

**核心思想**：你不需要自己 `new` 对象，Spring 帮你创建并管理所有对象（称为 **Bean**），需要时自动"注入"给你。

**传统写法 vs Spring 写法：**

```java
// 传统写法：自己创建所有依赖（耦合严重，难以测试）
public class DeviceController {
    private DeviceService service = new DeviceServiceImpl(
        new DeviceDao(),
        new RedisUtils(),
        new SysParamsService()  // 还要继续 new 它的依赖...
    );
}

// Spring 写法：声明"我需要什么"，Spring 自动注入
public class DeviceController {
    private final DeviceService deviceService;      // Spring 自动注入
    private final RedisUtils redisUtils;            // Spring 自动注入

    // 构造器注入（推荐方式）
    public DeviceController(DeviceService deviceService, RedisUtils redisUtils) {
        this.deviceService = deviceService;
        this.redisUtils = redisUtils;
    }
}
```

**Spring 怎么知道要注入什么？** 通过注解标记：

```
┌─────────────────────────── Spring IoC 容器 ────────────────────────────┐
│                                                                        │
│  启动时扫描 xiaozhi 包下所有带注解的类，创建 Bean 实例放入容器            │
│                                                                        │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐     │
│  │ @RestController   │  │ @Service          │  │ @Component        │     │
│  │ DeviceController  │  │ DeviceServiceImpl │  │ RedisUtils        │     │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘     │
│           │                     │                      │               │
│           │   需要 DeviceService?                      │               │
│           ├────────────────────→ 自动注入 ◄─────────────┤               │
│           │   需要 RedisUtils?                         │               │
│           ├────────────────────────────────────────────→┘               │
│                                                                        │
│  本项目中的三种注入方式：                                                │
│                                                                        │
│  方式1：构造器注入（DeviceController 使用此方式）                        │
│    public DeviceController(DeviceService s, RedisUtils r) { ... }     │
│                                                                        │
│  方式2：@Autowired 字段注入（BaseServiceImpl 使用此方式）               │
│    @Autowired protected M baseDao;                                     │
│                                                                        │
│  方式3：@AllArgsConstructor + Lombok（DeviceServiceImpl 使用此方式）    │
│    @AllArgsConstructor  // Lombok 自动生成全参数构造器                   │
│    public class DeviceServiceImpl { ... }                              │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 1.1 `@Bean` vs `@Component` — 两种注册 Bean 的方式

Spring IoC 容器里的对象都叫 Bean，但有两种方式把对象放进去：

**方式 1：`@Component`（打标签，Spring 自动创建）**

适用于：**你自己写的类**，不需要复杂组装。

```java
@Component                              // ← 告诉 Spring：直接把这个类创建成 Bean
public class RedisAspect {

    @Value("${renren.redis.open}")      // ← Spring 自动从配置文件注入值
    private boolean open;
}
```

Spring 看到 `@Component`，会自动 `new RedisAspect()`，再把 `@Value` 对应的配置值填进去。你什么都不用管。

**方式 2：`@Bean`（手工组装后注册）**

适用于：**第三方框架的类**（你不能改它源码加 `@Component`），或**需要复杂初始化**的对象。

```java
@Configuration
public class MybatisPlusConfig {

    @Bean                               // ← 告诉 Spring：这个方法的返回值是一个 Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        var obj = new MybatisPlusInterceptor();          // 你自己 new
        obj.addInnerInterceptor(new DataFilterInterceptor());    // 你自己组装
        obj.addInnerInterceptor(new PaginationInnerInterceptor());
        obj.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        obj.addInnerInterceptor(new BlockAttackInnerInterceptor());
        return obj;                                      // 交给 Spring 管理
    }
}
```

`MybatisPlusInterceptor` 是 MyBatis-Plus 框架的类，你不能在它源码上加 `@Component`。而且它需要往里面塞 4 个子插件，不是简单 new 一下就行。所以必须用 `@Bean` 手工组装。

**类比理解：**

```
@Component = 给标准配置的电脑贴个"入库"标签
             仓库管理员（Spring）扫码直接收进去

@Bean      = 你在工厂里亲手组装一台定制电脑
             装好 CPU、内存、显卡，然后交给仓库管理
```

**判断用哪个的决策树：**

```
这个类是你自己写的吗？
  │
  ├── 是 → 这个类创建时需要复杂的手工组装吗？
  │         │
  │         ├── 不需要 → 用 @Component（或它的变体）
  │         │             @Service        → Service 层
  │         │             @RestController → Controller 层
  │         │             @Configuration  → 配置类
  │         │             @Component      → 其他通用组件
  │         │
  │         └── 需要   → 用 @Bean（写在 @Configuration 类里）
  │
  └── 不是（第三方框架的类）→ 只能用 @Bean
                              因为你改不了别人的源码加 @Component
```

**本项目中的实际对应：**

```
┌──────────────────────────────────────────────────────────────────┐
│  用 @Component（及其变体）的                                       │
│                                                                  │
│  @RestController  DeviceController     ← 你写的 Controller       │
│  @Service         DeviceServiceImpl    ← 你写的 Service          │
│  @Component       RedisUtils           ← 你写的工具类             │
│  @Component       RedisAspect          ← 你写的切面类             │
│  @Configuration   MybatisPlusConfig    ← 你写的配置类（自身）      │
│  @Mapper          DeviceDao            ← 你写的 Dao 接口          │
│                                                                  │
│  共同点：都是 xiaozhi 包下你自己写的类                              │
├──────────────────────────────────────────────────────────────────┤
│  用 @Bean 的                                                      │
│                                                                  │
│  MybatisPlusInterceptor   ← MyBatis-Plus 的类 + 需要装 4 个插件  │
│  RedisTemplate            ← Spring Data Redis 的类 + 要配序列化   │
│  RestTemplate             ← Spring 的类 + 要配超时参数             │
│  SecurityManager          ← Shiro 的类 + 要挂载 Realm             │
│  ShiroFilterFactoryBean   ← Shiro 的类 + 要配 URL 过滤规则        │
│                                                                  │
│  共同点：都是第三方框架的类，或需要复杂初始化                        │
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 "在类上打标签"是什么意思

就是你**打开 `.java` 文件，在 `class` 上面加一行注解**：

```java
// 这个文件在你的项目里，你可以随便改它
// 所以你能"在类上打标签"

@Component          // ← 这就是"在类上打标签"
public class RedisAspect {
    ...
}
```

```java
// 这个类是 MyBatis-Plus 框架的源码，打包在 jar 里
// 你打不开它来加 @Component
// 所以你"不能在类上打标签"，只能用 @Bean

public class MybatisPlusInterceptor {   // ← 别人写的，你改不了
    ...
}
```

"能不能打标签"的本质就是：**这个 `.java` 文件在不在你的项目里，你能不能编辑它**。

另外，`@Service`、`@RestController`、`@Configuration` 这些注解和 `@Component` **做的事情完全一样**——都是告诉 Spring "请把这个类创建成对象，放进 IoC 容器"。它们底层是同一个东西：

```
@Component          → 通用标签（"我是个 Bean"）
  │
  ├── @Service          → 语义化的 @Component（"我是 Service 层的 Bean"）
  ├── @RestController   → 语义化的 @Component（"我是 Controller 层的 Bean"）
  └── @Configuration    → 语义化的 @Component（"我是配置类的 Bean"）
```

你把 `DeviceServiceImpl` 上的 `@Service` 换成 `@Component`，程序一样能跑。区分它们纯粹是为了**让人看代码时一眼知道这个类属于哪一层**。

### 1.3 Bean 的单例特性与普通对象的区别

**Spring 管理的 Bean 默认都是单例**——整个应用只创建一个实例，所有地方注入的是同一个对象。

```
IoC 容器里：

  "deviceServiceImpl" → 0x7A3B（内存中只有这一个对象）

DeviceController 注入的 deviceService ──→ 0x7A3B ──┐
                                                     │  同一个对象
AgentServiceImpl 注入的 deviceService  ──→ 0x7A3B ──┘
```

**但是你自己 `new` 出来的对象不受 Spring 管理，每次 new 都是新的独立对象**：

```java
@Service
@AllArgsConstructor
public class DeviceServiceImpl {

    private final DeviceDao deviceDao;        // ← Bean（Spring 管理的），单例
    private final RedisUtils redisUtils;      // ← Bean（Spring 管理的），单例

    public void bindDevice(String code) {
        DeviceEntity entity = new DeviceEntity();  // ← 不是 Bean（你自己 new 的）
        entity.setMac(code);                       //   每次调用都创建新的，互相独立
        deviceDao.insert(entity);
    }
}
```

**两类对象的区分：**

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Spring 管理的 Bean（带注解的类）                              │
│  → 单例，全局只有一个实例，容器启动时创建                       │
│  → DeviceServiceImpl、RedisUtils、DeviceDao ...              │
│                                                             │
│  你自己 new 的普通对象（Entity、DTO、VO、Result 等）           │
│  → 每次 new 都是全新的，互相独立，用完就回收                    │
│  → DeviceEntity、LoginDTO、AgentCreateDTO、Result ...        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**用本项目举个具体例子——10 个用户同时绑定设备：**

```
10 个用户同时调用 POST /device/bind

                 DeviceController（单例，只有 1 个）
                         │
                         ▼
                 DeviceServiceImpl（单例，只有 1 个）
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
   用户A的请求      用户B的请求      用户C的请求
          │              │              │
   new DeviceEntity() new DeviceEntity() new DeviceEntity()
   mac="AAA"         mac="BBB"         mac="CCC"
          │              │              │
          ▼              ▼              ▼
      各自独立地写入数据库（互不影响）
```

Controller 和 Service 是单例（共用一个），但每次请求处理的数据（Entity/DTO）是独立的新对象。就像饭店只有一个厨师（单例 Service），但每桌客人点的菜（Entity）是各自独立的。

### 1.4 Tomcat、DispatcherServlet 与 Controller 的关系

前端发请求后，不是 Tomcat 直接找 Controller，中间还有一层 **DispatcherServlet**（Spring 的前端控制器）：

```
POST /xiaozhi/device/bind/agent001/ABC123

  ▼ Tomcat 接收请求
  │
  │  Tomcat 不认识 Controller，它只认识 Servlet
  │  Spring Boot 启动时注册了一个 DispatcherServlet 到 Tomcat
  │
  ▼ Tomcat 把请求转给 DispatcherServlet
  │
  │  DispatcherServlet 内部维护一个映射表（启动时扫描所有 Controller 构建）：
  │
  │  ┌─────────────────────────────────────────────────────────────────┐
  │  │  POST /device/bind/{agentId}/{deviceCode}                      │
  │  │       → DeviceController.bindDevice(agentId, deviceCode)       │
  │  │                                                                │
  │  │  GET  /device/list/{agentId}                                   │
  │  │       → DeviceController.getUserDevices(agentId)               │
  │  │                                                                │
  │  │  POST /user/login                                              │
  │  │       → LoginController.login(loginDTO)                        │
  │  │                                                                │
  │  │  GET  /agent/list                                              │
  │  │       → AgentController.list(params)                           │
  │  │  ...几百条映射规则...                                            │
  │  └─────────────────────────────────────────────────────────────────┘
  │
  ▼ 匹配到：DeviceController.bindDevice()
  │
  ▼ 调用这个方法，把 URL 里的 agent001 填到 agentId，ABC123 填到 deviceCode
```

**映射表是怎么建的？** 启动时 Spring 扫描所有 `@RestController` 类，把类上的 `@RequestMapping` 和方法上的 `@PostMapping/@GetMapping` 拼起来：

```java
@RestController
@RequestMapping("/device")                        // ← 类级别前缀：/device
public class DeviceController {

    @PostMapping("/bind/{agentId}/{deviceCode}")   // ← 方法级别
    public Result<Void> bindDevice(...) { ... }
}

// 拼接结果：POST /device/bind/{agentId}/{deviceCode} → bindDevice()
```

### 1.5 Dao 与 XML 的映射机制

Dao 接口和 XML 文件靠**全限定类名 + 方法名**精确匹配。以 `DeviceDao` 为例：

```
Java 接口                                XML 文件
──────────                               ──────────

包名+类名                     ═══════     namespace
xiaozhi.modules.device.dao.DeviceDao      xiaozhi.modules.device.dao.DeviceDao

方法名                        ═══════     id
getAllLastConnectedAtByAgentId             getAllLastConnectedAtByAgentId

参数名                        ═══════     #{参数名}
String agentId                            #{agentId}
```

具体对应关系：

```java
// DeviceDao.java —— Java 接口
@Mapper
public interface DeviceDao extends BaseMapper<DeviceEntity> {
    Date getAllLastConnectedAtByAgentId(String agentId);
//       ─────────────────────────────         ───────
//       方法名（对应 XML 里的 id）             参数名（对应 #{agentId}）
}
```

```xml
<!-- DeviceDao.xml —— SQL 文件 -->
<mapper namespace="xiaozhi.modules.device.dao.DeviceDao">
<!--               ─────────────────────────────────────
                   必须和 Java 接口的全限定类名完全一致 -->

    <select id="getAllLastConnectedAtByAgentId" resultType="java.util.Date">
    <!--         ─────────────────────────────
                 必须和 Java 接口的方法名完全一致 -->
        SELECT last_connected_at FROM ai_device
        WHERE agent_id = #{agentId}
        ORDER BY last_connected_at DESC LIMIT 0,1
    </select>
</mapper>
```

Spring Boot 通过 `application.yml` 里的配置找到 XML 文件：

```yaml
mybatis-plus:
  mapper-locations: classpath*:/mapper/**/*.xml
  # 扫描 resources/mapper/ 下所有子目录的 .xml 文件
```

**什么时候需要 XML，什么时候不需要：**

`DeviceDao extends BaseMapper<DeviceEntity>`，`BaseMapper` 提供了通用 CRUD 方法，这些**不需要写 XML**。MyBatis-Plus 根据 `@TableName` 注解自动生成 SQL：

```java
@TableName("ai_device")
public class DeviceEntity extends BaseEntity {
    private String mac;
    private String agentId;
}

// BaseMapper 自动生成的 SQL（不需要写 XML）：
// insert      → INSERT INTO ai_device (mac, agent_id, ...) VALUES (?, ?, ...)
// selectById  → SELECT * FROM ai_device WHERE id = ?
// updateById  → UPDATE ai_device SET mac=?, agent_id=? WHERE id = ?
// deleteById  → DELETE FROM ai_device WHERE id = ?
```

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  不需要 XML（MyBatis-Plus 自动生成 SQL）                      │
│                                                              │
│  deviceDao.insert(entity)           ← BaseMapper 提供       │
│  deviceDao.selectById(id)           ← BaseMapper 提供       │
│  deviceDao.updateById(entity)       ← BaseMapper 提供       │
│  deviceDao.deleteById(id)           ← BaseMapper 提供       │
│  deviceDao.selectList(wrapper)      ← BaseMapper 提供       │
│                                                              │
│  简单的 CRUD 全部自动搞定，不用写一行 SQL                      │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  需要 XML（复杂查询，MyBatis-Plus 搞不定的）                   │
│                                                              │
│  deviceDao.getAllLastConnectedAtByAgentId(agentId)           │
│  → 需要 JOIN、子查询、特殊排序等，必须手写 SQL                 │
│  → 写在 DeviceDao.xml 里，通过 namespace + id 映射           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. 自动配置与启动原理

`@SpringBootApplication` 是整个项目的起点，它等于三个注解的组合：

```
@SpringBootApplication
  │
  ├── @Configuration           → 本类是配置类，可以定义 @Bean
  │
  ├── @EnableAutoConfiguration → 根据 pom.xml 中的依赖自动配置
  │                               引入了 spring-boot-starter-web？
  │                               → 自动启动 Tomcat、配置 MVC
  │                               引入了 spring-boot-starter-data-redis？
  │                               → 自动创建 RedisConnectionFactory
  │                               引入了 mybatis-plus-boot-starter？
  │                               → 自动扫描 @Mapper、配置 SqlSession
  │                               引入了 liquibase-core？
  │                               → 自动执行 db/changelog 中的迁移脚本
  │
  └── @ComponentScan           → 扫描当前包（xiaozhi）及所有子包
                                  找到所有 @Controller/@Service/@Component/@Configuration
                                  创建 Bean 实例放入 IoC 容器
```

**启动流程：**

```
main() 方法
  │
  ▼ SpringApplication.run(AdminApplication.class)
  │
  ▼ 创建 IoC 容器
  │
  ▼ 扫描 xiaozhi 包下所有注解类
  │   → @Configuration: ShiroConfig, RedisConfig, SwaggerConfig, AsyncConfig, MybatisPlusConfig ...
  │   → @RestController: DeviceController, AgentController, LoginController ...
  │   → @Service: DeviceServiceImpl, AgentServiceImpl ...
  │   → @Component: RedisUtils, FieldMetaObjectHandler, RedisAspect ...
  │   → @Mapper: DeviceDao, AgentDao, SysUserDao ...
  │
  ▼ 自动配置
  │   → 启动内嵌 Tomcat（端口 8002）
  │   → 创建 Druid 数据源连接 MySQL
  │   → 创建 Lettuce 连接 Redis
  │   → 执行 Liquibase 数据库迁移
  │   → 注册 Knife4j API 文档
  │
  ▼ 执行 @Bean 方法
  │   → ShiroConfig: 创建 SecurityManager、过滤器链
  │   → RedisConfig: 创建 RedisTemplate
  │   → MybatisPlusConfig: 注册分页插件等
  │
  ▼ 执行 SystemInitConfig（@DependsOn("liquibase")）
  │   → 校验版本、初始化 server.secret、预热配置缓存
  │
  ▼ 打印 "http://localhost:8002/xiaozhi/doc.html"
  │
  ▼ 开始接收 HTTP 请求
```

---

## 3. 配置文件与 Profile

Spring Boot 用 YAML 文件管理配置，支持多环境切换：

```
application.yml          → 通用配置（所有环境共享）
  │                          端口、MyBatis、Swagger、XSS 等
  │
  └── spring.profiles.active: dev    → 激活 dev 环境
          │
          ▼
application-dev.yml      → 开发环境
  │                          本地 MySQL: localhost:3306
  │                          本地 Redis: localhost:6379
  │
application-test.yml     → 测试环境（如有）
  │                          测试服务器地址
  │
application-prod.yml     → 生产环境（如有）
                             线上服务器地址
```

**配置优先级**：`application-dev.yml` 中的值会覆盖 `application.yml` 中的同名配置。

**本项目核心配置项：**

```yaml
# application.yml 关键配置
server:
  port: 8002                          # 服务端口
  servlet:
    context-path: /xiaozhi            # URL 前缀，所有接口以 /xiaozhi 开头

spring:
  profiles:
    active: dev                       # 激活开发环境配置
  messages:
    basename: i18n/messages           # 国际化文件位置

mybatis-plus:
  mapper-locations: classpath*:/mapper/**/*.xml   # MyBatis XML 扫描路径
  typeAliasesPackage: xiaozhi.modules.*.entity    # 实体类包扫描
  global-config:
    db-config:
      id-type: ASSIGN_ID             # 主键策略：手动赋值
```

---

## 4. 统一响应与全局异常处理

**统一响应格式**：所有接口都返回相同结构的 JSON，前端只需按一种格式解析。

```java
// 所有 Controller 方法的返回类型
public class Result<T> {
    private int code;      // 0=成功，其他=错误码
    private String msg;    // 提示消息
    private T data;        // 业务数据
}
```

```json
// 成功响应
{"code": 0, "msg": "success", "data": {"token": "abc123", "expire": 43200}}

// 失败响应
{"code": 10062, "msg": "激活码错误", "data": null}
```

**全局异常处理**：业务代码中只需 `throw` 异常，不用到处写 try-catch。

```
代码中抛异常                           RenExceptionHandler 自动捕获
───────────────                        ──────────────────────────────

throw new RenException(                @ExceptionHandler(RenException.class)
    ErrorCode.ACTIVATION_CODE_ERROR    public Result<Void> handle(RenException ex) {
);                                         return new Result<>().error(ex.getCode(), ex.getMsg());
                                       }
        │                                       │
        └───────────── 自动匹配 ────────────────┘
                                                │
                                                ▼
                                    {"code":10062, "msg":"激活码错误"}
```

`@RestControllerAdvice` 就是一个"全局兜底网"，能捕获所有 Controller 抛出的异常并转为统一 JSON。

> 更深入的原理解析见第四部分 [13. RenExceptionHandler 全局异常处理原理](#13-renexceptionhandler-全局异常处理原理)。

---

# 第二部分：项目架构全景

> 理解了核心概念后，来看 manager-api 整体是怎么搭建起来的。

## 5. 系统全局架构

```
                              ┌─────────────────────────────────────────┐
                              │             客户端                       │
                              │                                         │
                              │  ┌──────────┐ ┌──────────┐ ┌────────┐ │
                              │  │ ESP32 设备│ │Web 智控台 │ │Mobile  │ │
                              │  │ (硬件)    │ │ (Vue)    │ │(uni-app)│ │
                              │  └─────┬────┘ └────┬─────┘ └───┬────┘ │
                              └────────┼───────────┼───────────┼──────┘
                                       │           │           │
                          ┌────────────┼───────────┼───────────┼───────────┐
                          │            ▼           ▼           ▼           │
                          │  ┌──────────────────────────────────────────┐  │
                          │  │         manager-api (Java:8002)          │  │
                          │  │                                          │  │
                          │  │   /ota/**     → 设备 OTA 上报（anon）     │  │
                          │  │   /device/**  → 设备管理（oauth2）        │  │
                          │  │   /agent/**   → 智能体管理（oauth2）      │  │
                          │  │   /config/**  → 配置下发（server）        │  │
                          │  │   /user/**    → 登录注册（anon/oauth2）   │  │
                          │  │   /models/**  → 模型配置（oauth2）        │  │
                          │  │   /admin/**   → 系统管理（oauth2）        │  │
                          │  │                                          │  │
                          │  └───────┬──────────────┬───────────────────┘  │
                          │          │              │                      │
                          │          ▼              ▼                      │
                          │  ┌──────────────┐  ┌──────────────┐           │
                          │  │   MySQL      │  │   Redis      │           │
                          │  │  (数据持久化) │  │  (缓存/会话)  │           │
                          │  └──────────────┘  └──────────────┘           │
                          │                                               │
                          │  ┌──────────────────────────────────────────┐  │
                          │  │      xiaozhi-server (Python:8000/8003)   │  │
                          │  │                                          │  │
                          │  │   WebSocket 8000  ← ESP32 设备对话        │  │
                          │  │   HTTP 8003       ← 内部 API              │  │
                          │  │                                          │  │
                          │  │   启动时调用:                              │  │
                          │  │     POST /config/server-base              │  │
                          │  │     POST /config/agent-models             │  │
                          │  │   对话结束后:                              │  │
                          │  │     POST /agent/chat-history/report       │  │
                          │  └──────────────────────────────────────────┘  │
                          │                                               │
                          │  ┌──────────────┐  ┌──────────────┐           │
                          │  │  RAGFlow     │  │  阿里云短信   │           │
                          │  │ (知识库RAG)  │  │  (SMS SDK)   │           │
                          │  └──────────────┘  └──────────────┘           │
                          │                                               │
                          │                  服务端                        │
                          └───────────────────────────────────────────────┘
```

---

## 6. manager-api 内部架构

```
┌─────────────────────────────────── manager-api ──────────────────────────────────┐
│                                                                                  │
│  ┌─────────────────────────── 过滤器链（请求入口）──────────────────────────────┐ │
│  │                                                                              │ │
│  │  Tomcat → XssFilter(XSS清洗) → ShiroFilter(安全认证) → DispatcherServlet    │ │
│  │                                    │                                         │ │
│  │                    ┌───────────────┼───────────────┐                         │ │
│  │                    ▼               ▼               ▼                         │ │
│  │               ┌────────┐     ┌──────────┐    ┌──────────┐                   │ │
│  │               │  anon  │     │  server  │    │  oauth2  │                   │ │
│  │               │ (免检)  │     │(密钥比对) │    │(Token认证)│                   │ │
│  │               └────────┘     └──────────┘    └──────────┘                   │ │
│  └──────────────────────────────────────────────────────────────────────────────┘ │
│                                        │                                          │
│  ┌─────────────────────────── Controller 层 ───────────────────────────────────┐ │
│  │                                                                              │ │
│  │  DeviceController   AgentController   ConfigController   LoginController    │ │
│  │  OTAController      ModelController   AdminController    SysParamsController │ │
│  │  TimbreController   KnowledgeBaseController   VoiceCloneController   ...    │ │
│  │                                                                              │ │
│  └──────────────────────────────────┬───────────────────────────────────────────┘ │
│                                     │                                             │
│  ┌─────────────────────────── Service 层 ──────────────────────────────────────┐ │
│  │                                                                              │ │
│  │  DeviceService     AgentService      ConfigService     SysUserTokenService  │ │
│  │  OtaService        ModelConfigService KnowledgeBaseService  CaptchaService  │ │
│  │  TimbreService     VoiceCloneService  LLMService       SysParamsService     │ │
│  │                                                                              │ │
│  └──────────────────────────────────┬───────────────────────────────────────────┘ │
│                                     │                                             │
│  ┌─────────────────────────── Dao 层 + Redis ─────────────────────────────────┐  │
│  │                                                                             │  │
│  │  DeviceDao  AgentDao  SysUserDao  SysParamsDao  ModelConfigDao   ...       │  │
│  │  ↕ mapper/*.xml（MyBatis SQL）                                              │  │
│  │                                                          RedisUtils         │  │
│  │  ↕ MySQL                                                 ↕ Redis            │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                   │
│  ┌─────────────────────────── 通用基础层 (common) ─────────────────────────────┐  │
│  │                                                                              │  │
│  │  BaseEntity / BaseDao / BaseService / BaseServiceImpl    ← 基类体系          │  │
│  │  Result / RenException / RenExceptionHandler             ← 响应与异常        │  │
│  │  RedisConfig / RedisUtils / RedisKeys                    ← Redis 封装        │  │
│  │  SwaggerConfig / MybatisPlusConfig / AsyncConfig         ← 全局配置          │  │
│  │  FieldMetaObjectHandler / DataFilterInterceptor          ← 自动填充与拦截    │  │
│  │  XssFilter / SqlFilter / XssUtils                        ← 安全防护          │  │
│  │  SM2Utils / AESUtils / PasswordUtils                     ← 加密工具          │  │
│  │                                                                              │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 一次完整请求的生命周期

以"用户绑定设备"为例：

```
前端 POST /xiaozhi/device/bind/agent001/ABC123
Header: Authorization: Bearer e10adc3949...
  │
  ▼ ① Tomcat 接收（端口 8002）
  │
  ▼ ② XssFilter 清洗请求参数（防 XSS 攻击）
  │
  ▼ ③ ShiroFilter 路由匹配
  │   /device/** → oauth2 过滤器
  │
  ▼ ④ Oauth2Filter 提取 Bearer Token
  │   └→ new Oauth2Token("e10adc3949...")
  │   └→ executeLogin()
  │
  ▼ ⑤ Oauth2Realm 认证
  │   └→ 查 sys_user_token 表（Token 是否有效）
  │   └→ 查 sys_user 表（获取用户信息）
  │   └→ 构建 UserDetail 存入 Subject
  │
  ▼ ⑥ DispatcherServlet 路由
  │   POST + /device/bind/{agentId}/{deviceCode}
  │   → 匹配 DeviceController.bindDevice()
  │
  ▼ ⑦ @RequiresPermissions("sys:role:normal") 权限检查
  │   └→ Oauth2Realm.doGetAuthorizationInfo()
  │   └→ 用户权限集合是否包含 "sys:role:normal"
  │
  ▼ ⑧ DeviceController.bindDevice("agent001", "ABC123")
  │   │
  │   └→ deviceService.deviceActivation("agent001", "ABC123")
  │       │
  │       ├→ redisUtils.get(key)         // 从 Redis 查激活码
  │       ├→ deviceDao.insert(entity)    // 写入 MySQL ai_device 表
  │       └→ redisUtils.delete(keys)     // 清理 Redis 缓存
  │
  ▼ ⑨ 返回 Result
  │   new Result<>() → Jackson 序列化为 JSON
  │
  ▼ ⑩ 响应
  │
  {"code": 0, "msg": "success", "data": null}
```

---

## 8. 安全认证三通道

```
┌───────────────────────────── 三种认证通道 ─────────────────────────────────┐
│                                                                            │
│  ┌──────────────────── anon（匿名通道）────────────────────────┐           │
│  │                                                              │           │
│  │  /ota/**          设备 OTA 上报                              │           │
│  │  /user/login      登录                                      │           │
│  │  /user/captcha    验证码                                    │           │
│  │  /user/register   注册                                      │           │
│  │  /doc.html        API 文档                                  │           │
│  │  /agent/play/**   音频播放                                   │           │
│  │                                                              │           │
│  │  特点：不做任何认证，直接放行到 Controller                      │           │
│  └──────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  ┌──────────────────── server（服务间通道）─────────────────────┐           │
│  │                                                              │           │
│  │  /config/**                        配置下发                  │           │
│  │  /agent/chat-history/report        聊天上报                  │           │
│  │  /agent/chat-summary/**            聊天摘要                  │           │
│  │  /agent/chat-title/**              聊天标题                  │           │
│  │                                                              │           │
│  │  认证方式：                                                  │           │
│  │    Header: Authorization: Bearer {server.secret}             │           │
│  │    → ServerSecretFilter 取 token                             │           │
│  │    → 与 sys_params 表中 server.secret 做字符串比对            │           │
│  │    → 匹配放行，不匹配返回 401                                │           │
│  │                                                              │           │
│  │  特点：不产生用户上下文（SecurityUser.getUser() 为空）         │           │
│  └──────────────────────────────────────────────────────────────┘           │
│                                                                            │
│  ┌──────────────────── oauth2（用户通道）──────────────────────┐            │
│  │                                                              │           │
│  │  /**  （所有未匹配上述规则的请求）                             │           │
│  │                                                              │           │
│  │  认证方式：                                                  │           │
│  │    Header: Authorization: Bearer {用户Token}                 │           │
│  │    → Oauth2Filter 提取 Token                                │           │
│  │    → Oauth2Token 封装                                       │           │
│  │    → Oauth2Realm 查 sys_user_token 表验证                   │           │
│  │    → 查 sys_user 表获取用户信息                              │           │
│  │    → 构建 UserDetail 存入 Shiro Subject                     │           │
│  │                                                              │           │
│  │  授权模型：                                                  │           │
│  │    超级管理员 → sys:role:superAdmin + sys:role:normal        │           │
│  │    普通用户   → sys:role:normal                              │           │
│  │                                                              │           │
│  │  Token 来源（登录流程）：                                     │           │
│  │    前端密码 → SM2加密 → 后端SM2解密 → 提取验证码校验          │           │
│  │    → BCrypt比对密码 → TokenGenerator生成Token                │           │
│  │    → 存入 sys_user_token 表（12小时过期）                    │           │
│  │                                                              │           │
│  │  完整流程详见 → 10-OAuth2用户Token认证完整流程.md              │           │
│  └──────────────────────────────────────────────────────────────┘           │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. 业务模块依赖关系

```
┌────────────────────────────────── modules 模块依赖图 ──────────────────────────────┐
│                                                                                    │
│                              ┌──────────┐                                          │
│                              │ security │ ← 所有需要认证的模块都依赖               │
│                              │ (认证授权) │                                          │
│                              └─────┬────┘                                          │
│                                    │                                               │
│          ┌─────────────────────────┼──────────────────────────┐                    │
│          │                         │                          │                    │
│          ▼                         ▼                          ▼                    │
│    ┌──────────┐            ┌──────────────┐           ┌────────────┐              │
│    │  device  │            │    agent     │           │    sys     │              │
│    │ (设备管理) │            │  (智能体管理) │           │ (系统管理)  │              │
│    └─────┬────┘            └──────┬───────┘           └─────┬──────┘              │
│          │                        │                         │                      │
│          │                        ├──────────┐              │                      │
│          │                        │          │              │                      │
│          │                        ▼          ▼              │                      │
│          │                 ┌──────────┐ ┌────────┐          │                      │
│          │                 │  model   │ │  llm   │          │                      │
│          │                 │(模型配置) │ │(LLM调用)│          │                      │
│          │                 └──────────┘ └────────┘          │                      │
│          │                        │                         │                      │
│          │                        ▼                         │                      │
│          │                 ┌────────────┐                   │                      │
│          │                 │ knowledge  │                   │                      │
│          │                 │ (知识库)    │                   │                      │
│          │                 └────────────┘                   │                      │
│          │                                                  │                      │
│          │                 ┌────────────┐  ┌────────────┐   │                      │
│          │                 │  timbre    │  │ voiceclone │   │                      │
│          │                 │  (音色)     │←→│ (声音克隆)  │   │                      │
│          │                 └────────────┘  └────────────┘   │                      │
│          │                                                  │                      │
│          ▼                                                  ▼                      │
│    ┌──────────────────────────────────────────────────────────────┐                │
│    │                     config（配置中枢）                        │                │
│    │  聚合 sys + device + agent + model 数据，生成配置下发给 Python │                │
│    └──────────────────────────────────────────────────────────────┘                │
│          │                                                                         │
│          ▼                                                                         │
│    ┌──────────┐                                                                    │
│    │   sms    │ ← 独立模块，仅被 security（验证码）调用                             │
│    │ (短信)    │                                                                    │
│    └──────────┘                                                                    │
│                                                                                    │
│  所有模块共同依赖：common（基础层 — 基类、工具、Redis、异常、XSS防护等）             │
└────────────────────────────────────────────────────────────────────────────────────┘
```

---

# 第三部分：骨架与增强件 — 理解项目本质

> 看了这么多文件，很容易被复杂性吓到。这一部分帮你抓住本质。

## 10. 最小骨架 vs 增强件

如果去掉 Shiro、XSS 防护、Redis、Lombok、Knife4j、Liquibase 等所有"锦上添花"的组件，整个项目的**本质**只有一条链路：

```
前端请求  →  Controller  →  Service  →  Dao  →  MySQL
```

这是 Spring Boot 最核心的 MVC 三层架构。其他所有东西都是围绕这条链路做增强：

```
┌─────────────── 增强件（可拆卸，去掉项目照样跑）────────────────────┐
│                                                                    │
│  Shiro        在 Controller 之前加了一道安检门（过滤器链）           │
│  XssFilter    在 Shiro 之前加了一道请求清洗                        │
│  Redis        给 Service 加了一个缓存搭档，减少数据库查询           │
│  AOP          在方法周围包了一层透明外衣（事务、日志、Redis开关）    │
│  全局异常      在最外层兜底，任何地方 throw 都能统一转 JSON          │
│  Lombok       编译期少写代码，运行时完全不存在                     │
│  Knife4j      生成 API 文档页面，跟业务无关                       │
│  Liquibase    启动时自动建表，之后就不管了                         │
│  i18n         错误消息翻译，不影响主流程                           │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘

┌─────────────── 骨架（不可去掉）────────────────────────────────────┐
│                                                                    │
│  Spring Boot   IoC 容器 + 自动配置 + 内嵌 Tomcat                  │
│  Controller    接收请求，返回 JSON                                 │
│  Service       业务逻辑                                            │
│  Dao + MyBatis-Plus  数据库操作                                    │
│  MySQL         数据存储                                            │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 11. 增强件是怎么"套"上去的

一次完整请求经过的所有层（从外到内）：

```
  ┌─ 最外层：全局异常兜底 ──────────────────────────────────────────────┐
  │  RenExceptionHandler                                               │
  │  任何层抛出异常都会被这里捕获，统一转为 JSON                          │
  │                                                                    │
  │  ┌─ 第1层过滤器：XssFilter ──────────────────────────────────────┐ │
  │  │  清洗请求参数中的恶意 HTML/JS 代码                              │ │
  │  │                                                                │ │
  │  │  ┌─ 第2层过滤器：ShiroFilter ──────────────────────────────┐  │ │
  │  │  │  根据 URL 匹配 anon/server/oauth2，做安全认证             │  │ │
  │  │  │                                                          │  │ │
  │  │  │  ┌─ Controller 层 ────────────────────────────────────┐  │  │ │
  │  │  │  │  接收请求参数，调用 Service                          │  │  │ │
  │  │  │  │                                                    │  │  │ │
  │  │  │  │  ┌─ AOP: @Transactional ──────────────────────┐   │  │  │ │
  │  │  │  │  │  Service 方法自动开启事务                     │   │  │  │ │
  │  │  │  │  │                                              │   │  │  │ │
  │  │  │  │  │  ┌─ Service 层 ────────────────────────┐    │   │  │  │ │
  │  │  │  │  │  │  执行业务逻辑                        │    │   │  │  │ │
  │  │  │  │  │  │                                      │    │   │  │  │ │
  │  │  │  │  │  │  ┌─ AOP: RedisAspect ───────────┐   │    │   │  │  │ │
  │  │  │  │  │  │  │ Redis 开关控制               │   │    │   │  │  │ │
  │  │  │  │  │  │  │ ┌─ RedisUtils ──────────┐   │   │    │   │  │  │ │
  │  │  │  │  │  │  │ │ 读写 Redis 缓存       │   │   │    │   │  │  │ │
  │  │  │  │  │  │  │ └──────────────────────┘   │   │    │   │  │  │ │
  │  │  │  │  │  │  └────────────────────────────┘   │    │   │  │  │ │
  │  │  │  │  │  │                                      │    │   │  │  │ │
  │  │  │  │  │  │  ┌─ Dao 层 ────────────────────┐    │    │   │  │  │ │
  │  │  │  │  │  │  │ 读写 MySQL 数据库            │    │    │   │  │  │ │
  │  │  │  │  │  │  └──────────────────────────────┘    │    │   │  │  │ │
  │  │  │  │  │  └──────────────────────────────────────┘    │   │  │  │ │
  │  │  │  │  └──────────────────────────────────────────────┘   │  │  │ │
  │  │  │  └─────────────────────────────────────────────────────┘  │  │ │
  │  │  └──────────────────────────────────────────────────────────┘  │ │
  │  └────────────────────────────────────────────────────────────────┘ │
  └────────────────────────────────────────────────────────────────────┘
```

---

# 第四部分：深度机制详解

> 第一、二、三部分让你理解了"是什么"和"在哪"，第四部分深入讲解"怎么做到的"。

## 12. AOP 面向切面编程

### 12.1 通俗理解

AOP（Aspect-Oriented Programming）可以理解为**在不修改原有代码的情况下，给方法"套一层透明外衣"**。

生活类比：你进出小区不需要自己开门关门，保安（切面）会帮你做这件事。你只管走路（业务逻辑），安检（横切关注点）由保安在你进出时自动执行。

```
没有 AOP：                              有 AOP：

public void save() {                    public void save() {
    开启事务();         ← 手动           │
    try {                                │  @Transactional 自动
        插入数据();     ← 业务              插入数据();  ← 只写业务
        提交事务();     ← 手动           │
    } catch (Exception e) {             │  @Transactional 自动
        回滚事务();     ← 手动           │
    }                                   }
}
```

### 12.2 本项目中 AOP 的三个应用

**应用 1：`@Transactional` — 自动事务管理**

```java
// BaseServiceImpl.java
@Transactional(rollbackFor = Exception.class)
public boolean insert(T entity) {
    return retBool(baseDao.insert(entity));
}
```

你只需要加一个 `@Transactional` 注解，Spring AOP 自动帮你：
- 方法开始前 → 开启数据库事务
- 方法正常结束 → 提交事务
- 方法抛异常 → 回滚事务

```
调用 insert() 时实际发生的：

┌── AOP 代理 (@Transactional) ──────────────┐
│                                            │
│  ① 开启事务 connection.setAutoCommit(false) │
│                                            │
│  ② 执行你的代码 baseDao.insert(entity)      │
│                                            │
│  ③-a 没异常？→ 提交 connection.commit()      │
│  ③-b 有异常？→ 回滚 connection.rollback()    │
│                                            │
└────────────────────────────────────────────┘
```

**应用 2：`RedisAspect` — Redis 开关控制**

```java
// RedisAspect.java

@Slf4j
@Aspect        // ← 声明这是一个切面
@Component     // ← 交给 Spring 管理
public class RedisAspect {

    @Value("${renren.redis.open}")   // 从配置文件读取开关
    private boolean open;

    // 拦截 RedisUtils 类的所有方法
    @Around("execution(* xiaozhi.common.redis.RedisUtils.*(..))")
    public Object around(ProceedingJoinPoint point) throws Throwable {
        Object result = null;
        if (open) {                        // 开关打开：正常执行
            try {
                result = point.proceed();  // ← 执行原方法
            } catch (Exception e) {
                log.error("redis error", e);
                throw new RenException(ErrorCode.REDIS_ERROR);
            }
        }
        // 开关关闭：什么都不做，返回 null
        return result;
    }
}
```

原理图：

```
配置 renren.redis.open = true 时：

  调用 redisUtils.get("key")
    │
    ▼
  RedisAspect.around() 拦截
    │
    ├── open == true
    │   └── point.proceed() → 真正执行 Redis 操作 → 返回结果
    │
    └── open == false（比如 Redis 宕机时临时关闭）
        └── 直接返回 null，不调用 Redis → 程序不会崩溃
```

关键词解释：
- `@Aspect` — 声明这个类是一个"切面"
- `@Around` — 环绕通知，在目标方法的前后都能插入逻辑
- `execution(* xiaozhi.common.redis.RedisUtils.*(..))` — 匹配表达式，拦截 `RedisUtils` 类的所有方法
- `ProceedingJoinPoint` — 代表被拦截的那个方法，`point.proceed()` 就是"继续执行原方法"

**应用 3：`@Async` — 异步方法**

```java
// DeviceServiceImpl.java
@Async   // ← AOP 拦截，在线程池中执行此方法
public void updateDeviceConnectionInfo(String agentId, String deviceId, String appVersion) {
    // 更新设备连接时间（耗时操作，不阻塞主流程）
}
```

`@Async` 让方法在独立线程中执行，主线程不用等它完成就能返回响应给前端。

---

## 13. RenExceptionHandler 全局异常处理原理

### 13.1 它解决了什么问题

**没有全局异常处理时：**

```java
// 每个 Controller 方法都要写 try-catch，代码臃肿
@PostMapping("/bind")
public Result<Void> bindDevice(String code) {
    try {
        deviceService.deviceActivation(code);
        return new Result<>();
    } catch (RenException e) {
        return new Result<Void>().error(e.getCode(), e.getMsg());
    } catch (DuplicateKeyException e) {
        return new Result<Void>().error("数据已存在");
    } catch (Exception e) {
        return new Result<Void>().error("系统错误");
    }
}
```

**有全局异常处理后：**

```java
// Controller 方法干干净净，只写业务
@PostMapping("/bind")
public Result<Void> bindDevice(String code) {
    deviceService.deviceActivation(code);  // 抛异常？不管，有人兜底
    return new Result<>();
}
```

### 13.2 工作原理

`@RestControllerAdvice` 告诉 Spring：**这个类是所有 Controller 的"异常兜底网"**。任何 Controller 方法抛出的异常，都会被这里的 `@ExceptionHandler` 方法按类型匹配并处理。

```
                    异常从哪里抛出都可以
                    ────────────────────

  Controller                   Service                    Dao
       │                          │                        │
  throw new                 throw new                 数据库报
  RenException(...)         RenException(...)         DuplicateKeyException
       │                          │                        │
       └──────────────────────────┼────────────────────────┘
                                  │
                            异常向上冒泡
                                  │
                                  ▼
                  ┌────────────────────────────────────┐
                  │    RenExceptionHandler              │
                  │    @RestControllerAdvice            │
                  │                                    │
                  │    按异常类型匹配处理方法：           │
                  │                                    │
                  │    RenException                    │
                  │    → handleRenException()          │
                  │    → {"code":10062, "msg":"激活码错误"}│
                  │                                    │
                  │    DuplicateKeyException           │
                  │    → handleDuplicateKeyException() │
                  │    → {"code":10002, "msg":"数据已存在"}│
                  │                                    │
                  │    UnauthorizedException           │
                  │    → handleUnauthorizedException() │
                  │    → {"code":403, "msg":"没有权限"}  │
                  │                                    │
                  │    MethodArgumentNotValidException │
                  │    → 参数校验失败                   │
                  │    → {"code":10034, "msg":"xxx不能为空"}│
                  │                                    │
                  │    NoResourceFoundException        │
                  │    → 404 资源不存在                 │
                  │    → {"code":404, "msg":"资源不存在"} │
                  │                                    │
                  │    Exception（兜底）                │
                  │    → handleException()             │
                  │    → {"code":500, "msg":"系统错误"}  │
                  │                                    │
                  └────────────────────────────────────┘
                                  │
                                  ▼
                          前端收到统一 JSON
```

### 13.3 异常匹配优先级

Spring 按**异常类型的精确度**匹配，越具体的优先：

```
RenException extends RuntimeException extends Exception
     ①                                       ⑥

DuplicateKeyException extends DataAccessException extends RuntimeException extends Exception
     ②                                                                            ⑥

UnauthorizedException extends AuthorizationException extends ShiroException extends Exception
     ③                                                                            ⑥

NoResourceFoundException extends Exception
     ⑤                          ⑥

MethodArgumentNotValidException extends Exception
     ④                              ⑥
```

如果抛出的是 `RenException`，先匹配 `①`；如果没有匹配上任何具体类型，最终兜底到 `⑥ Exception.class`。

### 13.4 源码逐方法解读

```java
@Slf4j
@AllArgsConstructor
@RestControllerAdvice   // ← 关键注解：全局异常拦截 + 返回 JSON
public class RenExceptionHandler {

    // ① 处理业务异常（代码中主动 throw new RenException(...)）
    @ExceptionHandler(RenException.class)
    public Result<Void> handleRenException(RenException ex) {
        Result<Void> result = new Result<>();
        result.error(ex.getCode(), ex.getMsg());
        return result;
        // 示例：throw new RenException(ErrorCode.ACTIVATION_CODE_ERROR)
        // 返回：{"code":10062, "msg":"激活码错误"}
    }

    // ② 处理数据库唯一键冲突（INSERT 重复数据时自动抛出）
    @ExceptionHandler(DuplicateKeyException.class)
    public Result<Void> handleDuplicateKeyException(DuplicateKeyException ex) {
        Result<Void> result = new Result<>();
        result.error(ErrorCode.DB_RECORD_EXISTS);
        return result;
        // 返回：{"code":10002, "msg":"数据已存在"}
    }

    // ③ 处理 Shiro 权限不足（@RequiresPermissions 检查失败时抛出）
    @ExceptionHandler(UnauthorizedException.class)
    public Result<Void> handleUnauthorizedException(UnauthorizedException ex) {
        Result<Void> result = new Result<>();
        result.error(ErrorCode.FORBIDDEN);
        return result;
        // 返回：{"code":403, "msg":"没有权限"}
    }

    // ④ 处理参数校验失败（@Valid + @NotBlank 等校验不通过时抛出）
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleMethodArgumentNotValidException(MethodArgumentNotValidException ex) {
        // 提取第一条校验错误消息
        String errorMsg = ex.getBindingResult().getAllErrors().stream()
                .map(err -> err.getDefaultMessage())
                .filter(Objects::nonNull)
                .findFirst()
                .orElse("参数不能为空");
        return new Result<Void>().error(ErrorCode.PARAM_VALUE_NULL, errorMsg);
        // 示例：@NotBlank(message = "用户名不能为空")
        // 返回：{"code":10034, "msg":"用户名不能为空"}
    }

    // ⑤ 处理 404（请求路径不存在）
    @ExceptionHandler(NoResourceFoundException.class)
    public Result<Void> handleNoResourceFoundException(NoResourceFoundException ex) {
        return new Result<Void>().error(404, "资源不存在");
    }

    // ⑥ 兜底：所有未被上面捕获的异常
    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception ex) {
        log.error(ex.getMessage(), ex);  // 记录完整堆栈到日志
        return new Result<Void>().error();
        // 返回：{"code":500, "msg":"系统错误"}
    }
}
```

---

## 14. XssFilter — XSS 攻击防护

### 14.1 什么是 XSS 攻击

XSS（Cross-Site Scripting，跨站脚本攻击）是指攻击者在输入框中注入恶意 JavaScript 代码。如果后端不做处理直接存入数据库，其他用户查看这条数据时，浏览器会执行这段恶意代码。

```
攻击示例：

用户在"设备别名"输入框输入：
  <script>alert('你的Cookie被偷了')</script>

如果后端不过滤，直接存入数据库，其他用户查看设备列表时：
  浏览器渲染这段 HTML → 执行 <script> → 弹窗 → 甚至偷取 Cookie 发给攻击者
```

### 14.2 XssFilter 的工作原理

本项目通过 Servlet Filter 机制，在**请求到达 Controller 之前**，对所有参数做 HTML 清洗：

```
前端请求（可能包含恶意代码）
  │
  ▼
XssFilter.doFilter()
  │
  ├── 路径在排除列表中？（如 /ota/**）
  │   └── 是 → 直接放行，不过滤
  │
  └── 不在排除列表中
      │
      ▼
      用 XssHttpServletRequestWrapper 包装原始请求
      │
      │  这个包装器重写了以下方法：
      │
      ├── getInputStream()     → 对 JSON Body 做 XSS 清洗
      ├── getParameter()       → 对 URL 查询参数做 XSS 清洗
      ├── getParameterValues() → 对多值参数做 XSS 清洗
      ├── getParameterMap()    → 对参数 Map 做 XSS 清洗
      └── getHeader()          → 对请求头做 XSS 清洗
      │
      │  所有清洗都调用 XssUtils.filter()
      │  内部使用 Jsoup 白名单，只保留安全的 HTML 标签
      │
      ▼
  ShiroFilter（拿到的已经是清洗后的数据）
      │
      ▼
  Controller（拿到的参数是安全的）
```

### 14.3 清洗效果示例

```
清洗前（用户输入/攻击者注入）          清洗后（XssUtils.filter 处理）
────────────────────────────          ─────────────────────────────
<script>alert('xss')</script>    →   （空字符串，script 标签被移除）
<img onerror="alert(1)">        →   <img>（移除危险属性）
<a href="javascript:void(0)">   →   <a>（移除 javascript 协议）
正常文本                         →   正常文本（不受影响）
```

### 14.4 涉及的文件与职责

```
┌── XSS 防护体系 ──────────────────────────────────────────────────────┐
│                                                                      │
│  XssConfig.java                                                      │
│  └── 当 renren.xss.enabled=true 时注册 XssFilter 到 Servlet 容器     │
│                                                                      │
│  XssProperties.java                                                  │
│  └── 读取配置：开关（enabled）和排除 URL 列表（excludeUrls）           │
│                                                                      │
│  XssFilter.java                                                      │
│  └── Servlet 过滤器入口                                              │
│      ├── shouldNotFilter()：判断 URL 是否在排除列表中                 │
│      ├── 在排除列表 → chain.doFilter(原始 request)                   │
│      └── 不在列表   → chain.doFilter(new XssHttpServletRequestWrapper)│
│                                                                      │
│  XssHttpServletRequestWrapper.java                                   │
│  └── 包装原始请求，重写 getInputStream/getParameter/getHeader 等     │
│      所有读取参数的方法都经过 xssEncode() 清洗                        │
│                                                                      │
│  XssUtils.java                                                       │
│  └── 基于 Jsoup Safelist 做 HTML 白名单过滤                         │
│                                                                      │
│  SqlFilter.java                                                      │
│  └── SQL 注入防护，检测并剥离 SQL 关键字（SELECT、DROP 等）           │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 14.5 XssFilter 在请求链路中的位置

```
前端请求
  │
  ▼ ① XssFilter（XSS 清洗）     ← order = Integer.MAX_VALUE（最低优先级中较高）
  │    清洗参数中的恶意 HTML/JS
  │
  ▼ ② ShiroFilter（安全认证）    ← order = Integer.MAX_VALUE - 1（紧随其后）
  │    验证 Token / server.secret
  │
  ▼ ③ DispatcherServlet（路由）
  │    匹配 Controller 方法
  │
  ▼ ④ Controller
  │    此时拿到的参数已经是安全的
  │
  ▼ ⑤ Service → Dao → MySQL
      存入数据库的数据不含恶意代码
```

---

# 附录 A：注解速查手册

## A.1 Spring / Spring Boot 注解

| 注解 | 位置 | 作用 | 本项目示例 |
|------|------|------|-----------|
| `@SpringBootApplication` | 启动类 | 启用自动配置 + 组件扫描 + 配置类 | `AdminApplication` |
| `@Configuration` | 类 | 声明为配置类，内部可定义 `@Bean` | `ShiroConfig`、`RedisConfig` |
| `@Bean` | 方法 | 方法返回值作为 Bean 注册到容器 | `RedisConfig.redisTemplate()` |
| `@Component` | 类 | 通用组件，Spring 管理其生命周期 | `RedisUtils`、`FieldMetaObjectHandler` |
| `@Service` | 类 | 业务逻辑层标记（语义化的 `@Component`） | `DeviceServiceImpl` |
| `@RestController` | 类 | REST 控制器（`@Controller` + `@ResponseBody`） | `DeviceController` |
| `@RequestMapping` | 类/方法 | 定义 URL 路径前缀 | `@RequestMapping("/device")` |
| `@GetMapping` | 方法 | 处理 GET 请求 | `getUserDevices()` |
| `@PostMapping` | 方法 | 处理 POST 请求 | `bindDevice()` |
| `@PutMapping` | 方法 | 处理 PUT 请求 | `updateDeviceInfo()` |
| `@PathVariable` | 参数 | 从 URL 路径取参数 | `/bind/{agentId}` → `@PathVariable String agentId` |
| `@RequestBody` | 参数 | 从请求 JSON Body 反序列化为对象 | `@RequestBody LoginDTO login` |
| `@RequestParam` | 参数 | 从 URL 查询参数取值 | `?page=1` → `@RequestParam int page` |
| `@Autowired` | 字段/构造器 | 自动注入依赖 | `BaseServiceImpl` 中注入 `baseDao` |
| `@Resource` | 字段 | 按名称注入（Java 标准注解） | `Oauth2Realm` 中注入 `ShiroService` |
| `@Lazy` | 字段/参数 | 延迟初始化，避免循环依赖 | `Oauth2Realm` 中 `@Lazy @Resource ShiroService` |
| `@Value` | 字段 | 从配置文件注入值 | `RedisAspect` 中 `@Value("${renren.redis.open}")` |
| `@Async` | 方法 | 方法异步执行（在线程池中运行） | `DeviceServiceImpl.updateDeviceConnectionInfo()` |
| `@Transactional` | 方法/类 | 开启事务，异常时自动回滚 | `BaseServiceImpl.insert()` |
| `@RestControllerAdvice` | 类 | 全局异常处理 + 全局数据绑定 | `RenExceptionHandler` |
| `@ExceptionHandler` | 方法 | 处理指定类型的异常 | `handleRenException(RenException ex)` |
| `@EnableAsync` | 配置类 | 启用 `@Async` 异步支持 | `AsyncConfig` |
| `@EnableScheduling` | 配置类 | 启用 `@Scheduled` 定时任务 | `RAGTaskConfig` |
| `@Aspect` | 类 | 声明为 AOP 切面类 | `RedisAspect` |
| `@Around` | 方法 | AOP 环绕通知 | `RedisAspect.around()` |
| `@ConditionalOnProperty` | 类 | 当配置项为特定值时才生效 | `XssConfig` |
| `@DependsOn` | 类/方法 | 指定 Bean 初始化顺序依赖 | `SystemInitConfig` 依赖 `liquibase` |

## A.2 Lombok 注解

Lombok 在**编译时**自动生成代码，减少手写 getter/setter/构造器等样板代码。

| 注解 | 位置 | 自动生成 | 本项目示例 |
|------|------|----------|-----------|
| `@Data` | 类 | `getter` + `setter` + `toString` + `equals` + `hashCode` | 几乎所有 Entity 和 DTO |
| `@Getter` | 类/字段 | 仅 getter 方法 | 枚举类 |
| `@Setter` | 类/字段 | 仅 setter 方法 | — |
| `@Slf4j` | 类 | 创建 `private static final Logger log` 日志对象 | `DeviceServiceImpl`、`RenExceptionHandler` |
| `@AllArgsConstructor` | 类 | 包含所有字段的构造器（配合 Spring 构造器注入） | `DeviceServiceImpl`、`ShiroServiceImpl` |
| `@RequiredArgsConstructor` | 类 | 包含所有 `final` 字段的构造器 | `ServerSecretFilter` |
| `@NoArgsConstructor` | 类 | 无参构造器 | — |
| `@Builder` | 类 | 建造者模式 | — |

**Lombok 前后对比：**

```java
// 不用 Lombok：需要手写约 50 行代码
public class UserDetail implements Serializable {
    private Long id;
    private String username;
    private Integer superAdmin;
    private String token;
    private Integer status;

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    // ... 还有 3 个字段的 getter/setter，以及 toString、equals、hashCode
}

// 用 Lombok：只需 1 个注解
@Data
public class UserDetail implements Serializable {
    private Long id;
    private String username;
    private Integer superAdmin;
    private String token;
    private Integer status;
}
// 编译时自动生成所有 getter/setter/toString/equals/hashCode
```

**`@Slf4j` 的效果：**

```java
// 不用 Lombok
public class DeviceServiceImpl {
    private static final Logger log = LoggerFactory.getLogger(DeviceServiceImpl.class);
}

// 用 @Slf4j
@Slf4j
public class DeviceServiceImpl {
    // 自动生成上面那行代码，直接用 log.info("xxx")
}
```

**`@AllArgsConstructor` 与 Spring 构造器注入配合：**

```java
// 不用 Lombok
@Service
public class DeviceServiceImpl {
    private final DeviceDao deviceDao;
    private final RedisUtils redisUtils;
    private final OtaService otaService;

    public DeviceServiceImpl(DeviceDao deviceDao, RedisUtils redisUtils, OtaService otaService) {
        this.deviceDao = deviceDao;
        this.redisUtils = redisUtils;
        this.otaService = otaService;
    }
}

// 用 @AllArgsConstructor：自动生成上面的构造器
@Slf4j
@Service
@AllArgsConstructor
public class DeviceServiceImpl {
    private final DeviceDao deviceDao;       // Spring 自动注入
    private final RedisUtils redisUtils;     // Spring 自动注入
    private final OtaService otaService;     // Spring 自动注入
}
```

## A.3 MyBatis-Plus 注解

| 注解 | 位置 | 作用 | 本项目示例 |
|------|------|------|-----------|
| `@TableName("表名")` | Entity 类 | 指定对应的数据库表 | `@TableName("sys_user_token")` |
| `@TableId` | Entity 字段 | 标记主键字段 | `BaseEntity.id` |
| `@TableField` | Entity 字段 | 指定数据库字段映射 | `@TableField(fill = FieldFill.INSERT)` |
| `@TableField(fill = FieldFill.INSERT)` | Entity 字段 | 插入时自动填充 | `creator`、`createDate` |
| `@TableField(fill = FieldFill.INSERT_UPDATE)` | Entity 字段 | 插入和更新时自动填充 | `updater`、`updateDate` |
| `@TableField(exist = false)` | Entity 字段 | 该字段不对应数据库列 | 临时计算字段 |
| `@Mapper` | Dao 接口 | 标记为 MyBatis Mapper，Spring 自动代理 | `DeviceDao`、`SysUserDao` |

## A.4 Shiro 注解

| 注解 | 位置 | 作用 | 本项目示例 |
|------|------|------|-----------|
| `@RequiresPermissions("权限字符串")` | Controller 方法 | 检查当前用户是否具有指定权限 | `@RequiresPermissions("sys:role:normal")` |
| `@RequiresRoles("角色")` | Controller 方法 | 检查当前用户是否具有指定角色 | 本项目未使用 |
| `@RequiresAuthentication` | Controller 方法 | 要求当前用户已认证 | 本项目未使用 |

## A.5 Swagger / OpenAPI 注解

| 注解 | 位置 | 作用 | 本项目示例 |
|------|------|------|-----------|
| `@Tag(name = "分组名")` | Controller 类 | API 文档分组标题 | `@Tag(name = "设备管理")` |
| `@Operation(summary = "描述")` | Controller 方法 | API 文档接口描述 | `@Operation(summary = "绑定设备")` |
| `@Schema(description = "描述")` | DTO/VO 类或字段 | 字段说明 | `TokenDTO` 中 `@Schema(description = "密码")` |

## A.6 Validation 校验注解

| 注解 | 位置 | 作用 | 本项目示例 |
|------|------|------|-----------|
| `@Valid` | Controller 参数 | 触发参数校验 | `@Valid @RequestBody DeviceUpdateDTO dto` |
| `@NotBlank` | DTO 字段 | 不为空且不为空白串 | `LoginDTO.username` |
| `@NotNull` | DTO 字段 | 不为 null | — |
| `@Size(min, max)` | DTO 字段 | 字符串长度范围 | — |
| `@Pattern(regexp)` | DTO 字段 | 正则匹配 | — |
