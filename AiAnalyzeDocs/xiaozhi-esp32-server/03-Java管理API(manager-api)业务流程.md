# Java 管理 API (manager-api) 内部业务流程

## 1. 模块概览

`manager-api` 是基于 Spring Boot 3 的管理后台 REST API 服务，端口 8002，context-path `/xiaozhi`，为智控台前端和 Python 核心服务提供数据管理与配置下发能力。

```
manager-api/src/main/java/xiaozhi/
├── AdminApplication.java           # Spring Boot 启动入口
├── common/                         # 公共基础设施
│   ├── config/                     # Swagger, RestTemplate, MyBatis-Plus, Async
│   ├── redis/                      # RedisConfig, RedisUtils, RedisKeys
│   ├── exception/                  # 全局异常处理
│   ├── xss/                        # XSS防护
│   ├── service/                    # BaseService, CrudService
│   ├── dao/                        # BaseDao
│   ├── entity/                     # BaseEntity
│   └── utils/                      # 工具类(Result, AES, SM2, JSON等)
└── modules/
    ├── security/                   # 认证授权(Shiro)
    ├── device/                     # 设备管理 + OTA
    ├── agent/                      # 智能体管理
    ├── model/                      # 模型管理
    ├── knowledge/                  # 知识库管理
    ├── config/                     # 配置下发
    ├── sys/                        # 系统管理(用户/参数/字典)
    ├── timbre/                     # 音色管理
    ├── voiceclone/                 # 语音克隆
    └── sms/                        # 短信服务
```

## 2. 公共基础设施 (common/)

### 2.1 请求处理链路

```
HTTP请求进入
 ├── XssFilter (可选, renren.xss.enabled=true)
 │   └── XssHttpServletRequestWrapper: HTML清洗
 ├── DelegatingFilterProxy("shiroFilter")
 │   ├── anon: OTA/下载/文档/登录等
 │   ├── server: /config/**, /agent/chat-history/report 等(服务间)
 │   └── oauth2: /** 其余(用户Bearer)
 ├── Controller → @RequiresPermissions 权限检查
 ├── Service → 业务逻辑
 ├── Dao → MyBatis-Plus + Mapper XML → MySQL
 └── 异常 → RenExceptionHandler (@RestControllerAdvice)
     ├── RenException → {code, msg}
     ├── UnauthorizedException → 401
     ├── DuplicateKeyException → "数据库中已存在"
     └── Exception → 500 内部错误
```

### 2.2 核心配置

| 配置类 | 功能 |
|--------|------|
| SwaggerConfig | Knife4j API文档分组 |
| RestTemplateConfig | HTTP客户端(30s超时) |
| MybatisPlusConfig | 数据权限拦截器 + 分页 + 乐观锁 + 防全表操作 |
| AsyncConfig | @EnableAsync 线程池(核心2, 最大4, 队列1000) |
| RedisConfig | RedisTemplate JSON序列化 |
| WebMvcConfig | CORS + Jackson(Long→String, 时间格式, Locale) |

### 2.3 Redis Key 命名规范

```
RedisKeys 常量定义:
├── 验证码: captcha:{uuid}
├── 设备: device:{macAddress}
├── OTA下载: ota:download:{uuid}
├── 模型: model:name:{id}, model:config:{id}
├── 智能体: agent:devices:{agentId}, agent:info:{id}
├── 音频播放: audio:play:{uuid}
├── 短信: sms:count:{phone}, sms:interval:{phone}
├── 服务配置: server:config
├── 临时注册: tmp:register:{mac}
└── 更多...
```

## 3. 安全模块 (modules/security/)

### 3.1 架构

```
                 ┌─────────────────────┐
                 │    FilterConfig     │
                 │  (注册ShiroFilter)  │
                 └──────────┬──────────┘
                            │
                 ┌──────────▼──────────┐
                 │    ShiroConfig      │
                 │  (过滤器链定义)     │
                 └──────────┬──────────┘
                ┌───────────┼───────────┐
                ▼           ▼           ▼
         ┌────────────┐ ┌────────┐ ┌──────────────┐
         │Oauth2Filter│ │  anon  │ │ServerSecret  │
         │(用户Bearer)│ │        │ │Filter        │
         └──────┬─────┘ └────────┘ └──────┬───────┘
                ▼                          ▼
         ┌────────────┐           ┌──────────────┐
         │Oauth2Realm │           │ 比对系统参数  │
         │(Token→User)│           │ server.secret│
         └────────────┘           └──────────────┘
```

