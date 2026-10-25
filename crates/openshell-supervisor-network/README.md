# 网络监督器

本 crate 禁止 unsafe 代码，包括其本地测试。类型化 socket API 为透明代理提供原始目标地址查询。

Socket 归属测试将已连接 socket 的副本作为子进程的 stdin 传入。重新执行测试二进制可检查同一二进制标识的多个持有者；执行不同二进制可检查共享 socket 标识不明确的情况。父测试结束时，测试夹具会杀死并回收子进程。

手动代理性能基线仍统计内存分配。其仅用于测试的全局分配器使用 `openshell-driver-vm::allocation_tracking`；unsafe 分配器实现属于 VM crate。VM 依赖禁用了默认特性，因此该插桩不需要计算运行时或仅限 Unix 的沙箱库。它仅是开发依赖。

## 沙箱实例输入

`run_networking` 接收一个 `SandboxInstanceNetwork`，包含沙箱标识、沙箱实例的 `GatewayEndpoint`、用于提供方令牌授予的 SPIFFE Workload API socket、`policy.local` 拒绝来源以及代理 CA 文件。这些均不从进程环境读取，因此一个进程可运行多个沙箱实例。每个实例拥有自己的提供方令牌缓存和 HTTP 客户端；为一个沙箱获取的令牌绝不会提供给另一个沙箱。没有 Workload API socket 时，令牌授予按失败关闭处理。托管多个实例的进程从每实例的 `SandboxInstanceDenialLog` 提供 `/v1/denials`，而非共享的简写日志文件。