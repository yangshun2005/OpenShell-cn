# CI

本文档说明 OpenShell 的持续集成如何针对 Pull Request 运行，重点介绍贡献者需要做什么才能让 PR 得到测试。

本地测试命令见 [TESTING.md](TESTING.md)。PR 约定见 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 概述

在 NVIDIA 自托管 runner 上运行的 PR CI 使用 NVIDIA 的 copy-pr-bot。该机器人将受信任的 PR 提交镜像到本仓库内部的 `pull-request/<N>` 分支。受门控的工作流由推送到这些分支触发，而不是由原始 PR 触发。

当 PR 未被自动镜像时，任何对本仓库拥有 Write、Maintain 或 Admin 权限的人都可以用 `/ok to test <SHA>` 接纳其当前修订。这包括拥有仓库访问权限的外部维护者。手动接纳不会改变机器人对就绪 PR 的自动信任策略。

`Branch Checks` 在 copy-pr-bot 镜像 PR 后自动运行。`Required CI Gates` 发布 PR head 状态，用于验证镜像存在、是最新的，并且运行了预期的基于推送的工作流。E2E 套件需主动启用，因为成本更高且会发布临时镜像。

合并队列验证是 `main` 的第二道集成门禁。PR 通过必需的 PR head 状态后，维护者将其加入合并队列。GitHub 创建一个临时合并组分支，合并最新的 `main`、排队的 PR 以及任何更早排队的 PR。随后，相同的必需 `OpenShell / ...` 状态上下文会针对合并组 SHA 发布，之后 GitHub 才会合并。

### Protobuf API 兼容性

`Protobuf Compatibility` 在 Branch Checks 和 Pre-release 中通过共享的 `check-protobuf-compatibility` action 和 Nix app 运行。该 app 从 flake lockfile 提供 Python、Buf 和 Git。它对 `proto/` 模块使用 Buf 的 `FILE` 策略，覆盖 SDK 描述符和扩展契约。仅用于存储的 protobuf 仍受其独立的持久性检查约束。

Branch Checks 将预期合并树与 PR 目标提交比较，因此目标上的新增内容不会在过时的 PR 中看起来像删除。合并队列使用其基础提交。两条路径都从该目标可达的标签解析 train，防止 PR 选择自己的 train。检出必须包含完整历史和标签；检查器不改变 HEAD 和工作文件。

当前 train 的版本与最新稳定版本比较。在 `0.x` 期间，次版本递增允许破坏性变更：例如稳定版 `0.1.2` 之后的 `0.2.0-pre.N`。补丁版本递增如 `0.1.3-pre.N` 则拒绝破坏性变更。提交消息不影响此决定。没有活跃 train 时也拒绝破坏性变更。允许的发现仍以警告形式可见；Schema 错误、缺少基线、合并冲突和工具失败始终导致检查失败。

发布资格将带标签的候选版本累积地与上一个稳定版本比较，使用相同的次版本对补丁版本策略。其结果属于资格配置文件的一部分：失败会阻止稳定发布，并使 Pre-release 工作流失败，但不阻止预发布发布。资格证据保留在 Actions 中。这检查 protobuf 兼容性；SDK/配置兼容性和迁移审查仍是独立的资格工作。

获取目标和标签以在本地检查已提交的分支变更，将 `origin/main` 替换为其他目标：

```shell
git fetch origin main --tags
nix run .#check-protobuf-compatibility -- origin/main
```

同一命令也接受发布标签，以将其与上一个稳定版本进行资格比较。分支名和提交 SHA 则选择分支比较：

```shell
nix run .#check-protobuf-compatibility -- refs/tags/v0.2.0-pre.1
```

对于有意的次版本 train 不兼容，请在 PR 中记录 Buf 发现、关联 Issue、消费者影响和迁移计划。保持发现可见；不要禁用该作业或添加宽泛的 Buf 忽略规则。

Windows PR 检查需主动启用：添加 `test:windows`，然后在当前 Windows MSVC 运行上选择 **Re-run all jobs**。后续镜像的提交会自动运行它们。Windows 检查不是合并所必需的，也不在合并队列中运行。Main 和手动运行也会构建发布二进制文件，并设置 `continue-on-error: true`，因此 Windows 失败不会导致工作流失败。

### 咨询性兼容性审查

