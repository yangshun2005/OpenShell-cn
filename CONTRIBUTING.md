# 为 OpenShell 做贡献

OpenShell 以 Agent 优先的方式构建。我们用 Agent 设计和实现系统，由人类负责产品决策和项目路线图。

## 关键规则

**你必须理解自己的代码。** 用 AI Agent 写代码不仅可以接受，而且这个项目就是这么运作的。但你必须能解释你的改动做了什么、如何与系统其余部分交互。如果做不到，就不要提交。

提交自己看不懂的 Agent 生成代码——无论看起来多干净——都是在浪费维护者的时间，你的 PR 会被关闭。屡次违规者将被禁止参与项目。

## AI 使用

OpenShell 是 Agent 优先，而非仅限 Agent。这个区别很重要：

- **要**用 Agent 探索代码库、运行诊断、生成代码、迭代实现。
- **要**使用 `.agents/skills/` 中的技能——它们的存在就是为了让你的 Agent 更高效。
- **要**反复追问你的 Agent，直到你理解改动中的每个边界情况和交互。
- **不要**提交离开 Agent 就解释不了的代码。
- **不要**用 Agent 代替对系统的理解。请阅读 RFC、crate README 和已发布的文档。

## 首次贡献者

我们采用担保（vouch）制度。原因是 AI 让生成看似合理但质量低劣的贡献变得轻而易举，我们不能再默认信任。

