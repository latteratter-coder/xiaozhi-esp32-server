# 移动端智控台 (manager-mobile) 内部业务流程

## 1. 模块概览

`manager-mobile` 是面向手机/小程序的轻量智控台，基于 uni-app + Vue 3 + TypeScript 构建。

```
manager-mobile/src/
├── main.ts                 # 应用入口
├── App.vue                 # 根组件(TabBar设置)
├── pages.json              # 页面路由配置
├── manifest.json           # 应用清单(小智)
├── router/
│   └── interceptor.ts      # 路由拦截器
├── api/
│   ├── auth.ts             # 认证相关API
│   ├── agent/agent.ts      # 智能体API
│   ├── device/device.ts    # 设备API
│   ├── chat-history/       # 聊天记录API
│   └── voiceprint/         # 声纹API
├── http/request/
│   └── alova.ts            # Alova HTTP客户端
├── store/                  # Pinia 状态管理
├── pages/                  # 页面组件
├── components/             # 公共组件
├── i18n/                   # 国际化
├── hooks/                  # 组合式函数
└── layouts/                # 布局组件
```

## 2. 应用启动流程

```
main.ts
 ├── createSSRApp(App)
 ├── 注册 Pinia store
 ├── routeInterceptor()       # 路由拦截器
 ├── VueQueryPlugin
 └── initI18n()               # 国际化初始化

App.vue onLaunch()
 ├── usePageAuth()            # 页面认证检查
 └── t('tabBar.*') → setTabBarItem()  # 国际化TabBar文案
```

## 3. 页面结构

### 3.1 TabBar 页面 (三个主标签)

| Tab | 页面 | 功能 |
|-----|------|------|
| 首页 | `pages/index/index` | 智能体列表、创建、管理 |
| 配网 | `pages/device-config/index` | WiFi/超声波配网 |
| 设置 | `pages/settings/index` | 语言、服务器地址、退出 |

### 3.2 普通页面

| 页面路径 | 功能 |
|----------|------|
| `pages/login/index` | 登录 |
| `pages/register/index` | 注册 |
| `pages/forgot-password/index` | 找回密码 |
| `pages/agent/index` | 智能体详情入口 |
| `pages/agent/edit.vue` | 编辑智能体(名称/模型/音色) |
| `pages/device/index` | 设备管理(绑定/解绑) |
| `pages/chat-history/*` | 会话列表/聊天详情 |
| `pages/voiceprint/index` | 声纹管理 |

## 4. HTTP 请求层 (Alova)

### 4.1 客户端配置

```typescript
// http/request/alova.ts
createAlova({
  baseURL: getEnvBaseUrl(),   // 支持 server_base_url_override 覆盖
  requestAdapter: adapterUniapp,
  beforeRequest(method) {
    headers['Accept-language'] = uni.getStorageSync('app_language')
    headers['Authorization'] = `Bearer ${authInfo.token}`
    // HTTPS页面禁止HTTP请求
  },
  responded: {
    // statusCode !== 200 → toast提示
    // code !== 0 + 401 → 清token → 跳登录
    // 成功 → 返回 data 字段(业务数据unwrap)
  }
})
```

### 4.2 认证机制

```
非 meta.ignoreAuth 的请求:
 ├── uni.getStorageSync('token') 解析
 ├── 无 token → reLaunch('/pages/login/index')
 └── 有 token → Authorization: Bearer {token}
```

## 5. 状态管理 (Pinia)

| Store | State | 持久化 | 用途 |
|-------|-------|--------|------|
| `user` | userInfo, token | userInfo key | 用户登录态 |
| `config` | pubConfig | config key | 公共配置(注册开关/SM2公钥) |
| `lang` | currentLang | app_language | 当前语言 |
| `provider` | providerList | provider key | 供应器列表缓存 |
| `speedPitch` | volume, speed, pitch | speedPitch key | TTS参数 |
| `plugin` | pluginList | 不持久化 | 当前编辑智能体的插件 |

## 6. 核心页面业务流程

### 6.1 登录流程 (pages/login/index)

```
打开页面
 ├── getPublicConfig() → GET /user/pub-config
 │   └── 获取SM2公钥、注册开关
 └── getCaptcha() → GET /user/captcha

点击登录
 ├── sm2Encrypt(password, pubKey)
 ├── POST /user/login
 ├── uni.setStorageSync('token', data)
 ├── GET /user/info → userStore.setUserInfo()
 └── uni.reLaunch('/pages/index/index')
```

