# OAuth2用户Token认证完整流程

## 三种客户端认证方式对比

manager-api 有三种不同的客户端，每种使用不同的认证机制：

| | Web用户（管理后台） | ESP32设备 | xiaozhi-server（Python后端） |
|---|---|---|---|
| **Token获取方式** | 用户名+密码登录后生成 | OTA时manager-api下发 | 配置文件中写死`server.secret` |
| **Token本质** | MD5(UUID)随机字符串 | `HMAC签名.timestamp` | 明文密钥字符串 |
| **存储位置** | MySQL `sys_user_token`表 | 不存储（设备内存） | 配置文件 |
| **验证方式** | 查数据库比对 | 重新计算HMAC签名比对 | 字符串直接比对 |
| **验证组件** | `Oauth2Realm` | `AuthManager`（Python） | `ServerSecretFilter` |
| **有效期** | 12小时 | 默认30天（可配置） | 无期限 |
| **用途** | 管理设备、改配置、看数据 | 连WebSocket实时对话 | 调manager-api的`/config`接口 |

---

## 一、Web用户Token认证流程

### 1.1 Token生成（登录时）

用户通过管理后台登录后，manager-api生成token：

```java
// LoginController.login()
@PostMapping("/login")
public Result<TokenDTO> login(@RequestBody LoginDTO login) {
    // 1. 验证用户名密码
    SysUserDTO userDTO = sysUserService.getByUsername(login.getUsername());
    if (!PasswordUtils.matches(login.getPassword(), userDTO.getPassword())) {
        throw new RenException(ErrorCode.ACCOUNT_PASSWORD_ERROR);
    }
    
    // 2. 生成token并返回
    return sysUserTokenService.createToken(userDTO.getId());
}

// SysUserTokenServiceImpl.createToken()
public Result<TokenDTO> createToken(Long userId) {
    // 查看该用户是否已有token
    SysUserTokenEntity tokenEntity = baseDao.getByUserId(userId);
    
    if (tokenEntity == null) {
        // 从未登录过：生成新token
        token = TokenGenerator.generateValue();  // = MD5(UUID.randomUUID())
        insert(tokenEntity);
    } else {
        // 已登录过
        if (tokenEntity.getExpireDate().getTime() < now) {
            // 过期了：重新生成
            token = TokenGenerator.generateValue();
        } else {
            // 未过期：继续用旧的
            token = tokenEntity.getToken();
        }
        updateById(tokenEntity);  // 更新过期时间（往后延12小时）
    }
    return token;
}
```

**Token格式**：`MD5(UUID)`，例如 `a1b2c3d4e5f67890...`（32位十六进制）

**存储**：MySQL `sys_user_token`表
```
| user_id | token | expire_date | update_date |
| 1       | a1b2..| 2026-04-26 20:00 | 2026-04-26 08:00 |
```

### 1.2 Token验证（每次请求）

每次API请求都会经过Shiro的Filter链：

```
请求进入
    │
    ▼
Oauth2Filter.isAccessAllowed()
    │
    ├── OPTIONS请求 → true（放行）
    │
    └── 其他请求 → false（进入onAccessDenied）
            │
            ▼
        onAccessDenied()
            │
            ├── Token为空 → 返回401 → 拒绝
            │
            └── Token不为空 → executeLogin()
                    │
                    ▼
                createToken()
                    │
                    │  注意：这里是"包装"，不是"生成"
                    │  String token = getRequestToken(request);  // 从Header提取
                    │  return new Oauth2Token(token);  // 包成对象
                    │
                    ▼
                SecurityManager交给Realm
                    │
                    ▼
                Oauth2Realm.doGetAuthenticationInfo()
                    │
                    │  String accessToken = token.getPrincipal();
                    │  SysUserTokenEntity entity = shiroService.getByToken(accessToken);
                    │
                    ├── 查不到 → throw IncorrectCredentialsException（无效）
                    ├── 过期了 → throw IncorrectCredentialsException（无效）
                    └── 查到了 → 认证成功，返回用户信息
                    │
                    ▼
                认证成功 → 放行到Controller
```

**核心验证代码**：
```java
// Oauth2Realm.doGetAuthenticationInfo()
protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) {
    String accessToken = (String) token.getPrincipal();  // 用户传来的token
    
    // 查数据库
    SysUserTokenEntity tokenEntity = shiroService.getByToken(accessToken);
    
    if (tokenEntity == null) {
        throw new IncorrectCredentialsException();  // token不存在
    }
    
    if (tokenEntity.getExpireDate().getTime() < System.currentTimeMillis()) {
        throw new IncorrectCredentialsException();  // token过期
    }
    
    // 查用户信息
    SysUserEntity userEntity = shiroService.getUser(tokenEntity.getUserId());
    
    // 检查账号状态
    if (userEntity.getStatus() == 0) {
        throw new LockedAccountException();  // 账号被锁定
    }
    
    return new SimpleAuthenticationInfo(userDetail, accessToken, getName());
}
```

### 1.3 关键理解

**`createToken()`不是生成新token！**

```java
protected AuthenticationToken createToken(ServletRequest request, ...) {
    // 从请求Header取出用户已有的token
    String token = getRequestToken((HttpServletRequest) request);
    
    // "包装"成Shiro需要的对象类型
    return new Oauth2Token(token);  // 只是包一层，没生成新东西
}
```

