# OpenShell Gateway 配置（RPM）

在 Fedora 和 RHEL 系统上通过 RPM 包安装 OpenShell gateway 时的配置参考。

首次安装请参阅 QUICKSTART.md。故障排查请参阅 TROUBLESHOOTING.md。

## 默认配置

RPM 在 `/usr/share/openshell-gateway/gateway.toml.default` 提供默认 TOML 配置模板。`openshell-gateway.service` 首次启动时，若 `~/.config/openshell/gateway.toml` 不存在，systemd 单元会将此模板复制到该路径。

默认值针对无根 Podman 使用场景调优：

```toml
[openshell]
version = 2

[openshell.gateway]
compute_driver = "podman"
```

RPM 不覆盖 `bind_address`。主监听器使用内置默认值 `127.0.0.1:17670`。使用主机网络的 Podman supervisor 连接同一回环监听器，因此 gateway 不暴露其他主机接口。

`compute_driver = "podman"` 将计算驱动固定为 Podman。若不设置，gateway 按 Kubernetes、Podman、Docker 的顺序自动检测。固定驱动可避免主机同时安装 Docker 时选错驱动。

### 自定义配置

直接编辑 `~/.config/openshell/gateway.toml`。包自带的模板 `/usr/share/openshell-gateway/gateway.toml.default` 在运行时不会被读取，且可能在 RPM 升级时变更。用户的活动副本会被保留。schema-v2 升级期间，服务只替换可识别的、由包生成的 v1 副本，绝不重写已编辑的配置。生成证书或启动 gateway 之前，服务会对生效配置运行 `openshell-gateway config preflight`，校验失败则停止。

如需在不编辑 TOML 文件的情况下添加跨升级持久生效的环境变量覆盖，将其写入 `~/.config/openshell/gateway.env`：

```shell
# 示例：显式在一个主机接口上暴露主监听器
OPENSHELL_BIND_ADDRESS=192.168.1.10
```

如需在内置 Docker、Podman 或 Kubernetes 驱动中选定确切的受信任运行时构建产物，设置完整的带标签或摘要固定的引用：

```shell
OPENSHELL_SANDBOX_RUNTIME_IMAGE=registry.example.com/openshell/sandbox@sha256:<digest>
OPENSHELL_SUPERVISOR_IMAGE=registry.example.com/openshell/supervisor@sha256:<digest>
```

`OPENSHELL_SANDBOX_RUNTIME_IMAGE` 和 `OPENSHELL_SUPERVISOR_IMAGE` 优先于所选驱动 TOML 表中的显式镜像字段；驱动 TOML 优先于编译时的发布默认值。Preflight 和启动使用相同的解析结果。这些值是受信任的操作员输入，不是沙箱请求字段。请将镜像仓库凭据保存在 Podman 的凭据存储中，不要嵌入镜像引用。

如需完全覆盖 TOML 配置文件路径：

```shell
# 在 ~/.config/openshell/gateway.env 中
OPENSHELL_GATEWAY_CONFIG=/path/to/custom/gateway.toml
```

如需一次性服务覆盖且跨包升级持久生效：

```shell
systemctl --user edit openshell-gateway
```

## TLS（mTLS）

RPM 默认启用 mTLS 用户认证。CLI 客户端出示有效的客户端证书；supervisor 使用 gateway CA 和沙箱作用域的 bearer token。主监听器使用 `127.0.0.1:17670`；Podman supervisor 会话使用同一监听器。

### 自动生成的证书

首次启动时，systemd 用户服务运行 `openshell-gateway generate-certs --output-dir ~/.local/state/openshell/tls --server-san host.openshell.internal`，使用 `rcgen` 生成证书（与 CLI 生成本地 mTLS 包所用的例程相同）。该单元将 `OPENSHELL_LOCAL_TLS_DIR` 设为该路径，并在证书生成和 gateway 启动时使用同一值。如需使用自定义包位置，在启动服务前于 `~/.config/openshell/gateway.env` 中设置 `OPENSHELL_LOCAL_TLS_DIR`。

| 文件 | 用途 | 位置 |
|------|---------|----------|
| CA 证书 | 信任根 | `~/.local/state/openshell/tls/ca.crt` |
| CA 私钥 | 签发服务器和客户端证书 | `~/.local/state/openshell/tls/ca.key` |
| 服务器证书 | Gateway TLS 身份 | `~/.local/state/openshell/tls/server/tls.crt` |
| 服务器私钥 | Gateway TLS 密钥 | `~/.local/state/openshell/tls/server/tls.key` |
| 客户端证书 | CLI 和沙箱身份 | `~/.local/state/openshell/tls/client/tls.crt` |
| 客户端私钥 | CLI 和沙箱密钥 | `~/.local/state/openshell/tls/client/tls.key` |

