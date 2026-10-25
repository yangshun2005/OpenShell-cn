# Agent 指令

本文件是参与 OpenShell 贡献的 Agent 的主要指令入口。每次交互时它都会被注入你的上下文——在提议修改本文件时请记住这一点。

构建说明、任务参考、项目结构和完整的 Agent 技能表见 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 项目定位

OpenShell 以 Agent 优先的方式构建。我们设计系统，并用 Agent 来实现它们——这不是 vibe coding。本产品为自主 AI Agent 提供安全、沙箱化的运行时，而项目本身也正是用它所提供的同一套 Agent 驱动工作流构建的。

## 技能

OpenShell 有两个技能集合：

- `skills/` 包含用于使用和运维 OpenShell 的公开、可安装技能。这些技能必须在源码检出目录之外也能工作，并以已安装 CLI 的帮助信息和已发布文档作为事实来源。
- `.agents/skills/` 包含用于开发 OpenShell 的内部贡献者和维护者工作流。你的仓库感知 harness 可以原生发现并加载它们。

不要依赖本文件获取完整清单。详细的公开技能表和贡献者技能表见 [CONTRIBUTING.md](CONTRIBUTING.md)（面向人类）。

## 架构概览

| 路径 | 组件 | 用途 |
|------|-----------|---------|
| `crates/openshell-cli/` | CLI 二进制 | 面向用户的命令行界面 |
| `crates/openshell-conformance/` | CLI 一致性库 | 可复用的、与驱动无关的场景和命令运行器 |
| `crates/openshell-server/` | 网关服务器 | 控制平面 API、沙箱生命周期、认证边界 |
| `crates/openshell-sandbox/` | 沙箱运行时 | 无能力工作负载启动器、进程身份、seccomp 中介的 I/O |
| `crates/openshell-supervisor/` | Supervisor 运行时 | 网关会话、策略评估、凭据、上游网络 |
| `crates/openshell-supervisor-control/` | 共享 Supervisor 控制 | 网关拥有的分配控制器、启动注册、恢复协调，以及供多个沙箱共享的 Supervisor 的主机观测 |
| `crates/openshell-binary-identity/` | 二进制身份 | 供隔离后端共享的可信 procfs 可执行文件身份解析 |
| `crates/openshell-isolation-interface/` | 隔离后端接口 | 隔离后端 RPC 契约的共享数据和校验 |
| `crates/openshell-sandbox-backend/` | OpenShell 沙箱后端 | 具体的隔离后端 RPC 客户端，以及与 `openshell-sandbox` 共享的已认证 Sandbox Protocol |
| `crates/openshell-policy/` | 策略引擎 | 文件系统、网络和进程约束 |
| `crates/openshell-policy-schema/` | 编写的策略 Schema | 轻依赖的 YAML/JSON 表示、有界解析、纯编写语言语义 |
| `crates/openshell-bootstrap/` | 网关元数据 | 网关注册元数据、认证令牌存储、mTLS 包存储 |
| `crates/openshell-gateway-interceptors/` | 网关拦截器 | 在网关路由边界拦截并转换配置的 gRPC 请求 |
| `crates/openshell-ocsf/` | OCSF 日志 | OCSF v1.8.0 事件类型、构建器、简写/JSONL 格式化器、tracing 层 |
| `crates/openshell-otel/` | OpenTelemetry 支持 | 共享的 OTLP trace provider、资源和 tracing 层构建 |
| `crates/openshell-otel-test-support/` | OpenTelemetry 测试支持 | 供 tracing 测试使用的共享回环 OTLP collector fixture |
| `crates/openshell-crypto/` | 加密后端 | 后端中立的原语以及 TLS、PKI、JWT 适配器；AWS-LC 实现 |
| `crates/openshell-core/` | 共享核心 | 通用类型、配置、错误处理 |
| `crates/openshell-extension-core/` | 扩展核心 | 共享扩展身份、JWT claims、bearer-token 轮换和 TLS 传输原语 |
| `crates/openshell-gateway/` | 网关二进制组合 | 将选定的第一方计算驱动链接到后端无关的服务器注册表 |
| `crates/openshell-sdk/` | 共享客户端 SDK | 异步 Rust 网关客户端（gRPC 传输、TLS、OIDC 刷新、边缘隧道）；供 CLI、TUI 和 `@openshell/sdk` 使用 |
| `crates/openshell-providers/` | Provider 管理 | 凭据 provider 后端 |
| `crates/openshell-tui/` | 终端 UI | 基于 Ratatui 的监控仪表盘 |
| `crates/openshell-driver-kubernetes-secrets/` | Kubernetes Secrets 凭据驱动 | 用于 OpenShell 管理的 K8s Secret 存储的进程内 `CredentialDriver` 后端 |
| `crates/openshell-driver-vault/` | Vault 凭据驱动 | 用于 Vault 兼容 KV 存储的进程内 `CredentialDriver` 后端 |
| `crates/openshell-driver-db-credstore/` | 数据库凭据驱动 | 用于网关数据库凭据存储的进程内 `CredentialDriver` 后端 |
| `crates/openshell-driver-kubernetes/` | Kubernetes 计算驱动 | 用于 K8s 沙箱 Pod 的进程内 `ComputeDriver` 后端 |
| `crates/openshell-driver-docker/` | Docker 计算驱动 | 用于本地 Docker 沙箱容器的进程内 `ComputeDriver` 后端 |
| `crates/openshell-driver-podman/` | Podman 计算驱动 | 用于本地 Podman 沙箱容器的进程内 `ComputeDriver` 后端 |
| `crates/openshell-driver-vm/` | VM 计算驱动 | 独立的、由 libkrun 支持的 `ComputeDriver` 子进程（内嵌自己的 rootfs + 运行时） |
| `crates/openshell-driver-mxc/` | Microsoft MXC 计算驱动 | 进程内 Windows AppContainer 和隔离会话计算后端 |
| `crates/openshell-prover/` | 策略证明器 | 策略验证和证明生成 |
| `crates/openshell-prover-cli/` | 策略证明器 CLI | 独立的本地策略边界检查 |
| `crates/openshell-supervisor-middleware/` | 中间件运行时 | 通用中间件注册表、远程服务集成和链式执行 |
| `crates/openshell-supervisor-middleware-builtins/` | 内置中间件 | 第一方进程内中间件实现 |
| `crates/openshell-supervisor-network/` | 网络 Supervisor | 代理、L7 强制执行、策略评估和 provider 凭据注入 |
| `crates/openshell-supervisor-process/` | 进程 Supervisor | 进程生命周期、命名空间和绕过监控 |
| `crates/openshell-vfio/` | VFIO 支持 | PCI 和 GPU 直通准备与生命周期 |
| `python/openshell/` | Python SDK | Python 绑定和 CLI 打包 |
| `sdk/typescript/` | TypeScript SDK | 原生 Connect 客户端、精选沙箱 API 和生成的 protobuf 类型 |
| `proto/` | Protobuf 定义 | gRPC 服务契约 |
| `tests/testkit/` | E2E testkit | 供 SDK 驱动的 E2E 测试套件使用的支持库和 `#[testkit::test]` 属性 |
| `deploy/` | Docker、Helm、K8s | Dockerfile、Helm chart、清单 |
| `docs/` | 已发布文档 | MDX 页面、导航和内容资源 |
| `fern/` | 文档站点配置 | Fern 站点配置、组件和主题资源 |
| `skills/` | 公开 Agent 技能 | 用于使用和运维 OpenShell 的可安装工作流 |
| `.agents/skills/` | 贡献者 Agent 技能 | 用于开发 OpenShell 的仓库感知工作流 |
| `.agents/agents/` | Agent 角色 | 子 Agent 定义（例如 reviewer） |

