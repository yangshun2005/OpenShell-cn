# OIDC 登录

包：`openshell/v1/oidc`

oidc 包为 OpenShell 网关提供 OIDC 认证。支持四种 OAuth2 流程：基于浏览器的授权码 + PKCE、键盘回退、设备码（RFC 8628）和客户端凭证。

## 快速开始

```go
import "github.com/NVIDIA/OpenShell/sdk/go/openshell/v1/oidc"

// Gateway-aware login (reads OIDC config from gateway metadata)
token, err := oidc.Login(ctx, "my-gateway")
if err != nil {
    log.Fatal(err)
}
```

## 函数

| 函数 | 说明 |
|----------|-------------|
| `Login(ctx, gatewayName, opts...)` | 通过浏览器或键盘流程进行交互式登录 |
| `DeviceLogin(ctx, opts...)` | 设备授权授予（RFC 8628） |
| `ClientCredentials(ctx, opts...)` | 非交互式客户端凭证授予 |

### Login

```go
func Login(ctx context.Context, gatewayName string, opts ...LoginOption) (*oauth2.Token, error)
```

执行交互式 OIDC 登录。提供 `gatewayName` 时，从网关的 `metadata.json` 读取 OIDC 颁发者和客户端 ID。默认流程打开浏览器，使用 PKCE 进行授权码交换。若提供方不支持 PKCE（S256），则不带 PKCE 继续。

令牌持久化到网关配置目录的 `oidc_token.json`。后续调用复用有效的缓存令牌。

独立使用时（无网关），传入空 `gatewayName` 并配合 `WithIssuer` 和 `WithClientID`。结合 `WithInMemory` 可跳过磁盘持久化。

### DeviceLogin

```go
func DeviceLogin(ctx context.Context, opts ...LoginOption) (*oauth2.Token, error)
```

执行 OAuth2 设备授权授予（RFC 8628）。流程向提供方请求设备码和用户码，通过 `WithDisplayFunc`（或 stdout）显示，然后轮询令牌端点直到用户完成授权。

需要 `WithIssuer` 和 `WithClientID`，或 `WithGateway`。

### ClientCredentials

```go
func ClientCredentials(ctx context.Context, opts ...LoginOption) (*oauth2.Token, error)
```

执行非交互式 OAuth2 客户端凭证授予。需要 `WithIssuer`、`WithClientID` 和 `WithClientSecret`（或 `WithGateway` 配合 `WithClientSecret`）。客户端密钥不会出现在错误消息中。

### NewClientCredentialsAuth

```go
func NewClientCredentialsAuth(opts ...LoginOption) (types.AuthProvider, error)
```

返回用于 `v1.Config.Auth` 或 `gateway.WithAuth` 的惰性可续期认证提供方。它在内存中缓存访问令牌，在过期前 30 秒内续期，合并并发交换，续期失败时按失败关闭处理。应用轮换密钥材料时使用 `WithClientSecretProvider`。

## 选项

| 选项 | 说明 |
|--------|-------------|
| `WithIssuer(url)` | OIDC 提供方颁发者 URL |
| `WithClientID(id)` | OAuth2 客户端 ID |
| `WithClientSecret(secret)` | OAuth2 客户端密钥（用于客户端凭证） |
| `WithClientSecretProvider(provider)` | 为每次交换解析客户端密钥 |
| `WithAudience(audience)` | 可选的 OAuth2 资源服务器受众 |
| `WithScopes(scopes...)` | 自定义作用域（客户端凭证无隐式作用域） |
| `WithCallbackPort(port)` | localhost 回调的固定端口（默认：先试 8000，再试 18000） |
| `WithTimeout(d)` | 认证流程超时（默认：2 分钟） |
| `WithKeyboardFlow()` | 使用键盘流程代替浏览器 |
| `WithInMemory()` | 跳过令牌磁盘持久化 |
| `WithDisplayFunc(fn)` | 设备码流程的自定义显示 |
| `WithGateway(name)` | 从网关元数据解析 OIDC 配置 |

## 错误处理

包定义了用于 `errors.Is()` 匹配的哨兵错误：

| 错误 | 说明 |
|-------|-------------|
| `ErrDiscovery` | OIDC 发现文档获取失败 |
| `ErrAuthCode` | 授权码交换失败 |
| `ErrDeviceCode` | 设备码流程失败 |
| `ErrClientCredentials` | 客户端凭证交换失败 |
| `ErrTimeout` | 认证流程超时 |
| `ErrCallbackServer` | localhost 回调服务器失败 |
| `ErrTokenPersist` | 令牌读/写失败 |
| `ErrOIDCConfig` | OIDC 配置缺失或无效 |

```go
token, err := oidc.Login(ctx, "my-gateway")
if errors.Is(err, oidc.ErrDiscovery) {
    // Provider unreachable
}
if errors.Is(err, oidc.ErrTimeout) {
    // User did not complete login in time
}
```

## 网关集成

当网关的 `metadata.json` 包含 `oidc_issuer` 和 `oidc_client_id` 字段时，`Login` 和 `DeviceLogin` 可自动解析配置：

```go
// Login reads OIDC config from the gateway
token, err := oidc.Login(ctx, "my-gateway")

// Then use the token with the SDK client
client, err := v1.NewClient(v1.Config{
    Address: "gateway.example.com:443",
    Auth:    v1.StaticToken(token.AccessToken),
})
```

## 流程

### 浏览器流程（默认）

1. 从颁发者发现 OIDC 端点
2. 生成 PKCE 码验证器和 S256 挑战
3. 打开浏览器到授权 URL
4. 启动 localhost 回调服务器接收授权码
5. 使用 PKCE 验证交换授权码获取令牌
6. 将令牌持久化到磁盘

### 键盘流程

与浏览器流程相同，但打印授权 URL 供用户手动复制。用户将授权码粘贴回终端。使用 `WithKeyboardFlow()` 启用。

### 设备码流程（RFC 8628）

1. 向提供方请求设备码和用户码
2. 显示验证 URL 和用户码
3. 按提供方要求的间隔轮询令牌端点
4. 通过增加轮询间隔处理 `slow_down` 响应

### 客户端凭证流程

1. 从颁发者发现 OIDC 端点
2. 用客户端 ID 和密钥交换访问令牌
3. 无需用户交互