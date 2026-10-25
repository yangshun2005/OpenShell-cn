---
authors:
  - "@elezar"
state: implemented
links:
  - https://github.com/NVIDIA/OpenShell/issues/1492
  - https://github.com/NVIDIA/OpenShell/pull/1589
  - https://github.com/NVIDIA/OpenShell/pull/1744
  - https://github.com/NVIDIA/OpenShell/pull/1785
  - https://github.com/NVIDIA/OpenShell/pull/1815
  - https://github.com/NVIDIA/OpenShell/pull/1861
  - https://github.com/NVIDIA/OpenShell/pull/2086
  - https://github.com/NVIDIA/OpenShell/pull/2092
---

# RFC 0006 - 驱动配置透传

## 摘要

OpenShell 沙箱创建支持调用方提供 `driver_config`，用于存放不属于可移植公共 API 的计算驱动专属设置。公共 API 在 `SandboxTemplate` 上携带一个以驱动名为键的信封；网关为当前计算驱动选出对应块，并仅将该内部块转发到 `DriverSandboxTemplate.driver_config`。

选中的驱动负责其配置块的嵌套 Schema、校验、安全策略和兼容性行为。网关仅负责稳定的信封、驱动选择，以及调用方提供的 `driver_config` 与网关计算的 `platform_config` 之间的隔离。

## 动机

Issue #1492 指出了沙箱 API 中反复出现的一个缺口：有用的平台特性被阻塞在向 `SandboxSpec`、`SandboxTemplate`、网关转换代码以及每个客户端添加一等字段上。对于含义由单个计算驱动拥有的特性，这种耦合代价过高。

该问题在各驱动中均有体现。Kubernetes 需要 Pod 调度和运行时控制，以及针对集群特定 GPU 栈的资源参数。Docker 和 Podman 需要带本地引擎安全规则的每沙箱挂载配置。GPU 精确设备选择需要驱动原生标识符：Docker 和 Podman 使用 CDI 名称，VM 驱动使用 PCI BDF 或设备 ID。这些细节都不应强制引入可移植的 OpenShell 字段。

同时，OpenShell 仍需要控制平面边界。调用方输入不得合并到网关拥有的 `platform_config`，不得绕过类型化公共字段，不得替换认证或 supervisor 接线，也不得悄然削弱沙箱隔离。驱动拥有的配置路径为驱动提供了暴露平台特性的空间，同时保留了明确的校验归属方。

本 RFC 记录了 PR #1744、#1785、#1815、#1861、#2086 和 #2092 之后的已实现基线。遗留的后续问题保持可见：驱动身份别名、Schema 发现、无匹配警告，以及旧版 Kubernetes 平台资源透传的未来。

## 非目标

- 不为任何特定 GPU 栈添加一等支持。
- 不定义 OpenShell 拥有的 GPU 内存或 GPU 核心共享字段。
- 不将 `driver_config` 作为现有沙箱的动态更新机制。
- 不允许驱动配置覆盖网关计算的 `platform_config` 或类型化公共字段。
- 不对驱动配置键应用通配符匹配。
- 不让网关导入驱动专属配置 Schema。
- 不为每个 OpenShell 资源定义通用扩展机制。Provider 密钥、网关、策略和其他资源需要单独的归属、生命周期、授权、审计和安全分析。

## 提案

### 公共 API

`SandboxTemplate` 有一个调用方提供的 `driver_config` 字段：

```proto
message SandboxTemplate {
  // ... existing typed fields ...

  // Driver-keyed opaque config envelope supplied by the caller.
  // The gateway selects the block matching the active compute driver and
  // forwards only that inner Struct to DriverSandboxTemplate.driver_config.
  // The selected driver owns nested schema validation.
  google.protobuf.Struct driver_config = 11;
}
```

公共值是一个以精确驱动名为键的 JSON/protobuf `Struct` 信封。内置驱动键为：

- `kubernetes`
- `docker`
- `podman`
- `vm`

示例：

```json
{
  "driver_config": {
    "kubernetes": {
      "pod": {
        "runtime_class_name": "kata-containers",
        "node_selector": {
          "pool": "gpu"
        }
      }
    },
    "docker": {
      "cdi_devices": ["nvidia.com/gpu=0"]
    }
  }
}
```

驱动名匹配为精确匹配。`*`、`*/kubernetes` 或 `openshell.ai/*` 等通配符没有特殊含义。未选中的驱动块由网关忽略，且不由网关或选中的驱动校验。

