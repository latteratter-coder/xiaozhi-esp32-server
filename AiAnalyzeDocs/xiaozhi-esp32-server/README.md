# 小智 ESP32 服务端 - 代码架构分析文档

> 本文档由代码静态分析自动生成，对 xiaozhi-esp32-server 项目进行了全面的架构梳理和业务流程分析。

## 文档目录

### 基础架构（01-08）

| 序号  | 文档                                                     | 内容说明                                                              |
| --- | ------------------------------------------------------ | ----------------------------------------------------------------- |
| 01  | [项目架构总览](01-项目架构总览.md)                                 | 项目简介、整体架构图、四大核心模块、技术栈、部署架构、目录结构、配置体系                              |
| 02  | [Python核心服务业务流程](02-Python核心服务(xiaozhi-server)业务流程.md) | 启动流程、配置加载、WebSocket服务、连接状态机、消息处理、HTTP服务、Provider体系、工具系统、插件系统、认证体系 |
| 03  | [Java管理API业务流程](03-Java管理API(manager-api)业务流程.md)      | 公共基础设施、安全模块(Shiro)、设备模块(OTA/绑定)、智能体模块、模型模块、知识库模块(RAG)、配置下发、系统管理   |
| 04  | [前端智控台业务流程](04-前端智控台(manager-web)业务流程.md)              | 应用启动、路由架构、状态管理、HTTP封装、核心页面流程、导航结构、国际化、权限控制                        |
| 05  | [移动端智控台业务流程](05-移动端智控台(manager-mobile)业务流程.md)         | 应用入口、页面结构、HTTP层(Alova)、Pinia状态、核心页面流程、路由拦截                        |
| 06  | [服务间业务流程](06-服务间业务流程.md)                               | 通信矩阵、设备上电→绑定→对话全链路、配置拉取、热更新、聊天上报、知识库→RAGFlow                      |
| 07  | [API接口完整清单](07-API接口完整清单.md)                           | 全部REST API端点（含鉴权方式）、WebSocket消息类型                                 |
| 08  | [数据模型与存储](08-数据模型与存储.md)                               | 全部数据表结构、表关系图、Redis缓存策略                                            |

### 流程与机制（09-12）

| 序号 | 文档 | 内容说明 |
|------|------|---------|
| 09 | [三条关键流程深入解析](09-三条关键流程深入解析.md) | 设备绑定、配置拉取、语音对话三条核心流程的完整链路 |
| 10 | [OAuth2用户Token认证完整流程](10-OAuth2用户Token认证完整流程.md) | OAuth2 认证机制、Token 生命周期、Shiro 安全架构 |
| 11 | [manager-api源码目录功能详解](11-manager-api源码目录功能详解(小白入门).md) | 源码目录逐文件速查表，面向首次阅读者 |
| 12 | [Spring Boot入门与项目架构原理](12-Spring%20Boot入门与项目架构原理.md) | IoC/DI、自动配置、AOP、全局异常处理、XssFilter 等 Spring Boot 核心概念 |

### 深度分析（13-15）

| 序号 | 文档 | 内容说明 |
|------|------|---------|
| 13 | [xiaozhi-server Agent与记忆系统全解析](13-xiaozhi-server-Agent与记忆系统全解析.md) | **Python 端核心**：Agent 概念、CoT、chat() 递归源码逐行解读、短期/长期记忆、对话记录存储、完整调用链 |
| 14 | [agent/llm/timbre/voiceclone模块原理详解](14-agent-llm-timbre-voiceclone模块原理详解.md) | **Java 端模块**：智能体CRUD、聊天上报、LLM统一封装、音色管理、声音克隆、MCP接入 |
| 15 | [百万设备部署架构方案](15-百万设备部署架构方案(深度调研).md) | 扩展至百万设备的架构方案：接入层/计算层/存储层/消息层/安全/可观测/分阶段路线图 |

### 架构图

`diagrams/` 目录下的 `.drawio` 文件需安装 VS Code 插件 **Draw.io Integration**（`hediet.vscode-drawio`）查看：

| 文件                                | 内容                   |
| --------------------------------- | -------------------- |
| `14-01-module-overview.drawio`    | Java 端四模块关系总览        |
| `14-02-agent-create-flow.drawio`  | 智能体创建流程              |
| `14-03-chat-report-flow.drawio`   | 聊天记录上报流程             |
| `14-04-voice-clone-flow.drawio`   | 声音克隆完整流程             |
| `14-05-llm-timbre-system.drawio`  | LLM 调用 + 音色系统        |
| `14-06-data-flow-overview.drawio` | 四模块数据流全景             |
| `15-01-agent-architecture.drawio` | Python 端 Agent 架构全景图 |

## 项目架构一句话总结

**xiaozhi-esp32-server** 采用 **Python + Java + Vue** 三语言架构：
- **Python 核心服务** (8000/8003) 负责设备 WebSocket 通信和 AI 推理管线（ASR→LLM→TTS）
- **Java 管理 API** (8002) 负责配置管理、用户/设备/智能体CRUD、知识库、OTA
- **Vue 前端** 提供 PC 和移动端的智控台管理界面

三者通过 **HTTP REST + WebSocket** 协同，共享 **`server.secret`** 进行服务间认证。

## 关键技术选型

| 维度   | 选型                                |
| ---- | --------------------------------- |
| 设备通信 | WebSocket (Opus音频) + HTTP (OTA)   |
| AI推理 | 多厂商适配器模式 (OpenAI/阿里/讯飞/腾讯/百度/火山等) |
| 安全   | Apache Shiro + HMAC-SHA256 + SM2  |
| 存储   | MySQL + Redis                     |
| 知识库  | RAGFlow (HTTP API适配)              |
| 部署   | Docker + Docker Compose           |
