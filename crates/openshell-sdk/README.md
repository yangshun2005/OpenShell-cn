# openshell-sdk

`openshell-sdk` 是 OpenShell 网关共享的异步 Rust 客户端。它负责 gRPC 通道建立、TLS、OIDC 刷新和 Cloudflare Access 隧道，使 CLI、TUI 和语言绑定共用同一套客户端实现。调用方传入显式 bearer token；SDK 不访问文件系统，也不解析网关名称。

## 两层接口

- `OpenShellClient` — 精选的、面向沙箱的接口：健康检查、沙箱 CRUD、可复用沙箱模板 CRUD、就绪/删除等待，以及非流式 exec。
- `raw` — 直接访问生成的 tonic 客户端，用于精选接口尚未覆盖的 RPC（providers、policy、logs、settings、SSH、forwarding）。

## 认证与刷新

精选接口自动驱动 OIDC 刷新：请求前主动刷新，收到 `Unauthenticated` 时被动刷新。刷新为单飞模式，同一时间只有一个刷新在进行。

普通 `raw_grpc` 访问器不刷新；它返回绑定当前 token 的客户端。接入刷新器后，用 `raw_grpc_fresh` 在调用前刷新，用 `force_refresh` 在 raw RPC 返回 `Unauthenticated` 后恢复。

SDK 使用调用方实现的 `Refresh` trait；它本身不运行 OIDC 浏览器流程。其非交互式 refresh-token 交换还接受身份提供方可能要求的 scope，用于为刷新后的 access token 选择 API 资源。

## 传输模式

- 明文（本地开发）
- 服务器认证 TLS（系统根证书，或通过 `ca_cert` 固定私有 CA）
- 通过内存中 `ClientIdentity` 的双向 TLS
- HTTPS 上的 OIDC bearer（网关位于 OAuth2/OIDC IdP 之后）
- Cloudflare Access 隧道（托管网关）
- 不安全 TLS（开发/调试；禁用证书验证）

## 公开接口

`OpenShellClient::connect(ClientConfig)` 返回已连接的客户端，暴露 `health`、`create_sandbox`、`get_sandbox`、`list_sandboxes`、`list_all_sandboxes`、`delete_sandbox`、`create_sandbox_from_template`、`create_sandbox_template`、`get_sandbox_template`、`list_sandbox_templates`、`delete_sandbox_template`、`list_sandboxes_all_workspaces`、`list_sandbox_templates_all_workspaces`、`wait_ready`、`wait_deleted` 和 `exec`。精选类型（`SandboxSpec`、`SandboxRef`、`Health`、`ListOptions`、`SandboxTemplateListOptions`、`ExecOptions`、`SandboxPhase`）在可行处使用 SDK 形态的枚举而非原始 proto 整数。可复用模板资源以 `SandboxWorkloadTemplate` proto 别名暴露，调用方可填充完整的可移植工作负载形态和驱动配置。失败映射为带可判别 kind 的类型化 `SdkError`。

创建时设置 `SandboxSpec::service_exposures` 可注册具名或匿名的回环 HTTP 服务。每个 `ServiceExposure` 包含服务名、目标端口和授权模式；空名称选择匿名端点。授权默认剥离。仅当沙箱应用自行校验 bearer 凭证时才选择 `BearerPassthrough`。返回的 `SandboxRef::service_urls` map 以相同名称包含各路由 URL。

不带 workspace 参数的精选调用显式选择 `default` workspace。跨 workspace 列表使用单独的 `*_all_workspaces` 方法，需要 Platform Admin 权限。

对于已接受的沙箱删除，将其原始 ID 传给 `wait_deleted`，以免同名替换延长等待。默认客户端和 workspace 作用域客户端都接受可选的第三个参数；传 `None` 则等待名称消失。

```rust
let deletion = client.delete_sandbox(name, openshell_sdk::DeleteOptions::default()).await?;
if deletion.outcome == openshell_sdk::DeletionOutcome::Accepted {
    client.wait_deleted(
        name,
        std::time::Duration::from_secs(60),
        deletion.sandbox_id.as_deref(),
    ).await?;
}
```

精选 `list_*` 方法返回惰性 `Pager<T>`。每次 `next_page()` 调用获取一个逻辑页，返回带不透明续传 token 的 `Page<T>`。OIDC 认证可在 `Unauthenticated` 响应后重试该页一次。显式 `list_all_*` 便捷方法耗尽该分页器；`page_size` 始终控制单次网关请求，`page_token` 恢复已保存的遍历。

```rust
let mut pages = client.list_sandboxes(ListOptions {
    page_size: 100,
    ..Default::default()
});
while let Some(page) = pages.next_page().await? {
    for sandbox in page.items {
        println!("{}", sandbox.name);
    }
}
```

