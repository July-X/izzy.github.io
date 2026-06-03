---
title: "ELO + Lua 热更新：游戏竞技匹配的双引擎设计"
date: 2025-06-04 10:30:00
tags:
  - 游戏后端
  - ELO
  - Lua
  - Go
categories:
  - 开发日志
top_img: false
---

<!-- more -->

`game-baas` 的竞技排位系统有两个核心问题要解决：**评分算法怎么选**和**怎么不重启就改算法**。解决方案是 Go 内置 ELO + Lua 热更新的双引擎架构。

## ELO 算法：经典的 K=32 实现

ELO 评分系统的核心公式：

```
Ea = 1 / (1 + 10^((Rb - Ra) / 400))
Ra_new = Ra + K * (Sa - Ea)
```

Go 实现：

```go
const kFactor = 32.0

func builtinELO(ra, rb int, outcome string) (int, int) {
    ea := 1.0 / (1.0 + math.Pow(10, float64(rb-ra)/400.0))
    eb := 1.0 - ea

    var sa, sb float64
    switch outcome {
    case "a_win":
        sa, sb = 1.0, 0.0
    case "b_win":
        sa, sb = 0.0, 1.0
    case "draw":
        sa, sb = 0.5, 0.5
    }

    return int(math.Round(kFactor * (sa - ea))),
        int(math.Round(kFactor * (sb - eb)))
}
```

返回值是双方的分数变化量（delta），不是新分数。调用方加上当前分数得到新分数。

{% mermaid %}
graph TD
    A["比赛结束"] --> B["获取双方 ELO"]
    B --> C{"Lua 脚本可用?"}
    C -->|"是"| D["Lua VM 计算 delta"]
    C -->|"否"| E["Go builtinELO 计算"]
    D --> F["Redis Pipeline 原子更新"]
    E --> F
    F --> G["更新 ELO 分"]
    F --> H["更新赛季排行榜"]
    F --> I["记录比赛历史"]
{% endmermaid %}

## 双引擎：Lua 优先，Go 兜底

为什么需要 Lua 热更新？因为游戏上线后，ELO 的 K 值、段位阈值、赛季奖励规则可能需要频繁调整。如果每次改参数都要重新编译、部署、重启服务器，运维成本太高。

```go
// competitive.go — Report 流程
func (s *Service) Report(ctx context.Context, aID, bID string, outcome string) error {
    ra := getELO(ctx, aID)  // 默认 1000
    rb := getELO(ctx, bID)

    var da, db int
    if s.luaVM != nil {
        // 优先用 Lua 脚本计算
        result, err := s.luaVM.Call("calculate", ra, rb, outcome)
        if err == nil {
            da, db = result[0], result[1]
        } else {
            // Lua 失败，回退到 Go 内置
            da, db = builtinELO(ra, rb, outcome)
        }
    } else {
        da, db = builtinELO(ra, rb, outcome)
    }

    // Redis Pipeline 原子更新
    pipe := s.rdb.Pipeline()
    pipe.ZIncrBy(ctx, eloKey(aID), float64(da))
    pipe.ZIncrBy(ctx, eloKey(bID), float64(db))
    pipe.ZIncrBy(ctx, seasonBoardKey, float64(da), aID)
    pipe.ZIncrBy(ctx, seasonBoardKey, float64(db), bID)
    _, err := pipe.Exec(ctx)
    return err
}
```

Lua 脚本放在配置目录里，运行时通过 `gopher-lua` 加载。修改脚本后调用热加载接口，下一次 `Report` 调用就用新算法。

```lua
-- scripts/elo.lua
function calculate(ra, rb, outcome)
    local k = 32  -- 可以随时改 K 值
    local ea = 1 / (1 + 10 ^ ((rb - ra) / 400))
    local eb = 1 - ea
    local sa, sb
    if outcome == "a_win" then
        sa, sb = 1, 0
    elseif outcome == "b_win" then
        sa, sb = 0, 1
    else
        sa, sb = 0.5, 0.5
    end
    return math.floor(k * (sa - ea) + 0.5), math.floor(k * (sb - eb) + 0.5)
end
```

## 天梯段位系统

五个段位，每个段位有 ELO 阈值：

```go
type Tier string
const (
    TierBronze   Tier = "bronze"    // 0-999
    TierSilver   Tier = "silver"    // 1000-1499
    TierGold     Tier = "gold"      // 1500-1999
    TierPlatinum Tier = "platinum"  // 2000-2499
    TierDiamond  Tier = "diamond"   // 2500+
)

func tierForELO(elo int) Tier {
    switch {
    case elo >= 2500:
        return TierDiamond
    case elo >= 2000:
        return TierPlatinum
    case elo >= 1500:
        return TierGold
    case elo >= 1000:
        return TierSilver
    default:
        return TierBronze
    }
}
```

段位阈值也定义在 Lua 脚本里，可以热更新。赛季初所有玩家 ELO 重置到 1000（Silver 起步），上赛季数据归档到 `season:archive:{id}` 的 Redis Key 里。

## 赛季机制

赛季用 Redis Key 管理：

```go
// 赛季排行榜 Key
func seasonBoardKey(seasonID string) string {
    return fmt.Sprintf("competitive:season:%s:board", seasonID)
}

// 赛季重置：归档旧赛季，创建新赛季
func (s *Service) ResetSeason(ctx context.Context) error {
    oldSeason := s.currentSeason
    // 1. 归档旧赛季排行榜
    s.rdb.Rename(ctx, seasonBoardKey(oldSeason), seasonArchiveKey(oldSeason))
    // 2. 创建新赛季
    s.currentSeason = generateNewSeasonID()
    return nil
}
```

## 比赛历史

每场比赛双方各记录一条历史，保留最近 50 场：

```go
func (s *Service) recordMatch(ctx context.Context, playerID string, match MatchRecord) error {
    key := fmt.Sprintf("competitive:history:%s", playerID)
    data, _ := json.Marshal(match)
    s.rdb.LPush(ctx, key, data)
    s.rdb.LTrim(ctx, key, 0, 49)  // 保留最近 50 场
    return nil
}
```

用 Redis List 的 `LPush` + `LTrim` 组合，新记录插到头部，超出 50 条的尾部自动丢弃。

## 为什么 K=32？

K 值决定每场比赛分数变化的幅度：

| K 值 | 特点 | 适用场景 |
|------|------|----------|
| 16 | 变化小，收敛慢 | 职业联赛，需要精确排名 |
| 32 | 变化适中 | 大众竞技，平衡速度和精度 |
| 48 | 变化大，波动强 | 新赛季初期，快速拉开差距 |

game-baas 默认 K=32，通过 Lua 脚本可以随时调整。某些游戏会在赛季前 10 场用 K=48 快速定位，之后切回 K=32。

> ELO 算法的核心不是"精确"，是"公平"。K=32 + Lua 热更新 = 快速上线 + 持续迭代。