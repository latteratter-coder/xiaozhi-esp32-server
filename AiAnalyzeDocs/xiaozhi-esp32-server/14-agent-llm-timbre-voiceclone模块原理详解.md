# agent / llm / timbre / voiceclone 模块原理详解

> 基于源码的代码级实现分析，覆盖核心数据流、关键方法、表结构和模块间调用关系。

[TOC]

---

## 零、名词通俗解释

在开始之前，先用最通俗的话解释这几个模块各自是干什么的：

### 智能体（agent）是什么？有什么用？

**一句话**：智能体 = 你在后台配置出来的一个"AI 角色"。

想象你有一个 ESP32 硬件设备（类似智能音箱），它本身不知道"我该用什么语音识别引擎、什么大模型、什么声音来回答用户"。**智能体就是告诉设备这些答案的配置中心**。

具体来说，创建一个智能体时，你在后台选择/配置：
- **用哪个 ASR 模型**：设备收到用户语音后，用什么引擎把语音转成文字
- **用哪个 LLM 大模型**：文字转好后，发给哪个 AI 来生成回答（DeepSeek、通义千问等）
- **用哪个 TTS 音色**：AI 的文字回答，用什么声音念出来
- **系统提示词**：AI 的"人设"，比如"你是一个温柔的助手"
- **记忆模式**：AI 是否记住之前的对话
- **插件**：AI 能不能查天气、放音乐、看新闻
- **MCP 工具**：AI 能不能调用外部工具（搜索、计算等）

然后把这个智能体**绑定到设备**上（通过 `ai_device.agent_id`），设备就知道该怎么工作了。

所以 agent 模块不仅管智能体 CRUD，还管**聊天记录上报**（Python 服务每轮对话都会报给 Java）、**聊天总结**（用 LLM 总结对话记忆）、**声纹识别**（识别说话的人是谁）、**MCP 工具接入**等。

### timbre（音色）是用来干什么的？

**一句话**：timbre 管理的是"AI 用什么声音说话"。

TTS（Text-to-Speech，文字转语音）服务通常提供多种声音——男声、女声、温柔、活泼等。每种声音在厂商侧有一个编码（如 `zh_female_shuangkuaisisi`），timbre 模块就是管理这些声音的"目录"。

它解决的问题是：
- 一个 TTS 模型（比如火山引擎 TTS）下面有几十种音色可选
- 用户在后台创建智能体时，需要一个下拉列表来选音色
- timbre 就是提供这个列表，并且**还会把用户自己克隆训练成功的声音也合并进来**

### MCP 是用来干什么的？

**一句话**：MCP 让智能体能够"调用外部工具"。

MCP（Model Context Protocol，模型上下文协议）是一个开放协议，定义了 AI 模型如何调用外部工具。类似于 ChatGPT 的"插件"概念，但用的是标准化的 WebSocket + JSON-RPC 协议。

在这个项目中，MCP 的作用是：
- 管理员可以给智能体配置一个 MCP Server 地址
- 智能体通过 WebSocket 连接到这个 MCP Server
- 获取该 Server 提供的工具列表（比如"搜索"、"翻译"、"计算"等）
- 当用户提问时，AI 大模型可以判断是否需要调用这些工具来辅助回答

举个例子：用户问"今天北京的 PM2.5 是多少"，如果智能体配了一个提供"空气质量查询"工具的 MCP Server，AI 就会先调工具获取实时数据，再组织语言回答。

---

## 一、模块关系总览

> **架构图**：请用 VS Code Draw.io 插件打开 → [14-01-module-overview.drawio](14-01-module-overview.drawio)

四个模块的关系和职责：

