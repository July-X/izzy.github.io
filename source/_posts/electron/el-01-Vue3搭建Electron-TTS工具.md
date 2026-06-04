---
title: Electron 实战：用 Vue 3 搭建桌面 TTS 工具的技术选型
date: 2026-06-04
categories:
  - [Electron实战]
tags:
  - Electron
  - Vue 3
  - electron-egg
  - Naive UI
  - TTS
  - 技术选型
  - 问答
top_img: linear-gradient(135deg, #4a00e0 0%, #8e2de2 100%)
---

<!-- more -->

> 这是「Electron 实战」系列的第一篇。我的项目 **MiMo TTS Studio** 是一个桌面配音工作台，集成小米 MiMo TTS 引擎，支持多角色音色管理、批量并发生成、波形可视化。本文从"为什么做这个工具"出发，聊聊技术选型的决策过程。

---

## Q1：为什么选择 Electron 做一个桌面 TTS 工具？

**A：** 起因很实际——我需要给游戏项目（godot-game-one）的 NPC 配音。

```
需求链路：
  游戏需要 NPC 对话语音 → 需要大量台词的音频 → 
  手动逐条去网页端合成太慢 → 需要批量工具 → 
  需要管理多角色/多音色 → 需要可视化波形确认效果 → 
  最终产物 = 一个桌面应用
```

为什么不做成 Web 应用？

| 方案 | 优点 | 缺点 |
|------|------|------|
| 纯 Web | 无需安装、跨平台 | 无法直接操作本地文件系统、无法调用系统原生 API |
| Tauri | 包体更小、内存占用低 | Rust 生态对 TTS/音频处理库支持有限 |
| **Electron** | Node.js 生态丰富、文件 I/O 原生支持、前端技术栈无缝复用 | 包体较大（~150MB） |
| Flutter Desktop | 跨平台一致性好 | 音频生态不如 JS 成熟 |

最终选 Electron 的核心原因：**TTS 引擎对接 + 本地文件管理 + 前端快速迭代**，这三点恰好是 Electron 的强区。

> 选型原则：工具是为需求服务的。如果只是简单表单，Tauri 更合适；但涉及到文件系统深度操作和第三方 SDK 对接，Electron 的 Node.js 后端是降维打击。

---

## Q2：项目整体架构是怎样的？主进程和渲染进程怎么分工？

**A：** 采用经典的 **主进程 / 渲染进程分离架构**，通过 IPC 通信：

```
┌──────────────────────────────────────────────┐
│                  主进程 (Main)                 │
│  ┌─────────┐ ┌──────────┐ ┌──────────────┐   │
│  │ ee-core  │ │ Controller│ │ Service 层   │   │
│  │ 框架驱动 │ │ (IPC路由) │ │ (业务逻辑)    │   │
│  └────┬─────┘ └─────┬────┘ └──────┬───────┘   │
│       └──────────────┼──────────────┘          │
│                      ▼                         │
│              Preload Bridge                    │
│           (安全上下文隔离)                       │
├──────────────────────────────────────────────┤
│               渲染进程 (Renderer)               │
│  ┌──────────┐ ┌────────┐ ┌─────────────────┐  │
│  │ Vue 3    │ │ Pinia  │ │ 业务组件层        │  │
│  │ + Router │ │ Store  │ │ (视图/交互)      │  │
│  └──────────┘ └────────┘ └─────────────────┘  │
│         Naive UI + Less 主题                   │
└──────────────────────────────────────────────┘
```

### 为什么用 electron-egg 而不是裸写？

```javascript
// electron/main.js — 整个入口就这几行
const { ElectronEgg } = require('ee-core');
const app = new ElectronEgg();
app.register("ready", life.ready);
app.register("window-ready", life.windowReady);
app.run();
```

`electron-egg`（简称 `ee-core`）帮我省去了大量样板代码：

| ee-core 提供的能力 | 自己写要多少行 |
|-------------------|--------------|
| MVC 自动路由映射（Controller → IPC Channel） | ~200 行 |
| 窗口生命周期管理 | ~100 行 |
| 开发环境热更新 | ~80 行 |
| 生产环境打包配置 | electron-builder 集成 |
| 日志系统 | winston 封装 |

对于个人项目来说，**框架带来的开发效率提升远大于学习成本**。

### 关键设计：Controller / Service 分层

```
electron/
├── controller/          # IPC 入口层（参数校验、权限检查）
│   ├── tts.js          # ipcMain.handle('tts:generate', ...)
│   ├── file.js         # ipcMain.handle('file:read', ...)
│   └── config.js       # ipcMain.handle('config:get', ...)
└── service/            # 核心业务逻辑（可独立测试）
    ├── tts.js          # TTSService 类：Provider 抽象、缓存、重试
    └── config.js       # 配置持久化
```

这种分层的好处：**Service 层完全不依赖 Electron API**，理论上可以在 Node.js 脚本中独立运行和测试。

---

## Q3：渲染进程的技术栈怎么选的？为什么是 Vue 3 + Vite + Naive UI？

**A：** 这个组合是经过实际使用验证的：

### Vue 3 vs React

| 因素 | 选择结果 |
|------|---------|
| 我自己的熟练度 | Vue 3 更顺手 ✅ |
| 组合式 API | `setup()` + `<script setup>` 写组件非常舒服 |
| Pinia 状态管理 | 比 Redux/Vuex 3 简洁太多 |
| TypeScript 支持 | Vue 3 的类型推导已经很好了 |

### Vite

不用多说——秒级启动、HMR 即时响应。对比 webpack 的开发体验，Vite 是降维打击。

### Naive UI（关键决策）

这是最值得说的一个选型。桌面应用的 UI 库选择面其实不宽：

| UI 库 | 暗色主题 | Tree-shake | 中文文档 | 自定义程度 |
|-------|---------|------------|---------|-----------|
| Element Plus | ✅ | ✅ | ✅ 好 | 中等 |
| Ant Design Vue | ✅ | ✅ | ⚠️ 一般 | 较高 |
| **Naive UI** | **✅ 原生暗色** | **✅ 极致** | **✅ 优秀** | **极高** |
| Arco Design Vue | ✅ | ✅ | ✅ 好 | 高 |

Naive UI 最终胜出的原因：

1. **原生暗色主题**——桌面工具类应用暗色是刚需，Naive UI 切换暗色只需一行：
   ```vue
   <n-config-provider :theme="darkTheme">
     <App />
   </n-config-provider>
   ```
2. **极致 Tree-shake**——按需引入，打包体积比 Element Plus 小约 **30%**
3. **主题定制灵活**——CSS 变量覆盖，改个颜色变量就行
4. **TypeScript 友好**——完整类型定义，写代码时有完整的智能提示

```jsonc
// frontend/package.json 核心依赖
{
  "dependencies": {
    "vue": "^3.5.12",           // Vue 3.5 Composition API
    "vue-router": "^4.4.5",      // SPA 路由
    "pinia": "^3.0.4",           // 状态管理
    "naive-ui": "^2.44.1",       // UI 组件库
    "@vicons/antd": "^0.12.0"    // 图标集（Ant Design 风格）
  },
  "devDependencies": {
    "vite": "^5.4.11",            // 构建工具
    "less": "^4.2.1",             // CSS 预处理器
    "vitest": "^3.0.4",           // 单元测试
    "@vue/test-utils": "^2.4.6"   // 测试工具
  }
}
```

---

## Q4：主进程和渲染进程之间怎么通信的？IPC 设计有什么讲究？

**A：** 这是 Electron 项目最容易踩坑的地方。我的做法是 **统一封装一层**。

### IPC 频道命名规范

```javascript
// frontend/src/api/index.js — 所有 IPC 频道集中定义
export const IPC = {
  // TTS 相关
  TTS_GENERATE: 'tts:generate',
  TTS_GET_VOICES: 'tts:getVoices',
  TTS_HEALTH_CHECK: 'tts:healthCheck',

  // 文件操作
  FILE_SHOW_OPEN_DIALOG: 'file:showOpenDialog',
  FILE_READ_TEXT: 'file:readText',
  FILE_WRITE_TEXT: 'file:writeText',
  FILE_SCAN_DIR: 'file:scanDir',

  // 配置
  CONFIG_GET: 'config:get',
  CONFIG_SAVE: 'config:save',
};
```

**命名规则：`模块:动作`**，避免全局搜索时出现歧义。

### 渲染进程统一封装

```javascript
// frontend/src/utils/ipcRenderer.js — IPC 调用封装
const isElectron = !!window.electronAPI;

export async function invoke(channel, ...args) {
  if (!isElectron) {
    console.warn(`[IPC Mock] ${channel}`, args);
    return getMockData(channel); // 浏览器环境下返回 Mock 数据
  }
  return window.electronAPI.invoke(channel, ...args);
}

// 带超时的安全调用（防止主进程卡死导致渲染进程挂起）
export async function safeInvoke(channel, args, timeout = 30000) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), timeout);
  try {
    const result = await invoke(channel, args);
    clearTimeout(timer);
    return result;
  } catch (err) {
    if (err.name === 'AbortError') {
      throw new Error(`IPC timeout: ${channel} (${timeout}ms)`);
    }
    throw err;
  }
}
```

这个封装解决了两个痛点：

1. **浏览器兼容调试**——在 Chrome 里打开 `frontend/dist` 也能跑（Mock 数据），不需要每次都启动 Electron
2. **超时保护**——主进程如果卡住（比如网络请求超时），渲染进程不会无限等待

### 主进程 Controller 自动映射

```javascript
// electron/controller/tts.js — ee-core 自动注册为 IPC handler
const { Controller } = require('ee-core');
const TtsService = require('../service/tts');

class TtsController extends Controller {
  // 自动映射为 ipcMain.handle('tts:generate', ...)
  async generate() {
    const { text, voiceConfig } = this.ctx.request.body || {};
    const service = new TtsService(this.ctx.app);
    return await service.generate(text, voiceConfig);
  }

  async healthCheck() {
    const service = new TtsService(this.ctx.app);
    return await service.healthCheck();
  }
}

module.exports = TtsController;
```

ee-core 的约定：**文件名 = 模块名，方法名 = 动作名**，自动拼成 `模块:动作` 的 channel 名。零配置路由。

---

## Q5：窗口配置和预加载脚本有什么要注意的？

**A：** 这两点直接影响应用的**安全性**和**用户体验**。

### 窗口配置

```javascript
// electron/config/config.default.js
module.exports = {
  windows: {
    main: {
      width: 980,
      height: 650,
      minWidth: 800,
      minHeight: 500,
      frame: true,            // 保留原生标题栏（后续可改为自定义）
      backgroundColor: '#16181c',  // 深色背景（防止白闪）
      webPreferences: {
        nodeIntegration: false,     // ❌ 禁用 Node 集成（安全！）
        contextIsolation: true,     // ✅ 启用上下文隔离（必须！）
        preload: path.join(__dirname, '../preload/index.js'),
      },
    },
  },
};
```

**三个关键安全配置的含义：**

| 配置 | 含义 | 如果关掉会怎样 |
|------|------|--------------|
| `nodeIntegration: false` | 渲染进程不能直接用 `require()` | XSS 攻击可以直接调用 Node.js API |
| `contextIsolation: true` | 渲染进程运行在独立沙箱 | 同上 |
| `preload` | 桥接脚本，用 `contextBridge.exposeInMainWorld` 安全暴露 API | 没有 preload 就完全无法通信 |

> **永远不要在生产环境中关闭 contextIsolation！** 这是 Electron 安全模型的基础。

### 预加载脚本（Preload）

```javascript
// electron/preload/bridge.js — 安全暴露 API 给渲染进程
const { contextBridge, ipcRenderer } = require('electron');

contextBridge.exposeInMainWorld('electronAPI', {
  invoke: (channel, ...args) => ipcRenderer.invoke(channel, ...args),
  on: (channel, callback) => {
    const subscription = (_, ...args) => callback(...args);
    ipcRenderer.on(channel, subscription);
    return () => ipcRenderer.removeListener(channel, subscription);
  },
});
```

**只暴露两个方法：**
- `invoke` — 单次请求-响应（对应 `ipcRenderer.invoke`）
- `on` — 订阅事件（对应 `ipcRenderer.on`，带自动取消订阅）

**不暴露的东西更多：** `send`（单向推送）、`postMessage`（消息端口）、所有 Node.js API —— 最小权限原则。

---

## Q6：构建和打包是怎么做的？多平台构建有什么经验？

**A：** 用 `electron-builder` 打包，配合 ee-core 内置的构建流程。

### 构建流水线

```
开发阶段：
  npm run dev
  → 同时启动 Electron 主进程 + Vite dev server（HMR）

生产构建：
  Step 1: npm run build-frontend   → Vite build → frontend/dist/
  Step 2: npm run build-electron   → electron-builder 打包
  或一键：npm run build             → 按顺序执行上面两步
```

### 多平台配置

```jsonc
// cmd/builder.json — electron-builder 多平台配置
{
  "productName": "MiMo TTS Studio",
  "appId": "com.mimo.tts-studio",
  "directories": { "output": "release" },
  "files": ["electron/**", "preload/**"],
  "extraResources": [
    { "from": "frontend/dist", "to": "app/frontend/dist" }
  ],
  "mac": {
    "target": [{ "target": "dmg", "arch": ["arm64"] }],
    "icon": "build/icon.icns"
  },
  "win": {
    "target": [{ "target": "nsis", "arch": ["x64"] }],
    "icon": "build/icon.ico"
  },
  "linux": {
    "target": [{ "target": "AppImage" }]
  },
  "asar": true
}
```

### 几个踩过的坑

| 问题 | 解决方案 |
|------|---------|
| macOS ARM + Intel 双架构 | 分别用 `build-m-arm64` 和 `build-w` 构建，CI 时并行 |
| NSIS 安装包中文路径乱码 | builder.json 加 `"nsis": { "oneClick": false, "allowToChangeInstallationDirectory": true }` |
| asar 打包后文件读取失败 | 用 `process.resourcesPath` 拼路径，不要用 `__dirname` 相对路径 |
| 包体积过大（~200MB） | `extraResources` 只放 dist 不放 node_modules；生产依赖精简 |

---

## 总结

| 决策 | 选择 | 理由 |
|------|------|------|
| 桌面框架 | Electron | Node.js 文件 I/O + TTS SDK 对接能力 |
| 主进程框架 | ee-core | 减少 ~400 行样板代码，MVC 路由自动映射 |
| 前端框架 | Vue 3 + Vite | 熟悉度高、组合式 API 适合复杂状态管理 |
| UI 库 | Naive UI | 原生暗色主题、极致 tree-shake、高度可定制 |
| 进程通信 | contextBridge + IPC 封装 | 安全隔离 + 浏览器 Mock 兼容 |
| 打包工具 | electron-builder | 成熟稳定、多平台一键构建 |

下一篇我们将深入第一个核心技术模块——**TTS 引擎的双 Provider 架构设计与实现**。
