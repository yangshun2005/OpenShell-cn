# 隔离后端共享类型

本 crate 包含隔离后端 RPC 契约的共享数据和校验逻辑，禁止使用 unsafe 代码（测试中亦然）。

扩展契约见 [`isolation_backend.proto`](../../proto/isolation_backend.proto) 和 [`sandbox_protocol.proto`](../../proto/sandbox_protocol.proto)。supervisor 使用 `openshell-sandbox-backend` 中的具体 RPC 客户端。树内和外部后端均实现生成的服务。Linux 进程与强制执行辅助代码仍为 `openshell-sandbox` 私有。