---
name: openshell-policy-advisor
description: 当 OpenShell 沙箱返回 policy_denied、提及 policy.local，或需要提出范围狭窄的网络策略提案时使用。
---

# OpenShell 策略顾问

当请求因 `policy_denied` 失败时，如果用户任务仍需要该请求，不要把拒绝当作最终结果。读取 `/etc/openshell/skills/policy_advisor.md`，检查 `http://policy.local/v1/policy/current`，向 `http://policy.local/v1/proposals` 提交范围最窄的提案，等待 `/v1/proposals/{chunk_id}/wait?timeout=300`，仅在获得批准且 `policy_reloaded: true` 后重试。