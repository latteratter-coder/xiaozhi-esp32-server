# xiaozhi-server Agent 与记忆系统全解析

> 面向初学者的完整指南：从概念到源码，用生活类比帮你彻底理解 Python 端的 AI 核心。
>
> **相关文档：**
> - Java 端模块分析（agent/llm/timbre/voiceclone） → 见 `14-agent-llm-timbre-voiceclone模块原理详解.md`
> - 百万设备部署方案 → 见 `15-百万设备部署架构方案.md`

[TOC]

---

# 第一部分：核心概念

> 在看代码之前，先搞清楚三个概念：Agent、Chain of Thought、Function Calling。

## 1. 什么是 Agent？

### 1.1 最简定义

**Agent = LLM + 自主决策 + 工具调用 + 循环推理**

普通的 LLM 调用是"你问它答"——给一段话，回一段话。Agent 则是让 LLM **自己判断该做什么**，需要时**自己去调用工具**，拿到结果后**自己决定是否继续**。

### 1.2 五个层次对比

| 层次 | 名称 | 能力 | 例子 |
|:----:|------|------|------|
| L0 | 纯 LLM | 只能生成文本，不能执行任何动作 | 你问"北京天气"，它编一个答案 |
| L1 | LLM + 固定流程 | 按预设代码流程调工具，LLM 不参与决策 | 写死 if/else：识别到"天气"就查 API |
| L2 | LLM + Function Calling | LLM **自己决定**要不要调工具、调哪个 | 问天气它调天气 API，聊天就不调 |
| L3 | **Agent（完整）** | 调工具→拿结果→**再次推理**→可能再调工具→循环 | 问"比较北京上海天气"→先查北京→再查上海→总结 |
| L4 | 多 Agent 协作 | 多个 Agent 各司其职，互相通信 | 搜索Agent + 分析Agent + 写作Agent |

**xiaozhi-server 处于 L3 级别——完整的 Agent。**

### 1.3 Agent 的四个核心要素

```
┌─────────────────────────────────────────────────┐
│                    Agent 循环                     │
│                                                   │
│  ① 感知 (Perception)                              │
│     接收用户输入（语音/文字）                        │
│                     ↓                              │
│  ② 推理 (Reasoning)                               │
│     LLM 分析输入，决定下一步：                       │
│     - 直接回答？                                    │
│     - 调用工具？调哪个？参数是什么？                  │
│                     ↓                              │
│  ③ 行动 (Action)                                  │
│     执行工具调用，获取真实世界数据                     │
│     （查天气/搜索/控制设备/放音乐...）                │
│                     ↓                              │
│  ④ 反思 (Reflection)                              │
│     拿到工具结果后再次推理：                          │
│     - 够了，可以回答 → 生成最终回答                   │
│     - 不够，还需要信息 → 回到 ② 继续循环              │
│                                                   │
│     最多循环 N 轮后强制回答                          │
└─────────────────────────────────────────────────┘
```

这个"推理→行动→反思→再推理"的循环，就是经典的 **ReAct 模式**（Reasoning + Acting）。

### 1.4 Agent 判据（缺一不可）

1. **LLM 做决策者**：不是人写 if/else 决定调什么，而是 LLM 自己判断
2. **能调用外部工具**：查天气、放音乐、搜索、控制设备——与真实世界交互
3. **工具结果回流 LLM**：工具返回的数据**喂回**给 LLM，让它基于真实数据生成回答
4. **可以多轮循环**：一次不够可以再调，形成"推理→行动→观察→再推理"的闭环

---

## 2. 什么是 Chain of Thought（CoT）？

### 2.1 通俗解释

**Chain of Thought（思维链）= 让 AI 先"想一想"，再给出答案。**

就像老师要求学生"写出解题步骤"一样。

### 2.2 例子对比

**不用 CoT**：

```
用户：小明有 5 个苹果，给了小红 2 个，又买了 3 个，还剩几个？
AI：6 个。
```

**用 CoT**：

```
用户：小明有 5 个苹果，给了小红 2 个，又买了 3 个，还剩几个？
AI：
<think>
1. 小明开始有 5 个苹果
2. 给了小红 2 个：5 - 2 = 3
3. 又买了 3 个：3 + 3 = 6
</think>
小明还剩 6 个苹果。
```

### 2.3 三种实现方式

