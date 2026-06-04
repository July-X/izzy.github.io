---
title: Go 内存模型：Happens-Before 规则与数据竞争实战
date: 2026-06-04
tags:
  - Golang
  - 内存模型
  - 并发安全
  - happens-before
categories: Golang学习
---

> 上一篇聊了 Channel 的底层实现——环形队列 + 等待队列 + 全局锁。但光知道 channel 怎么工作还不够，你还需要知道：**两个 goroutine 对同一个变量的读写，到底什么时候是安全的？** 这就是 Go 内存模型（Go Memory Model）要回答的问题。

<!-- more -->

## 背景：一个真实的数据竞争

在 slg-go 战斗引擎的开发过程中，我们遇到过这样一个潜在问题：

```go
// internal/battle/engine/engine.go:85-95 — BattleEngine 结构体
type BattleEngine struct {
    troopAttrs     map[uint32]*TroopAttr    // 只读，初始化后不变
    terrainMods    map[uint32]map[string]float64 // 只读

    lua            *luaBridge               // 可变

    scriptMu       sync.RWMutex             // 保护以下三个字段
    scriptCurrent  ScriptInfo
    scriptAudits   []ScriptAuditLog
    scriptVersions map[string]string
}
```

**来源**：`internal/battle/engine/engine.go:85-95`

`Execute()` 方法被多个 worker goroutine **并发调用**（scheduler 启动了 N 个 worker），它们同时读 `troopAttrs` 和 `terrainMods`。这两个 map 在 `NewEngine()` 初始化后**再也不会修改**。这安全吗？

答案是：**安全，但你必须理解为什么安全。** 如果哪天有人给 `troopAttrs` 加了一个"热更新兵种属性"的功能，直接写而不加锁，就会引入数据竞争（data race）。这种 bug 不一定会立刻 crash，但它会导致**不可复现的奇怪行为**——这才是最危险的。

---

## Q1：什么是 Happens-Before？

先说结论：

> **如果事件 A "happens before" 事件 B，那么 A 的内存修改对 B 可见。**

反过来，如果 A 和 B 之间**不存在 happens-before 关系**，编译器和 CPU 可以**自由重排序**它们的读写操作。这就是数据竞争的根源。

Go 内存模型保证了一组 happens-before 规则。只要你的代码遵循这些规则，就不需要额外的同步原语。

### 核心直觉

```
Goroutine 1:          Goroutine 2:
    write(x, 1)            y := read(y)    ← 能读到 1 吗？
                       x := read(x)

❓ 问：Goroutine 2 能读到 x=1 吗？

✅ 答：只有当 write(x,1) "happens before" read(x) 时才能保证读到。
   否则可能读到 0、读到 1、甚至读到"半写"的值（虽然 Go 里基本类型不会撕裂）。
```

---

## Q2：Go 的 Happens-Before 规则全集

Go 规范定义了以下 happens-before 关系（按使用频率排列）：

### 规则 1：Goroutine 创建

```go
// 主 goroutine 启动子 goroutine
go func() {
    // 这里能安全地读取主 goroutine 在 go 语句之前的所有写入
    println(config.Port)  // ✅ 安全
}()

config.Port = 8080  // 这个写入 happens-before 子 goroutine 启动
```

**slg-go 实例**：

```go
// scheduler.go:195-200 — Start 创建 N 个 worker
func (s *Scheduler) Start(ctx context.Context) error {
    for i := 0; i < s.workers; i++ {
        s.wg.Add(1)
        go s.worker(ctx, i)   // s.requests, s.engine 等字段对 worker 可见
    }
}
```

**来源**：`scheduler.go:195-200`

`Start()` 中 `s.requests` 已经创建完毕，`go s.worker()` 之后 worker 能安全看到这个 channel。这是规则 1 的典型应用。

### 规则 2：Channel 操作（最重要的一组）

Channel 是 Go 内存模型中最核心的同步原语：

| 操作 | Happens-Before 关系 |
|------|---------------------|
| **发送** → **接收完成** | 发送方的发送 happens-before 接收方接收完成 |
| **接收** → **发送完成** | 接收方的接收 happens-before 发送方发送完成 |
| **关闭 Channel** | close happens-before 从已关闭 channel 接收返回零值 |

```go
// 最经典的 channel 同步模式
var ch = make(chan int, 1)

// G1
ch <- 42        // (1) send

// G2
v := <-ch       // (2) recv
// 此时 (1) happens-before (2)，v 一定是 42
```

**无缓冲 channel 的特殊语义**：

```go
ch := make(chan int)  // 无缓冲

// G1
ch <- 42      // 阻塞直到 G2 接收

// G2
v := <-ch     // 阻塞直到 G1 发送
```

