<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 测试客户机

此原型使用 Nix、QEMU 和 Ansible 启动并配置一次性 Linux 虚拟机，用于测试 OpenShell 包和二进制文件。它支持 Apple Silicon macOS 上的 HVF、原生架构 Linux 主机上的 KVM，以及 Linux 上 KVM 不可用时的较慢 TCG 回退。

## 要求

- 启用 flakes 的 Nix。
- 带 HVF 的 Apple Silicon macOS，或原生架构 Linux 主机。Linux 在 `/dev/kvm` 可用时使用 KVM，否则回退到 QEMU TCG。
- 足够的本地容量，用于四 vCPU、4 GiB 客户机和一次性磁盘覆盖层。
- 原生架构构建产物。TCG 在 Linux 上模拟客户机 CPU，但不支持跨架构客户机。

首次运行会下载选定的云镜像和虚拟机运行时。Nix 在后续运行中复用这些不可变输入，而每个客户机都从全新的可写覆盖层启动。

在 Apple Silicon 上，测试客户机特意从单独固定的 Nixpkgs 修订版中取用 QEMU 11.0.2 及其匹配的 OVMF 固件。所有其他客户机运行时工具、开发依赖和交叉工具链依赖继续使用主当前 Nixpkgs 输入。这避免了 QEMU 11.1 的 HVF 客户机启动回归。仅在验证 ARM64 Ubuntu 客户机可在 HVF 下启动后，才更新这一对。

## 目录结构

```text
nix/test-guest/
├── README.md
├── default.nix
├── run.sh
├── cache.sh
├── cache-lib.sh
├── cache-seal.sh
├── distros/
│   ├── ubuntu-24-04.nix
│   ├── ubuntu-26-04.nix
│   ├── centos.nix
│   ├── fedora.nix
│   └── rocky.nix
└── configuration/
    ├── docker.yml
    ├── podman-rootful.yml
    ├── podman-rootless.yml
    ├── tasks/
    │   ├── podman-common.yml
    │   └── podman-rootless/
    │       ├── fedora.yml
    │       ├── shared.yml
    │       └── ubuntu.yml
    └── selinux.yml
└── provisioners/
    └── roles/
        ├── gateway-podman/
        ├── openshell-development/
        └── openshell-rpm/
```

- `default.nix` 组装客户机和缓存 flake 应用。它选择主机架构和加速方式，提供运行时工具，并将发行版配置和配置 playbook 作为 Nix 存储目录暴露。
- `run.sh` 负责一次性客户机生命周期：缓存查找、云镜像实现、cloud-init 种子创建、QEMU 启动、SSH 就绪、Ansible 执行、构建产物安装、客户机命令执行和清理。
- `cache.sh` 确保本地存在精确准备好的磁盘。它可以拉取或显式推送磁盘作为 OCI 构建产物。
- `cache-lib.sh` 定义运行器和缓存命令共享的确定性缓存标识和验证辅助函数。
- `cache-seal.sh` 在捕获前移除每实例状态并清零准备好的客户机中的空闲空间。
- `distros/*.nix` 定义不可变基础镜像目录。每条记录固定并导出镜像 URL 和哈希，并声明预期的 OS ID、版本和包族。
- `configuration/*.yml` 是主机执行的 Ansible playbook，将可选能力叠加到基础客户机上。配置保持独立，并按重复 `--with` 参数提供的顺序运行。
- `provisioners/roles/*` 是每次运行的 Ansible 角色，从复制或安装的构建产物组装系统。它们在构建产物传输后运行，从不属于准备好的 VM 缓存。
- `README.md` 记录支持的组合和开发者接口。

根 [`flake.nix`](../../flake.nix) 将此目录暴露为 `test-guest` 和 `test-guest-cache` 应用。Debian 构建产物创建仍在客户机测试框架之外的 [`tasks/scripts/package-deb.sh`](../../tasks/scripts/package-deb.sh) 中；运行器仅安装或复制已存在的构建产物。

## 支持的配置

