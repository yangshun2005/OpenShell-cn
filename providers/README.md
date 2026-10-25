<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 示例提供商配置档案

这些文件是可审查的示例。OpenShell 不会将它们编译进任何二进制文件，也没有网关会自动加载它们：网关的配置档案目录只包含运维人员导入的内容。

在平台作用域导入一个：

```shell
openshell provider profile lint   -f providers/github.yaml
openshell provider profile import -f providers/github.yaml --global
```

或导入整个目录：

```shell
openshell provider profile import --from providers --global
```

去掉 `--global` 则改为导入到当前工作区。

## 导入前先读文件头

每个文件开头都有一段注释，说明其预期的客户端二进制文件、这些路径所假定的镜像布局、凭据作用域、授予的端点访问权限，以及冒烟测试。请阅读它。配置档案的 `binaries` 列表是决定哪些进程可以访问其端点的控制项，而其中若干示例引用了特定参考镜像布局中的路径（`/sandbox/.venv`、`/app/.venv`、`/sandbox/.cursor-server`、`/usr/lib/node_modules/...`）。原样导入到不同的镜像中，这样的配置档案什么都匹配不到：目录仍会展示它，但凭据永远不会被注入，流量会被拒绝。

复制该文件，修改 `binaries` 和 `endpoints` 以匹配你的镜像和工作负载，然后导入你的副本。

## 调整配置档案

- 如果你的副本与示例有差异，给它一个不同的 `id`，以免两者在目录中混淆。
- 在满足工作负载的前提下，尽量收窄 `binaries`。放宽它以匹配所有镜像，会牺牲使凭据注入安全的二进制级最小权限。
- 将 `endpoints` 限制为凭据应当访问的主机。凭据只会发送到其配置档案声明的端点。
- 导入前运行 `openshell provider profile lint`。

完整 Schema 见 [提供商配置档案](https://docs.nvidia.com/openshell/latest/how-it-works/providers/profiles)。