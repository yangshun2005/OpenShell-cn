---
title: OPENSHELL-GATEWAY
section: 8
header: OpenShell Manual
footer: openshell-gateway
date: 2025
---

# NAME

openshell-gateway - OpenShell 网关服务器守护进程

# SYNOPSIS

**openshell-gateway** \[*OPTIONS*\]

**openshell-gateway** **config preflight** [**--path** *PATH* | **--** *GATEWAY_ARGS*...]

# DESCRIPTION

**openshell-gateway** 是 OpenShell 的控制平面服务器。它管理沙箱生命周期、存储提供商凭据、向沙箱下发网络和文件系统策略、管理提供商访问请求，并提供 CLI 到沙箱连接的 SSH 隧道端点。

通过 Linux 包安装时，网关作为 systemd 用户服务运行。打包的服务从内置默认值启动，仅当默认网关 TOML 路径文件存在时才读取该文件。

网关在单一端口上多路复用 gRPC 和 HTTP，默认由双向 TLS（mTLS）保护，除非 TOML 配置禁用 TLS。

# OPTIONS

**--bind-address** *IP*
:   所有监听器绑定的 IP 地址。默认：**127.0.0.1**。
    环境变量：**OPENSHELL_BIND_ADDRESS**。

**--port** *PORT*
:   gRPC/HTTP API 端口。默认：**17670**。
    环境变量：**OPENSHELL_SERVER_PORT**。

**--health-port** *PORT*
:   未认证健康检查端点（/healthz、/readyz）端口。设为 0 禁用。默认：**0**。
    环境变量：**OPENSHELL_HEALTH_PORT**。

**--metrics-port** *PORT*
:   Prometheus 指标（/metrics）端口。设为 0 禁用。默认：**0**。
    环境变量：**OPENSHELL_METRICS_PORT**。

**--log-level** *LEVEL*
:   日志级别：trace、debug、info、warn、error。默认：**info**。
    环境变量：**OPENSHELL_LOG_LEVEL**。

