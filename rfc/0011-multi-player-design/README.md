---
authors:
  - "@derekwaynecarr"
state: draft
links:
  - https://github.com/NVIDIA/OpenShell/issues/1977
---

# RFC 0011 - 多用户支持

## 摘要

本 RFC 提议为 OpenShell 增加多用户支持。目前，沙箱和 Provider 是网关全局的，没有所有权跟踪，用户之间也没有隔离。本提案引入工作区作为硬隔离边界，扩展角色模型（平台管理员、工作区管理员、用户），实现工作区范围的访问控制、按工作区的配额执行以及审计追踪增强。沙箱 Supervisor 仍是一种独立的主体类型，具有沙箱范围的身份验证，与用户角色模型分离。`default` 工作区为单用户部署保持向后兼容。

## 动机

OpenShell 目前是单用户体验。每个已认证用户都能看到所有沙箱和所有 Provider。没有资源所有权、租户隔离或委托管理的概念。这阻碍了多种采用场景：

- **企业团队**无法共享网关而不看到彼此的沙箱、凭据和活动。无法限定可见范围或执行按团队的资源限制。

- **CI/CD 和 Agent 编排**工作流需要具有工作区范围访问权限的机器身份。目前唯一的选择是全权限的 OIDC Token 或 mTLS 证书，除管理员/用户外没有更细的角色粒度。工作区为这些身份提供了范围边界。

- **合规和事件响应**团队需要审计追踪，将每个沙箱和控制平面操作归因到特定主体。现有的 OCSF 基础设施记录沙箱级事件，但没有一致地标记创建主体。

- **成本归因**在没有所有权元数据的情况下无法实现。运维人员无法回答"哪个团队消耗了多少活跃沙箱"。

现有代码库提供了基础：OIDC 身份验证、主体模型（User/Sandbox/Anon）、两级 RBAC、OCSF 事件基础设施以及 `ObjectMeta` 上的标签。缺口在于上层的隔离、所有权和治理层。

保持当前设计不变会将 OpenShell 限制在单运维、单团队部署，制约采用并迫使组织为每个团队运行一个网关。

## 非目标

- **跨网关联邦。** 本 RFC 将多用户范围限定在单个网关。多网关联邦（例如将用户路由到区域网关）是独立的问题。
- **细粒度 ABAC 或策略语言。** 角色模型使用粗粒度角色加工作区范围，而非基于属性的访问控制或 OPA/Rego 之类的策略 DSL 来做授权决策。
- **用户管理 UI/仪表盘。** 本 RFC 涵盖 API 和数据模型。管理 UI 是后续工作。
- **计费集成。** 资源上的主体归因支持成本归因；与计费系统的集成不在范围内。
- **沙箱间网络隔离。** 容器/Pod 级别的工作区之间网络隔离不在范围内；本 RFC 仅涉及控制平面隔离。
- **多 Provider OIDC。** 本 RFC 假设配置单个 OIDC Provider。支持多个 OIDC Provider（例如同时支持企业 SSO 用于人类用户和 GitHub Actions 用于 CI/CD）需要基于签发者的 Token 路由、成员记录中带 Provider 限定的主体格式以及认证器链变更。这些是有价值的扩展，但不是核心工作区和角色模型所必需的。后续 RFC 可以在不改变工作区或成员抽象的情况下增加多 Provider 支持。

## 提案

### 系统角色

角色模型从当前的两级（管理员/用户）扩展为三种用户角色：

| 角色 | 描述 |
|------|------|
| **平台管理员** | 运行时角色，对所有工作区具有完全可见性。创建工作区、分配工作区管理员、设置网关级默认策略。 |
| **工作区管理员** | 在单个工作区内管理用户、Provider、策略和配额。不能更改网关基础设施或访问其他工作区。 |
| **用户** | 在分配的工作区内创建沙箱并访问所有沙箱。使用这些工作区中可用的凭据。通过工作区成员记录分配给人类和机器身份。 |

### 沙箱 Supervisor

沙箱 Supervisor 不是用户角色——它是一种独立的主体类型，具有自己的身份验证和授权路径。用户角色是 `Principal::User` 的属性，通过 OIDC 声明或工作区成员记录解析。沙箱 Supervisor 通过网关签发的 JWT 以 `Principal::Sandbox` 身份认证，其 subject 绑定到单个沙箱 UUID。

