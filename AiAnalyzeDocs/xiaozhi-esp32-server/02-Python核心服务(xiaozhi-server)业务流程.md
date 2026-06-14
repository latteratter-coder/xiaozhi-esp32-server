# Python 核心服务 (xiaozhi-server) 内部业务流程

## 1. 模块概览

`xiaozhi-server` 是设备对话的核心服务，负责与ESP32硬件通信、AI推理管线编排、插件执行、MCP协议等。

```
xiaozhi-server/
├── app.py                    # 入口：环境检查→配置加载→启动WS+HTTP
├── config/
│   ├── config_loader.py      # 配置加载与合并
│   ├── settings.py           # 启动校验
│   ├── logger.py             # Loguru日志初始化
│   └── manage_api_client.py  # 智控台HTTP客户端
├── core/
│   ├── websocket_server.py   # WebSocket服务
│   ├── http_server.py        # HTTP服务(OTA/Vision)
│   ├── connection.py         # 单连接状态机(核心)
│   ├── auth.py               # HMAC认证
│   ├── handle/               # 消息处理器集合
│   ├── api/                  # HTTP API Handler
│   ├── providers/            # AI能力提供者
│   └── utils/                # 工具类
└── plugins_func/             # 插件系统
```

## 2. 启动流程 (app.py)

```
main()
 ├── check_ffmpeg_installed()          # 检查ffmpeg依赖
 ├── load_config()                     # 加载配置(可能拉取智控台)
 ├── 确定 auth_key                      # server.auth_key → manager-api.secret → 随机UUID
 ├── monitor_stdin()                   # 异步消费stdin
 ├── GCManager(300).start()            # 定时GC(5分钟周期)
 ├── WebSocketServer(config).start()   # 启动WebSocket服务(端口8000)
 ├── SimpleHttpServer(config).start()  # 启动HTTP服务(端口8003)
 ├── 日志打印各服务地址
 ├── validate_mcp_endpoint()           # MCP端点校验
 └── wait_for_exit()                   # 等待SIGINT/SIGTERM
     └── finally: 清理GC、取消所有任务
```

## 3. 配置加载流程 (config/)

### 3.1 config_loader.py - 配置加载核心

```
load_config()
 ├── 检查缓存 cache_manager(CacheType.CONFIG)
 ├── read_config("config.yaml")         # 读取默认配置
 ├── read_config("data/.config.yaml")   # 读取用户覆盖配置
 ├── merge_configs(default, custom)     # 递归合并，用户优先
 ├── 若 manager-api.url 存在:
 │   └── get_config_from_api_async()
 │       ├── init_service(config)           # 初始化 ManageApiClient 单例
 │       ├── get_server_config()        # POST /config/server-base
 │       ├── 标记 read_config_from_api = true
 │       └── 合并：保留本地 server.ip/port/http_port/vision_explain/auth_key
 ├── ensure_directories()               # 确保日志/数据目录
 └── 写入缓存
```

### 3.2 manage_api_client.py - 智控台HTTP客户端

**认证**：所有请求头带 `Authorization: Bearer {secret}`

**重试策略**：最多6次重试，间隔10秒，仅对网络错误和 408/429/5xx 重试

**业务错误码**：
- `10041` → DeviceNotFoundException（设备未找到）
- `10042` → DeviceBindException（需绑定设备）

**提供的API调用**：

| 方法 | 请求 | 用途 |
|------|------|------|
| `get_server_config()` | POST /config/server-base | 启动时拉取全站配置 |
| `get_agent_models(mac, client_id, module)` | POST /config/agent-models | 连接时拉取设备私有配置 |
| `report(data)` | POST /agent/chat-history/report | 上报聊天记录 |
| `generate_and_save_chat_title(session_id)` | POST /agent/chat-title/{id}/generate | 生成会话标题 |
| `generate_and_save_chat_summary(session_id)` | POST /agent/chat-summary/{id}/save | 生成会话摘要 |
| `get_correct_words(mac)` | POST /config/correct-words | 获取智能体替换词 |
| `lookup_address_book(caller_mac, nickname)` | GET /device/address-book/lookup | 通讯录查找目标设备 |

### 3.3 logger.py - 日志初始化