**--db-url** *URL*
:   用于状态持久化的 SQLite 数据库 URL。未设置时，网关将 SQLite 状态存储在 *~/.local/state/openshell/gateway/* 下。
    环境变量：**OPENSHELL_DB_URL**。

**--db-max-connections** *N*
:   数据库连接池的连接上限，所有依赖数据库的请求共享此上限。必须至少为 1；使用 Postgres 时至少为 2，因为 SSH 身份锁在查询连接池期间会占用一个连接。未设置时，网关使用后端默认值：Postgres 为 **10**，磁盘 SQLite 为 **5**。内存 SQLite 数据库始终只有一个连接。
    TOML 键：`database_max_connections`。
    环境变量：**OPENSHELL_DB_MAX_CONNECTIONS**。

**--db-lock-max-connections** *N*
:   Postgres 变更锁连接池的连接上限，每个副本在其数据库连接池之上额外打开此连接池。每个变更守卫占用一个锁连接。必须至少为 1。未设置时，网关使用 **4**。SQLite 没有锁连接池，忽略此项。
    TOML 键：`database_lock_max_connections`。
    环境变量：**OPENSHELL_DB_LOCK_MAX_CONNECTIONS**。

**--compute-driver** *DRIVER*
:   计算驱动。只能选择一个驱动。选项：**podman**、**docker**、**kubernetes**、**vm**。未设置时，网关按 Kubernetes、Podman、Docker 的顺序自动检测。VM 需显式启用。
    环境变量：**OPENSHELL_COMPUTE_DRIVER**。

**--tls-cert** *PATH*
:   服务器 TLS 证书文件路径。存在本地生成的 TLS 包时默认使用该包。除非设置了 **--disable-tls**，否则必填。
    环境变量：**OPENSHELL_TLS_CERT**。

**--tls-key** *PATH*
:   服务器 TLS 私钥文件路径。存在本地生成的 TLS 包时默认使用该包。除非设置了 **--disable-tls**，否则必填。
    环境变量：**OPENSHELL_TLS_KEY**。

**--tls-client-ca** *PATH*
:   用于客户端证书验证（mTLS）的 CA 证书路径。单独设置（不带 **--oidc-issuer**）时，客户端证书为必需，TLS 握手会拒绝未认证连接。与 **--oidc-issuer** 同时设置时，客户端证书被接受但非必需。启用 mTLS 认证时，客户端证书可认证本地单用户 CLI 调用方；沙箱 supervisor 仍使用网关签发的 bearer token 认证。
    环境变量：**OPENSHELL_TLS_CLIENT_CA**。

**--enable-mtls-auth** *BOOL*
:   为本地单用户 Docker、Podman 和 VM 网关启用 mTLS 客户端证书认证。对于启用了客户端证书验证且无 OIDC issuer 的本地网关，默认开启。Kubernetes 计算驱动不支持。
    环境变量：**OPENSHELL_ENABLE_MTLS_AUTH**。

**--enable-operator-auth** *BOOL*
:   启用直接 mTLS 操作员认证，需持有包含 **OU=operator** 的可信客户端证书。授予跨所有工作区的用户/平台管理员能力，以及访问运行时凭据检索的权限。默认关闭。需要 TLS 和客户端 CA，独立于普通 mTLS 用户认证和 OIDC。不要将操作员证书与 bearer 头组合使用。
    环境变量：**OPENSHELL_ENABLE_OPERATOR_AUTH**。

**--disable-tls**
:   完全禁用 TLS，监听明文 HTTP。当绑定地址为 **0.0.0.0**（RPM 默认值）时，禁用 TLS 会使 API 在无认证的情况下暴露给整个网络。仅在网关位于 TLS 终止反向代理之后时使用，或将 **--bind-address** 限制为 **127.0.0.1**。
    环境变量：**OPENSHELL_DISABLE_TLS**。

**--server-san** *SAN*
:   网关服务器证书上配置的主题备用名称。可重复指定或通过 **OPENSHELL_SERVER_SAN** 传入逗号分隔值。通配符 DNS SAN 还会启用该域名下的沙箱服务 URL。
    环境变量：**OPENSHELL_SERVER_SAN**。

计算驱动设置（如沙箱镜像、回调端点、镜像拉取策略、网络名称、VM 状态目录和客户机 TLS 材料）通过 **--config** 传入的 TOML 文件配置。

# CONFIGURATION PREFLIGHT

在启动守护进程前验证网关配置：

    openshell-gateway config preflight [--path PATH | -- GATEWAY_ARGS...]

未指定路径时，preflight 验证非空的 OPENSHELL_GATEWAY_CONFIG。若该变量未设置，则可选地验证自动发现的 XDG 配置。两者均不存在时，仍会验证有效的守护进程参数。显式指定的路径缺失、旧版 schema-v1 文件、无效 TOML、符号链接或非普通文件会以非零状态失败。Preflight 合并文件和环境值，并执行只读启动检查：选择器和套接字规范化、已注册计算驱动配置、速率限制、TLS 和 mTLS、拦截器以及中间件。当所选文件省略选择器时，它会验证可自动检测驱动的已配置表，但不运行基于套接字或进程的检测探测。它不构造计算驱动，也不连接传输层。Preflight 从不修改文件，并报告失败的输入已被保留。

**--** 之后的参数替换 **--path** 模式，并作为精确的网关守护进程调用进行解析。包包装器使用此形式，以便命令行覆盖在到达启动之前得到验证。

显式选择的本地 **vm** 驱动还会检查 **mke2fs** 或 **mkfs.ext4**、**debugfs** 和 **e2fsck**。使用操作系统的包管理器安装 e2fsprogs 1.43 或更新版本，然后以网关服务的账户、工作目录、配置和环境运行 preflight。该命令报告所选可执行文件路径和版本。受限的服务 **PATH** 可能选择与交互式 shell 不同的工具；请在该服务的环境中包含安装的 bin 和 sbin 目录。

每个可执行文件仅接收 **-V**，超时五秒，每个流输出限制 8 KiB。缺失、不可执行、不支持或失败的工具返回非零状态，并附带安装或修复指导。Preflight 不创建镜像或运行时状态，也不启动 VM 驱动。其他驱动不需要这些工具。远程驱动端点会报告未执行主机工具检查；请在该主机的驱动服务环境中检查本地 VM 配置。

Debian 和 Ubuntu 的 systemd 用户单元在证书生成之前运行 preflight，同时保留其 EnvironmentFile 和裸 ExecStart 行为。snap 包装器通过 preflight 重放其有效的守护进程参数。它首先使用非空的 OPENSHELL_GATEWAY_CONFIG。否则，只要规范的服务特定 gateway.toml 路径存在或是符号链接，就传入该路径。旧版系统服务使用 SNAP_COMMON/gateway.toml；用户服务使用 SNAP_USER_COMMON/.config/openshell/gateway.toml。损坏的符号链接会在网关启动前使 preflight 失败。修正或手动迁移操作员拥有的 v1 文件，然后在重启服务前再次运行 preflight。

# SYSTEMD INTEGRATION

包在 */usr/lib/systemd/user/openshell-gateway.service* 安装 systemd 用户单元。使用标准 systemd 命令管理网关：

    systemctl --user enable --now openshell-gateway
    systemctl --user status openshell-gateway
    systemctl --user restart openshell-gateway
    systemctl --user stop openshell-gateway

