---
title: Go 协程调度器：G-M-P 模型与调度策略
date: 2026-06-04
tags:
  - Golang
  - 调度器
  - G-M-P模型
  - goroutine
categories: Golang学习
---

> 前两篇分别讲了 Channel 的底层（hchan 数据结构）和内存模型（happens-before 规则）。但还有一个更基础的问题没回答：**goroutine 为什么这么轻量？调度器是怎么管理数万个 goroutine 的？** 这篇深入 Go 调度器的 G-M-P 模型，结合 slg-go 项目的真实并发场景分析。

<!-- more -->

## 背景：slg-go 里的 Goroutine 开销

在 slg-go 项目中，单个 Logic Node 进程的 goroutine 分布大致如下：

```go
// cmd/gate/main.go — Gateway 进程启动时的 goroutine
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // 1. HTTP 服务（net/http 内部为每个请求分配一个 goroutine）
    gw.Start(ctx)   // → 底层是 http.Server，每个请求一个 G

    // 2. 心跳监控 goroutine × 1
    go func() {
        ticker := time.NewTicker(5 * time.Second)
        for {
            select {
            case <-ctx.Done(): return
            case <-ticker.C:
                gw.Stats()  // 收集指标
            }
        }
    }()

    // 3. 每个 WebSocket 连接 = 2 个 goroutine（读 + 写）
    // ws.go:29-147

    // 4. 战斗调度器 = N 个 worker goroutine
    // scheduler.go:195-200, 默认 N=4
}
```

**来源**：`cmd/gate/main.go`, `cmd/chat/ws.go`, `internal/battle/scheduler/scheduler.go`

估算一下总 goroutine 数量：

| 来源 | 数量 | 说明 |
|------|------|------|
| HTTP 请求处理 | ~1000 | 取决于并发连接数 |
| WebSocket 连接 | ~20,000 | 每连接 2G = 读+写 |
| 战斗 Worker | 4 | 固定 worker pool |
| 后台任务 | ~10 | 心跳、metrics、etcd 保活等 |
| **合计** | **~21,000** | 单进程 |

21,000 个 goroutine，总内存约 **40-50MB**（每个 G 约 2KB 初始栈）。如果用 Java Thread 呢？默认栈 1MB/线程 → **21GB 内存**。这就是为什么 Go 能做到百万在线而 Java 不行。

---

## Q1：G-M-P 三者到底是什么？

Go 调度器的核心数据结构：

{% mermaid %}
flowchart LR
    subgraph G["G (Goroutine)"]
        G1["G1<br/>用户代码"]
        G2["G2<br/>用户代码"]
        G3["G3<br/>用户代码"]
        GN["..."]
    end

    subgraph M["M (Machine / OS Thread)"]
        M1["M1:OS线程<br/>绑定P1"]
        M2["M2:OS线程<br/>绑定P2"]
        MN["..."]
    end

    subgraph P["P (Processor)"]
        P1["P1<br/>本地运行队列<br/>[G5,G6,G7]"]
        P2["P2<br/>本地运行队列<br/>[G8,G9]"]
        PN["..."]
    end

    G1 --> P1
    G2 --> P1
    G3 --> P2
    P1 --> M1
    P2 --> M2
{% endmermaid %}

### G — Goroutine（协程）

```
type g struct {
    stack       stack       // 栈空间：起始地址 + 边界
    m           *m          // 当前绑定的 M（执行时才有值）
    sched       gobuf       // 保存调度信息（SP/PC）—— 切走时存，恢复时读
    goid        int64       // goroutine ID
    atomicstatus uint32     // 状态：_Gidle/_Grunnable/_Grunning/_Gwaiting 等
    // ...
}
```

**关键特性**：
- 初始栈只有 **2KB**（vs Java Thread 的 1MB），按需增长（最大可达 1GB）
- 创建成本极低（只需分配 `g` 结构体 + 2KB 栈），所以可以随便 `go func()`
- 用户态切换：不经过内核，切换开销约 **几十纳秒**（vs 线程切换的微秒级）

### M — Machine（操作系统线程）

```
type m struct {
    g0      *g      // 特殊 goroutine：负责调度的"根协程"
    curg    *g      // 当前正在执行的 goroutine
    p       *p      // 绑定的 Processor
    nextp   *p      // 即将绑定的 P（syscall 时暂存）
    spinning bool    // 是否正在自旋寻找可运行的 G
    // ...
}
```

