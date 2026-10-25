# 测试

## 运行测试

```bash
mise run test          # Rust + Python 单元测试
mise run e2e           # 端到端测试（启动由 Docker 支撑的网关）
mise run ci            # 全部：lint、编译检查与测试
```

## 测试布局

```text
crates/*/src/          # 内联 #[cfg(test)] 模块
crates/*/tests/        # Rust 集成测试
python/openshell/      # Python 单元测试（*_test.py 后缀）
e2e/python/            # Python E2E 测试（test_*.py 前缀）
e2e/rust/              # Rust CLI E2E 测试
tests/testkit/         # SDK 驱动 E2E 套件的支持库
```

## Rust 测试

单元测试以内联 `#[cfg(test)] mod tests` 块的形式与源码放在一起。集成测试放在 `crates/*/tests/`，命名为 `*_integration.rs`。

异步代码使用 `#[tokio::test]`：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn store_round_trip() {
        let store = Store::connect("sqlite::memory:").await.unwrap();
        store.put("sandbox", "abc", "my-sandbox", b"payload").await.unwrap();
        let record = store.get("sandbox", "abc").await.unwrap().unwrap();
        assert_eq!(record.payload, b"payload");
    }
}
```

仅运行 Rust 测试：

```bash
mise run test:rust     # cargo test --workspace
```

Rust 校验会检查受跟踪的 Cargo lockfile；运行 `mise run rust:lockfiles:check` 可直接检查。若某个 lockfile 已过期，用其相邻的 manifest 通过 Cargo 刷新，审阅 diff 后提交更新。

### 基于 PostgreSQL 的测试

需要真实 PostgreSQL 服务器的测试（例如跨两个 store 的 advisory-lock 并发）标记为 `#[ignore]`，命名为 `postgres_*`。运行方式：

```shell
mise run test:rust:postgres
```

该任务用 Docker 或 Podman 启动一次性 PostgreSQL 容器（通过设置 `CONTAINER_ENGINE` 选择），逐个运行测试，然后删除容器。除 replay 和 provider refresh 锁测试使用默认 schema 外，每个测试都在自己的临时 schema 中运行。要使用自己的临时数据库，设置 `OPENSHELL_TEST_POSTGRES_URL`。该任务会覆盖 `OPENSHELL_REPLAY_TEST_DATABASE_URL`，使旧测试使用同一数据库。切勿指向运行中网关使用的数据库：测试会获取集群范围的 advisory 锁。
Branch Checks 在 x86_64 Linux 上运行这些测试。
对负载敏感的容量检查命名为 `bench_postgres_*`，因此该任务会跳过它们。在空闲机器上用 `mise run test:rust:postgres:bench` 运行。

### 原生 Windows 校验

使用现有 Rust/MSVC 工具链运行 `mise run --skip-tools pre-commit`。Windows 现在通过 PowerShell 检查受跟踪的 Cargo lockfile，不再跳过。

`mise run --skip-tools sdk:ts:ci` 在 Windows 上选择 x64 Biome 可执行文件（包括在模拟下运行它的 ARM64 主机），通过其 Windows `.cmd` 启动器解析 protobuf 插件，并在 Node 本身为 ARM64 时安装匹配的锁定 ARM64 Rolldown 绑定。该辅助脚本保持 lockfile 解析不变；不得升级无关的测试依赖。

`mise run --skip-tools go:ci` 保留竞态检测，但 Windows ARM64 除外（Go 不支持）。Windows token 文件测试显式跳过 POSIX 模式位断言；这些跳过并不建立 Windows ACL 保护。运行 Unix shell fixture 检查时，使用 LF 文本文件的检出。

## Python 单元测试

Python 单元测试使用 `*_test.py` 后缀约定（而非 `test_*` 前缀），与源码一起放在 `python/openshell/` 中。它们使用基于 mock 的模式和伪造的 gRPC stub：

```python
def test_exec_python_serializes_callable_payload() -> None:
    stub = _FakeStub()
    client = _client_with_fake_stub(stub)

    def add(a: int, b: int) -> int:
        return a + b

    result = client.exec_python("sandbox-1", add, args=(2, 3))
    assert result.exit_code == 0
```

仅运行 Python 单元测试：