Supervisor 的作用域限定为单个沙箱，类似于 Kubernetes kubelet 身份。它通过网关签发的 JWT 或引导证书进行认证，并被限制为仅能操作自己沙箱的 RPC。授权由路由器层的静态方法允许列表（`is_sandbox_callable`）和逐处理器的范围守卫（`ensure_sandbox_principal_scope`）执行，后者验证 JWT 的沙箱 UUID 与请求目标匹配。Supervisor 从不经过 RBAC 角色检查或工作区成员查找。

Supervisor 从其首次设置轮询返回的 `GetSandboxConfigResponse.workspace` 字段获知其工作区。它缓存该值并在后续工作区范围的 RPC（策略同步、策略分析、草稿策略查询）中传递。这避免了服务端对沙箱主体的特殊处理，同时保持 Supervisor 的 JWT 限定在单个沙箱 UUID。

### 角色到 RPC 的访问矩阵

访问按域分组。在每个域内，访问级别（无、读、读写）适用于该组中的所有 RPC，除非另有说明。所有用户角色都限定在其工作区内，平台管理员除外，其跨工作区操作。沙箱 Supervisor 列仅为完整性而包含——它使用独立的身份验证和授权路径（见上文沙箱 Supervisor 部分）。

| 域 | 平台管理员 | 工作区管理员 | 用户 | 沙箱 Supervisor |
|--------|---------------|-----------------|------|--------------------|
| 工作区生命周期（`Create`、`Get`、`List`、`Delete`） | 读写 | 读（自己的） | 读（自己的） | 无 |
| 工作区成员（`Add`、`Remove`、`List`） | 读写 | 读写（自己的工作区，不能分配管理员） | 读（自己的工作区） | 无 |
| 沙箱生命周期（`Create`、`Get`、`List`、`Delete`） | 读写 | 读写（自己的工作区） | 读写（自己的工作区） | 无 |
| 沙箱数据平面（`Exec`、`ForwardTcp`、`CreateSshSession`） | 完全 | 完全（自己的工作区） | 完全（自己的工作区） | 无 |
| 沙箱可观测性（`GetSandboxLogs`、`ListSandboxPolicies`、`GetSandboxPolicyStatus`） | 读 | 读（自己的工作区） | 读（自己的工作区） | 无 |
| Provider 管理（`Create`、`Get`、`List`、`Update`、`Delete`） | 读写 | 读写（自己的工作区） | 读（无凭据） | 无 |
| Provider 挂载（`Attach`、`Detach`、`ListSandboxProviders`） | 读写 | 读写（自己的工作区） | 读写（自己的工作区） | 无 |
| 服务（`Expose`、`Get`、`List`、`Delete`） | 读写 | 读写（自己的工作区） | 读写（自己的工作区） | 无 |
| 网关配置读取（`GetGatewayConfig`） | 读 | 读 | 读 | 无 |
| 网关配置写入（`UpdateConfig` 带 `global: true`） | 读写 | 无 | 无 | 无 |
| 沙箱配置和策略（`GetSandboxConfig`、非全局 `UpdateConfig`） | 读写 | 读写（自己的工作区） | 读（自己的工作区） | 读写（自己的沙箱，仅策略同步） |
| 策略草稿检查（`GetDraftPolicy`、`GetDraftHistory`） | 读 | 读（自己的工作区） | 读（自己的工作区） | 仅自己沙箱的 `GetDraftPolicy` |
| 策略草稿决策（`Approve`、`Reject`、`Edit`、`Undo`、`Clear`） | 读写 | 读写（自己的工作区） | 无 | 无 |
| 策略分析提交（`SubmitPolicyAnalysis`） | 无 | 无 | 无 | 自己的沙箱 |
| Supervisor 路径（`ConnectSupervisor`、`RelayStream`、`IssueSandboxToken`、`RefreshSandboxToken`、`GetSandboxProviderEnvironment`、`PushSandboxLogs`、`ReportPolicyStatus`） | 无 | 无 | 无 | 自己的沙箱 |

**控制平面审计日志。** 每个变更性 gRPC 调用都会发出 OCSF `ApiActivity` 事件，记录主体、操作、目标资源和时间戳。这些事件通过现有 OCSF 基础设施发出，并导出为结构化 JSONL 供外部系统消费（见下文审计追踪部分）。

### 工作区

工作区是一等资源，也是硬隔离边界。工作区内的沙箱、Provider 和策略对其他工作区不可见。每个资源恰好属于一个工作区。`default` 工作区用于单用户向后兼容。工作区创建仅限管理员：平台管理员创建工作区并分配工作区管理员。自助创建工作区可以后续作为网关配置选项添加。

