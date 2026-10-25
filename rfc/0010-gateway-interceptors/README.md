---
authors:
  - "@anewberry"
state: accepted
links:
  - https://github.com/NVIDIA/OpenShell/issues/1919
---

# RFC 0010 - 网关拦截器

## 摘要

运维人员和外部集成方需要一种灵活的方式来定制网关 API 行为，以满足自身需求——例如强制租户隔离、配额、命名规范或策略权威。目前，任何此类定制都必须硬编码到网关处理器中或推入驱动，这混淆了职责，也无法适应部署特定的需求。

本 RFC 提出一个一等公民的扩展系统，让外部服务在明确定义的阶段观察、修改、验证、拒绝或审计网关操作。我们称之为**网关拦截器**。

网关拦截器和驱动服务于不同的扩展需求。网关拦截器角色在网关操作周围添加业务逻辑。驱动替换或提供平台功能的实现，例如沙箱如何在 Docker、Kubernetes 或 VM 上被供应。

本 RFC 将网关拦截器限定于网关 API 操作。网关拦截器可以在明确定义的阶段观察、修改、验证、拒绝或审计网关操作。未来的 RFC 可能将网关拦截器扩展到其他网关功能，例如事件驱动或工作流行为，但这不在首次实现的范围内。

驱动继续拥有平台实现——网关功能实际如何提供。网关拦截器拥有资源写入的网关级治理：租户隔离、配额、命名、策略权威和驱动配置限制。

网关数据库仍是记录系统。网关拦截器在网关操作周围添加治理；它们不替换网关拥有的状态。

## 动机

在自己的环境中运行 OpenShell 的运维人员需要将部署特定的规则应用于核心 OpenShell 未编码的网关操作。示例包括：

- 从外部事实来源同步策略和提供者。
- 强制一个系统范围的沙箱策略并拒绝自定义沙箱策略。
- 在接受策略写入之前对照外部权威进行验证。
- 将驱动配置载荷限制为已批准的 Schema 或固定值。
- 限制每个用户最多运行多少个沙箱。

这些不是核心 OpenShell 语义。它们因部署而异，且集合随时间变化，因此不适合作为一组固定的内置选项。

OpenShell 已经扩展了其两个子系统。驱动（RFC 0001）为平台和基础设施层提供实现。沙箱出口中间件（RFC 0009）在 supervisor 代理中运行，管控 agent 的出站请求可以携带什么。网关拦截器为网关控制平面完善了这一模式：API 操作本身的扩展点，租户配额、策略权威和命名等部署特定规则属于此处。

其中一些可能随时间作为内置网关默认值发布。网关拦截器服务不替换这一点——它们让部署在其规则不同时扩展或覆盖内置默认值，而无需等待上游变更。

没有专门的机制，运维人员只能将这些规则作为网关 Fork 或本地补丁来维护。

## 非目标

- 替换计算驱动或添加第二个计算供应接口。
- 让网关拦截器绕过网关认证、授权、策略安全验证或驱动 Schema 验证。
- 将沙箱运行时强制执行移出沙箱 supervisor 和代理。
- 替换网关数据库作为记录系统。
- 为配额、名称策略、策略包或驱动配置策略添加新的一等公民网关资源类型。

## 提案

添加一个网关拦截器框架，具有明确的阶段、RPC 方法选择器、确定性排序、有界执行、审计日志和保守的失败行为。

网关拦截器不替换网关功能。它们在网关操作周围添加治理和业务逻辑：默认值、验证、拒绝和审计。替换核心功能的实现方式仍是驱动和其他提供者式接口的角色。

设计保持两个边界不变：

- 网关数据库仍是网关拥有状态的记录系统。
- 现有网关和驱动验证仍在网关拦截器修改之后运行。

### 网关拦截器

网关拦截器在网关 API 操作期间运行，例如创建沙箱、导入提供者配置、更新策略或应用沙箱配置。它只能在 `modify_operation` 中修改网关准备的操作。它可以在 `modify_operation` 或 `validate` 中拒绝。它可以在所有阶段附加警告和审计注解。

网关拦截器服务暴露一个或多个绑定。绑定是服务声明的规则，将服务映射到阶段、网关 RPC 方法和选择器。网关使用绑定来决定何时调用服务。

公共 gRPC 服务和方法标识 API 操作。v1 选择器词汇使用完全限定的 RPC 名称，例如 `openshell.v1.OpenShell/CreateSandbox`。这使绑定配置与运维人员已经熟悉的公共 API 保持关联，并避免另一个兼容性表面。