CLI 通过 `--driver-config-json` 暴露该信封。每个驱动块内的嵌套键使用 snake_case。顶层驱动键（如 `kubernetes`）不属于嵌套驱动 Schema。

### 驱动 API

网关将调用方提供的配置与网关计算的平台配置分开：

```proto
message DriverSandboxTemplate {
  // ... existing fields ...

  // Opaque, platform-specific configuration computed by the gateway.
  google.protobuf.Struct platform_config = 11;

  // Caller-provided config for the selected driver only.
  // This is the inner block selected from public SandboxTemplate.driver_config.
  // The selected driver owns nested schema validation.
  google.protobuf.Struct driver_config = 12;
}
```

驱动仅接收其选中的内部配置块，从不接收完整的公共信封。驱动可在校验前将该 `Struct` 解码为本地 Rust 结构体或驱动本地 protobuf 消息，但这些 Schema 保留在选中的驱动 crate 内。网关不得导入 Kubernetes、Docker、Podman、VM 或树外驱动配置类型。

驱动本地的每沙箱配置与网关进程配置（如 `[openshell.drivers.<name>]` TOML 表）不同。进程配置由运维方拥有，包含命名空间、默认镜像、运行时路径、TLS 材料、服务账号和安全门控等值。每沙箱 `driver_config` 仅暴露选中驱动已文档化的调用方安全创建时参数。

### 网关行为

网关仅处理顶层信封：

- 空或未设置的 `driver_config` 等同于无驱动专属配置。
- 请求可包含多个驱动的配置块。
- 网关选择键与选中计算驱动名精确匹配的块。
- 若无匹配块，网关不转发任何驱动配置。
- 匹配块存在时，必须是 JSON 对象 / protobuf Struct。
- 未选中的驱动块被忽略且不校验。
- 网关仅将匹配的内部 Struct 转发给驱动。
- 网关不检查、校验、合并或重写选中驱动配置块内的嵌套字段。

容忍未选中块是为了让一个可复用沙箱模板能为多个可能的网关携带驱动专属配置。未来的 CLI、TUI 或网关警告可在无信封键匹配活动驱动时帮助检测可能的拼写错误，但该警告不得将未选中块变为错误。

### 已实现的驱动 Schema

本 RFC 拥有信封和校验边界。详细的字段级参考属于驱动 README 文件和已发布文档。已实现的 Schema 族为：

| 驱动 | 已实现的配置族 |
|---|---|
| Kubernetes | `pod.node_selector`、`pod.tolerations`、`pod.runtime_class_name`、`pod.priority_class_name`、`containers.agent.resources.requests` 和 `containers.agent.resources.limits`。 |
| Docker | `cdi_devices` 用于精确 GPU 选择，`mounts` 用于 `bind`、`volume` 和 `tmpfs` 挂载，`bind` 挂载可选 `selinux_label`。 |
| Podman | `cdi_devices` 用于精确 GPU 选择，`mounts` 用于 `bind`、`volume`、`tmpfs` 和 `image` 挂载，`bind` 挂载可选 `selinux_label`。 |
| VM | `gpu_device_ids` 用于精确 GPU 选择，目前限制为一项。 |

Kubernetes 拒绝未知的嵌套驱动配置字段。其运行时类优先级为：

1. 类型化公共 `SandboxTemplate.runtime_class_name`；
2. `driver_config.kubernetes.pod.runtime_class_name`；
3. 网关配置的 `default_runtime_class_name`；
4. 无运行时类。

Docker、Podman 和 VM 的精确 GPU 选择要求存在类型化 GPU 请求。Docker 和 Podman 通过 `cdi_devices` 使用 CDI 设备 ID。VM 使用 `gpu_device_ids`，目前最多接受一项。Kubernetes 不支持通过 `driver_config` 进行精确 GPU 设备选择，并拒绝不支持的精确选择键，而不是悄然回退到通用 GPU 请求。

Docker 和 Podman 挂载配置有意受限：

- 用户提供的挂载默认只读，除非显式指定 `read_only: false`。
- 命名卷必须已存在。驱动在创建前校验，不创建或删除它们。
- 主机绑定挂载要求运维方设置 `[openshell.drivers.<driver>].enable_bind_mounts = true`。
- Docker 和 Podman 本地驱动中由 `bind` 或 `rbind` 选项支持的命名卷被视为主机绑定挂载，需要相同的选择加入。
- 挂载目标不得替换工作区根、容器根、supervisor 文件、`/etc/openshell`、`/etc/openshell-tls`、认证材料或网络命名空间路径。
- 挂载 `source`、`target` 和 `subpath` 值若包含前后空白则被拒绝，以免歧义路径通过校验。
- `bind` 挂载接受可选的 `selinux_label`，值为 `shared`（`:z`）或 `private`（`:Z`），用于启用 SELinux 的主机。卷、tmpfs 和镜像挂载不接受 SELinux 标签。

