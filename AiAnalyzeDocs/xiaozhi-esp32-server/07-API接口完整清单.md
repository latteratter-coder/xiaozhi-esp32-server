# API 接口完整清单

> 所有 manager-api 的接口均以 `http://<host>:8002/xiaozhi` 为前缀。

## 1. 安全与用户 (/user)

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/user/captcha` | anon | 获取图形验证码 |
| POST | `/user/smsVerification` | anon | 发送短信验证码 |
| POST | `/user/login` | anon | 用户登录 |
| POST | `/user/register` | anon | 用户注册 |
| GET | `/user/info` | oauth2 | 获取当前用户信息 |
| PUT | `/user/change-password` | oauth2 | 修改密码 |
| PUT | `/user/retrieve-password` | anon | 找回密码(短信验证) |
| GET | `/user/pub-config` | anon | 获取公共配置 |

## 2. 设备管理 (/device)

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| POST | `/device/bind/{agentId}/{deviceCode}` | oauth2 | 激活码绑定设备 |
| POST | `/device/register` | oauth2 | 生成设备注册码 |
| GET | `/device/bind/{agentId}` | oauth2 | 已绑定设备列表 |
| POST | `/device/bind/{agentId}` | oauth2 | 设备在线数据查询 |
| POST | `/device/unbind` | oauth2 | 解绑设备 |
| PUT | `/device/update/{id}` | oauth2 | 更新设备信息 |
| POST | `/device/manual-add` | oauth2 | 手动添加设备 |
| POST | `/device/tools/list/{deviceId}` | oauth2 | MCP工具列表 |
| POST | `/device/tools/call/{deviceId}` | oauth2 | MCP工具调用 |

## 3. OTA设备端 (/ota)

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| POST | `/ota/` | anon | 设备OTA上报 |
| POST | `/ota/activate` | anon | 激活状态探测 |
| GET | `/ota/` | anon | 健康检查 |

## 4. OTA管理端 (/otaMag)

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/otaMag` | oauth2(超管) | 固件分页列表 |
| GET | `/otaMag/{id}` | oauth2(超管) | 固件详情 |
| POST | `/otaMag` | oauth2(超管) | 保存固件 |
| PUT | `/otaMag/{id}` | oauth2(超管) | 更新固件 |
| DELETE | `/otaMag/{id}` | oauth2(超管) | 删除固件 |
| GET | `/otaMag/getDownloadUrl/{id}` | oauth2(超管) | 获取下载URL |
| GET | `/otaMag/download/{uuid}` | anon | 下载固件 |
| POST | `/otaMag/upload` | oauth2(超管) | 上传固件 |
| POST | `/otaMag/uploadAssetsBin` | oauth2(超管) | 上传资源bin |

## 5. 智能体 (/agent)

### 5.1 智能体主接口

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/agent/list` | oauth2 | 智能体列表 |
| GET | `/agent/all` | oauth2 | 全部智能体 |
| GET | `/agent/{id}` | oauth2 | 智能体详情 |
| POST | `/agent` | oauth2 | 创建智能体 |
| PUT | `/agent/{id}` | oauth2 | 更新智能体 |
| DELETE | `/agent/{id}` | oauth2 | 删除智能体 |
| PUT | `/agent/saveMemory/{macAddress}` | oauth2 | 保存记忆 |
| GET | `/agent/{id}/sessions` | oauth2 | 会话列表 |
| GET | `/agent/{id}/chat-history/{sessionId}` | oauth2 | 聊天详情 |
| GET | `/agent/{id}/chat-history/user` | oauth2 | 用户语音消息 |
| GET | `/agent/{id}/chat-history/audio` | oauth2 | 音频历史 |
| POST | `/agent/audio/{audioId}` | oauth2 | 音频操作 |
| GET | `/agent/play/{uuid}` | anon | 播放音频 |

### 5.2 聊天记录(服务间)

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| POST | `/agent/chat-history/report` | server | 聊天记录上报 |
| POST | `/agent/chat-history/getDownloadUrl/{agentId}/{sessionId}` | oauth2 | 获取下载URL |
| GET | `/agent/chat-history/download/{uuid}/current` | anon | 下载当前会话 |
| GET | `/agent/chat-history/download/{uuid}/previous` | anon | 下载历史会话 |

### 5.3 摘要与标题(服务间)

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| POST | `/agent/chat-summary/{sessionId}/save` | server | 生成会话摘要 |
| POST | `/agent/chat-title/{sessionId}/generate` | server | 生成会话标题 |

### 5.4 智能体模板

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/agent/template/page` | oauth2 | 模板分页 |
| GET | `/agent/template/{id}` | oauth2 | 模板详情 |
| POST | `/agent/template` | oauth2 | 新增模板 |
| PUT | `/agent/template` | oauth2 | 更新模板 |
| DELETE | `/agent/template/{id}` | oauth2 | 删除模板 |
| POST | `/agent/template/batch-remove` | oauth2 | 批量删除 |

