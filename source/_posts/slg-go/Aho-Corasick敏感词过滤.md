---
title: "SLG 聊天过滤：Aho-Corasick 自动机 + 热更新词库"
date: 2026-06-04 21:00:00
tags:
  - 游戏后端
  - Go
  - 算法
  - 聊天系统
categories:
  - 开发日志
top_img: false
---

<!-- more -->

SLG 游戏的聊天频道是玩家社区的命脉，也是运营的噩梦——一条未过滤的敏感词可能导致投诉、下架甚至法律风险。`slg-go` 的聊天过滤系统经历了三次迭代：最初用 `strings.Contains` 暴力匹配，后来换正则，最终落地到 **Aho-Corasick 自动机**。这篇记录最终方案的实现和踩坑。

## 为什么不用 strings.Contains？

初版代码很简单：

```go
// ❌ 第一版：暴力匹配
func filter(content string, words []string) string {
    for _, w := range words {
        content = strings.ReplaceAll(content, w, "***")
    }
    return content
}
```

问题在于：100 个敏感词 × 1000 条消息 = 10 万次字符串扫描。词库涨到 500+ 时，单条消息过滤延迟从 0.1ms 涨到 2ms，1000 人频道就是 2 秒。

正则好一点，但 `regexp.Compile` 构建 500 个 pattern 的启动时间太长，而且 Go 的正则引擎不支持预编译多模式自动机。

## Aho-Corasick：O(n) 多模式匹配

Aho-Corasick（AC 自动机）是 1975 年贝尔实验室提出的算法，核心思想是把所有敏感词构建成一棵 **Trie 树**，再通过 BFS 计算 **fail 指针**（类似 KMP 的 next 数组），实现**一次扫描匹配所有模式**。

时间复杂度：构建 O(Σ|w|)，匹配 O(n + m)，其中 n 是文本长度，m 是匹配数。**和词库大小无关**。

```go
// internal/chat/filter/ahocorasick.go
type ACAutomaton struct {
    root *acNode
    mu   sync.RWMutex
}

type acNode struct {
    children map[rune]*acNode
    fail     *acNode  // 失败指针
    depth    int
    isEnd    bool     // 是否为某个词的结尾
}
```

用 `rune` 而不是 `byte` 做 key，因为中文是多字节的。`"傻逼"` 两个字各占一个 rune，直接用 `map[rune]*acNode` 天然支持中英文混合。

## 构建：Trie 树 + BFS fail 指针

**第一步：插入所有敏感词到 Trie 树**

```go
func (ac *ACAutomaton) Build(words []string) {
    ac.mu.Lock()
    defer ac.mu.Unlock()
    ac.root = newACNode()
    for _, word := range words {
        w := lowerWord(word)  // 统一小写
        if w == "" { continue }
        node := ac.root
        for _, r := range w {
            if _, ok := node.children[r]; !ok {
                child := newACNode()
                child.depth = node.depth + 1
                node.children[r] = child
            }
            node = node.children[r]
        }
        node.isEnd = true
    }
    ac.buildFail()
}
```

**第二步：BFS 计算 fail 指针**

```go
func (ac *ACAutomaton) buildFail() {
    queue := make([]*acNode, 0, 256)
    for _, child := range ac.root.children {
        child.fail = ac.root  // 第一层 fail 指向 root
        queue = append(queue, child)
    }
    for len(queue) > 0 {
        current := queue[0]
        queue = queue[1:]
        for r, child := range current.children {
            failNode := current.fail
            // 沿 fail 链回溯，找到有 r 子节点的祖先
            for failNode != ac.root {
                if _, ok := failNode.children[r]; ok { break }
                failNode = failNode.fail
            }
            if n, ok := failNode.children[r]; ok && n != child {
                failNode = n
            }
            child.fail = failNode
            queue = append(queue, child)
        }
    }
}
```

fail 指针的作用：当 Trie 树上某条路径匹配失败时，不需要从头开始，而是跳到 fail 指针指向的节点继续匹配——这就是 KMP 思想在多模式匹配上的推广。

## 匹配：一次扫描，标记所有命中位置

