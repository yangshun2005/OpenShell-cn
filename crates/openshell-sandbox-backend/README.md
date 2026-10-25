# OpenShell 沙箱后端

本 crate 实现了具体的 supervisor 侧 RPC 客户端，以及与 `openshell-sandbox` 共享的版本化控制协议。

protobuf 服务取代了 Rust 后端 trait。supervisor 持有由 RPC 支撑的具体生命周期、进程、exec、转发和中介句柄。确认（confirmation）消费绑定的句柄，校验强制执行证据，并产出就绪句柄；启动（starting）消费该就绪句柄。内置的 Linux 运行时和外部后端都实现生成的这些服务。Linux 进程辅助函数仍保留在运行时服务器内部。

## 边界启动与路由

supervisor 使用生成的 `IsolationBackend` 控制服务协商能力，并在读取镜像策略之前打开边界。每条物理连接都重复该握手，包括凭据纪元变更和传输恢复。启动会话和代次标识重试；描述符必须绑定到同一预置资源。在运行时生命周期内重新打开返回同一句柄。驱动删除会终止运行时并使该句柄失效。

隔离协议 1.0 包含边界路由，并要求双方都提供 `openshell.sandbox.protocol`。共享扩展协商器检查主版本号及双方所需能力。内置运行时不宣告 `openshell.sandbox.suspend-resume`，对这些可选 RPC 返回 `UNIMPLEMENTED`。

`SandboxProtocol.Exchange` 和 `.Mediate` 以 `OpenBoundary` 返回的句柄开始。服务器认证流的 bearer，对照其预置的沙箱和代次检查句柄，然后接受类型化 protobuf 消息。`ExchangeFrame` 承载控制、进程 I/O、网络决策和 TCP 中继数据。`MediateFrame` 承载 OpenMediation 握手和 DNS 消息。缺失绑定、重复绑定以及不匹配操作或方向的帧都会被拒绝。线上没有嵌套 JSON 或长度/通道分帧。私有进程内适配器保留 Rust 字节流接口。

打开边界只校验并预留路由。它既不附加 supervisor，也不运行工作负载代码。策略发现之后是现有的附加、中介设置、确认和显式启动序列。现有连接注册表继续管理 supervisor 替换和恢复。

沙箱和代次标识来自已验证的 bearer，而非 `OpenBoundaryRequest` 上的重复字段。附加提供策略；确认报告来自后端自有预置状态的资源声明，supervisor 将其与可信启动元数据比较。

请同时更新 supervisor 和沙箱运行时；旧版本对端会在启动时失败，而不是走回退路径。

## 端点注册的后端

`OpenShellRuntimeBackend::registered` 为运维注册的后端构造相同的客户端和生命周期句柄。可信端点注册和 `DelegatedLaunch` 元数据与驱动的不透明描述符分离，后者原样穿过 `OpenBoundary`。通用确认检查绑定身份、会话、代次、资源声明和驱动的预置证据摘要。原生证据由注册后端校验；Linux 证据校验仍在内置适配器中。

公开的 gRPC 契约位于
[`isolation_backend.proto`](../../proto/isolation_backend.proto) 和
[`sandbox_protocol.proto`](../../proto/sandbox_protocol.proto)。

## Exec 恢复

exec 信封携带 UUID 请求 ID 和一个设为创建后 30 秒的 protobuf 时间戳。传输恢复保留整个请求，包括该截止时间。相同 ID 且解码字段等价的重复变更会重放原始结果；冲突重用会被拒绝。Map 条目顺序无关紧要。接收方计算本地重放指纹；客户端不传输摘要。supervisor 和沙箱需要同步的墙上时钟。

沙箱在启动或重新附加 exec 之前检查过期。它保留每个已准入请求 ID 直到该截止时间，独立于进程和 I/O 保留。过期请求（包括延迟的首次尝试）即使其 ID 已被丢弃也会被拒绝。沙箱从不将其观察到的准入时钟回拨，因此时钟调整无法复活已丢弃的请求。过期 ID 在下一个 exec 请求到达时回收。

exec 请求数量没有生命周期上限。现有并发进程保留限制仍然适用。截止时间管理准入和恢复；它不终止已在运行的命令。恢复超时可能导致执行结果未知。

没有过期时间的 exec 信封会被拒绝。部署此协议变更时请同时更新 supervisor 和沙箱运行时。

## 边界审计校验

`OpenShellRuntimeBackend` 实现共享 Sandbox Protocol 的已认证主机侧。

对于内置 Linux 后端，客户端校验原生审计证据，拒绝不完整或外来的测量值。外部后端校验自身机制，可省略审计数据；它们无需复现 Linux 测量值。可选审计数据是 UTF-8 JSON 值；空字节表示缺失。成功确认断言后端能强制执行已准入策略；不支持的要求在工作负载启动前失败。两条路径都保留代次、会话、资源、身份和预置证据检查。

## 边界传输建立

`SandboxTransport` 属于此后端的运行时描述符。核心仅保留运维注册的端点；`DelegatedLaunch` 将这些端点转换为后端的传输，而不改变其序列化表示。

supervisor 为每个运行时描述符选择一个 `BoundaryTransport`，并使用其克隆进行策略发现和附加。它拨号描述符的端点，使用后端提供的 `BoundaryTransportConnector` 建立原始流，或者对于 `SandboxTransport::RuntimeInitiatedTcp`，接受沙箱运行时打开的连接。来自任何来源的每条流在承载请求之前都完成相同的固定 TLS 客户端握手；共享客户端还保留会话 bearer 认证、代次检查和重连处理。

对于运行时发起的连接，supervisor 在排队之前认证每个已接受的套接字。并发未认证握手在数量和时长上受限，最多两个已认证备用连接等待客户端，因此新连接无法挤占活跃通道。运行时保持一个备用连接打开，因此重连和凭据轮换从不等待新拨号。

新变体不改变现有编码。早于它们的二进制会拒绝运行时发起的描述符，而不是回退到其他模式。

`SandboxTransport::RuntimeInitiatedRouted` 通过一个监听器为多沙箱共享的 supervisor 的沙箱实例提供服务（`runtime_routed::RuntimeRouter`）。一个沙箱实例是稳定 Sandbox 资源的一个运行中代次。运行时首先发送固定前言：`OSRP`、版本 `1`、一个长度字节，以及一个 16 到 64 个小写字母、数字或连字符的不透明路由句柄。该句柄不可信；它只选择 supervisor 随后要求的、固定到某实例代次的 TLS 身份。未知、已移除和已替换的句柄在任何握手之前关闭。握手在全局和每路由上受限，已认证连接只填充其自身实例的备用队列。路由实例最多等待五分钟迎接首个连接（因为它在工作负载 Pod 调度之前注册），之后是通常的连接窗口。一个托管实例的启动材料和状态（`SandboxInstanceRegistration`、`SandboxInstanceStatusReport`）由 `openshell-supervisor-control` 定义，后者依赖本 crate 进行运行时描述符和路由校验。

运行时发起的传输支持 TCP、受保护的 Unix 套接字和主机 vsock 监听器。它们共享受限的 TLS 握手、已认证备用队列和相同的固定代次运行时身份检查。传统的 supervisor 拨号传输仍可用于现有描述符。