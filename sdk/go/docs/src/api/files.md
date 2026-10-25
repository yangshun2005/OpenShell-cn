# 文件

访问器：`client.Files()`

`FileInterface` 预留了上传和下载 API，同时保持沙箱查找和 SSH 会话生命周期行为稳定。独立 SDK 目前不提供 SSH 文件传输通道，因此这两个操作都会在执行本地校验或网关 RPC 之前返回 `v1.ErrTransportNotAvailable`。

## 上传

以编程方式检测传输通道是否可用：

```go
err := client.Files().Upload(ctx, "default", "sandbox-123", "./data/config.yaml", "/app/config.yaml")
if errors.Is(err, v1.ErrTransportNotAvailable) {
    // Use another transfer mechanism until an SSH transport is available.
} else if err != nil {
    log.Fatal(err)
}
```

## 下载

`Download` 有相同的可用性限制：

```go
err := client.Files().Download(ctx, "default", "sandbox-123", "/app/output.log", "./output.log")
if errors.Is(err, v1.ErrTransportNotAvailable) {
    // Use another transfer mechanism until an SSH transport is available.
} else if err != nil {
    log.Fatal(err)
}
```

另请参阅：[错误处理](../error-handling.md)、[测试](../testing.md)