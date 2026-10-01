---
kind: configuration_system
name: rlock 库的配置方式：构造函数注入与嵌入脚本
category: configuration_system
scope:
    - '**'
source_files:
    - lock.go
    - retry.go
    - go.mod
    - script/lua/lock.lua
    - script/lua/refresh.lua
    - script/lua/unlock.lua
---

## 概述

本仓库是一个 Go 分布式锁库（`redis-lock`），本身是**被调用的库**，而非独立运行的应用。因此不存在传统意义上的“应用配置系统”（如 `config.yaml`、`.env`、环境变量加载器、feature flag 等）。库的“配置”通过 **Go 构造函数的参数注入** 和 **编译期嵌入资源** 两种方式完成。

## 1. 运行时配置入口：构造函数注入

库的唯一公共入口是 `NewClient(client redis.Cmdable) *Client`（见 `lock.go:53-60`）：

- `client redis.Cmdable`：Redis 客户端连接由调用方传入，库内部不创建、不管理连接池或连接地址。
- `valuer func() string`：锁值的生成策略在 `NewClient` 中默认使用 `uuid.New().String()`，但字段暴露为可替换函数，注释写明“将来可以考虑暴露出去允许用户自定义”。

所有行为参数（重试策略、超时、续期间隔）均以方法参数的形式传递：

- `Lock(ctx, key, expiration, retry, timeout)` — `RetryStrategy` 接口定义在 `retry.go`，内置实现 `FixIntervalRetry`（固定间隔 + 最大次数）。
- `AutoRefresh(interval, timeout)` — 续期间隔与刷新超时直接作为参数传入。
- `SingleflightLock(...)` — singleflight 去抖以 key 为粒度，由库内部维护 `singleflight.Group`。

这意味着：**库本身没有配置文件加载逻辑；所有可调参数均由调用方在启动时显式构造。**

## 2. 静态资源：Lua 脚本通过 `//go:embed` 嵌入

三个 Lua 脚本（`script/lua/lock.lua`、`refresh.lua`、`unlock.lua`）通过 Go 的 `embed` 指令在编译期打包进二进制：

```go
//go:embed script/lua/unlock.lua
luaUnlock string
//go:embed script/lua/refresh.lua
luaRefresh string
//go:embed script/lua/lock.lua
luaLock string
```

这些脚本不是“运行时可编辑的配置”，而是原子性加锁/解锁/续期逻辑的实现载体，随库一起发布。

## 3. 测试与集成环境的“配置”

仓库中与外部依赖相关的唯一“配置”位于 `script/docker-compose.yml` 和 `script/setup.sh`，用于本地运行 Redis 实例以执行集成测试（`script/integrate_test.sh`）。这是测试基础设施配置，不属于库的运行配置。

## 4. 约束与约定

- 库不提供任何 `init()` 全局状态初始化；所有状态（Redis 连接、singleflight group、valuer）均绑定到 `*Client` 实例。
- 错误语义通过包级变量暴露（`ErrFailedToPreemptLock`、`ErrLockNotHold`），调用方应使用 `errors.Is` 判断（见 `lock.go:86-93` 的注释说明）。
- 重试策略通过接口 `RetryStrategy` 解耦，调用方可自行实现指数退避等策略，但库只内置 `FixIntervalRetry`。
- 无环境变量读取、无 YAML/TOML 解析、无远程配置中心集成——这些能力需要调用方在自身应用中提供，并传入 `redis.Cmdable`。

## 结论

该仓库不包含传统的应用配置系统。其“配置模型”是纯代码级的：构造函数参数 + 接口注入 + 编译期嵌入脚本。这符合 Go 库的典型设计——把连接、超时、重试等行为全部交给调用方控制，避免在库内部引入隐式的文件/环境变量加载路径。