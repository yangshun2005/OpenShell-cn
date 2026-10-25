# OpenShell supervisor

supervisor 负责加载并协调策略、维护提供方凭据、应用网络与 MCP 检查，并驱动已准入的隔离后端完成挂载、确认和工作负载启动。

## 后端启动

supervisor 二进制选择一个后端设置，并将其启动输入收集到公开的 `SandboxInstanceConfig` 中。共享启动流程分别接收该配置和受信任的后端设置。`backend_setup` 模块负责该后端的启动数据解码器、工作负载策略发现和客户端构建。描述符内容不能选择实现。

共享启动流程在将不透明载荷传给解码器之前，先检查已准入的后端名称。随后，在安装凭据或发现工作负载策略之前，将解码出的沙箱、会话和运行时代数与受信任的启动输入进行比较。不匹配则停止启动。

内置解码器还会将 VM 驱动的固定工作负载身份带入共享策略校验。启动流程及后续策略更新必须拒绝与该身份冲突的选择器。其他启动描述符不启用此 VM 专用检查。

supervisor 在构建并挂载所选客户端之前，先准入策略并准备凭据。它使用来自 `openshell-sandbox-backend` 的具体 RPC 客户端句柄 `BoundBoundary` 和 `ConfirmedBoundary`：确认已挂载的边界、准备网络中介，然后启动工作负载。后端通过 Sandbox Protocol 提供原生执行证据；客户端在暴露已确认句柄之前对其进行校验。

客户端接收 supervisor 的实时提供方状态、bearer 令牌槽和 CA 路径槽。提供方刷新、令牌轮换以及后续 CA 发布必须通过这些共享句柄保持可见。启动流程不会为其当前值创建独立副本。

`backend_setup`、`SandboxInstanceConfig`、`SandboxInstanceBindings` 和 `run_sandbox_with_backend` 为公开项，以便 supervisor 二进制选择已注册的后端。外部后端应实现 `IsolationBackend` 和 `SandboxProtocol` 服务，而不是链接此 crate。

启动结果将后端传输载荷与可选的已认证 CONNECT 监听器设置分开。共享网络仅在边界挂载和确认之后启动监听器；supervisor 拥有策略评估、实时凭据和监听器生命周期。这些设置不会向隔离接口或共享 Sandbox Protocol 描述符添加代理 Hook。

## 沙箱实例所有权

一个沙箱实例是稳定 Sandbox 资源的一个运行代数。网关连接以及承载于其上的网关 supervisor 会话可以在一个实例内重连，而不会创建另一个实例；代数仍是隔离标识符。

`run_sandbox_instance` 从三个输入监督一个沙箱实例。`SandboxInstanceConfig` 是唯一的启动配置：身份、命令、策略输入、认证包、后端描述符和文件位置。`SandboxInstanceBindings` 是其所有者一次性准备的实时状态：从认证包签发的 `GatewayCredentials`、OCSF 上下文和 JSON 设置、日志接收器，以及由 `policy.local` 提供的拒绝历史。`SandboxInstanceControls` 携带所有者的停止信号和就绪状态。其余一切，包括凭据续期、策略、提供方凭据和令牌授予、中间件凭据、后端连接和访问平面，均由实例创建。网关客户端接收一个 `GatewayEndpoint`，它将 URL 与该实例的 `GatewayCredentials` 配对；不会在进程范围内安装任何内容。

专用进程包装器保持现有进程契约。二进制从其参数和环境加载启动配置，并准备 `SandboxInstanceBindings::dedicated`，与日志推送共享一个 `GatewayCredentials`，与其日志层共享 OCSF JSON 设置，因此任一网关客户端的续期都能保持两者最新。`run_sandbox_with_backend` 设置进程范围的 OCSF 上下文，将就绪套接字和端口绑定到实例就绪状态，并在 SIGTERM 或 SIGINT 时停止实例。

