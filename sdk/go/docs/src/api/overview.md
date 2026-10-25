# API 概览

OpenShell Go SDK 通过子客户端模式暴露类型化接口。
每个接口通过具体 `Client` 上的类型化访问器获取。

## 接口摘要

### 顶层接口

| 接口 | 访问器 | 描述 |
|-----------|----------|-------------|
| [SandboxInterface](sandboxes.md) | `client.Sandboxes()` | 创建、管理和监听沙箱生命周期 |
| [SandboxTemplateInterface](sandbox-templates.md) | `client.SandboxTemplates()` | 管理可复用的沙箱模板 |
| [ExecInterface](exec.md) | `client.Exec()` | 运行命令、流式输出、交互式会话 |
| [ProviderInterface](providers.md) | `client.Providers()` | 管理计算提供商及其生命周期 |
| [ServiceInterface](services.md) | `client.Services()` | 在沙箱内暴露和管理 HTTP 服务 |
| [FileInterface](files.md) | `client.Files()` | 向沙箱上传文件或从沙箱下载文件 |
| [HealthInterface](health.md) | `client.Health()` | 检查网关健康状态 |
| [SSHInterface](ssh.md) | `client.SSH()` | 创建到沙箱的 SSH 会话和隧道 |
| [TCPInterface](tcp.md) | `client.TCP()` | 将 TCP 连接转发到沙箱端口 |
| [ConfigInterface](config.md) | `client.Config()` | 读取和更新沙箱及网关配置 |
| [PolicyInterface](policy.md) | `client.Policy()` | 管理策略建议草稿 |

### 便捷包

| 包 | 入口 | 描述 |
|---------|-------------|-------------|
| [gateway](gateway.md) | `gateway.NewClient(name)` | 读取 CLI 网关配置并自动装配客户端 |

### 提供商子接口

通过 `client.Providers()` 访问：

| 接口 | 访问器 | 描述 |
|-----------|----------|-------------|
| [ProfileInterface](profiles.md) | `client.Providers().Profiles()` | 管理提供商类型配置 |
| [RefreshInterface](refresh.md) | `client.Providers().Refresh()` | 配置凭证刷新策略 |

## 接口

每个接口都有参考页面，包含方法签名和用法示例：

- **[Sandboxes](sandboxes.md)**：创建沙箱、等待就绪、监听状态变化、管理提供商、获取日志。
- **[Sandbox Templates](sandbox-templates.md)**：创建可复用的工作负载模板并从中启动沙箱。
- **[Exec](exec.md)**：以一次性、流式或交互模式执行命令。
- **[Providers](providers.md)**：注册和管理计算提供商。包含配置和凭证刷新的子客户端。
- **[Services](services.md)**：在沙箱内暴露和管理 HTTP 服务。
- **[Files](files.md)**：向沙箱上传文件或从沙箱下载文件。
- **[Health](health.md)**：检查网关健康状态。
- **[SSH](ssh.md)**：创建到沙箱的 SSH 会话和隧道。
- **[TCP](tcp.md)**：将 TCP 连接转发到沙箱端口。
- **[Config](config.md)**：读取和更新沙箱及网关配置。
- **[Policy](policy.md)**：管理策略建议草稿。
- **[Profiles](profiles.md)**：管理提供商类型配置（通过 `client.Providers().Profiles()`）。
- **[Refresh](refresh.md)**：配置凭证刷新策略（通过 `client.Providers().Refresh()`）。

## 通用模式

SDK 方法遵循以下约定：

- 网络操作接收 `context.Context`；列表构造函数返回分页器，
  每次 `NextPage` 调用接收请求上下文
- 可能失败的方法返回 `(result, error)`
- 列表方法接收可变选项，`ListAll` 是显式的全量路径
- 来自网关的错误携带带类型化代码的 `StatusError`（参见[错误处理](../error-handling.md)）