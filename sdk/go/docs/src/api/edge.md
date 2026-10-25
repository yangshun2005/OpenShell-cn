# Edge

包：`openshell/v1/edge`

edge 包提供通过边缘代理（如 Cloudflare Access）连接 OpenShell 网关的工具。包含用于边缘代理请求头的认证包装器，以及用于在仅支持 HTTP/1.1 的代理上传输 gRPC 的 WebSocket 隧道代理。

## Cloudflare Access

用 Cloudflare Access 请求头（`cf-access-jwt-assertion` 和 `CF_Authorization` cookie）包装任意 `AuthProvider`：

```go
import "github.com/NVIDIA/OpenShell/sdk/go/openshell/v1/edge"

base := v1.StaticToken("my-gateway-token")
auth, err := edge.CloudflareAccess(base, os.Getenv("CF_ACCESS_TOKEN"))
if err != nil {
    log.Fatal(err)
}
client, err := v1.NewClient(v1.Config{
    Address: "gateway.example.com:443",
    Auth:    auth,
})
```

CloudflareAccess 可与任意认证提供者组合，包括用于自动刷新令牌的 `RefreshableToken`：

```go
tokenSource := oauth2Config.TokenSource(ctx, initialToken)
refreshAuth, err := v1.RefreshableToken(tokenSource)
if err != nil {
    log.Fatal(err)
}
auth, err := edge.CloudflareAccess(refreshAuth, cfToken)
```

## WebSocket 隧道

`TunnelProxy` 通过 WebSocket 隧道桥接 gRPC 连接，适用于拒绝标准 HTTP/2 POST 请求的边缘代理。隧道携带自己的边缘令牌用于代理认证，独立于应用层认证提供者。

```go
tunnel, err := edge.NewTunnelProxy(
    "wss://gateway.example.com/ws",
    os.Getenv("CF_ACCESS_TOKEN"),
)
if err != nil {
    log.Fatal(err)
}
defer tunnel.Close()

auth := v1.StaticToken("my-gateway-token")
client, err := v1.NewClient(v1.Config{
    Address: tunnel.Addr(),
    Auth:    auth,
    TLS:     &v1.TLSConfig{Insecure: true}, // local tunnel
})
```

## 函数

| 函数 | 描述 |
|----------|-------------|
| `CloudflareAccess(base, edgeToken)` | 用 Cloudflare Access 请求头包装 AuthProvider |
| `NewTunnelProxy(url, edgeToken, opts...)` | 创建用于 gRPC-over-HTTP/1.1 的 WebSocket 隧道代理 |

## TunnelProxy 方法

| 方法 | 描述 |
|--------|-------------|
| `Addr()` | gRPC 客户端拨号的本地监听地址 |
| `Close()` | 优雅排空进行中的连接并关闭 |

## TunnelOption

| 构造函数 | 作用 |
|-------------|--------|
| `WithTunnelTLS(cfg)` | 为 WebSocket 连接配置 TLS |
| `WithTunnelLogger(l)` | 设置隧道事件的日志记录器 |
| `WithCloseTimeout(d)` | 覆盖优雅关闭超时（默认 5s） |

## 线程安全

所有导出的函数和方法均可安全并发使用。`Close` 是幂等的，可安全多次调用。