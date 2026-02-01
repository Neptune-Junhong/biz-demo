# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

注意全程使用中文和我沟通！

## 仓库概览

CloudWeGo 生产级业务示例仓库，包含 5 个独立的微服务演示项目，均使用 Go 语言，基于 Hertz（HTTP 框架）和 Kitex（RPC 框架）构建。

## 项目结构

| 项目 | 说明 | 关键技术 |
|------|------|---------|
| **bookinfo/** | Istio Service Mesh 集成示例 | kitex-xds, OpenTelemetry, Wire DI |
| **book-shop/** | 电商平台（商品、用户、订单） | GORM, Redis, Elasticsearch, etcd |
| **easy_note/** | 笔记服务（Hertz+Kitex 协作入门） | hz/kitex 代码生成, OpenTelemetry, etcd |
| **open-payment-platform/** | 支付网关（泛化调用） | Kitex generic call, ent ORM, Nacos, Clean Architecture |
| **gomall/** | 电商教学项目（含17章教程） | cwgo 代码生成, Consul, Go workspace |

每个项目都是独立的 Go module，有自己的 `go.mod`。gomall 使用 `go.work` 管理多模块工作区。

## 通用架构模式

所有项目遵循相同的微服务模式：**Hertz 作为 HTTP 网关 + Kitex 作为 RPC 后端**。

- HTTP 入口层（Hertz）接收请求，通过 RPC 调用后端服务
- 每个 RPC 服务独立部署，有自己的 `handler.go`（业务逻辑）和 `main.go`（启动入口）
- 生成的代码在 `kitex_gen/`（Kitex）和 `biz/`或`hertz_gen/`（Hertz）目录中，**不要手动编辑**
- IDL 文件（`.thrift` / `.proto`）在各项目的 `idl/` 目录中

## 常用命令

### 全局测试和检查（在仓库根目录执行）

```bash
# 运行所有模块的单元测试（遍历所有 go.mod，排除 tutorial/vendor/third_party）
make test

# 运行所有模块的 go vet
make vet
```

`hack/util.sh` 中的 `util::find_modules` 负责发现所有需要测试的模块，排除 `gomall/tutorial`、`gomall/rpc_gen`、`vendor`、`third_party`。

### 单个项目测试

```bash
cd <project-dir>  # 如 bookinfo, book-shop, easy_note 等
go test -race -covermode=atomic -coverprofile=coverage.out ./...
```

### Lint

CI 使用 golangci-lint v8，配置文件为根目录 `.golangci.yml`，对每个模块单独执行：

```bash
golangci-lint run --config=.golangci.yml
```

格式化使用 gofumpt（带 extra-rules）。CI 中使用 `--only-new-issues` 只检查新增问题。

### 代码生成

**Kitex RPC 代码**（从 Thrift IDL）：
```bash
kitex -module <module-path> idl/<service>.thrift
```

**Hertz HTTP 代码**（从 Thrift/Proto IDL）：
```bash
hz new -idl idl/<service>.thrift
```

**Gomall 使用 cwgo**（从 Proto IDL），通过 Makefile：
```bash
cd gomall
make gen svc=<service-name>          # 生成完整代码
make gen-client svc=<service-name>   # 仅生成客户端
make gen-server svc=<service-name>   # 仅生成服务端
```

### Gomall 项目特有命令

```bash
cd gomall
make init           # 初始化 .env 配置（首次执行）
make run svc=<name> # 运行指定服务
make tidy           # 所有模块执行 go mod tidy
make lint           # gofmt + gofumpt 格式化
make env-start      # docker-compose 启动中间件
make env-stop       # docker-compose 停止中间件
```

### 其他项目启动基础设施

```bash
cd book-shop && make start      # 启动 docker-compose 依赖
cd open-payment-platform && make start
```

## CI/PR 检查

PR 会自动执行三类检查：

1. **License Header**：Apache-2.0 许可证头检查（skywalking-eyes），所有 `.go` 和 `.s` 文件必须包含许可证头，生成代码除外（见 `.licenserc.yaml`）
2. **Typos**：拼写检查（crate-ci/typos），配置在 `_typos.toml`
3. **Lint**：golangci-lint 对每个模块独立运行（通过 `hack/resolve-modules.sh` 动态生成矩阵）

## 关键约定

- Go 版本：1.23，所有模块需要 `replace github.com/apache/thrift => github.com/apache/thrift v0.13.0`
- 新增 `.go` 文件必须包含 Apache-2.0 许可证头（Copyright CloudWeGo Authors）
- Commit message 遵循 AngularJS 约定
- `gomall/tutorial/` 是教程代码，不参与 CI 测试和 lint