| 模块 | 颜色 | 核心职责 | 关键 Controller |
|------|:----:|---------|----------------|
| **agent** | 蓝色 | 智能体 CRUD、聊天上报、总结、声纹、MCP、标签 | `/agent/*`, `/agent/chat-history/*`, `/agent/voice-print/*`, `/agent/mcp/*` |
| **llm** | 黄色 | 统一封装 OpenAI 风格的 LLM 调用（总结、标题） | 无 Controller，纯 Service |
| **timbre** | 绿色 | TTS 音色管理，合并普通音色与克隆音色 | `/ttsVoice/*` |
| **voiceclone** | 红色 | 声音克隆（上传训练音频、调火山引擎训练） | `/voiceClone/*`, `/voiceResource/*` |

**模块间调用关系**：

| 调用方 | → 被调用方 | 方法 | 说明 |
|--------|-----------|------|------|
| AgentChatSummaryService | → LLMService | `generateSummary` / `generateTitle` | 总结聊天记忆、生成会话标题 |
| OpenAIStyleLLMServiceImpl | → ModelConfigService | `getModelByIdFromCache` | 从 Redis 缓存读取模型配置 |
| AgentService | → TimbreService | `getTimbreNameById` | 查询音色名称用于展示 |
| TimbreService | → VoiceCloneService | `getTrainSuccess` | 获取训练成功的克隆音色，合并到音色列表 |
| VoiceCloneService | → ModelConfigService | `getModelConfig` | 读取火山引擎 TTS 的 appid/access_token |

---

## 二、agent 模块

### 2.1 智能体创建流程

> **流程图**：请用 VS Code Draw.io 插件打开 → [14-02-agent-create-flow.drawio](14-02-agent-create-flow.drawio)

**流程概要**：`前端 POST /agent` → `AgentController` → `AgentServiceImpl.createAgent(dto)`

**关键实现细节**：

- 智能体 ID 若为空自动生成 UUID，`agentCode` 生成 `AGT_` 前缀
- 从 `AgentTemplateService` 获取默认模板，填充 ASR/VAD/LLM/VLLM/TTS、音色、记忆模型、系统提示词等
- `chatHistoryConf` 根据记忆模型决定：无记忆(`Memory_nomem`)→0(不记录)，否则→2(文本+音频)
- `slmModelId`(小模型)若空，自动取已启用 LLM 列表中的默认或第一个
- 创建完成后**自动批量插入 3 个系统插件映射**：音乐、天气、新闻

### 2.2 智能体更新流程

更新(`PUT /agent/{id}`)比创建复杂得多，涉及**插件差量同步**和**记忆策略副作用**：

**步骤**：

1. 接收 `AgentUpdateDTO`
2. 加载现有实体 `AgentInfoVO`
3. 逐字段 patch 非空值
4. 若 `dto.functions != null`，执行插件差量同步：
   - 新增 → `saveBatch`
   - 更新 → `updateBatchById`
   - 删除 → `removeBatchByIds`
5. 根据 `memModelId` 类型处理副作用：

| memModelId 类型 | 副作用 |
|:---------------:|--------|
| 无记忆 | 删除全部聊天记录+音频，清空 summaryMemory |
| 仅上报 | 清空 summaryMemory |
| 其他 | 保留不动 |

6. 保存上下文源 `contextProviders`
7. 校验 LLM 意图参数
8. `updateById` 写库

### 2.3 聊天记录上报（核心数据流）

> **流程图**：请用 VS Code Draw.io 插件打开 → [14-03-chat-report-flow.drawio](14-03-chat-report-flow.drawio)

这是系统中**写入量最大**的流程——Python Server 每轮对话都会调用。

**完整数据流**：

```
ESP32 设备 --[WebSocket 音频流]--> Python Server (xiaozhi-server)
    Python: Opus 解码 → PCM → WAV → Base64 编码
    Python: 根据 chatHistoryConf 决定是否携带音频
Python Server --[POST /agent/chat-history/report]--> Java API (manager-api)
    Java: AgentChatHistoryBizService.report()
    Java: SELECT ai_device WHERE mac=? LEFT JOIN ai_agent
    Java: 根据 chatHistoryConf 分支写入
```