| 发行版 | Docker | Rootful Podman | Rootless Podman | SELinux | 包格式 |
| --- | --- | --- | --- | --- | --- |
| Ubuntu 24.04 | 是 | 是 | 否 | 否 | `.deb` |
| Ubuntu 26.04 | 是 | 是 | 是 | 否 | `.deb` |
| CentOS Stream 10 | 否 | 是 | 否 | 是 | `.rpm` |
| Fedora 44 | 否 | 是 | 是 | 是 | `.rpm` |
| Rocky Linux 9 | 是 | 是 | 否 | 是 | `.rpm` |

`snapd` 配置适用于 Ubuntu，为 snap Store 安装实验准备 snapd。将其与 `docker` 组合可复现系统 Docker 金丝雀测试，或单独使用以验证 `install.sh` 在 Docker 缺失时安全失败。

`podman-rootless` 配置 OpenShell 测试使用的显式 rootless Podman 客户机设置。它支持 Fedora 和 Ubuntu 26.04 或更高版本。Ubuntu 添加 AppArmor 规则，允许 `pasta` 接收 Podman 停止信号。Fedora 安装从属 ID 工具以及 rootless 存储和网络辅助工具。两种配置都验证 rootless 模式和 OpenShell 沙箱回调所需的 `pasta` 网络辅助工具。Ubuntu 24.04 附带 Podman 4，不提供该辅助工具。

`podman-rootful` 安装 Podman，启用其系统 API socket，并为 `gateway-podman` provisioner 记录所选模式。随后 provisioner 以 root 启动选定的 OpenShell 安装。它不写入 rootful 专用网关设置；网关会发现其服务账户可用的 Podman socket。rootless 配置以相同方式记录其模式，并以 `openshell` 用户运行网关。

列出可用的发行版和配置：

```shell
nix run .#test-guest -- --list
```

## 打开交互式 VM

启动基础 Ubuntu VM：

```shell
nix run .#test-guest -- --distro ubuntu-24-04
```

在打开 SSH 会话前应用 Docker 配置：

```shell
nix run .#test-guest -- --distro ubuntu-24-04 --with docker
```

其他组合使用相同接口：

```shell
nix run .#test-guest -- --distro rocky --with docker
nix run .#test-guest -- --distro fedora --with podman-rootful
nix run .#test-guest -- --distro ubuntu-26-04 --with podman-rootless
nix run .#test-guest -- --distro fedora --with podman-rootless
```

配置可重复：

```shell
nix run .#test-guest -- \
  --distro ubuntu-24-04 \
  --with docker \
  --with podman-rootless
```

确保 CentOS、Fedora 或 Rocky 上 SELinux 处于强制模式：

```shell
nix run .#test-guest -- \
  --distro rocky \
  --with docker \
  --with selinux \
  -- getenforce
```

`--with selinux` 安装所需工具，持久化 `SELINUX=enforcing`，实时应用强制模式并验证结果。它在 Ubuntu 以及 SELinux 完全禁用且需要重启才能启用的客户机上失败。

## Ansible 配置

配置是存储在 `nix/test-guest/configuration/` 下的 Ansible playbook。Ansible 在主机上使用 VM 的临时 SSH 密钥和回环端口运行。客户机不安装 Ansible。

配置按命令行提供的顺序运行。OpenShell 包和复制的文件在所有配置成功后安装。

`--install` 包和 `--copy` 文件由专用的每次运行传输步骤应用。`--copy` 保留每个源文件的普通权限位，除非提供八进制模式。它们不存储在准备好的 VM 缓存条目中。

## 系统 provisioner

`--provision NAME` 在包和复制的构建产物就位后应用目标特定的系统设置。与 `--with` 不同，provisioner 不缓存。因此它们可以安装并启动 OpenShell 系统，而不会将准备好的客户机镜像耦合到特定构建或驱动配置。

`openshell-development` 期望以下复制的客户机路径：

- `/usr/local/bin/openshell`
- `/usr/local/bin/openshell-gateway`
- `/usr/local/lib/openshell-sandbox.tar`

