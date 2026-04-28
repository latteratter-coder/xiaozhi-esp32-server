# 前端智控台 (manager-web) 内部业务流程

## 1. 模块概览

`manager-web` 是面向 PC/大屏的智控台管理后台前端，基于 Vue 2 + Element UI 构建。

```
manager-web/src/
├── main.js                # 应用入口
├── App.vue                # 根组件
├── router/index.js        # 路由定义 + 导航守卫
├── store/index.js         # Vuex 状态管理
├── i18n/                  # 国际化(6种语言)
├── apis/
│   ├── api.js             # API聚合出口
│   ├── httpRequest.js     # flyio HTTP封装
│   └── module/            # 各模块API定义
│       ├── user.js        # 用户/登录
│       ├── agent.js       # 智能体
│       ├── device.js      # 设备
│       ├── model.js       # 模型
│       ├── admin.js       # 管理员/参数
│       ├── dict.js        # 字典
│       ├── ota.js         # OTA固件
│       ├── knowledgeBase.js # 知识库
│       ├── voiceClone.js  # 语音克隆
│       └── voiceResource.js # 语音资源
├── views/                 # 页面组件
└── components/            # 公共组件
```

## 2. 应用启动流程

```
main.js
 ├── Vue.use(ElementUI)                  # 注册Element UI
 ├── Vue.prototype.$eventBus = new Vue() # 事件总线(跨组件通信)
 ├── 注册 router, store, i18n
 └── new Vue({...}).$mount('#app')

App.vue created()
 ├── 从 localStorage 恢复 userInfo, pubConfig
 ├── 移动端检测 → VUE_APP_H5_URL 跳转
 └── CDN模式: Alt+C 缓存查看器, SW状态
```

## 3. 路由架构

### 3.1 路由表

| 路径 | 名称 | 页面 | 功能 |
|------|------|------|------|
| `/` | login | login.vue | 登录页 |
| `/login` | login | login.vue | 登录页 |
| `/register` | register | register.vue | 注册页 |
| `/retrieve-password` | - | retrievePassword.vue | 找回密码 |
| `/home` | home | home.vue | 首页(智能体列表) |
| `/role-config` | RoleConfig | roleConfig.vue | 智能体配置 |
| `/device-management` | DeviceManagement | DeviceManagement.vue | 设备管理 |
| `/user-management` | UserManagement | UserManagement.vue | 用户管理(超管) |
| `/model-config` | ModelConfig | ModelConfig.vue | 模型配置(超管) |
| `/knowledge-base-management` | KnowledgeBaseManagement | KnowledgeBaseManagement.vue | 知识库 |
| `/knowledge-file-upload` | KnowledgeFileUpload | KnowledgeFileUpload.vue | 文件上传 |
| `/params-management` | - | ParamsManagement.vue | 系统参数 |
| `/ota-management` | - | OtaManagement.vue | OTA固件管理 |
| `/dict-management` | - | DictManagement.vue | 字典管理 |
| `/server-side-management` | - | ServerSideManager.vue | 服务器管理 |
| `/provider-management` | - | ProviderManagement.vue | 模型供应器 |
| `/agent-template-management` | - | AgentTemplateManagement.vue | 智能体模板 |
| `/template-quick-config` | - | TemplateQuickConfig.vue | 模板快速配置 |
| `/feature-management` | - | FeatureManagement.vue | 功能配置 |
| `/voice-resource-management` | - | VoiceResourceManagement.vue | 语音资源 |
| `/voice-clone-management` | - | VoiceCloneManagement.vue | 语音克隆 |
| `/voice-print` | - | VoicePrint.vue | 声纹管理 |

### 3.2 导航守卫

```javascript
// 白名单式保护，仅以下路由需要token
const protectedRoutes = ['home', 'RoleConfig', 'DeviceManagement',
  'UserManagement', 'ModelConfig', 'KnowledgeBaseManagement', 'KnowledgeFileUpload']

router.beforeEach((to, from, next) => {
  if (protectedRoutes.includes(to.name)) {
    const token = localStorage.getItem('token')
    if (!token) → 跳转登录页
  }
  next()
})
```

> 注意：params/ota/dict/server等管理页不在protectedRoutes中，实际依赖接口401拦截。

## 4. 状态管理 (Vuex)

```javascript
// store/index.js - 单文件，无modules
state: {
  token: null,        // 登录token (JSON字符串)
  userInfo: null,     // 用户信息
  pubConfig: null     // 公共配置(注册开关/SM2公钥/菜单等)
}

mutations: {
  setToken(state, token)      // localStorage.setItem('token', token)
  setUserInfo(state, info)    // localStorage.setItem('userInfo', JSON.stringify(info))
  setPubConfig(state, config) // localStorage.setItem('pubConfig', JSON.stringify(config))
  clearAuth(state)            // 清除所有登录态
}

actions: {
  logout() → clearAuth + 跳转登录页
  fetchPubConfig() → Api.user.getPubConfig()
}
```

## 5. HTTP 请求层

### 5.1 请求封装 (httpRequest.js)

基于 **flyio** 库，链式调用模式：

```javascript
sendRequest()
  .url('/user/login')
  .method('POST')
  .data({username, password})
  .success(callback)
  .send()
```

### 5.2 统一请求头

```javascript
headers: {
  'Accept-Language': i18n.locale,    // 国际化
  'Authorization': 'Bearer ' + JSON.parse(store.getters.getToken).token  // 已登录时
}
```

### 5.3 响应处理

