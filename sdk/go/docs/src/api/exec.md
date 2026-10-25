# Exec

访问器：`client.Exec()`

在沙箱中执行命令，支持三种模式：一次性执行（`Run`）、流式（`Stream`）或交互式终端（`Interactive`）。

## Run

执行命令并将所有输出收集为单个结果。

```go
result, err := client.Exec().Run(ctx, "default", "sbx-123", []string{"ls", "-la"})
if err != nil {
    log.Fatal(err)
}

fmt.Println("Exit code:", result.ExitCode)
fmt.Println("Stdout:", result.Stdout)
fmt.Println("Stderr:", result.Stderr)
```

SDK 收集所有流式事件，组装 stdout/stderr，返回单个 `ExecResult`。

`ExecResult` 包含命令结束后的完整输出：

| 字段       | 类型   | 描述             |
|------------|--------|------------------|
| `Stdout`   | string | 捕获的标准输出   |
| `Stderr`   | string | 捕获的标准错误   |
| `ExitCode` | int    | 进程退出码       |

## Stream

执行命令并在输出块到达时逐块处理。

```go
stream, err := client.Exec().Stream(ctx, "default", "sbx-123", []string{"tail", "-f", "/var/log/app.log"})
if err != nil {
    log.Fatal(err)
}
defer stream.Close()

for {
    chunk, err := stream.Next()
    if err == io.EOF {
        break
    }
    if err != nil {
        log.Fatal(err)
    }

    if chunk.Stream == v1.StreamStdout {
        fmt.Print(string(chunk.Data))
    }
}

exitCode, err := stream.ExitCode()
if err != nil {
    log.Fatal(err)
}
fmt.Println("Exited with:", exitCode)
```

## Interactive

与命令建立双向终端会话。

```go
session, err := client.Exec().Interactive(ctx, "default", "sbx-123", []string{"/bin/bash"}, 80, 24)
if err != nil {
    log.Fatal(err)
}
defer session.Close()

// Send input
_, err = session.Write([]byte("echo hello\n"))
if err != nil {
    log.Fatal(err)
}

// Read output
buf := make([]byte, 4096)
n, err := session.Read(buf)
if err != nil {
    log.Fatal(err)
}
fmt.Print(string(buf[:n]))

// Handle terminal resize
if err := session.Resize(120, 40); err != nil {
    log.Fatal(err)
}

// Get exit code after session ends
exitCode, err := session.ExitCode()
if err != nil {
    log.Fatal(err)
}
fmt.Println("Exited with:", exitCode)
```

SDK 将双向流封装为 `InteractiveSession`，提供 `Read`/`Write`/`Resize` 方法。

## ExecStream

`ExecStream` 提供命令输出块的迭代器接口。反复调用 `Next()` 可在输出产生时接收。命令结束后，`Next()` 返回 `io.EOF`。

```go
type ExecStream interface {
    Next() (*ExecChunk, error)
    ExitCode() (int, error)
    Close() error
}
```

| 方法       | 描述                                                    |
|------------|---------------------------------------------------------|
| `Next`     | 返回下一个输出块。完成时返回 `io.EOF`。                 |
| `ExitCode` | 返回进程退出码。在 `Next` 返回 `io.EOF` 后调用。        |
| `Close`    | 释放底层流资源。                                        |

## InteractiveSession

`InteractiveSession` 实现 `io.Reader` 和 `io.Writer`，用于与运行中的进程双向通信。适用于终端模拟、REPL 交互或任何需要持续输入的命令。

```go
type InteractiveSession interface {
    Read(p []byte) (int, error)
    Write(p []byte) (int, error)
    Resize(cols, rows uint32) error
    ExitCode() (int, error)
    Close() error
}
```

| 方法       | 描述                                                         |
|------------|--------------------------------------------------------------|
| `Read`     | 从进程读取输出到提供的缓冲区。                               |
| `Write`    | 向进程发送输入。                                             |
| `Resize`   | 更新终端尺寸（列数和行数）。                                 |
| `ExitCode` | 等待最终 gRPC 状态；返回观察到的退出码以及后续的流错误。     |
| `Close`    | 中止 RPC 并释放资源。                                        |

SDK 会话还实现了可选的 `InteractiveSessionControl` 接口，该接口嵌入 `InteractiveSession` 并新增 `CloseWrite() error` 和 `Cancel() error`。现有的自定义实现只需提供上述原始方法。

调用 `CloseInteractiveInput(session)` 可结束 stdin 和 resize 输入而不取消输出。支持时调用 `CloseWrite()`；否则返回 `ErrorUnimplemented` 且不关闭会话。调用 `CancelInteractive(session)` 在支持时调用 `Cancel()`，旧版会话回退到 `Close()`。SDK 的关闭和取消方法具有幂等性。

`Read` 需与等待 `ExitCode` 并发排空；否则有界输出缓冲区可能阻止完成。输入关闭后，写入和 resize 返回 `io.ErrClosedPipe`。请求 EOF 不等同于终端 Ctrl-D 按键。

## ExecChunk

`ExecStream.Next()` 返回的每个块携带一段进程输出及其来源流。

```go
type ExecChunk struct {
    Data   []byte
    Stream StreamType
}
```

| 字段     | 类型       | 描述                                       |
|----------|------------|--------------------------------------------|
| `Data`   | `[]byte`   | 进程的原始输出字节。                       |
| `Stream` | StreamType | `StreamStdout` 或 `StreamStderr`。         |

另见：[错误处理](../error-handling.md)、[测试](../testing.md)