### 3.2 Shiro 过滤器链

| 路径模式 | 过滤器 | 说明 |
|----------|--------|------|
| `/ota/**` | anon | 设备OTA(匿名) |
| `/otaMag/download/**` | anon | 固件下载 |
| `/user/login`, `/user/register` | anon | 登录注册 |
| `/user/captcha`, `/user/pub-config` | anon | 验证码/公共配置 |
| `/agent/play/**`, `/voiceClone/play/**` | anon | 音频播放(UUID) |
| `/agent/chat-history/download/**` | anon | 聊天记录下载 |
| `/config/**` | server | 服务间配置(Bearer=server.secret) |
| `/agent/chat-history/report` | server | 聊天记录上报 |
| `/agent/chat-summary/**`, `/agent/chat-title/**` | server | 摘要/标题生成 |
| `/**` | oauth2 | 其余需用户Token |

### 3.3 登录流程

```
POST /user/login
 ├── 图形验证码校验(uuid + captcha)
 ├── SM2解密密码(SM2公钥来自系统参数)
 ├── SysUserService.getByUsername()
 ├── PasswordUtils.matches(rawPassword, encodedPassword)
 ├── SysUserTokenServiceImpl.createToken(userId)
 │   ├── 查现有token → 未过期则更新过期时间
 │   └── 无效/过期 → 生成新UUID token，写 sys_user_token
 └── 返回 TokenDTO {token, expire, clientHash}
```

### 3.4 API 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/user/captcha` | 图形验证码 |
| POST | `/user/smsVerification` | 短信验证码发送 |
| POST | `/user/login` | 登录 |
| POST | `/user/register` | 注册 |
| GET | `/user/info` | 当前用户信息 |
| PUT | `/user/change-password` | 修改密码 |
| PUT | `/user/retrieve-password` | 找回密码 |
| GET | `/user/pub-config` | 公共配置(注册开关/SM2公钥/备案/菜单) |

## 4. 设备模块 (modules/device/)

### 4.1 职责

设备生命周期管理：OTA注册、激活码绑定、设备列表、在线状态查询、MCP工具转发。

### 4.2 OTA 上报流程（设备侧入口）

```
POST /xiaozhi/ota/  (Shiro anon)
 └── DeviceServiceImpl.checkDeviceActive(mac, ...)
     ├── 按 MAC 查 ai_device
     ├── 已绑定设备:
     │   ├── 检查固件更新 → OtaService.getLatestOta(type)
     │   │   └── 版本比较 → 有新版本: 生成下载UUID → Redis
     │   ├── 组装 WebSocket 连接信息:
     │   │   ├── server.websocket 随机选一条URL
     │   │   └── 若 server.auth_enabled → HMAC token
     │   └── 组装 MQTT 信息(若配置)
     └── 未绑定设备:
         └── buildActivation()
             ├── 生成6位激活码
             ├── Redis存储 激活码→MAC 映射
             └── 返回 activation 字段
```

### 4.3 设备绑定流程

```
POST /device/bind/{agentId}/{deviceCode}  (需登录)
 ├── Redis查询激活码 → MAC地址
 ├── 校验设备未被其他用户绑定
 ├── deviceDao.insert(DeviceEntity)
 │   ├── userId = 当前登录用户
 │   ├── agentId = 指定智能体
 │   └── macAddress = MAC
 ├── 清理Redis激活数据
 └── 返回设备信息
```

### 4.4 API 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| POST | `/device/bind/{agentId}/{deviceCode}` | 激活码绑定 |
| POST | `/device/register` | 生成激活码 |
| GET | `/device/bind/{agentId}` | 已绑定设备列表 |
| POST | `/device/unbind` | 解绑设备 |
| PUT | `/device/update/{id}` | 更新设备信息 |
| POST | `/device/manual-add` | 手动添加设备 |
| POST | `/device/tools/list/{deviceId}` | MCP工具列表(转发) |
| POST | `/device/tools/call/{deviceId}` | MCP工具调用(转发) |
| POST | `/ota/` | 设备OTA上报(anon) |
| POST | `/ota/activate` | 激活探测 |
| GET | `/otaMag` | 固件列表(管理) |
| POST | `/otaMag` | 保存固件 |
| POST | `/otaMag/upload` | 上传固件 |
| GET | `/otaMag/download/{uuid}` | 下载固件(anon) |