```
成功: code === 0 / 'success' / undefined
401: clearAuth + 跳转登录页
其它错误: Element UI Message 提示后端 msg
```

## 6. 核心页面业务流程

### 6.1 登录页 (login.vue)

```
打开页面
 ├── fetchPubConfig() → GET /user/pub-config
 │   └── 获取 SM2公钥、注册开关、备案号等
 └── getCaptcha() → GET /user/captcha?uuid={uuid}

点击登录
 ├── SM2加密密码 (使用pubConfig中的公钥)
 ├── POST /user/login {username, password(加密), uuid, captcha}
 ├── store.commit('setToken', JSON.stringify(data))
 ├── GET /user/info → store.commit('setUserInfo', data)
 └── router.push('/home')
```

### 6.2 首页 (home.vue)

```
进入页面
 ├── 加载功能开关 (featureManager)
 │   └── 读取 pubConfig.systemWebMenu.features
 └── GET /agent/list → 智能体列表

交互操作:
 ├── "配置角色" → /role-config?agentId={id}
 ├── "设备管理" → /device-management?agentId={id}
 ├── "删除" → DELETE /agent/{id}
 ├── "搜索" → GET /agent/list?keyword=...
 └── "聊天记录" → 弹出 ChatHistoryDialog
```

### 6.3 智能体配置 (roleConfig.vue)

```
进入页面
 ├── GET /agent/{id} → 智能体详情
 ├── GET /models/names → 可选模型列表
 └── 渲染配置表单

保存配置
 └── PUT /agent/{id}
     ├── agentName, systemPrompt
     ├── 各模型ID (asr/vad/llm/tts/memory/intent)
     ├── TTS音色和参数
     ├── 插件配置
     └── 标签/MCP/上下文提供者
```

### 6.4 设备管理 (DeviceManagement.vue)

```
进入页面
 ├── GET /device/bind/{agentId} → 已绑定设备列表
 └── GET /admin/dict/data/type/FIRMWARE_TYPE → 固件类型字典

操作:
 ├── "绑定设备" → POST /device/bind/{agentId}/{code}
 ├── "解绑" → POST /device/unbind
 ├── "手动添加" → POST /device/manual-add
 └── "在线状态" → POST /device/bind/{agentId} (转发MQTT查询)
```

### 6.5 模型配置 (ModelConfig.vue)

```
进入页面
 └── GET /models/list → 模型分页列表

操作:
 ├── "新增" → POST /models/{type}/{provider}
 ├── "编辑" → PUT /models/{type}/{provider}/{id}
 ├── "删除" → DELETE /models/{id}
 ├── "启用/禁用" → PUT /models/enable/{id}/{status}
 └── "设为默认" → PUT /models/default/{id}
```

### 6.6 知识库管理 (KnowledgeBaseManagement.vue)

```
进入页面
 └── GET /datasets → 知识库分页列表

操作:
 ├── "创建" → POST /datasets
 ├── "编辑" → PUT /datasets/{id}
 ├── "删除" → DELETE /datasets/{id}
 ├── "文档管理" → 跳转 /knowledge-file-upload
 └── "检索测试" → POST /datasets/{id}/retrieval-test
```

### 6.7 OTA管理 (OtaManagement.vue)

```
进入页面
 └── GET /otaMag → 固件分页列表

操作:
 ├── "上传固件" → POST /otaMag/upload (multipart)
 ├── "编辑" → PUT /otaMag/{id}
 ├── "删除" → DELETE /otaMag/{id}
 └── "下载链接" → GET /otaMag/getDownloadUrl/{id}
```

### 6.8 服务器管理 (ServerSideManager.vue)

```
进入页面
 └── GET /admin/server/server-list → 实例列表

操作:
 └── "发送指令" → POST /admin/server/emit-action
     └── 向目标 xiaozhi-server 发 WebSocket 控制消息
```

### 6.9 功能配置 (FeatureManagement.vue)

通过修改系统参数 `system-web.menu` (id=600) 控制各功能模块的显示/隐藏。

## 7. 导航结构

```
HeaderBar 导航栏
├── 智能体管理
│   ├── /home (首页/智能体列表)
│   ├── /role-config (智能体配置)
│   └── /device-management (设备管理)
├── 音色克隆 (featureStatus.voiceClone)
│   ├── /voice-resource-management
│   └── /voice-clone-management
├── 知识库 (featureStatus.knowledgeBase)
│   └── /knowledge-base-management
├── 模型配置 (仅超管)
│   └── /model-config
├── 更多 (仅超管)
│   ├── /params-management (系统参数)
│   ├── /user-management (用户管理)
│   ├── /ota-management (OTA管理)
│   ├── /dict-management (字典管理)
│   ├── /provider-management (供应器管理)
│   ├── /agent-template-management (智能体模板)
│   ├── /server-side-management (服务器管理)
│   └── /feature-management (功能配置)
└── 用户菜单
    ├── 修改密码
    └── 退出登录
```

## 8. 国际化

支持6种语言：简体中文、繁体中文、英语、德语、越南语、巴西葡萄牙语。

默认语言优先级：`localStorage.userLanguage` → 浏览器语言 → 中文。

语言切换通过 `$eventBus` 发 `languageChanged` 事件通知全局组件更新。

## 9. 权限控制

| 层级 | 机制 |
|------|------|
| 路由级 | protectedRoutes 白名单 + token检查 |
| 接口级 | Bearer token → 401时清除登录态 |
| 功能级 | userInfo.superAdmin 控制菜单可见性 |
| 特性级 | pubConfig.systemWebMenu.features 控制功能开关 |
