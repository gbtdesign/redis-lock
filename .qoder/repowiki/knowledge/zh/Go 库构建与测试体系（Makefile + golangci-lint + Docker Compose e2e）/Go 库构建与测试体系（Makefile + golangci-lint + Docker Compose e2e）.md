---
kind: build_system
name: Go 库构建与测试体系（Makefile + golangci-lint + Docker Compose e2e）
category: build_system
scope:
    - '**'
source_files:
    - Makefile
    - .golangci.yml
    - script/docker-compose.yml
    - script/integrate_test.sh
    - script/setup.sh
    - script/goimports.sh
    - mocks/redis_cmdable.mock.go
---

## 1. 使用的系统与工具

- **语言与依赖**：Go 模块（`go.mod` / `go.sum`），最低 Go 版本由 `.golangci.yml` 声明为 `1.18`。
- **构建入口**：根目录 `Makefile`，所有本地开发任务通过 `make <target>` 触发。
- **单元测试**：`go test -race ./...`（并发安全检测开启）。
- **基准测试**：`go test -bench=. -benchmem ./...`。
- **代码质量**：`golangci-lint`（配置文件 `.golangci.yml`）+ `goimports`（脚本 `script/goimports.sh`）。
- **Mock 生成**：`mockgen`（目标包 `github.com/redis/go-redis/v9` 的 `Cmdable` 接口，输出到 `mocks/redis_cmdable.mock.go`）。
- **端到端测试**：基于 Docker Compose 启动 Redis 7.0（`bitnami/redis:7.0`），并通过 Go build tag `e2e` 过滤测试用例。
- **Git hooks**：`script/setup.sh` 将 `.github/pre-commit`、`.github/pre-push` 复制到 `.git/hooks/` 并安装 `golangci-lint`、`goimports`。

仓库中未发现 CI 流水线文件（如 GitHub Actions workflow）、Dockerfile 或发布脚本；构建与测试完全在本地通过 Makefile 驱动。

## 2. 关键文件

- `Makefile` — 全部构建/测试/lint/mock 目标的统一入口。
- `.golangci.yml` — golangci-lint 配置，锁定 Go 版本 `1.18`，跳过 `.idea`。
- `script/docker-compose.yml` — 定义集成测试所需的 Redis 服务（端口 `6379`，允许空密码）。
- `script/integrate_test.sh` — 编排 e2e 流程：down → up → `go test -race ./... -tags=e2e` → down。
- `script/setup.sh` — 初始化 Git hooks 并安装 lint 工具。
- `script/goimports.sh` — goimports 格式化脚本（被 `make fmt` 调用）。
- `mocks/redis_cmdable.mock.go` — mockgen 生成的 Redis Cmdable 接口 mock。

## 3. 架构与约定

- **Makefile 作为唯一入口**：所有可执行任务都通过 `.PHONY` 目标暴露，包括 `ut`、`bench`、`fmt`、`lint`、`tidy`、`check`、`setup`、`e2e`、`e2e_up`、`e2e_down`、`mock`。`check` 组合了 `fmt` 和 `tidy`，是提交前的推荐检查点。
- **e2e 与 UT 分离**：单元测试默认运行（`make ut`），端到端测试需要 `-tags=e2e` 编译标记，由 `script/integrate_test.sh` 自动设置。Redis 容器生命周期由该脚本管理（先 down 再 up，测试结束后 down）。
- **Lua 脚本双份存放**：`demo/lock.lua`、`demo/refresh.lua`、`demo/unlock.lua` 与 `script/lua/lock.lua`、`script/lua/refresh.lua`、`script/lua/unlock.lua` 各有一份，分别对应 demo 演示与集成测试环境使用。
- **Mock 生成固定化**：`make mock` 固定调用 mockgen 生成 `Cmdable` 接口的 mock 到 `mocks/` 目录，并使用 `.license.go.header` 作为版权头模板。
- **Git hook 初始化**：`make setup`（即 `sh ./script/setup.sh`）负责复制 pre-commit/pre-push 钩子并安装 golangci-lint 与 goimports，使本地开发环境与 lint 规则对齐。

## 4. 约定与约束

- **Go 版本约束**：`.golangci.yml` 中 `run.go: '1.18'` 强制 golangci-lint 以 Go 1.18 语义运行，构成最低版本约束。
- **e2e 测试必须带 tag**：`script/integrate_test.sh` 显式传入 `-tags=e2e`，因此任何依赖 Redis 的测试都必须用 `//go:build e2e` 或 `// +build e2e` 包裹，否则不会被 `go test ./...` 默认执行。
- **Redis 镜像与端口**：`script/docker-compose.yml` 固定使用 `docker.io/bitnami/redis:7.0`，映射宿主机 `6379`，且 `ALLOW_EMPTY_PASSWORD=yes`，意味着 e2e 环境不要求密码。
- **Lint 配置位置**：golangci-lint 通过 `make lint` 以 `-c .golangci.yml` 方式运行，所有规则变更需修改该文件。
- **Mock 源接口不可变**：mockgen 直接指向第三方包 `github.com/redis/go-redis/v9.Cmdable`，若上游接口变更需同步更新 `Makefile` 中的 mockgen 命令。
- **无 CI/Dockerfile/发布脚本**：仓库未包含 GitHub Actions workflow、Dockerfile 或自动化发布流程，构建与发布不在本仓库内实现（可能由外部系统或手动完成）。