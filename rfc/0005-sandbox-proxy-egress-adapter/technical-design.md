# 技术设计附录

本附录包含主 RFC 背后的实现级设计细节。

## 现有运行时边界

`openshell-supervisor-network::run::run_networking` 是当前网络启动边界。它构建策略本地上下文、等待策略二进制符号链接解析、创建身份缓存、写入 TLS CA、构建 TLS 状态、解析推理路由、接入提供方凭据和令牌授权，并启动代理。supervisor 中间件工作扩展了此边界，增加了中间件注册表构建和重载行为。

这是一个有用的外层边界，但还不是代理适配器边界。代理仍需要内部的 `EgressIntent` 和 `EgressDecision` 边界，以便 CONNECT、转发 HTTP、本地路由和未来的原生 TCP 捕获不会重复策略和中继编排。第一个实现里程碑仅接入当前接口；后续里程碑将新适配器添加到同一契约。

## 共享数据边界

### EgressIntent

`EgressIntent` 是用户态尝试执行操作的规范化描述。

它应携带：

- 入口传输：CONNECT、转发 HTTP、透明 TCP、本地 HTTP、策略 DNS 或元数据回环；
- 请求的目标主机/端口或捕获的原始 IP/端口；
- 适配器/运行时收集的可选进程身份输入；
- 转发代理流量的可选首个 HTTP 请求；
- 可选的本地服务路由；
- 策略代次，以及对于策略 DNS/透明 TCP，独立的 DNS 映射代次和关联句柄。

适配器构建 intent。它们不应查询端点元数据、选择 TLS 模式或选择中继。

### EgressDecision

`EgressDecision` 是验证和中继代码消费的策略结果。

它应携带：

- 允许或拒绝；
- 用于所有策略派生字段的单一顶层策略代次；
- 确定性的匹配策略标识符；
- 策略是用户编写、提供方派生还是本地服务内部；
- 确定性的匹配端点标识符和端点元数据；
- 进程身份可用性及用于评估的任何身份字段；
- 目标和允许的 IP 约束；
- TLS 行为；
- 协议强制执行；
- 凭据注入计划；
- supervisor 中间件计划；
- 配置 HTTP 检查时创建固定每请求 L7 评估器所需的请求策略选择；
- 日志上下文和拒绝原因。

中继代码应读取此决策。它不应再次查询 OPA 以获取端点元数据、TLS 模式、允许的 IP、凭据行为、中间件选择或中继选择。长寿命 HTTP 中继仍通过 `RelayContext` 中携带的代次固定 L7 评估器评估每个请求；这是请求授权，而非端点重新物化。后续原生协议处理器使用相同模式，通过代次固定的协议评估器进行每命令或每查询决策。

## 协议强制执行

使用从端点策略派生的协议强制执行值：

| 策略协议 | 强制执行 | 中继行为 |
|-----------------|-------------|----------------|
| 省略 / `tcp` | 无 | L4 授权加字节中继，可选 HTTP 嗅探用于凭据注入 |
| `rest` | HTTP | HTTP 请求解析器加 REST 规则，可选请求体和 WebSocket 文本帧凭据重写 |
| `graphql` | HTTP | HTTP 请求解析器加 GraphQL-over-HTTP 规则 |
| `json-rpc` | HTTP | HTTP 请求解析器加有界 JSON-RPC-over-HTTP 方法检查 |
| `mcp` | HTTP | HTTP 请求解析器加有界 MCP Streamable HTTP 方法/工具检查 |
| `websocket` | HTTP | HTTP 升级策略后接 WebSocket 帧策略或 GraphQL-over-WebSocket 策略 |
| 未来 `redis`、`postgres`、`mysql`、... | 协议处理器 | 协议特定处理器拥有帧处理、中间件钩子和消息循环 |

`protocol: tcp` 实际上是默认 L4 模式。它不应运行原生协议处理器。避免使用术语"提供方"来表示处理器概念，因为提供方在 OpenShell 中已是第一类凭据和路由域。具体原生处理器在共享调度契约之后落地。

## 建议类型

确切的 Rust 形态可以演进，但边界应如下所示：