| 方式 | 实现 | 代表模型 |
|------|------|---------|
| **Prompt CoT** | 在提示词中加"请一步步思考" | 任何模型 + 手动 Prompt |
| **模型内置 CoT** | 模型训练时就学会了先思考再回答，输出 `<think>...</think>` 标签 | DeepSeek-R1、QwQ、Kimi-k1.5 |
| **外部编排 CoT** | 框架层面强制拆步骤（如 LangChain 的 Agent Chain） | LangChain / AutoGen |

### 2.4 xiaozhi-server 对 CoT 的处理

**结论：兼容了 CoT，但没有主动要求 CoT。**

1. 不在系统提示词中要求 LLM 做 CoT
2. 但如果你配的 LLM 自带 CoT（如 DeepSeek-R1），它会输出 `<think>...</think>` 标签
3. xiaozhi-server 会**过滤掉思考过程，只把最终回答推给 TTS**

代码实现（`core/providers/llm/openai/openai.py`）：

```python
is_active = True
for chunk in responses:
    content = delta.content
    if content:
        if "<think>" in content:
            is_active = False                      # 进入思考区域，暂停输出
            content = content.split("<think>")[0]
        if "</think>" in content:
            is_active = True                       # 离开思考区域，恢复输出
            content = content.split("</think>")[-1]
        if is_active:
            yield content  # 只有非思考区域的文本才推给 TTS
```

**为什么要过滤？** 因为这是语音交互系统，不过滤的话设备会念出："让我想一想，用户问的是天气，北京今天... 我需要调用天气工具... 好的，北京今天天气晴朗"。用户只需要听到最后那句话。

| 模型 | 是否内置 CoT | 效果 |
|------|:-----------:|------|
| DeepSeek-V2/V3 | 否 | 直接输出回答 |
| **DeepSeek-R1** | **是** | 先输出 `<think>思考过程</think>`，再输出回答 |
| **QwQ (通义)** | **是** | 同上 |
| GPT-4o | 否 | 直接输出回答 |
| Kimi-k1.5 | 是 | 同上 |

策略：**不限制你用什么模型，如果模型自带思考过程就自动过滤掉**。

---

## 3. 两种 Function Calling 协议

xiaozhi-server 兼容两种工具调用格式：

**格式 1：OpenAI 原生 `tool_calls`**（主流模型支持）

LLM 在流式响应中返回结构化数据：

```json
{"tool_calls": [{"id": "call_123", "function": {"name": "get_weather", "arguments": "{\"location\":\"北京\"}"}}]}
```

**格式 2：文本 `<tool_call>` 标签**（兼容不支持原生 Function Calling 的模型）

LLM 输出纯文本中嵌入 JSON：

```
<tool_call>
{"name": "get_weather", "arguments": {"location": "北京"}}
</tool_call>
```

`system_prompt.py` 中有完整的格式教学 Prompt，专门给不支持原生 Function Calling 的模型使用。

---

# 第二部分：chat() Agent 循环——源码逐行解读

> 用"前台接待员"的类比，把 `core/connection.py` 的 `chat()` 方法拆成 9 个阶段讲透。
>
> **架构图**：请用 VS Code Draw.io 插件打开 → [15-01-agent-architecture.drawio](15-01-agent-architecture.drawio)

## 4. 先用一个生活场景来理解

**忘掉代码，先想象这个场景：**

你是一个前台接待员，面前坐着一个客人。你手边有一本通讯录（工具），里面有天气查询电话、音乐点播电话、新闻热线等。

客人说：**"今天北京天气怎么样？"**

你的工作流程是：
1. 你**听到**客人的话（接收输入）
2. 你**判断**：这个问题我自己回答不了，需要打电话查天气（决定调工具）
3. 你**拿起电话**打给天气服务，得到"北京今天晴，25度"（执行工具）
4. 你**挂掉电话**，把结果组织成好听的话告诉客人：**"今天北京天气不错哦，晴天25度，很适合出去走走~"**（把工具结果交给 LLM 组织语言）

如果客人说的是：**"你好呀"**——你不需要打任何电话，直接微笑回答就行。

**`chat()` 方法就是这个"前台接待员"的完整工作流程。**

---

## 5. chat() 源码逐行通俗解读

把 `connection.py` 的 `chat()` 方法拆成 **9 个阶段**，每个阶段先说**"它在干什么"**，再贴**对应的源码**。

### 阶段 1：新客人来了，做登记（第 861-882 行）

**通俗理解**：客人（用户）说了一句话，先登记下来。

