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

说实话，最开始我是想用 Nakama 的。

Nakama 功能确实全——认证、好友、排行榜、实时对战、存储、Lua 脚本、多语言 SDK，该有的都有。但当我真的拿它给一个 3 人独立团队做原型时，光配 PostgreSQL + Docker Compose 就花了大半天。团队里没有专职后端，每次改个字段都要跑迁移脚本，Debug 的时候要翻三套日志。

那一刻我意识到：**Nakama 解决的问题规模和这个团队面对的问题规模不匹配。**

---

## 一个真实的踩坑经历

当时的情况是这样的：

> 团队要做一个放置类小游戏，需要排行榜、好友、简单的实时聊天。3 个人，一个策划一个美术一个程序，程序还是客户端转的，后端经验约等于零。

我推荐了 Nakama。然后：

**第 1 天**：Docker Compose 起服务，PostgreSQL 初始化，配了半天网络。程序小哥问我"为什么不用 MySQL"，我说 Nakama 要求用 PG。他说"那我本地没装 PG 怎么办"。

**第 3 天**：排行榜跑通了，但自定义字段要写 Lua 模块。小哥第一次写 Lua，语法倒是简单，但调试手段约等于零——`print` 大法，加 `log.Info` 看日志。

**第 7 天**：上游 Nakama 更新了一个 breaking change，我们的自定义模块编译不过了。花了一天查文档、看 changelog、改代码。

**第 10 天**：项目暂停，小哥说"我还是先把客户端做好吧，后端等有专职的人再说"。

这个经历让我开始想：**有没有一种方案，让没有后端经验的人也能 5 分钟跑起来？**

---

## 选型对比：不是 Nakama 不好，是不适合

我把几个候选方案做了个对比：

```
              复杂度    功能完整度    学习成本    运维成本
Nakama        ★★★★★    ★★★★★        ★★★★      ★★★★★
自研(Express) ★★       ★★            ★★        ★★
自研(Go+Redis) ★★★     ★★★★          ★★★       ★★
```

Nakama 功能最全，但复杂度和运维成本也最高。自研 Express 最简单，但功能不够。Go + Redis 是个折中点——功能够用，复杂度可控，运维简单。

最终选了 Go + Redis，原因是：

**1. 部署简单**。Redis 一行命令就起，Go 编译成单个二进制。Docker 镜像 20MB，`scp` 到服务器就能跑。对比 Nakama 的 PostgreSQL + 多容器编排，省了一半的运维心智。

**2. 学习曲线平**。Go 的语法简单，标准库够用。不用学 ORM，不用写迁移脚本，Redis 的数据结构（Hash、Sorted Set、List）天然覆盖排行榜、好友、聊天的存储需求。

**3. 源码可控**。这一点很多人忽略。Nakama 上游的 breaking change 我踩过坑，不想再踩第二次。自有代码意味着改什么、什么时候改，完全自己说了算。

---

## 纯 Redis 存储：省掉了什么？

不用 PostgreSQL 意味着省掉了：
- ORM 配置
- 数据库迁移脚本
- 备份策略设计（Redis AOF 一行配置）
- 连接池调优
- SQL 查询优化

代价是：
- 没有复杂查询能力（不能 `JOIN`、不能 `GROUP BY`）
- 数据量有上限（单机 Redis 内存限制）
- 没有事务保证（Redis 事务和 SQL 事务不是一个东西）

对于 2-10 人团队的独立游戏来说，这些代价完全可以接受。排行榜用 Sorted Set，好友关系用 Set，聊天记录用 List，KV 存储用 Hash。一个 Redis 实例搞定一切。

```go
// 排行榜：一行代码提交分数
func (s *Service) Submit(ctx context.Context, board string, score int64) error {
    key := fmt.Sprintf("lb:%s:%s", tenantID, board)
    return s.rdb.ZAdd(ctx, key, redis.Z{
        Score:  float64(score),
        Member: playerID,
    }).Err()
}

// 查询 Top 10：一行代码
func (s *Service) TopN(ctx context.Context, board string, n int64) ([]redis.Z, error) {
    key := fmt.Sprintf("lb:%s:%s", tenantID, board)
    return s.rdb.ZRevRangeWithScores(ctx, key, 0, n-1).Result()
}
```

对比 Nakama 的排行榜模块，代码量少了 10 倍。不是因为我们写得更好，是因为 Redis Sorted Set 天然就是为排行设计的。

---

## 配套 Godot SDK

后端再简单，客户端调不通也是白搭。game-bass 配套了 Godot 4.x SDK，封装了 HTTP 请求、错误处理、WebSocket 连接管理：

```gdscript
# 5 行代码完成排行榜功能
var game_bass = preload("res://addons/game-bass/game_bass.gd").new()
game_bass.connect_to_server("http://localhost:8080")
await game_bass.auth_anonymous()
await game_bass.leaderboard_submit("daily", 1500)
var top10 = await game_bass.leaderboard_top("daily", 10)
```

这个 SDK 的设计原则是：**客户端开发者不需要理解后端 API 细节**。调用一个函数，拿到结果，完事。

---

## 商业模式：卖模板，不做 SaaS

game-bass 不是 SaaS 平台，不托管服务器。商业模式很简单：

- **模板源码**（2000-5000 元/份）：买断制，拿到完整源码，自己部署
- **定制开发**（2-5 万/项目）：基于模板做功能定制

为什么不做成 SaaS？因为独立团队最怕的就是依赖第三方平台。平台一关停，后端就没了。买断源码的好处是：**代码在你手里，跑在你服务器上，谁也关不掉。**

---

## 什么时候该用 Nakama？

game-bass 不是万能的。以下场景建议直接用 Nakama：

- 团队有专职后端工程师
- 需要复杂的实时对战（如 MOBA、FPS）
- 需要多语言 SDK（Unity、Unreal、Cocos）
- 项目规模大，需要 PostgreSQL 的复杂查询能力

game-bass 适合的场景是：小团队、低实时性需求（放置、卡牌、回合制）、快速原型验证、预算有限。

---

> 这篇文章不是在说 Nakama 不好。Nakama 是一个很成熟的框架，但它解决的问题规模和 2-10 人独立团队面对的问题规模不匹配。做技术选型，匹配比"最好"更重要。