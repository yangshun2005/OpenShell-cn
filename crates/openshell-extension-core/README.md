# OpenShell 扩展核心

`openshell-extension-core` 包含两个或更多 OpenShell 扩展机制共享的协议无关原语。目前负责扩展身份与受众值、扩展 JWT 声明契约、可刷新的 bearer 凭据及其按服务存储、以及面向 HTTP、HTTPS 和 Unix socket 的出站 gRPC 传输构建。

`ExtensionKind` 为非穷尽枚举，新增扩展点可加入共享的身份与受众模型而不破坏使用方。但新类型在网关接入接受之前，仍须定义自己的注册、传输和授权语义。

出站 HTTPS 认证通过共享的 `ExtensionServerTrust` 策略选择。目前已实现平台根证书和运维方提供的 CA 包。非穷尽的策略边界允许将来一次性加入 SPIFFE X.509-SVID 或公钥固定支持，而无需分别修改中间件和拦截器客户端。Unix socket 保留其本地操作系统访问边界。

中间件或拦截器专用的 protobuf 客户端、策略选择、编排和生命周期管理仍留在各自所属的 crate 中。网关签名权限也留在 `openshell-server`。此归属规则可防止本 crate 随扩展支持增长而变成通用堆放地。