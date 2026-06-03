---
title: "不做 Nakama：从零搭建游戏后端模板的原因"
date: 2025-06-04 10:00:00
tags:
  - 游戏后端
  - Go
  - Nakama
  - 架构
categories:
  - 开发日志
top_img: false
---

<!-- more -->

`game-baas` 的定位很直接：面向 2-10 人独立游戏团队的游戏后端模板商店。买模板 5 分钟部署，Docker 一键启动，Godot SDK 即插即用。很多人问为什么不用 Nakama，答案是：**Nakama 太重了，而我们的目标用户不需要那么重的东西。**

## Nakama 好在哪，重在哪

Nakama 是 Heroic Labs 开源的游戏服务器框架，功能全面：认证、好友、排行榜、实时对战、存储、Lua 脚本、多语言 SDK。但对独立团队来说有几个痛点：

| 痛点 | Nakama | game-baas |
|------|--------|-----------|
| 数据库依赖 | PostgreSQL（必须） | 纯 Redis（零 SQL） |
| 启动复杂度 | Docker Compose 多容器 | 单容器 + Redis |
| 学习成本 | Go + Lua + 自定义模块 | Go + Lua，接口更少 |
| 运维门槛 | PostgreSQL 备份/迁移/调优 | Redis AOF，一行配置 |
| 源码可控 | 上游不稳定的 breaking change | 自有代码，完全可控 |

核心矛盾：Nakama 是为中大型游戏团队设计的，它的 PostgreSQL 依赖、复杂的模块系统、多语言 Runtime 对独立团队来说是负担而非能力。

## game-baas 的架构选择

### 纯 Redis 存储

所有数据（玩家、排行榜、好友、聊天记录）都存在 Redis 里。不用 SQL 数据库，不用 ORM，不用迁移脚本。

```go
// internal/leaderboard/leaderboard.go
// 排行榜用 Redis Sorted Set，天然有序
func (s *Service) Submit(ctx context.Context, board string, score int64) error {
    key := fmt.Sprintf("lb:%s:%s", tenantID, board)
    return s.rdb.ZAdd(ctx, key, redis.Z{
        Score:  float64(score),
        Member: playerID,
    }).Err()
}
```

Redis Sorted Set 天然支持排行查询（`ZREVRANGE`）、分数更新（`ZINCRBY`）、范围查询（`ZRANGEBYSCORE`），一个数据结构覆盖排行榜的全部需求。

### Go 语言的务实选择

Go 不是"最优雅"的语言，但对游戏后端来说是最务实的：

1. **编译成单个二进制**：Docker 镜像只有 ~20MB，部署只需 `scp` 一个文件
2. **goroutine 天然适合并发**：每个 WebSocket 连接一个 goroutine，比 Node.js 的事件循环更直观
3. **标准库足够强**：HTTP、JSON、加密、模板都在标准库里，不需要第三方依赖
4. **交叉编译简单**：`GOOS=linux GOARCH=amd64 go build` 一行出 Linux 二进制

### 模块化但不过度

{% mermaid %}
graph TD
    A["cmd/server"] --> B["router"]
    B --> C["auth"]
    B --> D["player"]
    B --> E["leaderboard"]
    B --> F["chat"]
    B --> G["competitive"]
    B --> H["idle"]
    B --> I["realtime"]
    C --> J["Redis"]
    D --> J
    E --> J
    F --> J
    I --> K["WebSocket"]
{% endmermaid %}

每个模块是 `internal/` 下的一个 Go 包，通过 `router/` 统一注册 HTTP 端点。模块之间不互相依赖，只通过 `core/` 包定义的接口通信。

```go
// core/core.go — 定义 Handler 签名
type Handler func(ctx context.Context, p *Payload) (interface{}, error)

// Tenant/Player context 注入
type ctxKey struct{}
func WithTenant(ctx context.Context, t *Tenant) context.Context { ... }
func TenantFrom(ctx context.Context) *Tenant { ... }
```

15+ 个 API 端点，每个端点一个 Handler 函数，没有中间件链、没有 DI 框架、没有代码生成。

## 配套 Godot SDK

独立团队最缺的不是后端代码，是前后端联调的时间。game-baas 配套了 Godot 4.x SDK，开发者在 GDScript 里直接调用：

```gdscript
# addons/game-baas/game_baas.gd
func leaderboard_submit(board: String, score: int) -> void:
    var result = await _http_post("/api/leaderboard/submit", {
        "board": board,
        "score": score
    })
```

SDK 封装了 HTTP 请求、错误处理、WebSocket 连接管理，开发者不需要理解后端 API 细节。

## 商业模式：卖模板，不做 SaaS

game-baas 不是 SaaS 平台，不托管服务器。商业模式是：

- **模板源码**（2000-5000 元/份）：买断制，拿到完整源码，自己部署
- **定制开发**（2-5 万/项目）：基于模板做功能定制

这个模式的好处：用户有完全的代码控制权，不会因为平台关停而丢失后端。

## 和 Nakama 的关系

game-baas 不是 Nakama 的竞品，是 Nakama 的"轻量替代方案"。适合的场景：

- 2-10 人独立团队，没有专职后端工程师
- 游戏类型是放置、卡牌、回合制等低实时性需求
- 需要快速原型验证，不想花 2 周配置 PostgreSQL
- 预算有限，买不起商业游戏后端服务

> 做技术选型不是选"最好的"，是选"最匹配当前阶段的"。Nakama 很好，但它解决的问题规模和独立团队面对的问题规模不匹配。