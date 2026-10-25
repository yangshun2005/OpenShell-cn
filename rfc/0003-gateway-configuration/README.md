---
authors:
  - "@TaylorMutch"
state: implemented
---

# RFC 0003 - Gateway 配置文件

## 摘要

为 OpenShell gateway 引入基于 TOML 的配置文件，将 gateway 设置——核心服务器选项、TLS、OIDC、可观测性监听器以及各驱动参数——统一到单个结构化文件中。CLI 标志和受支持的 `OPENSHELL_*` 环境变量仍保持更高优先级。Schema 版本 2 有意拒绝旧版文件字段和位置。

## 动机

在本 RFC 之前，gateway 仅通过 CLI 标志和 `OPENSHELL_*` 环境变量配置。这对简单的单节点部署可行，但随着部署规模增长而失效：

- **标志过多**——gateway 暴露了约 40 个可配置参数（TLS、OIDC、四个计算驱动、三个监听器）。冗长的 `docker run` 命令和 Kubernetes 清单中的 `args:` 数组难以阅读、对比和审计。
- **驱动耦合**——Docker、Podman、Kubernetes 和 VM 驱动共享一个扁平的 CLI 命名空间，没有结构分离。大多数标志只适用于一个驱动，但 CLI 语法无法表达这种归属关系。
- **Helm 摩擦**——chart 的 `statefulset.yaml` 携带一长串 `OPENSHELL_*` 变量的 `env:` 块，每个都映射到 `values.yaml` 中的一个键。挂载配置文件可减少 chart 的模板化面积。
- **密钥管理**——仅环境变量的配置无法自然与 Kubernetes `ConfigMap` 和投射的 `Secret` 卷组合。

## 非目标

- 沙箱工作负载策略（OPA 规则、网络规则）——沙箱通过控制平面 API 从 gateway 接收策略；本 RFC 不改变这一点。
- 不重启 gateway 进程的热重载配置。
- 支持 TOML 以外的配置格式。不计划 JSON 或 YAML 变体。
- CLI 客户端（`openshell` 二进制）的新配置 schema——本 RFC 仅涵盖服务器进程（`openshell-gateway`）。
- 计算驱动的自动检测逻辑。gateway 已在未配置驱动时自动检测活动驱动（Kubernetes → Podman → Docker）；文件只是提供了另一种显式指定驱动的方式。

## 提案

### 配置来源与优先级

启动时合并三个来源，优先级从高到低：

```
CLI 标志  >  OPENSHELL_* 环境变量  >  TOML 配置文件  >  内置默认值
```

TOML 文件是可选的。如果既未设置 `--config` 也未设置 `OPENSHELL_GATEWAY_CONFIG`，gateway 行为与之前完全一致。文件中的任何字段都会被 CLI 标志或匹配的环境变量覆盖。

### 加载文件

文件路径通过以下方式提供：

```
--config /path/to/gateway.toml
OPENSHELL_GATEWAY_CONFIG=/path/to/gateway.toml
```

文件必须具有 `.toml` 扩展名。路径缺失是硬错误。配置的文件必须声明确切受支持的 schema 版本；已存在的空文件会被拒绝。

### TOML Schema

文件以 `[openshell]` 表为根。此命名空间为未来组件（CLI、沙箱、路由器）共享单个配置文件预留了空间，避免键冲突。

`[openshell.gateway]` 仅承载 gateway 全局设置。仅对特定计算驱动有意义的字段位于 `[openshell.drivers.<name>]` 下，由该驱动拥有。

