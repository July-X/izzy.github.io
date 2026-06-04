---
title: Go 并发进阶（中）：sync 包深挖
date: 2026-06-04
categories:
  - [Golang学习]
tags:
  - Go
  - sync
  - Mutex
  - RWMutex
  - WaitGroup
  - Once
  - Pool
  - 并发安全
  - 问答
top_img: linear-gradient(135deg, #c471f5 0%, #fa71cd 100%)
---

> sync 包是 Go 并发安全的基石。Mutex、RWMutex、WaitGroup、Once、Pool——每个都简单但都有坑。选对用对，性能差几倍；选错了，就是隐蔽的 data race。

---

## Q1：Mutex vs RWMutex 怎么选？

**A：**

```go
// Mutex —— 互斥锁（写锁）
var mu sync.Mutex

func (s *Store) Set(key string, val string) {
    mu.Lock()
    defer mu.Unlock()
    s.data[key] = val
}

// RWMutex —— 读写锁（读多写少场景）
var rwmu sync.RWMutex

func (s *Store) Get(key string) string {
    rwmu.RLock()         // 读锁，多个 reader 可以同时持有
    defer rwmu.RUnlock()
    return s.data[key]
}

func (s *Store) Set(key string, val string) {
    rwmu.Lock()          // 写锁，独占
    defer rwmu.Unlock()
    s.data[key] = val
}
```

| 特性 | Mutex | RWMutex |
|------|-------|---------|
| 写-写 | 互斥 | 互斥 |
| 读-读 | 互斥 | **可以并行** |
| 读-写 | 互斥 | 互斥 |
| 开销 | 低 | 比 Mutex 稍高 |
| 适用 | 读写差不多 | **读多写少** |

**选择标准**：

```
读操作 >> 写操作？ → RWMutex（如缓存、配置读取）
读写比例接近或不确定？ → Mutex（更简单安全）
```

### ⚠️ RWMutex 的陷阱

```go
// ❌ 升级陷阱！不能从读锁升级为写锁（死锁）
rwmu.RLock()
val := getSomething()
if needModify(val) {
    // rwmu.Lock()   // 死锁！！！RWMutex 不支持锁升级
}
rwmu.RUnlock()

// ✅ 解决方案：先释放读锁再获取写锁
rwmu.RLock()
val := getSomething()
rwmu.RUnlock()

if needModify(val) {
    rwmu.Lock()
    modify()
    rwmu.Unlock()
}
```

## Q2：WaitGroup 怎么正确使用？

**A：** 等待一组 goroutine 完成：

```go
func fetchAllData(ctx context.Context) (*AllData, error) {
    var wg sync.WaitGroup
    result := &AllData{}
    errCh := make(chan error, 3)

    // Add 必须在 goroutine 启动之前！
    wg.Add(1)
    go func() {
        defer wg.Done()
        d, err := fetchUsers(ctx)
        if err != nil { errCh <- err; return }
        result.Users = d
    }()

    wg.Add(1)
    go func() {
        defer wg.Done()
        d, err := fetchOrders(ctx)
        if err != nil { errCh <- err; return }
        result.Orders = d
    }()

    wg.Add(1)
    go func() {
        defer wg.Done()
        d, err := fetchPoints(ctx)
        if err != nil { errCh <- err; return }
        result.Points = d
    }()

    wg.Wait()
    
    select {
    case err := <-errCh:
        return nil, err
    default:
        return result, nil
    }
}
```

**常见错误**：

```go
// ❌ 错误1：Add 在 goroutine 内部
go func() {
    wg.Add(1)       // 可能 Wait 已经执行完了
    defer wg.Done()
}()

// ❌ 错误2：Done 调用次数少于 Add
wg.Add(3)
go func() { /* 如果 panic 了，Done 没调用 */ }()
wg.Wait()           // 永远阻塞！

// ✅ 正确：defer Done 防止 panic 时遗漏
go func() {
    defer wg.Done() // 即使 panic 也会执行
    doWork()
}()
```

## Q3：Once 是做什么的？单例模式？

**A：** 确保**无论多少个 goroutine 同时调用，某个函数只执行一次**：

```go
var (
    instance *Database
    once     sync.Once
)

func GetDB() *Database {
    once.Do(func() {
        fmt.Println("初始化数据库连接...")
        instance = connectDB("postgres://...")
    })
    return instance
}

// 无论多少个地方同时调 GetDB()，
// "初始化数据库连接..." 只打印一次
```

**和 `if instance == nil` 手动检查的区别**：

```go
// ❌ 不安全的"懒加载"
var instance *Database
func GetDBUnsafe() *Database {
    if instance == nil {      // 两个 goroutine 可能同时通过这里
        instance = connectDB() // 初始化两次！
    }
    return instance
}

// ✅ 安全的 Once
func GetDBSafe() *Database {
    once.Do(func() { instance = connectDB() })
    return instance
}
```

Once 还常用于：
- **只加载一次配置**
- **只注册一次 handler**
- **只启动一次后台服务**

## Q4：Pool 对象池怎么用？什么场景下有收益？

**A：** 复用对象减少 GC 压力：

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        // 池子空时创建新对象
        return new(bytes.Buffer)
    },
}