基于 Loguru，从 `config.yaml` 的 `log:` 节读取格式、级别、输出目录；支持控制台 + 文件轮转（10MB/文件、保留30天）。每个连接可通过 `create_connection_logger` 绑定 `selected_module` 标签。

## 4. WebSocket 服务 (core/websocket_server.py)

### 4.1 生命周期

```
WebSocketServer.__init__()
 ├── initialize_modules()              # 按 selected_module 初始化 VAD/ASR/LLM/Memory/Intent
 │   (注意：TTS 不在此初始化，而是 per-connection)
 └── AuthManager(auth_key, expire)     # 构造认证管理器

WebSocketServer.start()
 └── websockets.serve(handler, host, port, process_request=_http_response)
     ├── _http_response()              # 非Upgrade HTTP请求返回200 "Server is running"
     └── _handle_connection()
         ├── 提取 device-id (header 或 query)
         ├── _handle_auth(websocket)   # 认证
         │   ├── 白名单设备 → 跳过
         │   └── Bearer token → AuthManager.verify_token()
         ├── ConnectionHandler(...).handle_connection(websocket)
         └── finally: websocket.close()
```

### 4.2 热更新机制

```
update_config()
 ├── get_config_from_api_async()       # 重新拉取配置
 ├── check_vad_update()                # 检查VAD配置变更
 ├── check_asr_update()                # 检查ASR配置变更
 └── initialize_modules()              # 重新初始化全局模块
```

## 5. 连接处理 (core/connection.py) - 核心状态机

### 5.1 连接建立流程

```
ConnectionHandler.handle_connection(websocket)
 ├── 获取 loop, headers, device_id, client_ip(支持X-Real-IP/X-Forwarded-For)
 ├── 检测 MQTT 网关 (?from=mqtt_gateway)
 ├── 初始化活动时间戳 (first_activity_time, last_activity_time)
 ├── _check_timeout() 启动超时检测任务(每10秒检查一次)
 ├── 构造 welcome_msg (含 session_id, audio_params)
 ├── asyncio.create_task(_background_initialize())  # 不阻塞收包
 │   ├── _initialize_private_config_async()
 │   │   ├── get_private_config_from_api(device_id, client_id, selected_module)
 │   │   │   → POST /config/agent-models + POST /config/correct-words
 │   │   ├── 处理 DeviceNotFoundException → need_bind=true
 │   │   ├── 处理 DeviceBindException → need_bind=true, bind_code
 │   │   ├── 解析私有配置: VAD/ASR/LLM/TTS/VLLM/Memory/Intent
 │   │   ├── 解析 prompt/voiceprint/summaryMemory/mcp_endpoint
 │   │   ├── 注入 correct_words 到 TTS 配置
 │   │   └── 在线程池中 initialize_modules() 初始化各组件
 │   ├── _initialize_components() (在线程池中执行)
 │   │   ├── _initialize_tts() → DefaultTTS 兜底
 │   │   ├── tts.open_audio_channels() → 打开语音合成通道
 │   │   ├── _initialize_asr() → 本地ASR共享/远程ASR独立实例
 │   │   ├── _initialize_voiceprint() → 声纹识别初始化
 │   │   ├── asr.open_audio_channels() → 打开语音识别通道
 │   │   ├── _initialize_memory() → 记忆初始化(可能创建专用LLM)
 │   │   ├── _initialize_intent() → 意图识别(可能创建专用LLM)
 │   │   │   └── UnifiedToolHandler() → 异步初始化工具系统
 │   │   ├── _init_prompt_enhancement() → 增强提示词
 │   │   └── _inject_tool_call_fewshot() → 注入few-shot示例
 │   └── bind_completed_event.set()     # 标记初始化完成
 └── async for message in websocket:
     └── _route_message(message)
```

> **运行验证**：本节描述的 `_initialize_private_config_async()` 绑定状态判定（DeviceNotFoundException → `need_bind=True`、DeviceBindException → `need_bind=True, bind_code`）、`_route_message()` 消息路由拦截、`check_bind_device()` 绑定码播报等关键路径，均已通过 Docker 容器 + Mock manager-api 端到端验证（3 种绑定状态场景：已绑定 / 未注册 / 注册未绑定），详细证据见 **17-设备启动流程运行验证报告.md** §5（绑定设备流程验证）、§7.3（全模块部署完整时序）。