所有可拦截的网关 API RPC 都经过相同的标准阶段流水线。网关拒绝引用运行中网关版本未知 RPC 的网关拦截器绑定，除非 RPC 选择器为空以匹配所有可拦截 RPC。

网关拦截器应适用于所有相关网关 RPC，而不是手工维护的子集。新的网关 RPC 应通过使用共享网关 API 执行路径进入网关拦截器流水线，而不是通过添加每 RPC 的网关拦截器 Hook 或更新单独的允许列表。RPC 只有在不属于本 RFC 范围内的网关 API 操作时才能选择退出，例如低级流式传输或 supervisor 内部调用，且退出应在代码审查中明确。

这让 OpenShell 在其已支持的网关操作周围添加部署特定的业务逻辑，同时保持运行时读取的本地性和确定性。

### 操作阶段

操作阶段是有序的。后续阶段可以看到先前阶段的结果。所有可拦截的网关 API RPC 使用相同的阶段和相同的顺序，因此网关拦截器作者和运维人员不需要每 RPC 的阶段规则。

| 阶段 | 允许修改 | 用途 | 示例 |
|---|---:|---|---|
| `modify_operation` | 是 | 在网关准备操作输入后应用部署默认值或受控变更。 | 规范化标签、标记默认沙箱策略、选择提供者配置，或将资源限制钳制到部署默认值。 |
| `validate` | 否 | 在持久化、供应或其他副作用之前强制执行部署特定规则。 | 强制租户配额、拒绝允许互联网出口的策略更新，或对照已批准的 Schema 验证驱动配置。 |
| `post_commit` | 否 | 在成功持久化或供应后发出审计或通知外部系统。 | 发送审计记录、通知库存系统，或在成功写入后触发协调任务。 |

网关在网关拦截器执行之前拒绝格式错误的原始 API 请求。网关拦截器操作有效的网关操作，而非低级请求解码或形状修复。

网关不变量在修改之后运行，因此网关拦截器无法在系统中留下无效对象。操作特定的内置验证（包括适用的驱动验证）仍是网关拥有的执行路径的一部分，因此驱动仍是驱动拥有 Schema 的权威。

### 网关拦截器请求契约

每个网关拦截器是一个注册的服务实例，实现 `GatewayInterceptor` gRPC 服务：

```proto
service GatewayInterceptor {
  rpc Describe(DescribeRequest) returns (InterceptorManifest);
  rpc Evaluate(InterceptorEvaluation) returns (InterceptorResult);
}

// Empty today so Describe can grow additively later.
message DescribeRequest {}

enum GatewayInterceptorPhase {
  GATEWAY_INTERCEPTOR_PHASE_UNSPECIFIED = 0;
  GATEWAY_INTERCEPTOR_PHASE_MODIFY_OPERATION = 1;
  GATEWAY_INTERCEPTOR_PHASE_VALIDATE = 2;
  GATEWAY_INTERCEPTOR_PHASE_POST_COMMIT = 3;
}
```

网关拦截器请求必须稳定，并与公共网关 API 关联，而非 Rust 处理器内部实现。

```proto
message InterceptorEvaluation {
  // Gateway-configured gateway interceptor service name.
  string interceptor_name = 1;
  // Service-declared binding that selected this evaluation.
  string binding_id = 2;
  // Fully qualified public gateway gRPC service name.
  string rpc_service = 3;
  // Public gateway RPC method name.
  string rpc_method = 4;

  // Authenticated gateway principal for the API request.
  string principal = 5;
  // Additional gateway-provided request context.
  map<string, string> context = 6;

  oneof phase {
    // Evaluation that may modify the prepared gateway operation.
    ModifyOperationEvaluation modify_operation = 7;
    // Evaluation that may reject, but not mutate, the prepared operation.
    ValidateEvaluation validate = 8;
    // Evaluation after the gateway operation has committed.
    PostCommitEvaluation post_commit = 9;
  }
}

message ModifyOperationEvaluation {
  // Gateway-prepared operation the gateway proposes to execute.
  google.protobuf.Struct proposed_operation = 1;
  // Gateway-owned resource state loaded before applying the proposed operation.
  google.protobuf.Struct current_state = 2;
}

message ValidateEvaluation {
  // Gateway-prepared operation after defaulting and modification.
  google.protobuf.Struct proposed_operation = 1;
  // Gateway-owned resource state loaded before applying the proposed operation.
  google.protobuf.Struct current_state = 2;
}

message PostCommitEvaluation {
  // Gateway operation that was committed.
  google.protobuf.Struct proposed_operation = 1;
  // Gateway-owned resource state loaded before applying the committed operation.
  google.protobuf.Struct current_state = 2;
}
```