在任一 Podman 配置后与 `gateway-podman` 组合。该角色使用记录的模式选择相应的服务账户。它仅为开发构建产物生成配置；RPM 安装保留其打包的服务和首次启动配置。例如，在 rootless provisioner 完成后检查开发网关：

```shell
nix run .#test-guest -- \
  --distro fedora --with podman-rootless --with selinux \
  --copy ./openshell:/usr/local/bin/openshell \
  --copy ./openshell-gateway:/usr/local/bin/openshell-gateway \
  --copy ./openshell-sandbox.tar:/usr/local/lib/openshell-sandbox.tar \
  --provision openshell-development \
  --provision gateway-podman \
  -- /usr/local/bin/openshell status
```

`openshell-rpm` 期望已使用 `--install` 安装 OpenShell。它使用 RPM 拥有的 `/usr/bin` 二进制文件和 `openshell-gateway` 用户服务，不使用复制的开发构建产物或 supervisor 归档。与 `gateway-podman` 组合，在运行冒烟一致性测试前启动已安装版本。

## 准备好的 VM 缓存

`test-guest-cache` 应用确保为某个精确发行版、主机架构和有序配置列表存在准备好的磁盘。它首先检查本地缓存，可选拉取匹配的 OCI 构建产物，或在未命中时构建并验证新的本地条目：

```shell
nix run .#test-guest-cache -- \
  --distro ubuntu-24-04 \
  --with docker
```

配置 OCI 仓库和受信任的清单摘要，将其用作共享后备缓存：

```shell
nix run .#test-guest-cache -- \
  --distro ubuntu-24-04 \
  --with docker \
  --repository ghcr.io/nvidia/openshell/test-guest-cache \
  --digest sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
```

该命令从不隐式发布。通过 ORAS 的 Docker 兼容凭据配置认证后，添加 `--push`：

```shell
nix run .#test-guest-cache -- \
  --distro ubuntu-24-04 \
  --with docker \
  --repository ghcr.io/nvidia/openshell/test-guest-cache \
  --push
```

成功推送会打印不可变的 `repository@sha256:...` 引用。通过受信任的 CI 配置将该摘要提供给消费者。不允许按可变标签拉取。拉取的本地条目记录其清单摘要，仅在匹配请求的受信任摘要时复用。

缓存构建会启动并配置一次性 VM，运行内部密封脚本，将覆盖层展平为独立 QCOW2 磁盘，并在提交条目之前验证全新启动。OCI 构建产物包含元数据和 `disk.qcow2.zst` 层。

键包括固定的基础镜像标识、客户机架构、有序配置文件摘要、Ansible 版本、缓存代次和密封脚本摘要。已安装的包、复制的二进制文件、转发的端口和客户机命令从不缓存。

正常 `test-guest` 运行在重新检查磁盘校验和和 QCOW2 结构后，自动使用精确有效的本地条目。本地未命中时，运行器调用缓存构建器并在继续前存储准备好的磁盘。然后它从该条目创建全新的可写覆盖层、cloud-init 实例、机器 ID 和 SSH 身份。设置 `OPENSHELL_TEST_GUEST_CACHE_DISABLE=1` 可绕过本地查找和自动填充。

默认缓存目录为 `${XDG_CACHE_HOME:-$HOME/.cache}/openshell/test-guest`。在缓存命令上使用 `--cache-dir` 覆盖，或对任一应用使用 `OPENSHELL_TEST_GUEST_CACHE_DIR`。

缓存命令选项：

```text
--distro NAME       Base distro: ubuntu-24-04, ubuntu-26-04, centos, fedora, or rocky
--with NAME         Apply docker, podman-rootful, podman-rootless, selinux, or snapd; repeatable
--repository REF    OCI repository without a tag
--digest DIGEST     Trusted OCI manifest digest required for pulls
--cache-dir PATH    Override the local prepared-disk cache directory
--push              Publish the ensured entry to the repository
```

## 安装 OpenShell 包

使用仓库的 `package:deb:arm64` mise 任务打包现有 ARM64 Linux 二进制文件：