```bash
mise run test:python   # uv run pytest python/
```

## E2E 测试

E2E 测试针对运行中的网关运行。默认情况下，`mise run e2e` 使用 Docker 计算驱动启动一个临时独立网关，运行套件，然后清理。要针对现有明文网关运行套件，设置 `OPENSHELL_GATEWAY_ENDPOINT`：

```bash
OPENSHELL_GATEWAY_ENDPOINT=http://127.0.0.1:18080 mise run e2e
```

原始端点模式仅支持 HTTP。网关需要 mTLS 时，使用具名网关配置。

`mise run test:gateway-config` 无需运行中的运行时即可校验生成的网关 TOML。它覆盖当前 schema 以及 Podman 树内与外部驱动配置的边界。

### testkit（`tests/testkit/`）

`testkit` 是 SDK 驱动端到端套件的支持库。测试是带必需文档注释的 `#[testkit::test]` 函数。它们像 CLI 一样查找网关：通过 `OPENSHELL_GATEWAY_ENDPOINT`、`OPENSHELL_GATEWAY` 或活动网关；`cx.cli` 运行由 `OPENSHELL_BIN` 指定的 CLI。编写和运行测试的方法见 [tests/testkit/README.md](tests/testkit/README.md)。

### Python E2E（`e2e/python/`）

`mise run e2e:python` 从 `e2e/python/Dockerfile.workload` 构建 `openshell/e2e-python:dev`，并仅将其用于测试网关。这个基于 Noble 的 fixture 提供 `sandbox` 用户、Python 工具、Git 和可写的 `/sandbox/.venv`。其 Python 版本来自 `.python-version`，以便 cloudpickle 代码对象与测试运行器匹配。生产工作负载默认仍为未修改的 NVIDIA Ubuntu 镜像。

Rust Docker harness 也会为需要工具或具名用户的测试选择此 fixture。显式 `--from` 或 `--template` 参数以及默认镜像测试保留各自的镜像选择。Docker sandbox 和支持容器 fixture 使用固定镜像名；无需覆盖工作负载镜像。Podman、VM 和 Kubernetes 保留其现有的固定、可拉取 fixture 和镜像设置。仅一致性运行不需要 Docker fixture。用 `mise run e2e:workload:build` 单独构建 Docker fixture。

测试使用 `conftest.py` 中的 `sandbox` fixture 创建真实 sandbox：

```python
def test_exec_returns_stdout(sandbox):
    with sandbox(delete_on_exit=True) as sb:
        result = sb.exec(["echo", "hello"])
        assert result.exit_code == 0
        assert "hello" in result.stdout
```

#### `Sandbox.exec_python`

`exec_python` 用 `cloudpickle` 序列化 Python 可调用对象，发送到 sandbox 并返回结果。由于 cloudpickle 按引用序列化模块级函数（在 sandbox 内会失败），请使用以下模式之一：

**工厂函数返回的闭包：**

```python
def _make_adder():
    def add(a, b):
        return a + b
    return add

def test_addition(sandbox):
    with sandbox(delete_on_exit=True) as sb:
        result = sb.exec_python(_make_adder(), args=(2, 3))
        assert result.stdout.strip() == "5"
```

**本地类的绑定方法：**

```python
def test_multiply(sandbox):
    class Calculator:
        def multiply(self, a, b):
            return a * b

    with sandbox(delete_on_exit=True) as sb:
        result = sb.exec_python(Calculator().multiply, args=(6, 7))
        assert result.stdout.strip() == "42"
```

#### 共享 Fixture（`e2e/python/conftest.py`）

| Fixture | 作用域 | 用途 |
|---|---|---|
| `sandbox_client` | session | 连接到活动网关的 gRPC 客户端 |
| `sandbox` | function | 返回 `Sandbox` 上下文管理器的工厂 |

### Rust CLI E2E（`e2e/rust/`）

基于 Rust 的 e2e 测试，将 `openshell` CLI 二进制作为子进程执行。它们位于 `openshell-e2e` crate 中，使用共享 harness 管理 sandbox 生命周期、解析输出和清理。

暴露的服务 URL 使用虚拟主机名进行网关路由。主机侧测试必须将 TCP socket 直接连接到可达的网关监听地址（通常是回环），并在 HTTP `Host` 头中发送服务 URL 的 authority。不要解析 `*.openshell.localhost`；本地和 CI 环境对任意 `.localhost` 子域名的解析器支持各不相同。