### 4.5 数据模型

**DeviceEntity (ai_device)**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | String(UUID) | 主键 |
| userId | Long | 所属用户 |
| macAddress | String | MAC地址 |
| agentId | String | 绑定的智能体 |
| board | String | 硬件型号 |
| alias | String | 设备别名 |
| appVersion | String | 固件版本 |
| autoUpdate | Integer | 自动更新开关 |
| lastConnectedAt | Date | 最后在线时间 |

**OtaEntity (ai_ota)**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| firmwareName | String | 固件名 |
| type | String | 固件类型 |
| version | String | 版本号 |
| size | Long | 文件大小 |
| firmwarePath | String | 文件路径 |

## 5. 智能体模块 (modules/agent/)

### 5.1 职责

智能体的完整生命周期管理：CRUD、模板、标签、会话历史、音频上报、声纹、MCP接入点。

### 5.2 智能体创建流程

```
POST /agent  (需登录)
 └── AgentServiceImpl.save()
     ├── 设置 userId = 当前用户
     ├── 生成 agentCode
     ├── 设置默认模型ID(从默认模板获取)
     └── agentDao.insert()
```

### 5.3 聊天记录上报流程 (服务间调用)

```
POST /agent/chat-history/report  (Bearer = server.secret)
 └── AgentChatHistoryBizService.report(dto)
     ├── 构建 AgentChatHistoryEntity
     │   ├── macAddress, sessionId
     │   ├── chatType (1=用户, 2=智能体, 3=工具)
     │   ├── content
     │   └── audioBase64 (有音频时)
     ├── 音频存储处理(若有)
     └── chatHistoryDao.insert()
```

### 5.4 MCP 接入点流程

```
GET /agent/mcp/address/{agentId}
 └── AgentMcpAccessPointServiceImpl.getAgentMcpAddress()
     ├── 读取 server.mcp_endpoint 参数
     ├── AES/Hash 生成加密 token
     └── 拼接 WebSocket URL (含 token)

GET /agent/mcp/tools/{agentId}
 └── AgentMcpAccessPointServiceImpl.getAgentMcpToolsList()
     ├── 构建 WebSocket 连接到 MCP 端点
     ├── JSON-RPC: initialize 请求
     ├── JSON-RPC: tools/list 请求
     └── 返回工具列表
```

### 5.5 API 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/agent/list` | 智能体列表 |
| GET | `/agent/{id}` | 智能体详情 |
| POST | `/agent` | 创建智能体 |
| PUT | `/agent/{id}` | 更新智能体 |
| DELETE | `/agent/{id}` | 删除智能体 |
| GET | `/agent/{id}/sessions` | 会话列表 |
| GET | `/agent/{id}/chat-history/{sessionId}` | 聊天详情 |
| POST | `/agent/chat-history/report` | 聊天上报(server) |
| GET/POST/PUT/DELETE | `/agent/template/*` | 模板管理 |
| POST/PUT/DELETE/GET | `/agent/voice-print/*` | 声纹管理 |
| GET | `/agent/mcp/address/{agentId}` | MCP地址 |
| GET | `/agent/mcp/tools/{agentId}` | MCP工具列表 |
| GET/POST/DELETE | `/agent/tag/*` | 标签管理 |

### 5.6 数据模型

**AgentEntity (ai_agent)**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | String | 主键 |
| userId | Long | 所属用户 |
| agentCode | String | 智能体编码 |
| agentName | String | 名称 |
| systemPrompt | String | 系统提示词 |
| asrModelId / vadModelId / llmModelId / ... | Long | 各类模型ID |
| ttsVoiceId | Long | TTS音色ID |
| ttsLanguage / ttsVolume / ttsRate / ttsPitch | String | TTS参数 |
| chatHistoryConf | String | 聊天记录配置 |
| summaryMemory | String | 记忆摘要 |
| langCode / language | String | 语言 |

## 6. 模型模块 (modules/model/)

### 6.1 职责

管理 AI 模型配置（LLM/TTS/ASR/VAD/Memory/Intent等），包含模型供应商元数据（表单字段定义）。

### 6.2 模型保存流程

