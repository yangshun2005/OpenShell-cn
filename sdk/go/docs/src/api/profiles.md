# Profiles

访问器：`client.Providers().Profiles()`

管理 AI 模型提供商的提供商 Profile。Profile 定义 OpenAI、Anthropic 或自定义端点等提供商的连接信息、凭据和模型映射。

## List

`List` 返回可见提供商 Profile 的惰性分页器。用 `ListAll` 收集所有页。

```go
profiles, err := client.Providers().Profiles().ListAll(ctx, "default")
if err != nil {
    log.Fatal(err)
}
for _, p := range profiles {
    fmt.Printf("Profile: %s (%s)\n", p.ID, p.DisplayName)
}
```

## Import

从配置项导入一个或多个提供商 Profile。

```go
result, err := client.Providers().Profiles().Import(ctx, "default", []v1.ProfileImportItem{
    {
        Profile: v1.ProviderProfile{
            DisplayName: "OpenAI",
            Category:    v1.ProfileCategoryInference,
        },
        Source: "manual",
    },
})
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Imported: %v\n", result.Imported)
```

## Update

通过资源版本进行乐观并发控制来更新 Profile。

```go
profile, err := client.Providers().Profiles().Get(ctx, "default", "profile-id")
if err != nil {
    log.Fatal(err)
}

profile.DisplayName = "Updated Provider"
result, err := client.Providers().Profiles().Update(
    ctx, "default",
    profile.ID,
    profile.ResourceVersion,
    v1.ProfileImportItem{Profile: *profile, Source: "manual"},
)
if err != nil {
    // 冲突错误见 [错误处理](../error-handling.md)
    log.Fatal(err)
}
```

## Lint

校验 Profile 配置但不持久化。适合预检。

```go
items := []v1.ProfileImportItem{
    {
        Profile: v1.ProviderProfile{DisplayName: "Test"},
        Source:  "manual",
    },
}
result, err := client.Providers().Profiles().Lint(ctx, "default", items)
if err != nil {
    log.Fatal(err)
}
for _, d := range result.Diagnostics {
    fmt.Printf("[%s] %s: %s\n", d.Severity, d.Field, d.Message)
}
```

另见：[错误处理](../error-handling.md)、[测试](../testing.md)