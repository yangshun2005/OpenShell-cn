# OpenShell 策略顾问

当 OpenShell 拦截网络请求，且响应或日志显示 `policy_denied` 时，使用本指南。

## 目标

起草最小的策略提案，允许用户当前任务，同时不给沙箱授予广泛的新网络访问权限。由开发者批准或拒绝提案；不要试图绕过策略。

## 本地 API

沙箱本地策略 API 可通过 `http://policy.local` 访问：

- `GET /v1/policy/current` — 当前生效策略（YAML 格式）。
- `GET /v1/denials?last=10` — 本沙箱最近看到的网络/L7 拒绝记录（最新在前），以原始简写日志行返回。每行包含时间戳、类别、严重级别、动作、主机/端口、二进制、策略名称，以及（对于拒绝事件）简短原因。直接读取这些行即可，无需解析为结构化字段。
- `POST /v1/proposals` — 提交提案。202 响应携带 `accepted_chunk_ids`（网关接受的每个操作对应一个 ID）和 `rejection_reasons`（网关在提交时拒绝的每个操作对应一条记录）。两个数组合起来覆盖你发送的每个操作。
- `GET /v1/proposals/{chunk_id}` — 单个提案的即时状态。返回 `status`（`pending` / `approved` / `rejected`）、`rejection_reason`（审核者的自由文本备注，仅在拒绝时设置）和 `validation_result`（网关证明器对该分块的判定；可能为空）。
- `GET /v1/proposals/{chunk_id}/wait?timeout=<seconds>` — 阻塞等待该提案，直到开发者做出决定或超时。默认 60 秒，限制在 [1, 300]。超时时返回 `status: "pending"` 及 `timed_out: true`。批准时响应还携带 `policy_reloaded: true|false`，表示本地沙箱是否已加载包含已批准规则的策略。使用此端点代替轮询 `/v1/proposals/{chunk_id}`。

提案体包含 `intent_summary` 和一个或多个 `addRule` 操作。每个 `addRule` 携带一条完整的窄范围 `NetworkPolicyRule`。

## 工作流程

1. 读取拒绝响应体。使用 `layer`、`method`、`path`、`host`、`port`、`binary`、`rule_missing` 和 `detail` 作为证据。
2. 从 `/v1/policy/current` 获取当前策略。
3. 如果响应体不完整，从 `/v1/denials` 获取最近的拒绝记录。
4. REST API 优先使用 L7 REST 规则。**针对作用域内无凭据的主机的提案会自动批准**（见下文自动批准）。任何涉及凭据的访问或能力变更都会进入人工审核——这是设计使然。L7 仍是代理速度路径，因为证明器能精确描述变更（在哪个路径上添加了哪个方法）。当客户端可以使用显式代理但其线协议对 L7 检查不透明（`ssh`、`nc`、`git-remote-http`）时，省略 `protocol` 并保留默认 TLS 处理。**绝不要**提议 `protocol: tcp` 或 `tls: skip`：这些选择会绕过应用权限检查，需要管理员显式添加规则。
5. 起草最窄的规则：精确主机、精确端口、已知时精确二进制、精确方法，以及最小安全路径。
6. 提交提案，保存响应中的 `accepted_chunk_ids`，并告知开发者你提议了什么。如果响应还携带 `rejection_reasons`，说明网关在任何人工审核之前已在提交时拒绝了这些操作；修复后重新提交，再等待其余部分。
7. 对每个已接受的 chunk_id，调用 `GET /v1/proposals/{chunk_id}/wait?timeout=300` 并根据结果行动：
   - `status: "approved"` 且 `policy_reloaded: true` — 重试原先被拒绝的操作。合并后的策略已加载；请求应成功。如果仍以 `policy_denied` 失败，重新读取拒绝记录——你的规则可能不匹配。如果因其他原因失败，上报给用户。
   - `status: "approved"` 且 `policy_reloaded: false` — 批准已落地，但本地沙箱在 `/wait` 窗口内未观察到重新加载。以 `timeout=30` 再调用一次相同的 `/wait`。如果第二次响应仍为 `policy_reloaded: false`，上报给用户而非盲目重试；不要紧密循环。
   - `status: "rejected"` — 读取 `rejection_reason` 和 `validation_result`。`rejection_reason` 是审核者输入的内容；`validation_result` 是证明器的判定，通常能解释由自动检查驱动的拒绝。如果任一有内容，针对具体反馈处理并提交修订提案。如果两者都为空，起草实质不同的方案或询问用户。
   - `status: "pending"` 且 `timed_out: true` — 再次调用 `/wait`。
   - 任何非 2xx 响应 — 上报给用户；未经批准不要重试被拒绝的操作。

## 提案结构

一条完整的窄范围 REST 检查规则如下所示：