```rust
enum EgressTransport {
    Connect,
    ForwardHttp,
    TransparentTcp,
    PolicyDns,
    LocalHttp,
    MetadataLoopback,
}

struct EgressIntent {
    transport: EgressTransport,
    destination: RequestedDestination,
    process: ProcessIdentityEvidence,
    first_request: Option<ParsedHttpRequest>,
    local_route: Option<LocalRoute>,
    correlation: Option<ResolvedEndpointCorrelation>,
}

struct EgressDecision {
    policy_generation: PolicyGeneration,
    outcome: PolicyOutcome,
    matched_policy: Option<MatchedPolicy>,
    endpoint: Option<MatchedEndpoint>,
    process: EvaluatedProcessIdentity,
    request_processing: RequestProcessingPlan,
    log_context: EgressLogContext,
}

enum ProcessIdentityEvidence {
    Available(ProcessIdentity),
    Unavailable(ProcessIdentityUnavailableReason),
}

enum ProcessIdentityUnavailableReason {
    EndpointOnlyMode,
    DeclaredRuntimeMode(RuntimeMode),
    UnsupportedPlatform,
    LookupFailed,
}

struct EvaluatedProcessIdentity {
    evidence: ProcessIdentityEvidence,
    fields_used: Vec<ProcessIdentityField>,
}

struct MatchedPolicy {
    id: PolicyId,
    source: PolicySource,
}

enum PolicySource {
    User,
    ProviderDerived,
    LocalService,
}

struct MatchedEndpoint {
    id: EndpointId,
    destination: DestinationValidationPlan,
    tls: TlsPolicy,
    enforcement: ProtocolEnforcement,
}

struct DestinationValidationPlan {
    address_authorization: AddressAuthorization,
}

enum AddressAuthorization {
    DefaultPublicOnly,
    ExplicitAllowedIps(Vec<IpNet>),
    ExactDeclaredHost,
    ImplicitIpLiteral(IpAddr),
    TrustedGatewayAlias { expected_ip: IpAddr },
}

struct RequestProcessingPlan {
    middleware: SupervisorMiddlewarePlan,
    credentials: CredentialInjectionPlan,
}

enum ProtocolEnforcement {
    None,
    Http(HttpL7Config),
    ProtocolProcessor(ProtocolProcessorConfig),
}

enum HttpL7Protocol {
    Rest,
    Graphql,
    JsonRpc,
    Mcp,
    Websocket,
}

struct HttpL7Config {
    protocol: HttpL7Protocol,
    path: EndpointPathScope,
    allow_encoded_slash: bool,
    enforcement_mode: L7EnforcementMode,
    websocket_credential_rewrite: bool,
    request_body_credential_rewrite: bool,
    websocket_graphql_policy: bool,
    graphql_max_body_bytes: usize,
    json_rpc_max_body_bytes: usize,
    mcp_strict_tool_names: bool,
}

struct CredentialInjectionPlan {
    static_placeholders: StaticPlaceholderPlan,
    token_grant: Option<TokenGrantPlan>,
}

struct StaticPlaceholderPlan {
    http_target_query_header: bool,
    rest_request_body: bool,
    websocket_text_frames: bool,
}

struct TokenGrantPlan {
    provider_key: String,
    auth_style: TokenGrantAuthStyle,
    token_endpoint: String,
}

struct SupervisorMiddlewarePlan {
    stages: Vec<SupervisorMiddlewareStage>,
    min_body_limit: Option<usize>,
    registry_generation: PolicyGeneration,
}

struct SupervisorMiddlewareStage {
    policy_name: String,
    binding_id: String,
    operation: MiddlewareOperation,
    phase: MiddlewarePhase,
    order: i32,
    on_error: MiddlewareOnError,
    config: MiddlewareConfig,
}

enum MiddlewareOperation {
    HttpRequest,
    Future(String),
}

enum MiddlewarePhase {
    PreCredentials,
    Future(String),
}

struct RelayContext {
    decision: EgressDecision,
    request_policy: Option<PinnedRequestPolicy>,
    protocol_policy: Option<PinnedProtocolPolicy>,
    connector: UpstreamConnector,
    deadlines: RelayDeadlines,
    telemetry: RelayTelemetry,
}

struct ResolvedEndpointCorrelation {
    policy_generation: PolicyGeneration,
    mapping_generation: DnsMappingGeneration,
    mapping_id: DnsMappingId,
    synthetic_ip: IpAddr,
}

struct PinnedRequestPolicy {
    generation: PolicyGeneration,
    evaluator: TunnelPolicyEngine,
}

struct PinnedProtocolPolicy {
    generation: PolicyGeneration,
    evaluator: ProtocolPolicyEngine,
}
```

`UpstreamConnector` 是中继拥有的拨号边界。它封装已验证的目标，并允许中继或处理器仅在当前请求或协议策略允许后才打开上游连接。

`DestinationValidationPlan` 选择一种当前验证模式。所有模式均保留控制平面端口和云元数据阻止。默认和显式 IP 路径保留始终阻止的回环、链路本地和未指定地址检查。`ImplicitIpLiteral` 仅为显式声明的 IP 主机合成。`TrustedGatewayAlias` 可以接受运行时发现的单个网关 IP，但不会成为通用的私有地址豁免。