### 5.2 消息路由

```
_route_message(message)
 ├── 若 bind_completed_event 未就绪 → 等待最多1秒
 │   └── 超时 → _discard_message_with_bind_prompt()
 ├── 若 need_bind → 丢弃(可能播报绑定提示)
 ├── str 类型 → handleTextMessage(conn, message)
 └── bytes 类型:
     ├── MQTT带头(≥16字节) → _process_mqtt_audio_message() (时间戳重排)
     └── 普通音频 → asr_audio_queue.put(message)
```

### 5.3 对话核心流程

`chat()` 方法支持递归调用（最大深度 `MAX_DEPTH=5`），是 Agent 循环的核心。

```
ConnectionHandler.chat(user_text, depth=0)
 ├── depth==0 时:
 │   ├── 生成 sentence_id
 │   ├── dialogue.put(user消息)
 │   └── tts_text_queue.put(FIRST)        # 标记新一轮开始
 │
 ├── depth≥MAX_DEPTH 时:
 │   ├── force_final_answer = True        # 强制直接回答
 │   └── 注入系统提示"不要再调用工具"
 │
 ├── memory.query_memory()                # 查询记忆(线程安全future)
 ├── 构建 functions 列表 (仅 function_call 模式):
 │   ├── func_handler.get_functions()      # 真实工具列表
 │   └── depth==0 时追加 direct_answer 虚拟工具
 │       (让模型选择"调工具"还是"直接回答")
 │
 ├── llm.response() 或 response_with_functions()  # LLM流式推理
 │   ├── 每个文本片段 → tts_text_queue   # 实时送TTS
 │   ├── direct_answer 流式提取:
 │   │   └── 从 arguments 的 response 参数实时抽取文本 → tts_text_queue
 │   ├── emotion 表情提取(仅首轮)
 │   └── function_call 收集:
 │       ├── _merge_tool_calls()           # 合并多个并行工具调用
 │       └── 支持 <tool_call> 文本格式和原生 function calling
 │
 ├── 处理工具调用结果:
 │   ├── direct_answer → 已流式播报，写入对话历史
 │   ├── 真实工具 → enqueue_tool_report → handle_llm_function_call
 │   ├── _handle_function_result():
 │   │   ├── Action.RESPONSE → 直接TTS播报
 │   │   ├── Action.REQLLM → 工具结果写入dialogue → 递归 chat(None, depth+1)
 │   │   ├── Action.RECORD → 写入完整工具调用链(assistant→tool→assistant)
 │   │   └── Action.ERROR/NOTFOUND → 播报错误信息
 │   └── 工具超时: tool_call_timeout(默认30秒)
 │
 └── depth==0 时: tts_text_queue.put(LAST)  # 标记本轮结束
```

**direct_answer 虚拟工具**：不是真实工具，是路由机制——将"调不调工具"的二选一变为"调哪个"的多选，防止小模型误触发真实工具。仅在 depth=0 时注入，递归调用时不注入。

### 5.4 连接断开流程

```
finally → _save_and_close()
 ├── 守护线程1: generate_and_save_chat_title()   # 独立线程生成会话标题
 ├── 守护线程2: memory.save_memory()              # 独立线程保存记忆
 │   (两个线程均为 daemon=True，不阻塞主流程)
 └── close(websocket)                            # 立即执行，不等待线程完成
     ├── vad.release_conn_resources()            # 清理VAD资源
     ├── func_handler.cleanup()                  # 清理工具(关MCP等)
     ├── 取消超时检测任务
     ├── stop_event.set()                        # 触发停止事件
     ├── clear_queues()                          # 清空TTS/ASR/上报队列
     ├── websocket.close()
     ├── 关闭 TTS/ASR
     └── executor.shutdown(wait=False)
```

## 6. 消息处理体系 (core/handle/)

### 6.1 文本消息分发

```
handleTextMessage(conn, message)
 └── TextMessageProcessor.process_message()
     ├── json.loads(message)
     ├── 按 type 字段查找 TextMessageHandlerRegistry
     └── handler.handle(conn, msg_json)
```

**注册的处理器**：

