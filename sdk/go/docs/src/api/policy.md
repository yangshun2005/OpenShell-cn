# Policy

访问器：`client.Policy()`

通过基于草稿的工作流管理沙箱的网络策略。策略需经过草稿、审核和批准周期后才会生效。

## GetDraft

获取沙箱当前的草稿策略。

```go
draft, err := client.Policy().GetDraft(ctx, "default", "my-sandbox")
if err != nil {
    log.Fatal(err)
}
for _, chunk := range draft.Chunks {
    fmt.Printf("Chunk %s: rule=%s, status=%s, confidence=%.1f\n",
        chunk.ID, chunk.RuleName, chunk.Status, chunk.Confidence)
}
```

## ApproveAllDraftChunks

一次性批准所有待处理的草稿块。

```go
result, err := client.Policy().ApproveAllDraftChunks(ctx, "default", "my-sandbox")
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Approved %d chunks (skipped %d), policy version: %d\n",
    result.ChunksApproved, result.ChunksSkipped, result.PolicyVersion)
```

## GetStatus

查看沙箱当前的策略执行状态。

```go
status, err := client.Policy().GetStatus(ctx, "default", "my-sandbox")
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Active version: %d, revision status: %s\n",
    status.ActiveVersion, status.Revision.Status)
```

## List

`List` 返回一个针对单个沙箱策略修订版本的惰性分页器。使用 `ListAll` 收集所有页。

```go
revisions, err := client.Policy().ListAll(ctx, "default", "my-sandbox")
if err != nil {
    log.Fatal(err)
}
for _, rev := range revisions {
    fmt.Printf("Version %d: %s (status: %s)\n",
        rev.Version, rev.CreatedAt, rev.Status)
}
```

## RejectDraftChunk

拒绝指定的草稿块，并提供原因。

```go
err := client.Policy().RejectDraftChunk(ctx, "default", "my-sandbox", "chunk-abc", "Too permissive")
if err != nil {
    log.Fatal(err)
}
```

## EditDraftChunk

在批准前修改草稿块中提议的规则。

```go
err := client.Policy().EditDraftChunk(ctx, "default", "my-sandbox", "chunk-abc", &v1.NetworkPolicyRule{
    Name: "allow-api",
    Endpoints: []v1.PolicyNetworkEndpoint{{
        Host:     "api.example.com",
        Port:     443,
        Protocol: "tcp",
    }},
})
if err != nil {
    log.Fatal(err)
}
```

## 追加允许或拒绝规则

使用 `client.Config().Update` 配合 `AddAllowRules` 或 `AddDenyRules`，向已有的基础策略端点追加请求匹配器。两种操作都需要一个 `L7RuleTarget`，用于指定规则名、主机、所有受影响的端口以及该规则管辖的所有二进制文件。若缺少作用域或作用域与当前策略不一致，网关会拒绝该请求，且不会应用合并批次。

```go
result, err := client.Config().Update(ctx, "default", &v1.ConfigUpdate{
    Name: "my-sandbox",
    MergeOperations: []v1.PolicyMergeOperation{{
        AddAllowRules: &v1.AddAllowRules{
            Target: &v1.L7RuleTarget{
                RuleName: "api",
                Host: "api.example.com",
                Ports: []uint32{443, 8443},
                Binaries: []v1.PolicyNetworkBinary{
                    {Path: "/usr/bin/curl"},
                    {Path: "/usr/bin/wget"},
                },
            },
            Rules: []v1.L7Rule{{
                Allow: &v1.L7Allow{Method: "POST", Path: "/admin"},
            }},
        },
    }},
})
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Policy version: %d\n", result.Version)
```

此示例明确允许两个二进制文件在两个端口上发起新请求。若规则允许任意二进制文件，请设置 `AnyBinary: true` 并省略 `Binaries`。两个字段都省略是无效的；SDK 绝不会推断出任意二进制文件权限。

`Target.Path` 用于选择已有的端点路径，与追加的请求匹配器路径相互独立。当规则名、主机和端口已能唯一确定端点时，将其留为 `nil`。若要选择带特定路径的端点，传入指向该路径的指针；指向空字符串的指针会选择不带路径选择器的端点。使用 `AddDenyRules` 并采用相同的 target 结构和 `DenyRules` 载荷即可追加拒绝匹配器。

另见：[错误处理](../error-handling.md)、[测试](../testing.md)