**关键特性**：
- M 是真正的 OS 线程（由 OS 调度）
- 默认数量 = `GOMAXPROCS`（通常等于 CPU 核心数），但阻塞时会创建额外 M
- 每个 M 有一个特殊的 `g0`——**调度专用的 goroutine**，拥有更大的固定栈（64KB）

### P — Processor（逻辑处理器）

```
type p struct {
    id          int32
    status      uint32    // _Pidle/_Prunning/...
    m           *m       // 绑定的 M（nil 表示空闲）
    runq [256]guint32    // 本地运行队列（环形数组，固定大小 256）
    runnext     guint32  // 下一个要运行的 G（优先级高于 runq 头部）
    // ...
}
```

**关键特性**：
- 数量固定 = `GOMAXPROCS`（默认 CPU 核心数）
- **P 才是真正持有运行队列的实体**
- P 的本地队列大小只有 256，满了就放全局队列

### 三者的关系

```
P 是核心：M 必须绑定 P 才能执行 G。
没有 P 的 M 无法运行用户代码（只能执行 g0 或 sysmon）。

初始状态：
  GOMAXPROCS=8 → 8个P, 8个M（各绑一个P）

高并发时（如大量 I/O 阻塞）：
  M 可能被临时创建超过 GOMAXPROCS 个，
  但活跃的 P 始终只有 GOMAXPROCS 个。
```

---

## Q2：Goroutine 的状态转换

理解状态转换是理解调度行为的关键：

{% mermaid %}
stateDiagram-v2
    [*] --> _Gidle : 创建
    _Gidle --> _Grunnable : go func()
    _Grunnable --> _Grunning : P.runq 取出
    _Grunning --> _Grunnable : 时间片用完 / 抢占
    _Grunning --> _Gwaiting : channel收发/锁/sleep
    _Gwaiting --> _Grunnable : 条件满足/Goready
    _Grunning --> _Gdead : 函数返回
    _Grunning --> _Gcopystack : 栈增长
    _Gcopystack --> _Grunning : 栈扩容完成
    note right of _Gwaiting : "Gopark: 挂起当前G"
    note left of _Grunnable : "可运行队列等待"
{% endmermaid %}

### 从 slg-go 代码看状态转换

**场景一：Worker 正常取任务**

```go
// scheduler.go:337-356
func (s *Scheduler) worker(ctx context.Context, id int) {
    defer s.wg.Done()
    for {                              // G 在 _Grunning 状态
        select {                       // ← 这里可能触发 _Gwaiting
        case <-ctx.Done():
            return                    // → _Gdead
        case <-s.stopCh:
            return                    // → _Gdead
        case req, ok := <-s.requests:  // ← 如果 requests 为空
            if !ok { return }
            s.processBattle(ctx, req)
        }
    }                                 // 循环继续 → _Grunning
}
```

当 `s.requests` 为空且没有其他 case 就绪时：
1. 当前 G 从 `_Grunning` → `_Gwaiting`（调用 `Gopark`）
2. G 被放入 `requests` channel 的 `recvq`
3. M 解绑 P 去找其他 G 运行
4. 新任务到达时，G 被 Goready → `_Grunnable`
5. 某个 P 取到它 → `_Grunning`

**整个过程对开发者完全透明。**

**场景二：WebSocket 双 goroutine 协作**

```go
// ws.go:29-146 — 完整的状态流转
func newWebSocketHandler(hub *chat.Hub, tokenSecret string) http.Handler {
    return websocket.Handler(func(ws *websocket.Conn) {
        // === 主 goroutine（G-main）：读循环 ===
        ctx, cancel := context.WithCancel(context.Background())
        msgCh := make(chan *chat.Message, 128)

        // 启动写 goroutine
        go func() {                   // → G-write: _Grunnable → _Grunning
            defer writeWG.Done()
            for {
                select {
                case <-ctx.Done():     // → _Gwaiting（等 cancel）
                    return             // → _Gdead
                case msg := <-msgCh:   // → _Gwaiting（等消息）
                    websocket.JSON.Send(ws, ...)  // 网络I/O
                case env := <-controlCh: // → _Gwaiting（等信封）
                    websocket.JSON.Send(ws, ...)
                }
                // 每次 select 返回后 → _Grunning
            }
        }()

        // 主 goroutine 继续做读循环
        for {
            var cmd wsCommand
            websocket.JSON.Receive(ws, &cmd)  // 网络I/O阻塞
            // 处理命令...
        }
        // 连接断开后 cancel() → G-write 从 _Gwaiting → _Grunnable → _Gdead
    })
}
```

