---
title: Go 工程实践（上）：Error 处理最佳实践
date: 2026-06-04
categories:
  - [Golang学习]
tags:
  - Go
  - Error
  - 工程实践
  - 最佳实践
  - 问答
top_img: linear-gradient(135deg, #fa709a 0%, #fee140 100%)
---

> `if err != nil` 写到怀疑人生？这篇整理 Go 错误处理的正确姿势——从基础到进阶，让你不再害怕 `error`。

---

## Q1：Go 的 error 到底是什么？为什么不用 try-catch？

**A：**

```go
// error 就是一个接口，就这：
type error interface {
    Error() string
}
```

Go 选择 **显式错误返回** 而非异常机制的原因：

| | try-catch (Java/Python) | error return (Go) |
|--|------------------------|-------------------|
| 控制流 | 可以跳过任意层 | **逐层显式处理** |
| 忽略错误 | 容易忘记 catch | 编译器强迫你写 `if err != nil` |
| 性能 | 异常有栈开销 | 零开销（就是个接口值） |
| 可读性 | 正常逻辑和错误处理混在一起 | 分离清晰 |

代价就是代码里到处是 `if err != nil`。但这是**特性不是 bug**——它逼你面对每一个可能出错的地方。

## Q2：三种创建 error 的方式分别什么时候用？

**A：**

### 方式一：errors.New / fmt.Errorf — 最简单

```go
// 简单错误
err := errors.New("something wrong")

// 带格式化的错误
err := fmt.Errorf("user %d not found", userID)
```

适用场景：简单的、不需要特殊判断的错误。

### 方式二：自定义错误类型 — 需要程序化判断

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

// 使用时可以类型匹配
func validate(user *User) error {
    if user.Age < 0 {
        return &ValidationError{Field: "age", Message: "年龄不能为负数"}
    }
    return nil
}
```

### 方式三：Sentinel Error — 预定义的哨兵错误

```go
var (
    ErrUserNotFound = errors.New("user not found")
    ErrUnauthorized = errors.New("unauthorized")
)

func getUser(id int) (*User, error) {
    // ...
    return nil, ErrUserNotFound // 返回预定义的哨兵错误
}

// 调用方可以用 errors.Is 精确匹配
if errors.Is(err, ErrUserNotFound) {
    // 创建新用户
}
```

**选择指南**：

| 场景 | 推荐方式 |
|------|---------|
| 简单的一次性错误 | `errors.New` / `fmt.Errorf` |
| 需要携带额外信息 | 自定义 error 类型 |
| 需要精确匹配做分支处理 | Sentinel Error + `errors.Is` |

## Q3：Error Wrapping（%w）怎么用？这是 Go 1.13 之后的标准

**A：** 在多层调用中保留完整错误链：

```go
func GetConfig() (*Config, error) {
    data, err := os.ReadFile("config.json")
    if err != nil {
        return nil, fmt.Errorf("read config file: %w", err) // 包装
    }

    cfg, err := parseConfig(data)
    if err != nil {
        return nil, fmt.Errorf("parse config: %w", err) // 继续包装
    }
    return cfg, nil
}
```

**%v vs %w 的区别**：

```go
fmt.Errorf("context: %v", err)  // 只记录文本，不保留类型
fmt.Errorf("context: %w", err)  // 保留原始 error 类型，可用 Is/As 解包
```

**解包错误链**：

```go
_, err := GetConfig()
if err != nil {
    fmt.Println(err)                    // "parse config: invalid JSON"
    fmt.Printf("%+v\n", err)            // 带堆栈的完整链

    // Is: 精确匹配链中任一错误
    if errors.Is(err, os.ErrNotExist) { ... }

    // As: 提取特定类型的错误
    var pathErr *os.PathError
    if errors.As(err, &pathErr) {
        fmt.Println("出错的文件:", pathErr.Path)
    }
}
```

**规则**：在库/包的边界处用 `%w` 包装，让调用者能获取完整的错误上下文。

## Q4：panic 和 error 的边界在哪里？

**A：** 这是个重要的设计决策：

```go
// ✅ 应该返回 error 的场景
func Divide(a, b float64) (float64, error) {
    if b == 0 { return 0, errors.New("division by zero") }
    return a / b, nil
}

// ✅ 可以 panic 的场景（真正的不可恢复错误）
func MustGetEnv(key string) string {
    val := os.Getenv(key)
    if val == "" {
        panic(fmt.Sprintf("required env %s is missing", key))
    }
    return val
}
```

**边界规则**：

| 场景 | 用 error | 用 panic |
|------|---------|---------|
| 用户输入无效 | ✅ | ❌ |
| 网络请求失败 | ✅ | ❌ |
| 文件不存在 | ✅ | ❌ |
| 程序启动配置缺失 | — | ✅ |
| 不可变的编程错误（如数组越界）| — | ✅ |
| 数据库连接失败（初始化阶段）| — | ✅ |

**defer + recover 兜底**：

```go
func SafeRun(fn func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic recovered: %v", r)
        }
    }()
    fn()
    return
}
```

注意：**不要用 panic/error 做正常控制流**。Java 里 `try-catch` 当 if-else 用的坏习惯别带到 Go 来。

## Q5：第三方错误库值得用吗？

**A：** 看情况。

```go
// pkg/errors（经典但已不维护）
import "github.com/pkg/errors"

// 带堆栈的错误
err := errors.Wrap(originalErr, "context")
errors.StackTrace(err) // 打印完整调用栈

// go-multierror（合并多个错误）
import "github.com/hashicorp/go-multierror"

var result *multierror.Error
result = multierror.Append(result, err1)
result = multierror.Append(result, err2)
return result.ErrorOrNil()
```

**我的建议**：

- **Go 1.13+ 内置的 errors 包已经够用了**，大部分场景不需要第三方库
- 如果确实需要带堆栈的错误信息，用 `github.com/pkg/errors`
- 合并多个错误场景用 `hashicorp/go-multierror`
- 别为了用而用，标准库优先

## Q6：API 层的错误怎么处理才规范？

**A：** 一个实用的分层错误处理模式：

```go
package handler

type APIError struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
    Details string `json:"details,omitempty"`
}

func (e *APIError) Error() string { return e.Message }

var (
    ErrBadRequest     = &APIError{Code: 400, Message: "Bad Request"}
    ErrUnauthorized   = &APIError{Code: 401, Message: "Unauthorized"}
    ErrNotFound       = &APIError{Code: 404, Message: "Not Found"}
    ErrInternalServer = &APIError{Code: 500, Message: "Internal Server Error"}
)

func HandleError(w http.ResponseWriter, r *http.Request, err error) {
    var apiErr *APIError
    if errors.As(err, &apiErr) {
        w.WriteHeader(apiErr.Code)
        json.NewEncoder(w).Encode(apiErr)
        return
    }
    // 未预期的错误，返回 500 + 记录日志
    log.Printf("[ERROR] unexpected: %v\n", err)
    w.WriteHeader(500)
    json.NewEncoder(w).Encode(ErrInternalServer)
}
```

这样业务逻辑只管返回 error，HTTP 层统一翻译成 API 响应。

## 下期预告

下一篇聊 **Context 包**——Go 并发控制的核心。超时控制、取消信号、跨 goroutine 传值，理解了 Context 才算真正会用 Go 写后端服务。

> **TODO**: 检查一下你项目里的错误处理——有没有裸 `panic()` 散落在业务代码中？有没有应该用 `errors.Is` 却在用 `==` 直接比较字符串的地方？