客户端证书还会复制到 CLI 自动发现目录：

```
~/.config/openshell/gateways/openshell/mtls/
  ca.crt
  tls.crt
  tls.key
```

CLI 连接 `localhost` 或 `127.0.0.1` 上的 gateway 时会自动发现这些证书。

### 服务器证书 SAN

自动生成的服务器证书包含以下 Subject Alternative Name：

- `localhost`
- `openshell`
- `openshell.openshell.svc`
- `openshell.openshell.svc.cluster.local`
- `host.containers.internal`
- `host.docker.internal`
- `host.openshell.internal`
- `127.0.0.1`

如需从远程机器连接，需要外部管理的、带额外 SAN 的证书。参见 TROUBLESHOOTING.md 中的"Remote CLI access"。

### 使用外部管理的证书

如需使用来自外部 CA 或 cert-manager 的证书：

1. 将服务器证书、密钥和 CA 证书放到文件系统上。

1. 编辑 `~/.config/openshell/gateway.toml`：

   ```toml
   [openshell.gateway.tls]
   cert_path = "/path/to/server/tls.crt"
   key_path = "/path/to/server/tls.key"
   client_ca_path = "/path/to/ca.crt"
   ```

1. 将客户端证书放到 CLI 期望的位置：

   ```
   ~/.config/openshell/gateways/openshell/mtls/
     ca.crt
     tls.crt
     tls.key
   ```

### 轮换证书

删除 TLS 状态目录并重启 gateway：

```shell
rm -rf ~/.local/state/openshell/tls
systemctl --user restart openshell-gateway
```

gateway 会在下次启动时重新生成 PKI。

### 禁用 TLS

> **警告：** 禁用 TLS 后，gateway API 没有任何认证。请将绑定地址保持在 `127.0.0.1`，或将 gateway 置于执行自身认证的 TLS 终止反向代理之后。

如需禁用 TLS（不推荐用于生产环境）：

1. 编辑 `~/.config/openshell/gateway.toml`：

   ```toml
   [openshell.gateway]
   disable_tls = true
   ```

1. 若 `~/.config/openshell/gateway.toml` 中设置了 `guest_tls_*` 条目，删除或注释掉它们。

1. 重启 gateway。

## 沙箱 TLS

启用 TLS 时，Podman 驱动会将 gateway CA 绑定挂载到每个 supervisor 容器中，用于认证 gateway。supervisor 使用沙箱作用域的 bearer token 认证其 RPC。用户客户端证书和私钥不会挂载到 supervisor 或工作负载容器中。

以下 TOML 字段控制主机侧 CA 路径：

```toml
[openshell.gateway]
guest_tls_ca = "/home/user/.local/state/openshell/tls/ca.crt"
```

在 supervisor 容器内，CA 挂载于 `/etc/openshell/tls/client/ca.crt`。

在启用 SELinux 的系统上，Podman 驱动会自动为这些绑定挂载应用 `:z` 重标签选项。无需手动配置 SELinux。

## 配置参考

