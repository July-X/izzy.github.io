---
title: Go 并发进阶（下）：从零搭建 HTTP 服务
date: 2025-06-04
categories:
  - [Golang学习]
tags:
  - Go
  - HTTP
  - net/http
  - 中间件
  - Graceful Shutdown
  - 实战
  - 问答
top_img: linear-gradient(135deg, #0ba360 0%, #3cba92 100%)
---

> 这是 Golang 学习系列的收官篇。我们把前面 9 篇的所有知识——结构体、接口、错误处理、Context、Testing、并发模式、sync 包——串起来，从零搭建一个**生产级 HTTP 服务**。写完这篇，你就能独立写出像样的 Go 后端了。

---

## Q1：用标准库搭一个最小 HTTP 服务需要几行？

**A：** 真的很少：

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

func main() {
    // 注册路由
    http.HandleFunc("/health", healthHandler)
    http.HandleFunc("/api/users", usersHandler)

    // 启动服务
    fmt.Println("Server starting on :8080")
    // ❌ 这样启动无法优雅关闭！下面会改进
    log.Fatal(http.ListenAndServe(":8080", nil))
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
}

func usersHandler(w http.ResponseWriter, r *http.Request) {
    users := []User{
        {ID: 1, Name: "Izzy"},
        {ID: 2, Name: "AI"},
    }
    json.NewEncoder(w).Encode(users)
}
```

```bash
# 测试
curl http://localhost:8080/health
# {"status":"ok"}

curl http://localhost:8080/api/users
# [{"id":1,"name":"Izzy"},{"id":2,"name":"AI"}]
```

**10 行代码跑起来一个 REST API**——这就是 Go 的魅力。

## Q2：怎么设计中间件（Middleware）？

**A：** Go 的中间件模式很优雅——函数嵌套函数：

```go
// 中间件类型定义
type Middleware func(http.Handler) http.Handler

// 日志中间件
func Logger(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // 包装 ResponseWriter 以捕获状态码
        wrapped := &responseWriter{ResponseWriter: w, status: 200}
        
        next.ServeHTTP(wrapped, r)
        
        fmt.Printf("[%s] %s %s %d %v\n",
            time.Now().Format("2006-01-02 15:04:05"),
            r.Method,
            r.URL.Path,
            wrapped.status,
            time.Since(start),
        )
    })
}

// CORS 中间件
func CORS(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, OPTIONS")
        if r.Method == "OPTIONS" { return }
        next.ServeHTTP(w, r)
    })
}

