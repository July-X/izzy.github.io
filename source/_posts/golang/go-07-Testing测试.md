---
title: Go 工程实践（下）：Testing 测试
date: 2026-06-04
categories:
  - [Golang学习]
tags:
  - Go
  - Testing
  - 表驱动测试
  - Benchmark
  - 工程实践
  - 问答
top_img: linear-gradient(135deg, #89f7fe 0%, #66a6ff 100%)
---

> Go 的测试哲学：**测试代码也是一等公民**。不需要复杂的框架，不需要额外的依赖——标准库 `testing` 包 + 几个约定就够了。这篇聊聊怎么写出"很 Go 风格"的测试。

---

## Q1：Go 测试的基本规则是什么？

**A：** 三条铁律：

| 规则 | 说明 |
|------|------|
| 文件名 | `_test.go` 结尾（如 `user_test.go`）|
| 函数名 | `TestXxx` 开头（`Xxx` 首字母大写）|
| 参数 | 只接受 `*testing.T` |

```go
// user.go
func Add(a, b int) int {
    return a + b
}

// user_test.go
func TestAdd(t *testing.T) {
    result := Add(2, 3)
    if result != 5 {
        t.Errorf("Add(2,3) = %d; want 5", result)
    }
}
```

运行：
```bash
go test ./...              # 运行所有测试
go test -v ./...           # 详细输出
go test -run TestAdd ./... # 只跑某个测试
go test -cover ./...       # 查看覆盖率
```

就这么简单。没有 @Test 注解，没有 class 继承，没有 setup/teardown 装饰器。

## Q2：Table-Driven Test（表驱动测试）是什么？为什么说它是 Go 标志性写法？

**A：** 这是 Go 社区最推崇的测试模式：

```go
// ❌ 传统写法 —— 每个 case 一个函数，重复且冗长
func TestAdd_Positive(t *testing.T) { ... }
func TestAdd_Negative(t *testing.T) { ... }
func TestAdd_Zero(t *testing.T) { ... }

// ✅ Table-Driven Test —— 所有 case 在一个表里
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"正数相加", 2, 3, 5},
        {"负数相加", -1, -3, -4},
        {"零", 0, 0, 0},
        {"大数", 1000000, 2000000, 3000000},
        {"正负混合", -10, 20, 10},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Add(%d,%d) = %d; want %d",
                    tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

**为什么好？**

1. **新增 case 只需加一行数据**——不用写新函数
2. **`t.Run` 子测试**——每个 case 独立运行，失败时精确定位
3. **输入输出一目了然**——读测试就能理解函数行为

**实际项目中的样子**（slg-go 风格）：

```go
func TestMatchmaking_PairPlayers(t *testing.T) {
    tests := []struct {
        name      string
        players   []Player
        expectErr bool
        pairCount int
    }{
        {
            name: "偶数玩家正常配对",
            players: []Player{
                {Rating: 1500}, {Rating: 1520},
                {Rating: 1480}, {Rating: 1510},
            },
            expectErr: false,
            pairCount: 2,
        },
        {
            name: "奇数玩家一人轮空",
            players: []Player{
                {Rating: 1500}, {Rating: 1520},
                {Rating: 1480},
            },
            expectErr: false,
            pairCount: 1, // 第三个人轮空
        },
        // 更多 case...
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            mm := NewMatchmaker()
            pairs, err := mm.Pair(tt.players)
            
            if (err != nil) != tt.expectErr {
                t.Fatalf("unexpected error: %v", err)
            }
            if len(pairs) != tt.pairCount {
                t.Errorf("got %d pairs, want %d", len(pairs), tt.pairCount)
            }
        })
    }
}
```

## Q3：怎么 Mock 依赖？Go 没有 mock 框架吗？

**A：** 有框架但**不推荐过度使用**。Go 的方式是用接口做 mock：

```go
// 定义接口
type UserRepository interface {
    GetByID(ctx context.Context, id int) (*User, error)
}

// 真实实现
type DBUserRepo struct{ db *sql.DB }
func (r *DBUserRepo) GetByID(ctx context.Context, id int) (*User, error) { ... }

// Mock 实现（就在 _test.go 里）
type MockUserRepo struct {
    users map[int]*User
}

func (m *MockUserRepo) GetByID(_ context.Context, id int) (*User, error) {
    u, ok := m.users[id]
    if !ok { return nil, ErrNotFound }
    return u, nil
}