**chatHistoryConf 三种模式**：

| 值 | 模式 | 写入内容 |
|:--:|------|---------|
| 2 | 文本+音频 | INSERT `ai_agent_chat_audio` (LONGBLOB) + INSERT `ai_agent_chat_history` (含 audioId) |
| 1 | 仅文本 | INSERT `ai_agent_chat_history` (audioId=null) |
| 0 | 不记录 | 跳过写入 |

**上报 DTO 结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| macAddress | String | 设备MAC，用于关联智能体 |
| sessionId | String | 会话ID |
| chatType | Byte | 1=用户消息, 2=助手消息 |
| content | String | 文本内容 |
| audioBase64 | String | WAV音频的Base64(可选) |
| reportTime | Long | 秒级时间戳 |

**音频存储**：`ai_agent_chat_audio` 表，`audio` 字段为 **LONGBLOB**，直接存 WAV 二进制。前端播放通过 Redis 临时 UUID 换取流式下载。

### 2.4 聊天总结生成

**触发路径**：`POST /agent/chat-summary/{sessionId}/save` → `new Thread()` 异步执行（fire-and-forget）

**步骤**：

1. 查聊天记录（只取用户消息 `chatType=1`）
2. 过滤无意义内容（设备控制、天气等）和过短句子
3. 读智能体当前 `summaryMemory`（历史记忆）
4. 选择模型：优先 `slmModelId` → 默认 LLM → `llmModelId`
5. 调用 `LLMService.generateSummaryWithHistory(conversation, historyMemory, null, modelId)`
6. 将总结结果写回 `UPDATE ai_agent SET summary_memory=?`

**记忆模型决策**：

| memModelId | 行为 |
|:----------:|------|
| `Memory_nomem` | 跳过总结 |
| `Memory_mem_report_only` | 跳过总结 |
| `Memory_mem0ai` / `Memory_powermem` | 跳过（外部记忆服务管理） |
| `Memory_local_short` 等 | **执行总结**，写回 summaryMemory |

### 2.5 声纹管理

声纹管理采用**本地 DB + 外部声纹服务**双存储架构。

**流程**：

1. 前端 `POST /agent/voice-print` 提交 `(audioId, agentId)`
2. 从 `AgentChatAudioService` 获取 WAV 字节
3. 先调用声纹服务**识别接口**查重（`POST /voiceprint/identify`，相似度 > 0.5 则已存在）
4. 若不存在，调用**注册接口**（`POST /voiceprint/register`，`speaker_id=新UUID`）
5. 写入 `ai_agent_voice_print` 表

**表结构**：`ai_agent_voice_print` 存 `id`(UUID)、`agentId`、`audioId`(引用聊天音频)、`sourceName`、`introduce`

**外部服务**：地址从系统参数 `server.voice_print` 读取，URI query 中的 `key=` 作为 Bearer Token

### 2.6 MCP 接入点

MCP (Model Context Protocol) 允许智能体调用外部工具。通过 WebSocket JSON-RPC 协议交互。

**工具列表获取流程**：

1. 前端 `GET /agent/mcp/tools/{agentId}`
2. 服务端构造认证 token：`MD5(agentId)` → `AES 加密` → `URL 编码` → 拼接 `wss://.../call/?token=...`
3. 建立 WebSocket 连接
4. 发送 JSON-RPC `initialize` 请求（id=1），等待响应
5. 发送 `notifications/initialized` 通知
6. 发送 `tools/list` 请求（id=2），获取工具列表
7. 提取每个工具的 name，排序后返回给前端

### 2.7 标签系统

标签与智能体是**多对多**关系，通过 `ai_agent_tag_relation` 中间表维护：

| 表 | 字段 | 说明 |
|----|------|------|
| `ai_agent_tag` | id, tag_name, deleted | 标签定义（支持软删除） |
| `ai_agent_tag_relation` | id, agent_id, tag_id, sort | 多对多关联（带排序） |