// Recovery 中间件（防止 panic 导致整个服务崩溃）
func Recovery(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("[PANIC] %v\n%s\n", err, debug.Stack())
                http.Error(w, `{"error":"internal server error"}`,
                    http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

### 链式组合中间件

```go
func Chain(middlewares ...Middleware) Middleware {
    return func(final http.Handler) http.Handler {
        for i := len(middlewares) - 1; i >= 0; i-- {
            final = middlewares[i](final)
        }
        return final
    }
}

// 使用
mux := http.NewServeMux()
mux.HandleFunc("/api/...", handler)

handler := Chain(Recovery, Logger, CORS)(mux)

http.Server{Handler: handler}.ListenAndServe(":8080")
```

执行顺序：**Recovery → Logger → CORS → Handler**（从外到内）。

## Q3：怎么实现 Graceful Shutdown（优雅关闭）？

**A：** 这是生产环境必须有的能力：

```go
func main() {
    // 路由注册
    mux := http.NewServeMux()
    mux.HandleFunc("/health", healthHandler)
    mux.HandleFunc("/api/users", usersHandler)

    // 中间件链
    handler := Chain(Recovery, Logger, CORS)(mux)

    // 创建 Server（不直接调用 ListenAndServe）
    server := &http.Server{
        Addr:         ":8080",
        Handler:      handler,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    // 用 channel 监听系统信号
    go func() {
        fmt.Println("Server starting on :8080")
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("server error: %v", err)
        }
    }()

    // 等待中断信号
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    fmt.Println("\nShutting down server...")

    // 设置关闭超时，给正在处理的请求 15 秒缓冲时间
    ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Fatalf("server forced to shutdown: %v", err)
    }

    fmt.Println("Server exited gracefully")
}
```

**Graceful Shutdown 做了什么？**

| | 直接 kill | Graceful Shutdown |
|--|----------|-------------------|
| 正在处理的请求 | **立即断开** | **等待完成**（最多 15 秒）|
| 新请求 | — | **拒绝接受** |
| 连接资源 | 可能泄漏 | **正确释放** |
| 客户端体验 | 收到 connection reset | 正常收到响应 |

## Q4：怎么组织项目目录结构？实际项目的布局

**A：** 推荐的中小型项目布局：

```
my-server/
├── cmd/
│   └── server/
│       └── main.go          # 入口
├── internal/
│   ├── handler/             # HTTP 处理层
│   │   ├── user.go
│   │   └── middleware.go
│   ├── service/             # 业务逻辑层
│   │   └── user_service.go
│   ├── repository/          # 数据访问层
│   │   └── user_repo.go
│   └── model/               # 数据模型
│       └── user.go
├── pkg/                     # 可被外部引用的工具包
│   └── response/
│       └── api_error.go
├── configs/
│   └── config.yaml
├── scripts/
│   └── migrate.sql
├── test/
│   └── integration/
├── go.mod
├── go.sum
└── README.md
```

**分层职责**：

| 层 | 职责 | 依赖 |
|---|------|------|
| handler | 参数校验、调用 service、返回响应 | service |
| service | 业务逻辑编排、事务管理 | repository |
| repository | SQL 执行、缓存操作 | model |
| model | 结构体定义，纯数据 | 无 |

## Q5：完整示例——把所有知识串起来的 Mini API

```go
// cmd/server/main.go
func main() {
    // 1. 加载配置
    cfg := loadConfig()

    // 2. 初始化依赖
    db := connectDB(cfg.DatabaseURL)
    
    repo := NewUserRepo(db)
    svc := NewUserService(repo)
    h := NewUserHandler(svc)

    // 3. 路由 + 中间件
    mux := http.NewServeMux()
    mux.HandleFunc("/api/v1/users", h.ListUsers)     // GET
    mux.HandleFunc("/api/v1/users/", h.GetUser)        // GET /:id
    mux.HandleFunc("/api/v1/users/create", h.CreateUser) // POST

    handler := Chain(
        Recovery,
        Logger,
        CORS,
        AuthMiddleware(cfg.JWTSecret), // 认证中间件
    )(mux)

    // 4. 启动（带 graceful shutdown）
    runGracefully(handler, cfg.Port)
}

// internal/handler/user.go
func (h *UserHandler) ListUsers(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context() // ← Context 从这里开始传递
    
    // 带超时的数据库查询
    dbCtx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()
    
    users, err := h.svc.GetAll(dbCtx) // ← Context 一路传到 DB 层
    if err != nil {
        HandleError(w, r, err)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}
```

注意看 **Context 是怎么流动的**：
`Request → Handler → Service → Repository → DB Driver`
每一层都能响应取消和超时。

## Q6：怎么测试这个 HTTP 服务？

**A：** 用 `net/http/httptest`，不需要启动真实服务器：

```go
func TestListUsers(t *testing.T) {
    // Mock Repository
    mockRepo := &MockUserRepo{
        Users: []User{{ID: 1, Name: "Izzy"}},
    }

    svc := NewUserService(mockRepo)
    h := NewUserHandler(srv)

    // 创建测试请求
    req := httptest.NewRequest("GET", "/api/v1/users", nil)
    w := httptest.NewRecorder()

    // 直接调用 handler
    h.ListUsers(w, req)

    // 断言结果
    resp := w.Result()
    if resp.StatusCode != http.StatusOK {
        t.Errorf("expected 200, got %d", resp.StatusCode)
    }

    var users []User
    json.NewDecoder(resp.Body).Decode(&users)
    if len(users) != 1 || users[0].Name != "Izzy" {
        t.Errorf("unexpected response body")
    }
}
```

**集成测试（带真实 HTTP 层）**：

```go
func TestAPI_Integration(t *testing.T) {
    router := setupRouter()
    server := httptest.NewServer(router)
    defer server.Close()

    resp, _ := http.Get(server.URL + "/api/v1/users")
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        t.Fatal("expected 200")
    }
}
```

## 全系列回顾

我们用 10 篇文章覆盖了从零基础到能搭建生产服务的完整路径：

```
🏗️ 基础篇
  go-01  十问十答           → 变量、类型、流程控制、并发入门
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 OOP 篇
  go-02  结构体与方法        → 工厂模式、接收者选择、标签、内存对齐
  go-03  接口与鸭子类型      → 隐式接口、nil陷阱、error 本质
  go-04  组合优于继承        → 三种组合模式、设计模式
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚙️ 工程实践篇
  go-05  Error 处理         → %w包装、sentinel error、panic 边界
  go-06  Context 包         → 超时/取消/传值、传播规则、反模式
  go-07  Testing            → 表驱动测试、mock、benchmark
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🚀 并发进阶篇
  go-08  并发模式            → Pipeline/FanOut-In/Worker Pool/Select
  go-09  sync 包             → Mutex/RWMutex/WG/Once/Pool/atomic
  go-10  HTTP 服务实战       → 中间件/Graceful Shutdown/项目架构
```

> **TODO**: 用这篇的模板在你自己的服务器上搭一个 mini API——哪怕只是 CRUD 一个 User 表。亲手敲一遍代码比读十篇文章都管用。遇到问题随时回来聊。
