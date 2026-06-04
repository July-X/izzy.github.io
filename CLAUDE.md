# CLAUDE.md — 博客写作规范

## 项目概述

Hexo + Butterfly 主题的技术博客，部署在 GitHub Pages（july-x.github.io）。

## 写作核心原则

1. **真实性第一**：不得虚构/夸大概念。所有代码、设计逻辑必须来源于项目真实源码。找不到对应实现的代码，要么不写，要么标注"简化示意"。
2. **TODO 标注**：发现项目中不合理、可优化的地方，在文章结尾用 TODO 形式标注：
   ```
   > **TODO**: [问题描述] — [建议优化方向]
   ```
3. **不得虚构接口/函数**：不能凭空创造不存在的数据结构、API 接口、函数签名。
4. **篇幅**：1000~1500 字，聚焦一个技术点。
5. **去除 AI 味**：用事故/踩坑故事开头，加入真实对话引用、决策过程、修复前后对比。结尾用反问句替代金句式总结。
6. **表现形式多样化**：ASCII 时间线表格、决策过程分析、修复前后对比、真实对话引用、性能数据。

## 技术格式

1. Mermaid 用 `{% mermaid %}` 标签，不用 `` ```mermaid `` 代码块
2. GDScript 代码块用 `python` 标记
3. Mermaid 中的 `.()` `>` `<` `%` 用双引号包裹
4. Go 用 `go`，TypeScript/ArkTS 用 `typescript`

## 博文目录

```
source/_posts/
├── godot-game-one/     # 7篇
├── tts-tool/           # 4篇
├── game-bass/          # 4篇
├── easy-word/          # 4篇
├── slg-go/             # 4篇
```

## 源码项目

| 项目 | 路径 | 技术栈 |
|------|------|--------|
| godot-game-one | ~/2026/code/godot-game-one | Godot 4, GDScript |
| tts-tool | ~/2026/code/tts-tool | Electron, Vue 3, TypeScript |
| game-bass | ~/2026/code/game-bass | Go, Redis, gorilla/websocket |
| easy-word | ~/2026/code/easy-word | ArkTS, ArkUI, 鸿蒙 NEXT |
| slg-go | ~/2026/code/slg-go | Go, gopher-lua, Nacos, gRPC |

## 写作流程

1. subagent 探索源码，收集真实代码片段
2. 验证每个代码片段能在项目中找到
3. 撰写博文（遵循上述原则）
4. 更新 `source/_posts/技术博客写作计划.md`：新增的独立计划模块放到"前言"下方
5. `git add -A && git commit && git push origin main`

## Git 提交格式

- 新文章：`feat: 项目名 第N篇 - 主题`
- 修改文章：`fix: 描述` 或 `docs: 描述`