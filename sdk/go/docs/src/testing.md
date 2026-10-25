# 测试

SDK 附带一个 `fake` 包，提供所有客户端接口的内存实现。在测试套件中使用它，无需真实网关即可测试 SDK 交互。

```go
import "github.com/NVIDIA/OpenShell/sdk/go/openshell/v1/fake"
```

fake 客户端遵循与 `k8s.io/client-go/kubernetes/fake` 相同的模式：维护内存存储、支持 watch 事件广播，并返回与真实客户端相同的 `StatusError` 错误码。

## 创建 Fake 客户端

```go
func TestMyOperator(t *testing.T) {
    client := fake.NewClient()
    defer client.Close()

    ctx := context.Background()

    // Use client exactly like the real SDK
    sb, err := client.Sandboxes().Create(ctx, "default", "test-sandbox", &v1.SandboxSpec{})
    require.NoError(t, err)
    assert.Equal(t, "Provisioning", string(sb.Status.Phase))
}
```

返回的 `*fake.Client` 实现了 `v1.ClientInterface`，可传入任何接受该接口的位置。

## 种子数据

在测试运行前预填充 fake 客户端。种子资源可立即通过 `Get` 和 `List` 获取，无需经过 `Create`。

### AddSandbox

```go
client := fake.NewClient()

// Pre-seed a sandbox that already exists
client.AddSandbox(&types.Sandbox{
    Name: "existing-sandbox",
    Status: types.SandboxStatus{
        Phase: types.SandboxReady,
    },
    ResourceVersion: 5,
})

// Now Get returns it immediately
sb, err := client.Sandboxes().Get(ctx, "default", "existing-sandbox")
// sb.Status.Phase == "Ready"
```

### AddProvider

```go
client := fake.NewClient()

// Pre-seed a provider
client.AddProvider(&types.Provider{
    Name: "my-openai",
    Type: "openai",
    Spec: types.ProviderSpec{
        Credentials: map[string]string{
            "api_key": "sk-test-key",
        },
    },
})

// List returns the seeded provider
providers, _ := client.Providers().ListAll(ctx, "default")
// len(providers) == 1
```

## Sandbox 生命周期

fake 客户端实现了完整的 sandbox 生命周期。创建的 sandbox 初始为 `Provisioning` 阶段。调用 `WaitReady` 会同步将其转换为 `Ready`。

```go
client := fake.NewClient()
ctx := context.Background()

// Create starts in Provisioning
sb, err := client.Sandboxes().Create(ctx, "default", "my-sandbox", &v1.SandboxSpec{})
assert.Equal(t, types.SandboxProvisioning, sb.Status.Phase)

// WaitReady transitions to Ready (synchronous in fake)
sb, err = client.Sandboxes().WaitReady(ctx, "default", "my-sandbox")
assert.Equal(t, types.SandboxReady, sb.Status.Phase)

// Delete removes the sandbox
_, err = client.Sandboxes().Delete(ctx, "default", "my-sandbox")
assert.NoError(t, err)

// Get after delete returns NotFound
_, err = client.Sandboxes().Get(ctx, "default", "my-sandbox")
assert.True(t, v1.IsNotFound(err))
```

## Watch 事件

资源变更时，fake 客户端会广播 watch 事件。使用 watcher 测试事件驱动的代码。

```go
client := fake.NewClient()
ctx := context.Background()

// Start watching before making changes
watcher, err := client.Sandboxes().Watch(ctx, "default", "my-sandbox")
require.NoError(t, err)
defer watcher.Stop()

// Create a sandbox — triggers an ADDED event
client.Sandboxes().Create(ctx, "default", "my-sandbox", &v1.SandboxSpec{})

// Read the event from the channel
event := <-watcher.ResultChan()
assert.Equal(t, types.EventAdded, event.Type)
assert.Equal(t, "my-sandbox", event.Object.Name)
```

### StopOnTerminal

设置 `StopOnTerminal: true` 后，sandbox 进入终态（`Ready` 或 `Error`）时 watcher 会自动关闭。

```go
watcher, err := client.Sandboxes().Watch(ctx, "default", "my-sandbox", v1.WatchOptions{
    StopOnTerminal: true,
})
require.NoError(t, err)

// Create and transition to Ready
client.Sandboxes().Create(ctx, "default", "my-sandbox", &v1.SandboxSpec{})
client.Sandboxes().WaitReady(ctx, "default", "my-sandbox")

// Drain events — channel closes after the Ready event
var events []types.Event[*types.Sandbox]
for ev := range watcher.ResultChan() {
    events = append(events, ev)
}
// Channel is now closed
```

## 健康状态模拟

使用 `WithHealthResult` 模拟不健康或降级的网关。

```go
// Default: healthy gateway
client := fake.NewClient()
result, _ := client.Health().Check(ctx)
// result.Healthy == true, result.Version == "fake"

// Simulate unhealthy gateway
client = fake.NewClient(fake.WithHealthResult(&types.HealthResult{
    Healthy: false,
    Version: "1.2.3",
}))
result, _ = client.Health().Check(ctx)
// result.Healthy == false
```

## 错误行为

fake 客户端返回与真实客户端相同的 `StatusError` 错误码：

| 场景 | 错误码 |
|----------|------------|
| 对不存在的资源调用 `Get` | `ErrorNotFound` |
| 使用重复名称调用 `Create` | `ErrorAlreadyExists` |
| `Close()` 后的任何调用 | `ErrorUnavailable` |
| 未实现的操作（如 `GetLogs`） | `ErrorUnimplemented` |

```go
client := fake.NewClient()
client.Close()

_, err := client.Sandboxes().Get(ctx, "default", "anything")
assert.True(t, v1.IsUnavailable(err))
```

## 并发

fake 客户端的所有操作均可安全并发使用。内部存储使用基于互斥锁的同步机制。因此可在并行测试中从多个 goroutine 安全使用 fake 客户端。

另见：[错误处理](error-handling.md)、[API 参考](api/overview.md)