`Codex Compatibility Review` 在 Branch Checks 中运行，并与带标签的发布资格一起运行。分支检查将预期合并树与其目标比较，使用工作流修订版审查工具和目标基于标签的 train 策略。发布资格审查从上一个可达稳定版本到确切候选提交的累积 diff，使用与 protobuf 检查相同的基于标签的补丁/次版本策略。它审查公共 Rust、Python、Go 和 TypeScript SDK、API 行为、CLI 契约、配置、策略、Helm 和持久化状态。两个 Codex 工作流共享 `codex-release-review` 并发组，设置 `queue: max`，因此一次只运行一个审查，并且在分支、发布和手动运行之间最多保留 100 个待处理审查。发布兼容性审查独立于 Security Scan 调度；其他安全扫描器并行运行。它不替代 Buf，也不改变发布 train 的版本。

该 agent 先检查代码，然后使用带有只读 GitHub token 的 `gh` 查找每个发现的引入 PR 及其讨论。发布审查从提交关联中解析引入 PR。只有当 PR 作者或已验证的维护者明确接受特定风险时，发现才显示 `ACKED`。无法确定接受情况时，缺少上下文会产生 `UNKNOWN`；确认不会覆盖发布策略。报告以具体变更、消费者影响、操作和 PR 开头，覆盖详情折叠。

带发现的镜像 PR 分支运行在每次运行时（包括重跑）发布新的 PR 评论。无发现的运行不发布评论。发布者在评论前检查 PR 在审查的 head 处仍处于打开状态。合并组和手动分支运行保留报告但不发布 PR 评论。审查者具有读访问权限；单独的分支作业有权限发布评论。

审查者使用 Python Codex SDK（`openai-codex==0.160.0`）及其捆绑运行时，通过相同的 NVIDIA 端点、模型、中等推理努力和 `CODEX_SECURITY_API_KEY` 作为 Codex Security。工作流使用现有 Nix devShell 中的 `uv run`；SDK 不是 OpenShell 包依赖。它在 GitHub 托管的 runner 上运行，具有只读沙箱并读取裸 Git 仓库；候选侧的 agent 配置和指令不会被加载。其权限配置文件保持文件只读，并通过 Codex 网络代理路由 shell 网络访问，仅允许 `api.github.com` 用于 GitHub 查找。上游代理链和 SOCKS 访问被禁用。只有 GitHub token 和工具/证书路径传递给 agent shell 命令；推理凭据被排除。它不运行候选构建或测试。审查在 40 分钟作业超时内有 30 分钟执行限制。

此推出为仅报告：发现和审查失败仍为咨询性。资格调用方等待审查完成，但其结果不影响必需配置文件。其摘要区分完整、不完整和失败的执行与 agent 的兼容性评估。缺少凭据、无效基线、源不匹配、格式错误的报告和超时都不是干净的审查。完成的审查仍不能证明不存在破坏。

每次运行在 `compatibility-review-run-<run-id>-attempt-<attempt>` 中保留 `report.json` 和 `report.md` 90 天。报告记录两个修订、train、工具/模型元数据、覆盖缺口和可操作发现。原始 agent 转录和 PR 讨论不发布。资格摘要链接此独立咨询证据而不等待它；在此推出期间，兼容性审查仍缺失于阻塞性 RFC 配置文件中。

工作流在默认分支可用后，运行审查而不构建或发布版本：

```shell
gh workflow run codex-compatibility.yml -f candidate_ref=v0.1.3-pre.1
```

使用与被审查 train 相适应的现有发布标签。在决定修复发现、记录有意的次版本发布破坏或调查不确定结果之前，检查报告。启用阻塞性 agent 门禁是校准后的独立工作。

`Branch E2E Checks` 仅在选择 E2E 套件时构建二进制文件和镜像。DEB 和 RPM 包仅在选择集成资格时构建。核心集成资格在 Ubuntu 上使用 Docker 构建并安装 DEB，并在 Fedora 上使用 rootful 和 rootless Podman 安装 CLI 和 gateway RPM。这些通道使用匹配的运行时镜像运行一致性测试。Release Dev、Release 和 Pre-release 使用相同的包安装器。Fedora provider-refresh 测试也使用 RPM。Podman 驱动特定套件保留二进制安装器，因为其 fixture 配置其系统服务、本地 HTTP gateway 和 CLI 路径。手动 Integration Tests 工作流默认使用包安装器，并下载其矩阵选择的包。

三个主动启用的标签启用长时间运行的 E2E 套件：

