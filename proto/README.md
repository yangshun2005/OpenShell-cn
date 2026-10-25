<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Protobuf API 约定

本目录定义 OpenShell 的 gRPC 契约。这些约定是公开 protobuf API 设计的权威依据。新增 API 和修改现有 API 时均须遵循；生成的 SDK 命名由这些定义决定。

## 实体引用

- RPC 的主要目标资源使用 `name`。
- 关联资源引用使用该资源的角色名，如 `sandbox`、`provider`、`service` 或 `rule`。
- 规范实体引用不要加 `_name` 后缀，其值本身就是规范名称。描述性值、本地 map 键、实现标签以及非实体引用的配置注册名可以保留后缀，例如 `display_name`、`file_name`、`driver_name`、`runtime_class_name`、`rule_name` 和 `middleware_name`。
- 公开调用方通过规范名称引用实体。不可变 ID 仅保留在认证、持久化、计算驱动等内部边界。

示例：

```proto
message GetSandboxRequest {
  openshell.datamodel.v1.WorkspaceSelector workspace_scope = 2;
  string name = 1;
}

message ExposeServiceRequest {
  openshell.datamodel.v1.WorkspaceSelector workspace_scope = 5;
  string sandbox = 1;
  string name = 2;
}
```

`GetSandboxRequest.name` 标识 RPC 的主要资源。`ExposeServiceRequest.sandbox` 标识关联的 sandbox，`name` 标识被暴露的 service。

## 工作区作用域

- 公开的工作区作用域请求须在其他所有字段之前声明 `workspace_scope`。现有 wire 字段编号无需遵循声明顺序。
- `workspace_scope` 的类型为 `openshell.datamodel.v1.WorkspaceSelector`。
- 在单个工作区中操作的请求要求非空的规范 `WorkspaceSelector.workspace`。`default` 工作区是显式名称，不是省略值时的回退。
- 仅在明确记录并授权跨工作区访问的集合列表 RPC 上接受 `WorkspaceSelector.all_workspaces`。支持的公开集合为 sandboxes、sandbox templates、providers 和 services。
- 主要资源为工作区的请求使用 `name`，不使用 `workspace_scope`。
- 每个允许省略选择器的请求都须记录说明。当前例外为平台 provider-profile 作用域和已认证的 sandbox 引导。

## 字段与消息设计

- 每个 RPC 优先使用专用的请求和响应消息，包括当前为空的请求。
- 绝对时间使用 `google.protobuf.Timestamp`，经过时间使用 `google.protobuf.Duration`。
- 当省略与显式空值含义不同时，使用可选存在性。不要从 protobuf 标量的默认值推断存在性。
- 公开请求字段按语义阅读顺序排列。字段编号保持 wire 身份，因此可能与声明顺序不同。
- 在所属请求上记录对授权敏感的 selector 变体和省略语义。

## Schema 演进

- 字段编号和完全限定消息名视为持久的 wire 身份。
- 删除或重命名字段时，保留其旧编号和源名称。不得将其复用于其他含义。
- 对照公开描述符闭包和持久化存储的 protobuf 闭包审查变更。`openshell-server` 中的 `public_and_durable_schema_inventories_are_complete` 测试负责这两份清单。
- 契约变更后重新生成 Rust、Python、Go 和 TypeScript 绑定。提交变更前运行 `mise run pre-commit`、受影响的 SDK 检查和相关服务端测试。

## Supervisor 主机协议

[`supervisor_host.proto`](supervisor_host.proto) 是网关与其在各 sandbox 间共享的 supervisor 进程之间的私有流：supervisor 接入，网关发送完整分配快照，supervisor 上报其托管内容。这是版本锁定的部署协议，在网关监听器上提供，但仅授权给网关签发的 supervisor 令牌（`auth_mode: "supervisor"`）。`openshell.proto` 不导入它，因此 SDK 生成和 gRPC 反射会排除它；服务端的 schema 清单测试对其显式分类。`SandboxInstanceAssignment.launch_material` 携带 `openshell-supervisor-control` 现有的 JSON sandbox 实例注册信息，并标记为 secret。

## 隔离后端协议

[`isolation_backend.proto`](isolation_backend.proto) 定义后端发现、边界绑定以及可选的挂起/恢复 RPC。[`sandbox_protocol.proto`](sandbox_protocol.proto) 定义绑定边界上的类型化工作负载控制、进程 I/O 和网络中介。两者均为 supervisor 到后端的契约，不是网关 RPC。后端作者可基于这些定义独立生成绑定。协商的 Sandbox Protocol 版本涵盖 gRPC 服务、类型化消息、流式传输和生命周期语义。