```shell
OPENSHELL_CLI_BINARY="$PWD/target/aarch64-unknown-linux-musl/release/openshell" \
OPENSHELL_GATEWAY_BINARY="$PWD/target/aarch64-unknown-linux-gnu/release/openshell-gateway" \
OPENSHELL_DRIVER_VM_BINARY="$PWD/target/aarch64-unknown-linux-gnu/release/openshell-driver-vm" \
OPENSHELL_DEB_VERSION=0.0.0-local \
OPENSHELL_OUTPUT_DIR="$PWD/artifacts" \
nix develop --command mise run package:deb:arm64
```

在 Ubuntu VM 中安装包并运行命令：

```shell
nix run .#test-guest -- \
  --distro ubuntu-24-04 \
  --with docker \
  --install artifacts/openshell_0.0.0-local_arm64.deb \
  -- openshell --version
```

对于 x86_64 Linux 客户机，提供 x86_64 二进制文件并使用 `package:deb:amd64`。包架构必须与主机和客户机架构匹配。

`--install` 可重复。Ubuntu 接受 Debian 包；CentOS、Fedora 和 Rocky Linux 接受 RPM 包。此原型可以安装现有 RPM，但不构建 RPM。

## 直接复制文件

使用 `--copy SOURCE:DEST[:MODE]` 复制常规文件而不创建包。如果省略 `MODE`，客户机文件保留源文件的普通权限位。提供裸八进制模式如 `755` 以覆盖它们：

```shell
nix run .#test-guest -- \
  --distro ubuntu-24-04 \
  --copy ./openshell:/usr/local/bin/openshell:755 \
  -- openshell --version
```

目标必须是绝对客户机路径。显式模式使用 `000` 到 `777` 的裸八进制权限位。

## 运行器选项

```text
--distro NAME       Base distro: ubuntu-24-04, ubuntu-26-04, centos, fedora, or rocky
--with NAME         Apply docker, podman-rootful, podman-rootless, selinux, or snapd; repeatable
--install PATH      Install a .deb or .rpm package; repeatable
--copy SRC:DEST[:MODE]
                    Copy a regular file into the guest; use MODE when provided,
                    otherwise preserve the host mode; repeatable
--ssh-port PORT     Use a specific loopback SSH forwarding port
--forward-port HOST_PORT:GUEST_PORT
                    Forward a loopback host port to a guest port; repeatable
--keep              Preserve the disk overlay and logs after shutdown
--list              List distros, configurations, and provisioners
```

每个 `--forward-port` 仅在主机上绑定 `127.0.0.1`。两个端口必须是 1024 到 65535 的非特权值，每个主机端口只能出现一次。

`--` 之后的参数在客户机内执行。没有命令时，运行器打开交互式 SSH 会话。

## 生命周期

每次调用确保存在精确准备好的本地缓存条目。未命中时，缓存构建器实现哈希固定的云镜像，应用选定配置，密封并验证准备好的磁盘，并将其存储在本地。然后运行器：

1. 创建由准备好的缓存磁盘或固定云镜像支持临时 QCOW2 覆盖层。
2. 使用 HVF、KVM 或 Linux TCG 回退启动 QEMU。
3. 创建全新的 cloud-init 实例和临时 SSH 密钥。
4. 仅当基础未准备好时应用选定的 Ansible 配置。
5. 安装或复制提供的构建产物。
6. 打开 SSH 或执行请求的客户机命令。
7. 关闭 QEMU 并删除可写覆盖层。

准备好的缓存磁盘保持只读。测试特定状态仅存在于一次性覆盖层中。

使用 `--keep` 保留覆盖层、cloud-init 种子、SSH 密钥和串行日志以进行调试。运行器退出时会打印保留的目录。

## 当前限制

- 主机和客户机架构必须匹配。
- TCG 比硬件虚拟化慢，并使用更长的 SSH 就绪超时。
- 准备好的缓存条目是架构特定的，并匹配精确的有序配置列表。
- OCI 拉取传输完整的压缩独立磁盘；未实现增量磁盘层。
- 客户机端口仅在通过仅回环的 `--forward-port` 显式暴露时才能从主机访问。
- 运行器不构建 OpenShell、配置网关或选择 E2E 测试套件。