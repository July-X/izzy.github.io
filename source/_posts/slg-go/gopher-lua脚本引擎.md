---
title: "gopher-lua：不重启就改游戏逻辑的脚本引擎"
date: 2025-06-04 14:30:00
tags:
  - 游戏后端
  - Go
  - Lua
  - 热更新
categories:
  - 开发日志
top_img: false
---

<!-- more -->

SLG 游戏的策划需求变化频繁——调整建筑升级时间、修改资源产出比例、改战斗公式。如果每次改动都要重新编译部署 Go 服务，策划会疯，运维也会疯。`slg-go` 用 `gopher-lua` 嵌入 Lua 脚本引擎，让游戏逻辑可以热更新。

## 为什么选 Lua？

| 语言 | 嵌入难度 | 性能 | 热更新 | 生态 |
|------|----------|------|--------|------|
| Lua | 低（gopher-lua） | 中 | 原生支持 | 游戏行业标准 |
| JavaScript | 中（goja） | 中高 | 需额外机制 | Web 生态 |
| Python | 高（嵌入 CPython） | 低 | 受 GIL 限制 | ML 生态 |

Lua 天生为嵌入设计，`gopher-lua` 是纯 Go 实现，不依赖 CGO，交叉编译无障碍。

## 集成方式

```go
// internal/luaenv/loader.go
type LuaEnv struct {
    vm *lua.LState
    mu sync.RWMutex
}

func New() *LuaEnv {
    L := lua.NewState(lua.Options{SkipOpenLibs: false})
    env := &LuaEnv{vm: L}
    env.registerGoFunctions()
    return env
}

func (e *LuaEnv) registerGoFunctions() {
    // 注册 Go 函数到 Lua 全局表
    e.vm.SetGlobal("go_log", e.vm.NewFunction(func(L *lua.LState) int {
        msg := L.CheckString(1)
        log.Info("[LUA]", msg)
        return 0
    }))
}
```

## 热更新：看起来简单，实际上坑很多

上面的 `Reload` 代码是简化版。实际生产环境用这个版本会出问题——我踩过。

### 问题 1：gopher-lua 的 LState 不是线程安全的

`gopher-lua` 的 `LState`（Lua 虚拟机实例）**不是并发安全的**。一个 `LState` 同一时刻只能被一个 goroutine 使用。如果 10 个战斗请求同时调用同一个 `LState`，会 panic 或数据错乱。

最开始的方案是加 `sync.RWMutex`，读请求用 `RLock`，热更新用 `Lock`：

```go
// ❌ 最初的方案：看起来没问题，实际有坑
func (e *LuaEnv) CalculateDamage(attacker, defender *Unit) int {
    e.mu.RLock()
    defer e.mu.RUnlock()

    L := e.vm
    fn := L.GetGlobal("calculate_damage")
    L.Push(fn)
    // ... 调用 Lua 函数
}
```

坑在哪？`RLock` 允许多个 goroutine 同时读，但 `LState` 不支持并发读。10 个 goroutine 同时 `RLock` 成功，然后同时操作同一个 `LState`，直接炸。

### 问题 2：正在执行中的请求怎么办？

热更新的瞬间，可能有 20 个战斗请求正在旧 VM 里跑。如果直接 `old.Close()`，这些请求会全部崩溃。

### 最终方案：VM 池 + 引用计数 + 优雅退役

```go
type LuaEnv struct {
    current  atomic.Pointer[LuaPool]  // 当前使用的 VM 池
    mu       sync.Mutex               // 仅 Reload 时用
}

type LuaPool struct {
    vms    []*lua.LState
    idx    uint64                     // 原子递增，轮询分配
    refCnt atomic.Int64               // 引用计数
    closed atomic.Bool                // 标记为退役
}
```

每次调用从池中取一个 VM，用完归还。这样多个请求可以并发执行，互不干扰：

```go
func (e *LuaEnv) acquire() *lua.LState {
    pool := e.current.Load()
    pool.refCnt.Add(1)
    i := pool.idx.Add(1) % uint64(len(pool.vms))
    return pool.vms[i]
}

func (e *LuaEnv) release(pool *LuaPool) {
    n := pool.refCnt.Add(-1)
    // 如果池已退役且引用归零，安全销毁
    if n <= 0 && pool.closed.Load() {
        for _, vm := range pool.vms {
            vm.Close()
        }
    }
}
```

### Reload 的正确流程

```go
func (e *LuaEnv) Reload(scriptPath string) error {
    e.mu.Lock()
    defer e.mu.Unlock()

    // 1. 先验证新脚本语法
    testVM := lua.NewState()
    if err := testVM.DoFile(scriptPath); err != nil {
        testVM.Close()
        return fmt.Errorf("脚本语法错误: %w", err)
    }
    testVM.Close()

    // 2. 创建新的 VM 池（4 个 VM 实例）
    newPool := &LuaPool{vms: make([]*lua.LState, 4)}
    for i := range newPool.vms {
        vm := lua.NewState()
        e.registerGoFunctionsOn(vm)
        if err := vm.DoFile(scriptPath); err != nil {
            // 回滚：关闭已创建的 VM
            for j := 0; j <= i; j++ {
                newPool.vms[j].Close()
            }
            return err
        }
        newPool.vms[i] = vm
    }

    // 3. 原子切换：新请求开始用新 VM
    oldPool := e.current.Swap(newPool)

    // 4. 标记旧池退役
    oldPool.closed.Store(true)

    // 5. 等待旧池引用归零后销毁
    go func() {
        for oldPool.refCnt.Load() > 0 {
            time.Sleep(10 * time.Millisecond)
        }
        for _, vm := range oldPool.vms {
            vm.Close()
        }
        log.Info("[LUA] 旧 VM 池已安全销毁")
    }()

    log.Infof("[LUA] 热更新完成: %s", scriptPath)
    return nil
}
```