`Workspace` 资源使用标准 `ObjectMeta`（其自身 `ObjectMeta` 中的 `workspace` 字段未使用，遵循与 Kubernetes Namespace 对象相同的约定）。工作区级配置——配额限制、策略覆盖和工作区管理员角色绑定——是 Workspace 资源上的属性。网关暴露 `CreateWorkspace`、`GetWorkspace`、`ListWorkspaces` 和 `DeleteWorkspace` RPC。创建和删除需要平台管理员；获取和列表返回通过成员关系可见的工作区，平台管理员可以看到所有工作区。
沙箱和 Provider 创建操作验证引用的工作区存在，拒绝未知的工作区值。

`DeleteWorkspace` 使用受 Kubernetes namespace 删除启发的两阶段优雅删除协议。工作区有两个生命周期阶段：**Active**（默认）和 **Terminating**。

1. **标记 Terminating。** 调用 `DeleteWorkspace` 时，通过 CAS 写入原子地将工作区标记为 Terminating，设置 `ObjectMeta.deletion_timestamp_ms` 和 `WorkspaceStatus.phase = TERMINATING`。此操作不可逆——一旦进入 Terminating，工作区不能返回 Active。

2. **拒绝新资源。** `resolve_workspace` 返回工作区的终止状态。创建路径处理器（`CreateSandbox`、`CreateProvider`、`ExposeService`、`SetInferenceRoute`、`AddWorkspaceMember` 等）调用 `ensure_active()` 并以 `FAILED_PRECONDITION` 拒绝在 Terminating 工作区上的操作。读取和删除操作继续工作，以便管理员清理。

3. **扫描阻塞项。** 标记 Terminating 后，处理器扫描剩余的工作区范围资源。如果存在任何资源，调用返回 `FAILED_PRECONDITION` 并列出它们。工作区保持 Terminating；管理员清理资源后重试。

4. **完成删除。** 如果没有阻塞资源剩余，清理成员记录并删除工作区。

Terminating 阶段消除了原始设计中的 TOCTOU 竞态：没有它，并发的 `CreateSandbox` 可能在阻塞项扫描和删除之间提交新资源，使沙箱成为孤儿。通过先标记 Terminating，并发创建在扫描窗口期间被拒绝。

重入是幂等的：对已处于 Terminating 的工作区调用 `DeleteWorkspace` 会跳过 CAS 写入，直接进入阻塞项扫描。

`default` 工作区不能被删除。如果 `default` 工作区无法创建或验证，网关启动失败——这是致命错误，不是尽力而为的操作。

工作区成员上限为每个工作区 1000 名成员。`AddWorkspaceMember` 以资源耗尽错误拒绝超过此限制的添加。此上限限制了工作区删除期间的清理成本，并确保用于清理的成员列表是穷尽的。

`ObjectMeta` 新增 `workspace` 字段，按名称引用 Workspace。在工作区内，组织分组（团队、项目、成本中心）使用现有标签系统加众所周知的键约定（例如 `openshell.dev/team=infra`、`openshell.dev/project=alpha`），而非额外的专用字段。这样：

- 提供清晰的安全边界（工作区），而不过度建模组织层级。
- 允许通过标签在工作区内进行多个重叠分组。
- 保持 proto 表面最小：`workspace` 和 `deletion_timestamp_ms` 是 `ObjectMeta` 上仅有的新字段。`deletion_timestamp_ms` 字段在优雅删除开始时设置（0 = 未删除），并作为终止状态的权威来源。

#### 工作区用例

- **团队内的凭据隔离。** 每个用户在共享网关上获得自己的工作区，将其 API 密钥（例如每用户的 Claude 或 Codex 密钥）与其他用户隔离。这消除了每用户单独网关的需求，同时保持凭据隔离。

- **共享编码会话。** 所有工作区成员可以访问工作区中的任何沙箱，支持结对编程和协作调试，无需额外的访问授权。

- **CI/CD 和自动化。** 工作区将沙箱生命周期限定到特定流水线或项目。机器工作负载通过配置的 OIDC Provider 认证，并以适当角色添加为工作区成员。

- **Agent 框架集成。** 单个 OpenShell 网关可以划分为离散的工作区，使多个 Agent 框架实例（例如 OpenClaw）可以从同一网关获取沙箱并具有适当的隔离。这消除了 Agent 框架实例与 OpenShell 网关实例之间一对一关联的要求。

#### 未来工作：服务端发起的工作区清理

当前的优雅删除要求管理员在删除完成前手动移除所有工作区范围资源。未来的增强可以添加后台 GC 控制器，在工作区进入 Terminating 阶段时自动删除资源：