> **从旧版本升级？** 参见 TROUBLESHOOTING.md 中的
> ["Migrating from gateway.env"](TROUBLESHOOTING.md#migrating-from-gatewayenv)
> 一节，了解环境变量到 TOML 的映射，以及默认端口、绑定地址和数据库路径变更的说明。

Gateway 和驱动设置具有本地运行时默认值。当 `~/.config/openshell/gateway.toml` 存在时，gateway 会读取该文件。在启动环境中设置 `OPENSHELL_GATEWAY_CONFIG` 可使用其他文件。

使用 `systemctl --user edit openshell-gateway` 设置跨包升级持久生效的服务环境覆盖。

### Gateway 设置

| TOML 选项 | 默认值 | 描述 |
|-------------|---------|-------------|
| `bind_address` | `127.0.0.1:17670`（gateway 默认值） | 主 gRPC/HTTP API 监听器的地址。 |
| `compute_driver` | `"podman"`（RPM 默认值） | 未设置时，gateway 按 Kubernetes、Podman、Docker 的顺序自动检测。RPM 默认固定为 Podman；旧的 `compute_drivers` 列表会被拒绝。 |
| `[openshell.drivers.podman].default_image` | `nvcr.io/nvidia/base/ubuntu:24.04` | 默认沙箱镜像。 |
| `[openshell.drivers.podman].sandbox_runtime_image` | `ghcr.io/nvidia/openshell/sandbox:latest` | 受信任的沙箱运行时镜像。`OPENSHELL_SANDBOX_RUNTIME_IMAGE` 覆盖此字段。 |
| `[openshell.drivers.podman].supervisor_image` | `ghcr.io/nvidia/openshell/supervisor:latest` | 受信任的 supervisor 镜像。`OPENSHELL_SUPERVISOR_IMAGE` 覆盖此字段。 |
| `[openshell.gateway].guest_tls_ca` | 自动生成路径 | 注入所选本地驱动的 gateway CA，用于 supervisor 到 gateway 的 TLS。沙箱身份使用 bearer token。 |
| `[openshell.gateway.tls]` 路径 | 自动生成路径 | 服务器 TLS 证书、密钥和客户端 CA。 |
| `disable_tls` | 未设置 | 设为 `true` 以禁用 TLS。 |

TOML 中不接受数据库 URL。`OPENSHELL_DB_URL` 未设置时，gateway 使用 `sqlite:$XDG_STATE_HOME/openshell/gateway/openshell.db`。SQLite 数据库以 WAL 模式运行，`synchronous=FULL`（仅 SSH 会话签发使用 `NORMAL`），因此 `openshell.db-wal` 和 `openshell.db-shm` 与其位于同一目录，必须一起保留；使用 `sqlite3 openshell.db ".backup <copy>"` 备份。

### 驱动 TOML 设置

需要自定义驱动设置时，创建 `~/.config/openshell/gateway.toml`：

```toml
[openshell]
version = 2

[openshell.gateway]
compute_driver = "podman"

[openshell.drivers.podman]
network_name = "openshell"
default_image = "nvcr.io/nvidia/base/ubuntu:24.04"
image_pull_policy = "if_not_present"
health_check_interval_secs = 10
stop_timeout_secs = 10
```

### 镜像管理

gateway 在首次创建沙箱时自动拉取容器镜像。默认拉取策略为 `if_not_present`，即镜像只拉取一次，之后由 Podman 缓存。

如需更新缓存的镜像：

```shell
podman pull ghcr.io/nvidia/openshell/supervisor:latest
podman pull nvcr.io/nvidia/base/ubuntu:24.04
```

或在 `[openshell.drivers.podman]` 中设置 `image_pull_policy = "always"`，每次创建沙箱时都拉取。

如需固定具体镜像版本而非 `:latest`，在 `[openshell.drivers.podman]` 中设置：

```toml
sandbox_runtime_image = "ghcr.io/nvidia/openshell/sandbox:v0.0.37"
supervisor_image = "ghcr.io/nvidia/openshell/supervisor:v0.0.37"
default_image = "nvcr.io/nvidia/base/ubuntu:24.04"
```

对于隔离网络环境：

1. 在联网机器上拉取并保存镜像：

   ```shell
   podman pull ghcr.io/nvidia/openshell/supervisor:latest
   podman pull nvcr.io/nvidia/base/ubuntu:24.04
   podman save -o supervisor.tar ghcr.io/nvidia/openshell/supervisor:latest
   podman save -o sandbox.tar nvcr.io/nvidia/base/ubuntu:24.04
   ```

1. 将 tar 包传输到隔离网络主机并加载：

   ```shell
   podman load -i supervisor.tar
   podman load -i sandbox.tar
   ```

1. 将拉取策略设为 `never`：

   ```toml
   [openshell.drivers.podman]
   image_pull_policy = "never"
   ```

## 文件位置

| 用途 | 路径 |
|---------|------|
| Gateway 二进制 | `/usr/bin/openshell-gateway` |
| CLI 二进制 | `/usr/bin/openshell` |
| Systemd 用户单元 | `/usr/lib/systemd/user/openshell-gateway.service` |
| 默认 TOML 配置模板（只读） | `/usr/share/openshell-gateway/gateway.toml.default` |
| 活动 gateway TOML 配置 | `~/.config/openshell/gateway.toml` |
| 可选环境变量覆盖 | `~/.config/openshell/gateway.env` |
| TLS 证书 | `~/.local/state/openshell/tls/` |
| CLI 客户端证书 | `~/.config/openshell/gateways/openshell/mtls/` |
| Gateway 数据库 | `~/.local/state/openshell/gateway/openshell.db` |