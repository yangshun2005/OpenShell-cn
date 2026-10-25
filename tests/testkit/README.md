# testkit

端到端测试套件的共享支持。套件通过 Rust SDK 和已安装的 `openshell` CLI 测试运行中的 OpenShell 网关。

这是一个独立的 Cargo 工作区，有自己的 lockfile：

| Crate | 用途 |
|---|---|
| `testkit` | 查找并连接网关，创建和清理沙箱，运行 CLI。也构建 `testkit-ready`。 |
| `testkit-macros` | `#[testkit::test]` 属性。 |

## 编写测试

测试是一个 `async fn`，接收 `&mut testkit::Context` 并返回 `testkit::Result`：

```rust
use testkit::Context;

/// A long-running sandbox becomes ready, runs a command with exec, and is gone
/// from get and list after delete.
#[testkit::test]
async fn long_running_exec_delete(cx: &mut Context) -> testkit::Result {
    let sandbox = cx.sandbox("lr").command(["sleep", "infinity"]).create().await?;
    sandbox.wait_ready().await?;
    // ...
    Ok(())
}
```

- **文档注释是必需的。** 它是测试的描述。
- **函数名就是 nextest 测试名。**
- **其他属性会作用于测试。** 例如 `#[ignore = "reason"]` 会跳过该测试，除非用 `--run-ignored` 运行。`#[should_panic]` 会被拒绝，因为 testkit 测试在任何失败时都会 panic。要检查某项操作被拒绝，在函数体内断言失败，例如用 `cx.cli([...]).await?.failure()?` 和 `testkit::ensure!`。

每个测试结束后都会执行清理，无论通过、失败还是 panic。清理只删除通过 `cx.sandbox(...)` 创建的沙箱。测试创建的其他资源（如 provider）需在测试内自行删除。

## 测试能做什么

`Context` 目前提供：

- `cx.sandbox(suffix)` 构建并创建名称在本次运行中唯一的沙箱。返回的 `Sandbox` 可等待就绪或结束、停止、启动、exec、读取其配置和草稿策略、设置配置项以及删除。
- `cx.expect_listed` 和 `cx.expect_absent` 在 list 和 get 中检查沙箱。
- `cx.cli([...])` 对同一网关运行已安装的 CLI，输出支持 `success()`、`failure()` 和 `json()`。
- `cx.client()` 用于 testkit 未封装的 SDK 调用。
- `poll_until` 用于等待其他状态。

## 对网关运行

测试查找网关的方式与 `openshell` CLI 相同：

1. `OPENSHELL_GATEWAY_ENDPOINT`：网关 URL。其凭据来自 `OPENSHELL_GATEWAY` 指定的网关，否则来自注册了该端点的网关。未注册任何网关的 URL 按原样使用：明文，或无客户端证书的 TLS。
2. `OPENSHELL_GATEWAY`：已在 CLI 中注册的网关名称。
3. CLI 的活动网关，由 `openshell gateway select` 设置。

已注册网关的端点和认证模式来自 `$XDG_CONFIG_HOME/openshell/gateways/<name>/metadata.json`。mTLS 的 CA 和客户端证书来自其 `mtls/` 目录。testkit 支持明文和 mTLS 网关。`OPENSHELL_WORKSPACE` 选择工作区（默认 `default`），`OPENSHELL_BIN` 选择 `cx.cli` 运行的 CLI（默认 PATH 上的 `openshell`）。CLI 始终指向与 SDK 相同的网关和工作区。每个测试都会记录其使用的网关。

等待网关就绪，然后运行套件。共享同一网关的测试需逐个运行：

```shell
export OPENSHELL_GATEWAY=<name>
cargo run --manifest-path tests/testkit/Cargo.toml --bin testkit-ready -- 300
cargo nextest run --manifest-path <suite>/Cargo.toml --test-threads 1
```

`testkit-ready [TIMEOUT_SECS]` 在网关未及时健康或 CLI 未报告已连接时以非零状态退出。

`e2e/with-docker-gateway.sh` 和 `e2e/with-podman-gateway.sh` 启动临时网关并导出 `OPENSHELL_GATEWAY` 和 `OPENSHELL_BIN`，套件在其下无需修改即可运行。

## 检查

```shell
cargo fmt --manifest-path tests/testkit/Cargo.toml --all -- --check
cargo clippy --locked --manifest-path tests/testkit/Cargo.toml --all-targets -- -D warnings
cargo nextest run --locked --config-file .config/nextest.toml --manifest-path tests/testkit/Cargo.toml
```

`mise run rust:lint`、`mise run rust:format:check` 和 `mise run test` 包含这些检查，Branch Checks 也会运行它们。