1. 发起一个 [Vouch Request](https://github.com/NVIDIA/OpenShell/discussions/new?category=vouch-request) 讨论。
2. 说明你想改什么、为什么改。
3. 用你自己的话写。AI 生成的担保请求会被拒绝。
4. 维护者批准后会评论 `/vouch`，该请求讨论将自动关闭。
5. **在开 PR 之前先获得担保。** 获得担保后即可提交 Pull Request。

**如果你未获得担保，你开的任何 Pull Request 都会被自动关闭。** PR 及其讨论会保留。获得担保后，请开新 PR（推荐）或重新打开已关闭的 PR。不能通过保持开启或转为草稿来绕过检查。拥有推送权限的组织成员和协作者可跳过此检查。

### 寻找工作

标记为 [`good first issue`](https://github.com/NVIDIA/OpenShell/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22) 的 Issue 范围明确、文档完善、对新贡献者友好。从那里开始。如需指导，请在 Issue 下评论。

开放的 Issue 不一定已被接受或可以开始实现。请查看仓库当前的 `state:*` 标签，状态不明时询问维护者。用户的直接请求授权 Agent 执行所请求的阶段，但不改变 Issue 的处理状态。

## 开 Issue 之前

先在开放和已关闭的 Issue 中搜索相同需求。Bug 报告和功能请求必须包含：

1. **用户故事：** 证明你本人使用 OpenShell，并描述你直接遇到或需要的具体用例和行为。如果缺少这一手信息，Agent 必须先向人类操作者询问再提交。
2. **问题陈述：** 简要说明当前行为哪里坏了或缺失。
3. **影响 / 为什么重要：** 当前行为的后果、现有的变通方案，以及为什么该方案不够。
4. **验收标准：** 定义成功的具体、可观察的结果。

功能请求还必须提出面向用户的工作流，并描述考虑过的替代方案，包括相关的 OpenShell 扩展点。当现有中间件、拦截器、提供者或其他扩展能满足用例时，优先走那条路。仅运行另一个服务不足以否定它。涉及配置、CLI、SDK 或其他用户体验的改动，需提供示例供人工审阅。Bug 报告必须包含仅使用 OpenShell 部署的复现步骤、OpenShell 版本和相关环境，以及在能实质澄清行为时提供一小段脱敏日志。不要仅为证明可复现性而安装第三方工具。所有 Issue 都必须完全围绕 OpenShell 来表述。

项目包含可选的 [Agent 技能](#agent-skills)，用于使用 OpenShell 和为仓库做贡献。有帮助时可以使用，但请用自己的话总结有用的结果，而不是粘贴诊断记录。

### 何时开 Issue

- 某个工作流的行为与你需要或合理预期的不同。
- OpenShell 不支持对你的工作流重要的结果。
- 现有文档或配置没有说明如何完成受支持的工作流。
- 安全漏洞必须遵循 [SECURITY.md](SECURITY.md)——**不要**用 GitHub Issue。

### 何时不要开 Issue

- 一般性问题或开放式讨论——使用 [GitHub Discussions](https://github.com/NVIDIA/OpenShell/discussions)。
- 安全漏洞——遵循 [SECURITY.md](SECURITY.md)。

## 提交改动之前

在维护者接受 Issue 之前，不要开始实质性的、有 Issue 支撑的工作，除非维护者直接要求你调查或实现。工作获得授权后，用你的 Agent 调查当前代码和行为。如果 Issue 中包含早期诊断，请验证而不是直接依赖。

按需使用 Agent 和仓库技能来理解受影响的代码、评估权衡、实现最小且连贯的改动并验证。Pull Request 应说明改了什么、如何测试；不能用 Agent 记录代替贡献者本人的理解。

每个 Pull Request 必须获得 [MAINTAINERS.md](MAINTAINERS.md) 中列出的某位维护者批准才能合并。这由 `OpenShell / Maintainer Approval` 状态检查强制执行，一旦其中一位审阅者批准，该检查即变绿。其他贡献者的审阅同样欢迎，并计入一般批准要求，但不满足此检查。

涉及执行机制本身的 Pull Request——`.github/workflows/`、`.github/actions/`、`MAINTAINERS.md`、`.github/CODEOWNERS` 下的任何内容，或 `tasks/scripts/` 中的执行脚本——还需获得 [.github/CODEOWNERS](.github/CODEOWNERS) 中列出的代码所有者批准。状态检查从 `main` 读取输入，但由审阅触发的工作流运行的是 Pull Request 自带的该工作流文件副本，因此正是 GitHub 原生的代码所有者要求，阻止了某个改动禁用本应拦截它的门禁。

维护者不会被自动请求。如果你的 Pull Request 一直闲置，请在 PR 中或 CNCF Slack 频道里请求审阅者，不要干等。

## Agent 技能

OpenShell 将使用产品的技能与开发仓库的技能分开存放。

### 使用 OpenShell 的技能

公开技能位于 `skills/`，无需检出 OpenShell 源码即可使用。用 `npx skills add NVIDIA/OpenShell` 安装。

| 技能 | 用途 |
| --- | --- |
| `openshell-cli` | CLI 用法、沙箱生命周期、提供者管理和 BYOC 工作流 |
| `debug-openshell-cluster` | 诊断网关部署和健康问题 |
| `debug-inference` | 诊断附加提供者的推理、原生端点，以及从已退役的托管端点迁移 |
| `generate-sandbox-policy` | 根据需求或 API 文档生成 YAML 沙箱策略 |

公开技能使用 `openshell --help` 获取已安装命令的语法，使用已发布的 OpenShell 文档获取产品概念和配置。它们不得依赖仓库相对路径的源码或文档文件。

### 贡献者 Agent 技能

贡献者和维护者技能位于 `.agents/skills/`。它们被标记为内部，因此 Agent Skills CLI 会将其排除在普通公开发现之外，但能感知仓库的 Agent 框架可以原生发现并加载它们。内部元数据是发现过滤器，不是访问控制边界。

| 类别        | 技能                     | 用途                                                                                             |
| --------------- | ------------------------- | --------------------------------------------------------------------------------------------------- |
| 贡献    | `create-spike`            | 调查问题，产出结构化的 GitHub Issue                                            |
| 贡献    | `create-rfc`              | 根据仓库模板创建 RFC 提案                                                   |
| 贡献    | `build-from-issue`        | 根据 GitHub Issue 规划和实现工作（维护者工作流）                                   |
| 贡献    | `create-github-issue`     | 创建结构良好的 GitHub Issue                                                                |
| 贡献    | `create-github-pr`        | 按规范创建 Pull Request                                                        |
| 审阅       | `review-github-pr`        | 总结 PR 差异和关键设计决策                                                         |
| 审阅       | `review-security-issue`   | 评估安全问题的严重性和修复方案                                                 |
| 审阅       | `fix-security-issue`      | 实施已批准的安全修复计划                                                     |
| 审阅       | `watch-github-actions`    | 监控 CI 流水线状态和日志                                                                 |
| 审阅       | `launch-openshell-gator`  | 启动并监督 OpenShell gator Agent，用于 Issue 和 PR 监控                             |
| 审阅       | `test-release-canary`     | 触发并迭代 Release Canary 工作流，对已发布构建产物做冒烟测试            |
| 维护     | `release-openshell`       | 选取最新的合格预发布版本，并将其提交标记为稳定版                                   |
| 分类          | `triage-issue`            | 评估、分类和分派社区提交的 Issue                                                  |
| 分类          | `review-triage-queue`     | 根据 GitHub 接收情况生成本地只读队列报告                                           |
| 分类          | `apply-triage-action`     | 应用、拒绝或推迟一条明确选定的分类建议                               |
| 平台        | `helm-dev-environment`    | 启动和管理本地 Kubernetes 开发环境                                       |
| 平台        | `tui-development`         | 基于 ratatui 的终端 UI 开发指南                                                 |
| 平台        | `build-openshell-mxc-windows` | 维护并验证仅构建的 x64 和 ARM64 Windows MSVC 通道                             |
| 文档   | `update-docs-from-commits` | 扫描近期提交，为用户可见改动起草文档更新                                  |
| 维护     | `sync-agent-infra`        | 检测并修复 Agent 优先基础设施文件之间的漂移                                        |
| 维护     | `release-captain`         | 执行每周一的预检和周二稳定版发布清单                               |
| 参考       | `sbom`                    | 生成 SBOM 并解析依赖许可证                                                      |

### Issue 工作流

社区 Issue 经过分类、技术验证和人工接受三个阶段。仓库的 `state:*` 标签记录这些阶段。应用或解读标签前请先阅读[标签发现](#label-discovery)；不要假设标签列表是固定的。Agent 可以评估证据并请求缺失信息。由维护者决定是否接受有效工作以及它在路线图中的位置。

对 Agent 的直接请求授权所请求的规划或实现。仅请求规划不授权实现。对于无人值守的工作，请检查当前状态描述、维护者指派和评论，以确定已授权的阶段。仅技术验证不授权实现。开始前检查是否已有负责人、分支或 PR。

不要将疑似漏洞作为公开 Issue 提交。遵循 [SECURITY.md](SECURITY.md)。授权审阅或修复时使用专门的安全技能。

### 标签发现

获取仓库标签及其描述的每一页：

```shell
gh api --paginate 'repos/{owner}/{repo}/labels?per_page=100' --jq '.[] | {name, description}'
```

`gh label list` 的默认输出仅限 30 个标签。不要把这第一页当作完整的工作流定义。如果描述缺失或含糊，请查看维护者说明和 Issue 评论以确定其含义。如果仍不清楚，请在更改状态或开始无人值守工作前请维护者定义。不要仅凭标签名称推断授权。

记录分类结果时，替换之前的评估状态，而不是累积评估标签。保留维护者接受、执行状态、路线图决策和无关标签。

## 前置条件

安装 [mise](https://mise.jdx.dev/)。它用于搭建开发环境。

```bash
# Install mise (macOS/Linux)
curl https://mise.run | sh
```

安装 `mise` 后，用 `mise activate` 激活，或[将其加入你的 shell](https://mise.jdx.dev/getting-started.html)。

Shell 配置示例：

```bash
# Bash
echo 'eval "$(~/.local/bin/mise activate bash)"' >> ~/.bashrc

# Fish
echo '~/.local/bin/mise activate fish | source' >> ~/.config/fish/config.fish

# Zsh
echo 'eval "$(~/.local/bin/mise activate zsh)"' >> ~/.zshrc
```

项目要求：

- Rust 1.94+
- Python 3.11+
- Docker（运行中）

### Z3 安装

`openshell-prover` crate 和独立的 `openshell-prover-cli` 二进制直接链接 Z3。`openshell-server` crate 依赖 prover，`openshell-gateway` 二进制 crate 又依赖 `openshell-server`。`openshell-cli` crate 不依赖 Z3。Nix 开发 shell 提供 Z3。在 macOS 和 Linux 上于该 shell 之外构建时，请安装系统 Z3 开发包；`z3-sys` 通过 `pkg-config` 发现它。链接器使用已安装的静态或共享库。Nix 开发 shell 提供静态 Z3 库。

```bash
# macOS
brew install z3

# Ubuntu / Debian
sudo apt install libz3-dev

# Fedora
sudo dnf install z3-devel
```

如需从源码构建 Z3，启用 `vendored-z3`（需要 CMake 和 C++ 编译器）：

```bash
cargo build -p openshell-prover --features vendored-z3
cargo build -p openshell-prover-cli --features vendored-z3
```

本地网关镜像和 E2E 构建会启用 `vendored-z3`，这样复制进去的网关二进制在运行时镜像中无需共享 Z3 库。

对于 x86-64 和 ARM64 Windows MSVC 构建，使用以下 Z3 路径之一：

- 预构建 Z3（`windows:*` 任务的默认方式）：`z3-sys` 在首次构建时为目标架构下载固定版本的 Z3 5.1.0 GitHub 发布包。Cargo 会复用其目标目录中解压的归档。Windows CI 使用 `READ_ONLY_GITHUB_TOKEN` 认证 GitHub API 请求，并将归档保留在架构特定的 Cargo 目标缓存中。本地冷构建时，可设置 `READ_ONLY_GITHUB_TOKEN` 以避免 GitHub API 匿名速率限制。
- 系统 Z3：将 `Z3_LIBRARY_PATH_OVERRIDE` 指向包含目标兼容 MSVC Z3 库的目录，将 `Z3_SYS_Z3_HEADER` 指向 `z3.h` 的完整路径。设置了 `Z3_LIBRARY_PATH_OVERRIDE` 时，`windows:*` 任务会自动使用此路径。

`openshell-prover` 本身不依赖 `bindgen`/`libclang`，因此仅构建此 crate 不需要 `LIBCLANG_PATH`：

```powershell
cargo build -p openshell-prover --target x86_64-pc-windows-msvc --features prebuilt-z3
```

### Windows 完整构建

要构建完整的 Windows 二进制集（包括 `openshell-gateway.exe` 和 `openshell.exe`），请使用 `windows:build:x64` mise 任务，而不是单 crate 的 `cargo build`。它默认下载固定版本的预构建 Z3 发布包。完整构建还会编译使用 `bindgen` 的 crate（例如 Windows 上的 MXC 驱动），因此需要 `libclang.dll`；如果 LLVM 不在默认搜索路径中，请将 `LIBCLANG_PATH` 设为包含 `libclang.dll` 的目录：

```powershell
$env:LIBCLANG_PATH='C:\Program Files\Microsoft Visual Studio\2022\<Edition>\VC\Tools\Llvm\x64\bin'
mise run --skip-tools windows:build:x64
```

要使用本地 x64 Z3 发布包而非预构建下载，请在运行任务前设置 `Z3_LIBRARY_PATH_OVERRIDE` 和 `Z3_SYS_Z3_HEADER`：

```powershell
$env:Z3_LIBRARY_PATH_OVERRIDE='C:\path\to\z3-5.1.0-x64-win\bin'
$env:Z3_SYS_Z3_HEADER='C:\path\to\z3-5.1.0-x64-win\include\z3.h'
mise run --skip-tools windows:build:x64
```

### macOS 构建工具

本地构建前请安装 Apple Command Line Tools：

```bash
xcode-select --install
```

## 入门

```bash
# One-time trust
mise trust

# Run a standalone gateway for local development
mise run gateway
```

## 构建 `openshell` CLI

在本仓库内，`openshell` 是 `scripts/bin/openshell` 处的本地快捷脚本。该脚本会：

1. 按需构建 `openshell-cli`。
2. 运行 `target/debug/openshell` 下的本地调试 CLI 二进制。

由于 `mise` 为本项目将 `scripts/bin` 加入 `PATH`，你可以直接在仓库中运行 `openshell`。

```bash
openshell --help
openshell sandbox create -- codex
```

### Rust 构建缓存

Mise 会保留已有的 `SCCACHE_DIR`，让每个环境自行选择编译器缓存的存储位置。未设置 `SCCACHE_DIR` 时，OpenShell 使用工作树本地的 `.cache/sccache` 目录。要让工作站上多个工作树共享缓存条目，请在激活 mise 前将该变量设为用户级目录。例如：

```shell
export SCCACHE_DIR="$HOME/.cache/openshell/sccache"
```

CI 可以选择不同目录或配置远程 sccache 后端，而无需改动工作站设置。Cargo 输出仍保留在各工作树的 `target/` 目录中。

OpenShell 不设置 `SCCACHE_BASEDIRS`。Sccache 在其机器本地守护进程启动时加载基础目录，但正确的工作区根目录因工作树而异。因此缓存复用取决于编译器输入：嵌入绝对路径的输出（某些构建中的 Rust 依赖）在不同工作树之间仍可能未命中。

## 容器镜像基础镜像覆盖

网关和 supervisor 的 Dockerfile 接受 `GATEWAY_BASE_IMAGE` 和 `SUPERVISOR_BASE_IMAGE` 构建参数。省略它们则保持固定的 distroless 默认值。要用自己的基础镜像构建，请通过对应任务