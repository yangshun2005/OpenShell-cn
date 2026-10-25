# openshell-core

OpenShell 各 crate 共享的类型、常量、配置和辅助函数。

## 隔离注册

`isolation_registration` 负责运维注册表的后端名称、字节流端点和共享校验。其可失败转换将注册信息经由 compute-driver protobuf 传递，再写回 supervisor 的 JSON 启动文件。这些路由不包含凭据或驱动方持有的启动坐标。

## 对象元数据

顶层面向用户的对象采用 Kubernetes 风格的元数据约定。元数据结构提供：

- 服务器生成的稳定 ID。
- 人类可读名称。
- 创建时间戳。
- 用于过滤和自动化的可选标签。

处理对象元数据的代码应使用 `openshell_core::metadata` 中的 trait，而非直接访问 protobuf 字段：

```rust
use openshell_core::{ObjectId, ObjectLabels, ObjectName};

let id = sandbox.object_id();
let name = sandbox.object_name();
let labels = sandbox.object_labels();
```

Trait 方法必须容忍元数据缺失，返回安全的空值而非 panic。

## 标签规则

标签遵循 Kubernetes 风格的键值约定：

- 键可包含可选的 DNS 前缀，后跟 `/`。
- 名称仅限字母数字字符及 `-`、`_`、`.`。
- 值使用相同字符集，可为空。
- 选择器使用逗号分隔的 `key=value` 对，采用 AND 语义。

在 API 入口处校验标签，然后再持久化对象。

## 推理配置

提供商推理配置位于此 crate，使 gateway、sandbox 和 router 对提供商默认值保持一致。配置定义：

- 认证头风格。
- 默认上游头。
- 客户端提供的透传头。
- 支持的推理协议形态。

不要在调用方重复提供商特定的推理行为。在此添加共享行为，然后由 gateway、sandbox 和 router 消费。

## 中间件契约

内置 supervisor 中间件标识符、主机选择器匹配和纯配置校验位于 `openshell_core::middleware`。策略准入和 supervisor 运行时消费同一契约，且不会引入策略 crate 对 supervisor 实现的依赖。

## Protobuf 结构体转换

在 `serde_json` 值与 `prost_types::{Struct, Value}` 之间转换时，使用 `openshell_core::proto_struct`。两个转换方向均在此模块中；JSON 到 protobuf 的转换是可失败的，因此调用方无法静默替换无法表示的数字。