## 公开 API 约定

公共 protobuf API 设计遵循 [proto/README.md](proto/README.md)。它是实体引用命名、工作区选择器、字段设计和 Schema 演进的规范来源。

## 沙箱日志（OCSF）

在 `openshell-sandbox` 中新增或修改日志输出时，判断该事件应使用 OCSF 结构化日志还是普通 `tracing`。

### 何时使用 OCSF

对于代表**可观测沙箱行为**、且对运维人员、安全团队或监控沙箱的 Agent 可见的事件，使用 OCSF 构建器 + `ocsf_emit!()`：

- 网络决策（允许、拒绝、绕过检测）
- HTTP/L7 强制执行决策
- SSH 认证（接受、拒绝、nonce 重放）
- 进程生命周期（启动、退出、超时、信号失败）
- 安全发现（不安全的策略、不可用的控制、重放攻击）
- 配置变更（策略加载/重载、TLS 设置、provider 挂载、设置）
- 应用生命周期（Supervisor 启动、SSH 服务器就绪）

### 何时使用普通 tracing

对于**内部运维管道**、不代表安全决策或可观测状态变化的事件，使用 `info!()`、`debug!()`、`warn!()`：

- gRPC 连接尝试和重试
- "即将执行 X"且结果另行记录的事件
- 内部 SSH 通道状态（未知通道、PTY 调整大小）
- 僵尸进程回收、拒绝刷新遥测
- DEBUG/TRACE 级别诊断