```toml
[openshell]
version = 2                      # 必需的 schema 版本

# ──────────────────────────────────────────────────────────────────────────────
# Gateway 全局设置
# ──────────────────────────────────────────────────────────────────────────────
[openshell.gateway]
# 监听器
bind_address          = "127.0.0.1:17670"  # 默认：127.0.0.1:17670（回环）
health_bind_address   = "0.0.0.0:8081"     # 可选；省略则禁用
metrics_bind_address  = "0.0.0.0:9090"     # 可选；省略则禁用

# 日志
log_level             = "info"

# 计算驱动——只能有一个驱动处于活动状态。省略时，gateway
# 自动检测驱动（kubernetes → podman → docker）。VM 从不自动检测。
compute_driver        = "kubernetes"

# 注意：database_url 是密钥，必须通过 OPENSHELL_DB_URL
# （或 --db-url）提供——不允许出现在文件中。
ssh_session_ttl_secs    = 86400

# 服务路由——`server_sans` 中的通配符 DNS SAN 也会启用该域名下的
# 沙箱服务 URL。`enable_loopback_service_http` 切换回环服务 URL
# 的明文 HTTP 路由。
server_sans                  = ["openshell", "*.dev.openshell.localhost"]
enable_loopback_service_http = true

# ──────────────────────────────────────────────────────────────────────────────
# TLS / mTLS——包管理的本地 TLS 可提供监听器默认值。
# ──────────────────────────────────────────────────────────────────────────────
# 对应 --disable-tls / OPENSHELL_DISABLE_TLS。明文监听器需显式设为 true；
# 此时必须省略 guest TLS 字段。
disable_tls           = false

# Gateway 拥有的 CA，注入到选定的本地驱动中用于 supervisor TLS。
guest_tls_ca          = "/etc/openshell/certs/ca.pem"

[openshell.gateway.tls]
cert_path             = "/etc/openshell/certs/gateway.pem"
key_path              = "/etc/openshell/certs/gateway-key.pem"
client_ca_path        = "/etc/openshell/certs/client-ca.pem"

# ──────────────────────────────────────────────────────────────────────────────
# OIDC——省略时，JWT bearer 认证被禁用
# ──────────────────────────────────────────────────────────────────────────────
[openshell.gateway.oidc]
issuer        = "https://idp.example.com/realms/openshell"
audience      = "openshell-cli"
jwks_ttl_secs = 3600
roles_claim   = "realm_access.roles"   # Keycloak 默认；Entra 用 "roles"，Okta 用 "groups"
admin_role    = "openshell-admin"
user_role     = "openshell-user"
scopes_claim  = ""                     # 空则禁用 scope 强制

# ──────────────────────────────────────────────────────────────────────────────
# 计算驱动——每个表由其驱动 crate 拥有和解析。
# 仅激活选定或自动检测到的驱动的表。
# ──────────────────────────────────────────────────────────────────────────────

[openshell.drivers.kubernetes]
namespace                    = "openshell"
default_image                = "nvcr.io/nvidia/base/ubuntu:24.04"
image_pull_policy            = "if_not_present"
supervisor_image             = "ghcr.io/nvidia/openshell/supervisor:latest"
supervisor_image_pull_policy = "if_not_present"
grpc_endpoint                = "https://host.openshell.internal:8080"
client_tls_secret_name       = "openshell-sandbox-tls"
host_gateway_ip              = "10.0.0.1"
ssh_socket_path              = "/run/openshell/ssh.sock"

[openshell.drivers.docker]
default_image     = "nvcr.io/nvidia/base/ubuntu:24.04"
image_pull_policy = "if_not_present"
sandbox_label     = "docker-dev"
grpc_endpoint     = "https://host.openshell.internal:8080"
network_name      = "openshell"
supervisor_bin    = "/usr/local/libexec/openshell/openshell-sandbox"  # 可选覆盖
supervisor_image  = "ghcr.io/nvidia/openshell/supervisor:latest"      # 用于提取 bin

[openshell.drivers.podman]
socket_path       = "/run/podman/podman.sock"
default_image     = "nvcr.io/nvidia/base/ubuntu:24.04"
image_pull_policy = "if_not_present" # always | if_not_present | never | newer
supervisor_image  = "ghcr.io/nvidia/openshell/supervisor:latest"
network_name      = "openshell"
stop_timeout_secs = 10

[openshell.drivers.vm]
state_dir       = "/var/lib/openshell/vm"
driver_dir      = "/usr/local/libexec/openshell"
grpc_endpoint   = "https://host.containers.internal:8080"
vcpus           = 2
mem_mib         = 2048
krun_log_level  = 1
```

### 驱动配置

每个 `[openshell.drivers.<name>]` 表从解析后的文件中提取，作为原始 TOML 值传递给驱动的初始化函数。驱动负责：

