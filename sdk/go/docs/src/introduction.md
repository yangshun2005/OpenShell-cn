# OpenShell Go SDK

OpenShell Go SDK 为 OpenShell 网关 API 提供符合 Go 习惯的客户端。它将底层 gRPC 协议封装在类型化接口之后，让管理沙箱、执行命令、处理 Provider 等操作变得简单直接。

## 核心特性

- **子客户端模式**：单个 `Client` 为每个 API 域（Sandboxes、SandboxTemplates、Exec、Providers、Files、Health、SSH、TCP、Config、Policy、Services）提供类型化访问器
- **简洁类型**：SDK 类型自包含，代码可直接使用 Go 惯用类型，无需额外依赖
- **测试用 Fake 客户端**：完整 `ClientInterface` 的内存实现，无需真实网关即可测试
- **监听与流式传输**：原生支持监听沙箱状态变化和流式输出命令结果
- **类型化错误处理**：提供 `IsNotFound`、`IsAlreadyExists`、`IsConflict` 等函数，精确分类错误

## 从哪里开始

初次使用 SDK，请阅读[快速开始](getting-started.md)指南，其中涵盖安装、连接网关、创建第一个沙箱、运行命令以及清理。

想深入了解 SDK 的结构，请参阅[架构](architecture.md)概览。

## API 参考

每个 SDK 接口都有专门的参考页面，包含方法签名和代码示例。

浏览完整的 [API 概览](api/overview.md)，一览全部 13 个接口。

## 指南

- [错误处理](error-handling.md)：StatusError、类型化错误检查、重试模式
- [测试](testing.md)：Fake 客户端用法、种子数据填充、监听事件测试

## 相关项目

- [**OpenShell**](https://github.com/NVIDIA/OpenShell)（由 NVIDIA 开发）：上游项目，定义了本 SDK 所封装的网关 API 和沙箱运行时。
- [**openshell-sdk-go**](https://github.com/NVIDIA/OpenShell/sdk/go)：本 SDK 在 GitHub 上的源码仓库。
- [**pkg.go.dev**](https://pkg.go.dev/github.com/NVIDIA/OpenShell/sdk/go/openshell/v1)：Go 包文档，包含类型签名和 godoc。