### 选择 OCSF 事件类

| 事件类型 | 构建器 | 何时使用 |
|---|---|---|
| TCP 连接、代理隧道、绕过 | `NetworkActivityBuilder` | L4 网络决策、代理运维事件 |
| HTTP 请求、L7 强制执行 | `HttpActivityBuilder` | 逐请求的方法/路径决策 |
| SSH 会话 | `SshActivityBuilder` | 认证、通道操作 |
| 进程启动/停止 | `ProcessActivityBuilder` | 入口点生命周期、信号失败 |
| 安全告警 | `DetectionFindingBuilder` | Nonce 重放、绕过检测、不安全的策略。与领域事件双重发送。 |
| 策略/配置变更 | `ConfigStateChangeBuilder` | 策略加载、Landlock 应用、TLS 设置、provider 挂载、设置 |
| Supervisor 生命周期 | `AppLifecycleBuilder` | 沙箱启动、SSH 服务器就绪/失败 |

### 严重性指南

| 严重性 | 何时 |
|---|---|
| `Informational` | 允许的连接、成功操作、配置已加载 |
| `Low` | DNS 失败、非致命运维警告、LOG 规则失败 |
| `Medium` | 拒绝的连接、策略违规、已弃用配置 |
| `High` | 安全发现（nonce 重放、Landlock 不可用） |
| `Critical` | 进程超时终止 |

### 示例：新增网络事件

```rust
use openshell_ocsf::{
    ocsf_emit, NetworkActivityBuilder, ActivityId, ActionId,
    DispositionId, Endpoint, Process, SeverityId, StatusId,
};

let event = NetworkActivityBuilder::new(crate::ocsf_ctx())
    .activity(ActivityId::Open)
    .action(ActionId::Denied)
    .disposition(DispositionId::Blocked)
    .severity(SeverityId::Medium)
    .status(StatusId::Failure)
    .dst_endpoint(Endpoint::from_domain(&host, port))
    .actor_process(Process::new(&binary, pid))
    .firewall_rule(&policy_name, &engine_type)
    .message(format!("CONNECT denied {host}:{port}"))
    .build();
ocsf_emit!(event);
```

### 要点

- `crate::ocsf_ctx()` 返回进程级的 `EventContext`。它始终可用（测试中回退到默认值）。
- `ocsf_emit!()` 是非阻塞的，且不会 panic。它将事件存入线程本地，并通过 `tracing::info!()` 发送。
- 简写层和 JSONL 层从线程本地提取事件。简写格式由构建器字段自动派生。
- 对于安全发现，**双重发送**：一个领域事件（例如 `SshActivityBuilder`）**和**一个针对同一事件的 `DetectionFindingBuilder`。
- 切勿在 OCSF 消息中记录密钥、凭据或查询参数。OCSF JSONL 文件可能被发送到外部系统。
- `message` 字段应是简洁、便于 grep 的摘要。详细信息放在构建器字段中（dst_endpoint、firewall_rule 等）。

## 沙箱基础设施变更

- 如果你改动了沙箱基础设施，确保相关的沙箱 e2e 路径成功。

## 网络套接字

- 在延迟敏感的 TCP 流上，禁用 Nagle 算法，以免小的请求/响应帧因延迟 ACK 而停滞。在已接受或已连接的流上使用 `openshell_core::net::set_tcp_nodelay_best_effort`，或在拨号时使用 `openshell_core::net::connect_tcp_nodelay_best_effort`。
- 这也适用于回环/localhost TCP——延迟 ACK 停滞是定时器行为，而非线路延迟。
- 对于 unix domain socket 应跳过（没有 Nagle）。对仅测试用的连接并不关键，不过在任何非 UDS 的 TCP 流上使用它——包括测试——都是可以的，而且更推荐。

