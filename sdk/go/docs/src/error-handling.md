# 错误处理

网关状态失败和 SDK 校验失败使用 `*v1.StatusError`，它携带机器可读的 `Code` 和人类可读的 `Message`。本地 I/O、传输层初始化和哨兵错误可能使用其他 Go 错误类型。SDK 为分类状态错误提供谓词函数。

## StatusError

当操作返回分类状态错误时，可直接检查它，或使用下方的便捷谓词。对于已文档化的哨兵错误（如 `ErrTransportNotAvailable`），使用 `errors.Is`。

```go
var se *v1.StatusError
if errors.As(err, &se) {
    fmt.Printf("code: %s, message: %s\n", se.Code, se.Message)
    // se.ErrorInfo and se.FieldViolations contain optional structured details.
}
```

| 字段     | 类型              | 描述                          |
|-----------|-------------------|--------------------------------------|
| `Code`    | `ErrorCode`       | 机器可读的错误分类 |
| `Message` | `string`          | 人类可读的错误描述      |
| `GRPCCode` | `int32` | SDK 分类前的原始 gRPC 代码 |
| `FieldViolations` | `[]FieldViolation` | 被拒绝的请求字段及描述 |
| `ErrorInfo` | `*ErrorInfo` | 服务端原因、域和元数据 |
| `RetryDelay` | `*time.Duration` | 对原本安全的操作建议的最小重试延迟 |
| `Cause` | `error` | 原始 gRPC 错误，包含未知详情 |

使用 `status.FromError(err)` 通过 SDK 的错误包装器检查完整的 gRPC 状态。延迟不能证明超时的变更操作未被应用。

## 谓词函数

使用以下顶层函数检查错误类型。它们通过 `errors.As` 支持包装后的错误。

| 函数             | ErrorCode            | 触发条件                                  |
|----------------------|----------------------|-------------------------------------------------|
| `v1.IsNotFound`      | `ErrorNotFound`      | 资源不存在（沙箱、提供方等） |
| `v1.IsAlreadyExists` | `ErrorAlreadyExists` | 同名资源已存在           |
| `v1.IsConflict`      | `ErrorConflict`      | 乐观并发冲突或无效状态转换 |
| `v1.IsUnavailable`   | `ErrorUnavailable`   | 网关不可达或客户端已关闭       |
| `v1.IsUnimplemented` | `ErrorUnimplemented` | 网关版本不支持该操作   |
| `v1.IsPermissionDenied` | `ErrorPermissionDenied` | 权限不足                  |
| `v1.IsInvalidArgument` | `ErrorInvalidArgument` | 请求参数无效                  |
| `v1.IsDeadlineExceeded` | `ErrorDeadlineExceeded` | 操作超时                       |
| `v1.IsCancelled`     | `ErrorCancelled`     | 操作被取消（上下文取消）   |

对于 `ErrorInternal`（服务端错误），没有便捷谓词。通过 `Code` 字段直接匹配：

```go
var se *v1.StatusError
if errors.As(err, &se) && se.Code == v1.ErrorInternal {
    fmt.Println("Internal server error:", se.Message)
}
```

## 常见模式

### 未找到

优雅处理缺失的资源：

```go
sb, err := client.Sandboxes().Get(ctx, "default", "my-sandbox")
if v1.IsNotFound(err) {
    fmt.Println("Sandbox does not exist, creating...")
    sb, err = client.Sandboxes().Create(ctx, "default", "my-sandbox", &v1.SandboxSpec{})
}
if err != nil {
    log.Fatal(err)
}
```

### 已存在

防止重复创建：

```go
_, err := client.Providers().Create(ctx, "default", &v1.Provider{
    Name: "openai",
    Type: "openai",
})
if v1.IsAlreadyExists(err) {
    fmt.Println("Provider already registered, skipping")
} else if err != nil {
    log.Fatal(err)
}
```

或者，使用 `Ensure` 进行幂等注册：

```go
// Create-or-update in a single call
provider, err := client.Providers().Ensure(ctx, "default", &v1.Provider{
    Name: "openai",
    Type: "openai",
})
```

### 冲突（乐观并发）

当两个客户端并发修改同一资源时，第二次写入会收到冲突错误。通过重新读取资源来重试：

```go
sb, _ := client.Sandboxes().Get(ctx, "default", "my-sandbox")

result, err := client.Sandboxes().AttachProvider(ctx,
    "default", "my-sandbox", "openai", sb.ResourceVersion,
)
if v1.IsConflict(err) {
    // Another client modified the sandbox — re-read and retry
    sb, _ = client.Sandboxes().Get(ctx, "default", "my-sandbox")
    result, err = client.Sandboxes().AttachProvider(ctx,
        "default", "my-sandbox", "openai", sb.ResourceVersion,
    )
}
if err != nil {
    log.Fatal(err)
}
```

### 不可用

处理网关连接问题：

```go
sb, err := client.Sandboxes().Get(ctx, "default", "my-sandbox")
if v1.IsUnavailable(err) {
    fmt.Println("Gateway is not reachable, check connection")
    // Implement retry with backoff
}
```

### 未实现

优雅检测不支持的操作：

```go
_, err := client.Sandboxes().GetLogs(ctx, "default", "my-sandbox")
if v1.IsUnimplemented(err) {
    fmt.Println("Log retrieval not supported by this gateway version")
}
```

## 退避重试

对于 `Unavailable` 或 `DeadlineExceeded` 等瞬态错误，仅在操作幂等或携带幂等/并发令牌时使用指数退避。超时不能证明变更请求未被应用。

```go
func withRetry(ctx context.Context, maxAttempts int, fn func() error) error {
    backoff := 100 * time.Millisecond

    for attempt := 0; attempt < maxAttempts; attempt++ {
        err := fn()
        if err == nil {
            return nil
        }

        // Only retry transient errors
        if !v1.IsUnavailable(err) && !v1.IsDeadlineExceeded(err) {
            return err
        }

        // Add jitter to prevent thundering herd after outages
        jitter := time.Duration(rand.Int63n(int64(backoff) / 2))

        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(backoff + jitter):
            backoff *= 2
        }
    }
    return fmt.Errorf("exhausted %d retry attempts", maxAttempts)
}
```

用法：

```go
var sb *v1.Sandbox
err := withRetry(ctx, 3, func() error {
    var e error
    sb, e = client.Sandboxes().Get(ctx, "default", "my-sandbox")
    return e
})
```

## 上下文取消

所有 SDK 方法都接受 `context.Context`。使用上下文截止时间和取消来控制超时：

```go
// 5-second timeout for a single call
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()

sb, err := client.Sandboxes().Get(ctx, "default", "my-sandbox")
if v1.IsDeadlineExceeded(err) {
    fmt.Println("Request timed out")
}
```

另见：[测试](testing.md)，了解 fake 客户端如何返回相同的错误码。