- `test:e2e` 在 `Branch E2E Checks` 中运行 Docker、rootless Podman、Kubernetes 和 VM E2E 套件，使用托管和独立计算驱动
- `test:e2e-gpu` 在 `Branch E2E Checks` 中运行 GPU E2E
- `test:e2e-kubernetes` 在 `Branch E2E Checks` 中运行 Kubernetes E2E，使用 HA Helm overlay（`replicaCount: 2` 和捆绑 PostgreSQL）以及凭据驱动套件（Kubernetes Secrets 加 Vault）

当存在多个标签时，`Branch E2E Checks` 为每个通用多架构构建产物集构建一次，并并行展开启用的套件。运行时特定的可复用工作流定义 Docker、Podman、VM 和 Kubernetes 通道。复合 action 负责可替换的 Podman、KVM、kind 和 mise 设置。每个通道仅依赖其消费的构建产物类别：VM 不等待容器驱动构建产物或 supervisor 镜像，GPU 不等待 gateway 镜像。Docker、Podman、GPU、Rust、Python、MCP 和 VM E2E 复用匹配的预构建 gateway 和 CLI 二进制文件，而不是在测试作业中编译调试二进制文件。独立驱动通道还复用无驱动 gateway 和计算驱动构建产物。Kubernetes 托管驱动通道消费已发布的 gateway 和 supervisor 镜像，而独立驱动通道从预构建二进制文件组合其 gateway 镜像。`OpenShell / E2E` 和 `OpenShell / GPU E2E` 必需状态在该工作流内的单独套件结果作业中评估。`test:e2e-kubernetes` 是可选的，因为 Kubernetes HA 和凭据驱动行为正在积极迭代：失败在工作流运行中可见，但不发布必需的 CI 门禁状态。

GitHub ruleset 应要求由 `Required CI Gates` 发布的 `OpenShell / ...` 状态加上直接的 `OpenShell / Trivy Changes` 结果，而不是推送触发的工作流作业本身。

### K3s 一致性版本基线

tmachine `ubuntu-k3s` 一致性通道将 Agent Sandbox v0.5.0 固定为 v1beta1 Sandbox API 的兼容性基线。它们不跟踪本地 K3s 开发默认值，当前为 v1.0.3。OpenShell 还通过其 v1alpha1 回退支持 v0.4.6，因此 v0.5.0 不是整体最低支持版本。

`ubuntu-k3s` 通道直接注册 gateway 的 ClusterIP Service，因此重建 Service 需要重新配置 fixture。`k3s` 通道运行一个明文 gateway。`k3s-ha` 通道运行支持的 HA 配置：三个 gateway 副本共享 PostgreSQL，提供 TLS，CLI 通过 mTLS 连接。在一致性测试前，每个安装器最多等待五分钟等待连接的 gateway；交互式 shell 跳过等待。K3s 失败将诊断信息上传为 `tmachine-diagnostics-*` 构建产物。

### 仅运行策略顾问一致性测试

在候选分支上手动调度 `Integration Tests`，使用来自同一提交构建的 `artifact-run-id`。当 `Release` 或 `Pre-release` 运行从不同提交调度时，还要将 `source-sha` 设置为发布标签解析的提交，以便运行时镜像和测试输入与候选二进制文件匹配。否则，`source-sha` 默认为构建产物运行的 head SHA。将 `category` 设置为 `policy-advisor`，`test-matrix` 设置为：

```json
[{"environment":"ubuntu-docker-rootful","installer":"binaries","testsuite":"policy-advisor"}]
```

这在已安装构建产物套件中运行 `mechanistic-proposal`、`new-hostname-proposal` 和 `policy-local` 一致性测试。构建产物运行必须包含候选 CLI 和 gateway 二进制文件以及运行时镜像。此手动运行不替代必需的 PR E2E 门禁。

## 信息性安全报告

安全工作流计算直接在 GitHub 托管的 runner 上运行，而不是 NVIDIA 自托管 runner。面向 PR 的工作流不接收机密，并在 fork Pull Request 上运行而不等待 copy-pr-bot。Codex Security 发布资格是例外：它仅对维护者创建的预发布标签运行，并在扫描步骤接收作用域限定的 NVIDIA Inference API 密钥。扫描器作业请求 `security-events: write` 以将 SARIF 发布到 Code Scanning。GitHub 允许来自 `pull_request` 运行的 Code Scanning 上传，即使 fork 和 Dependabot 上下文接收只读 `GITHUB_TOKEN`，因此这些扫描器直接上传结果：