```python
def chat(self, query, depth=0):
    #【depth=0 意思是"这是用户直接说的话"】
    #【depth>0 意思是"这是工具返回结果后，我再找 LLM 组织语言"】
    
    if depth == 0:
        # 给这轮对话起个编号（像排队号）
        current_sentence_id = str(uuid.uuid4().hex)
        
        # 把用户说的话记到"对话记录本"上
        self.dialogue.put(Message(role="user", content=query))
        
        # 通知音箱："准备好，马上要说话了"
        self.tts.tts_text_queue.put(TTSMessageDTO(sentence_type=SentenceType.FIRST))
    else:
        # 递归回来的，不需要重新登记，用原来的编号
        current_sentence_id = self.sentence_id
```

**类比**：你在本子上写下客人的问题，准备开始处理。

### 阶段 2：设置安全限制——最多打 5 个电话（第 884-899 行）

**通俗理解**：防止你一直打电话不回来。

```python
    MAX_DEPTH = 5  # 最多允许"打电话"5次
    
    if depth >= MAX_DEPTH:
        # 已经打了5次电话了！不能再打了！
        self.dialogue.put(Message(
            role="user",
            content="[系统提示] 已达到最大工具调用次数限制，直接给出答案。"
        ))
        force_final_answer = True
```

**类比**：公司规定，接待一个客人最多打 5 个电话，打完必须给出答复。

**为什么需要这个？** LLM 犯傻时：查天气失败 → 再查一次 → 又失败 → 再查... 没有限制就死循环了。

### 阶段 3：检查是不是在"偷懒"（第 901-964 行）

**通俗理解**：检查你最近是不是偷懒，该打电话的时候没打。

```python
    if depth == 0 and query is not None:
        turns_since_last = current_turn - self.tool_call_stats['last_call_turn']
        
        if turns_since_last > 3:
            # 已经连续3轮没打过电话了！可能在偷懒
            force_reminder = True
```

如果检测到偷懒，就临时贴一张便签：

```python
    if tool_call_reminder:
        self.dialogue.put(Message(
            role="user",
            content="【提醒】你有这些电话可以打：天气、音乐、新闻...",
            is_temporary=True  # ← 标记为"临时的"，事后会撕掉
        ))
```

**类比**：主管贴便签提醒你："记得该查的时候要查，别瞎编！"用完后撕掉。

### 阶段 4：翻看之前的笔记（查记忆）（第 972-980 行）

**通俗理解**：在回答之前，先翻翻之前对这个客人的记录。

```python
    if self.memory is not None and query:
        memory_str = self.memory.query_memory(query)
        # 比如查到："这个客人叫小明，住北京，上次问过跑步的事"
```

**类比**：翻开客户档案，看看这个人以前来过没、有什么偏好。

### 阶段 5：把所有信息一起交给大脑（调 LLM）（第 982-997 行）

**通俗理解**：你把客人的话、你的记忆、可用的电话列表，一起"想一下"。

```python
    if self.intent_type == "function_call" and functions is not None:
        llm_responses = self.llm.response_with_functions(
            self.session_id,
            self.dialogue.get_llm_dialogue_with_memory(memory_str, ...),
            functions=functions,  # ← 可用工具列表
        )
    else:
        llm_responses = self.llm.response(...)
```

**类比**：把客人的需求、客户档案、手边的通讯录全摊在桌上，开始思考怎么回答。

### 阶段 6：听 LLM 的回复——直接回答还是要打电话？（第 1002-1048 行）

**通俗理解**：LLM 的回复是一个字一个字冒出来的（流式），判断它是在直接回答还是在要求"去打电话"。

```python
    tool_call_flag = False
    tool_calls_list = []
    
    for response in llm_responses:
        content, tools_call = response
        
        if tools_call is not None:
            tool_call_flag = True
            self._merge_tool_calls(tool_calls_list, tools_call)
        
        if content is not None and not tool_call_flag:
            # LLM 直接在说话 → 立刻推给 TTS
            self.tts.tts_text_queue.put(TTSMessageDTO(content_detail=content))
```

| LLM 的决定 | 类比 | 代码中的表现 |
|-----------|------|-------------|
| "北京今天天气不错~" | 你直接回答客人 | content 有值，tools_call 为空 → 推给 TTS |
| "我需要查一下天气" | 你决定打电话 | tools_call 有值 → 记到 tool_calls_list |

