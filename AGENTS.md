# AGENTS.md — AI Agent 博客协作规范

本文件为 AI Agent（Cline、Claude 等）提供博客协作的规范和上下文。

## 仓库概述

这是一个 Hexo + Butterfly 主题的技术博客，部署在 GitHub Pages（july-x.github.io）。

- **框架**：Hexo 8.x + Butterfly 主题
- **部署**：GitHub Actions 自动构建，推送到 main 分支自动发布
- **站点地址**：https://july-x.github.io/

## 博客写作原则

### 核心规则

1. **真实性第一**：不得虚构/夸大概念。所有代码、设计逻辑都要来源于项目真实数据、代码、设计文档。
2. **代码验证**：写入文章的每个代码片段，必须能从项目源码中找到对应的真实实现。如果找不到，要么不写，要么标注"简化示意"。
3. **TODO 标注**：如果在分析项目代码时发现不合理、可优化的地方，在文章结尾用 TODO 形式标注：
   ```
   > **TODO**: [问题描述] — [建议优化方向]
   ```
4. **不得虚构接口/函数**：不能凭空创造不存在的数据结构、API 接口、函数签名。宁可少写也不能写错。
5. **发布时间不变**：博客上方的 `date` 是首次发布时间，后续修改博客不要改动这个时间。
6. **更新记录**：针对单个博客如果有新的改动，就在文末写上时间线，再在时间线下面开始新的更新。小范围细节调整用括号加删除线表示替换（如 `~~旧内容~~（新内容）`）。

### 写作风格

1. **去除 AI 味**：用事故/踩坑故事开头，加入真实对话引用、决策过程、修复前后对比。
2. **结尾用反问句**替代金句式总结。
3. **篇幅**：1000~1500 字，聚焦一个技术点。
4. **表现形式多样化**：不仅仅是代码+流程图，还可以用：
   - ASCII 时间线表格展示数据变化
   - 决策过程分析（"为什么选 A 不选 B"）
   - 修复前/修复后的效果对比
   - 真实对话引用增加可信度
   - 性能数据/监控截图描述

### 技术格式

1. **Mermaid 标签语法**：使用 `{% mermaid %}` 标签，不用 `` ```mermaid `` 代码块（会被 highlight.js 覆盖）。
2. **GDScript 语法高亮**：用 `python` 标记（highlight.js 不支持 `gdscript`）。
3. **Mermaid 特殊字符转义**：`.` `()` `>` `<` `%` 等字符用双引号包裹。
4. **Go 代码**：用 `go` 标记。
5. **TypeScript/ArkTS**：用 `typescript` 标记。

## 博文目录结构

```
source/_posts/
├── godot-game-one/     # Godot 4 游戏开发（7篇）
├── tts-tool/           # Electron + Vue TTS 工具（4篇）
├── game-bass/          # 游戏后端模板（4篇）
├── easy-word/          # 鸿蒙单词学习 App（4篇）
├── slg-go/             # SLG 百万在线服务器（4篇）
├── 技术博客写作计划.md
└── 你好，我是-Izzy.md
```

## 关注的源码项目

| 项目 | 路径 | 技术栈 | 博文数 |
|------|------|--------|--------|
| godot-game-one | ~/2026/code/godot-game-one | Godot 4, GDScript | 7 |
| tts-tool | ~/2026/code/tts-tool | Electron, Vue 3, TypeScript | 4 |
| game-bass | ~/2026/code/game-bass | Go, Redis, gorilla/websocket | 4 |
| easy-word | ~/2026/code/easy-word | ArkTS, ArkUI, 鸿蒙 NEXT | 4 |
| slg-go | ~/2026/code/slg-go | Go, gopher-lua, Nacos, gRPC | 4 |

## Git 工作流

1. 所有博文变更在本地提交
2. `git push origin main` 触发 GitHub Actions 自动构建部署
3. 提交信息格式：`feat: 项目名 第N篇 - 主题` 或 `fix: 描述`
4. 修改已发布文章用 `docs:` 或 `fix:` 前缀

## 常用命令

```bash
# 本地预览
npx hexo server

# 生成静态文件
npx hexo generate

# 提交并推送
git add -A && git commit -m "feat: ..." && git push origin main
```

## 写作流程

1. 使用 subagent 探索源码项目，收集关键代码片段
2. 验证代码片段的真实性（必须能从项目中找到）
3. 撰写博文（遵循写作原则）
4. 更新 `source/_posts/技术博客写作计划.md`：新增的独立计划模块放到"前言"下方
5. git add + commit + push
