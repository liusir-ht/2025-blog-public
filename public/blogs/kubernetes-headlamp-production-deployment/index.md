# Kubernetes 生产环境部署 Headlamp 集群管理界面

## 1. 背景

Kubernetes 日常运维通常依赖 `kubectl`，但在查看工作负载、事件、日志、资源关系和 RBAC 权限时，Web UI 更直观。

Headlamp 是 Kubernetes SIG 维护的 Web UI，支持：

- 查看 Pod、Deployment、StatefulSet、Service、Ingress 等资源；
- 查看 Pod 日志、事件和资源 YAML；
- 使用 Kubernetes RBAC 控制用户权限；
- 使用 ServiceAccount Token 或 OIDC 登录；
- 通过 Helm 部署到 Kubernetes 集群；
- 根据登录身份显示允许访问的资源。

本文以 Kubernetes 1.32、Helm 3 和 Headlamp Helm Chart `0.45.0` 为例，通过独立的 `headlamp` Namespace 部署。

整体链路：

```text
浏览器
  ↓ HTTPS
Ingress / LoadBalancer
  ↓
Headlamp Service
  ↓
Headlamp Pod
  ↓ 携带登录用户的 Token
Kubernetes API Server
  ↓
RBAC 鉴权
```

Headlamp 只负责提供管理界面，用户最终能看到或操作哪些资源，仍然由 Kubernetes RBAC 决定。

## 2. 核心 values.yaml

生产环境建议不要让 Headlamp Pod 自身拥有 `cluster-admin` 权限。关闭 Chart 默认的 ClusterRoleBinding，由独立的登录 ServiceAccount 承载用户权限。

```yaml
replicaCount: 2

image:
  #国内镜像   registry: m.daocloud.io/ghcr.io
  registry: ghcr.io
  repository: headlamp-k8s/headlamp
  pullPolicy: IfNotPresent
  # 留空时使用 Helm Chart 对应的 AppVersion
  tag: ""

config:
  inCluster: true
  baseURL: ""
  sessionTTL: 86400
  unsafeUseServiceAccountToken: false
  enableHelm: false
  staticPlugins:
    enabled: true

serviceAccount:
  create: true
  name: headlamp

# 不给 Headlamp Pod 自动绑定 cluster-admin
clusterRoleBinding:
  create: false

automountServiceAccountToken: true

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

securityContext:
  runAsNonRoot: true
  runAsUser: 100
  runAsGroup: 101
  privileged: false
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL

podDisruptionBudget:
  enabled: true
  minAvailable: 1

affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          topologyKey: kubernetes.io/hostname
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: headlamp
```
### headlamp 域名访问配置
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: headlamp
  namespace: headlamp
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.0.0/16"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - xxx.example.com
      secretName: https-2026-2027
  rules:
    - host: xxx.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: headlamp
                port:
                  number: 80
```

## 3. 安装 Headlamp

添加 Headlamp Helm Repository：

```bash
helm repo add headlamp https://kubernetes-sigs.github.io/headlamp/
```

更新 Helm Repository：

```bash
helm repo update
```

查看可用版本：

```bash
helm search repo headlamp/headlamp --versions
```

查看 Chart 信息：

```bash
helm show chart headlamp/headlamp --version 0.45.0
```

查看 Chart 默认配置：

```bash
helm show values headlamp/headlamp --version 0.45.0
```

安装或升级：

```bash
helm upgrade --install headlamp \
  headlamp/headlamp \
  --namespace headlamp \
  --create-namespace \
  --version 0.45.0 \
  -f values.yaml \
  --wait \
  --timeout 10m
```

检查 Helm Release：

```bash
helm -n headlamp list
```

检查工作负载：

```bash
kubectl -n headlamp get deploy,pod,svc,ingress -o wide
```

如果没有配置 Ingress，临时访问：

```bash
kubectl -n headlamp port-forward svc/headlamp 8080:80
```

浏览器打开：

```text
http://127.0.0.1:8080
```

## 4. 核心配置说明

### 镜像版本

```yaml
image:
  tag: ""
