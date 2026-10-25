# OpenShell Go SDK

[![Go Reference](https://pkg.go.dev/badge/github.com/NVIDIA/OpenShell/sdk/go.svg)](https://pkg.go.dev/github.com/NVIDIA/OpenShell/sdk/go)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](../../LICENSE)

> [!IMPORTANT]
> **[阅读完整文档](https://ro14nd.de/openshell-sdk-go/)**，获取指南、含 gRPC 映射的 API 参考以及测试模式。

用于与 [OpenShell](https://github.com/NVIDIA/OpenShell) 服务器交互的 Go SDK，提供符合 Go 习惯的绑定，涵盖 shell 会话管理、命令执行、Provider 配置和服务暴露。

## 为什么需要 Go SDK？

Go 是 Kubernetes 生态的语言。如果你想构建 operator、controller，或任何将 OpenShell 资源作为原生 Kubernetes 对象管理的自动化工具，就需要 Go 客户端。

本 SDK 参照 [`k8s.io/client-go`](https://github.com/kubernetes/client-go) 设计——这是每位 Go operator 开发者都熟悉的 Kubernetes 标准客户端库。模式似曾相识：

- **按资源划分的类型化子客户端**：`client.Sandboxes()`、`client.Providers()`、`client.Exec()`，就像 `clientset.CoreV1().Pods()`
- **领域类型与传输格式分离**：`types` 包中干净的 Go 结构体，公共 API 不泄漏 proto（类似 `k8s.io/api`）
- **Watch 原语**：基于 channel 的 watcher，提供 `ResultChan()` 和 `Stop()`，与 client-go 中的 `watch.Interface` 一致
- **函数式选项**：用于列表过滤、分页和 watch 配置的可变参数选项模式。所有入口点都会静默忽略 nil 选项，因此条件选项列表可以安全传入，无需过滤 nil 项。
- **可组合认证与令牌刷新**：封装 `oauth2.TokenSource`，实现自动令牌缓存和合并刷新，遵循 k8s client-go 的 `cachingTokenSource` 模式
- **用于测试的 Fake 客户端**：完整客户端接口的内存实现（类似 `k8s.io/client-go/kubernetes/fake`），无需真实网关即可测试 operator

## 快速开始

```go
import v1 "github.com/NVIDIA/OpenShell/sdk/go/openshell/v1"

// Connect to a gateway
client, err := v1.NewClient(v1.Config{
    Address: "gateway.example.com:443",
    Auth:    v1.StaticToken("my-token"),
})
if err != nil {
    log.Fatal(err)
}
defer client.Close()

// Create a sandbox and wait until it's ready
sandbox, err := client.Sandboxes().Create(ctx, "default", "my-sandbox", &v1.SandboxSpec{
    Template: &v1.SandboxTemplate{Image: "python:3.12"},
})
if err != nil {
    log.Fatal(err)
}
sandbox, err = client.Sandboxes().WaitReady(ctx, "default", sandbox.Name)
if err != nil {
    log.Fatal(err)
}

// Run a command
result, err := client.Exec().Run(ctx, "default", sandbox.Name,
    []string{"python3", "-c", "print('hello from sandbox')"},
    v1.ExecOptions{},
)
if err != nil {
    log.Fatal(err)
}
fmt.Println(string(result.Stdout))
```

### 分页

List 方法返回惰性分页器，不会发起请求。`NextPage` 使用传入的 context 获取一页，`ListAll` 则显式遍历所有页。`PageSize` 是单次请求的最大条数，`PageToken` 用于续接之前的查询。

```go
pager, err := client.Sandboxes().List("default", v1.ListOptions{PageSize: 100})
if err != nil {
    log.Fatal(err)
}
for {
    page, err := pager.NextPage(ctx)
    if err != nil {
        log.Fatal(err)
    }
    if page == nil {
        break
    }
    for _, sandbox := range page.Items {
        fmt.Println(sandbox.Name)
    }
}
```

### 自动令牌刷新

对于 OIDC 网关，使用 `RefreshableToken` 封装任意 `oauth2.TokenSource`，实现自动缓存和合并刷新：

```go
import "golang.org/x/oauth2"

tokenSource := oauth2Config.TokenSource(ctx, initialToken)
auth, err := v1.RefreshableToken(tokenSource,
    v1.WithLeeway(30*time.Second),
)
if err != nil {
    log.Fatal(err)
}
client, err := v1.NewClient(v1.Config{
    Address: "gateway.example.com:443",
    Auth:    auth,
})
if err != nil {
    log.Fatal(err)
}
defer client.Close()
```

并发调用方共享单次刷新调用。如果令牌源失败，SDK 会回退到缓存的令牌并记录警告日志。详见 [Auth](https://ro14nd.de/openshell-sdk-go/api/auth.html) 文档。

### 边缘代理请求头

当网关位于零信任反向代理之后时，使用 `WithExtraHeaders` 在标准认证之外附加代理专用请求头：

```go
base := v1.StaticToken("my-gateway-token")
auth, err := v1.WithExtraHeaders(base, map[string]string{
    "x-proxy-auth": "proxy-secret",
})
if err != nil {
    log.Fatal(err)
}
client, err := v1.NewClient(v1.Config{
    Address: "gateway.example.com:443",
    Auth:    auth,
})
```

对于 Cloudflare Access，使用 `edge` 包中的便捷构造函数：

```go
import "github.com/NVIDIA/OpenShell/sdk/go/openshell/v1/edge"

auth, err := edge.CloudflareAccess(base, os.Getenv("CF_ACCESS_TOKEN"))
```

对于位于拒绝 HTTP/2 的边缘代理之后的 gRPC，使用 WebSocket 隧道：

```go
tunnel, err := edge.NewTunnelProxy(
    "wss://gateway.example.com/ws",
    os.Getenv("CF_ACCESS_TOKEN"),
)
if err != nil {
    log.Fatal(err)
}
defer tunnel.Close()

client, err := v1.NewClient(v1.Config{
    Address: tunnel.Addr(),
    Auth:    v1.StaticToken("my-token"),
    TLS:     &v1.TLSConfig{Insecure: true}, // local tunnel, no TLS
})
```

### OIDC 登录

`oidc` 包提供网关感知的 OIDC 认证，支持浏览器、键盘、设备码和客户端凭据流程：

```go
import "github.com/NVIDIA/OpenShell/sdk/go/openshell/v1/oidc"

// Gateway-aware login: reads OIDC config from gateway metadata
token, err := oidc.Login(ctx, "my-gateway")
if err != nil {
    log.Fatal(err)
}

// Use the token with the SDK client
client, err := v1.NewClient(v1.Config{
    Address: "gateway.example.com:443",
    Auth:    v1.StaticToken(token.AccessToken),
})
```

对于无头环境，使用设备码流程：

```go
token, err := oidc.DeviceLogin(ctx,
    oidc.WithIssuer("https://auth.example.com"),
    oidc.WithClientID("my-app"),
)
```

对于一次性服务账号令牌交换，使用 `ClientCredentials`。对于长期运行的 SDK 客户端，附加可续期、仅内存的认证 Provider：

```go
auth, err := oidc.NewClientCredentialsAuth(
    oidc.WithGateway("my-gateway"),
    oidc.WithClientSecretProvider(func(context.Context) (string, error) {
        return os.Getenv("OPENSHELL_OIDC_CLIENT_SECRET"), nil
    }),
)
if err != nil {
    log.Fatal(err)
}

client, err := v1.NewClient(v1.Config{
    Address: "gateway.example.com:443",
    Auth:    auth,
    TLS:     tlsConfig,
})
if err != nil {
    log.Fatal(err)
}
defer client.Close()
```

可以使用显式的 `WithIssuer`、`WithClientID`、`WithScopes` 和 `WithAudience` 选项代替 `WithGateway`。该 Provider 会在过期前重复授权，且绝不将密钥或访问令牌写入磁盘。

所有选项和流程详见 [oidc 包文档](https://pkg.go.dev/github.com/NVIDIA/OpenShell/sdk/go/openshell/v1/oidc)。

完整演练详见[入门指南](https://ro14nd.de/openshell-sdk-go/getting-started.html)。

## 从 v0.0.101 迁移

1.0 之前的 SDK 有意包含源码不兼容的 API 修正：

- `TCP.Listen` 返回 `ForwardListener` 生命周期句柄。SDK 拥有 accept 循环；调用方通过 `Addr()` 拨号并调用 `Close()`，而不是调用 `Accept()` 或将句柄传给 `http.Serve`。
- 资源操作需要显式 workspace，带 workspace 的领域类型会保留该作用域。
- 若干公共结构体字段顺序已变更。请使用带键的结构体字面量。
- 首字母缩写词使用 Go 拼写，包括 `JSONRPCMaxBodyBytes`。
- Provider profile 时长使用精确的 `RefreshBefore`、`MaxLifetime` 和 `CacheTTL` 字段。旧的整秒字段已移除。

这些变更是有意为之，因为模块仍低于 v1。请将调用方作为一次迁移整体更新，不要依赖 v0.0.101 的 API 形态。

## 架构

```
Client
  ├── Sandboxes()   → SandboxInterface    (create, get, list, delete, watch, wait, logs)
  ├── Exec()        → ExecInterface       (run, stream, interactive)
  ├── Files()       → FileInterface       (upload, download)
  ├── Health()      → HealthInterface     (health check, gateway info, current user)
  ├── Services()    → ServiceInterface    (expose, get, list, delete)
  ├── Providers()   → ProviderInterface   (CRUD + ensure)
  │     ├── Profiles() → ProfileInterface (list, get, import, update, lint, delete)
  │     └── Refresh()  → RefreshInterface (configure, status, rotate, delete)
  ├── Workspaces()  → WorkspaceInterface  (create, get, list, delete, members)
  └── Policy()      → PolicyInterface     (draft review, approve, reject, merge, status)
```

所有领域类型位于 `openshell/v1/types/`。Proto 到 Go 的转换发生在内部转换层。公共 API 表面使用类型别名，因此使用方只需导入单个包。详见[架构](https://ro14nd.de/openshell-sdk-go/architecture.html)概览。

## 功能

使用 `CloseInteractiveInput(session)` 关闭 stdin 并调整输入大小，同时保持输出可读。SDK 会话实现可选的 `InteractiveSessionControl` 接口（`CloseWrite()` 和 `Cancel()`）；原始 `InteractiveSession` 接口对现有 mock 和包装器保持不变。对于不支持该能力的会话，输入关闭返回 `ErrorUnimplemented` 并保持会话打开。`CancelInteractive(session)` 在可用时使用 `Cancel()`，否则调用 `Close()`。SDK 的关闭/取消操作是幂等的；输入关闭后的写入和调整大小返回 `io.ErrClosedPipe`。在等待 `ExitCode()` 的同时并发排空 `Read`。`ExitCode()` 等待最终 gRPC 状态，并返回观察到的进程退出码以及后续的流错误。仅凭退出事件不能确立流成功完成。

| 功能 | 接口 | 文档 |
|---------|-----------|------|
| Sandbox 生命周期（创建、获取、列表、删除、watch、等待） | `SandboxInterface` | [Sandboxes](https://ro14nd.de/openshell-sdk-go/api/sandboxes.html) |
| 命令执行（收集、流式、交互式 PTY） | `ExecInterface` | [Exec](https://ro14nd.de/openshell-sdk-go/api/exec.html) |
| Provider 管理（CRUD + 幂等 ensure） | `ProviderInterface` | [Providers](https://ro14nd.de/openshell-sdk-go/api/providers.html) |
| Provider profile（列表、导入、lint、更新） | `ProfileInterface` | [Profiles](https://ro14nd.de/openshell-sdk-go/api/profiles.html) |
| 凭据刷新（配置、轮换、状态） | `RefreshInterface` | [Refresh](https://ro14nd.de/openshell-sdk-go/api/refresh.html) |
| 服务暴露（暴露、列表、删除） | `ServiceInterface` | [Services](https://ro14nd.de/openshell-sdk-go/api/services.html) |
| 文件传输 API（受传输能力门控） | `FileInterface` | [Files](https://ro14nd.de/openshell-sdk-go/api/files.html) |
| 策略管理（草稿审查、批准、拒绝、合并、全局策略） | `PolicyInterface` | [Policy](https://ro14nd.de/openshell-sdk-go/api/policy.html) |
| Sandbox 日志（流式获取） | `SandboxInterface` | [Sandboxes](https://ro14nd.de/openshell-sdk-go/api/sandboxes.html) |
| Workspace 管理（创建、获取、列表、删除、成员） | `WorkspaceInterface` | [Workspaces](https://ro14nd.de/openshell-sdk-go/api/workspaces.html) |
| Sandbox Provider 附加（附加、分离、列表） | `SandboxInterface` | [Sandboxes](https://ro14nd.de/openshell-sdk-go/api/sandboxes.html) |
| 网关信息和当前用户身份 | `HealthInterface` | [Health](https://ro14nd.de/openshell-sdk-go/api/health.html) |
| 健康检查 | `HealthInterface` | [Health](https://ro14nd.de/openshell-sdk-go/api/health.html) |
| SSH 隧道和 TCP 转发 | `SSHInterface`、`TCPInterface` | [SSH](https://ro14nd.de/openshell-sdk-go/api/ssh.html)、[TCP](https://ro14nd.de/openshell-sdk-go/api/tcp.html) |
| 认证：静态令牌、可刷新令牌（oauth2.TokenSource） | `AuthProvider` | [Auth](https://ro14nd.de/openshell-sdk-go/api/auth.html) |
| 边缘认证：额外请求头、Cloudflare Access、WebSocket 隧道 | `AuthProvider`、`edge.TunnelProxy` | [Edge](https://ro14nd.de/openshell-sdk-go/api/edge.html) |
| 类型化错误（`IsNotFound`、`IsAlreadyExists`、`IsConflict` 等） | `StatusError` | [错误处理](https://ro14nd.de/openshell-sdk-go/error-handling.html) |
| 实时 watch，终态自动停止 | `WatchInterface[T]` | [Sandboxes](https://ro14nd.de/openshell-sdk-go/api/sandboxes.html) |
| 用于测试的 Fake 客户端（无需 gRPC 服务器） | `fake.Client` | [测试](https://ro14nd.de/openshell-sdk-go/testing.html) |
| OIDC 登录和可续期服务认证 | `oidc.Login`、`oidc.DeviceLogin`、`oidc.ClientCredentials`、`oidc.NewClientCredentialsAuth` | [OIDC](https://pkg.go.dev/github.com/NVIDIA/OpenShell/sdk/go/openshell/v1/oidc) |
| 网关配置便捷方式（加载 CLI 网关配置、自动接线认证） | `gateway.NewClient`、`gateway.LoadConfig` | [Gateway](https://ro14nd.de/openshell-sdk-go/api/gateway.html) |

## 前置条件

- Go 1.25.13 或更高版本
- [mise](https://mise.jdx.dev)（推荐用于可复现构建）

## 构建与测试

```bash
git clone https://github.com/NVIDIA/OpenShell.git
cd OpenShell/sdk/go

mise run test    # Run tests with coverage
mise run lint    # Run golangci-lint
mise run ci      # Full CI pipeline (lint + build + test)
```

构建命令使用 [mise](https://mise.jdx.dev) 实现可复现的工具管理。

## 文档

完整 API 文档见 [OpenShell Go SDK 文档](https://ro14nd.de/openshell-sdk-go/)站点。

本地构建文档：

```bash
cargo install mdbook
mdbook serve docs
```

## 许可证

Apache-2.0。详见 [LICENSE](../../LICENSE)。

Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.