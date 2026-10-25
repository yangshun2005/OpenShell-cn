# OpenShell Helm Chart

<!--
本文件由 helm-docs 生成。
编辑 README.md.gotmpl 和 values.yaml，然后运行 `mise run helm:docs`。
-->

> **实验性** - Kubernetes 部署路径仍在积极开发中。预期会有粗糙边缘和破坏性变更。

此 Chart 将 OpenShell 网关部署到 Kubernetes 集群。它以 OCI 构建产物形式发布到 GHCR：`oci://ghcr.io/nvidia/openshell/helm-chart`。

默认情况下，此 Chart 还会创建沙箱所需的命名空间级资源。对于共享网关部署，请使用 `workspaceResources.enabled=false` 安装，然后在每个预配置的工作区命名空间中安装 `deploy/helm/openshell-workspace` Chart。网关和工作区发布版本随后可独立升级和删除。当一个网关服务多个预配置工作区命名空间时，使用 Kubernetes `operator` 工作区模式。

## 证书 Hook 元数据

使用 `pkiInitJob.podLabels` 和 `pkiInitJob.podAnnotations` 为 `<fullname>-certgen` 和 `<fullname>-certgen-backend-ca` Job 的 Pod 模板添加元数据。启用时，这些设置应用于两个 Hook，不影响网关或沙箱 Pod。自定义键冲突时，Chart 拥有的 Pod 标签优先。

例如，在已注入的命名空间中为 Hook Pod 禁用 Istio sidecar 注入：

```yaml
pkiInitJob:
  podAnnotations:
    sidecar.istio.io/inject: "false"
```

## 集群级与命名空间级对象

此 Chart 中的大多数对象是命名空间级的，位于发布版本命名空间中。只有两个是集群级的：

| 对象 | 默认名称 |
| --- | --- |
| `ClusterRole` | `<fullname>-node-reader-<release namespace>` |
| `ClusterRoleBinding` | `<fullname>-node-reader-<release namespace>` |

默认情况下，发布版本会创建两者，因此集群管理员安装的行为不变。在集群级 RBAC 由其他团队管理的集群上，将安装拆分为两步。

集群管理员为每个网关 ServiceAccount 应用一次集群级对象，使用与发布版本相同的 values 渲染：

```shell
helm template openshell oci://ghcr.io/nvidia/openshell/helm-chart --version <version> \
  --namespace openshell -f my-values.yaml \
  --set rbac.create=true \
  --set rbac.clusterScoped.create=true \
  --set agentSandbox.preflight.enabled=false \
  --show-only templates/clusterrole.yaml \
  --show-only templates/clusterrolebinding.yaml | kubectl apply -f -
```

然后命名空间管理员在省略集群级对象的情况下安装和升级发布版本，使用 [`ci/values-namespace-admin.yaml`](ci/values-namespace-admin.yaml) 或等效的 `--set`：

```shell
helm upgrade --install openshell oci://ghcr.io/nvidia/openshell/helm-chart --version <version> \
  --namespace openshell -f my-values.yaml \
  --set rbac.clusterScoped.create=false
```

网关 ServiceAccount 的名称和命名空间不变，因此预创建的 `ClusterRoleBinding` 仍与发布版本匹配。这也适用于 `serviceAccount.create=false`：`ClusterRoleBinding` 的 subject 跟随 `serviceAccount.name`，因此请用相同的 values 渲染管理员步骤。

### 安装者需要哪个标志

在默认的 `shared` 工作区模式下，发布版本还会创建一个命名空间级沙箱 `Role`，授予 Agent Sandbox（`agents.x-k8s.io`）权限。Kubernetes 禁止授予你不持有的权限，而内置的 `admin` ClusterRole 不覆盖该 CRD，因此仅持有 `admin` 的安装者无法创建它。此时仅 `rbac.clusterScoped.create=false` 不够，安装会失败并报 `attempting to grant RBAC permissions not currently held`。

| 工作区模式 | 安装者持有 | 使用 |
| --- | --- | --- |
| `shared` | 仅内置 `admin` | `rbac.create=false`，集群管理员预先创建所有网关 RBAC |
| `shared` | `admin` 加上命名空间中的沙箱权限 | `rbac.clusterScoped.create=false` |
| `managed`、`operator` | 仅内置 `admin` | `rbac.clusterScoped.create=false` |

`managed` 和 `operator` 不渲染命名空间级沙箱 `Role`，因此那里不需要额外授权。