```

留空时，镜像版本跟随 Helm Chart 的 `AppVersion`，可以减少 Chart 已升级但容器仍使用旧版本的问题。

生产环境应固定 Chart 版本：

```bash
--version 0.45.0
```

升级前先检查新 Chart 的默认值和变更内容，不建议长期直接使用未固定版本的安装命令。

### Headlamp Pod 权限

```yaml
clusterRoleBinding:
  create: false
```

Headlamp Chart `0.45.0` 的该选项默认会创建 ClusterRoleBinding，并引用 `cluster-admin`。生产环境建议显式关闭。

Headlamp Pod 使用：

```text
system:serviceaccount:headlamp:headlamp
```

登录用户使用另外创建的 ServiceAccount Token。两者不是同一个身份，也不应该共用权限。

### 禁止共享 Pod ServiceAccount 权限

```yaml
config:
  unsafeUseServiceAccountToken: false
```

该配置保持关闭时，Headlamp 使用登录用户提交的 Token 访问 Kubernetes API。每个用户的操作按自己的 RBAC 权限判断。

如果开启，所有用户可能共享 Headlamp Pod 的 ServiceAccount 身份，不利于权限隔离和审计。

### Helm 操作能力

```yaml
config:
  enableHelm: false
```

只需要查看 Kubernetes 资源时建议保持关闭，避免在 Headlamp 中直接执行 Helm 安装、升级和卸载操作。

### 高可用

```yaml
replicaCount: 2
```

配合 Pod 反亲和性和 PDB，让两个副本尽量分散到不同节点，并保证节点维护期间至少保留一个可用副本。

如果集群只有一个可调度节点，可以将副本数调整为 `1`，同时关闭 PDB，避免节点维护时影响驱逐。

## 5. 登录账号与 RBAC

### 管理员临时 Token

创建独立的管理员 ServiceAccount：

```bash
kubectl create serviceaccount headlamp-admin -n headlamp
```

授予集群管理员权限：

```bash
kubectl create clusterrolebinding headlamp-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=headlamp:headlamp-admin
```

生成 2 小时临时 Token：

```bash
kubectl create token headlamp-admin \
  -n headlamp \
  --duration=2h
```

输出内容就是 Headlamp 登录 Token。再次生成不会覆盖旧 Token，每个 Token 按自己的签发时间计算有效期。

### 管理员长期 Token

只有确实无法使用短期 Token 或 OIDC 时，才建议创建长期 Token：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: headlamp-admin-token
  namespace: headlamp
  annotations:
    kubernetes.io/service-account.name: headlamp-admin
type: kubernetes.io/service-account-token
```

应用：

```bash
kubectl apply -f headlamp-admin-token.yaml
```

获取 Token：

```bash
kubectl get secret headlamp-admin-token \
  -n headlamp \
  -o jsonpath='{.data.token}' | base64 -d; echo
```

这个 Token 通常没有普通临时 Token 的 `exp` 字段，会持续有效，直到删除 Secret、删除 ServiceAccount、轮换签名密钥或重建集群。

### 业务命名空间只读账号

创建只读登录账号：

```bash
kubectl create serviceaccount developer -n headlamp
```

只允许查看 `business-test`：

```bash
kubectl create rolebinding developer-view \
  --clusterrole=view \
  --serviceaccount=headlamp:developer \
  --namespace=business-test
```

这里的命名空间含义是：

| 配置 | 作用 |
|---|---|
| `ServiceAccount: headlamp/developer` | 登录身份所在位置 |
| `RoleBinding: business-test/developer-view` | 权限生效的业务命名空间 |
| `ClusterRole: view` | 被复用的只读权限规则 |

Token 跟着 ServiceAccount 创建：

```bash
kubectl create token developer \
  -n headlamp \
  --duration=2h
```

RoleBinding 跟着需要访问的业务 Namespace 创建。需要访问多个 Namespace 时，应分别创建多个 RoleBinding。

### 允许 Headlamp 列出 Namespace

只创建业务 Namespace 内的 RoleBinding 时，用户可以读取该 Namespace 的资源，但没有集群级的 `list namespaces` 权限。Headlamp 页面可能无法自动列出 Namespace。