### 6.2 首页 (pages/index/index)

```
进入页面
 └── z-paging 组件加载
     └── GET /agent/list → 智能体列表

交互:
 ├── "新建智能体" → POST /agent → 刷新列表
 ├── 点击智能体 → /pages/agent/index?agentId={id}
 ├── 左滑删除 → DELETE /agent/{id}
 └── 下拉刷新 / 上拉加载更多
```

### 6.3 智能体详情 (pages/agent/index)

```
进入页面
 └── GET /agent/{id} → 智能体详情

子页面导航:
 ├── 编辑 → /pages/agent/edit
 │   ├── 名称、系统提示词
 │   ├── 模型选择 (ASR/LLM/TTS/VAD/Memory/Intent)
 │   ├── 音色选择 → GET /models/{ttsModelId}/voices
 │   ├── TTS参数 (语速/音量/音调)
 │   └── PUT /agent/{id}
 ├── 插件管理 → GET /models/provider/plugin/names
 ├── MCP配置
 ├── 标签管理
 └── 设备管理 → /pages/device/index
```

### 6.4 设备管理 (pages/device/index)

```
进入页面
 └── GET /device/bind/{agentId} → 设备列表

操作:
 ├── "绑定" → POST /device/bind/{agentId}/{code}
 ├── "解绑" → POST /device/unbind
 ├── "手动添加" → POST /device/manual-add
 └── "更新" → PUT /device/update/{id}
```

### 6.5 聊天记录 (pages/chat-history/)

```
会话列表页:
 └── GET /agent/{id}/sessions → 会话分页

聊天详情页:
 ├── GET /agent/{id}/chat-history/{sessionId}
 └── 音频播放 → GET /agent/audio/{audioId}
```

### 6.6 声纹管理 (pages/voiceprint/index)

```
进入页面
 └── GET /agent/voice-print/list/{agentId}

操作:
 ├── "添加声纹" → POST /agent/voice-print
 ├── "编辑" → PUT /agent/voice-print
 └── "删除" → DELETE /agent/voice-print/{id}
```

### 6.7 设置页 (pages/settings/index)

```
├── 语言切换 → langStore.setLang() + uni.setStorageSync('app_language')
├── 服务器地址覆盖 → uni.setStorageSync('server_base_url_override', url)
└── 退出登录 → userStore.logout() → uni.reLaunch('/pages/login/index')
```

## 7. 路由拦截

```typescript
// router/interceptor.ts
对 navigateTo / reLaunch / redirectTo / switchTab 注册拦截:
 ├── 目标页在 needLoginPages 中
 ├── 且用户未登录 (userStore.userInfo.username 为空)
 └── 则 navigateTo('/pages/login/index?redirect=...')

注意: 当前 pages.json 未声明 needLogin 属性，
实际登录拦截主要依赖 HTTP 层的 token 检查。
```

## 8. API 模块与后端对应

| 前端 API 文件 | 后端模块 | 主要接口 |
|---------------|----------|----------|
| `api/auth.ts` | security | `/user/login`, `/user/info`, `/user/pub-config`, `/user/register` |
| `api/agent/agent.ts` | agent, model | `/agent/*`, `/models/names`, `/models/{id}/voices` |
| `api/device/device.ts` | device | `/device/bind/*`, `/device/unbind`, `/device/manual-add` |
| `api/chat-history/` | agent | `/agent/{id}/sessions`, `/agent/{id}/chat-history/*` |
| `api/voiceprint/` | agent | `/agent/voice-print/*` |

## 9. 国际化

与 Web 端对齐，支持：简体中文、繁体中文、英语、德语、越南语、巴西葡萄牙语。

初始语言来自 `useLangStore`，App.vue 根据 `t('tabBar.*')` 动态设置 TabBar 文案。

## 10. 与 Web 端的差异

| 维度 | manager-web | manager-mobile |
|------|-------------|----------------|
| 框架 | Vue 2 + Element UI | Vue 3 + uni-app + wot-design |
| HTTP | flyio | alova |
| 状态 | Vuex | Pinia (持久化) |
| 路由 | Vue Router | uni 路由 + 拦截器 |
| 功能范围 | 完整管理(含超管功能) | 轻量管理(智能体/设备/聊天/声纹) |
| 平台 | PC浏览器 | App/小程序/H5 |