使用 `rbac.create=false` 时，集群管理员也应用命名空间级 RBAC，将其添加到同一渲染中：

```shell
helm template openshell oci://ghcr.io/nvidia/openshell/helm-chart --version <version> \
  --namespace openshell -f my-values.yaml \
  --set rbac.create=true \
  --set rbac.clusterScoped.create=true \
  --set agentSandbox.preflight.enabled=false \
  --show-only templates/clusterrole.yaml \
  --show-only templates/clusterrolebinding.yaml \
  --show-only templates/role.yaml \
  --show-only templates/rolebinding.yaml \
  --show-only templates/peer-role.yaml | kubectl apply -f -
```

要改为授予安装者沙箱权限，将其绑定到携带与 Chart 的 `openshell-sandbox` Role 相同规则的 Role。

certgen Hook 和凭据驱动 RBAC 保留各自的标志（`pkiInitJob.enabled` 和 `server.credentialDrivers.kubernetesSecrets.rbac.create`）。

### 迁移现有发布版本

Helm 会删除离开发布版本清单的对象，因此在已拥有 `ClusterRole` 和 `ClusterRoleBinding` 的发布版本上设置 `rbac.clusterScoped.create=false` 会删除它们。网关随后会失去 TokenReview，直到集群管理员重新应用它们。先以集群管理员身份移交所有权，这样不会删除任何内容：

```shell
kubectl annotate clusterrole "openshell-node-reader-<namespace>" \
  helm.sh/resource-policy=keep --overwrite
kubectl annotate clusterrolebinding "openshell-node-reader-<namespace>" \
  helm.sh/resource-policy=keep --overwrite
```

这些对象随后在设置该标志的升级中存活，集群管理员从那时起拥有它们。全新安装不需要此步骤。

`rbac.clusterScoped.create` 独立于 `server.drivers.kubernetes.workspaceMode`。Managed 和 operator 模式改变 `ClusterRole` 的内容，但从不强制命名空间级发布版本应用它。更改影响 `ClusterRole` 规则的 values 后，重新运行集群管理员步骤。

## 前提条件

> **必需：** 你的集群 CNI 必须在每个沙箱命名空间中强制执行 Kubernetes `NetworkPolicy` 的入站和出站规则。OpenShell 创建策略，但即使没有 CNI 强制执行，Kubernetes 也会接受它们。没有强制执行时，沙箱工作负载可能直接连接并绕过 supervisor 网络策略。安装 OpenShell 前请验证 CNI 支持。

部署 OpenShell 前，必须在集群上安装 Kubernetes Agent Sandbox CRD 和控制器。使用以下命令安装：

```shell
kubectl apply -f https://github.com/kubernetes-sigs/agent-sandbox/releases/latest/download/sandbox.yaml
```

Chart 不安装此集群级依赖。默认情况下，当集群不提供任何受支持的 Sandbox API（`agents.x-k8s.io/v1beta1` 或 `agents.x-k8s.io/v1alpha1`）时，它会在创建网关资源前失败。对于离线 `helm template` 渲染（Helm 无法发现集群 API），使用 `agentSandbox.preflight.enabled=false` 禁用检查。

## 在 Kubernetes 上安装

```shell
helm install openshell oci://ghcr.io/nvidia/openshell/helm-chart --version <version>
```

## 在 OpenShift 上安装

