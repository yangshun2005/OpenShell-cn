# Refresh

访问器：`client.Providers().Refresh()`

管理提供商配置文件的凭据刷新计划。配置 API 密钥的自动轮换并监控刷新状态。

## GetStatus

检查特定提供商凭据的刷新状态。

```go
statuses, err := client.Providers().Refresh().GetStatus(ctx, "default", "openai", "default")
if err != nil {
    log.Fatal(err)
}
for _, s := range statuses {
    fmt.Printf("Key: %s, Last refresh: %s, Next: %s\n",
        s.CredentialKey, s.LastRefreshAt, s.NextRefreshAt)
}
```

## Configure

使用指定的策略和材料设置自动凭据刷新。

```go
status, err := client.Providers().Refresh().Configure(ctx, "default", &v1.RefreshConfig{
    Provider:      "openai",
    CredentialKey:  "default",
    Strategy:       v1.RefreshStrategyOAuth2ClientCredentials,
    Material: map[string]string{
        "client_id":     "my-client-id",
        "client_secret": "my-client-secret",
        "token_url":     "https://oauth.example.com/token",
    },
    SecretMaterialKeys: []string{"client_secret"},
})
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Refresh configured, next rotation: %s\n", status.NextRefreshAt)
```

## Rotate

手动触发立即凭据轮换。

```go
status, err := client.Providers().Refresh().Rotate(ctx, "default", "openai", "default")
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Rotated successfully at %s\n", status.LastRefreshAt)
```

另请参阅：[错误处理](../error-handling.md)、[配置文件](profiles.md)