- 管理员调用一次 `DeleteWorkspace`；服务端异步处理清理（先沙箱，然后 Provider、Profile、服务等）。
- `DeleteWorkspace` 可以立即返回并指示清理正在进行，或阻塞直到完成（可配置超时）。
- GC 控制器会遵守资源特定的顺序约束（例如在删除沙箱前分离 Provider）并重试瞬时故障。

这推迟到 Phase 2。目前，管理员必须在删除完成前手动清理资源。

### 所有权和访问控制

访问控制基于工作区成员关系。主体归因（谁创建或修改了资源）由控制平面审计日志处理，而非资源本身的字段。

#### 角色分配

角色分为两类：

- **全局角色**（平台管理员）通过身份提供者中的 OIDC 声明在外部分配（例如 JWT 中的 `openshell-platform-admin` 角色）。此角色是主体的属性，不属于任何工作区，授予跨工作区访问权限。网关在请求时从已认证的 Token 中评估它。

- **工作区范围角色**（工作区管理员、用户）通过存储在网关持久对象存储中的工作区成员记录在内部分配。这些角色是主体与特定工作区关系的属性。

授权层结合两者：网关首先从 JWT 评估全局角色，然后从目标工作区的成员记录解析工作区范围角色。如果主体具有平台管理员全局角色或具有足够角色的工作区成员关系，则对工作区范围 RPC 的请求被授权。

工作区成员关系通过三个 RPC 管理：

- `AddWorkspaceMember(workspace, principal_subject, role)`——平台管理员可以分配任何工作区范围角色。工作区管理员可以将用户添加到自己的工作区，但不能分配工作区管理员角色。
- `RemoveWorkspaceMember(workspace, principal_subject)`——相同的访问模式。
- `ListWorkspaceMembers(workspace)`——平台管理员可以列出任何工作区；工作区管理员和用户可以列出自己的工作区。

主体 subject 是配置的身份提供者中的 OIDC `sub` 声明。网关不维护用户目录——成员关系引用在认证时解析的 OIDC subject。成员记录持久化在持久对象存储中，按工作区和主体 subject 双重索引以实现高效查找。

主体对任何工作区范围 RPC 的请求，如果不是目标工作区的成员，则被拒绝。平台管理员绕过成员检查——其角色按定义授予跨工作区访问权限。

#### 资源级访问

在工作区内，访问因资源类型而异：

- **沙箱。** 所有工作区成员可以列出、获取、exec 进入并访问其工作区中的任何沙箱。凭据隔离发生在工作区边界——在工作区内，所有成员共享相同的信任域和相同的 Provider 凭据，因此按所有者限制沙箱访问没有安全收益。平台管理员可以使用列表 RPC 上的 `all_workspaces` `WorkspaceSelector` 变体跨工作区列出（见下文跨工作区列表操作）。

- **Provider。** 用户可以在其工作区内按名称列出和引用 Provider，但不能创建、更新或删除它们，也不能看到原始凭据材料。工作区管理员在其工作区内管理 Provider 生命周期。`ListProviders` 限定在调用者的工作区。

- **服务。** 服务是沙箱的子资源，以沙箱名称为键。它们在其 `ObjectMeta` 中携带父沙箱的工作区以保持一致的过滤，但工作区始终从沙箱继承，从不独立设置。所有工作区成员可以在其工作区中的任何沙箱上暴露、列出和删除服务。

- **Provider Profile。** Provider Profile 是描述 Provider 类型需要什么（凭据、端点、文件系统路径）的类型定义。Profile 具有两级范围：平台范围 Profile 由平台管理员管理，通过合并的工作区范围目录对工作区成员可见；直接查询平台范围（`--global`）需要平台管理员。工作区范围 Profile 由工作区管理员管理，仅在其工作区内可见。
  相同的 Profile ID 可以同时存在于平台和工作区范围——工作区 Profile 在工作区范围操作中遮蔽平台 Profile，当不存在工作区覆盖时以平台 Profile 作为回退。这允许平台管理员发布组织范围的默认值，工作区管理员可以按工作区自定义而不影响其他工作区。

  每个 Profile 在列表/获取响应中携带两个元数据字段：**source**（来源——`builtin`、`user` 或 `interceptor/{name}`）和 **scope**（可见性——`platform`、`workspace`，或对于 builtin 和 interceptor 等非用户范围来源为空）。这些字段由网关设置，在导入载荷中被忽略。内置 Profile（claude-code、github、nvidia 等）在所有范围中可见。

  `ListProviderProfiles` 返回在请求上下文中