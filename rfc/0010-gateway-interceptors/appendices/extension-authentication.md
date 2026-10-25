# 附录：扩展认证（Alpha）

> 本附录是 [RFC](../README.md) 的附录。阅读前请先熟悉 RFC 正文。

RFC 正文规定所有网关拦截器连接都需要认证，但未限定认证模型，并建议实现应支持 mTLS 和 bearer-token 认证。本附录记录 alpha 阶段实际构建的机制，取代正文中该段内容；正文其余部分不变。

网关拦截器与 supervisor 中间件共用同一实现。完整的声明契约、授权模型、密钥分发和残余风险统一记录在 [RFC 0009 的附录](../../0009-supervisor-middleware/appendices/extension-authentication.md)中。本附录只记录拦截器不同之处。

## Alpha 阶段交付内容

只交付正文建议中的 bearer-token 部分，不含 mTLS。网关为 `Describe`、`Evaluate` 和 provider-profile 快照调用附加短期、精确受众的 Ed25519 JWT。mTLS 仍作为后续加固项推迟。

拦截器只由网关调用，因此每个令牌都携带 `caller_kind: gateway`，且不含 `sandbox_id`。没有 supervisor 侧分发路径，也没有基于策略的授权步骤：网关在启动时根据配置自行签发凭据，并原地轮换。

## 传输

与中间件不同，拦截器保留正文中的 Unix domain socket 选项。网关本地 socket 只有唯一调用方可达，因此配置网关 JWT 签名时，`https://` 和 `unix://` 均可接受。HTTPS 端点可固定运维提供的 CA 证书包，并保留正常的主机名验证。

## 受众约定

受众默认值为 `urn:openshell:extension:interceptor:<name>`，也可按拦截器显式设置。拦截器可在 `Describe` 清单的 `expected_audience` 字段中声明其验证的受众。认证后的 `Describe` 成功后，网关将该值视为一致性断言；若与配置的受众不一致，则拒绝启动。严格的验证器可能在返回清单前就拒绝错误受众，因此这不提供受众发现能力。

正文已将服务不可用、清单无效或未授权绑定视为启动失败，这与现有姿态一致：拦截器配置问题在网关开始服务流量之前暴露。

## 兼容性

拦截器可设置 `allow_insecure_transport = true`，以保留不带凭据的明文 `http://` 端点。网关每次启动都会记录一条警告并指明该拦截器。此选项用于本地开发，以及扩展认证出现之前已配置的部署。

## 推迟项

mTLS 客户端认证、带重叠窗口的多密钥轮换，以及超出短期过期之外的重放防护，仍为后续工作，详见 RFC 0009 的附录。