**来源**：`cmd/chat/ws.go:29-146`

这里两个 goroutine 通过 `ctx` 和 `channel` 协作生命周期：
- 主 G 负责读，阻塞在网络 I/O 上
- G-write 负责写，阻塞在 `select` 上
- 任一方出错都通过 `cancel()` 通知另一方退出

---

## Q3：调度策略——Work Stealing 与 Hand Off

Go 调度器有两个核心调度策略：

### Work Stealing（工作窃取）

当 P 的本地队列空了：

```
P1 的 runq: []              ← 空了！
P2 的 runq: [G1, G2, G3]

→ P1 从 P2 的 runq 尾部偷一半 G
→ P1 的 runq: [G3]
→ P2 的 runq: [G1, G2]
```

**为什么从尾部偷？** 因为尾部的 G 更可能是最近才创建的，缓存热度更低。

### Hand Off（移交）

当 M 因系统调用阻塞（如网络 I/O）时：

```
M1 绑定 P1, 执行 G1
G1 调用了 read() 系统调用 → M1 将阻塞

→ M1 把 P1 移交给空闲的 M2
→ M2 继续执行 P1 上的其他 G
→ M1 在 syscall 完成后重新找一个 P 绑定
```

这就是为什么 slg-go 的 WebSocket handler 中，即使主 goroutine 阻塞在 `websocket.JSON.Receive()` 上，同一个 P 上的其他 goroutine 也不会受影响——P 会被移交给别的 M。

---

## Q4：抢占式调度——防止长运行 G 饿死其他 G

Go 1.14 之后引入了基于信号的异步抢占。在此之前只有协作式抢占（函数调用时的检查点）。

### 抢占机制

```
sysmon 线程（特殊 M，不绑定 P）每 10µs 检查一次：
├── 如果某个 G 运行超过 10ms（单次执行时间过长）
│   └── 向该 G 所在的 M 发送 SIGURG 信号
│       └── 信号处理函数保存 G 的上下文
│       └── 把 G 放回运行队列尾部
│       └── 调度器选择下一个 G 运行
```

这对战斗引擎有什么影响？

```go
// engine.go:127-189 — Execute 方法
func (e *BattleEngine) Execute(ctx context.Context, order *BattleOrder) (*BattleResult, error) {
    for round := 1; round <= maxRounds; round++ {   // 最多 200 轮
        // ... 大量计算 ...
        e.executeRound(first, second, terrainMod, ...)  // 每轮遍历所有兵种
        e.executeRound(second, first, terrainMod, ...)
        // ...
    }
}
```

**来源**：`engine.go:127-189`

如果一场战斗涉及大量兵种和 200 轮计算，单次 `Execute()` 可能耗时数十毫秒。此时抢占机制会确保这个 G 不会独占 M 太久——虽然对于纯计算密集型场景，更好的做法是把大计算拆分成多个步骤，主动让出：

```go
// 更好的做法：每轮检查 context，主动让步
for round := 1; round <= maxRounds; round++ {
    if ctx.Err() != nil { return nil, ctx.Err() }  // 检查取消
    e.executeRound(...)
    // 如果需要更好的响应性：
    // runtime.Gosched()  // 主动让出，把 G 放回队尾
}
```

> **TODO**: engine.go 的 `Execute()` 方法目前没有中间检查点，超长的战斗可能导致单个 worker 被长时间占用。考虑加入 `runtime.Gosched()` 或轮间 context 检查。

---

## Q5：Goroutine 栈的增长机制

这是 Go 能支持大量 goroutine 的关键技术之一：

```
初始状态（2KB）:
┌──────────────┐
│   guard page  │  ← 保护页（访问会触发 SIGSEGV）
│   栈空间 2KB  │
│   ...         │
└──────────────┘

栈不够用时（morestack）:
┌──────────────┐
│   guard page  │
│   栈空间 4KB  │  ← 双倍增长（copy-on-write，实际不立即分配物理内存）
│   ...         │
└──────────────┘

最终可能增长到:
┌──────────────┐
│   guard page  │
│   栈空间 1GB  │  ← 最大限制
└──────────────┘
```