| type     | 处理器                      | 功能                               |
| -------- | ------------------------ | -------------------------------- |
| `hello`  | HelloTextMessageHandler  | 解析音频参数、features、MCP初始化、回发welcome |
| `abort`  | AbortTextMessageHandler  | 客户端中止：清队列、发TTS stop              |
| `listen` | ListenTextMessageHandler | 语音监听控制(start/stop/detect)        |
| `iot`    | IotTextMessageHandler    | IoT设备描述符/状态                      |
| `mcp`    | McpTextMessageHandler    | MCP协议消息                          |
| `server` | ServerTextMessageHandler | 管理端控制(update_config/restart)     |
| `ping`   | PingMessageHandler       | WebSocket心跳                      |

### 6.2 音频处理流程

```
接收音频二进制 → asr_audio_queue
 └── ASR线程: asr_text_priority_thread
     └── handleAudioMessage(conn, audio_data)
         ├── 唤醒后短暂跳过VAD (just_woken_up, 2秒后恢复)
         ├── vad.is_vad(conn, audio)        # VAD检测
         │   ├── 说话中 → asr.receive_audio()
         │   └── 静音超时 → no_voice_close_connect
         └── no_voice_close_connect:
             └── 超时无声 → 结束语(end_prompt)或关闭连接

语音结束(listen stop / VAD判停):
 └── handle_voice_stop()
     ├── 解码 Opus → PCM
     ├── ASR识别 → 文本
     ├── enqueue_asr_report()             # 上报ASR结果
     └── startToChat(conn, text)

startToChat(conn, text)
 ├── 解析JSON格式(含speaker/language/content)
 ├── 绑定检查 → check_bind_device()
 ├── 输出字数限额 → max_out_size()
 ├── handleAbortMessage() (非manual且客户端在播时)
 ├── handle_user_intent(text)
 │   ├── 退出命令检测 (check_direct_exit)
 │   ├── 唤醒词检测 (checkWakeupWords)
 │   ├── 若 intent_type=="function_call" → 返回False(跳过意图分析)
 │   ├── 若 intent_type=="intent_llm":
 │   │   ├── intent.detect_intent(conn, dialogue, text)
 │   │   └── process_intent_result()
 │   │       ├── function_call → 工具执行
 │   │       ├── continue_chat → 返回False
 │   │       ├── result_for_context → 直接TTS播报
 │   │       └── speak_txt → 直接TTS播报
 │   └── 若 intent_type=="nointent" → 返回False
 ├── send_stt_message()                   # 发送识别文本到客户端
 └── executor.submit(conn.chat, text)     # 在线程池中启动对话
```

**listen detect 状态的特殊处理**：
- `[device_call]` 前缀：提取后文本直接播报给设备，不经过 LLM
- 唤醒词：根据 `enable_greeting` 配置决定是否回复“嘿，你好呀”
- 普通文本：进入 startToChat 流程

### 6.3 上报机制

上报功能受 `chat_history_conf` 配置控制：0=关闭，1=仅文本，2=文本+音频。

```
reportHandle:
 ├── enqueue_asr_report(conn, text, opus_data)   # ASR上报(chatType=1 用户)
 ├── enqueue_tts_report(conn, text, opus_data)   # TTS上报(chatType=2 智能体)
 └── enqueue_tool_report(conn, tool_name, tool_input, tool_result)  # 工具上报(chatType=3)
     ├── report_tool_call=True → 上报工具调用本身
     └── report_tool_call=False → 仅上报工具结果

→ report_queue → _report_worker (后台线程)
  └── _process_report(type, text, opus_data, report_time)
      ├── opus_to_wav() → Opus解码为WAV (16kHz/单声道)
      └── ManageApiClient.report({
            macAddress, sessionId, chatType,
            content, reportTime, audioBase64
          })
          → POST /agent/chat-history/report
```

## 7. HTTP 服务 (core/http_server.py + core/api/)

### 7.1 路由注册

```
SimpleHttpServer.start()
 ├── 若 read_config_from_api == false:
 │   ├── GET/POST/OPTIONS /xiaozhi/ota/       # OTA上报+健康检查
 │   └── GET/OPTIONS /xiaozhi/ota/download/{filename}  # 固件下载
 └── 始终注册:
     └── GET/POST/OPTIONS /mcp/vision/explain  # 视觉分析
```

### 7.2 OTA处理 (ota_handler.py)