可以额外创建最小权限：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: headlamp-namespace-reader
rules:
  - apiGroups: [""]
    resources:
      - namespaces
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developer-headlamp-namespace-reader
subjects:
  - kind: ServiceAccount
    name: developer
    namespace: headlamp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: headlamp-namespace-reader
```

应用：

```bash
kubectl apply -f headlamp-namespace-reader.yaml
```

用户会看到所有 Namespace 的名称，但仍然只能读取已经通过 RoleBinding 授权的业务 Namespace 内容。

## 6. RBAC 生效逻辑

多个 RoleBinding、ClusterRoleBinding 的权限会叠加并取并集：

```text
最终权限 = RoleBinding A ∪ RoleBinding B ∪ ClusterRoleBinding
```

Kubernetes 原生 RBAC 只有 `Allow`，没有 `Deny`：

- 后创建的 RoleBinding 不会覆盖先创建的 RoleBinding；
- 只读 RoleBinding 不能抵消另一个绑定授予的写权限；
- RoleBinding 只在它所在的 Namespace 生效；
- ClusterRole 可以被 RoleBinding 引用，但权限仍限制在 RoleBinding 所在的 Namespace；
- 集群级资源必须通过 ClusterRoleBinding 授权。

对应关系：

| 角色 | 绑定方式 | 最终范围 |
|---|---|---|
| Role | RoleBinding | RoleBinding 所在 Namespace |
| ClusterRole | RoleBinding | RoleBinding 所在 Namespace |
| ClusterRole | ClusterRoleBinding | 整个集群 |
| Role | ClusterRoleBinding | 不允许 |

## 7. Ingress 与访问安全

生产环境建议：

- 使用 HTTPS 暴露 Headlamp；
- 仅允许办公网、VPN 或可信来源访问；
- 不要直接通过公网暴露无保护的 Headlamp；
- 不要使用 Basic Auth 替代 Kubernetes 身份和 RBAC；
- 用户较多时优先接入 OIDC；
- 管理员优先使用短期 Token；
- 每个用户使用独立 ServiceAccount，避免共享 Token；
- 不要把 Token 写入 Git、脚本、Jenkins 日志或截图。

Headlamp 可能将用户 Token 保存在浏览器本地存储中，因此管理员不要在公共电脑或不可信浏览器上登录。

如果 Headlamp 运行在子路径，例如：

```text
https://example.com/headlamp/
```

需要同时配置：

```yaml
config:
  baseURL: /headlamp
```

并保证 Ingress 路径和健康检查路径与 `baseURL` 一致。使用独立域名时建议直接使用 `/`，配置更简单。

## 8. 验证

### 检查 Helm 和工作负载

```bash
helm -n headlamp status headlamp
```

```bash
kubectl -n headlamp get deploy,pod,svc,ingress
```

```bash
kubectl -n headlamp rollout status deploy/headlamp
```

### 查看日志

```bash
kubectl -n headlamp logs deploy/headlamp --tail=200
```

### 验证管理员权限

```bash
kubectl auth can-i '*' '*' \
  --as=system:serviceaccount:headlamp:headlamp-admin
```

预期：

```text
yes
```

### 验证业务只读权限

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:headlamp:developer \
  -n business-test
```

预期：

```text
yes
```

验证不能删除 Pod：

```bash
kubectl auth can-i delete pods \
  --as=system:serviceaccount:headlamp:developer \
  -n business-test
```

预期：

```text
no
```

验证不能读取其他 Namespace：

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:headlamp:developer \
  -n business-prod
```

如果没有授权，预期：

```text
no
```

### 查看账号最终权限

```bash
kubectl auth can-i --list \
  --as=system:serviceaccount:headlamp:developer \
  -n business-test
```

### 使用真实 Token 验证

```bash
DEVELOPER_TOKEN=$(kubectl create token developer \
  -n headlamp \
  --duration=2h)
```

```bash
kubectl --token="$DEVELOPER_TOKEN" \
  auth can-i list pods \
  -n business-test