将公布的服务 URL scheme 视为权威。对于 HTTPS，使用虚拟服务主机名进行 TLS SNI，并使用配置的网关信任根。监听器要求 mTLS 时，出示活动网关客户端身份；本地 e2e 包装器将这些材料注册在 `$XDG_CONFIG_HOME/openshell/gateways/$OPENSHELL_GATEWAY/mtls/` 下。拨号回环时不要将 HTTPS 服务 URL 降级为明文。在进入就绪循环前解析 URL 并加载 TLS 材料，使永久性配置错误立即失败。仅重试瞬时连接失败和文档化的就绪响应，并在超时诊断中包含最后一次观测结果。

在默认本地模式和 CI 等效的 HTTPS 模式下验证暴露服务测试：

```shell
mise run e2e:rust
OPENSHELL_ENABLE_LOOPBACK_SERVICE_HTTP=false mise run e2e:rust
```

当多个测试需要此行为时，将传输逻辑放入共享 Rust e2e harness，并要求调用方使用它，而不是重复实现 DNS、HTTP `Host`、TLS SNI 和 mTLS 处理。

套件：

- 通用套件（`--features e2e`）——驱动无关的 CLI 行为、sandbox 生命周期、同步、端口转发、策略和 provider 测试。
- CLI 一致性（`tests/suites/conformance`）——可移植的 Cargo 测试，覆盖生命周期、机制化草稿、文件传输和 sandbox 本地 API，包括 agent 编写的权限请求。驱动 E2E 运行完整的 Cargo 测试包。已安装产物一致性套件从 nextest 归档运行相同测试，并提供聚焦的 `policy-advisor` 测试套件用于手动集成运行。
- 驱动套件（`--features e2e-docker`、`e2e-podman`、`e2e-kubernetes` 或 `e2e-vm`）——CLI 一致性加上所选部署的通用和驱动特定覆盖。
- Docker 套件（`--features e2e-docker`）——包含仅 Docker 的覆盖，如 Dockerfile 镜像构建、Docker 预检和管理 Docker 网关启动。
- Docker GPU 套件（`--features e2e-docker-gpu`）——Docker 套件加上 GPU sandbox 冒烟覆盖。
- 配置推送套件（`--features e2e-config-push`）——CLI 一致性和针对 `config_delivery_mode = "push"` 的 Docker 网关的策略测试。用 `mise run e2e:rust:push` 运行。
- VM 套件（`--features e2e-vm`）——在 VM 上运行 e2e 测试。
- Kubernetes 凭据驱动套件（`--features e2e-kubernetes-credential-drivers`）——针对性的 Kubernetes Secrets 和 Vault provider 凭据存储覆盖。

Docker、Podman、Kubernetes 和 VM 运行器在运行 Rust 测试前先执行 CLI 一致性。将驱动的 feature 变量（`OPENSHELL_E2E_DOCKER_FEATURES`、`OPENSHELL_E2E_PODMAN_FEATURES`、`OPENSHELL_E2E_KUBERNETES_FEATURES` 或 `OPENSHELL_E2E_VM_FEATURES`）设为 `e2e-conformance-only` 可跳过 Rust 测试；运行器跳过时会记录提示。未设置或空值选择驱动的默认 Rust feature。该选择器由运行器处理，不是 Cargo feature。

```shell
OPENSHELL_E2E_KUBERNETES_FEATURES=e2e-conformance-only mise run e2e:kubernetes
```

GPU 设备选择测试将 OpenShell sandbox 与请求 `--device nvidia.com/gpu=all` 的普通 Docker 或 Podman 容器对比。探测镜像默认为 `deploy/docker/Dockerfile.images` 中 `gateway` 阶段使用的镜像；设置 `OPENSHELL_E2E_GPU_PROBE_IMAGE` 可覆盖。逐设备检查仅对运行时发现的设备列表报告的 NVIDIA CDI 设备 ID 运行，因此仅暴露 `nvidia.com/gpu=all` 的 WSL2 主机会跳过基于索引的用例。精确 CDI 设备选择通过 `--driver-config-json` 和活动的 Docker 或 Podman 驱动键传入。