### 5.5 声纹

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| POST | `/agent/voice-print` | oauth2 | 新增声纹 |
| PUT | `/agent/voice-print` | oauth2 | 更新声纹 |
| DELETE | `/agent/voice-print/{id}` | oauth2 | 删除声纹 |
| GET | `/agent/voice-print/list/{id}` | oauth2 | 声纹列表 |

### 5.6 MCP接入点

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/agent/mcp/address/{agentId}` | oauth2 | 获取MCP WebSocket URL |
| GET | `/agent/mcp/tools/{agentId}` | oauth2 | 获取MCP工具列表 |

### 5.7 标签

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/agent/tag/list` | oauth2 | 标签列表 |
| POST | `/agent/tag` | oauth2 | 创建标签 |
| PUT | `/agent/tag/{id}` | oauth2 | 更新标签 |
| DELETE | `/agent/tag/{id}` | oauth2 | 删除标签 |
| GET | `/agent/{id}/tags` | oauth2 | 智能体标签 |

## 6. 模型管理 (/models)

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/models/names` | oauth2 | 模型名称列表 |
| GET | `/models/llm/names` | oauth2 | LLM名称列表 |
| GET | `/models/{modelType}/provideTypes` | oauth2 | 供应商类型 |
| GET | `/models/list` | oauth2 | 模型分页列表 |
| POST | `/models/{modelType}/{provideCode}` | oauth2(超管) | 新增模型 |
| PUT | `/models/{modelType}/{provideCode}/{id}` | oauth2(超管) | 编辑模型 |
| DELETE | `/models/{id}` | oauth2(超管) | 删除模型 |
| GET | `/models/{id}` | oauth2 | 模型详情 |
| PUT | `/models/enable/{id}/{status}` | oauth2(超管) | 启用/禁用 |
| PUT | `/models/default/{id}` | oauth2(超管) | 设为默认 |
| GET | `/models/{modelId}/voices` | oauth2 | TTS音色列表 |

## 7. 模型供应商 (/models/provider)

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/models/provider` | oauth2(超管) | 供应商列表 |
| POST | `/models/provider` | oauth2(超管) | 新增供应商 |
| PUT | `/models/provider` | oauth2(超管) | 更新供应商 |
| POST | `/models/provider/delete` | oauth2(超管) | 删除供应商 |
| GET | `/models/provider/plugin/names` | oauth2 | 插件名列表 |

## 8. 知识库 (/datasets)

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/datasets` | oauth2 | 知识库分页列表 |
| GET | `/datasets/{dataset_id}` | oauth2 | 知识库详情 |
| POST | `/datasets` | oauth2 | 创建知识库 |
| PUT | `/datasets/{dataset_id}` | oauth2 | 更新知识库 |
| DELETE | `/datasets/{dataset_id}` | oauth2 | 删除知识库 |
| DELETE | `/datasets/batch` | oauth2 | 批量删除 |
| GET | `/datasets/rag-models` | oauth2 | RAG模型列表 |
| GET | `/datasets/{dataset_id}/documents` | oauth2 | 文档列表 |
| GET | `/datasets/{dataset_id}/documents/status/{status}` | oauth2 | 按状态筛选文档 |
| POST | `/datasets/{dataset_id}/documents` | oauth2 | 上传文档 |
| DELETE | `/datasets/{dataset_id}/documents` | oauth2 | 批量删除文档 |
| DELETE | `/datasets/{dataset_id}/documents/{document_id}` | oauth2 | 删除单个文档 |
| POST | `/datasets/{dataset_id}/chunks` | oauth2 | 分块操作 |
| GET | `/datasets/{dataset_id}/documents/{document_id}/chunks` | oauth2 | 分块列表 |
| POST | `/datasets/{dataset_id}/retrieval-test` | oauth2 | 检索测试 |

## 9. 配置下发 (/config) — 服务间

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| POST | `/config/server-base` | server | 全站配置树 |
| POST | `/config/agent-models` | server | 设备私有模型配置 |

## 10. 系统管理

### 10.1 管理员 (/admin)

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/admin/users` | oauth2(超管) | 用户分页 |
| PUT | `/admin/users/{id}` | oauth2(超管) | 更新用户 |
| DELETE | `/admin/users/{id}` | oauth2(超管) | 删除用户 |
| PUT | `/admin/users/changeStatus/{status}` | oauth2(超管) | 批量改状态 |
| GET | `/admin/device/all` | oauth2(超管) | 全量设备 |

