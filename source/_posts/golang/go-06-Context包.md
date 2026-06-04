---
title: Go 工程实践（中）：Context 包
date: 2025-06-04
categories:
  - [Golang学习]
tags:
  - Go
  - Context
  - 并发
  - 超时
  - 取消
  - 工程实践
  - 问答
top_img: linear-gradient(135deg, #a1c4fd 0%, #c2e9fb 100%)
---

> Context 是 Go 并发编程的核心。没有它，你的 goroutine 就像脱缰的野马——跑起来不知道怎么停，超时了也没法杀。这篇彻底搞懂 context。

---

## Q1：Context 到底解决什么问题？

**A：** 一个 goroutine 启动后，调用方面临三个问题：

| 问题 | 没有 Context | 有 Context |
|------|-------------|-----------|
| **取消任务** | 用户关了页面，后台 goroutine 还在跑 | `ctx.Cancel()` 一键停掉 |
| **超时控制** | 数据库卡住了？请求永远不返回 | `ctx.Timeout()` 自动取消 |
| **传值** | goroutine 需要请求 ID、用户信息 | `ctx.Value()` 跨层传递 |

```go
// 没有 context 的写法（别这么写）
func fetchData() ([]User, error) {
    ch := make(chan []User)
    go func() {
        data := queryFromDB() // 如果 DB 慢了，这里会一直卡着
        ch <- data
    }()
    return <-ch, nil // 无限等待...
}

// 用 context 的写法
func fetchData(ctx context.Context) ([]User, error) {
    ch := make(chan []User, 1)
    go func() {
        data := queryFromDB(ctx) // 可以响应 ctx 取消
        ch <- data
    }()
    select {
    case data := <-ch:
        return data, nil
    case <-ctx.Done():
        return nil, ctx.Err() // 超时或被取消了
    }
}
```

## Q2：四种 Context 怎么选？

**A：**

```go
// 1. Background / TODO —— 根上下文
ctx := context.Background()   // main 函数、初始化用
ctx := context.TODO()         // 还不确定用哪个的时候先占位

// 2. WithCancel —— 手动取消
ctx, cancel := context.WithCancel(parentCtx)
defer cancel() // 重要！一定要 cancel
go doSomething(ctx)
cancel()     // 随时可以调用取消

// 3. WithTimeout —— 带超时的取消
ctx, cancel := context.WithTimeout(parentCtx, 3*time.Second)
defer cancel()
// 3 秒后自动取消

// 4. WithDeadline —— 在某个时间点取消
deadline := time.Now().Add(5 * time.Minute)
ctx, cancel := context.WithDeadline(parentCtx, deadline)
defer cancel()

// 5. WithValue —— 传递请求范围的值
ctx = context.WithValue(parentCtx, "requestID", "abc-123")
```

**选择决策树**：

```
需要什么？
├── 只是一个根节点 → context.Background()
├── 需要手动取消 → WithCancel
├── 需要超时限制 → WithTimeout（最常用）
├── 需要在特定时间点截止 → WithDeadline
└── 需要跨函数传值 → WithValue（谨慎使用）
```

## Q3：WithTimeout 实际项目中怎么用？给个完整示例

**A：** HTTP 服务中的典型模式：

```go
func (s *Server) GetUser(w http.ResponseWriter, r *http.Request) {
    // 从 request 中获取 context（自带请求取消）
    ctx := r.Context()

    // 给数据库查询设一个独立的超时
    dbCtx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()

    user, err := s.repo.GetUser(dbCtx, userID)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "DB query timeout", http.StatusGatewayTimeout)
            return
        }
        if errors.Is(err, context.Canceled) {
            log.Printf("client disconnected: %v", ctx.Err())
            return
        }
        // 其他错误...
    }

    json.NewEncoder(w).Encode(user)
}
```

**关键细节**：
- `r.Context()` 在客户端断开连接时会自动 cancel
- `WithTimeout` 创建子 context，超时不影响父 context
- **`defer cancel()` 必须写！** 不写会导致资源泄漏

## Q4：WithValue 该不该用？听说社区有争议？

**A：** 确实有争议。先用代码看清楚：

### 正确用法

```go
type contextKey string

const (
    UserIDKey    contextKey = "userID"
    RequestIDKey contextKey = "requestID"
)

// Middleware 中设置值
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        userID := validateToken(token)
        
        // 将 userID 放入 context
        ctx := context.WithValue(r.Context(), UserIDKey, userID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Handler 中取出值
func Handler(w http.ResponseWriter, r *http.Request) {
    userID := r.Context().Value(UserIDKey).(string)
    // ...
}
```

### 为什么要有争议？

| 问题 | 说明 |
|------|------|
| 类型不安全 | `Value()` 返回 `interface{}`，需要类型断言 |
| 隐式依赖 | 函数签名看不出需要什么值 |
| 容易滥用 | 有人把什么都塞进 context |
| 测试麻烦 | 要手动构造带值的 context |

**最佳实践**：

1. **定义私有 key 类型**（`contextKey`），避免冲突
2. **只放请求范围的数据**（requestID、userID、traceID、language）
3. **不要放可选参数**——那应该用函数参数或选项模式
4. **不要放指针**——context 取出的值应该是不可变的

## Q5：Context 传播规则是什么？

**A：** 记住一张图：

```
Background (根)
├── WithCancel(A) ──── A 被取消 → A 和所有子节点都取消
│   ├── WithTimeout(B) ─── B 超时 → B 和它的子节点被取消，A 不受影响
│   │   └── WithValue(C) ─── C 是纯值传递，不能取消
│   └── WithCancel(D)
└── WithCancel(E) ──── E 和 A 是兄弟，互不影响
```

**核心规则**：

1. **父取消 → 所有子节点自动取消**
2. **子取消 → 不影响父和兄弟**
3. **一旦 cancel 就不可恢复**（context 是单向的）
4. **WithValue 创建的 context 不能 cancel**

## Q6：常见错误有哪些？

**A：**

### 错误 1：忘记 defer cancel()

```go
// ❌ 泄漏！
func bad() {
    ctx, _ := context.WithTimeout(context.Background(), 10*time.Second)
    doSomething(ctx)
}

// ✅ 正确
func good() {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    doSomething(ctx)
}
```

即使超时了也要 cancel，因为内部资源可能需要立即释放。

### 错误 2：把 context 放进结构体

```go
// ❌ 别这么做
type Service struct {
    ctx context.Context
}

// ✅ context 作为第一个参数传入
func (s *Service) DoSomething(ctx context.Context) error { ... }
```

context 应该在调用链中流动，不应该存储在结构体里。

### 错误 3：用 context 传函数选项

```go
// ❌ 这是 WithValue 的滥用
ctx = context.WithValue(ctx, "limit", 100)
ctx = context.WithValue(ctx, "offset", 0)

// ✅ 这才是正经做法
func Query(ctx context.Context, opts ...QueryOption) (*Result, error)
```

## Q7：标准库中哪些地方用了 Context？

**A：** 几乎所有涉及 I/O 的标准库都支持：

```go
// HTTP Server
http.Server.Shutdown(ctx)

// Database/sql
db.ExecContext(ctx, "INSERT INTO...")
db.QueryRowContext(ctx, "SELECT...")

// net/http Client
http.NewRequestWithContext(ctx, "GET", url, body)

// gRPC
client.Call(ctx, req)
server.RegisterHandler(func(ctx, req) {})

// os/exec
cmd.Start() + cmd.Wait() → exec.CommandContext(ctx, "ls", "-la")
```

**如果你在写的函数可能耗时或阻塞，加一个 `context.Context` 参数是 Go 社区的约定。**

## 下期预告

下一篇聊 **Testing：表驱动测试 + Benchmark**。Go 的测试哲学和别的语言很不一样——没有复杂的框架，但 `testing` 包配合 table-driven test 能写出非常优雅的测试代码。

> **TODO**: 回顾你 slg-go 项目里的 handler 或 service 方法——每个涉及网络/数据库调用的方法都有 `context.Context` 作为第一个参数吗？如果没有，现在就是重构的好时机。