`rpc_service` 和 `rpc_method` 字段是绑定使用的完全限定 RPC 选择器的拆分形式。例如，`openshell.v1.OpenShell/CreateSandbox` 变为 `rpc_service = "openshell.v1.OpenShell"` 和 `rpc_method = "CreateSandbox"`。

网关每次评估设置恰好一个 `phase` 变体。活动变体标识所选阶段并确定哪个载荷可用。`proposed_operation` 是状态加载、默认值处理和先前补丁之后的网关准备操作；它在 `modify_operation`、`validate` 和 `post_commit` 中都存在。`current_state` 在网关加载了先前网关拥有状态时作为只读上下文伴随准备操作阶段。

网关拦截器响应返回允许/拒绝结果、可选补丁和所选网关操作的诊断元数据。

```proto
message InterceptorResult {
  bool allowed = 1;
  string reason = 2;
  string status_code = 3;
  repeated JsonPatch patches = 4;
  repeated string warnings = 5;
  map<string, string> audit_annotations = 6;
}
```

网关将有效的 `InterceptorResult` 投射到网关操作上。`allowed = true` 让操作继续。`allowed = false` 在 `modify_operation` 或 `validate` 中使用 `status_code` 和 `reason` 拒绝网关 API 操作。只有 `modify_operation` 接受补丁，且这些补丁应用于 `modify_operation.proposed_operation`。`current_state` 是只读上下文，永远不会被补丁修改。`warnings` 和 `audit_annotations` 在适用时投射到网关响应元数据和日志中。`post_commit` 在网关操作提交后运行，因此不能拒绝或修改操作。`post_commit` 结果中 `allowed = false` 或包含补丁是网关拦截器契约违规。

`binding_id` 由网关拦截器服务拥有。它标识选择该评估的服务声明绑定。

### 网关拦截器端点

框架使用一个 protobuf/gRPC 服务契约。网关拦截器端点通过 gRPC 连接，可以是远程端点或 Unix 域套接字。

所有网关拦截器连接都需要认证。确切的认证模型不在本 RFC 范围内，但实现应支持 mTLS 和 bearer token 认证。

### 选择与排序

选择应围绕网关拦截器服务，而非单个阶段/RPC 路由。运维人员通常应配置少量此类服务和服务特定设置。服务告诉网关它支持哪些 RPC 绑定。

网关配置 TOML 中的 `[[openshell.gateway.interceptors]]` 表代表一个网关拦截器服务实例。在网关启动或配置重载期间，网关调用服务上的 `Describe` RPC。响应描述服务的默认绑定：

```proto
message InterceptorManifest {
  repeated InterceptorBinding bindings = 1;
}

message InterceptorBinding {
  string id = 1;
  repeated GatewayInterceptorPhase phases = 2;
  repeated string rpcs = 3;
  int32 order = 4;
  string on_error = 5;
  InterceptorSelector selector = 6;
}

message InterceptorSelector {
  repeated string principals = 1;
  map<string, string> labels = 2;
}
```

默认情况下，网关启用服务清单返回的绑定。运维人员可以配置服务一次，然后在需要禁用、收窄或重排行为时可选地覆盖特定绑定。覆盖应只收窄服务声明的选择器，除非未来的 RFC 明确允许扩展。

空选择器字段匹配所有值。网关覆盖可以收窄服务声明的选择器，例如将绑定限制到特定 RPC。补丁能力从所选阶段派生，而非单独的绑定标志。`modify_operation` 中的绑定可以返回零个或多个补丁。`validate` 或 `post_commit` 中的绑定不得返回补丁。

远程策略提供者的网关配置示例：

```toml
[[openshell.gateway.interceptors]]
name = "policy-provider"
grpc_endpoint = "https://policy-provider.example.com:8443"
order = 100
failure_policy = "fail_closed"
timeout = "500ms"
```

服务清单保持常见配置简洁。运维人员不需要知道每个行为在哪个阶段运行；服务暴露这些绑定。

网关从启用的绑定构建执行计划。选择评估服务声明的 RPC、阶段、主体和标签选择器，然后应用网关配置的收窄覆盖。

网关拦截器按固定阶段顺序运行。在阶段内，匹配的绑定按以下确定性排序运行：

1. 配置的网关拦截器服务 `order`。
2. 网关覆盖后的服务声明绑定 `order`。
3. 网关拦截器服务名称。
4. 绑定 ID。

补丁按绑定执行顺序应用。无效补丁、冲突补丁或从非修改阶段返回的补丁是无效的网关拦截器结果。

