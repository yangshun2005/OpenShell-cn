---
authors:
  - "@elezar"
state: accepted
links:
  - https://github.com/NVIDIA/OpenShell/issues/1338
  - https://github.com/NVIDIA/OpenShell/pull/1340
  - https://github.com/NVIDIA/OpenShell/pull/1360
  - https://github.com/NVIDIA/OpenShell/issues/1492
  - https://github.com/NVIDIA/OpenShell/pull/1675
  - https://github.com/NVIDIA/OpenShell/pull/1815
---

# RFC 0004 - 沙箱资源需求

## 摘要

本 RFC 提议用 `SandboxSpec` 上的类型化资源需求替换 GPU 专用的沙箱请求字段。资源需求描述可移植的工作负载需求，影响驱动选择与供给：

- **compute** 需求：CPU 和内存。
- **device** 需求：GPU 及其他类加速器资源。
- 未来类型化域，如数据集，待其语义定义后再加入。

网关使用资源需求预筛选已配置的计算驱动，再由选中的驱动校验并供给请求。`SandboxTemplate.resources` 仍是平台原生实现层和逃生舱，不是可移植的驱动选择接口。

当前实现落地了该模型的第一阶段（聚焦 GPU）：`resource_requirements.gpu` 替换旧版 GPU 专用请求字段，并带可选数量。本 RFC 中更通用的 compute/device/dataset 模型仍是后续扩展的目标方向。

## 动机

OpenShell 目前将 GPU 放置视为特例。公开的 `SandboxSpec` 和内部的 `DriverSandboxSpec` 都暴露 `gpu` 和 `gpu_device`，而驱动能力发现只报告 `supports_gpu` 和 `gpu_count`。这太窄：

- GPU 标识符是驱动特定的。Docker 和 Podman 使用 CDI 设备名，VM 驱动支持按 PCI BDF 或索引的设备 ID。
- 基于数量的放置和精确设备选择是两种不同的分配模式，不应重载到同一字段。
- CPU 和内存是常见的可移植需求，但如今调用方在公开 API 路径上必须使用后端形状的模板资源透传。
- 网关需要可移植的方式来决定哪个已配置驱动能服务沙箱请求。
- 未来资源（如数据集）不应再要求 `SandboxSpec` 上新增临时字段。

Issue #1338 识别了真实用户需求：Kubernetes 用户需要请求多个 GPU。PR #1340 通过将资源 JSON 传入 `SandboxTemplate.resources` 并让 `--gpu-count` 注入 `nvidia.com/gpu` 限制来解决这一即时需求。本 RFC 有意将其取代为长期 API 方向。Kubernetes 资源限制是有效的驱动实现，但可移植的 GPU 数量应属于类型化资源需求。若 CLI 暴露 JSON 透传，应命名并记录为驱动特定配置，而非可移植资源。

本提案受 Kubernetes 动态资源分配结构化参数启发：调度器可见的选择是结构化的，驱动特定配置保持独立并由资源驱动解释。暴露通用驱动特定配置面相关，但单独在 issue #1492 跟踪。

自本 RFC 初稿以来，已添加 `SandboxTemplate.driver_config` 作为驱动拥有的配置面。网关为活动驱动选择对应块并原样转发内部对象。Docker、Podman 和 VM 的精确 GPU 选择现在在此表达，而非可移植资源需求。

## 非目标

- 定义数据集分配、挂载、缓存或访问控制语义。数据集在本 RFC 中仅是激励性的未来域。
- 构建网关级调度器或预留系统。
- 从驱动暴露详细的每设备清单。
- 在 CLI 中暴露 JSON 格式的可移植资源请求。
- 定义通用驱动特定配置透传 API。Issue #1492 跟踪该相关 API 面。
- 在沙箱状态中发布已分配资源标识。
- 为 `gpu`、`gpu_device` 或 GPU 专用 `gpu_count` 请求字段保留 alpha 时代兼容性。旧版 GPU 专用请求字段有意不延续到本 RFC 旨在稳定的 API 形状中。

## 提案

### 公开请求模型

在 `SandboxSpec` 中添加资源需求，并从期望请求模型中移除 GPU 专用标量字段。

```proto
message SandboxSpec {
  string log_level = 1;
  map<string, string> environment = 5;
  SandboxTemplate template = 6;
  openshell.sandbox.v1.SandboxPolicy policy = 7;
  repeated string providers = 8;

  // Portable resource requirements used by the gateway for driver selection
  // and by drivers for provisioning.
  ResourceRequirements resource_requirements = 9;

  reserved 10;
  reserved "gpu_device";
}
```

