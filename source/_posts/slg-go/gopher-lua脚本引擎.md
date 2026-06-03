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

## 热更新：替换脚本不重启

```go
func (e *LuaEnv) Reload(scriptPath string) error {
    e.mu.Lock()
    defer e.mu.Unlock()

    // 创建新的 Lua VM
    newVM := lua.NewState(lua.Options{SkipOpenLibs: false})
    e.registerGoFunctionsOn(newVM)

    // 加载新脚本
    if err := newVM.DoFile(scriptPath); err != nil {
        newVM.Close()
        return err
    }

    // 原子替换
    old := e.vm
    e.vm = newVM
    old.Close()
    return nil
}
```

热更新流程：
1. 策划修改 `scripts/battle.lua`
2. 调用 `/api/admin/reload` HTTP 接口
3. 服务端创建新 Lua VM，加载新脚本
4. 验证通过后原子替换旧 VM
5. 下一次战斗计算用新公式

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