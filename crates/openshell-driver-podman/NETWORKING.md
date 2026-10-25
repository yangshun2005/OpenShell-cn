# Podman 网络

只有外部 supervisor 具备外部网络连接。工作负载容器使用 `network=none`；其回环 DNS 中继和 TCP socket 中介通过受保护的 Unix socket 到达 supervisor，而非 veth 或代理环境变量。

```text
工作负载容器                           supervisor 容器
agent -> sandbox -- 私有 UDS / gRPC -> 策略代理 -> 主机网络 -> 目标
                                            |
                                            +-- 已认证的网关会话
```

## 外层网络围栏

驱动创建的工作负载不带网络、主机别名、发布端口或额外能力。启动和重启前，驱动会检查 Podman inspect 响应。sandbox 在执行 agent 前安装 seccomp 中介和 Landlock。它不创建网络命名空间、不配置 nftables，也不要求 `CAP_NET_ADMIN`。

TCP 打开、TCP 字节流、DNS 请求/应答和生命周期操作共用已认证的 gRPC 通道。DNS 由 supervisor 解析并授权。不支持通用 UDP。

驱动将只读的、驱动自有的 `/etc/resolv.conf` 挂载到工作负载，内容为 `nameserver 127.0.0.53`。这防止断网的 Podman 网络命名空间用运行时解析器替换策略 DNS 中继。supervisor 以可信的具体目标地址应答 `host.openshell.internal`：原生 Linux 上为 `127.0.0.1`，macOS 上为配置的 Podman Machine 主机地址。该目标地址属于 supervisor 引导配置，sandbox 重启时会重新应用。

## Supervisor 网络

supervisor 伴生容器使用 Podman 的主机网络。在 Linux 上，它连接网关的主回环端点。在 macOS 上，Podman Machine 提供主机回环路由。上游企业代理仅作用于 supervisor。网关的 SSH 隧道通过 supervisor 中继的私有 Unix socket 连接，因此驱动不发布 supervisor 端口。

这些运行时管理的网络辅助组件位于工作负载信任边界之外。共享工作负载的用户命名空间可保留卷的 UID/GID 映射；但不共享工作负载的 PID、挂载或网络命名空间。

## 故障排查

用相同的 sandbox-ID 标签检查两个容器，区分 `openshell.ai/isolation-role=sandbox` 与 `openshell.ai/isolation-role=supervisor`。

- Sandbox 未通过资格探测：查看其日志，定位被拒绝的内核/运行时原语。不要添加能力或禁用运行时 seccomp。
- Sandbox 无法向 supervisor 认证：检查私有通道卷、匹配的用户命名空间映射和共享的 SELinux 标签。
- Supervisor 无法连接：检查其配置的网关端点、凭据、主机网络和网关的主监听器。
- DNS 或出站被拒：检查 supervisor 的策略决策。不要添加工作负载网络、解析器绕过或直连网关路由。
- 配对未就绪：检查 supervisor 健康 socket 和网关会话。仅工作负载容器运行并不代表就绪。

运行时选项参见[驱动概览](README.md)和
[Podman 运行时文档](https://docs.podman.io/en/latest/markdown/podman-run.1.html)。