# Fern 文档站点

OpenShell 使用 [Fern](https://buildwithfern.com/) 校验、预览并发布 [docs.nvidia.com/openshell](https://docs.nvidia.com/openshell/) 上的文档。本目录存放站点配置和展示文件，文档内容位于 `docs/`。

## 仓库结构

| 路径 | 用途 |
|---|---|
| `docs/` | MDX 页面、`docs/index.yml` 中的导航，以及页面级组件。 |
| `fern/docs.yml` | 站点、主题、版本、重定向和导航配置。 |
| `fern/fern.config.json` | Fern 组织信息和固定的 CLI 版本。 |
| `fern/components/` | 共享站点组件。 |
| `fern/assets/` | Logo 及其他共享资源。 |
| `fern/main.css` | 全站样式。 |

在常规源码检出中，`fern/docs.yml` 将 `dev` 版本指向 `docs/index.yml`。发布自动化在生成的 `docs-website` 分支上构建多版本配置，并将源码文档映射到正在发布的渠道。

## 本地开发

在仓库根目录启动本地 Fern 服务器：

```shell
mise run docs:serve
```

不启动服务器，仅校验配置、导航和链接：

```shell
mise run docs
```

这些任务从 `fern/fern.config.json` 读取 Fern CLI 版本，因此本地检查和 GitHub Actions 使用同一版本。编写规范和风格指南见 [docs/CONTRIBUTING.mdx](../docs/CONTRIBUTING.mdx)。

## Pull Request 预览

`.github/workflows/branch-docs.yml` 校验修改文档或 Fern 配置的 Pull Request。当工作流能访问 `FERN_TOKEN` 时，它会基于 Pull Request 检出发布 Fern 预览，并将预览 URL 添加到 Pull Request。此路径不使用也不更新 `docs-website` 分支。

## 带版本的生产站点

生成的 `docs-website` 分支包含 Fern 的完整生产输入。每个版本在 `fern/pages-<slug>/` 下有一份其源提交 `docs/` 目录树的精确副本，并在 `fern/versions/` 下有一个导航文件。`fern/.docs-snapshots.yml` 记录每个受管快照的原始源 ref、解析后的源提交和发布版本。

自动化站点使用以下版本类型：

| 版本 | 来源 | 更新策略 |
|---|---|---|
| `latest` | 最新的稳定发布版本。 | 可变。早于当前稳定版本的维护版本不能将其回退，除非维护者明确允许回滚。 |
| `dev` | `main` 上最近一次成功的 Release Dev 运行。 | 可变。自动化会拒绝较旧的版本，或来自不同提交的相同版本，除非维护者明确允许回滚。 |
| `vX.Y.Z` | 打标签的发布提交。 | 不可变。后续同步不能替换其源提交。 |

Release Dev 等待开发构建产物和 Helm chart，然后调用一次 `.github/workflows/sync-docs.yml`。该可复用工作流更新 `dev`，校验生成的站点，按需提交并推送分支，并发布一次生产站点。

Release 在发布构建产物、SDK 包、Helm chart 和 wheel 发布完成后，通过 `release-publish.yml` 对稳定标签执行相同流程。当发布版本不早于当前版本时，一次同步会将打标签的源码复制到不可变的 `vX.Y.Z` 快照和 `latest`，然后发布一次生产站点。版本选择器优先固定 `latest` 和 `dev`，随后按版本号降序排列版本化快照，其余历史条目按原有顺序排列。

同步工作流仍接受可选的 Fern 可用性徽章，但当前的 Release Dev 和 Release 任务不请求该徽章。下一次 `dev` 同步会用无徽章条目替换现有的 `dev` Beta 徽章。其他条目保留其徽章设置，直到该条目被同步或移除。

同步和发布工作流共享 `docs-website` 并发组，从而串行化写入和发布。排队的运行保持等待，不会互相替换。

`dev` 快照还负责 `docs-website` 上共享的 Fern 配置、组件、资源和 CSS。`latest` 快照复制其文档、导航和重定向，但不替换共享组件、资源或 CSS。这样站点配置与 `main` 保持一致，同时保留已发布的内容。

`dev` 同步会复制源 `fern/docs.yml` 中的顶层 `announcement`。该公告是全局回退，从源中移除它也会从 `docs-website` 中移除。每个快照同步仅将源版本公告复制到正在更新的渠道。版本公告会覆盖该版本的全局公告，因此 Release Dev 不能更改 `latest` 公告，Release 不能更改 `dev` 公告。

重定向与其可变快照同步。源以 `/openshell/dev/` 或 `/openshell/latest/` 开头的重定向属于该渠道。指向版本化目标的非版本化别名属于目标渠道；其他共享规则属于 `dev`。每次同步替换该渠道的规则，包括移除源中不存在的规则。稳定发布仅在提升 `latest` 时更新 `latest` 重定向，因此较旧的维护版本不能回滚线上路由。其他版本拥有的重定向保持不变。

## 手动维护与发布

维护者可以手动运行 `.github/workflows/sync-docs.yml` 来添加、刷新或移除历史版本快照。要在不更改内容的情况下修复可变渠道的过期重定向，请根据 `fern/.docs-snapshots.yml` 中记录的源提交和发布版本同步该渠道，并保留 `fern/docs.yml` 中的显示名称和可用性。使用 `main` 上更新后的自动化；仅在准备发布修复时选择生产发布。工作流会保留未选中的快照。手动同步默认禁用生产发布。

`.github/workflows/publish-docs-website.yml` 校验并发布现有的 `docs-website` 分支，不同步内容。其默认模式创建预览。选择生产模式会发布线上站点，因此仅用于有意的生产重新发布。

更改版本模型或任一发布工作流后，运行自动化同步测试：

```shell
mise run test:docs-website
```