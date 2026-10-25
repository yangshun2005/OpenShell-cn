# SSH

访问器：`client.SSH()`

为沙箱创建和管理 SSH 会话。支持直接 SSH 访问和通过 SSH 连接进行 TCP 隧道转发。

## CreateSession

为沙箱创建新的 SSH 会话。返回连接详情，包括主机、端口和认证凭据。

```go
session, err := client.SSH().CreateSession(ctx, "default", "sandbox-123")
if err != nil {
    log.Fatal(err)
}
fmt.Printf("SSH via %s://%s:%d\n", session.GatewayScheme, session.GatewayHost, session.GatewayPort)
```

## RevokeSession

撤销活跃的 SSH 会话，立即终止使用该会话的所有连接。

```go
deletion, err := client.SSH().RevokeSession(ctx, "default", session.Token)
if err != nil {
    log.Fatal(err)
}
if deletion.Outcome == v1.DeletionCompleted {
    fmt.Println("Session revoked")
}
```

## Tunnel

创建 SSH 隧道，为沙箱内端口提供双向流。该操作将 SSH 会话创建与 TCP 转发合并为一步，返回隧道的 `io.ReadWriteCloser`。

```go
tunnel, err := client.SSH().Tunnel(ctx, "default", "my-sandbox", 8080)
if err != nil {
    log.Fatal(err)
}
defer tunnel.Close()

// Use the tunnel as a regular io.ReadWriteCloser
_, err = tunnel.Write([]byte("GET / HTTP/1.0\r\n\r\n"))
if err != nil {
    log.Fatal(err)
}
```

另见：[TCP 转发](tcp.md)、[错误处理](../error-handling.md)