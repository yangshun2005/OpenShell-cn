# Fake

包：`openshell/v1/fake`

fake 包提供所有 SDK 客户端接口的内存假实现，供使用方测试套件使用。它遵循 `client-go/kubernetes/fake` 模式：内存存储、watch 事件广播，并对等价的错误条件（`NotFound`、`AlreadyExists`、`Unavailable`、`Unimplemented`）返回匹配的 `StatusError` 错误码。

## 快速开始

```go
import "github.com/NVIDIA/OpenShell/sdk/go/openshell/v1/fake"

func TestSandboxLifecycle(t *testing.T) {
    client := fake.NewClient()
    defer client.Close()

    ctx := context.Background()

    sb, err := client.Sandboxes().Create(ctx, "default", "my-sandbox", &v1.SandboxSpec{})
    require.NoError(t, err)
    assert.Equal(t, types.SandboxProvisioning, sb.Status.Phase)

    sb, err = client.Sandboxes().WaitReady(ctx, "default", "my-sandbox")
    require.NoError(t, err)
    assert.Equal(t, types.SandboxReady, sb.Status.Phase)

    deletion, err := client.Sandboxes().Delete(ctx, "default", "my-sandbox")
    require.NoError(t, err)
    assert.Equal(t, v1.DeletionCompleted, deletion.Outcome)
}
```

## 创建客户端

```go
func NewClient(opts ...ClientOption) *Client
```

返回一个 fake 客户端，实现 `v1.ClientInterface`，所有子客户端均已接好。默认健康结果为健康。使用选项自定义初始状态：

```go
client := fake.NewClient(
    fake.WithHealthResult(&types.HealthResult{Healthy: false}),
    fake.WithCurrentUser(&types.CurrentUser{Subject: "test-user"}),
    fake.WithGatewayInfo(&types.GatewayInfo{Version: "1.0.0"}),
)
```

## 预填充状态

将种子对象直接写入 fake 存储，用于测试准备：

```go
client := fake.NewClient()

client.AddSandbox("default", &types.Sandbox{
    Name:   "pre-existing",
    Status: types.SandboxStatus{Phase: types.SandboxReady},
})

client.AddSandboxTemplate("default", &types.SandboxWorkloadTemplate{
    Name: "gpu-kata",
    Spec: types.SandboxWorkloadTemplateSpec{
        Workload: &types.SandboxWorkloadConfig{Image: "python:3.12"},
    },
})

client.AddProvider("default", &types.Provider{
    Name: "my-provider",
    Spec: types.ProviderSpec{Type: "docker"},
})

client.AddWorkspace(&types.Workspace{Name: "staging"})
client.AddMember("staging", &types.WorkspaceMember{
    PrincipalSubject: "subject-123",
    Role:             types.WorkspaceRoleAdmin,
})
```

所有 `Add*` 方法都会深拷贝其参数；插入后修改输入不会影响已存储的对象。

## 子客户端覆盖范围

fake 客户端实现了 `v1.ClientInterface` 中所有稳定访问器，以及 `SandboxTemplates()` 等附加的具体客户端访问器：

| 访问器 | 接口 | 行为 |
|----------|-----------|----------|
| `Sandboxes()` | `SandboxInterface` | 完整 CRUD、Watch、WaitReady |
| `SandboxTemplates()` | `SandboxTemplateInterface` | 完整 CRUD |
| `Providers()` | `ProviderInterface` | 完整 CRUD、Ensure |
| `Workspaces()` | `WorkspaceInterface` | 完整 CRUD、Members |
| `Health()` | `HealthInterface` | 可配置结果 |
| `Policy()` | `PolicyInterface` | List、GetStatus（草稿操作返回 Unimplemented） |
| `Exec()` | `ExecInterface` | 返回 Unimplemented |
| `Files()` | `FileInterface` | 返回 Unimplemented |
| `Services()` | `ServiceInterface` | 返回 Unimplemented |
| `SSH()` | `SSHInterface` | 输入校验后返回 Unimplemented |
| `TCP()` | `TCPInterface` | 输入校验后返回 Unimplemented |
| `Config()` | `ConfigInterface` | 返回 Unimplemented |

## ClientOption

| 构造函数 | 作用 |
|-------------|--------|
| `WithHealthResult(r)` | 设置健康检查返回值 |
| `WithCurrentUser(u)` | 设置当前用户返回值 |
| `WithGatewayInfo(i)` | 设置网关信息返回值 |

## 线程安全

所有操作均可安全地由多个 goroutine 并发使用。`Close` 是幂等的，调用后所有后续操作返回 `Unavailable`。

另见：[测试指南](../testing.md)