`sandbox_instance_host::SandboxInstanceHost` 在一个进程中运行多个沙箱实例。每个实例获得自己的 Tokio 运行时；该运行时的每个线程以及驱动它的线程都绑定到实例的 `SandboxInstanceBindings`：OCSF 上下文、OCSF JSON 设置、日志接收器和拒绝日志。因此，派生的任务、阻塞工作、回调、重连和续期都会继承实例的身份，而实例之外的线程不携带任何身份，也不推送日志。OCSF 上下文还在中间件请求中提供沙箱身份。

host 是其实例的唯一注册表。`admit` 校验键是单个路径组件，然后在注册表锁下，对已占用的键再次接受相同的启动指纹，拒绝不同的指纹，并在键正在移除时拒绝任何准入。新实例在 host 根目录下获得一个空的私有目录，`sandbox_instance_launch` 在那里准备其绑定和主体。准入失败不会留下条目和文件。

移除实例会停止其工作负载，仅关闭其运行时（这会丢弃其运行时路由），并撤销其网关凭据；只有在这之后，才会删除其私有目录并释放其键，因此替换实例永远不会遇到旧实例的资源。移除在一个分离的任务上运行，所有移除者都加入该任务，因此断开的调用者无法取消它，并且它在释放键之前检查条目的序列号。如果目录无法删除，键保持保留并报告失败阶段，下一次移除会重试删除。发生 panic 的实例报告失败并以相同方式被撤销。仍在启动中的实例会立即被取消；沙箱运行时随后失去监督并故障关闭，与专用 supervisor 退出时一样。没有显式代理信任目录的托管实例使用其私有目录。托管进程必须安装 `LogPushLayer::thread_bound` 和一个写入 `SandboxInstanceDenialWriter` 的 `OcsfShorthandLayer`。

## 控制台日志

诊断信息和 OCSF 简写共享一个有界、非阻塞的 stderr 写入器。当 `ocsf_json_enabled=true` 时，同一写入器还会接收带时间戳的 `OCSF-JSON` 记录。每个格式化器在一次写入中提交完整行，因此并发生产者无法在队列中交错记录。1,024 行的队列在满时丢弃新行，而不是等待 stderr。

控制台 JSON 独立于可选的文件追加器安装，并使用独立于诊断过滤器的 INFO 过滤器。其运行时启用标志和目标 Schema 版本与现有 JSONL 文件层共享。supervisor 保留写入器守卫直到关闭。网关日志推送层继续仅发出简写。

## 安全运行时与父进程存活

supervisor 驱动安全隔离后端契约以及网络/策略编排。其库和可执行文件禁止 unsafe 代码。

对于 VM 主机 supervisor，VM 驱动将其父进程存活管道作为 stdin 传入，并设置私有的 `--parent-liveness-stdin` 标志。专用读取器在驱动关闭其写入端点或退出时终止 supervisor。驱动使用 close-on-exec 保护两个原始管道端点，并让 `Stdio` 将读取端点转移到子进程的 stdin。supervisor 使用标准 stdin 读取器，不采用数字原始描述符。此模式将 stdin 保留用于存活检测；沙箱工作负载输入通过边界协议传输。

## 可移植 supervisor 访问

supervisor 使用共享隔离后端契约。其网关会话、规范进程挂载和 TCP 就绪状态不需要 SSH 或 Unix 主机。

TCP 就绪状态仅在网关接受已认证会话后打开。会话丢失会关闭就绪状态；接受的重连会恢复它。丢弃就绪守卫会关闭监听器。请求的 Unix 就绪端点在不受支持的主机上会显式失败，甚至在会话接受之前也是如此。

可选的 Unix SSH 适配器仍与基于边界的进程 I/O 分开。进程访问多路复用器使用边界提供的流，并通过现有隔离契约委托终端调整大小和信号。它不拥有本地 PID、PTY 或进程组信号；原生进程操作属于沙箱或隔离后端。信号使用 `BoundarySignal`。

这些可移植控制平面基础不会使平台隔离后端合格，也不会启用 Windows 工作负载运行时。

## 共享 supervisor host