```
POST /models/{modelType}/{provideCode}
 └── ModelConfigServiceImpl.save()
     ├── 校验 provider 存在性
     ├── 敏感字段处理(API Key 掩码恢复)
     ├── modelConfigDao.insert()
     └── 清理 Redis 缓存 (model:name:{id}, model:config:{id})
```

### 6.3 模型删除保护

```
DELETE /models/{id}
 └── ModelConfigServiceImpl.delete()
     ├── 禁止删除默认模型
     ├── 检查 AgentDao 是否引用该模型
     ├── 检查 Intent 是否引用
     └── 通过 → modelConfigDao.deleteById()
```

### 6.4 API 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/models/names` | 模型名称列表 |
| GET | `/models/llm/names` | LLM名称列表 |
| GET | `/models/{modelType}/provideTypes` | 供应商类型列表 |
| GET | `/models/list` | 模型分页列表 |
| POST | `/models/{modelType}/{provideCode}` | 新增模型 |
| PUT | `/models/{modelType}/{provideCode}/{id}` | 编辑模型 |
| DELETE | `/models/{id}` | 删除模型 |
| GET | `/models/{id}` | 模型详情(敏感字段掩码) |
| PUT | `/models/enable/{id}/{status}` | 启用/禁用 |
| PUT | `/models/default/{id}` | 设为默认 |
| GET | `/models/{modelId}/voices` | TTS音色列表 |
| GET/POST/PUT/DELETE | `/models/provider/*` | 供应商管理 |

### 6.5 数据模型

**ModelConfigEntity (ai_model_config)**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| modelType | String | 模型类型(llm/tts/asr/vad/memory/intent) |
| modelCode | String | 模型代码 |
| modelName | String | 模型名称 |
| isDefault | Boolean | 是否默认 |
| isEnabled | Boolean | 是否启用 |
| configJson | JSONObject | 配置JSON(含API Key等) |
| docLink | String | 文档链接 |

## 7. 知识库模块 (modules/knowledge/)

### 7.1 职责

本地影子表 + 远程 RAG 系统双写：当前适配 RAGFlow。

### 7.2 创建知识库流程

```
POST /datasets
 └── KnowledgeBaseServiceImpl.create()
     ├── 校验名称唯一性
     ├── 获取 RAG 模型 configJson
     ├── KnowledgeBaseAdapterFactory.getAdapter("ragflow", config)
     ├── adapter.createDataset()           # 调RAGFlow API
     │   └── POST {ragflow_url}/api/v1/datasets
     ├── 成功 → knowledgeBaseDao.insert()  # 本地影子表
     └── 失败 → best-effort adapter.deleteDataset() 回滚
```

### 7.3 API 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/datasets` | 知识库分页列表 |
| GET | `/datasets/{dataset_id}` | 知识库详情 |
| POST | `/datasets` | 创建知识库 |
| PUT | `/datasets/{dataset_id}` | 更新 |
| DELETE | `/datasets/{dataset_id}` | 删除 |
| POST | `/datasets/{dataset_id}/documents` | 上传文档 |
| DELETE | `/datasets/{dataset_id}/documents/{document_id}` | 删除文档 |
| GET | `/datasets/{dataset_id}/documents/{document_id}/chunks` | 分块列表 |
| POST | `/datasets/{dataset_id}/retrieval-test` | 检索测试 |
| GET | `/datasets/rag-models` | RAG模型列表 |

## 8. 配置下发模块 (modules/config/)

### 8.1 职责

作为 **配置中枢**，聚合所有业务模块数据，为 Python 核心服务提供运行时配置。

### 8.2 全站配置 (server-base)

```
POST /config/server-base  (Bearer = server.secret)
 └── ConfigServiceImpl.getConfig(true)
     ├── 检查 Redis 缓存 (server:config)
     ├── SysParamsService.list() → 参数树
     │   └── param_code 按 '.' 拆分为嵌套Map
     │       例: "server.websocket" → {server: {websocket: "ws://..."}}
     ├── 获取默认 AgentTemplate 的模型ID
     └── buildModuleConfig(config)
         └── 按模型类型展开: {modelType: {modelId: configJson}}
```

### 8.3 设备私有配置 (agent-models)

