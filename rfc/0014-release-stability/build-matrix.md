# RFC 0014 补充 - 构建矩阵

本补充定义了 RFC 0014 所述发布系统的构建目标。这些目标遵循[拟议支持矩阵](https://github.com/NVIDIA/OpenShell/blob/d575cf85f8c520774cc3e307ec1bcdabf57cd682/docs/reference/support-matrix.mdx)的结构，描述预期最终状态。

这些表格列举了构建和能力目标，并不意味着平台、架构、安装方式、驱动、拓扑和能力的每一种交叉组合都受支持。[发布资格补充](release-qualification.md)定义了最小阻塞发布门禁：每个计算驱动一个一致性工作流，每种受支持的网关拓扑一个 Kubernetes 一致性工作流，每个安装包一个升级工作流，外加每个预发布版本一次破坏性 API 变更审查和一次安全审查。其他受支持的维度作为子用例或独立的 CI 控制来验证，而非发布任务的笛卡尔积。

## 平台

| 平台 | 安装方式 | 要求 |
| --- | --- | --- |
| macOS（Apple Silicon） | Homebrew | macOS 13.3 或更高版本 |
| Linux（x86_64、arm64） | APT/DEB、RPM、snap | glibc 2.28 或更高版本 |
| Windows（x86_64、arm64） | MSI、WinGet | 文档规定的最低 Windows 版本和 MSVC 工具链 |

包安装程序包含 CLI、TUI、网关和受支持的本地驱动。同时发布独立的 CLI 和 TUI 二进制文件，用于远程网关访问。

## 计算驱动

| 驱动 | 支持的宿主机 | 最低版本 | 要求 |
| --- | --- | --- | --- |
| Docker | macOS、Linux、Windows | 28.0.4 或更高版本 | Docker Engine 或 Docker Desktop |
| Podman | macOS、Linux、Windows | 5.x | Podman socket、无根网络和 cgroups v2 |
| MicroVM | macOS、Linux | macOS 13.3 或更高版本；Linux 上需 KVM | 宿主机虚拟化、macOS 上的 Hypervisor.framework、Linux 上的 KVM |
| Kubernetes | Kubernetes 集群 | 1.29 或更高版本 | Helm 3.x 及兼容的 Agent Sandbox 控制器和 CRD |

## GPU 支持

| 计算驱动 | 支持的环境 | 设备接口 | 要求与限制 |
| --- | --- | --- | --- |
| Docker | Linux；Windows 通过 WSL2 | NVIDIA CDI | 启用 CDI 且可见 NVIDIA 设备的运行时；默认和计数的 GPU 请求 |
| Podman | Linux；Windows 通过 WSL2 | NVIDIA CDI | 可见的 NVIDIA CDI 设备；默认和计数的 GPU 请求 |
| MicroVM | Linux | QEMU/VFIO PCI 直通 | IOMMU、VFIO、root 权限、兼容的沙箱镜像，每个沙箱一个 GPU |
| Kubernetes | Linux GPU 节点 | `nvidia.com/gpu` 扩展资源 | NVIDIA GPU Device Plugin 和兼容的沙箱镜像 |

发布资格接受当前 CUDA 驱动，并针对 Tesla Recommended Driver 分支进行验证。

## Kubernetes

| 组件 | 支持版本 | 说明 |
| --- | --- | --- |
| Kubernetes | 1.29 或更高版本 | Helm 部署和沙箱调度所需 |
| Helm | 3.x | 安装和升级 OpenShell chart 所需 |
| Agent Sandbox 控制器和 CRD | 兼容版本 | 安装 OpenShell chart 前所需 |
| 用户命名空间 | 1.33 或更高版本 | 可选；启用 `hostUsers: false` 以进行 UID 重映射 |

支持 GKE Standard、GKE Autopilot 和 OpenShift 4.x。OpenShift 使用 OpenShift 专用的 SCC 绑定和 chart 配置。

## SDK

| SDK | 最低版本 |
| --- | --- |
| Python | 3.12 |
| TypeScript | 5.7 |
| Rust | 1.90 |
| Go | 1.24 |