无缓冲 channel 的发送和接收是一个**原子握手事件**——发送完成和接收完成的 happens-before 关系是双向的。

### 规则 3：Lock / Unlock

```go
// slg-go scheduler.go:407-429 — finishBattle 中的锁用法
func (s *Scheduler) finishBattle(req *BattleRequest, result *engine.BattleResult) {
    var waiters []chan *engine.BattleResult

    s.mu.Lock()                          // (1) Lock
    state, ok := s.states[req.BattleID]
    if ok {
        state.done = true
        state.result = result            // (2) 写入共享状态
        waiters = append(waiters, state.waiters...)
        state.waiters = nil
    }
    inflight := s.countInflightLocked()
    s.mu.Unlock()                        // (3) Unlock

    // (3)Unlock happens-before 后续所有对 result 的读取
    for _, ch := range waiters {
        select {
        case ch <- result:               // (4) 通过 channel 发送 result
        default:
        }
    }
}
```

**来源**：`scheduler.go:407-429`

这里有两层 happens-before：
1. `(2)` 写入 `state.result` happens-before `(3)` Unlock（同一次 lock 内）
2. `(3)` Unlock happens-before 另一个 goroutine 的 Lock 成功后读取到新值

然后通过 `ch <- result` 把 happens-before 链延伸到了等待结果的 goroutine。

### 规则 4：Sync 包中的其他原语

| 原语 | Happens-Before 保证 |
|------|---------------------|
| `sync.Once.Do(f)` | f 的返回 happens-before 所有 Do 调用的返回 |
| `atomic.Value.Store/Load` | Store happens-before Load 返回新值 |
| `WaitGroup.Wait()` | Add/Done 的最后一次 Done happens-before Wait 返回 |

**`sync.Once` 在 slg-go 中的应用**：

```go
// scheduler.go:204-213 — Stop 只执行一次
func (s *Scheduler) Stop() {
    s.stopOnce.Do(func() {       // sync.Once 保证闭包只执行一次
        close(s.stopCh)
        s.wg.Wait()              // 等待所有 worker 退出
        close(s.requests)
    })
}
```

**来源**：`scheduler.go:204-213`

即使 `Stop()` 被多个 goroutine 同时调用，`stopOnce.Do()` 保证里面的代码只执行一次，且执行完之后，所有后续的 `Do()` 调用都能感知到效果。

---

## Q3：哪些写法看似安全其实不安全？

### ❌ 用 time.Sleep 代替同步

```go
// 错误！Sleep 不能建立 happens-before
go func() {
    data = []int{1, 2, 3}  // G1 写入
}()

time.Sleep(100 * time.Millisecond)  // 希望等 G1 写完
println(data[0])  // ❌ 编译器/CPU 可能重排序，不一定能看到更新
```

`Sleep` 只是让当前 goroutine 休眠，它**不建立任何 happens-before 关系**。另一个 goroutine 的写入对你来说可能不可见。

### ❌ 用布尔标志做信号（没有 atomic 或 channel）

```go
var ready bool

go func() {
    heavyInit()
    ready = true   // G1: 设置标志
}()

for !ready {}      // G2: 自旋等待  ⚠️ 数据竞争！
println(data)
```

问题：
1. **编译器优化**：编译器发现循环里 `ready` 没有被当前 goroutine 修改，可能把 `for !ready {}` 优化成死循环或直接缓存值
2. **CPU 重排序**：CPU 可能乱序执行 `ready = true` 和 `heavyInit()` 内部的写入
3. 即使碰巧能跑通，也是未定义行为

**正确做法**：

```go
// 方式一：channel（推荐）
ready := make(chan struct{})
go func() {
    heavyInit()
    close(ready)   // close happens-before 接收方返回
}()
<-ready           // 阻塞直到 ready 被关闭
println(data)

// 方式二：atomic
var ready atomic.Bool
go func() {
    heavyInit()
    ready.Store(true)   // atomic.Store 有 release 语义
}()
for !ready.Load() {}    // atomic.Load 有 acquire 语义
println(data)
```

---

## Q4：RWMutex 的 Happens-Before 详解

slg-go 中 RWMutex 用得最多的地方是 `engine.go` 和 `player_service.go`：

```go
// engine.go:489-493 — 读多写少的经典场景
func (e *BattleEngine) CurrentScript() ScriptInfo {
    e.scriptMu.RLock()         // 获取读锁
    defer e.scriptMu.RUnlock()
    return e.scriptCurrent     // 安全读取共享状态
}

// engine.go:397-441 — 写操作
func (e *BattleEngine) ReloadScriptWithVersion(...) (*ScriptAuditLog, error) {
    e.scriptMu.Lock()          // 获取写锁（独占）
    defer e.scriptMu.Unlock()
    // ... 修改 scriptCurrent, scriptAudits, scriptVersions ...
}
```