```
POST /xiaozhi/ota/
 ├── 读取 device-id / client-id
 ├── 获取设备型号与固件版本
 ├── 组装 server_time
 ├── 若配置 mqtt_gateway:
 │   └── 生成 MQTT client_id, username, HMAC密码
 ├── 否则:
 │   └── WebSocket URL + 可选 AuthManager.generate_token()
 ├── 扫描 data/bin/*.bin → 版本比较
 │   └── 有新版本 → firmware.url = /xiaozhi/ota/download/{filename}
 └── 返回 JSON {server_time, websocket, mqtt, firmware}

GET /xiaozhi/ota/download/{filename}
 ├── 防路径穿越检查
 └── 仅允许 data/bin/ 目录下文件
```

### 7.3 视觉处理 (vision_handler.py)

```
POST /mcp/vision/explain
 ├── _verify_auth_token()               # JWT认证(AuthToken)
 │   ├── Authorization: Bearer 校验
 │   └── Device-Id 与 token内设备一致
 ├── 读取 multipart: question + image
 ├── 可选: get_private_config_from_api() # 拉取设备配置
 ├── create_instance(VLLM)              # 创建视觉模型实例
 └── vllm.response(question, image_base64) → JSON结果
```

## 8. Provider 体系 (core/providers/)

### 8.1 架构模式

每个 Provider 域遵循统一模式：
- `base.py`：抽象基类定义接口
- 各厂商实现文件（扁平结构，每个厂商一个 `.py` 文件）
- `modules_initialize.py`：按配置开关实例化

```
providers/
├── asr/           # 语音识别 (14个实现)
│   ├── base.py    # ASRProviderBase
│   ├── dto/       # DTO定义(InterfaceType等)
│   ├── fun_local.py       # FunASR本地
│   ├── fun_server.py      # FunASR服务
│   ├── doubao.py          # 豆包(非流式)
│   ├── doubao_stream.py   # 豆包(流式)
│   ├── aliyun.py          # 阿里云(非流式)
│   ├── aliyun_stream.py   # 阿里云(流式)
│   ├── aliyunbl_stream.py # 阿里云百炼(流式)
│   ├── tencent.py         # 腾讯
│   ├── baidu.py           # 百度
│   ├── openai.py          # OpenAI Whisper
│   ├── xunfei_stream.py   # 讯飞(流式)
│   ├── qwen3_asr_flash.py # Qwen3 ASR
│   ├── sherpa_onnx_local.py # Sherpa ONNX本地
│   └── vosk.py            # Vosk本地
├── llm/           # 大语言模型 (9个实现)
│   ├── base.py    # LLMProviderBase
│   ├── system_prompt.py  # 系统提示词工具
│   ├── openai/    # OpenAI/兼容接口
│   ├── ollama/    # Ollama
│   ├── gemini/    # Google Gemini
│   ├── dify/      # Dify
│   ├── coze/      # Coze
│   ├── fastgpt/   # FastGPT
│   ├── homeassistant/ # Home Assistant
│   ├── xinference/ # Xinference
│   └── AliBL/     # 阿里百炼
├── tts/           # 语音合成 (19个实现)
│   ├── base.py    # TTSProviderBase
│   ├── dto/       # DTO定义(ContentType, SentenceType等)
│   ├── default.py  # DefaultTTS(兜底/音频文件播放)
│   ├── edge.py     # Edge TTS
│   ├── openai.py   # OpenAI TTS
│   ├── aliyun.py   # 阿里云
│   ├── aliyun_stream.py / alibl_stream.py  # 阿里云流式
│   ├── doubao.py   # 豆包
│   ├── tencent.py  # 腾讯
│   ├── cozecn.py   # Coze
│   ├── fishspeech.py # FishSpeech
│   ├── siliconflow.py # SiliconFlow
│   ├── gpt_sovits_v2.py / gpt_sovits_v3.py # GPT-SoVITS
│   ├── huoshan_double_stream.py # 火山双流
│   ├── index_stream.py # Index流式
│   ├── minimax_httpstream.py # MiniMax
│   ├── paddle_speech.py # PaddleSpeech
│   ├── xunfei_stream.py # 讯飞流式
│   └── custom.py   # 自定义TTS
├── vad/           # 语音活动检测
│   ├── base.py    # VADProviderBase
│   └── silero.py  # Silero VAD
├── memory/        # 记忆管理 (5个实现)
│   ├── base.py    # MemoryProviderBase
│   ├── nomem/     # 无记忆
│   ├── mem_local_short/ # 本地短期记忆
│   ├── mem_report_only/ # 仅上报模式
│   ├── mem0ai/    # mem0 AI记忆
│   └── powermem/  # PowerMem
├── intent/        # 意图识别 (3个实现)
│   ├── base.py    # IntentProviderBase
│   ├── nointent/  # 无意图
│   ├── function_call/ # Function Calling
│   └── intent_llm/   # LLM意图
├── vllm/          # 视觉语言模型
│   ├── base.py    # VLLMProviderBase
│   └── openai.py  # OpenAI Vision
└── tools/         # 工具系统(见下节)
```