// 测试
func TestUserService_GetUser(t *testing.T) {
    mockRepo := &MockUserRepo{
        users: map[int]*User{
            1: {ID: 1, Name: "Izzy"},
        },
    }
    
    svc := NewUserService(mockRepo)
    user, err := svc.GetUser(context.Background(), 1)
    
    if err != nil || user.Name != "Izzy" {
        t.Errorf("unexpected result")
    }
}
```

### 如果真的想用工具生成 mock：

```bash
# go mockery（最流行）
go install github.com/vektra/mockery/v2@latest
mockery --all --dir=./repo

# 或者 go generate + testify/mock（轻量）
//go:generate mockgen -source=user_repo.go -destination=mock_user_repo.go
```

但说实话，**手写的 mock 往往比生成的更清晰**——尤其是你的接口不复杂的话。

## Q4：Benchmark 怎么写？性能优化靠它

**A：**

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(2, 3)
    }
}

func BenchmarkJSONMarshal(b *testing.B) {
    data := User{Name: "Izzy", Age: 99}
    b.ResetTimer() // 重置计时器，排除准备时间
    
    for i := 0; i < b.N; i++ {
        json.Marshal(data)
    }
}

// 并发 benchmark
func BenchmarkAddParallel(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            Add(2, 3)
        }
    })
}
```

运行：
```bash
go test -bench=. -benchmem
# 输出:
# BenchmarkAdd-8          1000000000    0.3 ns/op    0 B/op    0 allocs/op
# BenchmarkJSONMarshal-8  5000000      240 ns/op    96 B/op    2 allocs/op
```

关键指标：
- **ns/op** — 每次操作耗时
- **B/op** — 每次操作分配的内存
- **allocs/op** — 每次操作的内存分配次数

## Q5：TestMain 和 Setup/Teardown 怎么做？

**A：**

```go
func TestMain(m *testing.M) {
    // 全局 setup：连接测试数据库、初始化配置等
    fmt.Println("=== 测试开始 ===")
    
    code := m.Run() // 运行所有测试
    
    // 全局 teardown：清理资源
    fmt.Println("=== 测试结束 ===")
    os.Exit(code)
}

// 子测试级别的 setup/teardown 用 helper 函数
func setupTestDB(t *testing.T) *sql.DB {
    t.Helper()
    db, err := sql.Open("sqlite3", ":memory:")
    if err != nil { t.Fatal(err) }
    
    // 创建表结构
    db.Exec(`CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)`)
    
    // 注册 cleanup
    t.Cleanup(func() { db.Close() })
    
    return db
}

func TestCreateUser(t *testing.T) {
    db := setupTestDB(t)
    // 使用 db 进行测试，结束后自动关闭
}
```

`t.Helper()` 让错误报告指向调用位置而非 helper 内部。
`t.Cleanup()` 是 Go 1.14+ 的特性，按 LIFO 顺序执行清理。

## Q6：覆盖率怎么看？多少才算合格？

**A：**

```bash
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out  # 生成 HTML 报告，浏览器打开
go tool cover -func=coverage.out # 按函数查看覆盖率
```

**实际建议**：

| 类型 | 目标覆盖率 |
|------|-----------|
| 核心业务逻辑 | 80%+ |
| 工具函数 | 90%+（应该好测）|
| main / 入口文件 | 不强求 |
| 生成代码（protobuf 等） | 不需要测 |

**别追求 100% 覆盖率！** 为了覆盖而写的测试往往是无意义的（比如测试 getter/setter）。关注**关键路径和边界情况**更重要。

## Q7：Go 测试的最佳实践清单

**A：**

```
□ 用 table-driven test 组织多个测试用例
□ 每个包至少有一个 _test.go 文件
□ 测试函数名描述被测行为（TestAdd_WhenNegative_ReturnsSum）
□ 用 t.Run 做子测试，失败时快速定位
□ 外部依赖用接口 mock，不要连真实数据库
□ 性能敏感代码写 benchmark
□ 关键路径覆盖率 >80%
□ 不要测试私有函数（测试公有行为即可）
□ 测试之间相互独立，不依赖执行顺序
□ 使用 testify/assert 减少重复的 if err != nil 判断（可选）
```

---

## 本阶段回顾

| 篇 | 核心话题 | 关键收获 |
|---|---------|---------|
| go-05 | Error 处理 | %w 包装、sentinel error、panic 边界、API 层错误设计 |
| go-06 | Context 包 | WithTimeout/WithCancel、传播规则、WithValue 正确用法、常见反模式 |
| go-07 | Testing | Table-driven test、接口 mock、benchmark、t.Cleanup、覆盖率目标 |

接下来进入最后一站：**并发进阶篇**——Worker Pool、sync 深挖、以及从零搭一个 HTTP 服务把所有知识串起来。

> **TODO**: 选你项目里一个核心函数，给它补上 table-driven test——你会发现写测试的过程本身就是对代码的一次审查，很多 bug 就是在构造测试 case 时发现的。
