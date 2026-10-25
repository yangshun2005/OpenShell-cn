# 沙箱

访问器：`client.Sandboxes()`

管理沙箱生命周期：创建、检查、删除、附加/分离提供者、等待就绪、监听状态变化、获取日志。

## 创建

使用给定名称和规格创建新沙箱。

```go
sb, err := client.Sandboxes().Create(ctx, "default", "my-sandbox", &v1.SandboxSpec{
    Template: &v1.SandboxTemplate{
        Image: "nvcr.io/nvidia/openshell:latest",
    },
    Providers: []string{"openai"},
}, v1.WithLabels(map[string]string{
    "team": "platform",
}), v1.WithServiceExposures(v1.ServiceExposure{TargetPort: 8080}))
fmt.Println(sb.ServiceURLs[""])
```

创建时服务暴露会向沙箱注册回环 HTTP 端点。`Service` 留空表示未命名端点，设置则创建命名端点。沙箱就绪后开始路由。`ServiceURLs` 返回按服务名索引的路由 URL；空键为未命名端点。

设置 `GPU: true` 请求当前驱动的默认 GPU 分配。需要指定 GPU 数量时设置 `GPUCount`；非 nil 的 `GPUCount` 同时隐含 `GPU`。

## 从模板创建

从可复用的沙箱工作负载模板创建新沙箱。模板提供镜像、环境、资源、驱动配置等工作负载字段。创建请求提供提供者、策略、命令、TTY 等治理字段。

```go
sb, err := client.CreateSandboxFromTemplate(ctx,
    "default",
    "my-sandbox",
    "gpu-kata",
    &v1.SandboxSpec{
        Providers: []string{"openai"},
        Policy:    policy,
    },
    v1.WithLabels(map[string]string{"team": "platform"}),
)
```

模板 CRUD 参见 [沙箱模板](sandbox-templates.md)。

## 获取

按名称获取沙箱。

```go
sb, err := client.Sandboxes().Get(ctx, "default", "my-sandbox")
fmt.Println(sb.Status.Phase) // "Ready", "Provisioning", etc.
```

### 工具服务器连接

`sb.Status.EndpointStatuses` 以单条记录展示每个已配置的工具服务器端点及其最近接受的网络结果。工具服务器连接失败时沙箱仍可为 `Ready`；端点结果不影响生命周期就绪状态。OpenShell 目前观察为 MCP over HTTP 配置的端点。

```go
for _, endpoint := range sb.Status.EndpointStatuses {
    fmt.Printf("%s:%v%s: %s, reported %s\n",
        endpoint.Host, endpoint.Ports, endpoint.Path,
        endpoint.LastResult, endpoint.LastReportedAt)
}
```

`LastResult` 是类型化的 `v1.EndpointResult`。例如，`v1.EndpointTransportFailed` 表示传输在收到 HTTP 响应前失败。`v1.EndpointHTTPResponseReceived` 表示服务器返回了低于 400 的 HTTP 状态；其响应体仍可能包含工具错误。两种结果均不表示当前可用性。

OpenShell 被动观察真实流量，不会使空闲观察过期。展示结果时请保留 `LastReportedAt`：它记录网关接受观察的时间，而非请求发生的时间。保留的证据可能在重置后被接受。`v1.EndpointNoObservedExchange` 的时间戳为空并保留配置的地址，因此端点在观察到流量前或证据失效后仍可识别。

`LastFailure` 在监督器已分类原因时解释 `v1.EndpointCredentialUnavailable` 结果，否则为 `nil`。后续每次观察会替换或清除它。`Code` 和 `Stage` 来自封闭集合，`Provider` 和 `CredentialKey` 指明配置的凭据，任何字段都不携带远程文本。`RequestID` 与工作负载收到的 `x-openshell-request-id` 头匹配，可用于关联被拒绝的请求。

