# OpenShell RPM 故障排查

RPM 部署的故障排查指南、CLI 兼容性说明、远程访问配置及升级流程。

## CLI 兼容性

RPM 将网关安装为 systemd 用户服务。标准 RPM 安装中，由于包依赖 Podman，网关会自动检测 Podman。已发布的在线文档和部分 CLI 命令假定使用 Docker/K3s 部署模型。本节说明哪些命令可用、哪些不可用，以及应改用哪些命令。

### 正常可用的命令

所有沙箱、提供方、策略和设置命令均通过 gRPC 与网关通信，无论部署模式如何，行为完全一致：

```
openshell status
openshell sandbox create|list|get|delete|connect|exec
openshell logs <sandbox>
openshell provider create|list|get|update|delete
openshell policy get|set|update|list|prove
openshell provider list
openshell sandbox provider list <sandbox>
openshell settings get|set
openshell forward start|stop|list
openshell term
openshell gateway add|select|info|list|remove
```

### 网关生命周期

RPM 部署中，网关服务的生命周期由 systemd 管理。直接使用 systemd 命令：

| 任务 | 命令 |
|------|---------|
| 启动网关 | `systemctl --user start openshell-gateway` |
| 停止网关 | `systemctl --user stop openshell-gateway` |
| 重启网关 | `systemctl --user restart openshell-gateway` |
| 查看状态 | `systemctl --user status openshell-gateway` |
| 查看日志 | `journalctl --user -u openshell-gateway` |
| 跟踪日志 | `journalctl --user -u openshell-gateway -f` |
| 移除 CLI 注册 | `openshell gateway remove [name]` |

### 从本地 Dockerfile 构建

使用 Podman 构建镜像，然后直接引用：

```shell
podman build -t localhost/my-sandbox:latest ./my-dir
openshell sandbox create --from localhost/my-sandbox:latest
```

## 远程 CLI 访问

自动生成的服务器证书仅包含 `localhost`、`127.0.0.1` 和 Podman 内部名称的 SAN。要从其他机器连接，请选择以下方式之一。

### 方式 1：SSH 隧道（最简单）

通过 SSH 转发网关端口，经 localhost 连接：

```shell
# 在远程 CLI 机器上：
ssh -L 17670:127.0.0.1:17670 user@gateway-host

# 在同一机器的另一个终端中：
# 先从网关主机复制客户端证书：
scp -r user@gateway-host:~/.config/openshell/gateways/openshell/mtls/ \
    ~/.config/openshell/gateways/openshell/mtls/

openshell gateway add --local https://127.0.0.1:17670
openshell status
```

### 方式 2：外部管理的证书

生成在 SAN 中包含服务器主机名或 IP 的证书。参见 CONFIGURATION.md 中的"使用外部管理的证书"。然后将 `~/.config/openshell/gateway.toml` 中的 `bind_address` 改为远程 CLI 可访问的接口，例如 `192.168.1.10:17670`，并重启网关。

放置服务器和客户端证书后，从远程 CLI 注册：

```shell
# 将客户端证书复制到远程 CLI 机器
mkdir -p ~/.config/openshell/gateways/openshell/mtls/
cp ca.crt tls.crt tls.key ~/.config/openshell/gateways/openshell/mtls/

openshell gateway add --local https://<gateway-hostname>:17670
```

### 防火墙

如需远程访问，在 firewalld 中开放网关端口：

```shell
sudo firewall-cmd --add-port=17670/tcp --permanent
sudo firewall-cmd --reload
```

仅限 localhost 访问（默认场景）无需修改防火墙。firewalld 不过滤回环流量。

即使端口对网络开放，mTLS 也能阻止未认证访问。

## 常见问题

### "No active gateway"

CLI 找不到已注册的网关。网关正在运行但尚未向 CLI 注册时会出现此问题。

```shell
openshell gateway add --local https://127.0.0.1:17670
```

### 网关启动失败

查看 journal 获取错误详情：

```shell
journalctl --user -u openshell-gateway --no-pager -n 50
```

常见原因：

**检测到 cgroups v1。** Podman 驱动需要 cgroups v2。检查版本：

```shell
stat -fc %T /sys/fs/cgroup
```

预期输出：`cgroup2fs`。若显示 `tmpfs`，启用 cgroups v2：

```shell
sudo grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=1"
sudo reboot
```

**Podman socket 不可用。** 确保已启用 socket 激活：

```shell
systemctl --user enable --now podman.socket
systemctl --user status podman.socket
```

**TLS 证书错误。** 若证书损坏，重新生成：

```shell
rm -rf ~/.local/state/openshell/tls
systemctl --user restart openshell-gateway
```

### 沙箱创建失败

**缺少 subuid/subgid。** 无根 Podman 需要从属 UID/GID 范围。若 journal 显示关于 `/etc/subuid` 的警告或容器创建失败：

```shell
grep $USER /etc/subuid /etc/subgid
# 若为空：
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $USER
```

**镜像拉取失败。** 验证 ghcr.io 可达：

```shell
podman pull nvcr.io/nvidia/base/ubuntu:24.04
```

### 镜像未更新

默认镜像拉取策略为 `if_not_present`——镜像仅拉取一次并缓存。更新方法：

```shell
podman pull nvcr.io/nvidia/base/ubuntu:24.04
podman pull ghcr.io/nvidia/openshell/supervisor:latest
```

或在 `~/.config/openshell/gateway.toml` 中设置 `image_pull_policy = "always"` 并重启网关。

### 注销后网关停止