`policy_generation`、可选的固定请求/协议评估器、端点元数据和中间件选择必须描述同一个策略快照。授权断言每个子物化都使用了该代次。如果代次在中继启动前发生变化，适配器收到过期策略拒绝，而非混合决策。

## 进程身份可用性

进程身份是证据，不是查找失败时伪造的字符串。嵌入式模式通常填充二进制、PID、祖先、命令行路径和二进制哈希数据。当需要二进制身份时，`LookupFailed` 和 `UnsupportedPlatform` 保持拒绝。显式配置的仅端点运行时记录 `Unavailable(EndpointOnlyMode)` 并保持当前仅端点策略行为。本 RFC 不会静默地将一种状态转换为另一种，也不会改变仅端点信任契约。

未来有意缺少本地身份的独立或边车运行时使用 `DeclaredRuntimeMode`，而非 `EndpointOnlyMode`，并向策略验证声明该能力。运行时契约必须将二进制/路径谓词定义为不可用，并在流量开始前拒绝不兼容策略，除非后续接受的策略设计指定了不同的失败关闭规则。

决策记录身份可用性和使用的字段，以便 OCSF 日志和拒绝响应区分二进制策略拒绝、身份查找失败和有意仅端点评估。测试必须证明，在需要身份时，空的合成 `exec.path` 无法满足二进制作用域规则。添加新的无身份部署模式，或更改二进制谓词在仅端点模式下的行为，需要后续运行时阶段的能力工作，不能偷偷塞入兼容性重构。

## 当前所有者与建议清理

| 当前所有者 | 当前职责 | 建议清理 |
|---------------|------------------------|------------------|
| `openshell-sandbox` | 编排器、策略轮询循环、拒绝/活动通道、元数据回环启动、仅网络生命周期 | 保留为编排；避免嵌入每入口代理策略决策 |
| `openshell-supervisor-network::run` | 网络启动和句柄 | 成为嵌入式和未来独立模式的稳定运行时 API |
| `openshell-supervisor-network::proxy` | CONNECT、转发 HTTP、本地路由调度、目标验证、拒绝渲染 | 拆分为适配器、授权、目标、中继选择和适配器响应渲染 |
| `openshell-supervisor-network::opa` | 策略引擎和 Rego 查询 | 返回确定性的 `EgressDecision` 数据，而非单独的策略和端点查找 |
| `openshell-supervisor-network::l7` | REST、GraphQL、JSON-RPC、MCP、WebSocket、推理辅助、TLS、令牌授权 | 保留为共享中继边界后的协议/中继实现 |
| `openshell-supervisor-network::policy_local` | `policy.local` 状态和路由 | 建模为具有显式限制和提议/等待行为的本地适配器 |
| `openshell-supervisor-middleware` | 中间件注册表、内置项、服务契约和链执行 | 视为由 `EgressDecision` 选择的中继钩子依赖，而非适配器特定策略逻辑 |
| `openshell-supervisor-process::netns` | nftables 绕过规则和命名空间辅助 | 保留为绕过强制执行的所有者；协调未来捕获规则与网络代理映射 |
| `openshell-supervisor-process::bypass_monitor` | nftables LOG 解析和 OCSF 绕过遥测 | 保留为绕过违规的遥测生产者 |
| `openshell-core::secrets` 和提供方凭据状态 | 静态占位符源和动态凭据元数据 | 提供凭据注入计划；不要将机密泄漏到决策日志 |

## 策略 DNS 和已解析 TCP 状态

策略 DNS 是查询驱动的，而非静态 `/etc/hosts` 快照。

1. 策略加载注册符合条件的原生 TCP 端点名称。
2. 用户态执行 DNS 查找。
3. 策略 DNS 检查规范化名称是否匹配一个端点，该端点的传输和协议契约在当前策略代次中启用通过策略 DNS 的原生 TCP。
4. 不符合条件的名称收到本地策略拒绝 DNS 响应，不进行上游查询。
5. 策略 DNS 通过受信任的上游 DNS 解析符合条件的名称，并通过端点元数据和 SSRF 控制过滤每个应答。
6. 适配器分配 supervisor 拥有的合成 IP，并创建活动映射，包含合成 IP、规范化名称、端点标识符、允许端口、已验证的真实地址、策略代次、独立 DNS 映射代次、不透明映射 ID 和过期时间。
7. 适配器在将合成 IP 以有界 TTL 返回给用户态之前，原子地发布映射和捕获状态。
8. 用户态稍后调用 `connect(synthetic_ip:port)`。
9. 透明 TCP 恢复合成原始目标，并要求未过期的精确映射，其允许端口包含请求端口。
10. 正常出口授权和中继选择针对与映射契约一致的策略代次运行。
11. 连接器仅拨号该映射中固定的真实地址。它不会在连接时独立重新解析名称。