运行 Docker 支撑的 Rust CLI e2e 套件：

```shell
mise run e2e:docker
```

针对 OpenShell CLI 配置中选择的网关运行可移植 CLI 一致性套件：

```shell
cargo build --package openshell-cli
OPENSHELL_BIN="$PWD/target/debug/openshell" \
  cargo test \
    --locked \
    --manifest-path tests/suites/conformance/Cargo.toml \
    --package openshell-test-conformance-cli \
    --no-fail-fast \
    -- \
    --test-threads=1 \
    --nocapture
```

测试开始前，网关必须已安装、可达并被选中。测试套件不会预置网关或选择计算驱动。将 `OPENSHELL_BIN` 设为其他可执行文件可测试不同的预构建 CLI。

阶段 1 场景验证完整的 CLI→网关→驱动路径，不依赖网关的安装方式或配置的驱动。它要求机器可读的 gRPC 状态，用配置的默认镜像创建唯一命名的分离 sandbox，通过在分页 JSON 列表输出中找到其唯一名称验证 sandbox 为 `Ready`，用运行特定标记执行 `echo`，删除 sandbox，并验证其名称不再出现。驱动套件启用相同 profile，而不是维护单独的冒烟实现。Sandbox 生命周期、标签矩阵、VM overlay 和 TLS 密钥权限断言仍为常规 E2E 覆盖。

每次调用在创建资源前打印十字符的运行 ID。一致性 sandbox 使用 `ct-<run-id>-01` 之类的名称。运行器跟踪确切名称并用于清理；阶段 1 不添加所有权标签。

运行器在成功和失败后都删除自有资源。若测试进程在清理前被中断，在不触碰无关网关状态的情况下定位残留：

```shell
openshell sandbox list --output json
openshell sandbox delete <sandbox-name>
```

网关支撑的 Rust E2E 任务针对配置的网关运行 CLI 一致性 Cargo 测试，然后运行仍适用的 lane 特定 Rust 测试。运行 Podman 支撑的 Rust CLI e2e 套件：

```shell
mise run e2e:podman
```

`nix run .#build-artifacts` 还会为 tmachine 的 `deb` 安装器构建 `artifacts/packages/openshell.deb`，目标为主机的 Linux 架构（`arm64` 或 `amd64`）。设置 `OPENSHELL_DEB_VERSION` 可覆盖默认包版本 `0.0.0-local`。

它还会为 tmachine 的 `rpm` 安装器构建 `artifacts/packages/rpm/openshell.rpm` 和 `artifacts/packages/rpm/openshell-gateway.rpm`。RPM 步骤在 CI 使用的 Fedora 容器中运行 `packit build locally` 和 `openshell.spec`，因此需要能为主机架构（`aarch64` 或 `x86_64`）运行 Linux 容器的 Docker 守护进程，例如 macOS 上的 Docker Desktop。包版本默认为分支 CI 计算出的开发版本；设置 `OPENSHELL_RPM_VERSION` 和 `OPENSHELL_RPM_RELEASE` 可覆盖。在 `nix run .#build-artifacts-binaries` 之后，`nix run .#build-artifacts-rpm` 仅重建 RPM。针对已安装的 RPM 运行一致性：

```shell
nix run .#build-artifacts
nix run .#tmachine -- test fedora-podman-rootful rpm conformance
```

在一次性无根 Podman guest 中运行可移植子集：

```shell
nix run .#build-artifacts
nix run .#tmachine -- test fedora-podman-rootless binaries e2e-podman
nix run .#tmachine -- test fedora-podman-rootless binaries driver-podman
```

以 shell `PODMAN_CI_TESTS` 数组形式打印确切的 tmachine 归档选择：

```shell
nix run .#generate-podman-e2e-ci-tests
```

`e2e-podman` 测试套件运行用相应 Rust feature 构建的 nextest 归档，并将其 Python 工作负载镜像预加载到无根 Podman 存储中。单独的 `driver-podman` 测试套件比较 OpenShell 与直接 Podman 在默认、`auto`、`keep-id` 和私有 profile 下的用户命名空间映射。E2E 归档排除仍依赖包装器拥有的网关控制、SPIFFE fixture