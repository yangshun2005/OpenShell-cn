# OpenShell Workspace Helm Chart

<!--
本文件由 helm-docs 生成。
请编辑 README.md.gotmpl 和 values.yaml，然后运行 `mise run helm:docs`。
-->

> **实验性** — 共享网关、多命名空间部署路径仍在设计中。

此 Chart 安装 OpenShell Kubernetes 沙箱所需的命名空间级 ServiceAccount、RBAC 和 NetworkPolicy。在每个平台管理的 workspace 命名空间中安装一次。它不创建命名空间，也不部署 OpenShell 网关。

先以 `workspaceResources.enabled=false` 安装网关 Chart，再以网关 ServiceAccount 身份安装此 Chart。当网关服务多个预置 workspace 命名空间时，将其 Kubernetes 驱动配置为 `operator` workspace 模式：

```shell
helm install openshell-workspace ./deploy/helm/openshell-workspace \
  --namespace app-a \
  --set gateway.serviceAccount.name=openshell \
  --set gateway.serviceAccount.namespace=openshell
```

保持 `sandboxServiceAccount.name` 与网关 Chart 的 `sandboxServiceAccount.name` 一致。两个 Chart 的默认值均为 `openshell-sandbox`。若网关启用了 `server.drivers.kubernetes.allowDriverConfig`，还需在每个 workspace release 中设置 `gateway.allowDriverConfig=true`，以便允许调用方选择的 PVC。

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| fullnameOverride | string | `""` | 覆盖生成的完整资源名称。 |
| gateway.allowDriverConfig | bool | `false` | 授予网关启用调用方驱动配置时所需的 PVC 元数据读取权限。需与网关 Chart 中的 server.drivers.kubernetes.allowDriverConfig 保持一致。 |
| gateway.networkPolicy.podSelector | object | `{"app.kubernetes.io/instance":"openshell","app.kubernetes.io/name":"openshell"}` | 选择允许访问沙箱 SSH 的网关 Pod 的标签。 |
| gateway.serviceAccount.name | string | `"openshell"` | 共享网关 ServiceAccount 的名称。 |
| gateway.serviceAccount.namespace | string | `"openshell"` | 包含共享网关 ServiceAccount 的命名空间。 |
| nameOverride | string | `""` | 覆盖生成资源名称中使用的 Chart 名称。 |
| networkPolicy.enabled | bool | `true` | 将沙箱 SSH 入站限制为仅共享网关 Pod 可访问。 |
| sandboxServiceAccount.annotations | object | `{}` | 添加到生成的沙箱 ServiceAccount 的注解。 |
| sandboxServiceAccount.create | bool | `true` | 创建分配给沙箱 Pod 的 ServiceAccount。 |
| sandboxServiceAccount.name | string | `"openshell-sandbox"` | 沙箱 ServiceAccount 名称。 |

----------------------------------------------
使用 [helm-docs v1.14.2](https://github.com/norwoodj/helm-docs/releases/v1.14.2) 从 Chart 元数据自动生成