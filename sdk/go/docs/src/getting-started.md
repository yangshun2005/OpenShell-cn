# 快速开始

本指南带你完成安装 OpenShell Go SDK、连接网关、创建沙箱、运行命令和清理的全过程。5 分钟内即可跑通。

## 前置条件

- Go 1.25.13 或更高版本
- 可访问的 OpenShell 网关（地址和认证令牌）

## 安装

将 SDK 添加到你的 Go 模块：

```bash
go get github.com/NVIDIA/OpenShell/sdk/go@latest
```

## 连接网关

提供网关地址和认证凭据来创建客户端：

```go
package main

import (
    "context"
    "fmt"
    "log"

    v1 "github.com/NVIDIA/OpenShell/sdk/go/openshell/v1"
)

func main() {
    client, err := v1.NewClient(v1.Config{
        Address: "gateway.example.com:443",
        Auth:    v1.StaticToken("my-token"),
    })
    if err != nil {
        log.Fatal(err)
    }
    defer client.Close()
```

生产环境应从环境变量读取令牌，不要硬编码：
`v1.StaticToken(os.Getenv("OPENSHELL_TOKEN"))`

`Config` 结构体还接受 TLS 配置和重试策略等可选字段。开发时连接本地无 TLS 网关，可使用 `v1.NoAuth()` 并设置 TLS 跳过验证。

## 检查网关健康状态

验证网关是否可达：

```go
    ctx := context.Background()

    health, err := client.Health().Check(ctx)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Gateway healthy: %v\n", health.Healthy)
```

## 创建沙箱

使用 Python 镜像创建沙箱：

```go
    sandbox, err := client.Sandboxes().Create(ctx, "default", "my-sandbox", &v1.SandboxSpec{
        Template: &v1.SandboxTemplate{Image: "python:3.12"},
        Environment: map[string]string{"LANG": "en_US.UTF-8"},
    })
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Created sandbox: %s\n", sandbox.Name)
```

## 等待沙箱就绪

沙箱需要一段时间来准备。用 `WaitReady` 阻塞等待，直到沙箱可以接受命令：

```go
    sandbox, err = client.Sandboxes().WaitReady(ctx, "default", sandbox.Name)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Sandbox is ready (phase: %s)\n", sandbox.Status.Phase)
```

## 运行命令

在沙箱内执行命令：

```go
    result, err := client.Exec().Run(ctx, "default", sandbox.Name, []string{"echo", "hello from OpenShell"})
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Output: %s", result.Stdout)
    fmt.Printf("Exit code: %d\n", result.ExitCode)
```

长时间运行的命令可用 `Stream` 增量接收输出，或用 `Interactive` 获得类似终端的会话。

## 清理

完成后删除沙箱：

```go
    deletion, err := client.Sandboxes().Delete(ctx, "default", sandbox.Name)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Deletion outcome: %v\n", deletion.Outcome)
}
```

## 后续步骤

- 浏览 [API 概览](api/overview.md) 了解所有可用接口
- 了解 [错误处理](error-handling.md)，用于生产代码
- 使用 fake 客户端为测试套件配置 [测试](testing.md)
- 阅读 [架构](architecture.md) 理解 SDK 设计