### 阶段 7：打电话（执行工具）（第 1098-1164 行）

**通俗理解**：如果 LLM 说要打电话，你就去打。

```python
    if tool_call_flag and len(tool_calls_list) > 0:
        # 先把所有电话一起拨出去（并发）
        futures = []
        for tool_call_data in tool_calls_list:
            future = asyncio.run_coroutine_threadsafe(
                self.func_handler.handle_llm_function_call(self, tool_call_data),
                self.loop,
            )
            futures.append(future)
        
        # 然后等所有电话都有回音
        for future in futures:
            result = future.result(timeout=30)
            tool_results.append(result)
        
        self._handle_function_result(tool_results, depth=depth)
```

**类比**：同时查天气和查新闻，两个电话同时拨，等都回复了一起处理——比一个一个打快。

### 阶段 8：处理电话结果——关键分支（第 1224-1277 行）

**通俗理解**：电话打完了，看结果怎么处理。**这是整个 Agent 最关键的一步。**

```python
    def _handle_function_result(self, tool_results, depth):
        for result, tool_call_data in tool_results:
            
            if result.action == Action.RESPONSE:
                # 直接告诉客人就行（比如播放音乐 → "好的，正在播放"）
                self.tts.tts_one_sentence(self, text)
                self.dialogue.put(Message(role="assistant", content=text))
            
            elif result.action == Action.REQLLM:
                # 原始数据需要 LLM 加工（比如天气数据 → 需要组织语言）
                need_llm_tools.append(result)
        
        if need_llm_tools:
            # 把电话结果记到"对话记录本"上
            self.dialogue.put(Message(role="assistant", tool_calls=[...]))
            self.dialogue.put(Message(role="tool", content="晴，25°C，风力3级"))
            
            # ★★★ 递归！再找 LLM 组织语言 ★★★
            self.chat(None, depth=depth + 1)
```

两种 Action 的类比：

- `Action.RESPONSE` = 你打电话点了首歌，对方说"已播放"，你直接告诉客人"放好了"
- `Action.REQLLM` = 你打电话查天气，对方报了一堆数字，你需要**重新组织语言**再告诉客人 → **触发递归**

所有 Action 类型：

| Action | 含义 | 后续行为 |
|--------|------|---------|
| `REQLLM` | 结果需要 LLM 加工 | 工具结果写入对话 → 递归 `chat(depth+1)` |
| `RESPONSE` | 直接回复用户 | TTS 直接念出 → 不再经过 LLM |
| `NONE` | 不做任何事 | 静默 |
| `ERROR` | 工具出错 | 回复错误提示 |
| `NOTFOUND` | 工具未找到 | 回复未找到提示 |

### 阶段 9：收尾（第 1166-1201 行）

```python
    if len(response_message) > 0:
        text_buff = "".join(response_message)
        self.dialogue.put(Message(role="assistant", content=text_buff))
    
    if depth == 0:
        # 通知音箱："这轮话说完了"
        self.tts.tts_text_queue.put(TTSMessageDTO(sentence_type=SentenceType.LAST))
        
        # 撕掉之前贴的临时便签（偷懒提醒）
        self.dialogue.dialogue = [
            msg for msg in self.dialogue.dialogue
            if not getattr(msg, 'is_temporary', False)
        ]
```

**类比**：客人走了，撕掉临时便签，整理好本子，等下一个客人。

---

## 6. 用一个完整例子串起来

用户说：**"查一下北京天气，然后放首歌"**

```
chat("查一下北京天气，然后放首歌", depth=0)
│
├─ 阶段1: 登记用户的话
├─ 阶段2: depth=0 < 5，不限制
├─ 阶段3: 偷懒检测（假设正常）
├─ 阶段4: 查记忆 → "用户喜欢周杰伦"
├─ 阶段5: 把一切交给 LLM
│
├─ 阶段6: LLM 返回 tool_calls:
│   [get_weather("北京"), play_music("周杰伦")]  ← 两个工具！
│
├─ 阶段7: 同时执行两个工具（并发）
│   ├─ get_weather("北京") → "晴，25°C"（Action.REQLLM）
│   └─ play_music("周杰伦") → "正在播放晚晴"（Action.RESPONSE）
│
├─ 阶段8: 处理结果
│   ├─ play_music 是 RESPONSE → 直接念："正在为您播放晚晴"
│   └─ get_weather 是 REQLLM → 需要 LLM 加工 → 递归！
│       │
│       └─ chat(None, depth=1)
│           ├─ 阶段5: LLM 看到天气数据
│           ├─ 阶段6: LLM 生成："北京今天晴天25度，很适合出门~"
│           │   （纯文本，不调工具 → 推给 TTS）
│           └─ 返回
│
├─ 阶段9: depth=0 收尾
│   ├─ 通知 TTS "说完了"
│   └─ 清理临时便签
└─ 完成
```

