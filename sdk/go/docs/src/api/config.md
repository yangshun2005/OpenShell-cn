# Config

访问器：`client.Config()`

获取并更新沙箱和网关的配置。

## GetSandbox

获取指定沙箱的当前配置。

```go
config, err := client.Config().GetSandbox(ctx, "default", "sandbox-123")
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Sandbox config: policy_version=%d, revision=%d\n",
    config.PolicyVersion, config.ConfigRevision)
```

## GetGateway

获取网关级配置。

```go
config, err := client.Config().GetGateway(ctx)
if err != nil {
    log.Fatal(err)
}
fmt.Printf("Gateway settings revision: %d\n", config.SettingsRevision)
```

## Update

应用配置更新。更新在应用前会先经过校验。

```go
result, err := client.Config().Update(ctx, "default", &v1.ConfigUpdate{
    Name:       "sandbox-123",
    SettingKey:  "idle_timeout",
    SettingValue: &v1.SettingValue{
        Type:      v1.SettingValueString,
        StringVal: "30m",
    },
})
if err != nil {
    // 校验错误参见 [错误处理](../error-handling.md)
    log.Fatal(err)
}
fmt.Printf("Config updated: revision=%d\n", result.SettingsRevision)
```

另请参阅：[错误处理](../error-handling.md)