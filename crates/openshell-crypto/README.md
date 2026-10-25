# OpenShell 加密后端

架构、贡献者规则以及生产环境 OpenSSL/FIPS 后续要求，参见 [密码学与 FIPS](CRYPTO.md)。

`openshell-crypto` 负责第一方加密后端的选择。AWS-LC 是唯一的生产实现。本次变更保留 TLS 算法、Ed25519 网关 JWT、P-256 证书密钥、原生信任根和凭据信封格式。该接口旨在为受监管部署支持系统 OpenSSL 后端，相关 Issue 见 [#900](https://github.com/NVIDIA/OpenShell/issues/900)。本阶段仅实现 AWS-LC。OpenSSL 概念验证将单独验证 provider 集成；这不构成 FIPS 构建或合规声明。

## 边界

`CryptoBackend` 定义随机数、增量 SHA-256 和 AES-256-GCM，不暴露加密库类型。`ProtocolBackend` 增加 Rustls、rcgen 和 jsonwebtoken 适配器。这些适配器保留各自协议库的校验和编码逻辑，不暴露 AWS-LC 类型。后续原生 OpenSSL TLS 集成可能需要额外的传输适配器；本接口不声称提供该适配器。

摘要的创建、更新和终结均返回 `Result`，以便后端报告 provider 和操作失败。调用方在更新出错后丢弃摘要；一次性 SHA-256 辅助函数传播首个失败，不做回退。

调用方使用 `aead`、`pki`、`tls` 和 `jwt`，或显式使用 `CryptoContext`。证书策略和 JWT 声明校验仍由现有调用方负责。`pki::KeyPair` 持有后端的 `SigningKey`，并直接实现 rcgen 0.14 的 `PublicKeyData` 和 `SigningKey` trait。后端负责生成/导入密钥、签名消息并导出 PKCS#8 DER。PEM 封装和证书编码仍留在门面层。公钥暴露算法特定字节和 SPKI；私钥导出可能失败，包括不可导出的密钥。证书编码从不需要导出私钥。`pki::Certificate` 将公开签发参数与编码后的证书分开保存，使生成的证书可作为签发者使用而无需保留私钥。持久化的代理 CA 使用 `pki::issuer_from_der`，并在签发的证书链中保留原始证书字节。持久化的 CA 密钥通过所选后端导入。Provider 资源必须在密钥整个生命周期内保持存活。默认证书序列号和密钥标识符通过所选摘要后端派生，保留现有 SHA-256 派生方式，不依赖 rcgen 编译时的加密实现。

## 构建选择

本 crate 的默认 `aws-lc` feature 选择当前实现。根工作区在其 `openshell-crypto` 依赖中统一声明该选择；应用 crate 继承该选择，不单独选择后端。独立示例通过其路径依赖选择自己的默认值。禁用 crate 默认 feature 会移除其 AWS-LC 实现，使用前必须提供显式上下文；未初始化即使用会以初始化 panic 失败关闭。此模式面向嵌入方和后端契约测试，不是可独立运行的 OpenShell 产品配置。

集成 feature（`tls-tonic`、`tls-kube`、`tls-sqlx`）激活依赖。`aws-lc` feature 有条件地提供它们的 AWS-LC 选择。这些库仍在内部自行选择加密实现，不查询 `CryptoContext`。Cargo feature 统一不会阻止直接使用这些库。第二个后端必须适配这些集成和应用启动路径。仅禁用本 crate 的 feature 不能从整个产品依赖图中移除依赖方自带的加密实现。

后续 OpenSSL 构建必须使用系统共享库、不做 vendoring、遵循系统 provider 配置，并验证最终链接产物。链接和模块资格属于构建/部署关注点，与 Rust 密钥和原语契约分离。概念验证将在添加生产 OpenSSL 后端之前测试这些关注点以及 Rustls provider 集成。

## 生命周期与姿态

启用该 feature 时，`default_context()` 惰性选择 AWS-LC。嵌入方可在首次使用前调用 `install_default_context(context)`。重新安装同一上下文的克隆会成功；安装不同上下文即使名称相同也会失败。不支持运行时切换后端。独立上下文可在不改变进程状态的情况下使用，包括在契约测试中。

`tls::provider()` 构造所选上下文的 provider。`tls::ensure_default_provider()` 保留已有的 Rustls 进程默认 provider 并返回它；它不证明所有权。直接构造的 OpenShell TLS 配置使用门面 provider，包括 CLI 健康检查连接器。下列依赖方自带的客户端保留各自的选择路径。若在门面使用前已显式选择上下文，这些配置使用该上下文，即使另一库已初始化 Rustls。否则它们保留嵌入方已安装的 provider。密钥加载、自定义校验算法和网关客户端证书校验器使用相同的配置-provider 策略。显式协议版本约束保持不变。与所请求版本不兼容的 provider 在构造时失败，不回退。此规则控制第一方配置；无法覆盖依赖方自带的 TLS。

JWT 初始化同样保留已有的进程 provider。`install_jwt_provider()` 报告本次调用是否安装了它；jsonwebtoken 10 没有公开 getter 可验证先前已安装的 provider。JWT 适配器保留该库的算法/密钥绑定和声明校验检查。请在 JWT 初始化前选择自定义上下文，包括依赖的间接初始化。两个全局 provider 都无法安全替换。

`CryptoContext::verify_posture(false)` 报告上下文的能力，而非进程全局对象的所有权。本阶段严格请求始终被拒绝。模块版本显式标记为未知，因为后端包装器未通过此处使用的 API 提供该信息。后端身份、启用的算法和一次成功操作均不构成部署 FIPS 证据。

## 兼容性与排除项

AEAD 存储随机 12 字节 nonce，并在密文后附加 16 字节 tag。AAD 构造、信封版本、密钥 ID 和编码仍由凭据驱动负责。随机 nonce 不保证唯一性；现有密钥容量和轮换约束仍然适用。认证失败不暴露明文，也不区分错误密钥、AAD、nonce、密文或 tag。

覆盖范围排除依赖方自带的 SSH/russh 原语、AWS SigV4 签名、SPIFFE RustCrypto 校验和非 Rust 客户端运行时。AWS SDK HTTPS 也保留其依赖方自带的选择。非安全内容哈希和普通调度随机数不迁移。原语能力报告不声称具备 SSH 或 PQC 能力。OpenSSL、严格策略、模块版本发现和部署资格属于后续工作。

运维证书指纹、运行时刷新重放哈希及派生 token ID、bearer 缓存指纹、Sandbox Protocol 请求摘要和请求 ID 熵、Kubernetes/VM 运行时 fence 熵、VM socket 目录熵以及 rootfs 暂存 token 熵使用所选后端。这些操作保留 SHA-256、适用处的 UUID v4 编码以及现有十六进制编码。后端失败会传播；调用方不得替换为其他实现或使用占位身份继续。

其余第一方用途需要显式分类；门面不构成全应用加密覆盖声明：

| 剩余用途 | 当前范围与后续工作 |
| --- | --- |
| Reqwest 客户端（OIDC/JWKS、token 交换、provider 刷新、Vault、token 授予、遥测） | 这些客户端在确保存在进程默认 provider 后使用 Rustls 的进程默认 provider；它们不一定遵循显式选择的 `CryptoContext`。OpenSSL 后续工作必须提供兼容的预配置 TLS 或其他受支持的传输集成，并测试实际 provider 使用。 |
| Kubernetes 客户端 TLS（`kube-client` 0.99） | 启用 AWS-LC 的客户端尝试将 AWS-LC 安装为 Rustls 进程默认 provider。它不查询 `CryptoContext`；OpenSSL 后续工作必须在声称所选后端覆盖前适配此传输。 |
| 外层 fence 证据摘要（`openshell-isolation-interface::contract`） | 该摘要通过 `Sha256Digest::compute` 使用 `sha2` 绑定生成和原生执行证据。此安全相关哈希仍在门面分发之外，必须在声称独占后端所有权前迁移并传播失败。 |
| VM 镜像/归档摘要校验和缓存 supervisor 校验（`openshell-driver-vm`） | 安全相关的产物完整性仍在此运行时凭据迁移之外。迁移它需要同时覆盖流式读取器、嵌入产物缓存和构建时摘要生成；必须在声称独占后端所有权前处理。 |
| Sandbox Protocol 请求信封之外的 UUID 生成 | 包括运行时身份、认证 nonce 和会话标识符以及普通关联 ID。这些仍使用 UUID 库的熵源。它们不由上下文姿态证明；后续工作必须分类每个用途，并通过当前不可失败的构造函数传播熵失败。 |
| DNS 策略契约指纹（`openshell-supervisor-network::policy_dns`） | 这些将缓存的 DNS 状态绑定到策略，具有安全相关性。它们仍在此迁移之外；迁移缓存身份 API 及其失败处理是独占后端所有权的必要条件。 |
| 凭据存储名称（`credentials`、Kubernetes Secrets、Vault） | 确定性命名空间派生，不是加密或授权证明。这些保留当前哈希实现和存储名称；不声称后端覆盖。 |
| Provider 刷新 advisory-lock 键 | 用于协调刷新的确定性数据库锁选择，不是认证或加密。这些仍在所选后端契约之外。 |
| 端点 ID 和分页指纹 | 确定性标识/请求匹配，不是认证。这些仍在所选后端契约之外。 |
| 调试 token 指纹 | 仅用于诊断显示；直接哈希仍在契约之外，即使输入是凭据。 |
| Schema 兼容性测试、内容缓存身份、重试抖动、测试夹具 | 非认证用途保持不变。 |

Cargo feature 选择不能阻止 crate 添加其他加密依赖。请对照此范围审查第一方安全敏感操作；未安装源码扫描边界 lint。

持久化代理 CA 加载使用后端密钥导入和 `pki::issuer_from_der`。该辅助函数使用不带校验 feature 的 `x509-parser` 读取 subject、key usage 和 subject key identifier。缺失标识符时使用所选后端对 SPKI 的 SHA-256，截断为 20 字节。它拒绝尾部多余字节、格式错误的元数据以及 rcgen 无法无损表示的 subject 名称（包括重复属性 OID 和多值 RDN）。解析不建立信任或验证签名；代理通过所选 TLS 后端单独检查证书/密钥匹配。rcgen 的 `x509-parser` feature 保持禁用：在 0.14.10 中它还会编译需要 AWS-LC 或 Ring 的 CSR 校验。CA 导入不需要 fork 或回退后端，解析器原先的 Ring 例外已从 `deny.toml` 移除。禁用默认 feature 时，独立门面也排除 AWS-LC。构建时 Z3 归档下载器保留 `deny.toml` 中窄范围允许的 Rustls/Ring 包装器；上下文能力不证明构建工具。

## 扩展与检查

实现两个 trait 时不要导入应用 crate。在操作整个生命周期内保留 provider/密钥资源，传播熵和认证失败，失败后绝不回退。记录不支持的算法，避免含密钥的诊断信息。添加协议适配器，而不是将证书或授权策略移入原语实现。

运行 `cargo test -p openshell-crypto` 进行已知答案加密、篡改、JWT 校验、密钥导入/导出、加密兼容性、上下文替换以及进程默认 provider 被抢占后的实际 TLS 握手测试。`cargo test -p openshell-crypto --no-default-features --test independent_backend` 在不使用 AWS-LC 或 Ring 的情况下运行独立的仅测试密钥和 AEAD 实现。使用 `cargo tree -p openshell-crypto --no-default-features --edges normal,build,dev` 检查该边界。Branch Checks 将此测试与工作区测试分开运行，并拒绝依赖树中的 Ring 或 AWS-LC，包括构建和测试依赖。现有 bootstrap、凭据存储、网关 TLS/OIDC 和代理测试覆盖已迁移的消费者。cargo-deny 的 Ring 限制及其显式包装器例外仍然有效。