```json
{
  "intent_summary": "Allow gh to update repository contents in NVIDIA/OpenShell only.",
  "operations": [
    {
      "addRule": {
        "ruleName": "github_api_repo_contents_write",
        "rule": {
          "name": "github_api_repo_contents_write",
          "endpoints": [
            {
              "host": "api.github.com",
              "port": 443,
              "protocol": "rest",
              "enforcement": "enforce",
              "rules": [
                {
                  "allow": {
                    "method": "PUT",
                    "path": "/repos/NVIDIA/OpenShell/contents/**"
                  }
                }
              ]
            }
          ],
          "binaries": [
            {
              "path": "/usr/bin/gh"
            }
          ]
        }
      }
    }
  ]
}
```

## 自动批准

自动批准通过 `proposal_approval_mode` 设置选择启用，通过标准设置模型管理。审核者在网关作用域（全集群）设置：`openshell settings set --global proposal_approval_mode auto`，或在沙箱作用域设置：`openshell settings set <name> proposal_approval_mode auto`。CLI 的 `openshell sandbox create --approval-mode auto` 是简写，在创建时写入沙箱作用域设置。两者都设置时网关作用域优先；默认（无设置）为 `"manual"`。

启用自动批准且证明器未发现新内容时，网关以 actor `system:auto` 批准该分块，`CONFIG:APPROVED` 审计事件携带 `auto=true`、`source=<mode>`、`prover_delta=empty` 和 `resolved_from=<gateway|sandbox>`。代理的 `/wait` 约 1 秒内返回已批准。当证明器确实发现内容——或设置为 `"manual"`/未设置——该分块进入 `pending` 等待人工审核。

证明器对每个提议的变更回答四个形式化问题。每个"是"回答都是独立的分类发现——没有严重级别分级。任何发现都会阻止自动批准。

- **`link_local_reach`** — 提案授予对链路本地 IP 范围（`169.254.0.0/16`、`fe80::/10`）或已知元数据主机名（如 `metadata.google.internal`）的访问。`169.254.169.254` 等云元数据端点属于此类。**绝不要**提议访问这些——无论沙箱自身持有什么，这些端点都会提供凭据。
- **`l7_bypass_credentialed`** — 提案让使用 L7 代理无法检查的线协议的二进制（`/usr/bin/git`、`/usr/lib/git-core/git-remote-http`、`/usr/bin/ssh`、`/usr/bin/nc`）访问作用域内存在沙箱凭据的主机。对 L7 不透明的线协议不受 L7 作用域限制；审核者必须决定是否信任该二进制持有凭据。
- **`credential_reach_expansion`** — 提案授予某二进制对之前无法访问的 (host, port) 的凭据访问。新的认证访问是明确的意图变更——审核者确认该二进制是否应能向该主机认证。
- **`capability_expansion`** — 提案在已有凭据访问的 (binary, host, port) 上添加新的 HTTP 方法。审核者能看到具体添加了哪个方法，并判断它是否属于代理任务的一部分。变更类方法（PUT、POST、PATCH、DELETE）是此发现的典型来源。

自动批准的内容（在 `auto` 模式下）：

- 证明器未发现四类中任何一类的提案——例如，针对作用域内无凭据主机的 L7 规则（从 CDN 获取公开内容、Schema URL、公共 API 发现）。

如果你的提案升级且希望自动批准，先看该主机是否真的需要凭据二进制。公开内容的 GET 通常不需要，切换到其他主机（或移除凭据依赖）即可消除该发现。凭据变更*本就应当*升级——提议窄规则并等待审核。

## 完善早前的自动建议规则

当沙箱观察到无法限定到 L7 的拒绝——例如，某二进制尝试连接代理在应用层尚未见过的主机——它会自动起草一个宽泛的 L4 提案，以便操作者有具体内容可看。这些机械性草稿与其他待处理提案一样对你可见。

如果你看到待处理的机械性 L4 草稿且能做得更好，只需为相同的 `(host, port, binary)` 提交一个完善的 L7 提案。网关会自动以"superseded by chunk X"为由拒绝机械性草稿——无需额外清理或 `supersedes_chunk_id`。新提交通过结构重叠胜出。

## 规范

- 不要提议通配符主机，如 `**` 或 `*.com`。
- 不要为修复单个被拒绝的 REST 请求而提议 `access: full`。
- 不要提议访问链路本地地址（`169.254.0.0/16`、`fe80::/10`）或已知元数据主机名（如 `metadata.google.internal`）。那里的云元数据端点可能交出主机凭据。
- 不要提议 `protocol: tcp` 或 `tls: skip`。如果任务需要原生 TCP 或原始 TLS 隧道，请管理员显式添加该访问。
- 不要在路径中包含查询字符串、令牌、凭据或机密值。
- 在 `intent_summary` 中说明不确定性，而非放宽规则。
- 如果 `git` 推送失败，那是与 GitHub REST API 访问不同的 L4 或协议特定路径。单独提议。

## 本地日志（只读）

两个本地文件补充 API，在调试策略行为时很有用：

- `/var/log/openshell.YYYY-MM-DD.log` — 沙箱活动的简写日志。`/v1/denials` 从中读取。
- `/var/log/openshell-ocsf.YYYY-MM-DD.jsonl` — 完整 OCSF JSON 事件，仅在 `ocsf_json_enabled` 设置开启时写入。`/v1/denials` 不使用；适合 SIEM 摄取。