```go
for _, endpoint := range sb.Status.EndpointStatuses {
    if f := endpoint.LastFailure; f != nil && f.Domain == v1.EndpointFailureDomainTokenGrant {
        fmt.Printf("%s/%s: %s at %s (request %s)\n",
            f.Provider, f.CredentialKey, f.Code, f.Stage, f.RequestID)
        if f.RecoveryAction == v1.RefreshRecoveryActionReauthorize {
            // Ask the user to replace the provider's OAuth grant.
        }
    }
}
```

### 令牌授予失败

`sb.Status.LastTokenGrantFailure` 给出任意目标最近一次已分类的授予拒绝，包括没有 MCP 清单条目的 REST 请求。它携带 `DestinationHost`、`DestinationPort` 以及与端点状态相同的受限 `EndpointFailure`。

```go
if observation := sb.Status.LastTokenGrantFailure; observation != nil && observation.Failure != nil {
    f := observation.Failure
    fmt.Printf("%s (port %d): %s, recovery %s, request %s\n",
        observation.DestinationHost, observation.DestinationPort,
        f.Code, f.RecoveryAction, f.RequestID)
}
```

操作前请用 `RequestID` 对照被拒绝的响应：更新的失败可能已替换此记录。同一目标、提供者和凭据的成功授予会清除它；凭据选择前的失败会在下次向该目标成功授予时清除。策略、提供者和监督器会话重置也会清除它。`nil` 表示无保留原因。这是最近一次观察，不是请求历史或服务器健康检查。

## 列表

`List` 构造惰性分页器；`NextPage` 每次获取一页。`PageSize` 控制每次请求，`ListAll` 显式耗尽分页器。

```go
// List all sandboxes
sandboxes, err := client.Sandboxes().ListAll(ctx, "default")

// Process one page at a time
pages, err := client.Sandboxes().List("default", v1.ListOptions{PageSize: 10})
page, err := pages.NextPage(ctx)

// With a page size and label filtering
sandboxes, err := client.Sandboxes().ListAll(ctx, "default", v1.ListOptions{
    PageSize:      10,
    LabelSelector: "team=platform",
})

// Platform Admin only: list across all workspaces
allSandboxes, err := client.Sandboxes().ListAll(ctx, "", v1.ListOptions{
    AllWorkspaces: true,
})
```

## 删除

按名称删除沙箱。

```go
deletion, err := client.Sandboxes().Delete(ctx, "default", "my-sandbox", v1.DeleteOptions{AllowMissing: true})
```

目标不存在时返回 `NotFound`，除非 `AllowMissing` 为 true。检查 `deletion.Outcome`：`DeletionAccepted` 表示清理待完成，`DeletionCompleted` 和 `DeletionAlreadyAbsent` 表示逻辑完成。未知值不表示完成。`deletion.SandboxID` 标识原始沙箱；不要将同名替换误认为该目标。若名称可复用，允许缺失并不能使重试安全。

## AttachProvider

将提供者附加到沙箱。`expectedResourceVersion` 启用乐观并发控制：传入沙箱当前的 `ResourceVersion`，确保自上次读取后无其他客户端修改。

```go
sb, _ := client.Sandboxes().Get(ctx, "default", "my-sandbox")

result, err := client.Sandboxes().AttachProvider(ctx,
    "default", "my-sandbox",
    "openai",
    sb.ResourceVersion,
)
fmt.Println(result.Attached) // true if newly attached
```

## DetachProvider

从沙箱分离提供者。使用与 `AttachProvider` 相同的乐观并发模式。

```go
sb, _ := client.Sandboxes().Get(ctx, "default", "my-sandbox")

result, err := client.Sandboxes().DetachProvider(ctx,
    "default", "my-sandbox",
    "openai",
    sb.ResourceVersion,
)
fmt.Println(result.Detached) // true if actually detached
```

## ListProviders

列出当前附加到沙箱的所有提供者。

```go
providers, err := client.Sandboxes().ListProviders(ctx, "default", "my-sandbox")
for _, p := range providers {
    fmt.Printf("provider: %s (type: %s)\n", p.Name, p.Type)
}
```

## WaitReady