1. **解析**——将表反序列化为自己的类型化配置结构体（如 `KubernetesComputeConfig`、`DockerComputeConfig`、`PodmanComputeConfig`、`VmComputeConfig`）。
2. **校验**——应用该驱动特有的跨字段检查。gateway 拥有的 CA 路径在此步骤之前校验并仅注入选定的本地驱动；supervisor 身份使用沙箱 bearer token。
3. **消费**——使用结果结构体初始化内部状态。

驱动作者定义并拥有其配置 schema。添加新驱动不需要修改 gateway 的核心 `Config` 结构体或本 RFC。

非选定或非自动检测驱动的 `[openshell.drivers.<name>]` 表会进行语法解析但不激活。

### 合并语义

字段级合并规则：

1. **`[openshell.gateway]`** 填充 `openshell_core::Config`（包括嵌套的 `[openshell.gateway.tls]` 和 `[openshell.gateway.oidc]` 表，分别映射到 `TlsConfig` 和 `OidcConfig`）。
2. **`[openshell.drivers.<name>]`** 传播到驱动 crate，由其反序列化为自己的结构体。驱动 schema 独立于 gateway 的核心 `Config` 演进。
3. **CLI / env** 逐字段覆盖步骤 1–2 设置的任何值。覆盖检查使用 clap 的 `ValueSource`——仅当对应标志未通过命令行或环境提供时，才从文件应用值。

`bind_address`、`health_bind_address` 和 `metrics_bind_address` 存储为 `SocketAddr`（IP + 端口）。CLI 将它们暴露为单个 `--bind-address` IP 加上 `--port`、`--health-port` 和 `--metrics-port`；CLI 覆盖应用于解析后的 `SocketAddr` 的对应部分。

`health_bind_address` 和 `metrics_bind_address` 可省略以禁用这些监听器（与当前 `--health-port 0` / `--metrics-port 0` 的行为一致）。

### 密钥

有一个字段被有意排除在 TOML schema 之外，必须通过环境变量或 CLI 标志提供：

| 字段 | 来源 |
|---|---|
| `database_url` | `OPENSHELL_DB_URL` / `--db-url` |

数据库 URL 通常嵌入凭据，明文文件泄露的后果比环境变量泄露严重得多。强制 URL 离开文件消除了最容易的意外提交路径。

如果该字段出现在 `[openshell.gateway]` 下，解析器会失败并给出明确错误，指引操作者使用 env/CLI 形式。

OIDC 设置（包括 `oidc.audience`、`oidc.admin_role` 等）**允许**出现在文件中。它们本身都不是凭据。但操作者对于任何原本会存入 Kubernetes `Secret` 的字段，仍应优先使用环境变量注入——TLS 材料路径、受限环境中的 OIDC issuer URL 等。通用指导：如果它会放在 `Secret` 资源中，就从环境变量获取；如果它会放在 `ConfigMap` 中，文件即可。

### 校验

反序列化在每个表级别使用 `#[serde(deny_unknown_fields)]`。无法识别的键是硬解析错误。这能尽早发现拼写错误，而不是静默忽略配置错误的字段。

合并文件 + env + CLI 后应用以下跨字段校验：

- 设置时，`bind_address`、`health_bind_address` 和 `metrics_bind_address` 必须使用不同端口。
- Gateway 监听器 TLS 需要 `cert_path` 和 `key_path`；`client_ca_path` 仅在监听器客户端证书验证时需要。启用 TLS 的 Docker、Podman 和 VM 驱动还需要完整的 gateway 拥有的 guest CA、证书和密钥包。Kubernetes 通过 Secret 投射 guest TLS。
- 合并 env + CLI 后 `database_url` 必须非空——每个受支持的驱动都需要它。该字段不接受来自文件（见上文密钥）。
- `compute_driver` 恰好选择一个驱动。省略时，gateway 回退到自动检测。自定义驱动需要带有 `socket_path` 的命名表，除非启动时提供显式 socket 覆盖。旧版 `compute_drivers` 列表被拒绝。

### Schema 兼容性

Schema 版本 2 要求 `version = 2`，选择驱动时使用单数 `compute_driver`，驱动拥有的字段位于 `[openshell.drivers.<name>]` 下。旧版 schema 版本和 `compute_drivers` 列表被拒绝。`OPENSHELL_DB_URL` 仍是必需的进程输入，不接受来自文件。

### 示例：最小 Kubernetes 部署

