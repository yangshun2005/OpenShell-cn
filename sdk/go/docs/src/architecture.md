# 架构

本页说明 OpenShell Go SDK 的内部结构。理解其设计有助于你熟悉 API 接口并写出符合惯例的代码。

## 客户端层级

SDK 遵循 Kubernetes client-go 的子客户端模式。单个 `Client` 为每个 API 域提供类型化访问器：

```text
Client
├── Sandboxes()   → SandboxInterface
├── Exec()        → ExecInterface
├── Providers()   → ProviderInterface
│   ├── Profiles()  → ProfileInterface
│   └── Refresh()   → RefreshInterface
├── Services()    → ServiceInterface
├── Files()       → FileInterface
├── Health()      → HealthInterface
├── SSH()         → SSHInterface
├── TCP()         → TCPInterface
├── Config()      → ConfigInterface
└── Policy()      → PolicyInterface
```

每个访问器返回一个接口。你操作的是接口，而非具体实现。这让子客户端易于 mock 和测试。

## 创建客户端

所有交互都从 `NewClient` 开始：

```go
client, err := v1.NewClient(v1.Config{
    Address: "gateway.example.com:443",
    Auth:    v1.StaticToken("my-token"),
})
if err != nil {
    log.Fatal(err)
}
defer client.Close()
```

`Config` 结构体控制：

| 字段 | 用途 |
|-------|---------|
| `Address` | 网关主机和端口 |
| `Auth` | 认证提供者（`StaticToken`、`NoAuth` 或自定义） |
| `TLS` | TLS 设置（CA 证书、跳过验证、客户端证书） |
| `Retry` | 瞬时故障的重试策略 |

## Proto 隔离

SDK 从不在公共 API 中暴露 protobuf 生成的类型。它在 `openshell/v1/` 中定义自己的 Go 类型，并在 gRPC 边界处与 proto 相互转换。

这意味着：

- 你的代码导入 `openshell/v1`，而非 `proto/openshellv1`
- 你操作的是普通 Go 结构体，而非 proto 消息
- 上游 OpenShell 的 proto Schema 变更不会破坏你的代码（SDK 在内部适配）
- 你可以对 SDK 类型使用标准 Go 模式（json.Marshal、fmt.Sprintf、reflect），不受 proto 约束

转换层位于 `openshell/v1/internal/converter/`，不属于公共 API。

## 子客户端模式

每个子客户端按特定 API 域组织方法。例如，`SandboxInterface` 提供 `Create`、`Get`、`List`、`Delete`、`WaitReady`、`Watch` 等。

子客户端访问开销很低。它们在 `Client` 初始化时创建一次，并复用同一底层 gRPC 连接：

```go
// These return the same sub-client instance every time
sandboxes := client.Sandboxes()
exec := client.Exec()
```

部分子客户端拥有自己的子客户端。`ProviderInterface` 暴露 `Profiles()` 和 `Refresh()`：

```go
profiles, err := client.Providers().Profiles().ListAll(ctx, "default")
status, err := client.Providers().Refresh().GetStatus(ctx, "default", "openai", "api-key")
```

## 函数式选项

许多 SDK 入口点接受可变选项参数来配置调用：

```go
auth, err := v1.RefreshableToken(tokenSource,
    v1.WithLeeway(30*time.Second),
    v1.WithLogger(logger),
)
```

选项按传入顺序应用，因此当后续选项设置相同字段时会覆盖前面的选项。**每个入口点都会静默忽略 nil 选项。** 你可以安全地按条件构建选项列表，无需过滤 nil 项：

```go
opts := []v1.RefreshOption{
    v1.WithLeeway(30 * time.Second),
    maybeLogger(), // may return nil
}
auth, err := v1.RefreshableToken(tokenSource, opts...)
```

## gRPC 层

底层上，SDK 通过 gRPC 与 OpenShell 网关通信。`proto/openshell.proto` 中的单个 `OpenShell` 服务定义了所有 RPC。SDK 将每个接口方法映射到一个或多个 RPC：

| 模式 | 示例 |
|---------|---------|
| 一元 RPC | `Create`、`Get`、`Delete` |
| 服务端流式 RPC | `Watch`、`Stream`、`GetLogs` |
| 客户端流式 RPC | 文件上传 |
| 双向流式 | `Interactive`、`Forward` |

gRPC 连接由 `Client` 管理。调用 `client.Close()` 会干净地关闭所有活动流和底层连接。

## 错误模型

所有 SDK 方法返回标准 Go 错误。来自网关的错误携带带类型错误码的 `StatusError`。使用 `Is*` 函数对错误分类：

```go
_, err := client.Sandboxes().Get(ctx, "default", "missing")
if v1.IsNotFound(err) {
    // sandbox does not exist
}
```

完整的错误检查和重试模式列表见[错误处理](error-handling.md)指南。

## 模拟客户端

为便于测试，SDK 提供 `openshell/v1/fake`，内含 `ClientInterface` 的内存实现。模拟客户端支持种子数据、watch 事件和健康模拟：

```go
fc := fake.NewClient()
fc.AddSandbox(&v1.Sandbox{Name: "test-sb", Status: v1.SandboxStatus{Phase: v1.SandboxReady}})
```

完整示例见[测试](testing.md)指南。