### 10.2 系统参数 (admin/params)

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/admin/params/page` | oauth2(超管) | 参数分页 |
| GET | `/admin/params/{id}` | oauth2(超管) | 参数详情 |
| POST | `/admin/params` | oauth2(超管) | 新增参数 |
| PUT | `/admin/params` | oauth2(超管) | 更新参数 |
| POST | `/admin/params/delete` | oauth2(超管) | 删除参数 |

### 10.3 字典类型 (/admin/dict/type)

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/admin/dict/type/page` | oauth2(超管) | 分页 |
| GET | `/admin/dict/type/{id}` | oauth2(超管) | 详情 |
| POST | `/admin/dict/type` | oauth2(超管) | 新增 |
| PUT | `/admin/dict/type` | oauth2(超管) | 更新 |
| POST | `/admin/dict/type/delete` | oauth2(超管) | 删除 |

### 10.4 字典数据 (/admin/dict/data)

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/admin/dict/data/page` | oauth2 | 分页 |
| GET | `/admin/dict/data/{id}` | oauth2 | 详情 |
| POST | `/admin/dict/data` | oauth2(超管) | 新增 |
| PUT | `/admin/dict/data` | oauth2(超管) | 更新 |
| POST | `/admin/dict/data/delete` | oauth2(超管) | 删除 |
| GET | `/admin/dict/data/type/{dictType}` | oauth2 | 按类型查询 |

### 10.5 服务器管理 (/admin/server)

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/admin/server/server-list` | oauth2(超管) | 实例列表 |
| POST | `/admin/server/emit-action` | oauth2(超管) | 发送管理指令 |

## 11. 音色 (/ttsVoice)

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/ttsVoice` | oauth2 | 音色分页列表 |
| POST | `/ttsVoice` | oauth2 | 新增音色 |
| PUT | `/ttsVoice/{id}` | oauth2 | 更新音色 |
| POST | `/ttsVoice/delete` | oauth2 | 批量删除 |

## 12. 语音克隆 (/voiceClone)

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/voiceClone` | oauth2 | 克隆列表 |
| POST | `/voiceClone/upload` | oauth2 | 上传音频 |
| POST | `/voiceClone/updateName` | oauth2 | 改名 |
| POST | `/voiceClone/audio/{id}` | oauth2 | 音频操作 |
| GET | `/voiceClone/play/{uuid}` | anon | 播放 |
| POST | `/voiceClone/cloneAudio` | oauth2 | 触发克隆 |

## 13. 语音资源 (/voiceResource)

| 方法 | 路径 | 鉴权 | 功能 |
|------|------|------|------|
| GET | `/voiceResource` | oauth2 | 资源列表 |
| GET | `/voiceResource/{id}` | oauth2 | 资源详情 |
| POST | `/voiceResource` | oauth2 | 新增资源 |
| DELETE | `/voiceResource/{id}` | oauth2 | 删除资源 |
| GET | `/voiceResource/user/{userId}` | oauth2 | 按用户查 |
| GET | `/voiceResource/ttsPlatforms` | oauth2 | TTS平台列表 |

## 14. Python 核心服务 HTTP 接口

| 方法 | 路径 | 端口 | 功能 |
|------|------|------|------|
| GET/POST | `/xiaozhi/ota/` | 8003 | OTA(仅非API模式) |
| GET | `/xiaozhi/ota/download/{filename}` | 8003 | 固件下载(仅非API模式) |
| GET/POST | `/mcp/vision/explain` | 8003 | 视觉识别 |

## 15. Python 核心服务 WebSocket 消息类型

### 15.1 客户端 → 服务端 (文本JSON)

| type | 功能 | 关键字段 |
|------|------|---------|
| `hello` | 握手与能力声明 | audio_params, features |
| `abort` | 中止当前操作 | - |
| `listen` | 语音控制 | mode(start/stop/detect), text |
| `iot` | IoT描述符/状态 | descriptors / status |
| `mcp` | MCP协议消息 | JSON-RPC payload |
| `server` | 管理端控制 | action(update_config/restart), content.secret |
| `ping` | 心跳 | - |

### 15.2 服务端 → 客户端 (文本JSON)

| type | 功能 |
|------|------|
| `hello` | 欢迎消息(含session_id) |
| `stt` | ASR识别文本 |
| `tts` | TTS控制(start/stop/sentence_start/sentence_end) |
| `llm` | LLM生成文本(emotion) |
| `pong` | 心跳回复 |

### 15.3 二进制帧

- **客户端→服务端**：Opus编码音频（MQTT网关带16字节头）
- **服务端→客户端**：Opus编码TTS音频（MQTT网关带16字节头）