```toml
[openshell]
version = 2

[openshell.gateway]
bind_address  = "0.0.0.0:8080"
compute_driver = "kubernetes"
# database_url 来自 env（如 valueFrom.secretKeyRef）。
# gateway 在 Envoy / ingress 后面以明文运行。
disable_tls = true

[openshell.drivers.kubernetes]
namespace        = "agents"
default_image    = "nvcr.io/nvidia/base/ubuntu:24.04"
supervisor_image = "ghcr.io/nvidia/openshell/supervisor:0.9.0"
grpc_endpoint    = "https://openshell-gateway.agents.svc:8080"
```

### Helm 集成

Helm chart 将 schema-v2 gateway TOML 渲染到 `ConfigMap`，挂载到 `/etc/openshell/gateway.toml`，并用该文件启动 gateway。密钥进程输入如 `OPENSHELL_DB_URL` 仍为 `Secret` 支持的环境条目，并保持更高优先级。Kubernetes 通过其配置的 Secret 投射沙箱 guest TLS，而不是将主机 guest 证书路径放入 gateway TOML。

```yaml
# values.yaml 摘录
gateway:
  config:
    bind_address: "0.0.0.0:8080"
    health_bind_address: "0.0.0.0:8081"
    metrics_bind_address: "0.0.0.0:9090"
    compute_driver: "kubernetes"
    drivers:
      kubernetes:
        namespace: agents
        default_image: nvcr.io/nvidia/base/ubuntu:24.04
        supervisor_image: ghcr.io/nvidia/openshell/supervisor:0.9.0
```

Chart 所有者可以逐节迁移：过渡期间 `OPENSHELL_*` 环境变量和 `ConfigMap` 共存，env 继续覆盖文件。

## 实现

已实现的 gateway 加载器使用 `serde` 解析 TOML，将文件值合并在环境和 CLI 来源之下，并拒绝未知字段。每个计算驱动仅反序列化其命名表。Helm 将 schema-v2 TOML 渲染到 ConfigMap，而密钥进程输入仍由环境支持。包模板、示例、测试和 gateway 架构文档使用相同的规范 schema。

## 风险

- **Serde `deny_unknown_fields` 很严格**——`openshell_core::Config` 或驱动配置结构体中的任何字段名变更都会成为使用该文件者的破坏性变更。将字段重命名视为版本化 schema 变更，并清晰呈现迁移错误。
- **文件中的密钥**——`database_url` 完全排除在 schema 之外（仅 env / CLI）。OIDC 设置仍允许在文件中，因为它们单独都不是凭据。操作者对于任何原本会放在 `Secret` 而非 `ConfigMap` 中的字段（TLS 材料路径、受限环境 OIDC issuer 等），仍应优先使用环境变量注入。文档必须显著说明这一点。
- **部分 TLS 配置**——监听器和 guest TLS 是独立的完整包契约。启动时拒绝部分包，并在构造驱动之前指出缺失的配置。
- **驱动 schema 漂移**——一旦每个驱动拥有自己的 TOML 表，驱动发布可以独立于 gateway 更改字段名。gateway 的 `version` 字段无法防止驱动侧破坏；需单独记录驱动配置稳定性。

## 替代方案

**仅扁平环境变量**——现状。避免新的文件格式和解析层，但未解决驱动命名空间问题，并使 Helm chart 冗长。已拒绝：长期 Kubernetes 方案需要基于文件的方法。

**用 YAML 代替 TOML**——YAML 已是 Kubernetes 生态中的主导格式，Helm values 也是 YAML。用 YAML 做 gateway 配置会与该生态对齐。缺点是 YAML 众所周知的陷阱（挪威问题、隐式类型、缩进敏感）。TOML 无歧义，并通过 `serde` 干净地映射到 Rust 结构体。对于主要由人编辑的配置文件，TOML 的清晰性胜出。Helm chart 仍可通过 `tpl` 从 YAML values 生成 TOML 文件。

**独立配置 crate**——将配置解析集中到专用的 `openshell-config` crate 而非 `openshell-server` 内部。如果其他二进制需要相同配置格式则值得；推迟到有具体需求时。

## 先例

- [Gitea](https://docs.gitea.com/administration/config-cheat-sheet) 和 [InfluxDB](https://docs.influxdata.com/influxdb/v2/reference/config-options/) 都