**`saveAgentTags` 逻辑**：先删除该智能体所有关联 → 新标签名批量创建 → 合并 tagId → 批量插入关联(带 sort 递增)

### 2.8 级联删除

删除智能体时清理的数据（按顺序）：

| 步骤 | 操作 | 说明 |
|:----:|------|------|
| 1 | `deviceService.deleteByAgentId` | 删除关联设备 |
| 2 | `agentChatHistoryService.deleteByAgentId(id, true, true)` | 删音频BLOB + 聊天记录 |
| 3 | `agentPluginMappingService.deleteByAgentId` | 删插件映射 |
| 4 | `agentContextProviderService.deleteByAgentId` | 删上下文源 |
| 5 | `agentService.deleteById` | 删智能体本体 |

> **已知遗漏**：`ai_agent_voice_print`(声纹)、`ai_agent_tag_relation`(标签关联)、`ai_agent_chat_title`(会话标题) 未在删除链中清理，可能产生孤儿数据。

---

## 三、llm 模块

### 3.1 架构：统一 OpenAI 风格封装

> **架构图**：请用 VS Code Draw.io 插件打开 → [14-05-llm-timbre-system.drawio](14-05-llm-timbre-system.drawio)（左半部分）

LLM 模块只有一个接口 `LLMService` 和一个实现 `OpenAIStyleLLMServiceImpl`，没有 Controller——它是纯内部服务，被 agent 模块调用。

核心思路：**不管后端用的是 DeepSeek、通义千问还是 Ollama，只要它兼容 OpenAI 的 `/chat/completions` 接口格式，就能用**。

### 3.2 接口方法

| 方法 | 用途 | temperature | max_tokens |
|------|------|:-----------:|:----------:|
| `generateSummary(conversation)` | 默认模板+默认模型 | 配置值/0.7 | 配置值/2000 |
| `generateSummary(conversation, promptTemplate, modelId)` | 自定义模板+指定模型 | 同上 | 同上 |
| `generateSummaryWithHistory(conv, history, template, modelId)` | **带历史记忆的总结**(主路径) | **0.2** | 2000 |
| `generateTitle(conversation, modelId)` | 生成会话标题 | **0.3** | **50** |
| `isAvailable()` / `isAvailable(modelId)` | 检查可用性(仅检查配置非空) | - | - |

### 3.3 Prompt 模板

**总结 Prompt**（DEFAULT_SUMMARY_PROMPT）：

```
你是一个经验丰富的记忆总结者，擅长将对话内容进行总结摘要，遵循以下规则：
1、总结用户的重要信息，以便在未来的对话中提供更个性化的服务
2、不要重复总结，不要遗忘之前记忆，除非原来的记忆超过了1800字，否则不要遗忘、不要压缩用户的历史记忆
3、用户操控的设备音量、播放音乐、天气、退出、不想对话等和用户本身无关的内容，这些信息不需要加入到总结中
4、聊天内容中的今天的日期时间、今天的天气情况与用户事件无关的数据，这些信息不需要加入到总结中
5、不要把设备操控的成果结果和失败结果加入到总结中，也不要把用户的一些废话加入到总结中
6、不要为了总结而总结，如果用户的聊天没有意义，请返回原来的历史记录也是可以的
7、只需要返回总结摘要，严格控制在1800字内
8、不要包含代码、xml，不需要解释、注释和说明
9、如果提供了历史记忆，请将新对话内容与历史记忆进行智能合并

历史记忆：
{history_memory}

新对话内容：
{conversation}
```

**标题 Prompt**（DEFAULT_TITLE_PROMPT）：

```
请根据以下对话内容，生成一个简洁的会话标题（约15字以内），只返回标题，不要包含任何解释或标点符号：
{conversation}
```

### 3.4 HTTP 请求构造

请求流程：

