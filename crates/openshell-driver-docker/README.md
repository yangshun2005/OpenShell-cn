# openshell-driver-docker

为本地和远程 OpenShell 网关提供 Docker 支持的计算驱动。

该驱动通过 `bollard` 经配置的 Docker API socket 管理沙箱资源。未设置 `socket_path` 时，它选择第一个响应 API ping 的标准本地 socket。显式选择 Docker 驱动但无候选响应时，回退到 `/var/run/docker.sock`。

网关配置 `[openshell.gateway.otlp]` 时，进程内驱动将 span 导出到与 `openshell-driver-docker` 相同的 OTLP/gRPC 收集器。独立驱动接受 `OPENSHELL_OTLP_ENDPOINT`，从网关 RPC 元数据延续 W3C trace context，并在优雅关闭时刷新 span。

## 运行时模型

调用方驱动配置默认禁用。现有卷需要管理员控制的审批标签；原始 bind 挂载没有受支持的标签解析器，在强制执行下会被拒绝。GPU 设备暂时豁免。在启动、重启和协调期间需再次检查标签。参见[资源准入配置](../../docs/how-it-works/gateways/configuration.mdx#external-resource-admission)。

驱动为每个沙箱创建两个容器：

- `openshell-sandbox` 是工作负载容器中的 PID 1。它负责工作负载进程树、seccomp 通知代理、强制 Landlock 基线、二进制身份、exec/signal/wait/PTY 操作以及回环转发。
- `openshell-supervisor` 运行在独立的伴随容器中。它负责网关会话、策略引擎、凭据、拦截 CA、SSH 中继、L7 检查、DNS 策略以及外部上游连接。

两个容器均以非 root 运行，不请求任何 capability，并设置 no-new-privileges。共享命名卷承载经过认证的 Unix socket 和沙箱引导材料。第二个仅 supervisor 可用的卷承载 supervisor JWT 和网关客户端凭据，且从不挂载到工作负载中。

工作负载使用 `network_mode=none`。其 seccomp 用户通知代理中介所有受支持的 TCP 和 DNS 操作，将其归属到调用二进制，并通过私有通道发送请求。supervisor 在打开上游连接前授权该请求。Docker 缺失的工作负载网络是中介失败或被绕过时的强制外层围栏。受信任的 supervisor 伴随容器发起已批准的出站流量。其运维方拥有的 `supervisor_network_mode` 默认为 `host`。设为 `bridge` 可在 Docker 桥接网络上运行 supervisor；`auto` 也会选择 host。

驱动通过 Docker archive API 从配置的 supervisor 镜像复制受信任的运行时字节。工作负载启动不依赖宿主机 bind 挂载或工作负载镜像提供的工具，因此同一路径适用于本地、远程和 VM 支持的 Docker 守护进程。

## 企业代理出站

`[openshell.drivers.docker]` 接受运维方拥有的企业代理字段 `https_proxy`、`no_proxy`、`proxy_auth_file`、`proxy_auth_allow_insecure`、`proxy_connect_by_hostname` 和 `proxy_ca_bundle`。沙箱环境、镜像内容和每沙箱驱动配置无法选择或覆盖它们。

驱动在网关启动时校验代理 URL 和跨字段关系。使用前有界读取代理凭据和 PEM CA 包。缺失、不可读、为空、超大、格式错误或不含证书的 CA 文件会失败关闭。CA 包需要 `https_proxy`，但当代理拦截目标 TLS 时，代理 URL 可以是 `http://`。

每次 supervisor 启动时，Docker 将凭据和 CA 内容复制到现有的仅 supervisor 可用的命名卷中。它只将固定容器路径 `/.openshell/supervisor/upstream-proxy-auth` 和 `/.openshell/supervisor/upstream-proxy-ca-bundle.pem` 传给 supervisor。它不 bind 挂载网关宿主机文件，从而保留远程守护进程支持，并使宿主机路径不出现在工作负载容器元数据中。

supervisor 信任配置的 CA，用于其到 HTTPS 代理的 TLS 连接，以及由 TLS 拦截代理重新签名的目标证书。它还将企业根证书纳入工作负载进程使用的生成合并信任包中。

## 身份与工作区

创建负载前，驱动固定镜像 ID，并通过已停止的元数据容器读取其 passwd/group 数据库。它将准入的策略身份或镜像 `Config.User` 回退解析为一个精确的非 root UID、主 GID 和补充组集合。Docker 以该身份启动 `openshell-sandbox`，沙箱对每个规范进程和 exec 进程使用同一身份。UID 或 GID 为零以及无法解析的符号身份会被拒绝。

绝对 OCI 工作目录成为工作区。空、根（`/`）或显式 `/sandbox` 声明使用 `/sandbox`。任何其他 workdir 必须已存在且不含符号链接组件。解析后的身份必须能够遍历每个父目录并写入和进入 workdir；OpenShell 不改变其所有权或模式。

镜像 `VOLUME` 声明和用户挂载不得覆盖 workdir、其父目录之一或保留的 `/.openshell` 运行时/通道树。OpenShell 要求内核在最终身份下验证访问，因此 POSIX ACL 和宿主机 LSM 决策仍具权威性。

## 容器契约

| 设置 | 用途 |
|---|---|
| 精确的非 root `user` 和 `group_add` | 使沙箱和工作负载具有相同的不可变 UID/GID/组身份，这是无 capability 观察所必需的。 |
| `cap_drop = ALL`、无 `cap_add`、no-new-privileges | 防止任一容器获取 Linux capability。 |
| Docker 默认 seccomp 和 AppArmor 配置 | 保留运行时加固；嵌套 seccomp 通知不可用时启动确认失败关闭。 |
| 工作负载上的 `network_mode = none` | 移除直接外部路由。 |
| supervisor 上的 `network_mode = host` | 让受信任的 supervisor 访问网关的主回环监听器并发起已批准的上游连接。 |
| `restart_policy = no` | 保持规范主进程退出为终态。 |
| `PidsLimit` | 应用配置的沙箱 PID 预算。省略 `sandbox_pids_limit` 使用 OpenShell 默认值。显式零无效。 |
| 私有命名卷 | 独立卷承载沙箱引导密钥、运行时发起的 Unix socket 和仅 supervisor 可用的凭据。supervisor 以只读方式挂载引导密钥，仅 socket 卷可写。 |
| 内存中的 `/run/openshell-supervisor-ca` tmpfs | 仅存放公共 supervisor CA 证书和信任包，而不使整个 `/run` 可写。 |
| CDI GPU 请求 | 分配驱动配置或基于计数选择所请求的精确已验证 CDI 设备。 |

新容器在 `/.openshell/channel/control/supervisor/control.sock` 连接 supervisor。运行时仍是 TLS/gRPC 服务器；supervisor 验证其固定身份并提供现有 bearer 凭据。已保存的描述符为现有容器保留旧版 Unix 监听器，包括跨重启。新一代要求 supervisor 和沙箱运行时二进制匹配。

## 停止、启动和删除

停止会终止 supervisor 伴随容器并停止工作负载容器而不移除它。Docker 保留工作负载可写层和附加卷。启动会暂存新的沙箱引导包，重启该工作负载，并创建新的 supervisor 伴随容器。持久停止的沙箱在网关重启后保持停止。

删除会强制移除两个容器、驱动拥有的运行时卷和宿主机私有的运行时描述符。缺失或更改的描述符和通道资源会失败关闭；驱动不运行旧的合并 supervisor 布局。

## 驱动配置挂载

网关从 `--driver-config-json` 转发 `docker` 块。支持的挂载类型：

- `bind`：绝对守护进程宿主机路径，仅当 `[openshell.drivers.docker].enable_bind_mounts = true` 且标签准入被显式禁用时允许。
- `volume`：现有命名卷。驱动从不创建或删除用户提供的卷。bind 支持的本地卷需要 `enable_bind_mounts = true`。
- `tmpfs`：内存文件系统，可选大小和模式。

宿主机 bind 挂载默认禁用，因为它们将守护进程宿主机路径暴露给沙箱请求。用户 bind 和卷挂载默认只读。目标必须是绝对、规范化路径，且不能与工作区根或 OpenShell 控制路径重叠。

示例：

```shell
docker volume create openshell-work

openshell sandbox create \
  --driver-config-json '{"docker":{"mounts":[{"type":"volume","source":"openshell-work","target":"/sandbox/work"}]}}' \
  -- claude
```

## 运行时镜像

`sandbox_runtime_image` 包含静态链接的 musl `/openshell-sandbox` 二进制。驱动将该二进制作为字节提取并暂存到已停止的工作负载中。`supervisor_image` 包含在 supervisor 伴随容器中运行的动态链接 glibc `/openshell-supervisor` 二进制。发布和网关镜像构建将匹配的镜像标签烘焙到二进制中。

## 网关会话与 TLS

`OPENSHELL_ENDPOINT` 和网关认证材料仅注入 supervisor 伴随容器。工作负载从不接收沙箱 JWT、网关客户端 TLS 密钥、策略权威或拦截 CA 私钥。

未配置端点时，supervisor 连接 `127.0.0.1:<gateway-port>`。网关不在 Docker 守护进程宿主机上时设置 `grpc_endpoint`。配置的 HTTPS 服务器证书必须在其 subject alternative names 中包含端点主机。

在 host 模式下，驱动将宿主机回环发布为 `host.openshell.internal` 的后端地址。策略 DNS 通过中介路径解析该保留名称，因此策略可以访问宿主机服务，无需 Docker 桥接、容器 DNS 别名或其他网关监听器。

对于 HTTPS 端点，supervisor 仅接收网关 CA，并使用其沙箱作用域 bearer token 认证 RPC。用户客户端证书和私钥不会传递给任一容器。

Docker Desktop 上的 host 模式需要在设置中启用 host networking。对于 bridge 模式或使用 `host.docker.internal` 的网关端点，驱动在初始化时解析一次 Docker 的 `host-gateway` 并缓存。临时容器从不可变 supervisor 镜像运行 `--help`，无网络、无 capability、无权限提升，根文件系统只读。驱动通过 archive API 下载 Docker 生成的 `/etc/hosts`，优先唯一 IPv4 地址，否则唯一 IPv6 地址。解析超时 30 秒；强制清理有独立 10 秒超时，在成功或失败后运行。

驱动将选定地址用于两个主机别名和运行时描述符的信任固定；工作负载 DNS 和 hosts 文件无法选择它。使用其他网关端点的 host 模式保留其网关地址固定。运维方显式拥有的 `host_gateway_ip` 覆盖发现并跳过临时容器。无需 supervisor 更改或每沙箱发现。

省略 `grpc_endpoint` 时，bridge 模式在 Desktop 或通配网关监听器上使用 `host.docker.internal`，其他非回环 Linux 监听器使用具体绑定地址。端口是网关绑定端口；重映射的容器发布需要显式端点。原生 Linux 仅回环监听器无法服务 bridge supervisor，并产生可操作的错误。显式端点保留 TLS 验证且从不重写；bridge 拒绝回环/未指定端点地址。仅绑定到回环的宿主机服务也无法从原生 Linux bridge supervisor 访问。

所选网络模式不改变沙箱强制的 Landlock 和 seccomp 资格。

supervisor 拥有这些安全关键变量：

- `OPENSHELL_ENDPOINT`
- `OPENSHELL_SANDBOX_ID`
- `OPENSHELL_SANDBOX`
- `OPENSHELL_SANDBOX_TOKEN_FILE`
- `OPENSHELL_SSH_SOCKET_PATH`
- `OPENSHELL_MAIN_PROCESS_SPEC`
- 启用 HTTPS 时的 `OPENSHELL_TLS_CA`

模板和沙箱环境编码在受保护的引导中，仅暴露给工作负载子进程。工作负载输入无法覆盖安全关键的 supervisor 变量。