公开沙箱 API 仍为 alpha。此迁移有意用类型化 `resource_requirements = 9` 消息替换旧 `bool gpu = 9` 字段，而非保留旧字段号。通过旧布尔值编码 GPU 意图的旧实时请求和持久化沙箱记录不迁移；调用方应使用匹配的 OpenShell CLI/API 版本，并在升级后需要新类型化形状时重新创建 GPU 沙箱。避免 alpha 时代保留字段使 proto 面更接近拟稳定的 API。

`SandboxTemplate.resources` 保留其作为平台原生工作负载配置的现有角色。它可包含 Kubernetes 风格的 CPU、内存和扩展资源请求与限制，但不是可移植资源契约。

已实现的第一阶段使用以下更窄的 proto 形状：

```proto
message ResourceRequirements {
  // GPU requirements for the sandbox. Presence indicates a GPU request.
  GpuResourceRequirements gpu = 1;
}

message GpuResourceRequirements {
  // Optional number of GPUs requested. When omitted, the request is for one
  // GPU using the selected driver's default assignment behavior.
  optional uint32 count = 1;
}
```

`gpu.count` 存在时必须大于零。`gpu` 存在且省略 `count` 时，有效数量为一。驱动在校验精确驱动配置设备列表时应用相同的有效数量规则。

CLI 不应为 `resource_requirements` 暴露 JSON 标志。常见可移植请求应使用类型化标志，如 CPU、内存和 GPU 数量标志；SDK/API 调用方应直接使用类型化 protobuf 消息。JSON 格式的驱动特定配置与可移植资源需求分离，通过 `SandboxTemplate.driver_config` 传递。

### 长期资源需求

为稳定的第一方资源概念使用类型化需求域，而非通过 `kind` 字段让每个请求都字符串化。

```proto
message SandboxResourceRequirements {
  // Fungible scalar workload requirements.
  ComputeResourceRequirements compute = 1;

  // Accelerator-like resources such as GPUs and MIG slices.
  repeated DeviceResourceRequirement devices = 2;

  // Future typed domain. Semantics are intentionally not defined in this RFC.
  repeated DatasetResourceRequirement datasets = 3;

  // Escape hatch for third-party or experimental resource domains.
  repeated GenericResourceRequirement extensions = 100;
}

message ComputeResourceRequirements {
  // Values use Kubernetes-style quantity strings because they are familiar and
  // already used by the driver resource model.
  string cpu_request = 1;
  string cpu_limit = 2;
  string memory_request = 3;
  string memory_limit = 4;
}

message DeviceResourceRequirement {
  // Optional local name for error messages and future status correlation.
  string name = 1;

  // Portable device class requested by the workload, such as "gpu",
  // "nvidia-gpu", or a future OpenShell-defined class name.
  string class_name = 2;

  // Number of devices in the class requested. Must be greater than zero.
  uint32 count = 3;

  // Portable labels or attributes the selected device must satisfy.
  ResourceSelector selector = 4;

  // Namespaced parameter blocks. The gateway may use namespace support for
  // prefiltering, but only drivers interpret the parameter values.
  repeated ResourceParameterBlock parameters = 5;
}

message ResourceSelector {
  // Exact-match portable attributes such as vendor=nvidia.
  map<string, string> match_attributes = 1;
}

message ResourceParameterBlock {
  // DNS-style parameter namespace, such as cdi.openshell.ai.
  string namespace = 1;
  google.protobuf.Struct parameters = 2;
}

message DatasetResourceRequirement {
  string name = 1;
  string class_name = 2;
  ResourceSelector selector = 3;
  repeated ResourceParameterBlock parameters = 4;
}

message GenericResourceRequirement {
  string kind = 1;
  string name = 2;
  uint32 count = 3;
  ResourceSelector selector = 4;
  repeated ResourceParameterBlock parameters = 5;
}
```

网关校验可移植信封：

- compute 数量必须是语法有效的数量字符串。
- device `class_name` 必须非空。
- device `count` 必须大于零。
- 参数命名空间键必须是 DNS 风格名称。
- 参数值必须符合现有请求大小限制。

网关不解释参数值。驱动必须拒绝包含其不支持参数命名空间的请求，网关可使用相同命名空间支持预筛选候选。

### Compute 需求