1. `getModelByIdFromCache(modelId)` 从 Redis 获取 `ModelConfigEntity`（含 `base_url`, `api_key`, `model_name`, `temperature`, `max_tokens`）
2. 规范化 URL：`base_url` + `/chat/completions`，`model_name` 默认 `gpt-3.5-turbo`
3. 构造请求体：

```json
{
  "model": "xxx",
  "messages": [{"role": "user", "content": "拼好的prompt"}],
  "temperature": 0.2,
  "max_tokens": 2000
}
```

4. 发送 `POST` 请求，Header 带 `Authorization: Bearer {api_key}`
5. 解析响应 `choices[0].message.content`

### 3.5 模型选择优先级

**LLMService 自身**：指定了 `modelId` → 直接用；未指定 → 取已启用 LLM 列表中 `isDefault=1` 的，没有则取第一个。

**AgentChatSummaryService 的模型选择**更复杂：优先 `slmModelId`(小模型) → 默认 LLM → `llmModelId`（大模型）。这样设计是因为总结任务不需要太强的模型，用小模型更省成本。

### 3.6 错误处理

- **无重试机制**：HTTP 调用只执行一次
- 失败时返回固定中文错误串，调用方通过字符串匹配判断成功
- `isAvailable` 仅检查 `base_url`/`api_key` 非空，**不发探测请求**

---

## 四、timbre 模块

### 4.1 数据模型

> **架构图**：请用 VS Code Draw.io 插件打开 → [14-05-llm-timbre-system.drawio](14-05-llm-timbre-system.drawio)（右半部分）

**`ai_tts_voice` 表（普通音色）**：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | String PK | 音色ID |
| tts_model_id | String FK | 关联的 TTS 模型（→ `ai_model_config`） |
| name | String | 音色名称（如"温柔女声"） |
| tts_voice | String | 厂商侧音色编码（如 `zh_female_shuangkuaisisi`） |
| languages | String | 支持的语言（逗号分隔） |
| voice_demo | String | 试听音频 URL |
| reference_audio | String | 参考音频 URL（用于声音复刻型 TTS） |
| reference_text | String | 参考文本 |
| sort | Long | 排序权重 |

### 4.2 音色列表组装（含克隆音色合并）

`getVoiceNames` 是音色系统的核心方法，它将**普通音色**和**克隆音色**合并为统一列表：

**步骤**：

1. 查 `ai_tts_voice` WHERE `tts_model_id=?` → 转为 `VoiceDTO[]`（`isClone=false`）
2. 若当前用户已登录：查 `ai_voice_clone` WHERE `model_id=?` AND `user_id=?` AND `train_status=2`（训练成功的克隆音色）
3. 克隆音色转为 `VoiceDTO[]`（`isClone=true`，name 前加"克隆-"前缀）
4. 将克隆音色**插入列表最前面**
5. 返回合并后的 `List<VoiceDTO>`

这样前端在音色选择下拉框中，**用户自己的克隆声音永远排在最前面**，且前端无需区分是普通音色还是克隆音色。

### 4.3 音色在智能体配置中的使用

智能体的 `ttsVoiceId` 可以指向**音色表**或**克隆表**，配置下发时（`ConfigService.getAgentModels`）的查找链：

1. 先查 `ai_tts_voice`（`timbreService.get(ttsVoiceId)`）
2. 找到 → 取 `ttsVoice` 作为 voice 参数，取 `referenceAudio`/`referenceText`
3. 找不到 → 查 `ai_voice_clone`（`voiceCloneService.selectById(ttsVoiceId)`）
4. 找到克隆 → 取 `voiceId`（火山 speaker_id）作为 voice，language 设为"普通话"
5. 特殊处理：若 TTS 类型是火山双流且 voice 以 `S_` 开头 → 额外设置 `resource_id = 'seed-icl-1.0'`

---

## 五、voiceclone 模块

### 5.1 数据模型

**`ai_voice_clone` 表**：