```
POST /config/agent-models  (Bearer = server.secret)
 Body: {macAddress, clientId, selectedModule}
 └── ConfigServiceImpl.getAgentModels()
     ├── MAC → DeviceDao → DeviceEntity
     ├── Device → AgentEntity (智能体)
     ├── 组装各模块配置:
     │   ├── VAD: modelConfig.configJson
     │   ├── ASR: modelConfig.configJson
     │   ├── LLM: modelConfig.configJson + systemPrompt
     │   ├── TTS: modelConfig.configJson + 音色信息
     │   │   ├── Timbre → private_voice/ref_audio/ref_text
     │   │   └── VoiceClone → 同上
     │   ├── Memory: modelConfig.configJson + summaryMemory
     │   ├── Intent: modelConfig.configJson + 附加LLM配置
     │   ├── 插件: AgentPluginMapping → 描述覆盖
     │   ├── MCP: mcp_endpoint 参数
     │   ├── context_providers: Agent上下文提供者
     │   └── 声纹: AgentVoicePrint 列表
     └── 返回聚合配置JSON
```

## 9. 系统管理模块 (modules/sys/)

### 9.1 API 端点

**管理员管理 (`/admin`)**

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/admin/users` | 用户分页列表(超管) |
| PUT | `/admin/users/{id}` | 更新用户 |
| DELETE | `/admin/users/{id}` | 删除用户 |
| PUT | `/admin/users/changeStatus/{status}` | 批量改状态 |
| GET | `/admin/device/all` | 全量设备视图 |

**系统参数 (`admin/params`)**

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/admin/params/page` | 参数分页 |
| POST | `/admin/params` | 新增参数 |
| PUT | `/admin/params` | 更新参数 |
| POST | `/admin/params/delete` | 删除参数 |

**字典管理 (`/admin/dict/*`)**

标准CRUD：类型和数据的分页、详情、增删改。

**服务器管理 (`/admin/server`)**

```
POST /admin/server/emit-action
 └── 向指定 xiaozhi-server 实例发WebSocket指令
     ├── 建立WebSocket连接(带JWT认证)
     ├── 发送 ServerActionPayloadDTO (含 server.secret)
     └── 等待响应
```

## 10. 音色模块 (modules/timbre/)

### 10.1 职责

维护 TTS 音色库，按模型维度管理音色元数据。

### 10.2 API 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/ttsVoice` | 音色分页列表 |
| POST | `/ttsVoice` | 新增音色 |
| PUT | `/ttsVoice/{id}` | 更新音色 |
| POST | `/ttsVoice/delete` | 批量删除 |

### 10.3 数据模型

**TimbreEntity (ai_tts_voice)**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | Long | 主键 |
| name | String | 音色名称 |
| ttsModelId | Long | 关联TTS模型 |
| ttsVoice | String | 音色标识 |
| languages | String | 支持语言 |
| referenceAudio | String | 参考音频 |
| referenceText | String | 参考文本 |
| voiceDemo | String | 示例音频 |

## 11. 语音克隆模块 (modules/voiceclone/)

### 11.1 语音克隆流程

```
POST /voiceClone/upload     # 上传参考音频
POST /voiceClone/cloneAudio # 触发克隆
 └── VoiceCloneServiceImpl
     ├── 调用TTS供应商API进行克隆训练
     ├── 更新 trainStatus / voiceId
     └── 训练失败 → trainError

GET /voiceClone/play/{uuid}  (anon)
 ├── Redis查询 UUID → 音频ID
 └── 返回音频流
```

### 11.2 API 端点

| 方法 | 路径 | 功能 |
|------|------|------|
| GET | `/voiceClone` | 克隆列表 |
| POST | `/voiceClone/upload` | 上传音频 |
| POST | `/voiceClone/cloneAudio` | 触发克隆 |
| GET | `/voiceClone/play/{uuid}` | 播放(anon) |
| GET/POST/DELETE | `/voiceResource/*` | 语音资源管理 |

## 12. 短信模块 (modules/sms/)

### 12.1 调用链路

```
POST /user/smsVerification (LoginController, anon)
 └── CaptchaServiceImpl.sendSMSValidateCode()
     ├── 图形验证码校验
     ├── 发送间隔检查 (60秒)
     ├── 每日次数检查 (系统参数 server.sms_max_send_count)
     ├── 生成验证码 → Redis存储
     ├── ALiYunSmsService.sendVerificationCodeSms()
     │   └── 阿里云 dysmsapi SDK
     │       ├── AK/SK 来自系统参数
     │       └── endpoint: dysmsapi.aliyuncs.com
     └── 失败 → 回滚当日计数
```

短信模块无独立 Controller，由安全模块的验证码服务调用。