func processRequest(data []byte) []byte {
    buf := bufPool.Get().(*bytes.Buffer) // 从池中取
    defer func() {
        buf.Reset()       // 归还前重置状态
        bufPool.Put(buf)  // 放回池中
    }()
    
    buf.Write(data)
    result := doSomething(buf.Bytes())
    return result
}
```

**适合放池子的对象**：

| 类型 | 为什么适合 |
|------|-----------|
| bytes.Buffer | 频繁分配/回收的大切片 |
| json.Encoder / Decoder | 有内部 buffer 可复用 |
| 自定义连接/请求对象 | 创建成本高 |

**不适合的场景**：
- 小对象（int、string 等）——Pool 本身有开销，小对象得不偿失
- 带状态的对象——归还时忘记 Reset 会出 bug
- **GC 压力不大的场景**——别过度优化

## Q5：atomic 包呢？什么时候用它代替 Mutex？

**A：** 对于简单的单个值操作，`sync/atomic` 比 Mutex 更快：

```go
// Mutex 方式
type Counter struct {
    mu    sync.Mutex
    count int64
}
func (c *Counter) Inc() {
    c.mu.Lock()
    c.count++
    c.mu.Unlock()
}

// atomic 方式 —— 更快
type AtomicCounter struct {
    count int64
}
func (c *AtomicCounter) Inc() {
    atomic.AddInt64(&c.count, 1)
}
func (c *AtomicCounter) Get() int64 {
    atomic.LoadInt64(&c.count)
}
```

| 场景 | 推荐 |
|------|------|
| 单个 int/bool/pointer 操作 | **atomic**（无锁，CAS 实现）|
| 多字段复合操作 | **Mutex/RWMutex** |
| 复杂临界区逻辑 | **Mutex** |

常用 atomic 函数：
```go
atomic.StoreInt64(&addr, val)   // 写
atomic.LoadInt64(&addr)          // 读
atomic.AddInt64(&addr, delta)    // 加
atomic.CompareAndSwapInt64(&addr, old, new) // CAS
atomic.SwapInt64(&addr, new)     // 交换并返回旧值
```

## Q6：Data Race 怎么检测？竞态条件太隐蔽了

**A：** Go 内置了强大的检测器：

```bash
# 编译时开启 race detector 运行测试
go test -race ./...

# 或者运行程序
go run -race main.go
```

**输出示例**：
```
==================
WARNING: DATA RACE
Read at 0x00... by goroutine X:
  store.go:23 Get()
Previous write at 0x00... by goroutine Y:
  store.go:15 Set()
==================
```

**建议**：
1. CI 中**必须加 `-race`** 跑测试
2. 本地开发也偶尔跑一下
3. 不要依赖它替代正确的并发设计——race detector 只能发现**实际触发到的竞争**

## Q7：锁的性能对比到底差多少？

**A：** 一组基准数据（粗略参考）：

| 操作 | 无锁 | atomic | Mutex | RWMutex(读) |
|------|-----|--------|--------|------------|
| 简单计数 (~ns/op) | ~15 | ~25 | ~100 | ~50 |
| 读密集场景 | 最快 | 快 | 较慢 | **接近无锁** |
| 写密集场景 | — | 快 | OK | 比 Mutex 慘 |

**结论**：对于 slg-go 这种高并发游戏后端：
- **玩家状态缓存** → RWMutex（读多写少）
- **全局匹配队列** → Mutex 或 channel
- **简单计数器** → atomic
- **复杂业务逻辑** → 别纠结锁了，用 channel + actor 模型更好

## 下期预告

最后一篇：**从零搭建 HTTP 服务**。把前面所有知识串起来——路由注册、中间件设计、优雅关闭、graceful shutdown。写完这篇，你就具备了独立搭建 Go 后端服务的完整能力。

> **TODO**: 用 `go test -race ./...` 跑一次你项目的测试。如果有 race 报告别慌，这正是发现隐患的好机会。