| 字段 | 类型 | 说明 |
|------|------|------|
| id | String PK | 克隆记录ID |
| name | String | 名称（自动生成 MMddHHmm 格式） |
| model_id | String FK | 关联的 TTS 模型（→ `ai_model_config`） |
| voice_id | String | 火山 speaker_id（训练成功后填入） |
| languages | String | 语言 |
| user_id | Long FK | 所属用户 |
| voice | LONGBLOB | **训练音频（WAV 二进制，最大 10MB）** |
| train_status | Int | 0=待训练, 1=训练中, 2=成功, 3=失败 |
| train_error | String | 失败原因 |

### 5.2 完整声音克隆流程

> **流程图**：请用 VS Code Draw.io 插件打开 → [14-04-voice-clone-flow.drawio](14-04-voice-clone-flow.drawio)

整个克隆分 **4 个阶段**：

**阶段 1：管理员开通资源**

- `POST /voiceResource`，携带 `{modelId, userId, voiceIds:["S_xxx","S_yyy"]}`
- 校验：必须是火山双流 TTS 模型、voiceId 须含 `S_` 前缀、同 model+voiceId 不重复
- 批量 `INSERT ai_voice_clone`（`trainStatus=0`, `voice=null`）

**阶段 2：用户上传训练音频**

- `POST /voiceClone/upload`，携带 `(id, MultipartFile)`，支持 WAV/MP3，最大 10MB
- `file.getBytes()` 后直接写入 `UPDATE ai_voice_clone SET voice=?`（LONGBLOB）

**阶段 3：用户触发克隆训练**

- `POST /voiceClone/cloneAudio`，携带 `{id}`
- 从 DB 读取 voice 字节和 ModelConfig
- 调用火山引擎 API（详见 5.3）
- 成功 → `UPDATE trainStatus=2, voiceId=返回的speaker_id`
- 失败 → `UPDATE trainStatus=3, trainError=错误信息`

**阶段 4：用户选用克隆音色**

- 在智能体设置中选择克隆音色 → `ttsVoiceId = ai_voice_clone.id`
- 配置下发时，ConfigService 会动态查找（见 4.3 节）

### 5.3 火山引擎克隆 API 详解

| 参数 | 值 | 说明 |
|------|-----|------|
| URL | `https://openspeech.bytedance.com/api/v1/mega_tts/audio/upload` | 火山语音克隆端点 |
| Authorization | `Bearer;{access_token}` | 注意分号在 Bearer 后（非标准格式） |
| Resource-Id | `seed-icl-1.0` | 固定资源标识 |
| appid | 从 config_json 读取 | 火山应用ID |
| speaker_id | 开通时预置的 `S_xxx` | 火山说话人ID |
| audio_bytes | Base64(WAV) | 整段训练音频 |
| source | 2 | 来源标识 |
| language | 0 | 语言(0=中文) |
| model_type | 1 | 模型类型 |

### 5.4 管理端 vs 用户端分工

| 角色 | Controller | 功能 |
|------|-----------|------|
| **管理员** | VoiceResourceController `/voiceResource` | 查看全部资源、为用户开通克隆槽位(批量创建voiceId)、查看TTS平台 |
| **普通用户** | VoiceCloneController `/voiceClone` | 查看自己的资源、上传训练音频、改名、触发训练、试听 |

### 5.5 音频存储方式

> **当前实现**：训练音频直接存 MySQL `LONGBLOB` 字段，最大 10MB/条。
> **百万级瓶颈**：大量 BLOB 数据会导致 MySQL 表膨胀、备份缓慢、IO 压力大。建议迁移到 MinIO/OSS。

---

## 六、四模块数据流全景

> **全景图**：请用 VS Code Draw.io 插件打开 → [14-06-data-flow-overview.drawio](14-06-data-flow-overview.drawio)

**端到端数据流路径**：