- `Workflow Security Reports` 运行 Actionlint 和 Zizmor。Actionlint 报告工作流语法和表达式发现。Zizmor 仅报告 High 严重性，即其最高级别。Nix 提供两个扫描器。它们将 SARIF 发布到 Code Scanning 并保留报告构建产物。
- `Dependency Review` 比较基础和目标依赖图，并以 High 或更高策略报告新引入的漏洞。它以仅警告模式运行。预检将不可用的 GitHub Dependency Graph 转为警告，因此工作流在仓库功能可用前保持中性。
- `CodeQL` 分析产品 Rust 代码、示例以及 Go、Python 和 TypeScript SDK。Rust `cfg(test)` 块、Rust 集成测试目标和 E2E 测试代码被排除。它在 `main` 上每晚运行，保持可手动调度用于诊断，将结果上传到 Code Scanning，并保留工作流构建产物。
- `Codex Security Release Qualification` 可以手动分析 `vX.Y.Z-pre.N` 候选版本与上一个稳定版本的对比。它暂时被排除在带标签发布资格之外。每次扫描审查从稳定版到候选版的累积 diff。它调用 `https://inference-api.nvidia.com/v1`，使用 `openai/openai/gpt-5.6-sol` 和中等推理努力。`CODEX_SECURITY_API_KEY` 机密必须包含授权用于该 NVIDIA 端点的 API 密钥。结果针对 `main` 上的候选提交上传，位于 train 共享的类别下，例如 `codex-security/v0.1.1`，因此后续候选会替换早期分析。斜杠限定的 NVIDIA 模型标识符阻止 Codex Security 0.1.24 强制执行 `--max-cost`，因此工作流依赖其超时、串行并发和推理账户的支出控制。原始报告不作为工作流构建产物保留。预发布创建和稳定提升强制执行仍是 RFC 0014 的一部分，不由本工作流实现。

发现不会使这些工作流失败。工具启动、配置、构建和分析失败仍会失败，因此损坏的扫描器不能显得健康。面向 PR 的报告也在合并组上运行；CodeQL 不是 PR 或合并队列检查。这些报告都不是必需状态或门禁合并。

使用以下命令在本地运行工作流定义扫描器：

```shell
nix develop --command actionlint -shellcheck= -pyflakes=
nix develop --command zizmor --offline --persona=regular --min-severity=high --no-exit-codes .
```

## 一起运行安全扫描

`Security Scan`（`.github/workflows/security-scan.yml`）为发布标签并行调用 CodeQL、Trivy、Cargo Deny 和 Workflow Security Reports（Actionlint/Zizmor）。Codex Security 暂时被排除；其独立工作流仍可用于手动调查。从 Actions 手动运行父工作流或从另一个工作流调用它。所有适用的子工作流分析候选快照。Cargo Deny 使用其现有的 NVIDIA 自托管 runner 和 CI 容器。

带标签发布将 Cargo Deny 发现视为当前实现的资格配置文件的失败。配置文件失败阻止稳定发布。预发布发布与资格并行运行；资格失败使父工作流失败并发送失败通知，而已发布的构建产物仍可用。CodeQL、Trivy 和 Zizmor 发现对带标签发布暂时为信息性：现有发现已审查并为 v0.1.0 接受，将在 0.1.x 发布中处理。扫描器失败仍使资格失败。

失败的预发布构建、发布或资格通过 webhook 发送 Slack 通知，并 @ 提及分诊工程师，附有每个失败作业和运行的链接。webhook 存储在仓库机密 `SLACK_OPENSHELL_TRIAGE_WEBHOOK_URL` 中，提及存储在仓库机密 `SLACK_OPENSHELL_TRIAGE_MENTION` 中。成功的预发布构建向同一 webhook 发布顶层消息而不提及，因此失败后的恢复在频道中可见。通知是非阻塞的，不影响发布结果；未设置 webhook 则跳过发送；未设置提及则发送时不提及。

```shell
gh workflow run security-scan.yml --ref main \
  -f candidate_ref=v0.1.1-pre.1 \
  -F fail-on-static-findings=false
```

要将其集成到更大的工作流中，请在创建候选标签并为扫描暂存源寻址构建产物的作业之后运行它。稳定发布在资格成功后进行；预发布发布与其并行运行。此示例假设存在一个 `build` 作业，其输出名为 `candidate_tag`、`gateway_image`、`sandbox_image` 和 `chart_ref`；请根据你的工作流调整这些名称：

```yaml
jobs:
  security:
    needs: build
    uses: ./.github/workflows/security-scan.yml
    permissions:
      actions: read
      contents: read
      packages: read
      security-events: write
    with:
      candidate_ref: ${{ needs.build.outputs.candidate_tag }}
      images: |
        ${{
```