`openshell-supervisor --mode=shared --gateway-endpoint <https-gateway-url> --token-file <file> --address <ip:port>`
在一个进程中托管多个沙箱的沙箱实例，用于 Kubernetes 网关 supervisor 拓扑。仅限 Unix。host 不打开控制监听器：它拨号网关并从网关接收其分配。

- **网关挂载。** host 拨号 `--gateway-endpoint`（必须为 `https`），像每个沙箱实例一样使用 `OPENSHELL_TLS_CA` 和 `OPENSHELL_GATEWAY_TLS_SERVER_NAME` 验证网关，并打开私有的 `openshell.supervisor.host.v1/SupervisorHost.Attach` 流（`proto/supervisor_host.proto`），以 `--token-file` 中的 **supervisor 令牌**作为其 bearer。网关使用其启动签名密钥在其自有配置文件下铸造该令牌（`typ` 为 `openshell-supervisor+jwt`，主题为 `openshell-supervisor:<gateway_id>`，无沙箱、代数或纪元声明），并通过其部署打包（Helm chart 的 `<release>-supervisor-token` Secret）保持文件新鲜。该令牌仅授权此流；沙箱、扩展、用户、操作员和对等凭据在此流上被拒绝。host 在每次连接尝试时重新读取文件，并将缺失、格式错误或过期的令牌视为稍后重试的理由，网关在打开流的令牌过期时结束流，因此 host 使用续期后的令牌重连。host 从不以任何其他身份行事，也没有 Kubernetes 身份。流以携带每进程实例 ID 和连接纪元的 hello 开始；网关取代同一进程的较旧流，并拒绝延迟的较旧连接。
- **分配。** 网关拥有期望集，并将其作为带有会话范围修订的完整 `AssignmentSnapshot` 发送。`shared_host::SharedHost::reconcile` 整体校验快照（最多 256 个实例，无重复路由），解码每个分配的启动材料（`openshell-supervisor-control` 的 JSON `SandboxInstanceRegistration`），将其准入到以路由为键、以注册摘要为指纹的 `SandboxInstanceHost` 注册表中，并移除快照省略的每个托管路由。相同的分配是空操作；无法准入的分配（无效材料，或活动路由的不同指纹）被报告为失败实例，且从不触碰正在运行的实例。快照在单独任务上一次协调一个，因此实例停止时心跳和状态仍持续流动；仅保留最新的未应用快照，并且一个连接在下一个连接开始之前完成其启动的协调。私有文件位于 `--session-root` 下（代理 CA、信任目录、SSH 套接字）；注册表拥有移除权，并保留路由直到实例结束且其文件被删除。
- **状态。** 每次协调后，以及每当实例被准入、阶段或就绪状态变化或被移除时，host 发送一个完整的 `HostStatus`，指明其已协调到的修订；就绪状态仅在网关接受实例自身的 supervisor 会话后设置。
- **断开连接。** 丢失网关流永远不会停止实例。host 以退避重连，报告其运行的内容，并应用新会话的第一个快照；只有省略实例的快照或进程关闭才会停止它。
- **运行时连接。** 一个路由监听器（`--runtime-listen`）接受每个实例发起的 Sandbox Protocol 连接；路由前言参见 `openshell-sandbox-backend`。
- **健康检查。** `--address` 是一个普通 TCP 监听器，接受并关闭 kubelet `tcpSocket` 探针连接；它不承载控制流量，仅报告进程正在运行。SIGTERM 和 SIGINT 停止每个实例，从而结束其工作负载的监督。

沙箱实例不会获得上游代理、额外 CA 包或提供方工作负载身份：当配置了上游代理或提供方 SPIFFE 身份时，Kubernetes 驱动会拒绝该拓扑，因此工作负载永远不会静默丢失它们。host 也不初始化任何 OTLP 跟踪或 OCSF 导出器，因为一个进程无法使用每个实例的收集器标头和身份导出该实例的记录。其控制台日志和每个实例的网关日志推送仍然保留，并且当网关配置 supervisor 导出时，Kubernetes 驱动会在启动时发出警告。