# VM 驱动运行时

> 状态：实验性。VM 支持正在积极开发中，可能发生变化。

本目录管理 `openshell-driver-vm` 的固定运行时输入：

```text
runtime/
  pins.env
  kernel/
    openshell.kconfig
```

`openshell-driver-vm` 内嵌 libkrun、libkrunfw、umoci（用于客户机侧 OCI 镜像解包），以及可移植、无特权的 `openshell-sandbox` 角色。
VM 不挂载客户机 NIC。边界通过已认证的 vsock 通道承载控制流、中介网络流和 DNS 流。

## 原因

原版 `libkrunfw` 内核不包含沙箱在每个 microVM 内所需的全部 cgroup、seccomp 和 Landlock 特性。
`kernel/openshell.kconfig` 扩展了 libkrunfw 内核，使 VM 沙箱在 supervisor 运行于宿主机时，仍保留客户机本地的进程、网络系统调用和文件系统强制限制。

## 构建脚本

| 脚本 | 平台 | 用途 |
|---|---|---|
| `tasks/scripts/vm/build-libkrun.sh` | Linux | 使用自定义内核配置从源码构建 libkrunfw 和 libkrun |
| `tasks/scripts/vm/build-libkrun-macos.sh` | macOS | 从预构建的 `kernel.c` 构建可移植的 libkrunfw 和 libkrun |
| `tasks/scripts/vm/package-vm-runtime.sh` | 任意 | 打包 `vm-runtime-<platform>.tar.zst`，包含库、umoci 和来源信息 |
| `tasks/scripts/vm/download-kernel-runtime.sh` | 任意 | 从 `vm-runtime` 发布版本下载运行时 tarball，并暂存压缩文件 |

## 本地流程

```shell
# 下载当前预构建运行时并暂存压缩产物
mise run vm:setup

# 构建可移植的 Linux 客户机沙箱和静态 guest-init 辅助程序
mise run vm:supervisor

# 构建网关、原生宿主机 supervisor 和 VM 驱动
OPENSHELL_VM_RUNTIME_COMPRESSED_DIR=$PWD/target/vm-runtime-compressed \
  cargo build -p openshell-gateway -p openshell-supervisor -p openshell-driver-vm
```

使用 `FROM_SOURCE=1 mise run vm:setup` 从源码构建运行时，而不是下载 `vm-runtime-<platform>.tar.zst`。

## CI 归属

`release-vm-kernel.yml` 是按需生产者，负责生成：

- `vm-runtime-linux-aarch64.tar.zst`
- `vm-runtime-linux-x86_64.tar.zst`
- `vm-runtime-darwin-aarch64.tar.zst`

这些构建产物保留在滚动更新的 `vm-runtime` 发布版本上。常规 `dev` 和 `v*` 发布工作流会下载它们，嵌入 `openshell-driver-vm`，并将驱动二进制文件发布在 `openshell-gateway` 旁边。

## 来源信息

`package-vm-runtime.sh` 会在每个运行时 tarball 中写入 `provenance.json`，包含平台、libkrunfw 提交、内核版本、umoci 版本、GitHub SHA 和构建时间。驱动在解压并加载运行时包时会记录此元数据。

发布工作流还会为每个运行时 tarball 发布 GitHub 构建产物证明。使用以下命令验证下载的运行时：

```bash
gh attestation verify vm-runtime-linux-x86_64.tar.zst -R NVIDIA/OpenShell
```