这些约束并不使主机绑定挂载变得安全。它们使不安全的宿主文件系统暴露变得显式且受运维方门控。

### 校验与受保护的不变量

选中的驱动在创建平台资源前校验其嵌套配置。稳定的已文档化 Schema 应拒绝：

- 未知字段，除非驱动明确记录扩展袋；
- 格式错误的值；
- 不支持的挂载或资源类型；
- 与类型化 OpenShell 字段的冲突；
- 覆盖网关计算的 `platform_config` 的尝试；
- 替换沙箱身份、认证、supervisor、策略、遥测或生命周期接线的尝试；以及
- OpenShell 未明确暴露的不安全平台控制。

对于公共 API 已直接建模的设置，类型化 OpenShell 字段具有权威性。驱动配置可添加驱动拥有的细节，但不得悄然覆盖类型化字段。例如，类型化 CPU、内存和 GPU 请求仍是可移植的资源意图。驱动配置可添加公共 API 未建模的 Kubernetes 专属扩展资源或容器资源细节。

`driver_config` 不得嵌入密钥、凭据、令牌、私钥或其他敏感值。驱动可允许引用现有平台对象（如 Kubernetes Secret 名称），但仅当该引用已文档化安全且驱动会校验它时。

### 生命周期与兼容性

`driver_config` 是创建时配置。更改驱动配置需要重建沙箱，除非未来 RFC 为特定驱动和键定义显式更新语义。

`driver_config` 信封、精确驱动名选择以及与网关计算的 `platform_config` 的隔离是该表面的稳定部分。嵌套的每驱动配置键仍为实验性：CLI `--driver-config-json` 标志已文档化为实验性，校验行为尚未最终确定。嵌套键可能变化，直到驱动将特定 Schema 标记为稳定。

当驱动稳定已文档化的键时，应增量演进它们，以清晰错误拒绝格式错误的输入，并在移除前弃用已文档化的键。若破坏性变更不可避免，优先使用显式版本化形状而非原地更改现有键。在此之前，调用方应将嵌套键视为可能变化，并针对已知驱动版本固定。

由于未选中的驱动块被忽略，另一驱动的过期配置可能直到该驱动被选中时才被发现。校验错误应尽可能包含驱动配置路径和可操作指导。

## 实施计划

核心 RFC 已实现：

1. PR #1744 添加了公共 `SandboxTemplate.driver_config` 字段、CLI `--driver-config-json` 路径、网关精确键选择、转发到 `DriverSandboxTemplate.driver_config`，以及初始 Kubernetes 驱动本地 Schema。
2. PR #1785 添加了 Docker 和 Podman 挂载 Schema、命名卷校验、只读默认值和绑定挂载安全门控。
3. PR #1815 将精确 GPU 设备选择从公共 API 移出，放入驱动专属的 `cdi_devices` / `gpu_device_ids` 字段。它还确立了必须拒绝不支持或未知的 Kubernetes GPU 选择键。
4. PR #1861 收紧了 Docker 和 Podman 命名卷安全，将带 `bind` 或 `rbind` 选项的本地卷视为主机绑定挂载，需要相同的运维方选择加入。
5. PR #2086 在 Docker 和 Podman 驱动中拒绝挂载 `source`、`target` 和 `subpath` 字段的前后空白。
6. PR #2092 在 Docker 和 Podman `bind` 挂载上添加了可选的 `selinux_label`（`shared`/`private`），用于启用 SELinux 的主机。

后续工作应单独跟踪：

- 当驱动配置表面足够稳定以支持 CLI/TUI 辅助时，添加机器可读的驱动配置 Schema 发现。
- 为树外驱动设计规范驱动身份和别名规则。
- 决定无匹配警告应属于网关、CLI/TUI 工具还是两者。
- 决定保留旧版 Kubernetes `platform_config.resources_raw` 行为多久，以及是否将剩余示例迁移到 `driver_config`。
- 保持驱动 README 和已发布参考文档作为每个已实现驱动 Schema 的字段级事实来源。