启用 lingering 使服务在注销后继续运行：

```shell
sudo loginctl enable-linger $USER
```

## SELinux

标准 Fedora 或 RHEL 无需 SELinux 配置。检测到 SELinux 时，Podman 驱动会自动对 TLS 绑定挂载应用 `:z` 重标记选项，使沙箱容器能通过 MAC 策略读取证书。

## 升级

升级 RPM 包后：

```shell
sudo dnf update openshell openshell-gateway
systemctl --user restart podman.socket
systemctl --user restart openshell-gateway
```

SQLite 数据库 Schema 在启动时自动迁移。重启期间运行中的沙箱会停止。

建议在包升级后重启 `podman.socket`：若升级期间磁盘上的 unit 文件发生变化，运行中的 socket 可能失效，导致网关在 `/run/user/<uid>/podman/podman.sock` 上出现连接错误。网关启动时会短暂重试，但失效的 socket 不会自行恢复。

包升级会保留已编辑的 `~/.config/openshell/gateway.toml` 文件。在 schema-v2 升级中，用户服务仅替换此前由 RPM 播种的 v1 文件的精确副本。若你编辑过该文件，请在重启服务前手动迁移；直接使用 `dnf` 或 `rpm` 升级不会触发 `install.sh` 中的破坏性升级保护。逐字段迁移步骤参见[网关配置文件](https://docs.nvidia.com/openshell/latest/how-it-works/gateways/configuration#migrate-to-schema-version-2)。新增的网关进程选项见 CONFIGURATION.md 和 `openshell-gateway --help`。

升级后获取新容器镜像：

```shell
podman pull ghcr.io/nvidia/openshell/supervisor:latest
podman pull nvcr.io/nvidia/base/ubuntu:24.04
```

### 将启用 TLS 的本地驱动迁移到 Schema 版本 2

Docker、Podman 和 VM supervisor 使用网关的 CA 认证网关，并使用沙箱 bearer token 认证 RPC。包管理的安装使用 `~/.local/state/openshell/tls` 下生成的 CA，因此 RPM 默认无需额外 TOML。若你使用自定义 `--tls-cert` 和 `--tls-key` 覆盖监听器且不使用该托管 CA，请在 `[openshell.gateway]` 下配置 `guest_tls_ca`。移除已弃用的 `guest_tls_cert` 和 `guest_tls_key` 字段。缺少所需 CA 时网关启动失败。禁用 TLS 时省略 `guest_tls_ca`。

### 从 gateway.env 迁移

先前版本在首次启动时生成 `~/.config/openshell/gateway.env`，并在启动时用于配置网关。网关现在从内置运行时默认值启动，并在 `~/.config/openshell/gateway.toml` 存在时读取该文件。

若你有 `gateway.env` 文件，它仍然有效：systemd unit 每次启动时通过 `EnvironmentFile` 读取它。你可以保留或删除它。新安装不再生成该文件。

要将设置迁移到 TOML，创建 `~/.config/openshell/gateway.toml` 并映射相关变量：

| 环境变量 | TOML 等效项 |
|---|---|
| `OPENSHELL_BIND_ADDRESS=A` + `OPENSHELL_SERVER_PORT=P` | `[openshell.gateway]` 下的 `bind_address = "A:P"` |
| `OPENSHELL_COMPUTE_DRIVER=podman` | `[openshell.gateway]` 下的 `compute_driver = "podman"` |
| `OPENSHELL_DISABLE_TLS=true` | `[openshell.gateway]` 下的 `disable_tls = true` |
| `OPENSHELL_TLS_CERT=PATH` | `[openshell.gateway.tls]` 下的 `cert_path = "PATH"` |
| `OPENSHELL_TLS_KEY=PATH` | `[openshell.gateway.tls]` 下的 `key_path = "PATH"` |
| `OPENSHELL_TLS_CLIENT_CA=PATH` | `[openshell.gateway.tls]` 下的 `client_ca_path = "PATH"` |
| `OPENSHELL_DB_URL=URL` | 仅环境变量——TOML 不接受；保留在环境变量或 drop-in 覆盖中 |
| `OPENSHELL_LOG_LEVEL=debug` | 仅环境变量——在 drop-in 中保留为 `Environment=OPENSHELL_LOG_LEVEL=debug` |

此版本的其他破坏性变更：

- **默认端口从 8080 改为 17670。** 若你在 `https://127.0.0.1:8080` 注册了网关，请重新注册：

  ```shell
  openshell gateway add --local https://127.0.0.1:17670
  ```

- **默认绑定地址从 `0.0.0.0` 改为 `127.0.0.1`。** 若你依赖网络可访问性而未显式指定绑定地址，请在 `~/.config/openshell/gateway.toml` 中绑定具体的可达接口：

  ```toml
  [openshell.gateway]
  bind_address = "192.168.1.10:17670"
  ```

  同时更新防火墙规则（如适用）：

  ```shell
  sudo firewall-cmd --remove-port=8080/tcp --permanent
  sudo firewall-cmd --add-port=17670/tcp --permanent
  sudo firewall-cmd --reload
  ```

- **数据库路径已变更**，从 `~/.local/state/openshell/gateway.db` 改为 `~/.local/state/openshell/gateway/openshell.db`。现有网关状态（已注册的沙箱等）不会自动迁移。要在升级中保留状态，请在重启前移动文件：

  ```shell
  mkdir -p ~/.local/state/openshell/gateway
  mv ~/.local/state/openshell/gateway.db \
     ~/.local/state/openshell/gateway/openshell.db
  ```