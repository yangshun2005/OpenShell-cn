# 共享 supervisor 控制

`openshell-supervisor-control` 是多个沙箱共享的 supervisor 的网关侧。一个共享 supervisor 进程托管一个网关的沙箱实例（每个稳定 Sandbox 的一个运行代次）；网关决定托管哪些实例，并通过私有 `openshell.supervisor.host.v1` 流把该集合提供给 supervisor，supervisor 用网关签发的 supervisor 令牌打开该流。该控制关系中与计算平台无关的部分都在这里，因此计算驱动只需适配存储和基础设施即可接入共享 supervisor，无需重新实现控制契约。

## 模块

- `controller`：`SupervisorHostController` 持有期望的 `SandboxInstanceAssignment` 集合，并将其提供给 host。在从持久记录中完整推导出集合之前，期望状态处于未初始化状态，因此重启后的网关绝不会把空集合当作权威状态；初始化后显式设置为空集合则会停止实例。全量替换会与列出记录前读取的修订版本进行比较并替换。同一时间只允许一个 host 会话：同一 host 进程更新的连接 epoch 或新进程会取代当前流，延迟到达的旧 hello 会被拒绝，来自已被取代的流的报告、从未发送过的修订版本的 ack、回退的 ack 以及超大报告都会被丢弃。打开流的令牌过期时流结束；已托管的实例继续运行。最多分配 256 个实例。
- `registration`：`SandboxInstanceRegistration` 是一个分配所携带、由驱动随代次持久化的 JSON 启动材料。它只授予 supervisor 该代次的网关和 Sandbox Protocol 凭据。`validate` 将路由、沙箱、代次、启动会话和代理 CA 绑定在一起；`from_launch_material` 和 `from_assignment` 解码有界材料且不引用值，这些值包含凭据。
- `status`：`SandboxInstanceStatusReport` 及其阶段，与流的状态消息相互转换；未知阶段视为失败。
- `coordinator`：`SharedSupervision` 是网关构造并交给计算驱动的对象。`launch` 验证驱动的注册信息，通过驱动的操作持久化编码后的启动材料，然后才分配实例；容量已满的 host 会在工作负载运行前被拒绝。`begin_recovery` 在驱动列出其持久记录前捕获修订版本，`complete_recovery` 复用当前分配，通过驱动的操作读取其余分配，将每个分配绑定到其记录的路由和沙箱，并在任何读取失败时保留之前的集合；与生命周期编辑发生竞争的列表会被拒绝并重新推导。`supervision` 和 `wait_for_ready` 在一处解释 host 的观测结果：断开的 host 为未知，尚未协调到该分配的 host 为降级，只有已协调越过该分配但未托管该实例、为另一个沙箱托管该实例，或报告该实例已结束的 host，才会使实例不可用。
- `credential`：`deliver_credentials` 通过驱动的操作投递当前 supervisor 令牌及每次续期，投递失败会重试，直到成功或被更新的令牌取代。
- `proto`：私有线协议，在 `openshell-core` 中生成。

## 归属

网关（`openshell-server`）在所选进程内驱动请求时构造 `SharedSupervision`，保留 controller 以服务 host 的 `Attach` 流，并发布其签发和续期的令牌（在其生命周期一半时续期），然后将协调交给驱动。驱动（目前是 `openshell-driver-kubernetes`）记录放置位置，持久化并读回启动材料，为工作负载配置通往 supervisor 的路径，结合协调器的判定检查自身的基础设施证据，并将令牌投递到 supervisor Pod 挂载的 Secret 中。supervisor（`openshell-supervisor`）只消费注册和状态类型；它负责准入、每实例运行时、路由和清理。