详见完整的 [OpenShift 安装指南](https://docs.nvidia.com/openshell/latest/kubernetes/openshift)。快速开始：

```shell
# 预创建 openshell 命名空间
oc create ns openshell

# 部署 openshell，使用覆盖项允许为网关分配 fsGroup 和 runAsUser 的 SCC
helm install openshell oci://ghcr.io/nvidia/openshell/helm-chart --version <version> -n openshell \
  --set server.disableTls=true \
  --set podSecurityContext.fsGroup=null \
  --set securityContext.runAsUser=null
```

在 OpenShift 4.22+ 上，通过 `BackendTLSPolicy` 支持端到端 TLS。详见 [OpenShift 安装指南](https://docs.nvidia.com/openshell/latest/kubernetes/openshift#end-to-end-tls-openshift-422)。

## 可用版本

| 标签 | 来源 | 说明 |
| --- | --- | --- |
| `<semver>`（例如 `0.6.0`） | 标记的 GitHub 发布版本 | 跟踪匹配的网关、沙箱和 supervisor 镜像版本。推荐用于生产。 |
| `<semver>-pre.N`（例如 `0.1.0-pre.3`） | 特定的预发布候选 | 不可变的候选固定版本，跟踪具有相同精确版本的镜像。 |
| `0.0.0-dev` | `main` 上的最新提交 | 浮动标签，每次推送时覆盖。`appVersion` 为 `dev`，因此镜像解析为 `:dev` 标签。 |
| `0.0.0-dev.<commit-sha>` | `main` 上的特定提交 | 每提交固定版本。Chart 版本和 `appVersion` 都使用完整的 40 字符提交 SHA，与 CI 推送的镜像标签匹配。 |

预发布和 `dev` 标签用于在发布前测试变更。生产部署应固定到稳定的标记发布版本。

## 配置

源默认值见 [`values.yaml`](values.yaml)。选定的覆盖文件：

- [`ci/values-gateway.yaml`](ci/values-gateway.yaml) - 仅网关配置
- [`ci/values-cert-manager.yaml`](ci/values-cert-manager.yaml) - cert-manager 集成
- [`ci/values-keycloak.yaml`](ci/values-keycloak.yaml) - Keycloak OIDC 集成
- [`ci/values-high-availability.yaml`](ci/values-high-availability.yaml) - 多副本外部 PostgreSQL 测试的 CI 覆盖
- [`ci/values-autoscaling.yaml`](ci/values-autoscaling.yaml) - 渲染可选网关 HorizontalPodAutoscaler 的 CI 覆盖
- [`ci/values-spire.yaml`](ci/values-spire.yaml) - SPIFFE/SPIRE 提供者令牌授权
- [`ci/values-spire-stack.yaml`](ci/values-spire-stack.yaml) - 用于本地开发的 SPIRE 加固 Chart values

### 数据库后端

默认情况下，OpenShell 使用 SQLite，并以 StatefulSet 运行网关，使数据库由每 Pod 的 PVC 支持：

```yaml
server:
  dbUrl: "sqlite:/var/openshell/openshell.db"
```

#### 数据 PVC 存储

使用 `server.dataStorageSize` 和 `server.dataStorageClass` 设置网关数据声明 `openshell-data` 的大小并选择其 StorageClass。两者都省略时，从集群默认 StorageClass 请求 `1Gi`。在没有默认 StorageClass 的集群上设置 `server.dataStorageClass`，否则网关 PVC 会保持 Pending。在 `workload.kind=deployment` 下这两个值都被忽略，因为不挂载 PVC。

```yaml
server:
  dataStorageSize: 20Gi
  dataStorageClass: fast-ssd
```

`volumeClaimTemplates` 不可变，因此请在安装时设置两者。要扩展现有声明，在其 StorageClass 允许扩展时就地扩展，然后重新创建 StatefulSet 使其模板匹配：

```shell
kubectl -n openshell patch pvc openshell-data-openshell-0 \
  -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
kubectl -n openshell delete statefulset openshell --cascade=orphan
helm upgrade openshell oci://ghcr.io/nvidia/openshell/helm-chart --version <installed-version> \
  --reuse-values -n openshell --set server.dataStorageSize=20Gi
```

重新创建的 StatefulSet 会原样重新接管现有声明，因此更改 StorageClass 意味着将数据复制到新声明。

#### 外部 PostgreSQL

当网关应连接到本 Chart 外部管理的数据库时，使用外部 PostgreSQL。OpenShell Chart 不部署数据库；请使用适合你环境的 Chart、operator 或托管服务单独安装 PostgreSQL，然后通过 Secret 传递连接 URI。

如果尚不存在，创建一个包含 PostgreSQL 连接 URI 的 Secret：

```bash
kubectl create secret generic my-pg-credentials -n openshell \
  --from-literal=uri="postgresql://user:pass@host:5432/dbname"
```

然后安装 Chart 指向该 Secret：

```bash
helm install openshell oci://ghcr.io/nvidia/openshell/helm-chart --version <version> \
  -n openshell \
  --set workload.kind=deployment \
  --set server.externalDbSecret=my-pg-credentials
```

对于外部数据库支持的多副本网关，使用 `workload.kind=deployment`。`workload.kind=statefulset` 仍可用于单副本 SQLite 安装以及明确需要 StatefulSet 身份或存储语义的运维人员。

### 凭据存储

默认情况下，Chart 使用网关的加密数据库凭据存储。网关将加密的提供者凭据信封写入 OpenShell 数据库。Chart 创建一个保留的 Kubernetes Secret 保存共享密钥加密密钥，并将该密钥注入每个网关 Pod，因此相同的默认值适用于单副本和外部数据库支持的 HA 部署。

当凭据应存放在集群或外部 Secret 后端时，使用 `gatewayConfig` 选择 `kubernetes-secrets` 或 `vault`。选择外部凭据驱动会禁用默认的凭据存储密钥加密密钥 Secret 和环境注入。该映射直接渲染为网关 TOML，因此使用网关的 snake_case 字段名：

```yaml
gatewayConfig:
  openshell.gateway:
    credential_drivers:
      - vault
  openshell.credential_drivers.vault:
    address: https://vault.vault.svc.cluster.local:8200
    mount: secret
    kv_version: "2"
    auth_method: kubernetes
    role: openshell-gateway
```

> `gatewayConfig` 只能包含非机密值。Helm 通用地序列化未知字段，无法确定任意字符串（如 `api_token`）是否机密。不要在此映射中放置密码、令牌、私钥、数据库 URL 或其他机密材料。改用 Secret 支持的环境变量、文件、卷或网关凭据驱动。Chart 会拒绝已知的不安全形式，如 `database_url`、内联 URL 凭据和 PEM 私钥；它不是通用机密扫描器。

对于 Kubernetes Secret 驱动，在同一映射中使用 `openshell.credential_drivers.kubernetes-secrets.namespace`。Chart 根据所选驱动派生所需的 RBAC；使用专用命名空间限制对 OpenShell 管理的 Secret 的访问。

#### OpenShift

在 OpenShift 上，将以下标志追加到上述任何 PostgreSQL 命令：

```
--set server.disableTls=true \
--set podSecurityContext.fsGroup=null \
--set securityContext.runAsUser=null
```

### 高可用性

仅在设置 `server.externalDbSecret` 时将 `replicaCount` 设为大于 `1`；默认 SQLite 数据库是每 Pod 的，无法协调多个网关副本。Chart 为网关到网关的中继流量创建一个无头 peer Service。StatefulSet Pod 通过该无头 Service 使用稳定的 Pod DNS 名称。Deployment Pod 通过 `OPENSHELL_PEER_ENDPOINT` 通告其 Pod IP，因为 Kubernetes 不为 Deployment 副本分配稳定的每 Pod DNS 名称。

网关 peer 流量使用 Kubernetes ServiceAccount 身份。每个网关 Pod 挂载一个投影的、绑定 Pod 的 ServiceAccount 令牌，受众为 `openshell-gateway-peer`；接收副本使用 Kubernetes TokenReview API 验证该令牌，验证实时 Pod UID 和 Helm 选择器标签，并仅授权内部 `PeerRelay` RPC。Chart 不创建或接受共享网关 peer Secret。

启用网关 TLS 时，peer 调用使用 Chart CA 和客户端 TLS Secret 进行服务器验证和 mTLS。客户端在直接连接到所属 Pod 时验证稳定的网关 Service DNS 名称。自定义 TLS Secret 必须在服务器证书中包含该 Service DNS 名称，并提供 `server.tls` 配置的 CA 和客户端凭据。

设置 `autoscaling.enabled=true` 为网关工作负载渲染 `autoscaling/v2` HorizontalPodAutoscaler。Deployment 或 StatefulSet 随后省略 `spec.replicas`，Chart 将其多副本检查应用于 `autoscaling.maxReplicas`。CPU 和内存目标需要匹配的 `resources.requests` 条目，或 `resources.limits` 条目（Kubernetes 会将其复制到 request 中）。使用 `autoscaling.metrics` 从指标适配器添加自定义指标。有关要扩展和告警的指标，请参阅[高可用性指南](https://docs.nvidia.com/openshell/latest/kubernetes/high-availability)。

在现有发布版本上启用自动扩缩会在该升级中从工作负载移除 `spec.replicas`。Kubernetes 可能将工作负载重置为一个副本，直到 HPA 将其扩展回至少 `autoscaling.minReplicas`，这会断开 supervisor 会话与其他网关 Pod 的连接；它们会重新连接到剩余副本。请在安装 Chart 时或维护窗口期间启用自动扩缩。在现有发布版本上，使用 `--reset-then-reuse-values` 升级，这需要 Helm 3.14 或更高版本。使用较旧的 Helm 时，使用 `helm get values <release> -o yaml > values.yaml` 保存 values，并使用 `--reset-values -f values.yaml` 升级。`--reuse-values` 保留前一 Chart 版本的默认值，这些默认值缺少自动扩缩 values（包括缩容 `behavior`），当该发布版本早于它们时。背景信息请参阅[将 Deployment 和 State