用户听到的是：
1. **"正在为您播放晚晴"**（play_music 直接回复，最先出声）
2. **"北京今天晴天25度，很适合出门~"**（天气数据经 LLM 加工后出声）
3. 背景开始播放音乐

---

## 7. 可用工具一览

### 7.1 内置插件（`plugins_func/functions/` 目录）

| 工具 | 功能 | Action |
|------|------|:------:|
| `get_weather` | 查天气（和风天气 API） | REQLLM |
| `play_music` | 播放音乐 | RESPONSE |
| `get_news_from_newsnow` | 获取新闻 | REQLLM |
| `get_news_from_chinanews` | 获取中新网新闻 | REQLLM |
| `get_time` | 查询时间 | REQLLM |
| `handle_exit_intent` | 退出对话 | RESPONSE |
| `change_role` | 切换 AI 角色 | RESPONSE |
| `search_from_ragflow` | RAGFlow 知识库搜索 | REQLLM |
| `hass_get_state` | 读取 Home Assistant 设备状态 | REQLLM |
| `hass_set_state` | 控制 Home Assistant 设备 | RESPONSE |
| `hass_play_music` | HA 播放音乐 | RESPONSE |

### 7.2 动态工具（五类执行器）

| 类型 | 来源 | 说明 |
|------|------|------|
| `SERVER_PLUGIN` | 上面的内置插件 | Python 代码注册 |
| `SERVER_MCP` | MCP Server 配置 | 通过 MCP 协议动态拉取 |
| `DEVICE_IOT` | 设备上报的 IoT 能力 | 控制灯/空调等 |
| `DEVICE_MCP` | 设备侧 MCP | 设备本地工具 |
| `MCP_ENDPOINT` | Java 端下发的 MCP 接入点 | 管理后台配置的外部工具 |

### 7.3 三种意图模式

| 配置 type | 工作方式 | 是否 Agent |
|-----------|---------|:----------:|
| `nointent` | 不做意图判断，直接进 LLM | 否（纯聊天） |
| `function_call` | 主 LLM 通过 Function Calling 自主决定 | **是** |
| `intent_llm` | 先用小 LLM 做意图分类，再分发 | **是**（两阶段） |

---

# 第三部分：记忆系统——用"人的记忆"来类比

> 先忘掉所有技术名词，用生活场景来理解短期记忆、长期记忆、对话记录存储。

## 8. 一个生活类比

想象你是一个咖啡店的老板，有一个叫小明的常客。

**场景1：小明正在店里跟你聊天**

你们聊了这些：
- 小明："来杯拿铁"
- 你："好的，大杯还是中杯？"
- 小明："大杯，少糖"
- 你："好的～"

这段对话你记在**脑子里**，不需要写纸上。如果小明等一下又说"加个甜点"，你能想起来他刚才点的是大杯少糖拿铁。

但是——**小明离开店后，你就不记得具体点了什么了**。

**→ 这就是"短期记忆"：当前对话的上下文，聊着的时候有，走了就没了。**

**场景2：小明上周来过一次**

虽然你不记得上周具体聊了啥，但你记得：
- "小明喜欢拿铁"
- "小明要少糖"
- "小明总是点大杯"

这些不是原话，而是你对小明这个人的**总结印象**。下次小明来了，你会说"还是老样子？大杯少糖拿铁？"

**→ 这就是"长期记忆"：你对一个人的印象总结，不是原始聊天记录，是提炼后的关键信息。**

**场景3：店里的收银系统**

每次小明点单，收银系统都会**自动记一笔**：
- 13:05 - 小明 - 大杯拿铁少糖 - ¥32

这个记录是给**店长查账用的**，不是给你记忆用的。你做咖啡不会去翻收银记录。

**→ 这就是"对话记录存储"：完整的聊天日志，存到 MySQL 里，给管理员查看用的。**

### 对应到代码

