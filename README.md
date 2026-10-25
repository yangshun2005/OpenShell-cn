

> **[中文化] OpenShell**
>
> 此项目是 [NVIDIA/OpenShell](https://github.com/NVIDIA/OpenShell) 的中文翻译版本。
> - 原项目 Stars: 19704
> - 主语言: Rust
> - 许可证: Apache-2.0
> - 翻译日期: 2026-10-25
> - 原始 README: [README_en.md](README_en.md)
>
> 说明：下方 README 正文为**原项目文档的中文翻译**，力求忠于原文；本页顶部的「中文快速上手」为**面向中文用户的补充导读**（非原文内容），目的是让你不用读完整篇 README 就能先把项目跑起来。
>
> 如有翻译不准确之处，欢迎提 Issue 或 PR。

---


## 🚀 中文快速上手（补充导读，非原文）

### 这是什么 / 解决什么问题

OpenShell 是 NVIDIA 开源的、面向自主 AI Agent 集群的安全私有运行时。Agent 在读取文件、安装软件包、调用 API、使用凭据时往往需要较大权限，OpenShell 通过策略（policy）声明每个 Agent 能访问什么，并在内核层面强制执行文件访问、系统调用和网络连接的限制，让 Agent 获得能力的同时不放开对数据、密钥和网络的访问。它还使用形式化验证，在策略变更生效前检查该变更会新增哪些访问权限。

### 快速上手（最短路径）

1. 确认环境满足要求（见下方「环境要求」）。
2. 执行安装脚本（同时会安装 CLI 和本地 gateway）：
   ```shell
   curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh
   ```
3. 创建一个沙箱：
   ```shell
   openshell sandbox create --name demo
   ```
4. 默认沙箱镜像是最小化的 Ubuntu，未安装任何 Agent。要运行真实 Agent，请按官方文档 [Run Your First Agent](https://docs.nvidia.com/openshell/latest/about/run-your-first-agent) 操作（该文档演示用 OpenCode 对接免费的 OpenRouter 模型，并展示如何在 Agent 需要新访问权限时进行批准）。
5. （可选）为你的编码 Agent 安装 OpenShell 技能：
   ```shell
   npx skills add NVIDIA/OpenShell
   ```

### 环境要求

- **操作系统**：Linux、Apple Silicon 上的 macOS，或带 WSL 2 的 Windows（实验性）。
- **运行时依赖**：Docker、Podman 或主机虚拟化（三者之一）。
- **Linux 细节**：安装脚本会安装 Debian 或 RPM 包；脚本中标注的最低 glibc 版本为 2.28。
- **macOS 细节**：仅 Apple Silicon；通过 Homebrew formula 安装，由 Homebrew 管理二进制布局和 launchd 服务生命周期。
- **Python SDK**：要求 Python >= 3.11（来自 pyproject.toml）。
- **Rust 构建**：workspace 声明 `rust-version = "1.94"`、`edition = "2024"`（来自 Cargo.toml）。
- **网络提示**：安装脚本从 GitHub 下载发布包，国内网络环境可能需要配置代理或加速下载；原文未提供镜像源说明。

### 常见坑与注意事项

- **默认沙箱不含 Agent**：`openshell sandbox create` 创建的是最小化 Ubuntu 镜像，想跑真实 Agent 需按官方文档额外配置。
- **平台限制**：Windows 支持为实验性（WSL 2）；macOS 仅支持 Apple Silicon。
- **需要容器/虚拟化后端**：必须已具备 Docker、Podman 或主机虚拟化之一，否则无法运行沙箱。
- **遥测默认开启**：OpenShell 会收集匿名遥测（仅运营类别与计数，不含沙箱名、主机名、文件路径、提示词、凭据等）。可在 gateway 上设置 `OPENSHELL_TELEMETRY_ENABLED=false`，Helm 安装则设 `server.telemetryEnabled=false` 关闭。
- **SDK 与 gateway 版本**：原文建议 SDK 与 gateway 尽量使用同一 OpenShell 版本。

---



---

<!-- markdownlint-disable MD033 MD041 -->

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="docs/brand/assets/openshell-banner-dark.png">
  <source media="(prefers-color-scheme: light)" srcset="docs/brand/assets/openshell-banner-light.png">
  <img alt="OpenShell" src="docs/brand/assets/openshell-banner-light.png" width="430">
</picture>

<!-- markdownlint-enable MD033 MD041 -->

[![License](https://img.shields.io/badge/License-Apache_2.0-blue)](https://github.com/NVIDIA/OpenShell/blob/main/LICENSE)
[![PyPI](https://img.shields.io/badge/PyPI-openshell-orange?logo=pypi)](https://pypi.org/project/openshell/)
[![Security Policy](https://img.shields.io/badge/Security-Report%20a%20Vulnerability-red)](SECURITY.md)
[![Documentation](https://img.shields.io/badge/docs-latest-brightgreen)](https://docs.nvidia.com/openshell/latest/index.html)

> [!IMPORTANT]
> **OpenShell 0.1.x 新增：** 稳定的发布节奏、新的隔离原语、扩展的扩展接口，以及新 API。[阅读 0.1.0 升级指南](https://docs.nvidia.com/openshell/latest/upgrade/0-1-0)。

OpenShell 是为自主 AI 智能体集群打造的安全、私密运行时。智能体在能读取文件、安装包、调用 API 和使用凭证时最有价值。OpenShell 赋予它们这些能力，同时不让它们无限制地访问你的数据、密钥或网络。你在策略中声明每个智能体能接触什么，OpenShell 负责强制执行。

## 工作原理

OpenShell 通过两种方式管控智能体的行为：它在运行时对内核进行插桩，对每次文件访问、系统调用和网络连接强制执行策略；并通过形式化验证，在策略变更应用前检查它将允许什么。

- **内核级强制执行。** 每个智能体运行在隔离沙箱中。内核控制限制它能访问哪些文件、能发起哪些系统调用，每个网络连接在离开沙箱前都要通过策略检查。智能体永远看不到真实凭证；OpenShell 只将其添加到发往已批准端点的请求中。
- **形式化验证的策略变更。** 策略变更获批前，OpenShell 使用形式化验证标记它将授予的高风险新访问，例如用凭证访问新主机或调用新的 API 方法，这些变更会等待人工审核。

参见[架构](https://docs.nvidia.com/openshell/latest/about/architecture)了解网关、supervisor 和沙箱如何协同工作。

## 快速开始

你需要 Linux、Apple Silicon 上的 macOS，或带 WSL 2 的 Windows（实验性），以及 Docker、Podman 或主机虚拟化。详见[支持矩阵](https://docs.nvidia.com/openshell/latest/about/support-matrix)。

```shell
curl -LsSf https://raw.githubusercontent.com/NVIDIA/OpenShell/main/install.sh | sh
openshell sandbox create --name demo
```

安装程序会配置 CLI 和本地网关。默认沙箱镜像是未安装任何智能体的精简 Ubuntu。要运行真实智能体，请遵循[运行你的第一个智能体](https://docs.nvidia.com/openshell/latest/about/run-your-first-agent)：它让 OpenCode 对接免费的 OpenRouter 模型，并演示如何在智能体需要时批准新访问。

## 深入探索

- [沙箱](https://docs.nvidia.com/openshell/latest/how-it-works/sandboxes/overview)：镜像、运行时、GPU 和生命周期。
- [策略](https://docs.nvidia.com/openshell/latest/how-it-works/policies/overview)：文件系统、网络和进程规则，以及用于审查变更的[顾问](https://docs.nvidia.com/openshell/latest/how-it-works/policies/advisor)和[证明器](https://docs.nvidia.com/openshell/latest/how-it-works/policies/prover)。
- [提供方](https://docs.nvidia.com/openshell/latest/how-it-works/providers/overview)：仅在已批准端点生效的凭证，包括[推理](https://docs.nvidia.com/openshell/latest/how-it-works/inference)。
- [网关](https://docs.nvidia.com/openshell/latest/how-it-works/gateways/overview)：沙箱、策略和访问的控制平面。
- [Kubernetes](https://docs.nvidia.com/openshell/latest/kubernetes/setup)：使用 Helm 部署网关。你的 CNI 必须强制执行 `NetworkPolicy`。
- [可扩展性](https://docs.nvidia.com/openshell/latest/extensibility/overview)：中间件、拦截器和计算驱动。
- [教程](https://docs.nvidia.com/openshell/latest/tutorials/first-network-policy)：分步的策略和智能体演练。
- [预发布和开发构建](https://docs.nvidia.com/openshell/latest/about/installation#prerelease-and-development-builds)：试用即将发布的版本或 `main` 上的最新提交。

## 智能体技能

为你的编码智能体安装公开的 OpenShell 技能：

```shell
npx skills add NVIDIA/OpenShell
```

这些技能教会你的智能体驱动 OpenShell CLI、编写沙箱策略，以及调试网关和推理路由。它们位于 [`skills/`](skills/)，无需检出 OpenShell 源码即可使用。

## SDK

SDK 将应用连接到 OpenShell 网关。它们不安装 CLI。尽可能让 SDK 和网关使用同一个 OpenShell 发布版本。

| 语言 | 安装 | 文档 |
|---|---|---|
| Python | `uv add openshell` | [README](python/openshell/) |
| TypeScript | `npm install @nvidia/openshell-sdk`（GitHub Packages） | [README](sdk/typescript/README.md) |
| Go | `go get github.com/NVIDIA/OpenShell/sdk/go@latest` | [README](sdk/go/README.md) |
| Rust | `cargo add openshell-sdk --git https://github.com/NVIDIA/OpenShell --tag <release-tag>` | [安装与用法](docs/sdk/rust.mdx) |

## 社区

- **问题和讨论：** [GitHub Discussions](https://github.com/NVIDIA/OpenShell/discussions)
- **Bug 报告和功能请求：** [GitHub Issues](https://github.com/NVIDIA/OpenShell/issues)，请使用 Issue 模板
- **安全漏洞：** 遵循 [SECURITY.md](SECURITY.md)。不要开 GitHub Issue。
- **路线图：** [OpenShell 路线图](https://github.com/orgs/NVIDIA/projects/233)和 [RFC 看板](https://github.com/orgs/NVIDIA/projects/233/views/6)
- **在云端试用：** [Brev Launchable](https://brev.nvidia.com/launchable/deploy/now?launchableID=env-3Ap3tL55zq4a8kew1AuW0FpSLsg)

OpenShell 以智能体优先构建：它用自身所赋能的智能体驱动工作流来开发。开发环境配置和贡献流程见 [CONTRIBUTING.md](CONTRIBUTING.md)，仓库编码规范见 [AGENTS.md](AGENTS.md)。

## 遥测

OpenShell 收集匿名遥测，仅限于运营类别和计数，以帮助改进项目。它不收集沙箱名称、主机名、文件路径、提示词、凭证、提供方或模型名称，或用户内容。要禁用，请在网关上设置 `OPENSHELL_TELEMETRY_ENABLED=false`，或对 Helm 安装设置 `server.telemetryEnabled=false`。你也可以完全编译掉遥测。详见[遥测](https://docs.nvidia.com/openshell/latest/observability/telemetry)，已发布的使用趋势见[社区遥测报告](telemetry/README.md)。

## 声明与免责

本软件会自动检索、访问或与外部材料交互。这些检索到的材料不随本软件分发，仅受单独的条款、条件和许可证约束。你全权负责查找、审查并遵守所有适用的条款、条件和许可证，并负责验证任何检索到的材料对你特定用例的安全性、完整性和适用性。本软件按"原样"提供，不附带任何形式的保证。作者对任何检索到的材料不作任何陈述或保证，对你因使用或无法使用本软件或任何检索到的材料而产生的任何损失、损害、责任或法律后果不承担任何责任。使用本软件和检索到的材料的风险由你自行承担。

## 许可证

本项目采用 [Apache License 2.0](https://github.com/NVIDIA/OpenShell/blob/main/LICENSE) 许可。