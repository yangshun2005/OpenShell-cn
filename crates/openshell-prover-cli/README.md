<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# OpenShell 策略证明器 CLI

本包构建独立的 `openshell-prover` 可执行文件。它是 `openshell-prover` 中可复用隔离引擎的轻量同步适配层，负责本地文件加载、命令解析、结果渲染和进程退出码。

```shell
openshell-prover check candidate.yaml --boundary boundary.yaml
openshell-prover check candidate.yaml --boundary boundary.yaml --output json
```

该命令检查完整组合后的候选策略是否处于运维人员提供的边界之内。它不发现网关、不获取策略状态，也不应用策略变更。

结果使用以下退出码：

| 退出码 | 含义 |
| --- | --- |
| `0` | 候选策略在边界之内。 |
| `1` | 候选策略超出边界。 |
| `2` | 用法、输入、输出或内部错误。 |
| `3` | 不支持的策略语义或求解无定论。 |
| `130` | 在 Unix 上被 Ctrl-C 中断；优雅的 JSON 输出会报告无定论的取消。 |

构建并测试本包：

```shell
cargo build -p openshell-prover-cli --bin openshell-prover
cargo test -p openshell-prover-cli
```

安装后的用法与结果解读指南见[策略证明器参考](../../docs/how-it-works/policies/prover.mdx)。

JSON 输出使用数值型 `schema_version` 表示结果契约，使用 `prover_version` 表示产生该结果的实现版本。使用方必须检查 `coverage.domains` 以获取机器可读的建模域声明。检查通过仅表示在文档所述假设下配置相符；它并不证明运行中的沙箱已安装其限制。