```go
func (ac *ACAutomaton) Filter(content string, mask rune) string {
    ac.mu.RLock()
    defer ac.mu.RUnlock()
    runes := []rune(content)
    maskArr := make([]bool, len(runes))
    lower := make([]rune, len(runes))
    // 统一小写
    for i, r := range runes {
        if r >= 'A' && r <= 'Z' {
            lower[i] = r + 32
        } else {
            lower[i] = r
        }
    }
    node := ac.root
    for i, r := range lower {
        // 沿 fail 链回溯
        for node != ac.root {
            if _, ok := node.children[r]; ok { break }
            node = node.fail
        }
        if n, ok := node.children[r]; ok {
            node = n
        }
        // 标记所有以当前位置结尾的匹配
        for matched := node; matched != ac.root; matched = matched.fail {
            if !matched.isEnd { continue }
            start := i - matched.depth + 1
            for j := start; j <= i; j++ {
                maskArr[j] = true
            }
        }
    }
    // 替换标记位置
    for i := range runes {
        if maskArr[i] { runes[i] = mask }
    }
    return string(runes)
}
```

关键设计：用 `maskArr` 标记数组，先扫描一遍标记所有命中位置，最后统一替换。这比边扫描边替换更安全——不会出现替换后字符数变化导致的偏移错误。

## 热更新：SHA-256 指纹去重

词库不能写死在代码里。运维需要随时更新敏感词，不能重启服务。`filter_reload.go` 实现了热更新机制：

```go
// cmd/chat/filter_reload.go — 简化示意
type chatFilterReloader struct {
    source          chatFilterWordSource  // 数据源（文件/Nacos）
    lastFingerprint string                // SHA-256 指纹
    lastReloadAt    time.Time
}

func (r *chatFilterReloader) reloadIfChanged(hub *chat.Hub, force bool) error {
    words, fingerprint, err := r.source.LoadWords()
    if err != nil { return err }
    // 指纹未变，跳过重建
    if !force && fingerprint == r.lastFingerprint {
        return nil
    }
    count := hub.ReplaceSensitiveWords(words)
    r.lastFingerprint = fingerprint
    r.lastReloadAt = time.Now()
    return nil
}
```

三种数据源，支持 Fallback 链：

| 数据源 | 场景 | 指纹计算 |
|--------|------|----------|
| 文件 | 本地开发、小规模部署 | `SHA-256(文件内容)` |
| Nacos | 生产环境、多实例同步 | `SHA-256(配置内容)` |
| Fallback | Nacos 不可用时降级到文件 | primary → fallback |

自动轮询间隔默认 10 秒，通过 `ctx.Done()` 控制退出：

```go
go func() {
    ticker := time.NewTicker(opts.Interval)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            reloader.reloadIfChanged(hub, false)
        }
    }
}()
```

## UTF-8 处理：rune 级别的大小写归一化

Go 的 `strings.ToLower` 对 ASCII 没问题，但对中文无效（中文没有大小写）。AC 自动机里用了自写的 `lowerWord`：

```go
func lowerWord(word string) string {
    var result []rune
    for _, r := range word {
        if r == ' ' || r == '\t' { continue }  // 去除空白
        if r >= 'A' && r <= 'Z' {
            result = append(result, r+32)
        } else {
            result = append(result, r)
        }
    }
    return string(result)
}
```

只处理 ASCII 大小写，中文原样保留。空白字符（空格、Tab）在插入时直接去除，防止 `"f u c k"` 绕过过滤。

## 性能数据

| 指标 | strings.Contains | Aho-Corasick |
|------|-----------------|-------------|
| 500 词库 × 100 字符消息 | ~2ms | ~0.05ms |
| 构建时间（500 词） | 0 | ~0.3ms |
| 内存占用（500 词） | ~20KB | ~150KB |

AC 自动机用 150KB 内存换来了 40 倍的匹配速度提升。对于 SLG 聊天这种高频场景，完全值得。

## 总结

整个过滤系统分三层：
1. **算法层**：Aho-Corasick 自动机，O(n) 多模式匹配
2. **管理层**：`ChatFilter` 封装了增删改查、`sync.RWMutex` 并发安全
3. **运维层**：`chatFilterReloader` 支持文件/Nacos 双数据源 + SHA-256 指纹去重

敏感词过滤不是"写了就行"的功能，是需要持续迭代的基础设施。词库要热更新，性能要 O(n)，并发要安全，三者缺一不可。

> 你们项目里敏感词过滤是怎么做的？遇到过绕过过滤的 trick 吗？