### 8.2 核心接口定义

| Provider | 核心方法 | 说明 |
|----------|---------|------|
| **LLM** | `response(session_id, dialogue)` | 流式文本生成 |
| | `response_with_functions(session_id, dialogue, functions)` | 支持Function Calling |
| **ASR** | `open_audio_channels(conn)` | 开启音频通道(启动asr_text_priority_thread) |
| | `receive_audio(conn, audio, have_voice)` | 接收音频数据 |
| | `handle_voice_stop(conn, audio_data)` | 处理语音结束(Opus解码+识别) |
| **TTS** | 基于 `tts_text_queue` / `tts_audio_queue` | 文本队列消费→音频队列输出 |
| | `tts_one_sentence()` | 单句合成 |
| | `store_tts_text(sentence_id, text)` | 存储文本用于上报 |
| **VAD** | `is_vad(conn, audio)` | VAD检测 |
| **Memory** | `init_memory(role_id, llm, summary_memory, save_to_file)` | 初始化记忆 |
| | `query_memory(query)` | 查询相关记忆 |
| | `save_memory(dialogue, session_id)` | 保存记忆 |
| | `set_llm(llm)` | 设置记忆总结用LLM |
| **Intent** | `detect_intent(conn, dialogue_history, text)` | 意图识别(返回JSON字符串) |
| | `set_llm(llm)` | 设置意图识别用LLM |
| | `replyResult(context, text)` | 基于上下文生成回复 |
| **VLLM** | `response(question, image)` | 视觉问答 |

## 9. 工具系统 (core/providers/tools/)

### 9.1 统一工具架构

```
UnifiedToolHandler
 ├── ToolManager (工具管理器)
 │   ├── ServerPluginExecutor    # 服务端插件
 │   ├── ServerMCPExecutor       # 服务端MCP工具
 │   ├── DeviceIoTExecutor       # 设备IoT控制
 │   ├── DeviceMCPExecutor       # 设备MCP工具
 │   └── MCPEndpointExecutor     # MCP接入点工具
 └── 初始化流程:
     ├── 导入插件
     ├── server_mcp_executor.initialize()
     ├── _initialize_mcp_endpoint()     # 连接MCP端点
     └── hass_init.append_devices_to_prompt() # HomeAssistant设备
```

### 9.2 工具执行流程

```
handle_llm_function_call(function_name, arguments)
 ├── send_display_message()             # 向客户端显示工具名称
 ├── tool_manager.execute_tool(name, args)
 │   ├── 按工具名查找 ToolType
 │   └── 分发到对应 Executor.execute()
 └── 返回 ActionResponse
```

### 9.3 设备MCP

```
hello消息(features.mcp) → 创建MCPClient
 └── send_mcp_initialize_message()
     → 设备端回复 → handle_mcp_message()
     → 工具列表合并到 LLM tools
```

### 9.4 IoT处理

```
handleIotDescriptors(conn, descriptors)
 ├── 等待 func_handler.finish_init
 ├── 构建 IotDescriptor
 └── register_iot_tools()               # 注册为可调用工具

handleIotStatus(conn, status)
 └── 更新描述符属性值
```

## 10. 插件系统 (plugins_func/)

### 10.1 加载机制

```
loadplugins.auto_import_modules("plugins_func.functions")
 └── pkgutil 遍历子模块 → importlib.import_module()
     → 模块顶层 @register_function 装饰器执行

注册流程:
@register_function → FunctionItem → all_function_registry 全局注册表
```

