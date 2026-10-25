<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# OpenShell 网关服务器

服务器负责控制平面 RPC、认证、提供商凭据下发，以及沙箱生命周期编排。`openshell-gateway` crate 将服务器与选定的计算驱动组合在一起。

## 沙箱启动身份

新的启动（包括自动重启）在调用计算驱动之前，会提交一个新的运行时代次和递增的授权纪元。启动会话 ID 由持久化沙箱 ID 和代次派生。重试复用该代次和会话，网关重启后亦然。凭据续期保留两者；旧启动的凭据无法授权新启动。

## 运维人员凭据导出

`GetProviderCredentials` 从显式命名的工作区中，导出某个提供商所选配置文件声明的运行时环境键。它要求独立的 `Principal::Operator`，该身份仅在 `mtls_auth.operator_enabled` 启用时，由带有 `OU=operator` 的已验证 TLS 叶子证书建立。TLS 监听器单独提供其证书指纹，与用户身份分开；仅凭身份的普通服务路径无法建立运维人员身份。

运维人员显式继承用户/平台管理员权限，但不继承沙箱或对等身份。描述符授权和处理程序守卫双重保护导出 RPC。运维人员变更准入使用与具有相同证书主题的普通 mTLS 用户不同的调用方命名空间。凭据响应绝不进入持久化变更重放或通用拦截器路由。

导出解析器在铸造前验证整个选择，仅解析选定的凭据句柄，并在下发前对提供商修订、刷新纪元和配置文件声明进行围栏。驱动过期和提供商过期取较早的时间戳合并。OAuth 铸造输入、仅限 broker 的键，以及绑定到 supervisor 的动态授权不可导出。失败的响应仍可能已提交刷新副作用。缺失刷新所拥有的存储输出会触发读取时刷新；缺失静态输出及其他凭据后端错误会失败关闭，不回退到内联值。

## 刷新协调

显式轮换、后台刷新和运维人员读取时刷新共享同一刷新协调：

- 进程内弱键控互斥锁合并等待者，不会无限期保留已删除的提供商。
- 有界准入队列和执行器池限制分离的刷新工作。
- PostgreSQL 部署还会获取专用的按提供商/按凭据咨询锁；SQLite 使用本地互斥锁。Worker 密钥清理获取相同的锁，并在更改资源版本前重新加载状态。
- 持久化的 `refresh_in_progress` 标记先于上游铸造。`refresh_committing` 标记保护授权持久化与提供商句柄持久化之间的间隔。只有完成的提供商提交才发布 `refreshed` 状态，并在元数据注解中带有唯一的已完成铸造身份。等待者针对该身份合并，而非针对资源版本的簿记变更。
- 自动调用方在两把锁下重新检查实时恢复和重试截止时间。失败的铸造会阻塞排队的自动交换，但不会禁用计划中的 Worker 重试或单独请求的手动轮换。
- 分离的刷新任务在请求取消后仍会完成，保留颁发者返回的替换刷新令牌。
- 现有的代次/CAS 围栏对删除和重新配置仍然有效。模糊的轮换令牌交换会被搁置等待恢复，而非自动再次消耗授权。

崩溃或数据库锁会话丢失可能留下搁置的标记。调查颁发者状态后，重新配置或重新授权刷新授权。直接更改标记不是受支持的恢复流程。

## 针对性验证

```shell
cargo test -p openshell-server --features test-support,prebuilt-z3
cargo clippy -p openshell-server --all-targets --features test-support,prebuilt-z3 -- -D warnings
```

被忽略的 `postgres_refresh_locks_coordinate_independent_replica_sessions` 测试在 `mise run test:rust:postgres` 下运行（参见 `TESTING.md`），仍接受指向一次性 PostgreSQL 数据库的 `OPENSHELL_REFRESH_TEST_DATABASE_URL`。它在没有本地互斥锁的情况下测试独立会话。不要将此测试夹具指向现有部署数据库。