## 风险

- **驱动配置成为隐藏的公共 API。** 缓解措施是将已文档化的驱动键视为驱动拥有的公共 API，保持显式校验，并优先增量 Schema 演进。
- **网关无法校验未选中的块。** 这对可移植模板是有意为之，但意味着拼写错误可能较晚浮现。未来的 Schema 工具可在不改变网关语义的情况下 lint 所有块。
- **驱动 Schema 暴露不安全的平台控制。** 每个驱动必须保护网关和沙箱不变量。主机绑定挂载是最明显的例子：默认禁用，需要运维方选择加入，且仍带有已文档化的隔离警告。
- **API 可能在各驱动间碎片化。** 当行为确实是驱动专属时，这是可接受的。CPU、内存和通用 GPU 请求等可移植概念应继续使用类型化 OpenShell 字段。
- **Schema 发现仍为手动。** 当前用户依赖文档和驱动校验错误。这对已实现基线足够，但未来工具将需要机器可读 Schema。

## 备选方案

### 为每个驱动特性添加类型化字段

每个驱动专属特性都可获得类型化公共 API 字段和显式网关转发逻辑。

这保持公共 API 强类型，但网关仍是瓶颈，公共 API 围绕驱动专属细节增长，新驱动能力需要协调发布。

### 中心化公共 oneof 用于每驱动配置

公共 API 可使用中心 `oneof`，包含每个支持驱动的类型化配置消息，或网关可在调用驱动前将选中块转换为驱动专属 protobuf 消息。

这为客户端和网关提供生成类型，但将 Schema 所有权移回共享 API。每个新驱动配置键以及每个树外驱动形状都需要网关 proto 变更和协调发布。驱动本地类型化解码将所有权保留在选中的驱动。

### 将调用方配置合并到 platform_config

网关可将调用方提供的配置合并到现有网关计算的 `platform_config`。

这产生令人困惑的覆盖语义，并可能允许调用方替换网关拥有的字段。调用方提供的 `driver_config` 与网关计算的 `platform_config` 保持分离。

### 拒绝未选中的驱动块

网关可拒绝不针对选中驱动的 `driver_config` 块。

这能更早捕获一些拼写错误，但使可移植模板更难。可复用沙箱模板应能携带 Kubernetes、Docker、Podman 和 VM 配置块，并让活动网关仅应用选中的块。

### 现在使用通配符或命名空间驱动键

公共 API 可要求 `openshell.ai/kubernetes` 等键，或允许 `*/kubernetes` 等通配符。

命名空间键可能对树外驱动日后有用，但现在要求它们会将本 RFC 变成驱动身份清理。通配符使 Schema 所有权和优先级模糊。精确内置名称保持当前规则简单。

### 为所有顶层资源提供通用透传

每个 OpenShell 资源都可获得实现方拥有的配置块。

这会模糊所有权和校验边界。沙箱计算驱动有具体的选中驱动和创建时驱动模板。其他资源可能由 Provider 后端、策略引擎、身份系统或网关本身拥有，需要单独设计。

## 先例

Kubernetes CSI `StorageClass.parameters` 使用相同的所有权模式。Kubernetes 控制平面不解释每个 provisioner 的参数 Schema。它将参数传递给 CSI 驱动，由 CSI 驱动校验并使用它们。

Kubernetes Pod 规范也展示了原始透传的风险。它们很强大，但暴露任意 Pod 字段会让调用方覆盖 OpenShell 必须控制的身份、卷、安全和生命周期细节。因此驱动配置暴露狭窄的驱动拥有 Schema，而非原始平台对象。

RFC 0004 将可移植沙箱资源需求与驱动专属配置分开。本 RFC 定义了 RFC 0004 有意排除在范围外的驱动专属配置表面。

Docker 和 Podman 挂载工作提供了 OpenShell 内部的运维先例：驱动配置可暴露有用的本地运行时特性，但宿主文件系统暴露必须默认拒绝且受运维方门控。

## 开放问题

- 若 OpenShell 日后支持命名空间驱动键，树外驱动应使用什么规范驱动身份格式和别名规则？
- 哪个 Schema 发现表面应暴露驱动配置支持、规范键、Schema 版本和未知字段行为？
- 无匹配警告应由网关、CLI/TUI 工具还是两者发出？
- 现有 Kubernetes `platform_config.resources_raw` 行为应无限期保留、迁移到 `driver_config`，还是仅文档化为兼容路径？