### 10.2 内置插件

| 插件 | 功能 |
|------|------|
| `get_weather` | 天气查询 |
| `get_time` | 时间查询 |
| `get_news_from_chinanews` | 中国新闻 |
| `get_news_from_newsnow` | NewsNow新闻 |
| `hass_get_state` | HomeAssistant状态查询 |
| `hass_set_state` | HomeAssistant状态设置 |
| `hass_play_music` | HomeAssistant播放音乐 |
| `search_from_ragflow` | RAGFlow知识库检索 |
| `handle_exit_intent` | 退出意图处理 |
| `play_music` | 本地音乐播放 |
| `web_search` | 网络搜索 |
| `call_device` | 呼叫设备 |
| `change_role` | 切换角色 |

**Action 枚举类型**：

| Action | 含义 |
|--------|------|
| `ERROR`(-1) | 错误 |
| `NOTFOUND`(0) | 没有找到函数 |
| `NONE`(1) | 不做任何操作 |
| `RESPONSE`(2) | 直接回复前端 |
| `REQLLM`(3) | 调用函数后再请求LLM生成回复 |
| `RECORD`(4) | 记录工具调用到对话历史，不调用LLM |

### 10.3 执行路径

```
LLM function_call → UnifiedToolHandler
 → ToolManager.execute_tool()
 → ServerPluginExecutor.execute(func_name, args)
   ├── 按 ToolType 决定是否传入 conn
   └── 调用插件函数 → ActionResponse
```

## 11. 认证体系

### 11.1 WebSocket/OTA设备认证 (core/auth.py - AuthManager)

- **算法**：HMAC-SHA256(client_id | username | timestamp) → urlsafe Base64
- **Token格式**：`{signature}.{timestamp}`
- **有效期**：默认30天
- **使用场景**：WebSocket连接握手、OTA返回token

### 11.2 HTTP Vision认证 (core/utils/auth.py - AuthToken)

- **算法**：JWT + AES-GCM
- **使用场景**：视觉识别API、设备MCP API
- **校验**：Bearer token + Device-Id头匹配

## 12. 端到端完整链路：一句话→AI回复

```
1. 设备WebSocket连上 → WebSocketServer认证
2. ConnectionHandler.handle_connection 建立
3. {"type":"hello",...} → 更新音频参数/MCP → 回发welcome_msg
4. 二进制Opus帧 → asr_audio_queue → handleAudioMessage → VAD检测
5. VAD判停 / listen stop → handle_voice_stop → ASR识别出文字
6. startToChat(text):
   ├── JSON解析(speaker/language/content)
   ├── 意图识别 (退出/唤醒词/function_call跳过)
   └── chat():
       ├── memory.query_memory()
       ├── LLM流式推理 → 文本片段进TTS队列
       │   ├── direct_answer 流式抽取 → TTS队列
       │   └── 若function_call → 工具执行 → 结果写回dialogue → 可能递归chat
       └── TTS队列消费 → sendAudioHandle → 音频流控 → WebSocket二进制帧
7. 上报: ASR文本/TTS文本/工具调用 → report_queue → POST /agent/chat-history/report
8. 断开: save_memory(守护线程) + generate_chat_title(守护线程) → close
```

## 13. 提示词与声纹管理

### 13.1 PromptManager

每个连接创建独立的 `PromptManager` 实例，负责：
- **快速提示词**：初始化时直接设置用户配置的 prompt
- **增强提示词**：后续根据上下文信息（时间、天气、位置等）动态构建增强版 prompt
- **上下文更新**：从 context_providers 获取外部信息

### 13.2 VoiceprintProvider

每个连接独立初始化声纹识别：
- 从配置的 `voiceprint` 节读取声纹服务地址和说话人配置
- 支持声纹相似度匹配，用于识别说话人身份
- 声纹信息注入到对话上下文中，影响 LLM 的回复

### 13.3 替换词注入

从智控台拉取的私有配置中包含 `correct_words`，会注入到当前 TTS 模块配置中：
```
private_config["correct_words"] → config["TTS"][selected_tts_module]["correct_words"]
```
用于 ASR 识别结果的文本替换，提高专业词汇的识别准确率。
