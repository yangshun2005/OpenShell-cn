<!-- SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved. -->
<!-- SPDX-License-Identifier: Apache-2.0 -->

# openshell-prover

OpenShell 沙箱策略的形式化验证器。将策略、其附带的凭据集以及二进制能力注册表编码为 Z3 SMT 模型，然后运行可达性查询，检测审查者应关注的凭据可达性和能力变化。

该 crate 还暴露一个独立的边界包含 API，用于检查完全组合后的候选策略是否处于运维方提供的边界之内。`openshell-prover-cli` 包为该 API 提供本地文件封装。边界包含与旧版提案风险查询回答的是不同问题；在托管策略迁移完成前，网关调用方继续使用提案风险 API。

两个验证器入口均使用有界的 `openshell-policy-schema` 解析器。边界包含会将未知字段以及托管的 `metadata`/`review` 注解视为无效输入予以拒绝，然后针对其支持的模型检查符合 Schema 的控件。它使用共享的文件系统默认值、端口选择和访问预设语义。

边界包含模型接受网络二进制选择器、端点主机和路径选择器、以及 REST 允许和拒绝的方法和路径选择器中的 ASCII 字面量。当任一策略在这些字段中使用非 ASCII 字面量时，返回 `unsupported_policy_shape`。此边界不适用于文件系统路径或无关的策略文本。网络选择器字段中的嵌入 NUL 字节同样不受支持。ASCII 通配符按运行时匹配语言建模，因此可以匹配非 ASCII 运行时值。无法精确解码和验证的求解器字符串会产生 `invalid_witness`，而非反例证据。

## 边界包含 API 兼容性

边界包含模块是可复用的 Rust API，但此契约并不使整个 `openshell-prover` crate 成为稳定的已发布 SDK。通过 `CheckOptions::new` 构造选项，并且仅在结果为 `CheckResult::Within` 时接受策略：

```rust
use std::time::Duration;
use openshell_prover::containment::{
    CheckOptions, CheckResult, check_within_boundary, parse_policy_str,
};

let boundary = parse_policy_str("version: 1\n")?;
let candidate = parse_policy_str("version: 1\n")?;
let result = check_within_boundary(
    &boundary,
    &candidate,
    CheckOptions::new(Duration::from_secs(10)),
);
let authorized = matches!(result, CheckResult::Within(_));
# Ok::<(), Box<dyn std::error::Error>>(())
```

`ReasonCode`、`CheckDomain`、`Protocol` 和 `Counterexample` 允许新增变体。`CheckOptions`、`CheckCoverage`、`WithinEvidence` 以及现有反例变体允许新增字段。使用 `..` 和通配分支匹配这些类型，使用其访问器和 `as_str()` 标识符，并将未知值视为失败关闭结果。现有标识符字符串保持稳定。

`CheckResult` 有意封闭为四种状态：`Within`、`Exceeds`、`Unsupported` 和 `Inconclusive`。`FilesystemAccess` 同样封闭为 `Read` 和 `Write`。新增结果状态或文件系统访问模式属于破坏性 API 变更。Rust 源码兼容性与 CLI JSON Schema 契约相互独立。CLI 的数字 `schema_version` 为该 JSON 契约的版本，而 `prover_version` 标识产生结果的实现。结果的 `coverage.domains` 列表是已建模权限的机器可读声明。

网关用它来门控智能体编写的策略提案的自动批准：任何发现都会阻止自动批准，空的增量则允许该块通过（前提是审查者通过网关或沙箱作用域的 `proposal_approval_mode` 设置选择启用）。

## 它判定什么

验证器回答四个形式化问题。每个"是"答案都是独立的分类发现——没有严重性分级。分类定义在 [`finding::category`](src/finding.rs) 中。

| 分类 | 验证器判定的问题 |
|---|---|
| `link_local_reach` | 该策略是否授予对 `169.254.0.0/16` 或 `fe80::/10` 中主机的可达性？ |
| `l7_bypass_credentialed` | 是否允许使用非 HTTP 线协议（依据二进制注册表的 `bypasses_l7` 标志）的二进制访问凭据作用域内的主机？ |
| `credential_reach_expansion` | 是否允许某二进制在凭据作用域内访问某个 (host, port)，而该二进制此前无法访问该端点？ |
| `capability_expansion` | 在某二进制已凭据可达的 (binary, host, port) 上，是否新增了 HTTP 方法？ |

前两个是无条件风险。后两个是*增量*属性——网关对基线策略和合并后策略分别运行验证器，仅呈现新增路径。

## 证据结构

每个发现携带一个或多个 [`FindingPath::Exfil`](src/finding.rs) 条目：

```rust
pub struct ExfilPath {
    pub binary: String,
    pub endpoint_host: String,
    pub endpoint_port: u16,
    pub mechanism: String,        // 人类可读描述
    pub policy_name: String,      // 路径经过的规则
    pub category: String,         // 分类常量之一
    pub method: String,           // capability_expansion 时填充；否则为空
}
```