Compute 需求是可替代的 CPU 和内存需求。它们与设备不同，因为通常不需要精确标识或驱动特定选择。

本 RFC 仅将 CPU 和内存标准化为初始可移植 compute 需求。其他 compute 形状约束（如临时存储、大页、PID 限制、共享内存或类似 cgroup 支持的限值）可后续添加，但仅当其请求/限制语义清晰且能映射到多个驱动时。此类约束的驱动特定支持应留在驱动特定配置中，直到足够可移植以进入第一方 API。

示例请求：

```yaml
resourceRequirements:
  compute:
    cpuRequest: "2"
    cpuLimit: "4"
    memoryRequest: 4Gi
    memoryLimit: 8Gi
```

示例实现：

| 驱动 | 实现 |
|---|---|
| Kubernetes | 填充 Pod 容器 `resources.requests.cpu`、`resources.limits.cpu`、`resources.requests.memory` 和 `resources.limits.memory`。 |
| Docker | 应用支持的运行时限制，如 CPU 配额/NanoCPUs 和内存限制。当驱动能评估主机容量时，请求作为容量检查。 |
| Podman | 应用支持的运行时限制，如 CPU 配额和内存限制。当驱动能评估主机容量时，请求作为容量检查。 |
| VM | 将 CPU 和内存限制映射到 VM vCPU 数量和客户机内存分配。当驱动无法表示分离的请求/限制语义时，可要求请求与限制相等。 |

Compute 需求描述驱动供给的沙箱工作负载，而非每个运行时管理的辅助进程。若驱动后续在单独容器、边车或 Pod 中运行代理、监督器或其他控制面辅助进程，可通过驱动特定配置应用固定开销或暴露辅助特定设置。这些辅助资源是驱动实现细节，除非后续 RFC 将其提升为可移植资源需求。

驱动必须拒绝无法满足的 compute 需求。不得静默接受无效的限制或请求。

### Device 需求

Device 需求涵盖 GPU 及其他类加速器资源。第一个标准设备类是 `gpu`。

可移植 GPU 语义限于：

- `class_name`
- `count`
- `selector.match_attributes` 中的精确匹配属性

在当前聚焦 GPU 的实现中，`ResourceRequirements` 仅实现 `gpu.count` 部分。精确设备 ID 尚不是可移植选择器属性；它们仍为驱动拥有配置。Docker 和 Podman 使用 `driver_config.cdi_devices`，VM 使用 `driver_config.gpu_device_ids`。

驱动原生 GPU 细节通过命名空间参数表达。示例参数命名空间：

| 命名空间 | 目标驱动 | 示例字段 |
|---|---|---|
| `cdi.openshell.ai` | Docker, Podman | `deviceId: "nvidia.com/gpu=all"` |
| `kubernetes.openshell.ai` | Kubernetes | `resourceName: "nvidia.com/gpu"`, `resourceClassName: "nvidia-gpu"` |
| `vm.openshell.ai` | VM | `deviceId: "0000:2d:00.0"`, `deviceIdType: "bdf"` |

请求任意 NVIDIA GPU 的示例：

```yaml
resourceRequirements:
  devices:
    - name: gpu
      className: gpu
      count: 1
      selector:
        matchAttributes:
          vendor: nvidia
```

请求四个 GPU 的示例。Kubernetes 驱动可实现为 `limits["nvidia.com/gpu"] = "4"`，但公开请求保持可移植：

```yaml
resourceRequirements:
  devices:
    - name: training-gpus
      className: gpu
      count: 4
```

请求 Docker 或 Podman 支持的 CDI GPU 示例：

```yaml
resourceRequirements:
  devices:
    - name: gpu
      className: gpu
      count: 1
      parameters:
        - namespace: cdi.openshell.ai
          parameters:
            deviceId: nvidia.com/gpu=all
```

按 BDF 请求 VM GPU 示例：

```yaml
resourceRequirements:
  devices:
    - name: gpu
      className: gpu
      count: 1
      parameters:
        - namespace: vm.openshell.ai
          parameters:
            deviceId: "0000:2d:00.0"
            deviceIdType: bdf
```

示例实现：

