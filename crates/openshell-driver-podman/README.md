# openshell-driver-podman

Podman 计算驱动运行在网关内，通过 Unix socket 使用原生 libpod REST API。每个沙箱有两个独立容器：

- `openshell-sandbox` 在工作负载容器中承载 agent 进程。
- `openshell-supervisor` 在独立的伴生容器中评估策略、持有网关凭据并代理已批准的出站流量。

新容器通过独立的私有 Unix socket 卷 `/.openshell/channel/control/control.sock` 连接 supervisor；工作负载网络保持 `none`。supervisor 的 bootstrap-key 挂载保持只读。重启元数据记录方向，缺失元数据时保留旧版运行时监听器。为新容器部署匹配的 supervisor 和 sandbox 运行时二进制。

驱动负责提供放置、身份、凭据、传输和生命周期。共享隔离接口提供 exec、attach、signal、terminate、二进制身份、DNS、TCP 和回环转发语义。

## 运行时姿态

调用方驱动配置默认禁用。现有卷需要管理员控制的批准标签；在强制执行下，bind 和补充镜像挂载会被拒绝。仅凭私有卷名称不能证明所有权。GPU 设备暂时豁免。准入在启动、重启前以及定期对运行中的工作负载执行。
参见[资源准入配置](../../docs/how-it-works/gateways/configuration.mdx#external-resource-admission)。

| 属性 | 工作负载 | Supervisor |
|---|---|---|
| UID/GID | 固定的非 root 工作负载身份 | 相同的映射身份 |
| 能力 | 全部丢弃；不添加 | 全部丢弃；不添加 |
| Seccomp | 运行时默认加沙箱安装的过滤器 | 运行时默认 |
| 网络 | `none`；仅回环 | Podman 主机网络 |
| 网关 JWT 和上游凭据 | 从不挂载 | Podman secrets |
| 用户卷和 CDI 设备 | 仅工作负载 | 从不挂载 |
| 通道 | 私有命名卷，可写 | 同一卷，只读 |

Podman 在工作负载运行前创建命名空间和卷所有权。无根操作使用操作员的 Podman 服务和从属 ID 配置；无需为任一容器添加能力。supervisor 仅加入工作负载的**用户命名空间**，以保留共享卷访问的 UID/GID 映射。PID、挂载和网络命名空间保持独立。通道卷使用共享 SELinux 重标记（`:z`）。

在启动任一容器前，驱动将卷相对归档直接上传到通道和工作区卷目标。对已停止的 Podman 容器执行 rootfs 上传不会填充嵌套命名卷。重启仅将通道 bootstrap 恢复到现有通道卷，保留工作区。工作负载先于 supervisor 启动，以便 supervisor 加入时其用户命名空间已存在；已停止的 supervisor 在下次启动时重新解析该命名空间。驱动创建由工作负载最终 UID 和 GID 拥有的托管工作区卷，因此工作负载从不以 root 启动。自定义镜像工作区没有工作区卷或上传。

运行时必须通过沙箱的非特权强制执行探测，包括嵌套 seccomp 通知和 Landlock。不支持的运行时默认值会失败关闭；不要切换到非受限 profile 或添加能力。

## 受保护通道和网络强制执行

```text
agent -> openshell-sandbox === authenticated gRPC / private UDS === supervisor -> network
         network=none           TCP, DNS, control streams         policy + gateway JWT
```

工作负载没有外部接口或发布端口。Seccomp socket 中介通过一个经过认证的 gRPC 连接承载 TCP 和 DNS。DNS 仍由 supervisor 中介；不支持通用 UDP。驱动在隔离的工作负载网络命名空间中设置 `net.ipv4.ip_unprivileged_port_start=0`，使沙箱的回环 DNS 中继无需能力即可绑定端口 53。沙箱内不运行 nftables 或嵌套网络命名空间设置。

通道仅包含沙箱 bootstrap 和沙箱侧 TLS 身份。supervisor 私钥和运行时描述符保留在 supervisor 的私有文件系统中。Landlock 拒绝 agent 访问顶层 `/.openshell` 控制层级。驱动在启动和重启前验证 Podman 报告的 `network=none` 围栏。主机网络适用于 supervisor，而非 agent。

网关会话使用沙箱 JWT 和可选的服务器认证 TLS。仅网关 CA 交付给 supervisor；用户客户端证书和私钥不挂载到任一容器。沙箱/supervisor 通道始终使用其独立的、每沙箱的 mutual TLS 材料。这些是不同的认证关系。

## 身份和可信二进制

工作负载和 supervisor 镜像均通过不可变镜像 ID 固定。驱动从已停止的工作负载镜像容器读取账户文件；它从不执行镜像来解析账户。策略身份字段独立覆盖 OCI `USER`。命名用户/组针对该镜像解析，包括补充组。Root 和未解析身份在配置前失败。镜像不得预填充保留的 `/.openshell` 层级；这防止镜像控制的符号链接将私有控制状态别名到用户挂载中。

`sandbox_runtime_image` 提供静态链接的 musl `/openshell-sandbox` 二进制。Podman 的只读镜像卷将其交付给工作负载；无法使用镜像卷的用户命名空间模式保留可信二进制提取路径。`supervisor_image` 在工作负载外提供动态链接的 glibc `/openshell-supervisor` 二进制。镜像和请求环境属于 agent 子进程，从不属于 supervisor 进程。

## OCI 工作目录

OpenShell 从工作负载镜像读取 `WORKDIR`。如果未设置、为 `/` 或 `/sandbox`，OpenShell 使用其托管的 `/sandbox` 工作区卷。自定义路径必须为绝对且规范化，且不能与 `/proc`、`/sys`、`/dev`、OpenShell 保留路径或工作负载的私有控制和 CA 挂载重叠。镜像卷和驱动挂载不能覆盖它；嵌套在其下的挂载仍然有效。

自定义路径保留在镜像的容器文件系统中，及其所有权和权限。工作负载以最终非 root 用户启动，该用户必须能够访问并写入该目录。Agent 命令使用该路径作为工作目录。

自定义工作区内容在同一工作负载容器的停止/启动后保留，但随该容器删除而移除。它们不位于托管命名卷中，也不会复制到替换容器。

## 生命周期和就绪

Create 构建两个已停止的容器，并在启动任一容器前暂存私有归档。沙箱在 supervisor 认证并确认公共边界契约前不执行 agent。创建失败仅移除该尝试创建的容器，然后清理驱动拥有的卷和 secrets。

Stop 保留两个容器、工作区、通道和 secrets。Start 从 supervisor 私有文件系统中的副本恢复已消耗的沙箱 bootstrap，验证围栏，并启动同一对容器。supervisor 启动失败会停止工作负载。Delete 先移除伴生容器，然后移除工作负载、通道、工作区和驱动拥有的 secrets。用户拥有的卷保留。

仅工作负载容器出现在沙箱 list/watch 结果中。就绪使用 supervisor 的私有健康 socket；没有 shell、旧版标记或 TCP 监听器捷径。Watch 协调和 supervisor 退出/移除事件会停止伴生容器不可用的运行中工作负载。网关在发布 Ready 前还要求经过认证的 supervisor 会话。

## 挂载、GPU 和配置

用户 `bind`、`volume`、`tmpfs` 和 `image` 挂载以及 CDI GPU 选择保持原生 Podman 特性，仅适用于工作负载。Bind 挂载需要操作员的 `enable_bind_mounts` 选择加入和禁用标签准入。补充镜像挂载也需要禁用准入。驱动 JSON 需要 `allow_driver_config = true`。保留控制路径和工作区根不能被替换。用户拥有的卷从不创建或删除。

参见[网关配置](../../docs/how-it-works/gateways/configuration.mdx)了解操作员设置，以及 [NETWORKING.md](NETWORKING.md)了解 supervisor 网络。supervisor 使用 Podman 的主机网络并拥有上游代理设置。省略 `health_check_interval_secs` 以禁用 Podman 的定期健康命令。显式零无效。OpenShell 仍以 supervisor 的认证健康信号作为就绪门槛。

网关 OTLP 配置继续在 `openshell-driver-podman` 服务下导出计算驱动 span，保留网关跟踪上下文。