| 生活类比 | 技术名词 | 代码位置 | 生命周期 |
|---------|---------|---------|---------|
| 当面聊天时脑子里的记忆 | **短期记忆** (Dialogue) | `core/utils/dialogue.py` | 正在聊 → 有；断开 → 没了 |
| 对小明的印象总结 | **长期记忆** (Memory Provider) | `core/providers/memory/` | 永久保存到文件或数据库 |
| 收银系统的流水记录 | **对话记录** (Java 上报) | `core/handle/reportHandle.py` | 永久存在 MySQL 中 |

---

## 9. 短期记忆——"脑子里的对话"

对应源码 `core/utils/dialogue.py`：

```python
class Dialogue:
    def __init__(self):
        self.dialogue: List[Message] = []  # 就是一个 Python 列表！
    
    def put(self, message: Message):
        self.dialogue.append(message)  # 每说一句就往后面加一条
```

**它就是一个列表，非常简单。** 比如一段对话后，列表里长这样：

```python
[
    Message(role="system",    content="你是小智，一个聪明的语音助手..."),
    Message(role="user",      content="今天天气怎么样"),
    Message(role="assistant", content=None, tool_calls=[get_weather(...)]),
    Message(role="tool",      content="北京晴，25°C"),
    Message(role="assistant", content="今天北京天气不错，晴天25度~"),
    Message(role="user",      content="那推荐穿什么"),
    Message(role="assistant", content="建议穿一件薄外套~"),
]
```

**关键点**：

1. **只在内存里**——没有写到任何文件或数据库
2. **设备断开 = 列表销毁**——下次连接是一个全新的空列表
3. **发给 LLM 时全带上**——每次调 LLM，都把这个完整列表发过去，这就是为什么 LLM 能"记住"你前面说了什么

**类比**：就像你跟朋友打电话，打电话的时候你记得前面说了啥，挂了电话就开始忘了。

---

## 10. 长期记忆——"对你的印象笔记"

**核心思想**：每次断开连接前，用 LLM 把当前对话**总结成一段摘要**，保存到文件里。下次连接时，把这段摘要塞到 system prompt 里，LLM 就"记得"了。

### 10.1 保存（断开连接时）

对应源码 `core/providers/memory/mem_local_short/mem_local_short.py`：

```python
async def save_memory(self, msgs, session_id=None):
    # msgs 就是短期记忆（Dialogue 的那个列表）
    
    # 第1步：把对话列表拼成文本
    msgStr = ""
    for msg in msgs:
        if msg.role == "user":
            msgStr += f"User: {msg.content}\n"
        elif msg.role == "assistant":
            msgStr += f"Assistant: {msg.content}\n"
    
    # 第2步：把上次的记忆也附上
    if self.short_memory:
        msgStr += "历史记忆：\n" + self.short_memory
    
    # 第3步：让 LLM 做一次总结！
    result = self.llm.response_no_stream(
        short_term_memory_prompt,  # "你是时空记忆编织者，请总结..."
        msgStr,
    )
    
    # 第4步：存到 YAML 文件
    self.short_memory = result
    self.save_memory_to_file()  # → 写入 data/.memory.yaml
```

**通俗解释**：

1. 把当前对话记录和旧笔记拼在一起
2. 让 LLM **重新总结一遍**（"这个人叫小明，住北京，喜欢跑步"）
3. 把新的总结写入文件

**关键点**：长期记忆不是存原始对话，而是 **LLM 总结后的摘要**。就像你不会记住跟朋友每句话说了啥，但你会记住"他最近在减肥"。

### 10.2 读取（每轮对话时）

```python
async def query_memory(self, query: str) -> str:
    return self.short_memory  # 直接返回整段摘要
```

然后在 `chat()` 里，组装消息时把记忆塞进 system prompt：

```python
messages = self.dialogue.get_llm_dialogue_with_memory(memory_str, ...)
# system prompt 变成：
# "你是小智...
#  <memory>
#  {"身份图谱":{"现用名":"小明"}, "记忆立方":[...]}
#  </memory>"
```

**所以每次用户说话，都会把"长期记忆摘要 + 短期对话历史"一起发给 LLM**。对于 `mem_local_short` 方案，摘要约 1000 字，对现代 LLM 的 128K 上下文窗口来说不到 1%，开销很小。

### 10.3 五种记忆方案对比

