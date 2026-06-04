---
title: Electron 实战：electron-builder 多平台构建实战
date: 2026-06-04
categories:
  - [Electron实战]
tags:
  - Electron
  - electron-builder
  打包
  多平台
  macOS
  Windows
  工程化
  - 问答
top_img: linear-gradient(135deg, #0c3483 0%, #a2b6df 100%)
---

<!-- more -->

> MiMo TTS Studio 需要打包为 macOS（ARM64）、Windows（x64）、Linux 三平台的安装包。本文记录 electron-builder 的配置细节、踩过的坑、以及如何把包体积控制在合理范围。

---

## Q1：构建流水线是怎样的？

**A：**

```
开发阶段：
  npm run dev
  → Vite dev server (前端 HMR) + Electron 主进程热重载

生产构建（两步走）：
  Step 1: npm run build-frontend     → Vite build → frontend/dist/
  Step 2: npm run build-electron       → electron-builder → release/

一键构建：
  npm run build                        → 按顺序执行上面两步
```

```json
// package.json scripts
{
  "dev": "concurrently \"npm run dev:frontend\" \"npm run dev:electron\"",
  "dev:frontend": "vite",
  "dev:electron": "electron .",
  "build-frontend": "vite build",
  "build-electron": "electron-builder --config cmd/builder.json",
  "build-w": "npm run build-frontend && npm run build-electron -w --x64",
  "build-m": "npm run build-frontend && npm run build-electron -m --arm64",
  "build-m-arm64": "npm run build-frontend && npm run build-electron -m --arm64",
  "build-l": "npm run build-frontend && npm run build-electron -l"
}
```

**关键点：必须先 build-frontend 再 build-electron**——electron-builder 会把 `frontend/dist` 打进安装包。

---

## Q2：builder.json 核心配置详解？

**A：**

```jsonc
// cmd/builder.json — electron-builder 完整配置
{
  "productName": "MiMo TTS Studio",
  "appId": "com.mimo.tts-studio",

  // 输出目录
  "directories": {
    "output": "release"
  },

  // 要打包的文件（排除 node_modules 等无关文件）
  "files": [
    "electron/**",
    "preload/**"
    // 注意：不包含 frontend/ 和 node_modules/ — 它们通过 extraResources 处理
  ],

  // 额外资源（原样复制到应用内）
  "extraResources": [
    {
      "from": "frontend/dist",
      "to": "app/frontend/dist",
      "filter": ["**/*", "!**/*.map"]   // 排除 sourcemap
    }
  ],

  // ===== macOS 配置 =====
  "mac": {
    "target": [
      { "target": "dmg", "arch": ["arm64"] }   // Intel Mac 用户可通过 Rosetta 运行
    ],
    "icon": "build/icon.icns",
    "category": "public.app-category.productivity",
    "hardenedRuntime": true,          // 启用运行时安全保护
    "gatekeeperAssess": false,      // 不自动提交公证（开发阶段）
    "entitlements": "build/entitlements.mac.plist",
    "entitlementsInherit": "build/entitlements.mac.plist"
  },

  // ===== Windows 配置 =====
  "win": {
    "target": [
      { "target": "nsis", "arch": ["x64"] }
    ],
    "icon": "build/icon.ico",
    "requestedExecutionLevel": "asInvoker"   // 不需要管理员权限
  },

  // NSIS 安装程序配置
  "nsis": {
    "oneClick": false,                    // 不用一键安装，让用户选目录
    "allowToChangeInstallationDirectory": true,
    "installerIcon": "build/icon.ico",
    "uninstallerIcon": "build/icon.ico",
    "createDesktopShortcut": true,
    "createStartMenuShortcut": true,
    "shortcutName": "MiMo TTS Studio"
  },

  // ===== Linux 配置 =====
  "linux": {
    "target": [{ "target": "AppImage" }],
    "icon": "build/icons/",
    "category": "AudioVideo;Audio"
  },

  // 通用配置
  "asar": true,                          // asar 打包（保护源码 + 加载速度）
  "compression": "maximum"               // 最大压缩（牺牲打包速度换更小的包体积）
}
```

### 各平台产物

| 平台 | 格式 | 典型大小 | 安装方式 |
|------|------|---------|---------|
| macOS ARM64 | `.dmg` | ~95 MB | 拖入 Applications |
| Windows x64 | `Setup.exe` (NSIS) | ~98 MB | 双击运行安装器 |
| Linux | `.AppImage` | ~100 MB | chmod +x 后直接运行 |

---

## Q3：asar 打包策略怎么配？哪些文件该进去？

**A：** `asar` 是 Electron 的归档格式——把多个文件打包成一个类似 tar 的归档文件：

```
asar 的作用:
  ├── 保护源码（不能直接查看 JS 文件）
  ├── 加速加载（一次 IO 读整个 asar vs 数百次 stat）
  └── 减少文件数量（安装包内少了几千个小文件）

asar 内的文件仍然可以通过 process.resourcesPath 访问
```

### 关键决策：什么放 files，什么放 extraResources

| 目录/文件 | 放哪里 | 原因 |
|-----------|--------|------|
| `electron/**/*.js` | **files**（进入 asar）| 主进程核心代码，需要保护 |
| `preload/**/*.js` | **files**（进入 asar）| 安全敏感代码 |
| `frontend/dist/**` | **extraResources** | 前端静态资源，不需要在 asar 里 |
| `node_modules/`（生产依赖） | **都不放** | 通过 package.json dependencies 自动处理 |
| `node_modules/`（devDependencies） | **都不放** | devDependencies 在打包时自动排除 |

```javascript
// 运行时读取 extraResources 文件的正确姿势
const path = require('path');
const resourcePath = process.resourcesPath;  // Electron 提供的特殊变量
const frontendDist = path.join(resourcePath, 'app', 'frontend', 'dist');

// ❌ 错误写法（asar 内路径不对）
const wrongPath = path.join(__dirname, '..', 'frontend', 'dist');

// ✅ 正确写法
const rightPath = path.join(process.resourcesPath, 'app', 'frontend', 'dist');
```

---

## Q4：包体积优化有哪些手段？

**A：** 从初始的 ~200MB 降到 ~95MB 的经验总结：

| 手段 | 节省量 | 难度 |
|------|--------|------|
| **1. 排除 devDependencies** | ~40 MB | 低（electron-builder 默认行为）|
| **2. extraResources 只放 dist 不放 node_modules** | ~30 MB | 中（配置调整）|
| **3. compression: maximum** | ~15 MB | 低（一行配置）|
| **4. 排除 sourcemap** | ~5 MB | 低（filter 配置）|
| **5. 精简 production dependencies** | ~10 MB | 高（需逐一确认）|

```jsonc
// package.json — 依赖精简示例
{
  "dependencies": {
    // 必须的
    "ee-core": "^3.6.0",
    "electron-builder": "^26.3.5",

    // 可选的（如果不用可移除）
    // "iconv-lite": "^0.6.3",        ← 编码检测备选，非必需
    // "winston": "^3.x",             ← ee-core 自带日志
    // "axios": "^1.x",              ← 用原生 fetch 替代
  },
  "devDependencies": {
    "vite": "^5.4.11",
    "vue": "^3.5.12",
    "naive-ui": "^2.44.1",
    // ... 这些全部不会被打包进发布版本
  }
}
```

### 体积分析命令

```bash
# 查看生成的 DMG/AppImage 内容
du -sh release/MiMo\ TTS\ Studio-dmg/
du -sh release/mac-arm64/

# 查看 asar 内容
npx asar list release/mac-arm64/Resources/app.asar | head -50

# 分析具体哪个依赖占空间大
du -sh node_modules/*/ | sort -hr | head -10
```

---

## Q5：macOS ARM64 和 x64 双架构怎么处理？

**A：** 当前只构建了 **ARM64**（Apple Silicon），原因：

| 因素 | 说明 |
|------|------|
| 目标用户 | 开发者为主，MacBook Pro M 系列 |
| 包体积 | 单架构比双架构小约 **40%** |
| 构建时间 | 单架构快一倍以上 |
| Rosetta 兼容 | x64 用户可通过 Rosetta 2 运行 ARM64 版本 |

如果未来需要支持 x64 原生：

```jsonc
// builder.json — 改为双架构
"mac": {
  "target": [
    { "target": "dmg", "arch": ["arm64", "x64"] }
  ]
}

// 或分别构建两个 DMG：
"mac": {
  "target": [
    { "target": "dmg", "arch": ["arm64"] },    // MiMo-TTS-Studio-arm64.dmg
    { "target": "dmg", "arch": ["x64"] }       // MiMo-TTS-Studio-x64.dmg
  ]
}
```

CI/CD 中可以**并行构建**两个架构（如果同时有 Mac Intel 和 Apple Silicon 的 runner）。

---

## Q6：Windows NSIS 安装包有什么坑？

**A：**

### 中文路径问题

```
❌ 默认配置下，用户安装到含中文/空格的路径可能失败
✅ 解决方案：
  "nsis": {
    "oneClick": false,                           // 让用户自己选目录
    "allowToChangeInstallationDirectory": true,   // 允许修改安装路径
  }
```

### 数字签名（可选但推荐）

```
未签名的 EXE:
  Windows SmartScreen 弹出 "Windows 保护了你的电脑" 警告
  用户需要点击"更多信息"→"仍要运行"

已签名（购买代码签名证书后）:
  安装过程无警告
  显示发布者名称（如 "Izzy"）

# 签名命令（需要在 Windows 上操作）
signtool sign /f certificate.pfx /p password /t http://timestamp.digicert.com /fd hash release/Setup.exe
```

### 杀软误报

Electron 应用经常被 Windows Defender 标记为"未知发行商"。解决方案：
1. **代码签名证书**（~$200/年）— 最有效
2. **提交给微软审核** — 免费，但周期长（数天到数周）
3. **引导用户添加白名单** — 临时方案，体验差

---

## Q7：CI/CD 怎么自动化构建三平台？

**A：** 如果使用 GitHub Actions：

```yaml
# .github/workflows/build.yml
name: Build MiMo TTS Studio

on:
  push:
    tags: ['v*']            # 只有打 tag 时才触发构建

jobs:
  build-mac:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run build-m
      - uses: actions/upload-artifact@v4
        with:
          name: mac-dmg
          path: release/*.dmg

  build-win:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run build-w
      - uses: actions/upload-artifact@v4
        with:
          name: windows-setup
          path: release/*.exe

  build-linux:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: sudo apt-get update && sudo apt-get install -y libgtk-3-dev
      - run: npm ci
      - run: npm run build-l
      - uses: actions/upload-artifact@v4
        with:
          name: linux-appimage
          path: release/*.AppImage

  release:
  needs: [build-mac, build-win, build-linux]
  runs-on: ubuntu-latest
  if: startsWith(github.ref, 'refs/tags/')
  steps:
      - uses: actions/download-artifact@v4
      - uses: softprops/action-gh-release@v1
        with:
          generate_release_notes: true
          files: '*/*'
```

### 本地快速构建（无需 CI）

```bash
# 只想测试 macOS 构建
npm run build-m

# 想快速验证 Windows 构建但不装 Windows？
# 可以用 cross-env + wine（Linux）或直接跳过（大部分逻辑是跨平台的）

# 最快的验证方式：只跑 Vite build（不执行 electron-builder）
npm run build-frontend
# 然后手动 npm run dev 启动 Electron 加载 dist 目录
```

---

## 总结

| 话题 | 关键结论 |
|------|---------|
| **构建流程** | 先 `vite build` 再 `electron-builder`，两步分离 |
| **多平台配置** | 一个 builder.json + 不同 target/arch 组合 |
| **asar 策略** | 核心 JS 进 asar，前端静态资源走 extraResources |
| **体积控制** | ~200MB → ~95MB（排除 devDeps + maximum 压缩 + 精简依赖）|
| **macOS** | 当前仅 ARM64，Rosetta 兼容 x64 |
| **Windows** | NSIS + oneClick=false 解决中文路径问题；代码签名解决 SmartScreen |
| **Linux** | AppImage 格式，零安装即用 |
| **CI/CD** | GitHub Actions 并行构建三平台，tag 触发 + auto-release |

---

> 到这里，「Electron 实战」系列 9 篇文章全部完成！从技术选型到多平台构建，覆盖了一个 Electron 桌面应用的完整生命周期。每篇文章都基于 MiMo TTS Studio 的真实源码——不是 Hello World 教程，而是从真实项目中提炼的经验。