```rust
use openshell_sdk::{
    ClientConfig, OpenShellClient, SandboxTemplateCreateSpec,
    SandboxWorkloadConfig, SandboxWorkloadTemplate, SandboxWorkloadTemplateSpec,
};

# async fn run() -> Result<(), openshell_sdk::SdkError> {
let client = OpenShellClient::connect(ClientConfig::new("http://127.0.0.1:8080")).await?;
client
    .create_sandbox_template(SandboxWorkloadTemplate {
        metadata: Some(openshell_sdk::raw::proto::datamodel::v1::ObjectMeta {
            name: "python".to_string(),
            ..Default::default()
        }),
        spec: Some(SandboxWorkloadTemplateSpec {
            workload: Some(SandboxWorkloadConfig {
                image: "registry.example.com/agents/python:latest".to_string(),
                ..Default::default()
            }),
            ..Default::default()
        }),
    })
    .await?;

let _sandbox = client
    .create_sandbox_from_template(SandboxTemplateCreateSpec {
        template_name: "python".to_string(),
        policy: Some(openshell_sdk::raw::proto::SandboxPolicy {
            version: 1,
            ..Default::default()
        }),
        ..Default::default()
    })
    .await?;
# Ok(())
# }
```

## 等待 provider 变更

Provider 附加、分离和更新响应包含 `ProviderMutationReceipt`：一条保存的记录，标识为一个沙箱请求的确切变更。将该记录传给 `provider_readiness::wait_for_provider`，等待当前沙箱运行时确认已应用该变更。分离以 `Revoked` 完成；附加和更新以 `Ready` 完成。

```rust
use std::time::Duration;
use openshell_sdk::{OpenShellClient, raw::ProviderMutationReceipt};
use openshell_sdk::provider_readiness::{
    ProviderWaitOutcome, wait_for_provider,
};

async fn wait_for_change(
    client: &OpenShellClient,
    change: &ProviderMutationReceipt,
) -> Result<(), Box<dyn std::error::Error>> {
    let mut grpc = client.raw_grpc_fresh().await?;
    let result = wait_for_provider(&mut grpc, change, Duration::from_secs(30)).await?;
    match result.outcome {
        ProviderWaitOutcome::Complete => println!("The sandbox applied the change."),
        ProviderWaitOutcome::TimedOut => println!("Still waiting; check the same change again."),
        ProviderWaitOutcome::Terminal => println!("The change failed, was withheld, or was replaced."),
    }
    Ok(())
}
```

截止时间到期时，结果保留最后已知状态。后续变更无法满足对原始请求的等待。`provider_status` 查询一次，`wait_for_provider_until` 接受共享截止时间，用于等待一次 provider 更新所选中的沙箱。状态响应包含配置标识和安全原因类别，不含凭证或原始安装错误。

对于普通静态凭证，在更新就绪后启动新客户端以接收更新后的引用。现有进程保留其修订作用域引用；等待成功不会重定向它们，也不证明旧上游密钥可退役。分离完成后，保留的引用无法解析，新进程也不会收到它们。

状态的 `operation` 字段是公共操作的历史结果，以 receipt ID 为键。这些辅助方法根据实时 provider 状态及其匹配证据完成；历史已应用操作不能覆盖已断开、已过期或被取代的实时结果。

这些辅助方法使用 raw 客户端的认证槽。它们本身不执行 OIDC 刷新。若请求返回 `Unauthenticated`，遵循上文 raw 客户端刷新指引，然后继续等待同一变更 ID。

## 模块

| 模块 | 用途 |
|---|---|
| `client` | 高层 `OpenShellClient` 和精选沙箱接口。 |
| `config` | `ClientConfig`、`AuthConfig`、`ClientIdentity`。 |
| `transport` | 通道构建、TLS 解析、请求拦截器。 |
| `auth` | 用于附加 bearer token 的 `EdgeAuthInterceptor`。 |
| `oidc` | 传输层的 OIDC token 处理。 |
| `refresh` | `Refresh` trait 和单飞刷新合并。 |
| `edge_tunnel` | Cloudflare Access 隧道拨号器。 |
| `error` | `SdkError` 分类。 |
| `pagination` | 惰性 `Pager<T>` 和响应 `Page<T>`。 |
| `types` | 精选请求/响应类型和 proto 转换。 |
| `raw` | 逃生舱，重新导出生成的 tonic 客户端。 |
| `provider_readiness` | 检查并等待确切 provider 变更生效。 |

## 说明

- 仅异步。Tonic 原生异步；需要阻塞调用的调用方可用自己的运行时包装。
- 随着更多 RPC 从 `raw` 毕业，精选接口将扩展。