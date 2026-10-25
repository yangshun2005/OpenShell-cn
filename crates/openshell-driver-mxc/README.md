# openshell-driver-mxc

基于 Windows 上 **Microsoft MXC**（`wxc-exec`）的 OpenShell 计算驱动。

## 设计

调用方驱动配置默认禁用，因此基于命令的 MXC 工作流需要管理员显式启用。主机文件系统授权没有可信的标签解析器，在启用资源准入时会被拒绝。有关独立控制项以及选择退出的安全后果，请参阅[资源准入配置](../../docs/how-it-works/gateways/configuration.mdx#external-resource-admission)。

本驱动实现网关常规的进程内 `ComputeDriver` 契约，并链接进 `openshell-gateway`。它设置了 `driver_reports_runtime_readiness`，因此网关接受驱动上报的就绪状态，无需监督者会话。创建时的规范 `SandboxPolicy` 由 `DriverSandboxSpec.policy` 承载。`process_container` 启动一次性 AppContainer，是默认后端。需显式启用的 `isolation_session` 后端使用状态感知的 `provision` → `start` → `exec` → `stop` → `deprovision` 生命周期。驱动自行启动并监控配置的工作负载，并自行上报就绪状态；不存在沙箱内监督者或 `ConnectSupervisor` 中继。

## 能力矩阵

| 能力 | MXC 驱动 | 补齐所需 |
|---|---|---|
| 文件系统策略（读写 / 只读授权） | ✅ 预配时 AppContainer 共享 | — |
| 受管控的出站流量（CONNECT 代理 + OPA + L7） | 在 `process_container` 上通过 `egress_proxy` 提供；驱动为每个沙箱启动主机 CONNECT 代理，生成 HTTPS MITM 信任材料，并将 CA 证书包注入沙箱进程环境变量 | 网关事件总线接线后续工作 |
| 网络策略 | 在 `process_container` 上拆分为 MXC `network.proxy` + 裁剪后的 OpenShell 策略；`isolation_session` 仍拒绝网络配置 | 持久会话需 MXC 反馈项 M1 |
| 网络中间件 | ❌ 启动前拒绝，因为 MXC 主机代理未接收网关中间件注册表 | 网关中间件注册表注入 |
| 进程策略（seccomp、uid/gid） | ❌ 主机侧治理设计；仅操作系统隔离 | 不推进 |
| 交互式 exec/connect/forward | ❌ exec 在驱动内运行，无客户端接入 | 网关交互式 exec 改造（后续工作） |
| 捆绑代理镜像 | ❌ 无 OCI 镜像；依赖 Windows 主机安装 | — |
| 重启持久性 | ❌ 内存注册表；重启后遗留活跃会话 | 后续工作 |
| 并发沙箱 | ⚠️ isolation_session v1 为单会话 | MXC 后端特性 |

文件系统强制执行的验证有两条路径：

- 对沙箱策略授权的路径写入成功。
- `process_container` 对沙箱策略之外的路径写入失败，报 Windows 拒绝访问，驱动上报失败的工作负载。

## 配置（`[openshell.drivers.mxc]`）

网关配置仅包含主机运行时设置：

```toml
[openshell.drivers.mxc]
wxc_exec_path = "C:\\path\\to\\wxc-exec.exe"
# Default: process_container. isolation_session is grant-only and opt-in.
backend = "process_container"
default_configuration_id = "composable"
pc_least_privilege = false
pc_capabilities = []
# Pattern-C governed egress. The address is a loopback seed; each sandbox
# receives a unique ephemeral proxy port.
egress_proxy = false
egress_proxy_addr = ""
debug = false
etw_audit = false
```

启用 `egress_proxy` 时，`egress_proxy_addr` 必须是回环 `IP:PORT` 种子。驱动保留配置的 IP，并为每个沙箱的 `network.proxy` 重定向分配唯一的临时端口。

为每个沙箱提供工作负载设置。公共配置以驱动名为键；网关仅将内层 `mxc` 对象转发给驱动：

```powershell
$config = '{"mxc":{"command":["cmd","/c","echo hello > C:\\\\work\\\\demo\\\\hello.txt"],"cwd":"C:\\\\work\\\\demo"}}'
openshell sandbox create --name mxc-demo --policy demo.yaml `
  --driver-config-json $config --env MODE=demo --no-tty
```

`command` 数组必填，并保留 Windows 参数边界。`cwd` 可选。环境变量来自标准沙箱和模板环境映射；驱动绝不从网关主机环境复制值。

启用受管控出站流量时，主机 CONNECT 代理执行网络策略。仍不支持实时策略替换或合并更新；要应用不同策略，请删除并重建沙箱。

启用 `etw_audit` 时，每个网关进程拥有一个独立的实时 ETW 会话，名称由稳定的 `OpenShell-MXC-ETW` 前缀、其进程 ID 和每次启动的判别符组成。启动另一个网关绝不会停止现有网关的捕获。优雅关闭通过其拥有的句柄停止会话。被强制终止的网关可能留下陈旧会话；审计示例仅移除编码的所属进程已不再运行且匹配的会话。

网关本地 OCSF JSONL 接收器仅适用于 Windows/MXC 路径，且需显式启用。设置 `OPENSHELL_OCSF_JSON=1` 启用，可选设置 `OPENSHELL_OCSF_LOG_DIR` 覆盖其默认的 `%PROGRAMDATA%\OpenShell\logs`。其他网关部署不初始化此本地文件接收器。

ETW 回调使用非阻塞队列，上限为 4,096 条记录和 16 MiB 复制的事件数据。超过任一限制的记录会被丢弃，而不是阻塞 ETW 泵或增长网关内存。网关会立即发出警告，指明审计覆盖缺口，并在过载持续期间将后续警告限速为每 30 秒一次。

审计归属仅在驱动拥有的 `wxc-exec` PID 及其内核进程启动键都与 ETW 记录附带的值匹配时才引导；命令文本绝不作为归属键。此代际键防止回收的 PID 继承前一个进程的归属，无论投递延迟如何。进程监控在退出时注销活跃 PID。已建立的身份、活动和关联向量链接保留五秒，以便已在途的 ETW 记录到达，但已注销 PID 的证据无法解析它们。没有匹配代际证据的记录保持未归属。

## 先决条件（实机运行）

- Windows 11 Insider 版本 ≥ 26300.8553
- `IsoSessionApp.dll` 存在并已注册
- `wxc-exec.exe` 使用 `--features isolation_session` 构建
- 任何强制执行的 App Control 策略允许 `openshell-gateway.exe` 和 `openshell.exe`。使用 `Microsoft-Windows-CodeIntegrity/Operational` 日志中的事件 3077 诊断可执行文件阻止。

针对进程内模拟垫片的离机冒烟测试（无需 `wxc-exec`，无需隔离会话），设置 `OPENSHELL_MXC_MOCK_WXC=1`。

## 策略映射

生产驱动在插入注册表项或调用 `wxc-exec` 之前，将标准驱动请求携带的类型化 `SandboxPolicy` 映射为 MXC 配置。因此映射失败会从 `CreateSandbox` 返回，不会留下部分沙箱。不存在进程内策略侧信道或 MXC 专用网关组合变体。

启用 `egress_proxy` 时，`EmbeddedPolicyMapper` 改用 `split_policy`：MXC 接收文件系统授权加回环 `network.proxy` 重定向，驱动从裁剪后的仅网络 `SandboxPolicy` 启动主机 CONNECT 代理。包含 `network_middlewares` 的策略会被同步拒绝，直到此主机代理路径能接收网关的内置和远程中间件注册表。代理使用配置的代理命令作为静态沙箱进程身份，因为 MXC 不暴露 Linux 风格的 procfs 套接字归属。对于 HTTPS L7 检查，主机代理生成每沙箱 CA，并将 `NODE_EXTRA_CA_CERTS`、`DENO_CERT`、`SSL_CERT_FILE`、`REQUESTS_CA_BUNDLE`、`CURL_CA_BUNDLE` 和 `GIT_SSL_CAINFO` 注入代理进程环境变量。它不会将生成的 CA 目录添加到 MXC 只读授权：已发布的 `wxc-exec` BaseContainer 构建要求对每个此类授权具有 `WRITE_DAC`，并拒绝用户拥有的代理临时目录。相反，驱动将沙箱唯一目录添加为内部读写共享，以便 HTTPS 客户端能读取注入的路径。该目录仅包含公共 CA 证书；临时 CA 私钥保留在主机代理内存中。驱动在应用沙箱和 TLS 覆盖之前，仅从网关主机播种 `SYSTEMROOT`、`WINDIR`、`PATH`、`COMSPEC` 和 `LOCALAPPDATA`，因此所需的 Windows 引导值保持可用，而不会暴露网关的完整环境。开发导出接口仍为 [`policy-to-mxc`](examples/policy-to-mxc.rs) 示例；尚无生产 `openshell policy export-mxc` 子命令。

如果受管控出站流量被禁用，任何网络规则都会失败关闭，而不是在没有执行路径的情况下启动。

[`tests/`](tests/) 下的对等性和矩阵测试在 Windows MSVC 通道上覆盖映射器。驱动自动执行此映射；没有单独的策略导出命令或示例。

## 为演示机打包演示

使用 [`examples/package-demo.ps1`](examples/package-demo.ps1) 将网关 EXE、CLI EXE、运行时 DLL（`libz3.dll`）、`demo.yaml`、网关配置和运行手册组装到一个文件夹，然后将该文件夹复制到演示 Windows 主机，并按照其中的 `mxc-demo-runbook.md` 操作。脚本会打印 SHA256 清单，以便操作员在移动前核对内容。

## 真实 MXC 测试通道

三个任务驱动真实 `wxc-exec.exe` 硬件；全部**跳过安全**——任何需要缺失二进制或后端的测试或场景都会打印 SKIP 原因并以 0 退出，而不是失败。

| 任务 | 运行内容 | 使用时机 |
|---|---|---|
| `windows:test:mxc-real:x64` | `tests/wxc_exec_real.rs` — 使用 `--ignored --test-threads=1` 的 Tier-2 调用方测试，包括通过主机代理的 HTTPS 请求 | 在任何有 `wxc-exec` 的 Windows 主机上预合并；试运行测试始终通过；强制执行测试自行探测门控 |
| `windows:e2e:mxc` | `examples/run-mxc-e2e.ps1` — Tier-3 场景运行器，真实二进制，探测门控 | 演示机 / 夜间；需要脚本目录中有网关 + CLI 二进制 |
| `windows:e2e:mxc:mock` | 同一运行器加 `-Mock` — 仅接线，无需真实 `wxc-exec` | 任何 Windows 主机（CI、开发机）；验证接线和网络拒绝场景 |

**探测脚本：** `examples/probe-mxc-host.ps1` 是操作员/CI 预检，输出 JSON 能力报告（操作系统版本、wxc-exec 路径/版本、试运行退出码、每后端试验结果和 `verdicts` 对象）。在真实 MXC 通道之前运行它，以了解给定主机上哪些会 PASS 与 SKIP：

探测每次运行使用唯一的、用户拥有的 Windows 临时目录。MXC 按字面处理配置路径（不展开 `%TEMP%`），每次运行的目录使 AppContainer+DACL 回退变更范围保持狭窄。

```powershell
powershell -NoProfile -ExecutionPolicy Bypass `
  -File crates/openshell-driver-mxc/examples/probe-mxc-host.ps1
```

**跳过语义：** `wxc_exec_real.rs` 中的测试标记为 `#[ignore = "requires real wxc-exec"]`——标准 `windows:test:x64` 套件从不运行它们。`OPENSHELL_WXC_EXEC_PATH` 覆盖默认的 `C:\mxc\wxc-exec.exe` 查找。有关开发机的经验能力快照（版本 26200，processcontainer 速度键未启用，isolation_session 缺失），请参阅 `docs4gtb/mxc-box-capabilities.md`。

## 推迟的工作

- **交互式 exec/connect/forward** — 网关交互式 exec 改造（后续工作）
- **重启持久性**（启动时取消预配孤立会话）→ 后续工作
- **GPU 直通** → 在主机侧治理设计中不推进