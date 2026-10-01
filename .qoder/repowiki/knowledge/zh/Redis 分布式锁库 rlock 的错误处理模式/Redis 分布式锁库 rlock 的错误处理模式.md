---
kind: error_handling
name: Redis 分布式锁库 rlock 的错误处理模式
category: error_handling
scope:
    - '**'
source_files:
    - lock.go
    - retry.go
    - demo/demo.go
---

## 1. 使用的错误处理系统

该仓库是一个 Go 语言实现的 Redis 分布式锁库，采用 **标准库 `errors` + `context` 哨兵错误** 的方式，没有引入第三方错误包装库（如 `pkg/errors`、`xerrors`）。核心策略是：
- 定义包级哨兵错误变量（sentinel errors），通过 `errors.New` 创建。
- 使用 `errors.Is` 进行错误匹配。
- 对可恢复的临时失败（如锁被占用、超时）配合重试策略；对不可恢复的网络/服务端错误直接向上返回。
- 没有使用 `panic/recover` 作为正常控制流。

## 2. 关键文件与包

- `lock.go`：定义包级哨兵错误、`Client` 和 `Lock` 类型的所有错误路径。
- `retry.go`：定义 `RetryStrategy` 接口与 `FixIntervalRetry` 实现，决定何时停止重试并产生最终错误。
- `demo/demo.go`：在示例中复用同一套错误语义（`ErrLockNotHold`、`ErrFailedToPreemptLock`）。

## 3. 架构与约定

### 3.1 哨兵错误定义

`lock.go` 中定义了两个包级哨兵错误：

```go
ErrFailedToPreemptLock = errors.New("rlock: 抢锁失败")
ErrLockNotHold         = errors.New("rlock: 未持有锁")
```

- `ErrFailedToPreemptLock`：表示加锁竞争失败（锁已被他人持有或最后一次重试耗尽）。
- `ErrLockNotHold`：表示调用方预期自己持有锁但实际未持有（例如解锁时 key 不存在或被他人覆盖），注释明确说明这通常意味着有人绕过了 rlock 直接操作 Redis。

这两个错误都带有 `rlock:` 前缀，便于日志过滤与识别来源。

### 3.2 错误传播与包装规则

`Client.Lock` 方法（第 84–93 行的注释）明确规定了最终返回 error 的语义：

| 条件 | 含义 |
|---|---|
| `errors.Is(err, context.DeadlineExceeded)` | 整体调用超时，无法确定是否加锁成功 |
| `errors.Is(err, ErrFailedToPreemptLock)` | 肯定没成功，且重试次数已耗尽 |
| 其他错误 | Redis 通信出错（网络异常、EOF、服务端崩溃等） |

具体实现中：
- 非超时的底层错误直接返回，不重试（第 106–110 行）。
- 当重试机会耗尽时，用 `fmt.Errorf("rlock: 重试机会耗尽，%w", err)` 包装最终错误，并通过 `%w` 保留原始错误以便 `errors.Is` 仍能穿透到 `ErrFailedToPreemptLock` 或底层的 `context.DeadlineExceeded`（第 117–121 行）。

### 3.3 不同 API 的错误语义

- `Client.Lock`：带重试。返回错误可能是 `context.DeadlineExceeded`、`ErrFailedToPreemptLock` 或其包装，也可能是底层 Redis 错误。
- `Client.SingleflightLock`：基于 `singleflight.Group.DoChan` 去抖，透传 `Lock` 的错误（第 73–75 行）。
- `Client.TryLock`：无重试。竞争失败直接返回 `ErrFailedToPreemptLock`（第 146 行）。
- `Lock.Refresh`：Lua 脚本返回值不为 1 时返回 `ErrLockNotHold`（第 225–227 行）。
- `Lock.Unlock`：`redis.Nil` 或 Lua 返回值不为 1 时返回 `ErrLockNotHold`（第 241–248 行）。

### 3.4 重试策略接口

`retry.go` 定义了 `RetryStrategy` 接口：

```go
type RetryStrategy interface {
    Next() (time.Duration, bool)
}
```

`Next()` 返回下一次重试间隔与是否继续重试。`FixIntervalRetry` 提供固定间隔重试实现。当 `Next()` 返回 `false` 时，调用方将其视为“重试耗尽”，构造最终错误。

### 3.5 上下文取消与超时

所有对外暴露的方法都接受 `context.Context`，并在以下位置检查 `ctx.Done()`：
- `SingleflightLock`：等待 singleflight 结果时监听 `ctx.Done()`（第 78–80 行）。
- `Lock`：每次重试前的定时器 select 监听 `ctx.Done()`（第 128–132 行）。
- `AutoRefresh`：内部使用 `context.WithTimeout` 包裹每次刷新调用（第 181、199 行）。

## 4. 约定与约束

- **禁止 panic 作为业务错误路径**：代码中没有 `panic` 调用；唯一一处 `defer func() { ... }` 用于防止重复解锁导致 channel 写入 panic（第 234–240 行），通过 `sync.Once` 保证只发送一次信号。
- **错误匹配必须使用 `errors.Is`**：文档注释明确要求调用方使用 `errors.Is` 判断返回错误（第 86–93 行），而不是字符串比较。
- **包装错误需使用 `%w`**：重试耗尽时的包装错误使用 `fmt.Errorf("... %w", err)`，确保 `errors.Is` 能穿透到哨兵错误。
- **Redis 特定错误映射**：`redis.Nil` 被显式转换为 `ErrLockNotHold`，将底层库错误归一化为领域错误。
- **上下文超时语义区分**：`context.DeadlineExceeded` 与 `ErrFailedToPreemptLock` 有明确的语义差异——前者不确定加锁结果，后者确定失败，调用方需要分别处理。
- **Demo 与库共享错误语义**：`demo/demo.go` 重新定义了同名的 `ErrLockNotHold` 和 `ErrFailedToPreemptLock`（第 32–33 行），与库中的错误独立存在，属于示例代码的本地副本，并非引用库中的错误变量。