已解析端点存储是由策略符合条件的查找产生的活动状态，由透明 TCP 连接消费。策略代次和 DNS 映射代次是独立的值：DNS 刷新可以替换映射而无需策略重载，而策略重载可以使端点契约不再当前的映射失效。没有映射、映射过期或端点/端口不匹配的捕获连接失败关闭。无关的裸 IP 连接不能仅因其目标为映射存储中存在的真实 IP 而继承策略 DNS 授权。合成 IP 是关联句柄，不是上游目标，绝不能直接路由或在过期应答仍可能引用先前映射时重新分配。

映射是沙箱作用域而非进程作用域。进程身份在捕获的 TCP 连接被授权时独立查找和评估。它不用于连接 DNS 和 TCP，因为名称解析可能被缓存、委托给解析器辅助程序或被不同进程消费，且一个进程解析的多个名称可能共享真实地址和端口。

## nftables 边界

当前 main 使用 nftables 而非 iptables 进行沙箱网络绕过强制执行。安装的 `inet` 表接受到沙箱代理、回环和已建立/相关流的流量，然后拒绝并可选记录其他 TCP/UDP 流量。绕过监视器读取这些日志行并发出 OCSF 网络和检测事件。

透明 TCP 捕获在后续功能阶段基于同一 nftables 基底构建：

- 捕获规则在通用绕过拒绝规则之前运行；
- 捕获规则作用域为活动合成 IP 和允许端口映射；
- 从适配器角度看，映射和捕获规则更新是原子的；
- 直接外部 DNS 仍被阻止；策略 DNS 是沙箱解析器，DNS-over-HTTPS 仍是普通策略控制的 HTTPS 出口；
- 拒绝/日志规则仍是不匹配 TCP/UDP 出口的回退；
- VM 或 Podman 驱动 nftables 规则是基础设施 NAT/隔离，不是代理策略强制执行点。

初始 CONNECT/转发重构不改变已安装的表。本节定义了共享适配器和决策边界在透明捕获落地时必须支持的消费者契约。

## 端点选择和 OPA

目前匹配的策略名称、L7 候选、首个 TLS/`allowed_ips` 端点和精确声明主机信号通过独立规则选择。OPA/Rego 应通过一个确定性授权结果返回策略和端点元数据。它不应让这些字段描述不同的匹配。

两种可接受的方法：

- 在加载或合并时拒绝重叠的端点元数据。
- 定义单一确定性优先级键，并将其用于策略名称和端点元数据。

当所选端点需要元数据时，端点元数据查询失败应失败关闭。它们不应静默降级为 L4 行为。顶层决策代次还必须匹配每个策略派生字段；物化期间重载产生过期决策，而非混合代次。

此语义切换与引入 Rust 类型是分开的。新查询首先在遗留查询旁以影子模式运行，通过内部遥测记录审计安全的不匹配，并保留遗留强制执行。在优先级规则被接受且不匹配情况被理解后，专用变更切换权威结果，并保留遗留评估器足够长时间以便立即回滚。

提供方派生策略使用保留的规则名称命名空间。网关和沙箱同步应阻止用户编写的 `_provider_*` 规则，`policy.local` 提议界面不应将提供方派生规则暴露为可编辑用户策略。`EgressDecision` 仍应标识提供方派生匹配以用于日志和调试。

## 凭据注入边界

凭据注入属于策略允许和 supervisor 中间件之后、上游写入之前的 HTTP/WebSocket 中继。

1. 授权选择端点并计算凭据注入计划。
2. Supervisor 中间件在凭据可见之前对已准入请求运行。
3. 如果中间件替换了正文，中继重新解析依赖正文的协议输入并重新评估请求策略。
4. HTTP 中继仅在其仍拥有端点强制执行模式下允许的请求时解析凭据。
5. 静态占位符值被解析并从日志中脱敏。
6. 端点绑定令牌授权获取或重用动态访问令牌。
7. 最终上游请求或 WebSocket 帧在写入前立即重写。

仅 L4 HTTP 和 HTTP 检查路径均可注入凭据。区别在于重写前是否评估 REST、GraphQL 或 WebSocket 策略。

凭据重写槽应显式：

- HTTP 系列流量的请求目标、查询值和头部；
- 仅当启用 `request_body_credential_rewrite` 时的 REST 请求体；
- 仅当启用 `websocket_credential_rewrite` 时的客户端到服务器 WebSocket 文本帧；
- 当 GraphQL-over-WebSocket 连接/控制消息携带在文本帧中且端点启用 WebSocket 重写路径时；
- 端点绑定提供方凭据的令牌授权头部。

请求体重写仅限 REST。它应缓冲有界 UTF-8 文本正文，包括 JSON、form-url-encoded 和 `text/*`，重新计算 `Content-Length`，保留不包含保留凭据标记的不支持