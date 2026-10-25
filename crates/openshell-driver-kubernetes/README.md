# openshell-driver-kubernetes

用于 OpenShell 集群部署的 Kubernetes 计算驱动。

调用方驱动配置默认禁用。外部资源引用在每个工作区模式下都需要管理员控制的审批标签，包括重启和调度门释放之前。GPU 设备暂时豁免。镜像拉取 Secret 是操作员选择的网关配置，而非调用方附加项。托管模式为每个沙箱运行时世代暂存一份不可变副本。
参见[资源准入配置](../../docs/how-it-works/gateways/configuration.mdx#external-resource-admission)。

调用方 PVC 挂载不能与驱动管理的 `/.openshell` 控制树或 `/run` 重叠，包括嵌套的 `subPath` 挂载。资源审批标签不允许在这些路径上挂载。请使用 `/data` 或 `/sandbox` 下的目录等数据路径。

驱动通过 Kubernetes API 创建、删除、获取和监视沙箱自定义资源。它与网关服务器在同一进程内运行，并通过 `workspace_mode` 支持三种工作区命名空间模式：

- **Shared**（默认）：所有沙箱渲染到单个静态命名空间。资源名使用 `{workspace}--{name}` 以避免冲突。
- **Managed**：驱动为每个工作区自动创建/删除 K8s 命名空间（`openshell-{gateway_id}-{workspace_name}`），并在每个命名空间中创建 ServiceAccount。在 OpenShift 上，它将 SCC 注解留给命名空间分配器，并等待命名空间自身的 MCS、UID 范围和补充组注解后再配置沙箱资源。已有 UID 范围但无 MCS 的命名空间必须重建，以便 OpenShift 分配完整的 SCC 注解集。
- **Operator**：工作区名称与预配置命名空间一一对应，通过恰好一个来源发现：标签选择器（`operator_namespace_label`）或即插即用允许列表文件（`operator_namespace_file`）。如果工作区命名空间不在当前允许列表中，沙箱创建将失败关闭。使用标签选择器时，在监视器首次列出命名空间之前运行的工作区检查最多等待 10 秒，然后返回 `Unavailable`，因此网关重启不会将带标签的工作区拒绝为未知。工作区删除仅移除网关状态；它从不删除或以其他方式访问操作员管理的 Kubernetes 命名空间。

当网关配置 `[openshell.gateway.otlp]` 时，Kubernetes 计算驱动 span 导出到同一 OTLP/gRPC 收集器，服务名为 `openshell-driver-kubernetes`。驱动保留网关跟踪上下文，并在其进程内和独立形式中使用相同的计算驱动 RPC span 名称。独立部署设置 `--gateway-name` 或 `OPENSHELL_GATEWAY_NAME`，以便导出的 span 携带与网关 span 相同的 `openshell.gateway.name` 资源属性。

创建 Agent Sandbox 资源时，驱动将活动 W3C 跟踪上下文序列化到控制器保留的 `opentelemetry.io/trace-context` 注解中。因此，启用 OTLP 的 Agent Sandbox 控制器可以将其异步协调 span 附加到发起它的 OpenShell 创建跟踪。

工作区命名空间模式假定对沙箱身份资源链拥有独占控制。在共享和托管模式下，只有驱动及其受信任的 Agent Sandbox 控制器可以管理沙箱命名空间、Sandbox CR、沙箱 Pod 或配置的沙箱 ServiceAccount。在操作员模式下，平台操作员拥有命名空间生命周期，但必须阻止其他主体创建或修改 Sandbox CR、使用伪造的 owner reference 创建沙箱 Pod，或使用配置的沙箱 ServiceAccount。将命名空间添加到操作员允许列表即视为授予此信任；允许列表不是租户隔离边界。

## 运行时模型

网关存储平台状态，并将沙箱工作负载创建委托给此驱动。Kubernetes 负责调度和 Pod 生命周期。工作负载 Pod 从 `sandbox_runtime_image` 暂存静态链接的 musl `openshell-sandbox` 二进制文件，而直接管理的 Pod 运行来自 `supervisor_image` 的动态链接 glibc `openshell-supervisor`。

沙箱拥有 agent 进程，应用 Landlock 和子进程 seccomp 过滤器，识别每个网络系统调用背后的二进制文件，并将中介流转发到 supervisor。supervisor 使用 JWT 向网关认证，加载策略和提供方状态，执行目标和 L7 授权，并打开上游连接。工作负载不接收网关凭据、提供方身份套接字或企业代理凭据。

两个 Pod 都以命名空间解析的非 root UID/GID 运行，设置 `allowPrivilegeEscalation: false`、`capabilities.drop: [ALL]` 和运行时默认 seccomp 配置文件。沙箱安装嵌套的 seccomp 用户通知过滤器，而不在 Pod spec 中请求任何能力。当运行时阻止所需的 seccomp 或 Landlock 操作时，启动将失败关闭。

supervisor Pod 对 Sandbox 资源具有直接的非控制器 owner reference。这将它的垃圾回收生命周期链接到沙箱，而不与 Agent Sandbox 控制器竞争工作负载 Pod 的所有权。

当网关导出 OTLP 跟踪时，驱动将 supervisor Pod 上的 `OPENSHELL_OTLP_ENDPOINT` 设置为网关端点，并将 `TRACEPARENT` 设置为创建该 Pod 的操作的跟踪上下文。supervisor 在那里导出其 span，并将其启动 span 的父级设为该上下文。该端点无法在驱动 TOML 中配置。

驱动在释放任何工作负载 Pod 之前创建一个命名空间范围的 `NetworkPolicy`。它选择所有 OpenShell 工作负载，拒绝所有工作负载出站流量，并允许 OpenShell supervisor Pod 访问沙箱 TLS 端口。经过认证的 Sandbox Protocol 将每个连接绑定到确切的沙箱和 supervisor Pod 身份。除非操作员策略限制，supervisor 对网关、DNS 和策略批准的上游连接具有正常出站流量。集群 CNI 必须为每个沙箱命名空间强制执行入站和出站 `NetworkPolicy`。Kubernetes 接受策略对象而不确认强制执行，因此操作员必须在运行沙箱前验证 CNI 支持。

### 控制连接方向

每个新运行时世代使用 `runtime-initiated` 控制。Sandbox 注解 `openshell.ai/sandbox-runtime-control-connection` 记录模式，绑定到 `openshell.ai/sandbox-runtime-control-generation`。缺少标记或当前运行时世代不同标识一个遗留的 `supervisor-initiated` 世代，该世代在被替换前继续工作。停止/启动会自动升级它：移除两个旧 Pod 后，驱动添加 supervisor Service，并在释放替换工作负载之前记录新模式和世代。重建两个 Pod 的网关恢复也会升级世代。协调从不迁移正在运行的世代。未知注解值将失败关闭；没有 rollout 标志。

只有拨号方向不同。在两种模式下，沙箱运行时都呈现世代固定的 TLS 服务器身份并提供 gRPC，supervisor 验证该身份，充当 gRPC 客户端，并发送 Sandbox Protocol JWT。每个沙箱的 Service `os-boundary-<id>` 始终选择配对的工作负载 Pod。单独的 `os-supervisor-<id>` Service 始终选择配对的 supervisor Pod。新网关创建两个 Service，在重启期间保留遗留 Service 的 UID、ClusterIP 和选择器，以便旧网关在滚动升级期间仍能替换世代。

对于 `runtime-initiated`：

- 引导 Secret 向运行时提供 Service ClusterIP 和端口。运行时以有上限的退避重试，直到 supervisor 监听，因此两个 Pod 在引导期间互不等待。Service 设置 `publishNotReadyAddresses`，因为 supervisor 只有在运行时通过它附加后才变为 Ready。工作负载仍然无法访问控制端口。
- 每个沙箱的 `NetworkPolicy` `os-control-<id>` 仅选择此沙箱的两个 Pod，并允许一个流：从其工作负载 Pod 到其 supervisor Pod 的 `boundary_port` TCP 连接，作为工作负载出站和 supervisor 入站。supervisor 可能已被托管 SSH 入站策略隔离入站；当该策略禁用时，此策略隔离它们。Exec、文件同步和日志不需要其他 supervisor 入站，因为 supervisor 打开这些连接。CNI 仍必须允许 kubelet 就绪探针。命名空间范围的策略不变。
- 策略的 UID 和 resourceVersion 绑定到边界资源声明和外层围栏证据中。如果策略消失、偏离或更改世代，或者活动控制 Service 不再选择记录模式所需的角色，协调将暂停工作负载。
- 两个 Service 和控制策略都由 Sandbox 资源拥有，并且仅随该沙箱删除。

在使用 `runtime-initiated` 之前，部署支持它的网关、supervisor 镜像和沙箱运行时镜像。较旧的二进制文件拒绝新的描述符形状并失败关闭。

在混合版本 rollout 期间，旧网关可以通过遗留 Service 创建自己的兼容 supervisor/运行时对来重启沙箱。它保留未知注解，因此模式标记仅在其记录的世代中被接受。不要用无法读取其描述符的二进制文件替换单个世代的 supervisor 或运行时二进制文件。

每个沙箱世代使用两个不可变引导 Secret。受信任的 init 容器将沙箱引导暂存到内存中，沙箱在启动不受信任代码之前将其移除。另一个 Secret 仅由 supervisor 挂载；当配置 `proxy_ca_bundle` 时，它还携带操作员的企业代理 CA 包，网关从其自己的文件系统读取，因此锚点保持在网关的信任域中，而不是沙箱命名空间。TLS 通道绑定命名空间、Sandbox CR、工作负载 Pod、supervisor Pod 和共享网络策略身份，以及 `runtime-initiated` 沙箱的每沙箱控制策略。停止删除工作负载和 supervisor Pod。启动轮换两个 Secret，并在释放新工作负载 Pod 之前创建新 supervisor Pod。它还从网关配置重新渲染受信任的 init 容器，因此由旧网关创建的沙箱以当前沙箱运行时镜像启动。共享网络围栏在命名空间的生命周期内保留。

Kubernetes 策略是累加的，API 不证明 CNI 强制执行它们。保持沙箱命名空间的管理性：不受信任的主体不得创建宽松策略、创建 Pod、读取引导 Secret 或伪造 OpenShell 角色标签。确切的 supervisor 到沙箱授权仍由 TLS、JWT 声明、会话世代和记录的 Pod UID 强制执行。

### 共享 supervisor 拓扑

使用 `supervisor_topology = "gateway"` 时，每个新运行时世代成为 Helm 部署在网关命名空间中的一个 supervisor 进程上的沙箱实例，而不是拥有自己的 supervisor Pod 和边界 Service。沙箱实例是稳定 Sandbox 资源的一个运行世代。世代启动时，其放置记录在 Sandbox 资源上（`openshell.ai/sandbox-runtime-supervisor-topology: gateway` 以及 `openshell.ai/sandbox-runtime-supervisor-session` 中的实例路由，其键保持既定名称）。每个后续生命周期决策都读取该记录，因此更改设置永远不会重新解释正在运行的世代，未知或不完整的记录将失败关闭。

- **所有权。** 网关拥有共享 supervisor 托管哪些实例。当网关二进制文件使用 `KubernetesComputeDriver::with_shared_supervision` 构建此驱动时，它构造 `openshell_supervisor_control::SharedSupervision`，通过私有 `openshell.supervisor.host.v1` 流向 supervisor 提供其分配，并铸造 supervisor 通过 TLS 拨号时使用的 supervisor 令牌（参见 `openshell-supervisor` README）；该令牌仅授权分配流。协调器拥有控制契约（启动材料在分配实例之前持久化，恢复是对列表之前读取的基线进行比较并替换，不完整的恢复永远不会替换已发布的集合，主机观察只解释一次）；此驱动将它们适配到 Kubernetes 对象。独立驱动二进制文件无法托管该拓扑，并拒绝选择它的配置。
- **Supervisor 令牌传递。** 网关在其生命周期的一半（`gateway_jwt` 扩展令牌 TTL，默认 15 分钟，上限为一小时）续订令牌，协调器重试失败的传递，直到成功或被更新的令牌取代。驱动的部分是写入：它替换 `service_namespace` 中 chart 创建的 `shared_supervisor.token_secret_name` Secret 的 `token` 键，supervisor Pod 挂载该 Secret，并在重试错误中指明缺失的 Secret 或被拒绝的补丁。
- **创建和启动。** 驱动创建每沙箱出站允许 `os-shared-control-<id>`，从工作负载 Pod 到共享 supervisor 的 Service 选择的 Pod，在运行时端口上，将其 UID 和 resourceVersion 绑定到边界证据中，并构建实例的注册（路由、沙箱名称、来自 Sandbox 资源的主进程 spec 和 agent 镜像、配置的运行时描述符、世代的启动凭据、其代理 CA 以及当前跟踪上下文）。协调器验证并编码它，通过驱动将启动材料持久化到世代的 `os-supervisor-<id>-<generation>` Secret（键 `instance.json`，由工作负载 Pod 拥有），然后在驱动释放工作负载 Pod 之前分配实例。分配不等待 supervisor，因此创建仍在网关持久化世代之前返回。运行时身份的最后一个组件是实例路由，任何 Pod UID 都无法等于它，因此 `authenticate_sandbox` 无法为共享世代担保。
- **就绪。** 当协调器报告其实例已被监督并被网关接受，且驱动找到其围栏和出站允许时，共享世代可用。supervisor 已协调过去且不托管、为另一个沙箱托管或报告已结束的实例不可用，协调将暂停工作负载。断开的 supervisor 是未知的，尚未协调到分配的 supervisor 是降级的；两者都不暂停任何东西。引导完成在引导宽限期内等待相同的判定。
- **停止和删除。** 驱动在暂停或删除工作负载之前撤回实例，世代清理删除其启动 Secret。每次协调还从可能运行的 Sandbox 资源恢复期望集合：驱动在列出它们之前开始恢复，列出可能运行的共享世代及其路由，并按请求读取每个启动 Secret 的字节；协调器重用当前分配，解码并将持久化材料绑定到记录的路由和沙箱，在任何读取失败时保留先前的集合，并拒绝与生命周期编辑竞争的列表。已删除 Sandbox 的实例和网关停机期间进行的停止会被撤回，重启的网关在发送任何内容之前恢复其分配。
- **隔离。** supervisor 令牌是其自己的配置文件，与每个实例的凭据分开；网关在所有其他 RPC 上拒绝它，并在分配流上拒绝所有其他身份，并在打开流的令牌过期时结束流。沙箱无法访问该流，路由前言仅选择 supervisor 要求的世代固定 TLS 身份。

该拓扑不支持上游代理或提供方 SPIFFE 身份，共享 supervisor 是其托管的沙箱的单点故障。共享沙箱实例不导出 supervisor OTLP 跟踪或 OCSF 事件；当网关配置该导出时，驱动在启动时发出警告。移除该设置后，协调暂停共享世代，重启创建共享世代从未拥有的专用世代边界 Service。

## Sandbox 资源

驱动与 `agents.x-k8s.io` `Sandbox` 自定义资源配合工作。它在运行时检测提供的 Sandbox API，为网关进程缓存选定的 API 版本，并在可用时使用 `v1beta1`，然后回退到 `v1alpha1`。在就地 Agent Sandbox 升级后重启网关，以便驱动再次检测提供的 API 版本。驱动事件将 Kubernetes 对象状态和平台事件映射到网关使用的共享计算驱动 protobuf 表面。

Kubernetes API 调用使用显式超时，因此当 API 服务器缓慢或不可用时，gRPC 处理器不会无限期阻塞。请求客户端的超时在每个请求发送时开始，因此池化连接空闲的时间不会缩短它。资源和 Event 监视在瞬态监视器错误后以 API 友好的退避就地恢复，避免网关侧监视重启及其相关的监视间隙。

## 工作区持久化

沙箱 Pod 使用 PVC 支持的 `/sandbox` 工作区。init 容器在首次启动时从镜像的原始 `/sandbox` 内容播种 PVC，并写入哨兵，以便后续启动跳过复制。

这是一个临时持久化模型。它在 Pod 重新调度之间保留用户文件，但复制基础工作区，并且不会自动将镜像更新应用于现有 PVC。未来的快照应替换它。

停止在停止其 Pod 的同时保留 Agent Sandbox 资源和工作区 PVC。驱动为 `v1beta1` 设置 `spec.operatingMode: Suspended`，或为 `v1alpha1` 设置 `spec.replicas: 0`。启动为同一资源设置 `Running` 或一个副本，因此替换 Pod 挂载现有声明。删除是唯一移除 Sandbox 资源及其拥有的存储的生命周期操作。驱动从发布的 `Suspended` 条件和后备 Pod 的删除两方面确认停止。遗留 `v1alpha1` 控制器省略可用的停止条件，因此仅 Pod 删除确认其停止。

工作区 PVC 大小默认为 `workspace_default_storage_size`。设置 `workspace_storage_class` 将 PVC 固定到特定 `StorageClass`；空值省略 `storageClassName`，因此应用集群的默认 `StorageClass`。没有默认 `StorageClass` 的集群必须设置此项，否则 PVC 保持 `Pending`，沙箱永远不会启动。这两个字段也可以在运行时通过 `OPENSHELL_K8S_WORKSPACE_DEFAULT_STORAGE_SIZE` 和 `OPENSHELL_K8S_WORKSPACE_STORAGE_CLASS` 提供。两者仅适用于 OpenShell 自动配置的工作区 PVC；当 `driver_config` 挂载在 `/sandbox` 下附加现有 PVC 时，它们无效，因为这会跳过默认 PVC。

## 凭据、TLS 和中继

两个 Pod 都设置 `automountServiceAccountToken: false`。supervisor 接收显式的受众绑定投影令牌，用于一次性 `IssueSandboxToken` 交换。驱动验证该令牌并返回从命名空间、不可变 Sandbox 资源 UID 和 supervisor Pod UID 派生的不透明运行时身份。重启需要恰好一个匹配的 Sandbox 资源，并在轮换 supervisor Pod UID 的同时保留其命名空间和 UID。网关要求经过认证的身份与持久绑定匹配，然后才返回 supervisor 使用的世代绑定会话 JWT。沙箱 Pod 不接收任何令牌。对于 HTTPS 网关连接，supervisor 仅从配置的 TLS Secret 读取 CA。共享模式直接投影 `ca.crt`；托管和操作员模式仅将 CA 暂存到 supervisor 引导 Secret 中。用户客户端证书和私钥不会挂载到任一 Pod。

网关使用 supervisor 中继进行连接、exec、日志