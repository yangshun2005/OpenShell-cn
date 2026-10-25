# TCP

访问器：`client.TCP()`

通过双向 gRPC 流将 TCP 连接转发到沙箱端口。

## Forward

向沙箱内指定端口打开双向 TCP 转发流。返回一个 `io.ReadWriteCloser`，通过 gRPC 流在调用方与沙箱端口之间代理数据。

```go
conn, err := client.TCP().Forward(ctx, "default", "sandbox-123", 8080)
if err != nil {
    log.Fatal(err)
}
defer conn.Close()

// Exchange protocol bytes with the service through the tunnel.
_, err = conn.Write([]byte("ping\n"))
if err != nil {
    log.Fatal(err)
}

buf := make([]byte, 4096)
n, err := conn.Read(buf)
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Response: %s\n", buf[:n])
```

## Listen

绑定一个本地地址，将每个连接转发到沙箱端口。返回的 `ForwardListener` 拥有自己的 accept 循环和所有桥接 goroutine；它是一个带 `Addr` 和 `Close` 的生命周期句柄，不是 `net.Listener`。

```go
forward, err := client.TCP().Listen(ctx, "default", "sandbox-123", 8080, 0)
if err != nil {
    log.Fatal(err)
}
defer forward.Close()

conn, err := net.Dial("tcp", forward.Addr().String())
if err != nil {
    log.Fatal(err)
}
defer conn.Close()
```

不要将 `ForwardListener` 传给 `http.Serve`。请用客户端拨号其地址，以访问沙箱服务暴露的协议。

TCP 转发比 [SSH 隧道](ssh.md) 更底层。需要直接访问端口且不想承担 SSH 会话开销时，使用 TCP 转发。

另见：[SSH 隧道](ssh.md)、[错误处理](../error-handling.md)