| 方案 | 通俗类比 | 查记忆方式 | 适用场景 |
|------|---------|----------|---------|
| `nomem` | 金鱼记忆，啥也不记 | 不查 | 不需要记忆 |
| `mem_report_only` | 自己不记，让秘书（Java）记 | 不查 | Java 端做总结 |
| **`mem_local_short`** | **在笔记本上写总结** | **把整本笔记翻开给 AI 看** | **默认方案，单机部署** |
| `mem0ai` | 用"记忆搜索引擎"（Mem0 云端） | 搜索"跟这个问题相关的记忆" | 需要语义搜索 |
| `powermem` | 用数据库 + 用户画像 | 搜索相关记忆 + 用户偏好 | 企业级部署 |

**`mem_local_short` vs `mem0ai` 最关键的区别**：

- **`mem_local_short`**：不管用户问什么，都把**整本笔记**给 AI 看。笔记不长时没问题。
- **`mem0ai`**：用户说"今天跑步了"，在笔记本里**搜索跟"跑步"相关的记忆**，只给相关片段。笔记很长时更精准。

---

## 11. 对话记录存储——"收银系统的流水"

**核心机制**：Python 端每产生一条消息，都立刻发一个 HTTP 请求给 Java 后端，Java 存到 MySQL。

对应源码 `core/handle/reportHandle.py`：

```python
# 用户说了一句话 → 立刻上报
def enqueue_asr_report(conn, text, audio=None):
    conn.report_queue.put({
        "chatType": 1,         # 1=用户消息
        "content": text,
        "audioBase64": audio,   # 可选：原始语音
    })

# AI 回了一句话 → 立刻上报
def enqueue_tts_report(conn, text, audio=None):
    conn.report_queue.put({
        "chatType": 2,         # 2=AI回复
        "content": text,
    })

# 调了一个工具 → 立刻上报
def enqueue_tool_report(conn, tool_name, tool_input, tool_result=None):
    conn.report_queue.put({
        "chatType": 3,         # 3=工具调用
        "content": f"{tool_name}: {tool_input}",
    })
```

后台线程 `_report_worker` 不断从队列取消息发给 Java。

**三个要点**：

1. **逐条实时上报**：不是攒够了一起发，而是每条消息产生后立刻入队
2. **有开关控制**：`chat_history_conf` 配置项可以关闭上报（=0）、只报文本（=1）、报文本+音频（=2）
3. **Python 不存原始记录**：`Dialogue` 是临时的，断开就没了；Java 的 MySQL 才是永久存储

**类比**：就像你跟朋友视频聊天，微信自动把聊天记录存到了云端。你关掉微信后可能忘了聊了啥，但打开聊天记录还能看到每一句话。

---

# 第四部分：全景总结

## 12. 短期记忆 vs 长期记忆——最终对比

| | 短期记忆 (Dialogue) | 长期记忆 (Memory) |
|---|---|---|
| **通俗类比** | 打电话时脑子里的记忆 | 对这个人的印象笔记 |
| **存什么** | 当前对话的每一句话（原文） | LLM 总结后的摘要（精炼版） |
| **存在哪** | 内存（Python 列表） | 文件/云端/数据库 |
| **什么时候有** | 正在聊天时 | 永久保存 |
| **什么时候没** | 断开连接后 | 一直都有 |
| **谁用它** | LLM 每次被调用时（作为上下文） | LLM 每次被调用时（塞进 system prompt） |
| **多大** | 一直增长，无上限 | 控制在一定长度内（如1000字） |

**它们是怎么配合工作的？**

每次调用 LLM 时，`get_llm_dialogue_with_memory()` 把两者组装在一起：

```
发给 LLM 的 messages = [
    {
        role: "system",
        content: "你是小智...
                  <memory>
                  用户叫小明，住北京，喜欢跑步  ← 长期记忆（印象笔记）
                  </memory>"
    },
    { role: "user",      content: "今天天气怎么样" },     ← ┐
    { role: "assistant", content: "今天北京晴天25度~" },  ← ├ 短期记忆
    { role: "user",      content: "那适合跑步吗" },       ← ┘（当前对话原文）
]
```

**一句话总结**：短期记忆让 AI 知道"你刚才说了啥"，长期记忆让 AI 知道"你是谁"，对话记录让管理员知道"你们聊了啥"。

---

## 13. 设备断开时发生了什么？