### 失败策略

失败策略是网关控制平面行为，用于网关无法获得或应用有效网关拦截器结果的情况。示例包括超时、传输失败、服务错误、无效响应、响应大小违规、无效阶段行为或补丁数量限制违规。`modify_operation` 或 `validate` 中有效的 `allowed = false` 结果不是错误策略情况；网关必须将其投射为操作拒绝。

每个绑定有有效的失败策略。网关从服务声明的绑定 `on_error` 值开始，应用网关拦截器服务级网关配置，然后应用任何绑定覆盖。

| 失败策略 | 行为 |
|---|---|
| `fail_closed` | 网关在提交副作用之前拒绝 API 操作。 |
| `fail_open` | 网关继续执行，不应用失败的网关拦截器结果，并发出警告和审计事件。 |

默认值：

- `modify_operation` 和 `validate` 绑定默认为 `fail_closed`。
- `post_commit` 绑定默认为 `fail_open`。

网关对每个网关拦截器服务调用强制执行超时和响应大小限制。每个绑定还有最大补丁数量。

### 可观测性与审计

每个网关拦截器结果应发出结构化网关日志，包含：

- 网关拦截器名称
- 绑定 ID
- 阶段
- RPC 服务和方法
- 主体
- 结果
- 原因
- 延迟
- 失败策略
- 补丁数量
- 审计注解

安全相关的拒绝应根据事件类别作为 OCSF 检测发现或配置/安全事件发出。非安全操作失败可以使用普通追踪。

### 示例：远程策略提供者

网关拦截器应从它想要保持的不变量开始，然后找到每个可以建立或削弱该不变量的网关 API RPC。例如，运维人员可能希望远程策略提供者成为沙箱策略决策的权威。

两个 RPC 对此不变量很重要：

- `openshell.v1.OpenShell/CreateSandbox` 建立初始沙箱策略。
- `openshell.v1.OpenShell/UpdateConfig` 更改沙箱或全局策略。

网关拦截器服务声明一个绑定来应用已批准的初始策略，另一个来守护后续策略变更：

```proto
InterceptorManifest {
  bindings: [
    {
      id: "sandbox-policy-default"
      phases: [GATEWAY_INTERCEPTOR_PHASE_MODIFY_OPERATION]
      rpcs: ["openshell.v1.OpenShell/CreateSandbox"]
      on_error: "fail_closed"
    },
    {
      id: "policy-authority"
      phases: [GATEWAY_INTERCEPTOR_PHASE_VALIDATE]
      rpcs: ["openshell.v1.OpenShell/UpdateConfig"]
      on_error: "fail_closed"
    }
  ]
}
```

处理器随后可以专注于选择绑定的活动阶段变体和 RPC 方法：

```rust
// Toy implementation of the GatewayInterceptor Evaluate RPC.
async fn evaluate(&self, req: InterceptorEvaluation) -> InterceptorResult {
    use interceptor_evaluation::Phase;

    match (req.rpc_method.as_str(), req.phase.as_ref()) {
        // CreateSandbox: ask the remote policy provider for the approved
        // initial policy and stamp it into the prepared operation input.
        ("CreateSandbox", Some(Phase::ModifyOperation(phase))) => {
            let approved_policy = self
                .policy_provider
                .initial_policy(&phase.proposed_operation)
                .await;

            InterceptorResult::allow().with_patch(JsonPatch::replace(
                "/policy",
                approved_policy,
            ))
        }

        // UpdateConfig: reject policy writes the remote provider does not approve.
        ("UpdateConfig", Some(Phase::Validate(phase))) => {
            let validation = self
                .policy_provider
                .validate_update(&phase.proposed_operation, &phase.current_state)
                .await;
            if !validation.allowed {
                return InterceptorResult::reject(
                    "PERMISSION_DENIED",
                    validation.reason,
                );
            }

            InterceptorResult::allow()
        }

        // The service should only receive bound RPCs, but defaulting to allow
        // keeps the handler safe if the manifest grows later.
        _ => InterceptorResult::allow(),
    }
}
```

网关配置可以保持精简，因为服务清单声明了绑定：

```toml
[[openshell.gateway.interceptors]]
name = "policy-provider"
grpc_endpoint = "https://policy-provider.example.com:8443"
order = 100
failure_policy = "fail_closed"
timeout = "500ms"
```

此示例说明了通用网关拦截器设计循环：

- 从不变量开始，然后识别每个可以建立或削弱它的 RPC。
- 按意图选择阶段：`modify_operation` 应用已批准的初始策略，`validate` 拒绝未授权的后续变更。
- 使用