```
ESP32 设备
  │ WebSocket 音频流
  ▼
Python Server (xiaozhi-server, Port 8000)
  │ 语音处理: Opus → PCM → WAV → Base64
  │ HTTP POST 上报
  ▼
Java API (manager-api)
  ├── agent 模块
  │   ├── 聊天上报 → MySQL (ai_agent_chat_history / ai_agent_chat_audio)
  │   ├── 总结生成 → 调 llm 模块 → 外部 LLM API → 写回 MySQL
  │   ├── 声纹注册 → 外部声纹服务
  │   └── MCP 工具 → WebSocket JSON-RPC → MCP Server
  ├── llm 模块 → POST /chat/completions → 任意 OpenAI 兼容 API
  ├── timbre 模块 → 读 MySQL (ai_tts_voice) + 合并克隆音色
  └── voiceclone 模块 → 火山引擎 mega_tts API
         │
         ▼
存储层: MySQL (6 张核心表) + Redis (模型缓存/音色缓存/临时UUID)
```

---

## 七、关键设计总结

| 设计点 | 实现方式 | 评价 |
|--------|---------|------|
| **聊天上报鉴权** | Shiro `server` 过滤器(服务间调用) | 合理，非用户Token |
| **音频存储** | MySQL LONGBLOB | 简单但不利于大规模 |
| **LLM调用** | 统一 OpenAI Chat Completions 格式 | 兼容性好，支持任何兼容API |
| **LLM重试** | 无 | 生产环境建议补充 |
| **总结异步** | `new Thread()` fire-and-forget | 简单但无监控，建议改 `@Async` |
| **声纹** | 本地DB + 外部服务双写 | 解耦合理 |
| **MCP** | WebSocket JSON-RPC 2.0 | 符合 MCP 协议规范 |
| **音色合并** | 普通音色 + 克隆音色统一 VoiceDTO | 前端无感知 |
| **克隆训练** | 火山 mega_tts API | 仅支持火山双流平台 |
| **级联删除** | 手动顺序删除 | 缺声纹/标签清理 |

---

## 八、drawio 图文件索引

| 文件 | 内容 | 用途 |
|------|------|------|
| [14-01-module-overview.drawio](14-01-module-overview.drawio) | 四模块关系总览 | 展示 agent/llm/timbre/voiceclone 之间的调用关系和外部依赖 |
| [14-02-agent-create-flow.drawio](14-02-agent-create-flow.drawio) | 智能体创建流程 | 从 DTO 到 DB 写入的完整泳道图 |
| [14-03-chat-report-flow.drawio](14-03-chat-report-flow.drawio) | 聊天记录上报流程 | ESP32→Python→Java→MySQL/Redis 的核心数据流 |
| [14-04-voice-clone-flow.drawio](14-04-voice-clone-flow.drawio) | 声音克隆完整流程 | 4 阶段：开通→上传→训练→使用 |
| [14-05-llm-timbre-system.drawio](14-05-llm-timbre-system.drawio) | LLM 调用 + 音色系统 | LLM 统一封装架构 + 音色合并逻辑 + 表结构 |
| [14-06-data-flow-overview.drawio](14-06-data-flow-overview.drawio) | 四模块数据流全景 | 端到端完整数据流（设备→Python→Java→外部服务→存储） |

> **查看方式**：安装 VS Code 插件 **Draw.io Integration**（`hediet.vscode-drawio`），直接双击 `.drawio` 文件即可在编辑器内打开查看和编辑。

---

> **参考源码路径**：
> - `modules/agent/service/impl/AgentServiceImpl.java`
> - `modules/agent/service/biz/impl/AgentChatHistoryBizServiceImpl.java`
> - `modules/agent/service/impl/AgentChatSummaryServiceImpl.java`
> - `modules/llm/service/impl/OpenAIStyleLLMServiceImpl.java`
> - `modules/timbre/service/impl/TimbreServiceImpl.java`
> - `modules/voiceclone/service/impl/VoiceCloneServiceImpl.java`
