# OpenShell RPM 快速入门

五分钟内从 `dnf install` 到运行沙箱。

## 前置条件

### Podman（无根模式）

网关使用无根 Podman 运行沙箱容器。确认已安装 Podman 且 cgroup 版本为 v2：

```shell
podman --version
podman info --format '{{.Host.CgroupsVersion}}'
```

cgroup 版本必须为 `v2`。若显示 `v1`，启用统一 cgroup 层级并重启：

```shell
sudo grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=1"
sudo reboot
```

### 从属 UID/GID 范围

无根容器需要从属 UID/GID 映射：

```shell
grep $USER /etc/subuid /etc/subgid
```

若为空，添加条目：

```shell
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $USER
```

### Podman socket

网关通过 Podman 的 API socket 与其通信。启用 socket 激活：

```shell
systemctl --user enable --now podman.socket
```

### 网络访问

首次创建沙箱时，网关从 ghcr.io 拉取容器镜像。确保主机可通过 HTTPS（端口 443）访问 ghcr.io。

对于隔离环境，先用 `podman pull` 预加载镜像，并在 `~/.config/openshell/gateway.toml` 中设置 `image_pull_policy = "never"`。详见 CONFIGURATION.md。

## 启动网关

```shell
systemctl --user enable --now openshell-gateway
```

首次启动时，网关自动生成：

- 用于 mTLS 的自签名 PKI 包（CA、服务器证书、客户端证书）

> **注意：** 主网关监听器使用回环默认地址 `127.0.0.1:17670`。使用主机网络的 Podman 监管进程也使用此监听器。首次启动时自动启用 mTLS 用户认证。监管进程连接使用网关 CA 和沙箱作用域 Bearer 令牌。详见 CONFIGURATION.md。

验证服务正在运行：

```shell
systemctl --user status openshell-gateway
```

## 向 CLI 注册网关

CLI 需要知道网关位置。注册它：

```shell
openshell gateway add --local https://127.0.0.1:17670
```

此命令会发现 `~/.config/openshell/gateways/openshell/mtls/` 下预置的 mTLS 证书，并将该网关设为活动网关。

验证连接：

```shell
openshell status
```

## 跨重启持久化

默认情况下，用户服务在注销时停止。要让网关在注销后及跨重启继续运行：

```shell
sudo loginctl enable-linger $USER
```

否则，登录会话结束时网关及所有运行中的沙箱都会被终止。任何无头或生产用途都必须启用此项。

## 创建第一个沙箱

在环境中设置 API 密钥，然后创建沙箱：

```shell
export ANTHROPIC_API_KEY=sk-...
openshell sandbox create -- claude
```

CLI 会检测代理，提示从本地环境创建凭据提供程序，从 ghcr.io 拉取沙箱镜像，并将你连接到运行中的沙箱。

其他代理：

```shell
openshell sandbox create -- opencode
openshell sandbox create -- codex
```

## 手动设置提供程序

如果希望在创建沙箱前配置提供程序：

```shell
# Create a provider from a local environment variable
openshell provider create --name anthropic --type anthropic --from-existing

# Or supply the credential directly
openshell provider create --name openai --type openai \
  --credential OPENAI_API_KEY=sk-...

# List configured providers
openshell provider list
```

## 配置提供程序支持的推理（可选）

授予沙箱访问模型提供程序的权限：

```shell
openshell sandbox provider attach <sandbox> openai

# Configure the workload to call the provider's native endpoint and select
# the model in the client.
```

## 后续步骤

- 参见 CONFIGURATION.md 了解 TLS 设置、环境变量和文件位置。
- 参见 TROUBLESHOOTING.md 了解 CLI 兼容性说明、远程访问和常见问题。
- 运行 `man openshell` 查看 CLI 参考。
- 运行 `man openshell-gateway` 查看网关守护进程参考。