查看日志：

    journalctl --user -u openshell-gateway
    journalctl --user -u openshell-gateway -f

该单元作为 **ExecStartPre** 步骤依次运行 **openshell-gateway config preflight** 和 **openshell-gateway generate-certs**。证书生成会创建用于 mTLS 和沙箱 JWT 签名材料的自签名 PKI 包，并在需要时为仅含 TLS 的旧安装添加缺失的 JWT 文件。打包的单元将 **OPENSHELL_LOCAL_TLS_DIR** 设为 *~/.local/state/openshell/tls*，证书生成和网关启动使用同一值。仅在需要自定义包位置时，在 *~/.config/openshell/gateway.env* 中覆盖 **OPENSHELL_LOCAL_TLS_DIR**。

随后网关从内置默认值启动，并在 *~/.config/openshell/gateway.toml* 存在时读取该文件。

用户服务还会为配置 preflight 和启动读取 *~/.config/openshell/gateway.env*。**OPENSHELL_SANDBOX_RUNTIME_IMAGE** 和 **OPENSHELL_SUPERVISOR_IMAGE** 接受完整的带标签或摘要固定的 OCI 引用，适用于内置 Docker、Podman 和 Kubernetes 驱动。这些变量优先于所选驱动 TOML 表中的显式镜像字段，后者优先于编译时的发布默认值。这些是受信任的网关输入，不能由沙箱请求提供。不要在镜像引用中嵌入镜像仓库凭据。

要使服务在注销后保持运行：

    sudo loginctl enable-linger $USER

# CONFIGURATION

systemd 用户单元通过以下命令启动网关：

    openshell-gateway

网关监听器、TLS、数据库和计算驱动设置均有本地默认值。需要覆盖时创建 *~/.config/openshell/gateway.toml*。网关拒绝 TOML 中的 `database_url`；需要不同数据库时请设置 **OPENSHELL_DB_URL**。

要在不创建 TOML 的情况下覆盖单项设置：

    systemctl --user edit openshell-gateway

这会创建一个在包升级后仍保留的 drop-in 覆盖。

# FILES

*/usr/bin/openshell-gateway*
:   网关二进制文件。

*/usr/lib/systemd/user/openshell-gateway.service*
:   Systemd 用户单元文件。

*~/.config/openshell/gateway.toml*
:   可选的网关 TOML 配置。

*~/.local/state/openshell/tls/*
:   自动生成的 TLS 证书和沙箱 JWT 签名密钥。

*~/.local/state/openshell/gateway/openshell.db*
:   网关状态的 SQLite 数据库。

*~/.config/openshell/gateways/openshell/mtls/*
:   供 CLI 自动发现的客户端 mTLS 证书。

# EXAMPLES

将网关作为 systemd 用户服务启动：

    systemctl --user enable --now openshell-gateway

从 CLI 检查网关健康状态：

    openshell gateway add --local https://127.0.0.1:17670
    openshell status

在 TOML 中覆盖 API 端口：

    $EDITOR ~/.config/openshell/gateway.toml
    systemctl --user restart openshell-gateway

# SEE ALSO

**openshell**(1)、**systemctl**(1)、**journalctl**(1)、**loginctl**(1)、
**podman**(1)

完整文档：*https://docs.nvidia.com/openshell/*