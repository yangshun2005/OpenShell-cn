# 健康检查

访问器：`client.Health()`

检查所连接的 OpenShell 网关的健康状态。

## 检查

对网关执行健康检查，返回整体状态和组件级详情。

```go
result, err := client.Health().Check(ctx)
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Gateway healthy: %v\n", result.Healthy)
```

适用于就绪探针，或在执行其他操作前验证连通性。

```go
// Quick connectivity check before starting work
if result, err := client.Health().Check(ctx); err != nil {
    log.Fatalf("Gateway unreachable: %v", err)
} else if !result.Healthy {
    log.Fatal("Gateway is not healthy")
}
```

另见：[错误处理](../error-handling.md)