**增长规则**：
- 初始 2KB
- 不足时翻倍（连续空间，copy-on-write）
- 最大 1GB（默认，可通过 `debug.SetMaxStack` 修改）
- **收缩**：GC 时如果栈只用了一半以下，会缩回一半

这意味着你不需要像写 C 那样预先估算栈大小——Go 会自动管理。这也是为什么 `go func()` 可以放心大胆地用。

---

## Q6：System Monitor（sysmon）—— 特殊的后台线程

Go 运行时有一个特殊的 M 叫 `sysmon`，它**不绑定任何 P**，直接运行在 OS 层面：

```go
// runtime/proc.go — sysmon 主循环（伪代码）
func sysmon() {
    for {
        sleep(10 microseconds)  // 每 10µs 醒来检查

        // 1. 抢占运行过久的 G
        if g.runningTooLong() {
            preempt(g)
        }

        // 2. 回收因 network poller 阻塞的 P
        retake(nanotime())

        // 3. 强制 GC（如果超过 2 分钟没 GC）
        if forcegc > 0 {
            gcStart()
        }
    }
}
```

sysmon 对 slg-go 这类服务器程序特别重要：
- 它保证了即使你的代码忘记让出 CPU，调度器仍然能正常工作
- 它处理了网络 I/O 完成后的 G 唤醒（epoll/kqueue 事件转交给 P）
- 它触发了定时 GC

---

## Q7：从架构角度看调度设计

slg-go 的三层架构中，每一层的调度模式都不一样：

| 层级 | 并发模型 | 调度特点 |
|------|---------|---------|
| **Gate**（网关） | 1连接≈2G（读写分离） | 大量短生命周期 G，依赖网络 I/O 阻塞自动 hand off |
| **Logic**（逻辑） | 分片 + RWMutex | G 主要做内存操作和 DB 查询，CPU 密集型少 |
| **Battle**（战斗） | Fixed Worker Pool（N=4） | G 是常驻 worker，循环消费 channel 任务 |

{% mermaid %}
flowchart TB
    subgraph Gate ["Gate 层 — 高并发 I/O"]
        G1["HTTP Handler G×N<br/>短生命周期"]
        G2["WS Read G×N<br/>阻塞在 recv"]
        G3["WS Write G×N<br/>阻塞在 select"]
    end

    subgraph Logic ["Logic 层 — 分片计算"]
        L1["Shard 0<br/>RWMutex 保护"]
        L2["Shard 1<br/>RWMutex 保护"]
        L3["...")
    end

    subgraph Battle ["Battle 层 — 固定池"]
        B1["Worker 0<br/>for-select 循环"]
        B2["Worker 1<br/>for-select 循环"]
        B3["Worker 2<br/>for-select 循环"]
        B4["Worker 3<br/>for-select 循环"]
    end

    Gate -->|"chan *Request"| Battle
    Battle -->|"resultChan"| Logic
    Logic -->|"DB/Cache"| Gate
{% endmermaid %}

**调度效率的关键洞察**：
- Gate 层的 G 大部分时间处于 `_Gwaiting`（等网络 I/O），占用 M 极少
- Logic 层竞争集中在 RWMutex 上，持锁时间应尽量短
- Battle 层的 Worker G 几乎一直在跑（`_Grunning`），是 CPU 密集型的，所以要控制 worker 数 ≤ `GOMAXPROCS`

---

## 总结

Go 调度器的设计哲学可以用一句话概括：

> **用少量的 OS 线程（M=GOMAXPROCS）高效地运行大量的用户态协程（G=数万~数十万），通过 P 做中介实现 work stealing 和负载均衡。**

理解 G-M-P 模型之后，你在写并发代码时会有几个直觉性的判断：
1. **goroutine 随便开**——2KB 栈 + 用户态切换，成本低到几乎免费
2. **不要共享内存，用 channel 通信**——避免锁竞争导致的上下文切换
3. **CPU 密集型任务控制并发度**——worker pool 大小 ≈ CPU 核心数
4. **I/O 密集型任务放开胆子开 goroutine**——大部分时间都在 `_Gwaiting`，不抢 CPU
5. **长运行计算加 Gosched 或拆成小步骤**——别饿死同 P 上的邻居

> 六篇下来，Go 并发的三个维度——通信原语（Channel）、内存安全（Happens-Before）、调度机制（G-M-P）——算是讲完了。接下来你想往哪个方向深入？是 Go 反射与泛型的实战用法，还是 Go 在分布式系统中的工程实践？
