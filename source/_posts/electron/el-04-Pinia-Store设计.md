---
title: Electron 实战：Pinia Store 设计 — 6 个 Store 的职责划分
date: 2026-06-04
categories:
  - [Electron实战]
tags:
  - Electron
  - Vue 3
  - Pinia
  - 状态管理
  - 架构设计
  - 问答
top_img: linear-gradient(135deg, #11998e 0%, #38ef7d 100%)
---

<!-- more -->

> MiMo TTS Studio 的前端状态管理用了 6 个 Pinia Store。这不是过度设计——每个 Store 对应一个清晰的业务域，Store 之间通过依赖注入（`useXxxStore()`）协作。本文拆解每个 Store 的职责边界和协作关系。

---

## Q1：6 个 Store 分别管什么？

**A：** 先看全景图：

```
┌─────────────────────────────────────────────────┐
│                   Store 架构                    │
│                                                 │
│  ┌──────────────┐                               │
│  │ configStore  │ ← 全局配置（Provider/导出格式） │
│  └──────┬───────┘                               │
│         │ 被读取                                │
│  ┌──────▼───────┐  ┌──────────────┐             │
│  │characterStore│  │   ttsStore   │ ◄── 核心调度  │
│  │ (角色管理)    │  │ (TTS生成/并发) │             │
│  └──────┬───────┘  └──────┬───────┘             │
│         │ 读写              │ 调用               │
│  ┌──────▼───────┐  ┌──────▼───────┐             │
│  │  lineStore   │  │  projectStore│              │
│  │ (台词CRUD)   │  │ (项目聚合层) │              │
│  └──────────────┘  └──────────────┘             │
│                                                 │
│  ┌──────────────┐  ┌──────────────┐             │
│  │ useTheme     │  │ useKeyboard   │            │
│  │ (深浅主题)    │  │ (快捷键)      │            │
│  └──────────────┘  └──────────────┘             │
└─────────────────────────────────────────────────┘
```

| Store | 文件 | 职责 | 状态数量 |
|-------|------|------|---------|
| **characterStore** | characterStore.js | 角色的 CRUD、磁盘持久化、项目目录扫描恢复 | ~10 |
| **lineStore** | lineStore.js | 单条台词的状态管理、最后生成结果 | ~8 |
| **ttsStore** | ttsStore.js | **TTS 核心引擎**：生成/试听/批量/并发池 | ~25 |
| **configStore** | configStore.js | 全局配置：API地址、导出格式、TTS Provider | ~15 |
| **useTheme** | useTheme.js | 深色/浅色主题切换 | ~3 |
| **useKeyboardShortcuts** | useKeyboardShortcuts.js | 键盘快捷键注册与处理 | ~5 |

---

## Q2：characterStore 是怎么实现磁盘持久化的？"角色即文件夹"是什么意思？

**A：** 这是整个数据模型的核心设计——**每个角色在磁盘上就是一个文件夹，角色属性 = 文件**：

```javascript
// characterStore.js — 数据结构

// 一个角色的内存对象
function createDefaultCharacter(name = '默认角色') {
  return {
    id: generateId('char'),           // 唯一 ID
    name,                              // 显示名
    voiceId: 'mimo_default',          // 当前音色
    voiceSource: 'preset',            // 'preset' | 'voicedesign' | 'voiceclone'
    voiceDesignPrompt: '',            // 音色设计 Prompt
    cloneVoicePath: '',                // 克隆参考音频路径
    cloneVoiceMime: 'audio/wav',
    styleTags: [],                     // 风格标签（如 ['温柔', '东北话']）
    speed: 1.0,                        // 语速
    lines: []                          // 台词数组
  };
}

// 角色的磁盘目录结构
function charDir(ch) {
  return (projectPath.value + '/' + ch.name).replace(/\/+/g, '/');
}
// 例: /Users/xxx/project/NPC张三/
```

**磁盘上的实际文件：**

```
projectPath/
└── NPC张三/                  ← 角色文件夹（名字即角色名）
    ├── voice.json           ← 音色配置
    ├── script.txt           ← 台词文本（每行一句）
    ├── lines_meta.json      ← 台词元数据（ID/音频路径/状态/时长）
    └── audio/               ← 生成的音频文件
        ├── NPC张三_L01.wav
        ├── NPC张三_L02.wav
        └── ...
```

### 为什么这样设计？

| 设计选择 | 好处 |
|---------|------|
| **角色=文件夹** | 文件管理器里一目了然，可以直接操作 |
| **script.txt 存纯文本** | 可以用任何编辑器编辑台词，Git 友好（diff 友好）|
| **voice.json 存 JSON** | 结构化配置，程序读写方便 |
| **audio/ 子目录** | 音频文件和配置分离，不会弄混 |
| **lines_meta.json** | 记录哪些台词已合成、状态如何，启动时快速恢复 |

### 保存到磁盘的实现

```javascript
async function saveCharacterToDisk(ch) {
  if (!projectPath.value) return;

  const dir = charDir(ch);
  const audioDir = dir + '/audio';

  // 确保目录存在
  await ensureDir(dir);
  await ensureDir(audioDir);

  // 1. 写入音色配置 → voice.json
  const voiceConfig = {
    voiceId: ch.voiceId,
    voiceSource: ch.voiceSource,
    voiceDesignPrompt: ch.voiceDesignPrompt,
    cloneVoicePath: ch.cloneVoicePath || '',
    styleTags: ch.styleTags,
    speed: ch.speed
  };
  await writeFile(dir + '/voice.json', JSON.stringify(voiceConfig, null, 2));

  // 2. 写入台词文本 → script.txt（纯文本，一行一句）
  const scriptText = ch.lines.map(l => l.text).join('\n');
  await writeFile(dir + '/script.txt', scriptText);

  // 3. 写入元数据 → lines_meta.json
  const metaData = ch.lines.map(l => ({
    id: l.id,
    audioPath: l.audioPath,
    status: l.status,         // 'pending' | 'generating' | 'done' | 'error'
    duration: l.duration
  }));
  await writeFile(dir + '/lines_meta.json', JSON.stringify(metaData, null, 2));
}
```

> **防抖落盘**：每次修改后不是立刻写盘，而是等 1 秒无操作后再写（通过 `watch` + `debounce` 实现），避免频繁 IO。

---

## Q3：lineStore 和 characterStore 的关系是什么？

**A：** `lineStore` 不独立拥有台词数据——它只是 `characterStore.currentCharacter.lines` 的**便捷操作层**：

```javascript
// lineStore.js
export const useLineStore = defineStore('line', () => {
  // ===== 状态 =====
  const isGenerating = ref(false);          // 是否正在生成中
  const lastGeneration = ref(null);         // 最近一次生成的结果
  const associatedAudio = ref(null);        // 关联的音频信息

  // ===== 台词 CRUD 操作 =====
  function addLine(text, charId) { /* ... */ }
  function updateLine(lineId, updates) { /* ... */ }
  function removeLine(lineId) { /* ... */ }
  function reorderLines(fromIdx, toIdx) { /* ... */ }
  function importLines(textArray) { /* ... */ }

  // 注意：这些操作的底层都是修改 characterStore 的 currentCharacter.lines
  // 然后自动触发 saveCharacterToDisk()
});
```

**为什么不把所有逻辑都放 characterStore？**

因为 `ttsStore` 需要频繁访问"当前正在生成"的状态（`isGenerating`、`lastGeneration`），如果这些状态塞进 characterStore，会让角色模型的职责变模糊。

分离的好处：
- **characterStore** 专注：角色增删改查 + 磁盘 IO
- **lineStore** 专注：单条台词的操作 + 生成状态
- **ttsStore** 专注：TTS 引擎调用 + 并发控制

三者关系：`ttsStore` 调用 `lineStore` 更新状态，`lineStore` 操作 `characterStore` 的数据，`characterStore` 负责最终落盘。

---

## Q4：ttsStore 作为核心调度者，它怎么协调其他 Store？

**A：** `ttsStore` 是整个前端最复杂的 Store——它是 TTS 操作的唯一入口，内部通过 `useXxxStore()` 获取其他 Store 的引用：

```javascript
export const useTtsStore = defineStore('tts', () => {
  // 声明式依赖（Pinia 允许在 setup 函数内调用其他 Store）
  const charStore = useCharacterStore();    // 但注意：这里不能直接调用！
  const configStore = useConfigStore();
  const lineStore = useLineStore();

  // ❌ 上面的方式有问题——会在 defineStore 时就创建实例
  // ✅ 正确做法：在实际函数内部延迟获取
```

**正确的跨 Store 协作模式：**

```javascript
async function doGenerateTTS(outputPath, outputFilename, options = {}) {
  // 在函数内部获取最新实例（避免 stale closure）
  const charStore = useCharacterStore();
  const configStore = useConfigStore();
  const lineStore = useLineStore();

  lineStore.isGenerating = true;       // ← 通知 lineStore：开始生成了

  try {
    // 从 configStore 读配置
    const provider = configStore.ttsProvider || 'mimo';
    const format = configStore.exportFormat;

    // 从 characterStore 读当前角色的音色参数
    const ch = charStore.currentCharacter;

    // 调用 IPC 发给主进程
    const result = await ipc.invoke(ipcApiRoute.ttsGenerate, params);

    // 写回 lineStore 的状态
    if (result?.success) {
      lineStore.lastGeneration = result;
      lineStore.associatedAudio = { type: 'audio', path: result.audioPath };
    }

    return result;
  } finally {
    lineStore.isGenerating = false;   // ← 无论成败，结束生成状态
  }
}
```

**调用链路图（以生成一条台词为例）：**

```
用户点击 "生成" 按钮
  ↓
ttsStore.generateLine(charId, lineId)
  ↓ ├── validateCloneConfig(ch)        // 校验克隆配置
  │   └── charStore.characters.find()
  ↓
  │   line.status = 'generating'
  │   charStore.saveCharacterToDisk()  // 持久化状态变更
  ↓
  │   ttsStore.doGenerateTTS(path, filename, options)
  │   ├── configStore → 读取 exportFormat / ttsProvider
  │   ├── charStore → 读取 voiceId / speed / voiceSource
  │   ├── ipc.invoke('tts:generate', params)  → 主进程 TTSService
  │   ↓
  │   │   lineStore.lastGeneration = result   // 记录结果
  │   │   lineStore.associatedAudio = {...}   // 关联音频
  │   ↓
  │   line.status = 'done' 或 'error'
  │   charStore.saveCharacterToDisk()          // 再次持久化
  ↓
返回结果 → UI 更新波形/播放器
```

---

## Q5：configStore 管了哪些全局配置？怎么保证配置不丢失？

**A：** `configStore` 是唯一一个**应用级全局配置**的 Store：

```javascript
// configStore.js
export const useConfigStore = defineStore('config', () => {
  // ===== TTS 引擎配置 =====
  const ttsProvider = ref('mimo');              // 'mimo' | 'voxCpm'
  const apiBase = ref('');
  const apiKey = ref('');

  // ===== VoxCPM 专属配置 =====
  const voxCpmEndpoint = ref('');
  const voxCpmApiKey = ref('');
  const voxCpmModel = ref('voxcpm2');

  // ===== 导出配置 =====
  const exportFormat = ref('wav');              // 'wav' | 'mp3'

  // ===== 功能开关 =====
  const allowSyntheticFallback = ref(true);
  const directorMode = ref({
    role: '',
    scene: '',
    guide: ''
  });

  // ===== 持久化到本地文件 =====
  async function saveConfig() {
    const data = { ttsProvider, apiBase, apiKey, /* ... */ };
    // 通过 IPC 写入主进程的配置文件
    await ipc.invoke(ipcApiRoute.configSave, data);
  }

  async function loadConfig() {
    const data = await ipc.invoke(ipcApiRoute.configGet);
    if (data) {
      Object.assign(/* 合并到各 ref */);
    }
  }

  return { ttsProvider, apiBase, apiKey, /* ... */, saveConfig, loadConfig };
});
```

### 配置持久化链路

```
configStore.saveConfig()
  ↓ IPC: 'config:save'
  ↓ 主进程 ConfigController.save()
  ↓ service/config.js → 写入 app/data/config.json
  ↓ 下次启动 → config.load() → 自动恢复
```

配置独立于角色数据存储——**换项目不丢配置，换电脑可以迁移配置文件**。

---

## Q6：useTheme 和 useKeyboardShortcuts 为什么也是 Store？

**A：** 严格来说它们更像是 **Composable（组合式函数）**，但用 Pinia Store 的原因是：

1. **需要跨组件共享状态** —— 多个组件需要知道当前是亮/暗主题
2. **需要响应式** —— 切换主题后所有组件立即更新
3. **保持一致性** —— 团队约定"共享状态都用 Store"

```javascript
// useTheme.js — 极简主题 Store
export const useTheme = defineStore('theme', () => {
  const isDark = ref(true);                       // 默认暗色

  function toggleTheme() {
    isDark.value = !isDark.value;
    document.documentElement.setAttribute(
      'data-theme', isDark.value ? 'dark' : 'light'
    );
  }

  // 启动时从 localStorage 恢复
  function initTheme() {
    const saved = localStorage.getItem('theme');
    if (saved !== null) isDark.value = saved === 'dark';
    toggleTheme();  // 应用初始值
  }

  return { isDark, toggleTheme, initTheme };
});
```

```javascript
// useKeyboardShortcuts.js
export const useKeyboardShortcuts = defineStore('shortcuts', () => {
  const shortcuts = ref([
    { key: 'Ctrl+Enter', action: 'generateCurrent', label: '生成当前台词' },
    { key: 'Ctrl+Shift+G', action: 'generateAll', label: '全部生成' },
    { key: 'Space', action: 'togglePlayPause', label: '播放/暂停' },
    { key: 'Ctrl+S', action: 'saveProject', label: '保存' },
  ]);

  function registerGlobal() {
    window.addEventListener('keydown', handleKeyDown);
  }

  function unregisterGlobal() {
    window.removeEventListener('keydown', handleKeyDown);
  }

  // ...
});
```

---

## 总结

| Store | 核心职责 | 依赖谁 | 被谁依赖 |
|-------|---------|--------|---------|
| **configStore** | 全局配置读写 | 无（仅 IPC） | ttsStore, 所有 UI 组件 |
| **characterStore** | 角色 CRUD + 磁盘持久化 | IPC (file) | lineStore, ttsStore |
| **lineStore** | 台词操作 + 生成状态 | characterStore | ttsStore, UI 组件 |
| **ttsStore** | TTS 引擎调度 + 并发控制 | configStore, characterStore, lineStore | UI 组件 |
| **useTheme** | 主题切换 | 无 | App.vue, Naive UI |
| **useKeyboardShortcuts** | 快捷键 | ttsStore | Index.vue |

**设计原则总结：**
1. **单向依赖**：configStore → characterStore → lineStore → ttsStore，不循环
2. **单一职责**：每个 Store 只管自己的领域
3. **延迟获取**：跨 Store 引用在函数内部用 `useXxxStore()` 获取，避免 stale closure
4. **磁盘是真相源**：内存状态可以丢，但磁盘文件不能丢；启动时从磁盘恢复

下一篇深入 **IPC 封装** —— 浏览器/Electron 双环境兼容的设计。