**来源**：`engine.go:397-493`

RLock/RUnlock 之间的 happens-before 关系：

```
Writer (goroutine W):        Reader (goroutine R):
    Lock()
    write(scriptCurrent)          
    write(scriptAudits)           
    Unlock()  ──────h.b.────►  RLock()
                                  read(scriptCurrent)   // ✅ 能看到 W 的写入
                                  read(scriptAudits)    // ✅ 能看到 W 的写入
                               RUnlock()
```

**关键细节**：RWMutex 的写锁释放（Unlock）和后续的读锁获取（RLock）之间存在 happens-before 关系。所以 writer 的所有修改对 reader 都可见。

但如果 reader 先拿到了 RLock，writer 还没 Unlock 呢？reader 看到的是**旧数据**——这在逻辑上完全正确（脚本还没更新嘛）。RWMutex 不保证你总是读到最新值，它只保证你不会读到**半写（torn read）**的状态。

---

## Q5：如何检测数据竞争？

Go 提供了内置的数据竞争检测器——`-race` 标志：

```bash
# 运行时开启竞争检测
go test -race ./...
go run -race main.go

# 甚至可以加到构建中
go build -race -o server ./cmd/gate/
```

开启 `-race` 后，运行时会监控所有内存访问，一旦检测到两个 goroutine 无同步地访问同一个变量（至少一个是写），就报告 data race。

### 典型输出示例

```
==================
WARNING: DATA RACE
Read at 0x00c0000a6018 by goroutine 8:
  internal/battle/engine.(*BattleEngine).Execute()
      /path/to/engine.go:129 +0x45

Previous write at 0x00c0000a6018 by goroutine 7:
  internal/battle/engine.(*BattleEngine).ReloadScriptWithVersion()
      /path/to/engine.go:424 +0x89

==================
```

**生产建议**：
- CI 流程中**必须包含 `-race` 测试**
- 性能测试时不要开 race（开销约 5-10x）
- staging 环境建议定期跑 race 检测的集成测试

---

## Q6：slg-go 中的同步设计总结

回顾整个项目的并发设计，可以看到清晰的分层：

{% mermaid %}
flowchart TB
    subgraph "通信层 — Channel"
        A["requests chan<br/>任务队列"]
        B["stopCh chan struct{}<br/>停止信号"]
        C["resultChan chan Result<br/>结果回传"]
    end

    subgraph "同步层 — Mutex"
        D["sync.RWMutex<br/>states map 保护"]
        E["sync.RWMutex<br/>script 状态保护"]
    end

    subgraph "幂等层 — Once/Atomic"
        F["sync.Once<br/>Stop 幂等"]
        G["atomic/int64<br/>metrics 计数器"]
    end

    A --> D
    C --> D
    B --> F
    D --> G
    E --> G
{% endmermaid %}

| 层级 | 原语 | slg-go 用例 | 职责 |
|------|------|------------|------|
| **通信** | buffered channel | `requests`, `msgCh` | 跨 goroutine 传递数据和所有权 |
| **通知** | unbuffered/nil channel | `stopCh`, `ctx.Done()` | 生命周期管理、广播退出 |
| **互斥** | sync.RWMutex | `mu`(scheduler), `scriptMu`(engine) | 保护共享可变状态 |
| **原子** | sync.Once / atomic | `stopOnce`, metrics | 幂等操作、高频计数器 |

**来源**：综合 `scheduler.go`, `engine.go`, `ws.go`

这套组合拳的核心原则：**能用 channel 通信就不要共享内存；必须共享内存时用 mutex 保护；计数类用 atomic；一次性操作用 Once。**

---

## 总结

Go 内存模型的精髓可以用一句话概括：

> **不建立在 happens-before 关系上的并发读写，其行为是未定义的。**

你不需要背下所有规则，但需要记住几个关键点：
1. **Channel 收发自带 happens-before**（最强保证）
2. **Lock/Unlock 之间有 happens-before**
3. **Goroutine 启动前的主 goroutine 写入对新 G 可见**
4. **Sleep、自旋、布尔标志都不能替代真正的同步**
5. **永远用 `-race` 检测数据竞争**

> 下篇聊 Go 协程调度器的 G-M-P 模型——理解调度器之后，你对 goroutine 为什么这么"便宜"、为什么能支持百万并发会有全新的认识。
