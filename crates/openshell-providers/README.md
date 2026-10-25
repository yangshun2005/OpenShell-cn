# openshell-providers

为沙箱运行时所需的凭据提供 Provider 发现与规范化。

网关持久化 Provider 记录。沙箱 supervisor 从网关获取解析后的 Provider 环境，并将凭据注入 agent 子进程。本 crate 将 Provider 特定的发现与规范化逻辑从 CLI 和网关控制流中分离出来。

## 职责

- 从环境变量和已知配置文件发现本地凭据。
- 将发现的数据规范化为 Provider 记录。
- 将 Provider 特定的解析规则保留在 Provider 模块中。
- 使用共享的 authored-YAML 兼容边界进行 profile 导入和导出：拒绝重复键和 null 对象且不做强制转换，保留可选 null，并为 YAML 1.1 读取器保留字符串类型。
- 避免记录凭据值。

## 非职责

- 持久化 Provider 记录。
- 授权 Provider CRUD 操作。
- 将凭据注入沙箱子进程。
- 路由推理请求。

这些由网关、沙箱 supervisor 和路由器负责。

## 安全说明

Provider 数据通常包含 API 密钥、bearer token 或本地账户配置。发现代码应返回结构化值，不得打印或追踪密钥。展示 Provider 数据的调用方必须默认对敏感字段进行脱敏。