```
设备断开连接
    │
    ▼
_save_and_close()
    │
    ├─── ① 保存长期记忆
    │    把 Dialogue（短期记忆）交给 LLM 做总结
    │    → "用户叫小明，今天查了天气和跑步"
    │    → 写入 data/.memory.yaml（覆盖旧的总结）
    │
    ├─── ② 生成会话标题
    │    POST 请求给 Java
    │    → Java 调 LLM 生成 ≤15字的标题
    │
    └─── ③ 销毁一切
         Dialogue 销毁 → 短期记忆没了
         ConnectionHandler 销毁 → 整个连接的状态没了
         
         但留下了：
         ✓ .memory.yaml 里有新的总结（下次连接用）
         ✓ MySQL 里有每一句对话的记录（管理员查看用）
```

---

## 14. 完整调用链

```
ESP32 设备
  │ ① WebSocket 音频流
  ▼
Python Server (xiaozhi-server)
  │ ② Opus 解码 → PCM
  │ ③ VAD 断句检测
  │ ④ ASR 语音识别 → 文字
  │ ⑤ 意图判断 (nointent / function_call / intent_llm)
  │ ⑥ chat(query, depth=0)
  │    ├─ 组装 messages (系统提示词 + 历史 + 记忆 + 工具描述)
  │    ├─ LLM 流式调用 (DeepSeek/通义/Ollama/...)
  │    │   ├─ 若有 <think>...</think> → 过滤掉（CoT 兼容）
  │    │   ├─ 若返回 tool_calls → 执行工具
  │    │   │   ├─ Action.REQLLM → 结果写回对话 → chat(None, depth+1) 递归
  │    │   │   └─ Action.RESPONSE → 直接 TTS 回复
  │    │   └─ 若纯文本 → 逐 token 推给 TTS
  │    └─ 最多递归 5 层，超出强制回答
  │ ⑦ TTS 合成语音
  │ ⑧ Opus 编码 → WebSocket 回传
  ▼
ESP32 设备播放回答
  │
  │ ⑨ Python → Java: POST /agent/chat-history/report (异步上报)
  ▼
Java API (manager-api)
  │ ⑩ 存入 MySQL (聊天记录 + 音频)
  │ ⑪ 异步调小 LLM 生成总结/标题
  ▼
完成
```

---

## 15. Python 端 vs Java 端职责分工

| 维度 | xiaozhi-server (Python) | manager-api (Java) |
|------|------------------------|---------------------|
| **核心角色** | 实时对话引擎（"AI 大脑"） | 管理后台（"配置中心"） |
| **LLM 调用频率** | 极高（每次用户说话） | 极低（对话结束后总结一次） |
| **LLM 用途** | 实时对话、意图识别、记忆总结 | 聊天记录总结、会话标题生成 |
| **有 Agent 能力** | **有**：Function Calling + 递归 + MCP | 没有 |
| **有 CoT 处理** | **有**：过滤 `<think>` 标签 | 没有 |
| **面向谁** | 面向设备（WebSocket 实时通信） | 面向管理前端（HTTP REST） |

---

## 16. drawio 图文件索引

| 文件 | 内容 |
|------|------|
| [15-01-agent-architecture.drawio](15-01-agent-architecture.drawio) | xiaozhi-server Agent 架构全景图 |

> **查看方式**：安装 VS Code 插件 **Draw.io Integration**（`hediet.vscode-drawio`），双击 `.drawio` 文件即可查看。

---

> **参考源码路径**：
> - `xiaozhi-server/core/connection.py` — `chat()` Agent 核心 + `_handle_function_result` 递归
> - `xiaozhi-server/core/providers/llm/openai/openai.py` — LLM 调用 + `<think>` 过滤
> - `xiaozhi-server/core/providers/llm/system_prompt.py` — `<tool_call>` 文本格式的 Prompt 模板
> - `xiaozhi-server/core/utils/dialogue.py` — 对话历史管理 + 记忆注入
> - `xiaozhi-server/core/providers/memory/mem_local_short/mem_local_short.py` — 本地记忆实现
> - `xiaozhi-server/core/providers/memory/mem0ai/mem0ai.py` — Mem0 向量记忆
> - `xiaozhi-server/core/providers/memory/powermem/powermem.py` — PowerMem 记忆
> - `xiaozhi-server/core/handle/reportHandle.py` — 上报逻辑 + `enqueue_*` 函数
> - `xiaozhi-server/plugins_func/register.py` — `Action` 枚举、工具注册机制
> - `xiaozhi-server/plugins_func/functions/*.py` — 内置工具实现
