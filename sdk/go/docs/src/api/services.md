# Services

访问器：`client.Services()`

暴露、检查和管理附加到沙箱的网络服务。服务通过受管端点，为沙箱内运行的端口提供外部访问。

## 暴露

将沙箱中的端口暴露为命名服务端点。将 `domain` 设为 `true`，可为该服务分配可 DNS 路由的域名。

```go
endpoint, err := client.Services().Expose(ctx, "default", "my-sandbox", "web", 8080, true)
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Service available at: %s\n", endpoint.URL)
```

## 列表

`List` 返回已暴露服务的惰性分页器。用 `ListAll` 收集所有页。

```go
services, err := client.Services().ListAll(ctx, "default", "my-sandbox")
if err != nil {
    log.Fatal(err)
}
for _, svc := range services {
    fmt.Printf("  %s -> port %d (%s)\n", svc.Name, svc.TargetPort, svc.URL)
}

// Platform Admin only: list services across all workspaces
allServices, err := client.Services().ListAll(ctx, "", "", v1.ListOptions{
    AllWorkspaces: true,
})
```

## 删除

移除已暴露的服务。底层沙箱端口在内部仍可访问，但不再能通过服务端点访问。

```go
deletion, err := client.Services().Delete(ctx, "default", "my-sandbox", "web")
if err != nil {
    log.Fatal(err)
}
```

另见：[错误处理](../error-handling.md)、[测试](../testing.md)