### 热更新的时间线

```
T+0s     策划调用 /api/admin/reload
T+0.01s  新脚本语法验证通过
T+0.05s  4 个新 VM 加载完成
T+0.05s  atomic.Swap 切换到新 VM 池
T+0.05s  新请求开始用新 VM
T+0.05s  旧 VM 池标记为退役
T+0.5s   旧 VM 池中最后一个请求完成
T+0.5s   旧 VM 池安全销毁
```

整个过程**零停机**，正在执行的请求用旧公式跑完，新请求用新公式。

### 还有一个坑：脚本里的全局状态

Lua 脚本里如果定义了全局变量（比如 `local counter = 0` 在模块级别），热更新后这个状态会丢失。

```lua
-- ❌ 有问题的写法
local kill_count = 0  -- 模块级全局变量

function on_kill()
    kill_count = kill_count + 1
    if kill_count >= 10 then
        trigger_achievement("first_blood")
    end
end
```

热更新后 `kill_count` 重置为 0，玩家已经杀了 9 个人，再杀一个不触发成就。

解决方案：**有状态的逻辑不放 Lua，只放无状态的计算逻辑**。`kill_count` 这种状态留在 Go 端管理，Lua 只负责算伤害、算时间、算公式。

```
Lua 负责的（无状态）：
  - calculate_damage(attacker, defender) → 数字
  - building_upgrade_time(level, type) → 数字
  - resource_output_rate(building_level) → 数字

Go 负责的（有状态）：
  - 玩家数据、成就计数、任务进度
  - 战斗中的临时状态（连杀、Buff 持续时间）
  - 任何需要跨请求持久化的数据
```

### 快速回滚

如果新脚本有 Bug（比如公式算错了），需要立刻回滚。保留旧脚本的路径，一行命令切回去：

```bash
# 回滚到上一个版本
curl -X POST http://localhost:8080/api/admin/reload \
  -d '{"path": "scripts/battle.lua.bak"}'
```

`Reload` 里的 `atomic.Swap` 保证回滚也是原子的，正在执行的请求不受影响。

## 战斗公式脚本化

```lua
-- scripts/battle.lua
function calculate_damage(attacker, defender)
    local base_damage = attacker.attack * (1 - defender.defense / (defender.defense + 100))
    local crit_chance = attacker.crit_rate / 100
    local is_crit = math.random() < crit_chance
    if is_crit then
        base_damage = base_damage * attacker.crit_damage
    end
    return math.floor(base_damage + 0.5)
end

-- 建筑升级时间（秒）
function building_upgrade_time(level, building_type)
    local base_time = 60
    local multiplier = 1.5
    if building_type == "castle" then
        multiplier = 2.0  -- 城堡升级更慢
    end
    return math.floor(base_time * multiplier ^ (level - 1))
end
```

Go 端调用 Lua 函数：

```go
func (e *LuaEnv) CalculateDamage(attacker, defender *Unit) int {
    e.mu.RLock()
    defer e.mu.RUnlock()

    L := e.vm
    fn := L.GetGlobal("calculate_damage")
    L.Push(fn)
    L.Push(lua.LNumber(attacker.Attack))
    L.Push(lua.LNumber(attacker.Defense))
    L.Push(lua.LNumber(attacker.CritRate))
    L.Push(lua.LNumber(attacker.CritDamage))
    L.Push(lua.LNumber(defender.Defense))
    L.Call(5, 1)
    return int(L.CheckNumber(-1))
}
```

## 为什么不用配置文件？

JSON/YAML 配置文件只能存参数，不能存逻辑。SLG 的战斗公式、建筑规则、科技树依赖关系是**逻辑**，不是参数。

```
配置文件能做的：攻击力 = 100
配置文件做不到的：如果目标有护盾，先扣护盾再扣血；护盾破碎时触发眩晕效果
```

Lua 脚本可以表达条件判断、循环、函数调用，覆盖 SLG 游戏的全部逻辑需求。

## 性能考量

`gopher-lua` 是纯 Go 实现的 Lua 5.1 解释器，性能约为 C Lua 的 1/10~1/5。但对于 SLG 游戏来说足够：

- 战斗计算频率低（每秒最多几十次）
- 单次计算量小（几十行 Lua 代码）
- 瓶颈在数据库 IO，不在脚本计算

如果未来性能不够，可以：
1. 热点函数用 Go 实现，只把参数调整留给 Lua
2. 缓存 Lua 函数调用结果
3. 升级到 LuaJIT（需要 CGO）

> 热更新的价值不是"省一次部署"，是"让策划能独立迭代"。策划改了脚本立刻生效，不需要等程序员排期。