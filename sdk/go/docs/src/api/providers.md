# Providers

访问器：`client.Providers()`

注册和管理计算提供方（AI 推理端点）。暴露 [Profiles](profiles.md) 和 [Refresh](refresh.md) 的子客户端。

## Create

向网关注册新提供方。

```go
provider, err := client.Providers().Create(ctx, "default", &v1.Provider{
    Name: "my-openai",
    Type: "openai",
    Spec: v1.ProviderSpec{
        Credentials: map[string]string{
            "api_key": "sk-...",
        },
        Config: map[string]string{
            "base_url": "https://api.openai.com/v1",
        },
    },
})
if err != nil {
    log.Fatal(err)
}
fmt.Println("Created provider:", provider.Name)
```

## Get

按名称获取提供方。

```go
provider, err := client.Providers().Get(ctx, "default", "my-openai")
if err != nil {
    log.Fatal(err)
}
fmt.Println("Provider type:", provider.Type)
```

## List

`List` 返回已注册提供方的惰性分页器。`ListAll` 跟随所有续页令牌；`PageSize` 控制每次请求的数量。

```go
// List all providers
providers, err := client.Providers().ListAll(ctx, "default")
if err != nil {
    log.Fatal(err)
}
for _, p := range providers {
    fmt.Println(p.Name, p.Type)
}

// With a smaller page size
providers, err = client.Providers().ListAll(ctx, "default", v1.ListOptions{
    PageSize: 10,
})

// Platform Admin only: list across all workspaces
allProviders, err := client.Providers().ListAll(ctx, "", v1.ListOptions{
    AllWorkspaces: true,
})
```

## Update

更新现有提供方的配置或凭据。

```go
provider, err := client.Providers().Get(ctx, "default", "my-openai")
if err != nil {
    log.Fatal(err)
}

provider.Spec.Credentials["api_key"] = "sk-new-key"
updated, err := client.Providers().Update(ctx, "default", provider)
if err != nil {
    log.Fatal(err)
}
fmt.Println("Updated provider:", updated.Name)
```

## Delete

按名称删除提供方。

```go
deletion, err := client.Providers().Delete(ctx, "default", "my-openai")
if err != nil {
    log.Fatal(err)
}
```

## Ensure

通过一次幂等调用创建或更新提供方。若同名提供方已存在则更新，否则新建。这是注册提供方的推荐方式，可避免重复注册时的 "already exists" 错误。

```go
provider, err := client.Providers().Ensure(ctx, "default", &v1.Provider{
    Name: "my-openai",
    Type: "openai",
    Spec: v1.ProviderSpec{
        Credentials: map[string]string{
            "api_key": "sk-...",
        },
    },
})
if err != nil {
    log.Fatal(err)
}
fmt.Println("Provider ready:", provider.Name)
```

Ensure 是 SDK 层的便捷方法。它先调用 `GetProvider`，再根据提供方是否已存在调用 `CreateProvider` 或 `UpdateProvider`。

## 子客户端

`ProviderInterface` 暴露两个子客户端访问器，用于相关操作。它们是纯客户端访问器，没有对应的 gRPC 调用。

### Profiles

`client.Providers().Profiles()` 返回 [ProfileInterface](profiles.md)，用于管理提供方类型配置。配置定义了不同提供方类型的模板和默认值。

### Refresh

`client.Providers().Refresh()` 返回 [RefreshInterface](refresh.md)，用于配置凭据刷新策略。用它为凭据会过期的提供方设置自动轮换。

## Provider

`Provider` 类型表示已注册的计算提供方。

| 字段              | 类型                   | 描述                                |
|-------------------|------------------------|--------------------------------------------|
| `ID`              | string                 | 服务器分配的唯一标识符          |
| `Name`            | string                 | 用户选择的名称（每个网关唯一）      |
| `Type`            | string                 | 提供方类型（如 `"openai"`、`"azure"`） |
| `CreatedAt`       | time.Time              | 创建时间戳                      |
| `Labels`          | map[string]string      | 键值元数据标签                  |
| `ResourceVersion` | uint64                 | 乐观并发版本             |
| `Spec`            | ProviderSpec           | 配置和凭据              |

## ProviderSpec

`ProviderSpec` 保存提供方特定的配置和凭据。

| 字段                  | 类型                      | 描述                                     |
|------------------------|---------------------------|-------------------------------------------------|
| `Credentials`          | map[string]string         | 认证凭据（如 API 密钥）     |
| `Config`               | map[string]string         | 提供方特定的配置值          |
| `CredentialExpiresAt`  | map[string]time.Time      | 凭据的过期时间戳           |

精选 Go API 在更新时使用零值 `time.Time` 来清除过期时间。因此，它无法在此 map 中表示 protobuf 最小时间戳（`0001-01-01T00:00:00Z`）。若需要该精确时间戳，请使用原始 protobuf API。其他基于指针的时间戳字段会保留它。

另见：[Profiles](profiles.md)、[Refresh](refresh.md)、[错误处理](../error-handling.md)、[测试](../testing.md)