阻塞直到沙箱进入 `Ready` 阶段，返回最终沙箱状态。底层通过 `Get` 以可配置间隔轮询（默认 500ms）。使用上下文取消或截止时间设置超时。

若沙箱进入 `Error` 阶段，WaitReady 立即返回 `StatusError`。

```go
// Wait with a 30-second timeout
ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
defer cancel()

sb, err := client.Sandboxes().WaitReady(ctx, "default", "my-sandbox")
if err != nil {
    log.Fatal(err)
}
fmt.Println(sb.Status.Phase) // "Ready"

// Custom poll interval
sb, err := client.Sandboxes().WaitReady(ctx, "default", "my-sandbox", v1.WaitOptions{
    PollInterval: 2 * time.Second,
})
```

没有专用的 WaitReady RPC。SDK 通过轮询 `GetSandbox` 直到沙箱阶段为 `Ready` 或 `Error` 来实现。

## 监听

打开服务器流式连接，实时观察沙箱状态变化。返回通过通道传递事件的 `WatchInterface[*Sandbox]`。

`WatchInterface[T]` 提供：

- `ResultChan() <-chan Event[T]` 返回事件通道
- `Stop()` 关闭流和通道

每个 `Event[T]` 携带：

- `Type`：`EventAdded`、`EventModified`、`EventDeleted` 或 `EventError` 之一
- `Object`：该时间点的 `*Sandbox`（`EventError` 时为 `nil`）

```go
watcher, err := client.Sandboxes().Watch(ctx, "default", "my-sandbox")
if err != nil {
    log.Fatal(err)
}
defer watcher.Stop()

for event := range watcher.ResultChan() {
    switch event.Type {
    case v1.EventModified:
        fmt.Printf("phase: %s\n", event.Object.Status.Phase)
    case v1.EventDeleted:
        fmt.Println("sandbox deleted")
        return
    case v1.EventError:
        fmt.Println("watch error")
        return
    }
}
```

设置 `StopOnTerminal: true` 使监听器在沙箱进入终态（`Ready` 或 `Error`）后自动关闭。适用于只关心结果的配置流程。

```go
watcher, err := client.Sandboxes().Watch(ctx, "default", "my-sandbox", v1.WatchOptions{
    StopOnTerminal: true,
})
if err != nil {
    log.Fatal(err)
}

for event := range watcher.ResultChan() {
    fmt.Printf("phase: %s\n", event.Object.Status.Phase)
}
// Channel closes after Ready or Error
```

## GetLogs

从沙箱获取日志条目。沙箱按名称查找（SDK 自动将名称解析为内部 ID）。使用函数选项过滤结果。

**可用选项：**

| 选项 | 描述 |
|--------|-------------|
| `WithLogLines(n uint32)` | 返回的最大日志行数 |
| `WithLogSince(t time.Time)` | 仅包含此时间及之后的条目 |
| `WithLogSources(sources ...string)` | 按来源过滤（如 `"gateway"`、`"sandbox"`） |
| `WithLogMinLevel(level string)` | 最低日志级别（如 `"WARN"`、`"ERROR"`） |

```go
// Get the last 50 log lines
result, err := client.Sandboxes().GetLogs(ctx, "default", "my-sandbox",
    v1.WithLogLines(50),
)
for _, line := range result.Lines {
    fmt.Printf("[%s] %s: %s\n", line.Level, line.Source, line.Message)
}

// Filter by source and level since a specific time
result, err := client.Sandboxes().GetLogs(ctx, "default", "my-sandbox",
    v1.WithLogSources("gateway"),
    v1.WithLogMinLevel("WARN"),
    v1.WithLogSince(time.Now().Add(-1*time.Hour)),
)
```

`LogResult` 包含：

- `Lines []LogLine`：按时间顺序排列的日志条目
- `BufferTotal uint32`：服务器缓冲区中可用的总行数

每个 `LogLine` 包含 `Timestamp`、`Level`、`Target`、`Message`、`Source` 和 `Fields`（结构化键值数据）。

另见：[错误处理](../error-handling.md)、[测试](../testing.md)