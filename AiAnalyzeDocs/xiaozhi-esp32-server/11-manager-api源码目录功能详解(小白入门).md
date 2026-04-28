# manager-api 源码目录速查手册

> 纯目录查阅工具，逐文件说明 manager-api 源码目录结构与每个文件的功能。
> 共覆盖 **293+ 个 Java 文件**和 **123 个资源文件**。
>
> **相关文档：**
> - 架构原理、Spring Boot 核心概念、AOP、全局异常、XSS 防护、注解速查 → 见 `12-Spring Boot入门与项目架构原理.md`
> - OAuth2 认证完整流程 → 见 `10-OAuth2用户Token认证完整流程.md`

---

## 目录

- [1. 技术栈速览](#1-技术栈速览)
- [2. 项目整体目录结构](#2-项目整体目录结构)
- [3. 启动入口](#3-启动入口)
- [4. common — 通用基础层（58 个文件）](#4-common--通用基础层58-个文件)
- [5. modules — 业务模块层（235 个文件）](#5-modules--业务模块层235-个文件)
  - [5.1 agent — 智能体模块（69 个文件）](#51-agent--智能体模块69-个文件)
  - [5.2 config — 配置中枢模块（5 个文件）](#52-config--配置中枢模块5-个文件)
  - [5.3 device — 设备管理模块（22 个文件）](#53-device--设备管理模块22-个文件)
  - [5.4 knowledge — 知识库模块（32 个文件）](#54-knowledge--知识库模块32-个文件)
  - [5.5 llm — 大语言模型模块（2 个文件）](#55-llm--大语言模型模块2-个文件)
  - [5.6 model — 模型配置模块（16 个文件）](#56-model--模型配置模块16-个文件)
  - [5.7 security — 安全认证模块（25 个文件）](#57-security--安全认证模块25-个文件)
  - [5.8 sms — 短信服务模块（2 个文件）](#58-sms--短信服务模块2-个文件)
  - [5.9 sys — 系统管理模块（46 个文件）](#59-sys--系统管理模块46-个文件)
  - [5.10 timbre — 音色管理模块（8 个文件）](#510-timbre--音色管理模块8-个文件)
  - [5.11 voiceclone — 声音克隆模块（8 个文件）](#511-voiceclone--声音克隆模块8-个文件)
- [6. resources — 资源文件（123 个文件）](#6-resources--资源文件123-个文件)
- [7. pom.xml — 项目依赖](#7-pomxml--项目依赖)
- [8. 分层架构说明](#8-分层架构说明)

---

## 1. 技术栈速览

| 技术 | 版本/说明 | 用途 |
|------|----------|------|
| Java | 21 (LTS) | 编程语言 |
| Spring Boot | 3.4.3 | Web 框架 |
| MyBatis-Plus | — | ORM 数据访问 |
| Druid | — | 数据库连接池 |
| MySQL | — | 关系型数据库 |
| Redis + Lettuce | — | 缓存与分布式数据 |
| Apache Shiro | jakarta | 安全认证与授权 |
| Liquibase | — | 数据库版本迁移 |
| Knife4j + SpringDoc | — | API 文档（Swagger） |
| Lombok | — | 减少样板代码 |
| Hutool | — | Java 工具库 |
| Maven | — | 构建工具 |

---

## 2. 项目整体目录结构

```
manager-api/
├── pom.xml                          # Maven 构建配置与依赖声明
├── src/main/java/xiaozhi/
│   ├── AdminApplication.java        # Spring Boot 启动入口
│   ├── common/                      # 通用基础层（58 个文件）
│   │   ├── annotation/              #   自定义注解
│   │   ├── aspect/                  #   AOP 切面
│   │   ├── config/                  #   全局配置类
│   │   ├── constant/                #   全局常量
│   │   ├── convert/                 #   类型转换器
│   │   ├── dao/                     #   Dao 基类
│   │   ├── entity/                  #   Entity 基类
│   │   ├── exception/               #   异常与全局处理
│   │   ├── handler/                 #   MyBatis-Plus 字段填充
│   │   ├── interceptor/             #   数据权限拦截器
│   │   ├── page/                    #   分页与 Token DTO
│   │   ├── redis/                   #   Redis 配置与工具
│   │   ├── service/                 #   Service 基类与接口
│   │   ├── user/                    #   用户信息载体
│   │   ├── utils/                   #   工具类集合
│   │   ├── validator/               #   参数校验工具
│   │   └── xss/                     #   XSS/SQL 注入防护
│   └── modules/                     # 业务模块层（235 个文件）
│       ├── agent/                   #   智能体（69 文件）
│       ├── config/                  #   配置中枢（5 文件）
│       ├── device/                  #   设备管理（22 文件）
│       ├── knowledge/               #   知识库（32 文件）
│       ├── llm/                     #   LLM 服务（2 文件）
│       ├── model/                   #   模型配置（16 文件）
│       ├── security/                #   安全认证（25 文件）
│       ├── sms/                     #   短信服务（2 文件）
│       ├── sys/                     #   系统管理（46 文件）
│       ├── timbre/                  #   音色管理（8 文件）
│       └── voiceclone/              #   声音克隆（8 文件）
└── src/main/resources/
    ├── application.yml              # 主配置文件
    ├── application-dev.yml          # 开发环境配置
    ├── logback-spring.xml           # 日志配置
    ├── mapper/                      # MyBatis XML 映射（16 文件）
    ├── db/changelog/                # Liquibase 数据库迁移（88 文件）
    ├── i18n/                        # 国际化文案（14 文件）
    └── lua/                         # Redis Lua 脚本（2 文件）
```

---

## 3. 启动入口

| 文件 | 说明 |
|------|------|
| `AdminApplication.java` | Spring Boot 启动类，`@SpringBootApplication` 开启组件扫描与自动配置。启动后打印 `http://localhost:8002/xiaozhi/doc.html`（Knife4j API 文档地址） |

---

## 4. common — 通用基础层（58 个文件）

通用层提供整个项目的基础设施，**所有业务模块都依赖这一层**。

### 4.1 common/annotation/ — 自定义注解

| 文件 | 说明 |
|------|------|
| `DataFilter.java` | 数据权限过滤注解，标记在方法上，配置表别名与用户/部门字段名 |
| `LogOperation.java` | 操作日志注解，通过 `value` 指定日志描述文案 |

### 4.2 common/aspect/ — AOP 切面

| 文件 | 说明 |
|------|------|
| `RedisAspect.java` | 环绕 `RedisUtils` 所有方法，当 `renren.redis.open=true` 时才执行 Redis 调用，异常时抛 `REDIS_ERROR` |

### 4.3 common/config/ — 全局配置类

| 文件 | 说明 |
|------|------|
| `SwaggerConfig.java` | 注册 SpringDoc 多组 API 分组（device、agent、models、ota 等）及全局文档信息 |
| `RestTemplateConfig.java` | 创建 `RestTemplate` Bean，使用 JDK HTTP 客户端，读超时 30 秒 |
| `AsyncConfig.java` | 启用 `@Async` 异步、`@EnableAspectJAutoProxy(exposeProxy=true)` AOP 代理暴露，注册线程池 |
| `MybatisPlusConfig.java` | 组装 MyBatis-Plus 插件链：数据权限拦截、分页、乐观锁、防全表更新删除 |

### 4.4 common/constant/ — 全局常量

| 文件 | 说明 |
|------|------|
| `Constant.java` | 全局常量与配置键（如 `AUTHORIZATION`、`SERVER_SECRET`、`SM2_PUBLIC_KEY`），以及多种枚举（训练状态、短信参数、数据操作标记等） |

### 4.5 common/convert/ — 类型转换

| 文件 | 说明 |
|------|------|
| `DateConverter.java` | `Converter<String, Date>` 实现，支持多种日期字符串格式自动转换 |

### 4.6 common/dao/ — Dao 基类

| 文件 | 说明 |
|------|------|
| `BaseDao.java` | 继承 MyBatis-Plus `BaseMapper<T>` 的泛型基接口，所有 Dao 可继承此接口获得通用 CRUD |

### 4.7 common/entity/ — 实体基类

| 文件 | 说明 |
|------|------|
| `BaseEntity.java` | 所有实体的抽象父类，包含 `id`（主键）、`creator`（创建者）、`createDate`（创建时间），支持 MyBatis-Plus 自动填充 |

### 4.8 common/exception/ — 异常处理

| 文件 | 说明 |
|------|------|
| `ErrorCode.java` | 全部错误码常量定义（5 位数字，前 2 位为模块编码），约 200+ 个错误码 |
| `RenException.java` | 自定义运行时异常，带错误码 `code` 和国际化消息 `msg` |
| `RenExceptionHandler.java` | `@RestControllerAdvice` 全局异常处理器，捕获所有异常统一转为 `Result` JSON 响应 |

### 4.9 common/handler/ — 字段自动填充

| 文件 | 说明 |
|------|------|
| `FieldMetaObjectHandler.java` | MyBatis-Plus `MetaObjectHandler` 实现，插入时自动填充 `creator`、`createDate`、`updater`、`updateDate`、`dataOperation` |

### 4.10 common/interceptor/ — 数据权限拦截

| 文件 | 说明 |
|------|------|
| `DataScope.java` | 封装一条 SQL 过滤条件片段 |
| `DataFilterInterceptor.java` | MyBatis 拦截器，查询前用 JSQLParser 将 `DataScope` 条件拼入 WHERE 子句 |

### 4.11 common/page/ — 分页与令牌

| 文件 | 说明 |
|------|------|
| `PageData.java` | 泛型分页结果封装，包含总条数 `total` 和数据列表 `list` |
| `TokenDTO.java` | 登录成功后返回给前端的令牌信息：`token`（值）、`expire`（过期秒数）、`clientHash`（客户端指纹） |

### 4.12 common/redis/ — Redis 配置与工具

| 文件 | 说明 |
|------|------|
| `RedisConfig.java` | 配置 `RedisTemplate<String, Object>`，键用 String 序列化，值用 JSON 序列化 |
| `RedisUtils.java` | Redis 操作工具类，封装 String/Hash/List 读写、过期设置、Lua 脚本执行等 |
| `RedisKeys.java` | 各业务场景的 Redis 键名生成方法（验证码、激活码、配置缓存、设备连接时间等） |

### 4.13 common/service/ — Service 基类

| 文件 | 说明 |
|------|------|
| `BaseService.java` | 服务层基础接口，定义 `insert`、`updateById`、`selectById`、`deleteById`、批量操作等通用方法 |
| `CrudService.java` | 在 `BaseService` 上扩展 `page`（分页）、`list`（列表）、`get`、`save`、`update`、`delete` |

### 4.14 common/service/impl/ — Service 基类实现

| 文件 | 说明 |
|------|------|
| `BaseServiceImpl.java` | `BaseService` 抽象实现，含分页参数解析、排序构建、`PageData` 组装、批量 SQL 执行，通过 `@Autowired` 注入 `baseDao` |
| `CrudServiceImpl.java` | `CrudService` 抽象实现，分页/列表委托抽象方法 `getWrapper()`，实体与 DTO 用 `ConvertUtils` 转换 |

### 4.15 common/user/ — 用户信息载体

| 文件 | 说明 |
|------|------|
| `UserDetail.java` | 登录用户信息：`id`、`username`、`superAdmin`（是否管理员）、`token`、`status`（状态） |

### 4.16 common/utils/ — 工具类集合（20 个文件）

| 文件 | 说明 |
|------|------|
| `Result.java` | 统一响应体 `{code, msg, data}`，所有 Controller 方法的返回类型 |
| `ResultUtils.java` | 静态工厂方法构造成功/失败的 `Result` |
| `JsonUtils.java` | 基于 Jackson 的 JSON 序列化/反序列化工具 |
| `ConvertUtils.java` | 基于 `BeanUtils` 的单对象与集合属性拷贝 |
| `DateUtils.java` | 日期格式化、解析、相对时间中文描述（如"3分钟前"） |
| `MessageUtils.java` | 通过 `MessageSource` 按错误码获取国际化文案 |
| `HttpContextUtils.java` | 获取当前请求对象、Bearer Token、Origin、客户端指纹、URI 等 |
| `IpUtils.java` | 从请求头解析客户端真实 IP（考虑 X-Forwarded-For 等代理头） |
| `SpringContextUtils.java` | 保存 `ApplicationContext`，提供按名/类型获取 Bean 的静态方法 |
| `SM2Utils.java` | 国密 SM2 加解密（十六进制密文）及密钥对生成 |
| `Sm2DecryptUtil.java` | 用 SM2 私钥解密前端传来的密码，拆分验证码与真实密码并校验 |
| `AESUtils.java` | AES/ECB/PKCS5Padding 加解密，密文为 Base64 字符串 |
| `HashEncryptionUtil.java` | MD5 及指定算法的十六进制消息摘要 |
| `ToolUtil.java` | String、List、Map、Set、数组的空/非空判断 |
| `TreeNode.java` | 泛型树节点基类（id、pid、子节点列表） |
| `TreeUtils.java` | 扁平列表按父 ID 构建树形结构 |
| `ResourcesUtils.java` | 从 classpath 读取文本资源（如 Lua 脚本） |
| `SensitiveDataUtils.java` | 敏感字段识别、字符串掩码、JSON 脱敏 |
| `JsonRpcTwo.java` | JSON-RPC 2.0 请求结构体（jsonrpc、method、params、id） |
| `PropertiesUtils.java` | 配置读取预留工具类 |

### 4.17 common/validator/ — 参数校验

| 文件 | 说明 |
|------|------|
| `AssertUtils.java` | 断言工具，空串/null/空集合时抛 `RenException` |
| `ValidatorUtils.java` | 使用 Hibernate Validator 做 Bean 分组校验，并提供国际手机号格式校验 |

### 4.18 common/validator/group/ — 校验分组

| 文件 | 说明 |
|------|------|
| `AddGroup.java` | 新增操作校验分组标记接口 |
| `UpdateGroup.java` | 修改操作校验分组标记接口 |
| `DefaultGroup.java` | 默认校验分组标记接口 |

### 4.19 common/xss/ — XSS/SQL 防护（6 个文件）

| 文件 | 说明 |
|------|------|
| `XssConfig.java` | 当 `renren.xss.enabled=true` 时注册 `XssFilter` 到所有请求路径 |
| `XssProperties.java` | 绑定 `renren.xss` 配置前缀（启用开关、排除 URL 列表） |
| `XssFilter.java` | Servlet 过滤器，对未排除的 URL 用 `XssHttpServletRequestWrapper` 包装请求 |
| `XssHttpServletRequestWrapper.java` | 包装请求，对 JSON Body、参数、Header 做白名单 HTML 清洗 |
| `XssUtils.java` | 基于 Jsoup `Safelist` 定义白名单并清洗 HTML 文本 |
| `SqlFilter.java` | 字符剥离与 SQL 关键字检测，防止简单 SQL 注入 |

---

## 5. modules — 业务模块层（235 个文件）

每个业务模块遵循统一分层：`controller/` → `service/` → `dao/` → `entity/`，辅以 `dto/`（请求参数）和 `vo/`（响应视图）。

### 5.1 agent — 智能体模块（69 个文件）

智能体是本项目的核心业务概念，对应一个可对话的 AI 助手实例。

#### controller/

| 文件 | 说明 |
|------|------|
| `AgentController.java` | 智能体管理 REST 接口：列表、创建、更新、删除、记忆、插件、上下文、会话等 |
| `AgentChatHistoryController.java` | 聊天历史上报（供 Python 服务调用）、下载、会话列表、导出 |
| `AgentMcpAccessPointController.java` | 查询智能体 MCP 接入地址与工具列表 |
| `AgentTemplateController.java` | 超管侧智能体模板的分页与增删改查 |
| `AgentVoicePrintController.java` | 智能体声纹的增删改查列表 |

#### dao/

| 文件 | 说明 |
|------|------|
| `AgentDao.java` | 智能体表 Mapper，含设备数统计、按 MAC 查默认智能体等自定义 SQL |
| `AgentChatTitleDao.java` | 会话标题表 Mapper |
| `AgentContextProviderDao.java` | 上下文源配置数据访问 |
| `AgentPluginMappingMapper.java` | 智能体与插件映射关系 Mapper |
| `AgentTagDao.java` | 标签数据访问 |
| `AgentTagRelationDao.java` | 智能体与标签关联数据访问 |
| `AgentTemplateDao.java` | 模板表 Mapper |
| `AgentVoicePrintDao.java` | 声纹表 Mapper |
| `AiAgentChatAudioDao.java` | 聊天音频表 Mapper |
| `AiAgentChatHistoryDao.java` | 聊天历史表 Mapper |

#### dto/

| 文件 | 说明 |
|------|------|
| `AgentChatHistoryDTO.java` | 单条聊天记录展示字段 |
| `AgentChatHistoryReportDTO.java` | 设备聊天上报请求体（MAC、会话、类型、内容、音频 Base64） |
| `AgentChatSessionDTO.java` | 会话列表项（会话 ID、时间、条数、标题） |
| `AgentChatSummaryDTO.java` | 会话总结结果封装 |
| `AgentCreateDTO.java` | 创建智能体请求体 |
| `AgentDTO.java` | 智能体列表/详情传输对象 |
| `AgentMemoryDTO.java` | 更新智能体记忆文本 |
| `AgentTagDTO.java` | 标签 ID 与名称 |
| `AgentUpdateDTO.java` | 更新智能体请求体（部分字段覆盖） |
| `AgentVoicePrintSaveDTO.java` | 新增声纹请求体 |
| `AgentVoicePrintUpdateDTO.java` | 修改声纹请求体 |
| `ContextProviderDTO.java` | 上下文源 URL 与请求头 |
| `IdentifyVoicePrintResponse.java` | 声纹识别服务返回结构 |
| `McpJsonRpcResponse.java` | MCP JSON-RPC 响应结构体 |

#### entity/

| 文件 | 说明 |
|------|------|
| `AgentEntity.java` | 表 `ai_agent`，智能体主配置（各模型 ID、音色、系统提示词、记忆等） |
| `AgentChatAudioEntity.java` | 表 `ai_agent_chat_audio`，聊天 Opus 音频二进制存储 |
| `AgentChatHistoryEntity.java` | 表 `ai_agent_chat_history`，聊天记录持久化 |
| `AgentChatTitleEntity.java` | 表 `ai_agent_chat_title`，会话标题 |
| `AgentContextProviderEntity.java` | 表 `ai_agent_context_provider`，上下文源配置 |
| `AgentPluginMapping.java` | 表 `ai_agent_plugin_mapping`，智能体与插件/知识库映射 |
| `AgentTagEntity.java` | 表 `ai_agent_tag`，标签定义 |
| `AgentTagRelationEntity.java` | 表 `ai_agent_tag_relation`，智能体与标签关联 |
| `AgentTemplateEntity.java` | 表 `ai_agent_template`，默认模板配置 |
| `AgentVoicePrintEntity.java` | 表 `ai_agent_voice_print`，声纹关联 |

#### service/ 与 service/impl/

| 文件 | 说明 |
|------|------|
| `AgentService.java` / `AgentServiceImpl.java` | 智能体 CRUD、权限校验、与设备/模型/模板/标签联动 |
| `AgentChatAudioService.java` / `AgentChatAudioServiceImpl.java` | 聊天音频字节存取 |
| `AgentChatHistoryService.java` / `AgentChatHistoryServiceImpl.java` | 会话分页、历史查询、按智能体删除 |
| `AgentChatSummaryService.java` / `AgentChatSummaryServiceImpl.java` | 基于 LLM 生成会话摘要与标题 |
| `AgentChatTitleService.java` / `AgentChatTitleServiceImpl.java` | 按 sessionId 保存或查询标题 |
| `AgentContextProviderService.java` / `AgentContextProviderServiceImpl.java` | 上下文源配置 CRUD |
| `AgentMcpAccessPointService.java` / `AgentMcpAccessPointServiceImpl.java` | 组装加密 MCP URL，WebSocket 拉取工具列表 |
| `AgentPluginMappingService.java` / `AgentPluginMappingServiceImpl.java` | 插件映射查询与知识库/模型参数填充 |
| `AgentTagService.java` / `AgentTagServiceImpl.java` | 标签与关联关系维护 |
| `AgentTemplateService.java` / `AgentTemplateServiceImpl.java` | 默认模板、排序、按模型类型更新 |
| `AgentVoicePrintService.java` / `AgentVoicePrintServiceImpl.java` | 声纹 HTTP 对接外部服务 |

#### service/biz/ 与 service/biz/impl/

| 文件 | 说明 |
|------|------|
| `AgentChatHistoryBizService.java` / `AgentChatHistoryBizServiceImpl.java` | 聊天上报业务入口：落库文本/音频、更新 Redis 与设备连接信息 |

#### Enums/

| 文件 | 说明 |
|------|------|
| `AgentChatHistoryType.java` | 聊天记录角色类型枚举（用户/智能体） |
| `XiaoZhiMcpJsonRpcJson.java` | MCP JSON-RPC 初始化/通知/工具列表等请求 JSON 常量 |

#### vo/

| 文件 | 说明 |
|------|------|
| `AgentChatHistoryUserVO.java` | 前端展示的聊天内容与音频 ID |
| `AgentInfoVO.java` | 智能体详情（附插件列表与上下文源） |
| `AgentTemplateVO.java` | 模板详情（附 TTS/LLM 模型名称） |
| `AgentVoicePrintVO.java` | 声纹列表展示字段 |

### 5.2 config — 配置中枢模块（5 个文件）

供 Python 服务（xiaozhi-server）拉取运行时配置。

| 文件 | 说明 |
|------|------|
| `controller/ConfigController.java` | 提供 `POST /config/server-base`（全局配置）和 `POST /config/agent-models`（按设备的智能体模型配置） |
| `dto/AgentModelsDTO.java` | 请求体：MAC 地址、clientId、客户端已实例化模块映射 |
| `init/SystemInitConfig.java` | 启动时校验版本并清空 Redis、初始化 server.secret、预热配置缓存 |
| `service/ConfigService.java` | 配置聚合服务接口 |
| `service/impl/ConfigServiceImpl.java` | 构建 Redis 缓存的服务端配置，按设备解析智能体的 VAD/ASR/LLM/TTS/Memory 等模型清单 |

### 5.3 device — 设备管理模块（22 个文件）

管理 ESP32 等硬件设备的注册、绑定、OTA 升级。

#### controller/

| 文件 | 说明 |
|------|------|
| `DeviceController.java` | 设备绑定、注册验证码、解绑、更新、手动添加、工具调用转发 |
| `OTAController.java` | 设备上报固件与激活状态（anon 匿名访问），返回 OTA/WebSocket/MQTT 配置 |
| `OTAMagController.java` | 超管固件包上传、分页、删除与下载管理 |

#### dao/

| 文件 | 说明 |
|------|------|
| `DeviceDao.java` | 设备表 Mapper，含按智能体查最近连接时间 |
| `OtaDao.java` | OTA 固件表 Mapper |

#### dto/（9 个文件）

| 文件 | 说明 |
|------|------|
| `DeviceBindDTO.java` | MAC、用户与智能体绑定信息 |
| `DeviceManualAddDTO.java` | 管理员手动录入设备的参数 |
| `DevicePageUserDTO.java` | 管理员分页查询设备的参数 |
| `DeviceRegisterDTO.java` | 设备注册请求（仅 MAC） |
| `DeviceReportReqDTO.java` | 设备固件与硬件上报请求体 |
| `DeviceReportRespDTO.java` | OTA 检测响应（时间、激活码、固件、WebSocket、MQTT） |
| `DeviceToolsCallReqDTO.java` | 工具名称与参数，供代调用 |
| `DeviceUnBindDTO.java` | 解绑请求中的设备 ID |
| `DeviceUpdateDTO.java` | 修改自动更新开关与设备别名 |

#### entity/

| 文件 | 说明 |
|------|------|
| `DeviceEntity.java` | 表 `ai_device`，设备与用户、智能体、MAC、连接时间等 |
| `OtaEntity.java` | 表 `ai_ota`，固件元数据与文件路径 |

#### service/ 与 service/impl/

| 文件 | 说明 |
|------|------|
| `DeviceService.java` / `DeviceServiceImpl.java` | 设备在线数据、激活检测、绑定解绑、HMAC Token、MQTT 配置生成等核心逻辑 |
| `OtaService.java` / `OtaServiceImpl.java` | 固件 CRUD、唯一性校验、按类型取最新版本 |

#### vo/

| 文件 | 说明 |
|------|------|
| `DeviceOtaVO.java` | OTA 响应结构封装 |
| `UserShowDeviceListVO.java` | 管理员设备列表展示 |

### 5.4 knowledge — 知识库模块（32 个文件）

对接 RAGFlow 等 RAG 系统，管理知识库与文档。

#### controller/

| 文件 | 说明 |
|------|------|
| `KnowledgeBaseController.java` | 知识库分页、详情、增删改及模型校验 |
| `KnowledgeFilesController.java` | 文档分页、上传、切片、检索 |

#### config/

| 文件 | 说明 |
|------|------|
| `KnowledgeBaseConfig.java` | 注册 `KnowledgeBaseAdapterFactory` Bean |
| `RAGTaskConfig.java` | 启用 `@EnableScheduling` 定时任务 |

#### dao/

| 文件 | 说明 |
|------|------|
| `DocumentDao.java` | 文档影子表数据访问 |
| `KnowledgeBaseDao.java` | 知识库表、插件映射清理、统计更新等 |

#### dto/（8 个文件，含多个子包）

| 文件 | 说明 |
|------|------|
| `DocumentDTO.java` | 文档同步扁平 DTO |
| `KnowledgeBaseDTO.java` | 知识库展示与编辑字段 |
| `KnowledgeFilesDTO.java` | 文档列表与解析进度 |
| `dto/agent/AgentDTO.java` | RAGFlow Agent API 聚合容器 |
| `dto/bot/BotDTO.java` | Bot 相关请求/响应聚合 |
| `dto/chat/ChatCompletionRequest.java` | OpenAI 兼容的聊天补全请求 |
| `dto/chat/ChatDTO.java` | 对话助手与消息聚合 |
| `dto/common/CommonDTO.java` | 通用扩展请求/响应 |
| `dto/dataset/DatasetDTO.java` | 数据集管理聚合 |
| `dto/document/ChunkDTO.java` | 文档切片增删改查聚合 |
| `dto/document/DocumentDTO.java` | RAGFlow 文档聚合 |
| `dto/document/RetrievalDTO.java` | 检索信息聚合 |
| `dto/file/FileDTO.java` | 文件上传管理聚合 |

#### entity/

| 文件 | 说明 |
|------|------|
| `DocumentEntity.java` | 表 `ai_rag_knowledge_document`，RAGFlow 文档影子记录 |
| `KnowledgeBaseEntity.java` | 表 `ai_rag_dataset`，本地知识库与 RAG 模型指针 |

#### rag/ — RAG 适配器

| 文件 | 说明 |
|------|------|
| `KnowledgeBaseAdapter.java` | 知识库后端操作抽象接口 |
| `KnowledgeBaseAdapterFactory.java` | 按类型创建并缓存适配器实例 |
| `RAGFlowClient.java` | RAGFlow HTTP 调用、鉴权与超时封装 |
| `rag/impl/RAGFlowAdapter.java` | RAGFlow 具体 API 实现 |

#### service/ 与 service/impl/

| 文件 | 说明 |
|------|------|
| `KnowledgeBaseService.java` / `KnowledgeBaseServiceImpl.java` | 知识库业务与 RAG 适配器、Redis 联动 |
| `KnowledgeFilesService.java` / `KnowledgeFilesServiceImpl.java` | 文档全生命周期与 RAGFlow/DB 同步 |
| `KnowledgeManagerService.java` / `KnowledgeManagerServiceImpl.java` | 级联删除（先文档再数据集） |

#### task/

| 文件 | 说明 |
|------|------|
| `DocumentStatusSyncTask.java` | 定时同步 RUNNING 文档状态并更新统计 |

### 5.5 llm — 大语言模型模块（2 个文件）

| 文件 | 说明 |
|------|------|
| `service/LLMService.java` | LLM 抽象接口：聊天总结、标题生成 |
| `service/impl/OpenAIStyleLLMServiceImpl.java` | 通过 OpenAI 兼容 HTTP 接口调用配置的 LLM |

### 5.6 model — 模型配置模块（16 个文件）

管理各类型模型（LLM、TTS、ASR、VAD 等）的配置和供应器（厂商）。

#### controller/

| 文件 | 说明 |
|------|------|
| `ModelController.java` | 模型名称查询、LLM 列表、模型配置 CRUD 与刷新模板 |
| `ModelProviderController.java` | 超管维护模型供应器分页与增删改 |

#### dao/

| 文件 | 说明 |
|------|------|
| `ModelConfigDao.java` | 模型配置表 Mapper |
| `ModelProviderDao.java` | 模型供应器表 Mapper |

#### dto/

| 文件 | 说明 |
|------|------|
| `LlmModelBasicInfoDTO.java` | LLM 模型基础信息（含 LLM 类型） |
| `ModelBasicInfoDTO.java` | 模型 ID 与名称简表 |
| `ModelConfigBodyDTO.java` | 创建/编辑模型配置请求体 |
| `ModelConfigDTO.java` | 模型配置展示（含供应器与敏感配置） |
| `ModelProviderDTO.java` | 供应器元数据与字段 JSON |
| `VoiceDTO.java` | 音色简讯（含演示地址、是否克隆） |

#### entity/

| 文件 | 说明 |
|------|------|
| `ModelConfigEntity.java` | 表 `ai_model_config`，各类型模型实例配置 |
| `ModelProviderEntity.java` | 表 `ai_model_provider`，供应器定义与表单字段 |

#### service/ 与 service/impl/

| 文件 | 说明 |
|------|------|
| `ModelConfigService.java` / `ModelConfigServiceImpl.java` | 模型配置与 Redis、智能体、敏感字段处理 |
| `ModelProviderService.java` / `ModelProviderServiceImpl.java` | 供应器维护及插件列表与用户知识库合并 |

### 5.7 security — 安全认证模块（25 个文件）

完整的登录/认证/授权体系，详见 `10-OAuth2用户Token认证完整流程.md`。

#### config/

| 文件 | 说明 |
|------|------|
| `FilterConfig.java` | 将 Shiro `DelegatingFilterProxy` 注册到 Servlet 容器 `/*` |
| `ShiroConfig.java` | Shiro 核心配置：SecurityManager、Realm、过滤器路由规则 |
| `WebMvcConfig.java` | 全局 CORS、Jackson 日期序列化等 WebMvc 配置 |

#### controller/

| 文件 | 说明 |
|------|------|
| `LoginController.java` | 验证码、短信、登录、注册、改密、找回密码、公共配置等 `/user` 接口 |

#### oauth2/

| 文件 | 说明 |
|------|------|
| `Oauth2Filter.java` | 从请求 Header 提取 Bearer Token，交给 Shiro 认证 |
| `Oauth2Realm.java` | 认证（Token 查库）+ 授权（分配权限） |
| `Oauth2Token.java` | 封装 Token 字符串的 Shiro `AuthenticationToken` |
| `TokenGenerator.java` | UUID → MD5 生成 32 位 hex Token 值 |

#### password/

| 文件 | 说明 |
|------|------|
| `BCrypt.java` | Blowfish 加盐哈希算法完整实现 |
| `BCryptPasswordEncoder.java` | BCrypt 的 `PasswordEncoder` 包装 |
| `PasswordEncoder.java` | 密码编码与校验接口 |
| `PasswordUtils.java` | 静态封装 BCrypt 加密与校验 |

#### secret/

| 文件 | 说明 |
|------|------|
| `ServerSecretFilter.java` | 服务间通信过滤器，校验 Bearer == `server.secret` |
| `ServerSecretToken.java` | 服务端密钥对应的 Shiro 认证令牌 |

#### service/ 与 service/impl/

| 文件 | 说明 |
|------|------|
| `CaptchaService.java` / `CaptchaServiceImpl.java` | 图形验证码生成校验、短信验证码发送与限额 |
| `ShiroService.java` / `ShiroServiceImpl.java` | 按 Token 查 token 实体、按 userId 查用户（供 Realm 调用） |
| `SysUserTokenService.java` / `SysUserTokenServiceImpl.java` | Token 生成、刷新、登出失效、改密联动 |

#### user/ 与 dto/ 与 dao/ 与 entity/

| 文件 | 说明 |
|------|------|
| `user/SecurityUser.java` | 从 Shiro Subject 获取当前用户/用户 ID/Token 的工具类 |
| `dto/LoginDTO.java` | 登录表单（用户名、密码、验证码 ID） |
| `dto/SmsVerificationDTO.java` | 短信验证码请求（手机号、图形验证码） |
| `dao/SysUserTokenDao.java` | Token 数据访问（查询、登出更新过期时间） |
| `entity/SysUserTokenEntity.java` | 表 `sys_user_token`，用户 Token 与过期时间 |

### 5.8 sms — 短信服务模块（2 个文件）

| 文件 | 说明 |
|------|------|
| `service/SmsService.java` | 发送短信验证码接口 |
| `service/imp/ALiYunSmsService.java` | 阿里云短信 SDK 实现，含失败回滚计数 |

### 5.9 sys — 系统管理模块（46 个文件）

系统级功能：用户管理、参数管理、字典、WebSocket 管理指令。

#### controller/

| 文件 | 说明 |
|------|------|
| `AdminController.java` | 超管分页用户、查看设备、重置密码、启停用户 |
| `ServerSideManageController.java` | WebSocket 服务列表与向 Python 端下发管理动作（重启/更新配置） |
| `SysDictDataController.java` | 字典数据分页与增删改查 |
| `SysDictTypeController.java` | 字典类型分页与增删改查 |
| `SysParamsController.java` | 系统参数分页、增删改、刷新配置、WebSocket 地址校验 |

#### dao/

| 文件 | 说明 |
|------|------|
| `SysDictDataDao.java` | 字典数据表 Mapper |
| `SysDictTypeDao.java` | 字典类型表 Mapper |
| `SysParamsDao.java` | 系统参数表 Mapper |
| `SysUserDao.java` | 系统用户表 Mapper |

#### dto/（10 个文件）

| 文件 | 说明 |
|------|------|
| `AdminPageUserDTO.java` | 管理员用户列表查询参数 |
| `EmitSeverActionDTO.java` | 向 Python 下发动作的目标 WebSocket 与动作类型 |
| `PasswordDTO.java` | 修改密码（原密码 + 新密码） |
| `RetrievePasswordDTO.java` | 找回密码（手机号、短信码、新密码） |
| `ServerActionPayloadDTO.java` | 发往 Python 的动作负载 |
| `ServerActionResponseDTO.java` | Python 动作执行结果 |
| `SysDictDataDTO.java` | 字典数据表单 |
| `SysDictTypeDTO.java` | 字典类型表单 |
| `SysParamsDTO.java` | 系统参数表单 |
| `SysUserDTO.java` | 用户管理 CRUD 字段 |

#### entity/

| 文件 | 说明 |
|------|------|
| `SysDictDataEntity.java` | 表 `sys_dict_data`，字典项 |
| `SysDictTypeEntity.java` | 表 `sys_dict_type`，字典类型 |
| `SysParamsEntity.java` | 表 `sys_params`，键值型系统参数 |
| `SysUserEntity.java` | 表 `sys_user`，登录用户 |

#### enums/

| 文件 | 说明 |
|------|------|
| `ServerActionEnum.java` | 通知 Python 的动作枚举（重启、更新配置） |
| `ServerActionResponseEnum.java` | 动作响应状态（success/fail） |
| `SuperAdminEnum.java` | 是否超级管理员枚举 |

#### redis/

| 文件 | 说明 |
|------|------|
| `SysParamsRedis.java` | 系统参数在 Redis Hash 中的增删改查封装 |

#### service/ 与 service/impl/（12 个文件）

| 文件 | 说明 |
|------|------|
| `SysDictDataService.java` / `SysDictDataServiceImpl.java` | 字典数据业务与 Redis 缓存 |
| `SysDictTypeService.java` / `SysDictTypeServiceImpl.java` | 字典类型与级联数据维护 |
| `SysParamsService.java` / `SysParamsServiceImpl.java` | 参数 CRUD、Redis 同步、SM2 密钥初始化 |
| `SysUserService.java` / `SysUserServiceImpl.java` | 用户生命周期与设备/智能体级联删除 |
| `SysUserUtilService.java` / `SysUserUtilServiceImpl.java` | 按用户 ID 解析用户名（避免循环依赖） |
| `TokenService.java` / `TokenServiceImpl.java` | 简单 token 字符串生成 |

#### utils/

| 文件 | 说明 |
|------|------|
| `WebSocketClientManager.java` | 可关闭的 WebSocket 客户端，支持同步请求/响应 |
| `WebSocketTestHandler.java` | 连接建立即完成 Future，用于连通性探测 |
| `WebSocketValidator.java` | 校验 ws/wss URL 格式并尝试短连接测试 |

#### vo/

| 文件 | 说明 |
|------|------|
| `AdminPageUserVO.java` | 管理员用户列表行展示 |
| `SysDictDataItem.java` | 下拉用的字典标签与键 |
| `SysDictDataVO.java` | 字典数据详情 |
| `SysDictTypeVO.java` | 字典类型详情 |

### 5.10 timbre — 音色管理模块（8 个文件）

| 文件 | 说明 |
|------|------|
| `controller/TimbreController.java` | TTS 模型下音色分页与增删改查 |
| `dao/TimbreDao.java` | 音色表 `ai_tts_voice` Mapper |
| `dto/TimbreDataDTO.java` | 音色表单字段 |
| `dto/TimbrePageDTO.java` | 按 TTS 模型 ID 分页查询参数 |
| `entity/TimbreEntity.java` | 表 `ai_tts_voice`，音色与 TTS 模型关联 |
| `service/TimbreService.java` | 音色业务接口 |
| `service/impl/TimbreServiceImpl.java` | 音色 CRUD、与声音克隆联动、缓存刷新 |
| `vo/TimbreDetailsVO.java` | 音色详情展示 |

### 5.11 voiceclone — 声音克隆模块（8 个文件）

| 文件 | 说明 |
|------|------|
| `controller/VoiceCloneController.java` | 用户分页与管理自己的克隆资源、训练、音频上传 |
| `controller/VoiceResourceController.java` | 超管分页与维护全站音色克隆资源 |
| `dao/VoiceCloneDao.java` | 声音克隆表 Mapper |
| `dto/VoiceCloneDTO.java` | 批量音色绑定请求 |
| `dto/VoiceCloneResponseDTO.java` | 克隆记录列表展示 |
| `entity/VoiceCloneEntity.java` | 表 `ai_voice_clone`，克隆任务与用户/模型关联 |
| `service/VoiceCloneService.java` | 克隆业务接口 |
| `service/impl/VoiceCloneServiceImpl.java` | 克隆全链路：分页、远程训练 API、用户与模型联动 |

---

## 6. resources — 资源文件（123 个文件）

### 6.1 配置文件

| 文件 | 说明 |
|------|------|
| `application.yml` | 主配置：端口 8002、context-path `/xiaozhi`、激活 dev Profile、国际化、上传限制、Knife4j、MyBatis-Plus 等 |
| `application-dev.yml` | 开发环境：Druid MySQL 数据源、Lettuce Redis 连接参数 |
| `logback-spring.xml` | Logback 日志：彩色控制台 + 滚动文件（`./logs`）+ 按大小/日期切割 |

### 6.2 mapper/ — MyBatis XML 映射（16 个文件）

| 文件 | 说明 |
|------|------|
| `mapper/agent/AgentDao.xml` | 智能体相关 SQL |
| `mapper/agent/AgentPluginMappingMapper.xml` | 插件映射 SQL |
| `mapper/agent/AgentTagDao.xml` | 标签 SQL |
| `mapper/agent/AgentTagRelationDao.xml` | 标签关联 SQL |
| `mapper/agent/AgentTemplateMapper.xml` | 模板 SQL |
| `mapper/agent/AiAgentChatHistoryDao.xml` | 聊天历史 SQL |
| `mapper/device/DeviceDao.xml` | 设备 SQL |
| `mapper/knowledge/KnowledgeBaseDao.xml` | 知识库 SQL |
| `mapper/model/ModelConfigDao.xml` | 模型配置 SQL |
| `mapper/model/ModelProviderDao.xml` | 供应器 SQL |
| `mapper/security/SysUserTokenDao.xml` | Token SQL |
| `mapper/sys/SysDictDataDao.xml` | 字典数据 SQL |
| `mapper/sys/SysDictTypeDao.xml` | 字典类型 SQL |
| `mapper/sys/SysParamsDao.xml` | 系统参数 SQL |
| `mapper/sys/SysUserDao.xml` | 用户 SQL |
| `mapper/voiceclone/VoiceCloneDao.xml` | 声音克隆 SQL |

### 6.3 db/changelog/ — 数据库迁移（88 个文件）

| 文件 | 说明 |
|------|------|
| `db.changelog-master.yaml` | Liquibase 主变更日志，按时间戳顺序引用下列 SQL 脚本 |
| `202503141335.sql` | 初始化核心系统表（sys_user、sys_user_token、sys_params 等） |
| `202503141346.sql` | 创建 AI 侧基础表（ai_model_provider、ai_model_config 等） |
| `202504082211.sql` | 初始化模型供应器种子数据 |
| `202504092335.sql` | 初始化智能体模板种子数据 |
| `202504112044.sql` | 初始化默认模型配置种子数据 |
| *...其余 82 个增量 SQL* | 表结构变更、索引优化、新增字段、种子数据补充等 |

### 6.4 i18n/ — 国际化（14 个文件）

| 文件 | 说明 |
|------|------|
| `messages.properties` | 默认语言业务提示文案 |
| `messages_zh_CN.properties` | 简体中文 |
| `messages_zh_TW.properties` | 繁体中文 |
| `messages_en_US.properties` | 美式英语 |
| `messages_de_DE.properties` | 德语 |
| `messages_pt_BR.properties` | 巴西葡萄牙语 |
| `messages_vi_VN.properties` | 越南语 |
| `validation.properties` | 默认语言校验消息 |
| `validation_zh_CN.properties` | 简体中文校验消息 |
| `validation_zh_TW.properties` | 繁体中文校验消息 |
| `validation_en_US.properties` | 英语校验消息 |
| `validation_de_DE.properties` | 德语校验消息 |
| `validation_pt_BR.properties` | 巴西葡萄牙语校验消息 |
| `validation_vi_VN.properties` | 越南语校验消息 |

### 6.5 lua/ — Redis Lua 脚本（2 个文件）

| 文件 | 说明 |
|------|------|
| `getKeyOrCreate.lua` | 若 key 不存在则 SET 并可设置过期时间，实现"获取或占位" |
| `emptyAll.lua` | 执行 `FLUSHDB` 清空当前 Redis 数据库 |

---

## 7. pom.xml — 项目依赖

### Spring Boot 与 Spring 生态

| 依赖 | 用途 |
|------|------|
| `spring-boot-starter-parent 3.4.3` | 父 POM，统一版本管理 |
| `spring-boot-starter-web` | Web 框架（内嵌 Tomcat） |
| `spring-boot-starter-aop` | AOP 面向切面编程 |
| `spring-boot-starter-websocket` | WebSocket 支持 |
| `spring-boot-starter-data-redis` | Redis 缓存 |
| `spring-boot-starter-validation` | 参数校验 |
| `spring-context-support` | Spring 上下文支持 |

### 安全与认证

| 依赖 | 用途 |
|------|------|
| `shiro-core / shiro-spring / shiro-web` | Apache Shiro 安全框架（jakarta 版） |
| `bcprov-jdk18on` | BouncyCastle 加密库（SM2 国密） |
| `easy-captcha` | 图形验证码生成 |

### 数据访问

| 依赖 | 用途 |
|------|------|
| `mysql-connector-j` | MySQL JDBC 驱动 |
| `druid-spring-boot-3-starter` | Druid 数据库连接池 |
| `mybatis-plus-boot-starter` | MyBatis-Plus ORM 增强 |
| `mybatis-spring` | MyBatis 与 Spring 集成 |
| `liquibase-core` | 数据库版本迁移 |

### API 文档

| 依赖 | 用途 |
|------|------|
| `knife4j-openapi3-jakarta-spring-boot-starter` | Knife4j API 文档 UI |
| `springdoc-openapi-starter-webmvc-ui` | SpringDoc OpenAPI |

### 工具库

| 依赖 | 用途 |
|------|------|
| `hutool-all` | Java 工具库（日期、加密、HTTP、JSON 等） |
| `jsoup` | HTML 解析与 XSS 清洗 |
| `commons-lang3` | 字符串/对象工具 |
| `guava` | Google 通用工具库 |
| `lombok` | 编译期代码生成（@Data、@Slf4j 等） |
| `jackson-datatype-jsr310` | Java 8 日期 JSON 序列化 |

### 外部服务

| 依赖 | 用途 |
|------|------|
| `dysmsapi20170525` | 阿里云短信 SDK |

---

## 8. 分层架构说明

每个业务模块的代码组织遵循统一的分层约定：

```
请求进入
  │
  ▼
controller/        ← 接收 HTTP 请求，参数校验，调用 Service
  │                   注解：@RestController、@RequestMapping、@GetMapping/@PostMapping
  │                   返回：Result<T>（统一响应格式）
  ▼
service/           ← 业务逻辑，事务控制
  │                   注解：@Service
  │                   继承：BaseService / CrudService
  ▼
dao/               ← 数据访问（SQL 操作）
  │                   注解：@Mapper
  │                   继承：BaseMapper<Entity>
  │                   配套：mapper/*.xml（复杂 SQL）
  ▼
entity/            ← 数据库表映射
                      注解：@TableName、@TableId、@TableField
                      继承：BaseEntity（id、creator、createDate）

辅助层：
  dto/             ← 请求参数对象（Data Transfer Object）
  vo/              ← 响应视图对象（View Object）
```

**继承关系图：**
```
MyBatis-Plus BaseMapper<T>
        ▲
        │
    BaseDao<T>  (common/dao/)
        ▲
        │
    DeviceDao / AgentDao / ...  (各模块 dao/)


    BaseService<T>  (common/service/)
        ▲
        │
    BaseServiceImpl<M, T>  (common/service/impl/，@Autowired baseDao)
        ▲
        │
    DeviceServiceImpl / AgentServiceImpl / ...  (各模块 service/impl/)


    BaseEntity  (common/entity/，id + creator + createDate)
        ▲
        │
    DeviceEntity / AgentEntity / ...  (各模块 entity/)
```