```

`kubectl auth can-i --as` 用于验证 RBAC 配置，`kubectl --token` 可以同时验证 Token 和 RBAC 的完整链路。

## 9. 常见问题

### 登录后什么都看不到

优先检查：

```bash
kubectl auth can-i list namespaces \
  --as=system:serviceaccount:headlamp:developer
```

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:headlamp:developer \
  -n business-test
```

如果第一个是 `no`、第二个是 `yes`，说明业务权限正常，但 Headlamp 无法自动列出 Namespace。可以在 Headlamp 中配置可访问 Namespace，或者增加前面的 `headlamp-namespace-reader` 权限。

### `kubectl create token` 再次执行会不会覆盖旧 Token

不会。每次调用 TokenRequest API 都会签发新的 Token，旧 Token 在过期前仍然有效。

### ServiceAccount 显示 `Tokens: <none>`

Kubernetes 1.24 以后，`kubectl create token` 生成的短期 Token 不会保存成 Secret，因此这是正常现象。

### Ingress 返回 404

检查：

```bash
kubectl -n headlamp get ingress headlamp -o yaml
```

重点确认：

```text
ingressClassName
host
path
service name
service port
```

### 页面可以打开但请求返回 403

这通常不是 Headlamp 故障，而是登录 Token 对应的 RBAC 权限不足。使用 `kubectl auth can-i` 验证具体资源和动作。

## 10. 升级与回滚

升级前备份当前 values：

```bash
helm -n headlamp get values headlamp -o yaml > headlamp-values-backup.yaml
```

更新仓库并查看版本：

```bash
helm repo update
```

```bash
helm search repo headlamp/headlamp --versions
```

升级到指定版本：

```bash
helm upgrade headlamp \
  headlamp/headlamp \
  --namespace headlamp \
  --version <CHART_VERSION> \
  -f values.yaml \
  --wait \
  --timeout 10m
```

查看历史版本：

```bash
helm -n headlamp history headlamp
```

回滚：

```bash
helm -n headlamp rollback headlamp <REVISION> --wait
```

## 11. 生产注意点

### 不要让 Headlamp Pod 默认拥有 cluster-admin

核心配置：

```yaml
clusterRoleBinding:
  create: false
```

管理员权限应该绑定到独立的登录 ServiceAccount，而不是 Headlamp Deployment 使用的 ServiceAccount。

### 长期 Token 不等于绝对永久

长期 Token Secret 通常没有固定过期时间，但以下操作仍会使其失效：

- 删除 Token Secret；
- 删除对应 ServiceAccount；
- 更换 ServiceAccount 签名密钥；
- 修改 API Server 认证配置；
- 重建集群。

### 删除长期管理员 Token

只撤销长期 Token：

```bash
kubectl delete secret headlamp-admin-token -n headlamp
```

撤销管理员权限：

```bash
kubectl delete clusterrolebinding headlamp-admin
```

删除前应确认是否还有其他管理员登录方式，避免失去集群管理入口。

### 优先使用 OIDC

当使用人数较多时，推荐接入企业 OIDC：

- 用户离职后可统一停用账号；
- 无需长期分发 ServiceAccount Token；
- 可以通过用户或用户组绑定 RBAC；
- 审计日志中的身份更清晰。

少量内部管理员可以使用短期 ServiceAccount Token；长期管理员 Token 只适合作为严格管控的应急方式。

## 总结

这套配置主要解决：

```text
Helm 部署 Headlamp
        ↓
关闭 Pod 默认 cluster-admin
        ↓
独立 ServiceAccount 登录
        ↓
RoleBinding / ClusterRoleBinding 控制权限
        ↓
Ingress HTTPS 提供访问入口
```

最终形成：

```text
用户 Token → Headlamp → Kubernetes API Server → RBAC
```

的 Kubernetes 可视化管理体系。

生产环境的关键原则是：Headlamp 只提供界面，权限必须由 Kubernetes RBAC 控制；普通用户使用 Namespace 级 RoleBinding，管理员使用独立账号，并优先选择短期 Token 或 OIDC。