`Oauth2Token`只是一个容器：
```java
public class Oauth2Token implements AuthenticationToken {
    private String token;
    
    public Oauth2Token(String token) {
        this.token = token;  // 直接存用户传来的字符串
    }
}
```

---

## 二、ESP32设备Token认证流程

### 2.1 Token生成（OTA时）

ESP32开机时请求OTA接口，manager-api下发HMAC签名token：

```java
// DeviceServiceImpl.checkDeviceActive()
if ("true".equalsIgnoreCase(authEnabled)) {
    String token = generateWebSocketToken(clientId, macAddress);
    websocket.setToken(token);
}

// DeviceServiceImpl.generateWebSocketToken()
public String generateWebSocketToken(String clientId, String username) {
    String secretKey = sysParamsService.getValue("server.secret", false);
    long timestamp = System.currentTimeMillis() / 1000;
    
    // 签名内容：clientId|username|timestamp
    String content = String.format("%s|%s|%d", clientId, username, timestamp);
    
    // HMAC-SHA256签名
    Mac hmac = Mac.getInstance("HmacSHA256");
    hmac.init(new SecretKeySpec(secretKey.getBytes(), "HmacSHA256"));
    byte[] signature = hmac.doFinal(content.getBytes());
    
    // Base64 URL-safe编码
    String signatureBase64 = Base64.getUrlEncoder().withoutPadding().encodeToString(signature);
    
    // 返回格式：signature.timestamp
    return String.format("%s.%d", signatureBase64, timestamp);
}
```

**Token格式**：`签名Base64.timestamp`
例如：`AbCdEfGhIjKlMnOp.1714123456`

### 2.2 Token验证（WebSocket连接时）

ESP32连接WebSocket时，xiaozhi-server验证token：

```python
# AuthManager.verify_token()
def verify_token(self, token: str, client_id: str, username: str) -> bool:
    try:
        # 1. 拆开token
        sig_part, ts_str = token.split(".")
        ts = int(ts_str)
        
        # 2. 检查过期
        if int(time.time()) - ts > self.expire_seconds:
            return False
        
        # 3. 用相同的secret_key重新计算签名
        expected_sig = self._sign(f"{client_id}|{username}|{ts}")
        
        # 4. 比对签名
        if not hmac.compare_digest(sig_part, expected_sig):
            return False
        
        return True
    except:
        return False
```

**验证逻辑**：不查数据库，纯数学计算——用相同的密钥和输入重新算签名，比对是否一致。

---

## 三、xiaozhi-server与manager-api通信认证

### 3.1 认证方式

xiaozhi-server调用manager-api的`/config/**`接口时，使用明文`server.secret`：

```python
# manage_api_client.py
headers = {
    "Authorization": "Bearer " + cls._secret,  # 配置文件里的secret
}
```

manager-api验证：

```java
// ServerSecretFilter.onAccessDenied()
protected boolean onAccessDenied(...) {
    String token = getRequestToken(request);  // 从Header取
    String serverSecret = sysParamsService.getValue("server.secret");  // 从数据库取
    
    // 直接字符串比对
    if (!serverSecret.equals(token)) {
        sendUnauthorizedResponse("无效的服务器密钥");
        return false;
    }
    return true;
}
```

### 3.2 安全考虑

- 这是服务器间通信，默认部署在内网
- 不面向用户和设备
- 安全依赖网络隔离（防火墙/内网），而非算法

---

## 四、热更新时的特殊通信

热更新是manager-api唯一主动连接xiaozhi-server的场景：

```
管理员点"更新配置"
      │
      ▼
manager-api伪造设备身份（UUID）
      │
      ▼
生成HMAC token（跟ESP32用同一套算法）
      │
      ▼
作为WebSocket客户端连接xiaozhi-server
      │
      ▼
发送指令：{action: "update_config", secret: "..."}
      │
      ▼
xiaozhi-server收到 → 验证secret → 热重载配置
```

---

## 五、Shiro框架核心概念

### Realm是什么？

Realm是Shiro框架中的"安全域"——一套独立的认证与授权规则。

比喻：公司大楼有多个入口，每个入口有不同的保安系统（人脸识别、刷卡、密码），每个保安系统就是一个Realm。

### 项目中的Realm

| Realm | 管辖范围 | 认证逻辑 |
|---|---|---|
| `Oauth2Realm` | Web用户 | 查数据库验证token |
| `ServerSecretFilter`配套逻辑 | xiaozhi-server | 比对明文secret |

### Shiro验证流程

```
请求 → Filter拦截 → 创建Token对象 → SecurityManager → Realm认证
         │                              │
         │                              │ Realm决定：
         │                              │ - 这个token我认不认？
         │                              │ - 去哪查用户信息？
         │                              │ - 用户有什么权限？
         │                              │
         └── Realm.supports()判断token类型
```

---

## 六、总结

三种认证方式各有适用场景：

| 方式 | 优点 | 适用场景 |
|---|---|---|
| 查数据库 | 可吊销、可管理、有状态 | 用户登录系统 |
| HMAC签名 | 不存储、防伪造、无状态 | 设备身份认证 |
| 明文secret | 简单直接、服务器间 | 后端内部通信 |