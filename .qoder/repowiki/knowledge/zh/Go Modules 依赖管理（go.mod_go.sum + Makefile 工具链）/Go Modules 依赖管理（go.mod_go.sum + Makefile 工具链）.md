---
kind: dependency_management
name: Go Modules 依赖管理（go.mod/go.sum + Makefile 工具链）
category: dependency_management
scope:
    - '**'
source_files:
    - go.mod
    - go.sum
    - Makefile
    - script/goimports.sh
    - mocks/redis_cmdable.mock.go
---

## 1. 使用的系统/方案

本项目采用 Go 官方模块系统（`go mod`）进行第三方依赖声明与版本锁定，未使用 vendor 目录、GOPROXY 代理或私有仓库配置。所有依赖通过 `go.mod` 声明、`go.sum` 校验。

## 2. 关键文件

- `go.mod` — 模块根定义、直接依赖与间接依赖的版本清单
- `go.sum` — 依赖的 SHA256 校验和（由 `go mod tidy` 维护）
- `Makefile` — 提供 `tidy`、`check`、`mock` 等依赖相关目标
- `script/goimports.sh` — 代码格式化脚本（被 `make fmt` 调用）
- `mocks/redis_cmdable.mock.go` — 由 `mockgen` 生成的 mock 文件，其生成命令在 `Makefile` 中固定

## 3. 架构与约定

### 3.1 模块与 Go 版本
- 模块路径：`redis-lock`
- 指定 Go 版本：`go 1.26.3`（`go.mod` 第 3 行），意味着构建环境需匹配该版本

### 3.2 依赖分类
`go.mod` 将依赖分为两类：
- **直接依赖**（`require` 块）：`github.com/google/uuid`、`github.com/gotomicro/redis-lock`、`github.com/redis/go-redis/v9`、`github.com/stretchr/testify`、`go.uber.org/mock`、`golang.org/x/sync`
- **间接依赖**（`// indirect` 标记）：`cespare/xxhash`、`davecgh/go-spew`、`pmezard/go-difflib`、`go.uber.org/atomic`、`gopkg.in/yaml.v3`

### 3.3 版本策略
- 所有依赖均使用精确版本号（无 `~`、`^` 等范围前缀），例如 `v9.21.0`、`v1.11.1`、`v0.6.0`，便于可重复构建
- 测试框架使用 `stretchr/testify`，Mock 生成使用 `go.uber.org/mock`（即 `mockgen`）
- Redis 客户端使用 `github.com/redis/go-redis/v9`，并通过 `mockgen` 针对 `Cmdable` 接口生成 `mocks/redis_cmdable.mock.go`

### 3.4 工具链集成
`Makefile` 暴露以下与依赖相关的目标：
- `make tidy` → 执行 `go mod tidy -v`，同步 `go.mod` 与 `go.sum`
- `make check` → 先 `make fmt`（调用 `script/goimports.sh`），再 `make tidy`，作为统一的依赖/格式检查入口
- `make mock` → 使用 `mockgen -copyright_file=.license.go.header -package=mocks -destination=mocks/redis_cmdable.mock.go github.com/redis/go-redis/v9 Cmdable` 重新生成 mock
- `make lint` → 通过 `.golangci.yml` 运行 golangci-lint

### 3.5 构建与测试辅助
- `script/docker-compose.yml` 用于 e2e 测试启动 Redis 实例
- `script/integrate_test.sh` 运行集成测试
- `script/setup.sh` 初始化环境

## 4. 约定与约束

- **版本锁定**：`go.mod` 中所有依赖均为精确版本，无版本范围符号；`go.sum` 存在并由 `go mod tidy` 维护（`Makefile` 中的 `tidy` 目标）。
- **Go 版本约束**：`go.mod` 显式声明 `go 1.26.3`，构建必须使用该版本或兼容版本。
- **Mock 生成规范**：mock 文件位于 `mocks/` 目录，通过 `Makefile` 的 `mock` 目标统一生成，并附带 `.license.go.header` 版权头。
- **依赖更新流程**：应通过 `make tidy`（即 `go mod tidy -v`）来同步依赖，而非手动编辑 `go.mod`。
- **无私有仓库/GOPRIVATE 配置**：未发现 GOPRIVATE、GOPROXY 或 go.sum 之外的私有源配置，所有依赖来自公共 Go 模块代理。
- **无 vendor 目录**：项目未启用 `go mod vendor`，依赖以远程下载方式获取。

注意：`go.mod` 中声明了 `github.com/gotomicro/redis-lock v0.0.3` 作为依赖，与本仓库同名但为不同包，属于外部依赖引用。