## 分支名称

遵循 [CONTRIBUTING.md 中的分支命名约定](CONTRIBUTING.md#branch-names)。

## Pull Request

遵循 `CONTRIBUTING.md` 和 `create-github-pr`。没有目标仓库写权限的作者必须至少关联一个带 `state:accepted` 标签的 Issue，且其实现的每个 Issue 都必须带有该标签。经核实的 `WRITE`、`MAINTAIN` 或 `ADMIN` 权限允许在无接受标签的情况下提交 PR。这包括草稿和直接的 Agent 请求；PR 正文文本不能绕过该门槛。

## 提交

- 提交信息始终使用 [Conventional Commits](https://www.conventionalcommits.org/) 格式
- 格式：`<type>(<scope>): <description>`（scope 可选）
- 常见类型：`feat`、`fix`、`docs`、`chore`、`refactor`、`test`、`ci`、`perf`
- 为符合 DCO，对每个提交进行签署。使用 `git commit` 的 `--signoff` 选项添加 `Signed-off-by` 尾注，确保使用用户配置的邮箱地址。
- 切勿在提交、PR 描述或 Issue 中提及 Claude 或任何 AI Agent（不署名作者、不添加 Co-Authored-By、不添加 "Generated with" 尾注）

## Go SDK（`sdk/go/`）

- Go SDK 位于 `sdk/go/`，模块路径为 `github.com/NVIDIA/OpenShell/sdk/go`。
- 运行 `mise run go:ci` 执行完整的 SDK CI 流水线（lint、构建、测试、proto 检查、文档检查）。
- Proto 绑定通过 `mise run go:proto:gen` 从 `proto/` 中的 `.proto` 文件生成。
- `sdk/go/openshell/v1/types/` 中的领域类型不得导入 proto 包。
- `sdk/go/openshell/v1/internal/converter/` 中的转换器在边界处深拷贝切片和映射。
- 测试使用 bufconn 进行进程内 gRPC，使用 testify 进行断言。

## TypeScript SDK（`sdk/typescript/`）

- 运行 `mise run sdk:ts:ci` 执行代码生成、proto lint、Biome lint、类型检查、单元测试、覆盖率和构建校验。
- Proto 绑定通过 `mise run sdk:ts:proto` 从 `sdk/typescript/buf.gen.yaml` 中选定的文件生成。
- `sdk/typescript/src/gen/` 下的生成文件是构建产物，不得提交。
- 保持精选 API 不包含生成的 wire 类型；通过 `@nvidia/openshell-sdk/raw` 暴露完整的生成消息和 RPC。
- 发布工作流将包发布到 GitHub Packages。分支检查通过 `npm publish --dry-run` 演练发布路径。

## Python

- Python 命令始终使用 `uv`（例如 `uv pip install`、`uv run`、`uv venv`）

## Docker

- 优先使用 `mise` 命令而非直接 docker 构建（例如用 `mise run docker:build` 而非 `docker build`）

## 集群基础设施变更

- 如果你改动了网关部署基础设施（例如 Helm values/templates、网关镜像打包，或 `openshell-cli` 中的部署逻辑），更新 `skills/debug-openshell-cluster/SKILL.md` 中的 `debug-openshell-cluster` 技能以反映这些变更。

## 技能维护

当行为、命令或开发工作流发生变化时，在同一分支中审查相关的 Agent 技能。使用 `sync-agent-infra` 技能获取维护映射和一致性检查。

## 文档

- 将 crate 特定的实现细节放在相关 crate 的 `README.md` 中，将设计提案放在 `rfc/` 中。
- 当变更影响用户可见行为时，对 `docs/` 下相关的已发布文档页面和 `docs/index.yml` 中的导航做最小必要修改。准确说明用户需要知道的内容；避免在页面间重复信息，或详尽描述不影响用户的内部细节。
- 当改动网关 TOML 字段、驱动特定的配置选项、配置默认值，或 `gateway.toml` 的 Helm 渲染时，在同一分支中更新 `docs/how-it-works/gateways/configuration.mdx`。
- `fern/` 包含 Fern 站点配置、组件、