| 驱动 | 实现 |
|---|---|
| Kubernetes | 将 `className=gpu,count=N` 转换为 Pod 资源限制，如 `limits["nvidia.com/gpu"] = "N"`，除非 Kubernetes 特定参数选择其他资源名或类。 |
| Docker | 存在精确 `driver_config.cdi_devices` 值时使用；否则从发现的清单中选择 `count` 个默认 CDI GPU ID。 |
| Podman | 存在精确 `driver_config.cdi_devices` 值时使用；否则从发现的清单中选择 `count` 个默认 CDI GPU ID。 |
| VM | 将 VM 参数转换为基于 BDF 或索引的设备分配。 |

Docker 和 Podman 默认选择在规范化 NVIDIA CDI 清单上轮询。优先使用索引 ID 并按数值排序；无索引 ID 时使用命名 ID。在仅 all 的 WSL2 运行时上，`nvidia.com/gpu=all` 可作为回退并计为一个可选设备。选择器在校验和创建默认 GPU 请求前刷新清单，因此驱动启动后添加或移除的 CDI 设备可影响后续创建。

显式 CDI ID 是不透明的。驱动不解析别名、不规范化 `nvidia.com/gpu=all`，也不在显式路径上特殊对待。精确 ID 列表不得包含重复项，长度必须匹配有效 GPU 数量。两个显式 ID 的列表配合 `--gpu` 且无数量会失败，因为有效数量为一。

Docker 和 Podman 不应解释 VM BDF/索引参数。VM 驱动不应解释 CDI 参数。网关命名空间预筛选应避免将明显不兼容的请求发送给这些驱动。

### 组合示例

CPU、内存和一个 GPU：

```yaml
resourceRequirements:
  compute:
    cpuRequest: "4"
    cpuLimit: "8"
    memoryRequest: 16Gi
    memoryLimit: 32Gi
  devices:
    - name: gpu
      className: gpu
      count: 1
      selector:
        matchAttributes:
          vendor: nvidia
```

Kubernetes 实现：

```yaml
resources:
  requests:
    cpu: "4"
    memory: 16Gi
  limits:
    cpu: "8"
    memory: 32Gi
    nvidia.com/gpu: "1"
```

Docker 或 Podman 实现：

```text
runtime CPU/memory limits derived from compute limits
CDI device injection derived from the selected gpu device requirement
```

VM 实现：

```text
VM vCPU count and memory allocation derived from compute limits
GPU passthrough derived from vm.openshell.ai parameters when present
```

### 具体实现

这些示例展示同一可移植请求在驱动选中后如何编译。精确序列化平台载荷仍由驱动拥有；这些是预期效果。

#### Kubernetes CPU 和内存

输入：

```yaml
resourceRequirements:
  compute:
    cpuRequest: "2"
    cpuLimit: "4"
    memoryRequest: 4Gi
    memoryLimit: 8Gi
```

Kubernetes Pod 容器资源：

```yaml
resources:
  requests:
    cpu: "2"
    memory: 4Gi
  limits:
    cpu: "4"
    memory: 8Gi
```

#### Kubernetes 多 GPU

输入：

```yaml
resourceRequirements:
  devices:
    - name: training-gpus
      className: gpu
      count: 4
```

Kubernetes Pod 容器资源：

```yaml
resources:
  limits:
    nvidia.com/gpu: "4"
```

若提供 `kubernetes.openshell.ai.resourceName`，驱动使用该资源名而非 `nvidia.com/gpu`。

#### Docker 或 Podman CDI GPU

输入：

```yaml
resourceRequirements:
  devices:
    - name: gpu
      className: gpu
      count: 1
      parameters:
        - namespace: cdi.openshell.ai
          parameters:
            deviceId: nvidia.com/gpu=0
```

Docker 或 Podman 运行时请求：

```text
--device nvidia.com/gpu=0
```

网关可将此请求预筛选到为 `gpu` 设备类声明 `cdi.openshell.ai` 参数命名空间的驱动。

#### 按 BDF 的 VM GPU

输入：

```yaml
resourceRequirements:
  devices:
    - name: gpu
      className: gpu
      count: 1
      parameters:
        - namespace: vm.openshell.ai
          parameters:
            deviceId: "0000:2d:00.0"
            deviceIdType: bdf
```

VM 驱动实现：

```text
attach host PCI device 0000:2d:00.0 to the sandbox VM
```

网关可将此请求预筛选到为 `gpu` 设备类声明 `vm.openshell.ai` 参数命名空间的类 VM 驱动。

#### 可移植资源与模板资源冲突

输入：

```yaml
resourceRequirements:
  devices:
    - name: gpu
      className: gpu
      count: 4
template:
  resources:
    limits:
```