网关的 `finding_delta` 以 `(finding query, binary, host:port, path category, method)` 为键索引路径，因此在已可达主机上新增方法只会呈现为恰好一条新路径（而非重新输出整个现有方法集）。

### 增量层的分类抑制

若 `capability_expansion` 路径的 `(binary, host, port)` 元组也出现在 `credential_reach_expansion` 增量中，网关会抑制这些路径。全新的凭据可达仅由可达性扩展发现描述，不再额外产生 N 条按方法的发现。

## 新增分类

1. 在 `src/finding.rs::category` 中添加常量。
2. 在 `src/queries.rs::check_credential_safety` 中添加检测新分类的分支，并为每条证据行发出一个 `ExfilPath`。将 `path.category` 设为新常量。
3. 在 `src/report.rs::format_path_line` 中添加 `match` 分支，渲染审查者看到的每路径显示字符串。
4. （网关）若新分类应被另一分类抑制，将抑制规则添加到 `crates/openshell-server/src/grpc/policy.rs::finding_delta`。
5. 在 `src/queries.rs` 中添加单元测试，在 `crates/openshell-server/src/grpc/policy.rs::tests` 中添加集成测试。

四个 v1 分类覆盖了 OpenShell 自动批准门控当前关注的形式化属性。其他分类（例如"引入破坏性方法"、"新增无 SNI 的出站 TLS"）将是增量式的——它们不会取代现有分类。

## 验证器*不*判定什么

- **操作的语义风险。** 验证器建模的是*二进制能否执行此操作？*，而非*这是否具有破坏性？*。`PUT /repos/.../contents/file.md` 和 `GET /repos/.../contents/file.md` 都是已认证操作；由审查者（或下游层，如 LLM 上下文审查器或意图文件）决定该操作是否被期望。
- **跨沙箱或跨二进制意图。** 模型是按沙箱的。若两个沙箱通过外部策略共享凭据，验证器分别对各自进行推理。
- **运行时行为。** 验证器分析书写形式的策略；它不观察代理的实际决策。代理是执行层；验证器是变更审查层。

## 输入

- **策略** — 由 `openshell-policy-schema` 使用共享的失败关闭解析器解码的编写 YAML，然后投影到验证器专用的 `PolicyModel`。缺失的 `version` 和未知的编写字段会像运行时解析一样被拒绝。缺失 `filesystem_policy` 时使用运行时生效的 `include_workdir: true`；显式存在的空对象保持 `false`。显式 `protocol: tcp` 视为 L4，与运行时分类一致。
- **凭据集** — 由 `crates/openshell-server/src/grpc/policy.rs::build_credential_set_for_sandbox` 中沙箱的附带提供方构建。v1 仅捕获存在性（主机粒度）；不建模作用域。
- **二进制注册表** — `crates/openshell-prover/registry/binaries/*.yaml` 中的 YAML 描述符。每个描述二进制的协议、`bypasses_l7` 标志和 `can_exfiltrate` 能力。

已接受风险文件、凭据描述符和二进制注册表是独立的文件格式。它们继续使用各自的 serde 定义，不构成编写策略 Schema 的一部分。

## 输出

- `Finding` 值列表，每个触发的分类一个。每个发现的 `query` 字段保存分类名称。
- 旧版报告渲染器（`report::render_compact` / `render_report`）为终端消费者格式化提案风险发现。
- 网关调用 `report::finding_shorthand` 构建持久化在每个草稿块上的 `validation_result` 字符串。
- 边界包含 API 向 `openshell-prover-cli` 返回类型化证据，后者负责独立命令的文本和 JSON 格式及退出码。

## Z3 模型布局

参见 `src/model.rs`。简要说明：

- 端点事实按规则、端点位置和生效端口索引。兄弟端点即使目标重叠也各自保留其授权。
- 查询入口在基础约束不一致或求解器无法得出结论时返回错误。调用方必须传播该错误；它不能被当作发现接受，也不能与空发现列表比较。

- 按 `(binary, endpoint)` 对的 Bool 排序编码策略可达性，由二进制能力标志（`can_exfiltrate`、`bypasses_l7`）过滤。
- 按 `(binary, host)` 的 Bool 排序编码凭据作用域内（每个沙箱一个凭据集）。
- 可达性公式将这些组合成 `queries::check_credential_safety` 循环迭代的 SAT 查询。

## 测试

- 各模块中的单元测试（`src/queries.rs`、`src/report.rs`、`src/policy.rs`）覆盖单个原语和分类发出。
- `src/lib.rs::tests` 中的集成测试针对 `testdata/` 中的测试数据策略，演练完整的解析 → build_model → run_all_queries 流水线。
- `crates/openshell-server/src/grpc/policy.rs::tests` 中的网关级验收测试锁定端到端 `validation_result` 结构和自动批准门控。