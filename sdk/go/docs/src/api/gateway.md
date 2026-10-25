# Gateway

包：`openshell/v1/gateway`

gateway 包读取 OpenShell Rust CLI 创建的磁盘网关配置，并构建完整接线的 SDK 客户端。它省去了定位配置文件、解析元数据、加载令牌和接线认证提供者的模板代码。

## 快速开始

```go
import "github.com/NVIDIA/OpenShell/sdk/go/openshell/v1/gateway"

// Connect to a named gateway
client, err := gateway.NewClient("prod")
if err != nil {
    log.Fatal(err)
}
defer client.Close()
```

## 函数

| 函数 | 说明 |
|----------|-------------|
| `NewClient(name, opts...)` | 从网关配置创建完整接线的 SDK 客户端 |
| `LoadConfig(name)` | 解析网关配置但不建立连接 |
| `ListGateways()` | 枚举所有可用网关 |

### NewClient

```go
func NewClient(name string, opts ...ClientOption) (*v1.Client, error)
```

为指定名称的网关创建完整配置的 SDK 客户端。若 `name` 为空，则使用活动网关（通过 `openshell gateway use` 设置）。

该函数解析网关目录、解析 `metadata.json`、惰性加载令牌、将认证模式映射到 SDK 认证提供者，并在委托给 `v1.NewClient` 前应用所有 `ClientOption` 值。

```go
// Named gateway
client, err := gateway.NewClient("prod")

// Active gateway
client, err := gateway.NewClient("")

// With options
client, err := gateway.NewClient("staging",
    gateway.WithTimeout(10 * time.Second),
    gateway.WithLogger(myLogger),
)
```

**错误**：`ErrGatewayNotFound`、`ErrConfigParse`、`ErrTokenLoad`、`ErrUnsupportedAuthMode`、`ErrInvalidGatewayName`、`ErrNoActiveGateway`

### LoadConfig

```go
func LoadConfig(name string) (*Config, error)
```

解析网关配置但不创建客户端连接。返回冻结快照；调用后磁盘文件的变更不会反映。

```go
cfg, err := gateway.LoadConfig("staging")
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Endpoint: %s, Auth: %s\n", cfg.Endpoint, cfg.AuthMode)
```

### ListGateways

```go
func ListGateways() ([]Info, error)
```

从用户和系统目录枚举所有可用网关。用户网关优先显示。重名时以用户为准。未配置网关时返回空切片（而非错误）。

```go
gateways, err := gateway.ListGateways()
for _, gw := range gateways {
    fmt.Printf("%s (active=%v, source=%s)\n", gw.Name, gw.Active, gw.Source)
}
```

## 类型

### Config

```go
type Config struct {
    Name     string       // Validated gateway name
    Endpoint string       // Host:port of the gateway
    AuthMode AuthMode     // Resolved auth mode
    Source   ConfigSource // User or System origin
    Dir      string       // Absolute path to gateway config directory
}
```

### Info

```go
type Info struct {
    Name   string       // Gateway name from directory listing
    Active bool         // Whether this is the active gateway
    Source ConfigSource // User or System origin
}
```

### AuthMode

| 值 | 常量 | SDK AuthProvider |
|-------|----------|-----------------|
| `""` 或 `"none"` | `AuthModeNone` | `v1.NoAuth()` |
| `"plaintext"` | `AuthModePlaintext` | `v1.NoAuth()` + 不安全 TLS |
| `"cloudflare_jwt"` | `AuthModeCloudflareJWT` | 惰性边缘令牌认证 |
| `"oidc"` | `AuthModeOIDC` | 带磁盘源的 `v1.RefreshableToken` |
| `"mtls"` | `AuthModeMTLS` | 不支持（使用 `WithAuth`） |

### ClientOption

| 构造函数 | 作用 |
|-------------|--------|
| `WithLogger(l)` | 在 SDK 客户端上设置日志器 |
| `WithTimeout(d)` | 设置连接超时 |
| `WithTLS(cfg)` | 覆盖网关配置中的 TLS 设置 |
| `WithAuth(provider)` | 覆盖自动解析的认证提供者 |
| `WithRetryPolicy(p)` | 设置重试策略 |

## 错误处理

所有错误均支持 `errors.Is` 进行分类：

```go
client, err := gateway.NewClient("my-gateway")
if errors.Is(err, gateway.ErrGatewayNotFound) {
    fmt.Println("Gateway not configured. Run: openshell gateway add my-gateway")
}
if errors.Is(err, gateway.ErrTokenLoad) {
    fmt.Println("Token expired or missing. Run: openshell gateway login my-gateway")
}
```

| 错误 | 含义 |
|-------|---------|
| `ErrGatewayNotFound` | 用户或系统路径中无网关目录 |
| `ErrConfigParse` | metadata.json 缺失或格式错误 |
| `ErrTokenLoad` | 令牌文件缺失或不可读 |
| `ErrUnsupportedAuthMode` | 无法识别的 auth_mode 值 |
| `ErrInvalidGatewayName` | 名称未通过校验 |
| `ErrNoActiveGateway` | 未配置活动网关 |

## 磁盘布局

该包从以下位置读取网关元数据：

```
$XDG_CONFIG_HOME/openshell/          (user, default: ~/.config/openshell/)
├── active_gateway                   # Plain text: active gateway name
└── gateways/
    └── <name>/
        ├── metadata.json            # {"endpoint":"...","auth_mode":"...","name":"..."}
        ├── edge_token               # Cloudflare edge JWT (plaintext)
        ├── cf_token                 # Legacy edge token (fallback)
        └── oidc_token.json          # OIDC token bundle

/etc/openshell/gateways/             (system, fallback)
```

同名时用户网关优先于系统网关。

## 线程安全

所有导出函数（`NewClient`、`LoadConfig`、`ListGateways`）均可从多个 goroutine 并发安全调用。令牌加载使用内部同步。