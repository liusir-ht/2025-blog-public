# CronScale Operator 说明文档

## 背景

很多业务的流量具有明显周期性，例如白天高峰需要提前扩容，夜间低峰需要自动缩容。如果完全依赖人工调整，容易出现扩容不及时、缩容遗漏和资源浪费。

`CronScale Operator` 使用 Kubernetes CRD 描述定时扩缩容规则，由 Operator 监听 `CronScale` 资源并在指定时间执行动作。除了调整 Deployment 副本数，它还支持在扩容前提前把业务镜像拉取到目标节点，降低 Pod 启动时的镜像拉取等待时间。

## 项目信息

| 项目 | 内容 |
| --- | --- |
| 项目名称 | CronScale Operator |
| Git 地址 | `https://github.com/liusir-ht/CronScale-Operator-public.git` |
| 主模块 | `liuchong.cn/m` |
| agent 模块 | `cronscale-agent` |
| CRD Group | `application.liuchong.cn` |
| CRD Version | `v1` |
| CRD Kind | `CronScale` |
| CRD ShortName | `crs` |
| 默认部署命名空间 | `cronscale-operator-system` |
| 开发框架 | Kubebuilder / controller-runtime |

拉取项目：

```bash
git clone https://github.com/liusir-ht/CronScale-Operator-public.git
cd CronScale-Operator-public
```

如果使用 SSH：

```bash
git clone git@github.com:liusir-ht/CronScale-Operator-public.git
cd CronScale-Operator-public
```

## 功能概览

| 功能 | 说明 |
| --- | --- |
| 定时扩容 | 到达 `spec.add.scaleTime` 后，将目标 Deployment 调整到 `spec.add.targetReplicas` |
| 定时缩容 | 到达 `spec.minus.scaleTime` 后，将目标 Deployment 调整到 `spec.minus.targetReplicas` |
| HPA 兼容 | 如果目标 Deployment 配置了同名 HPA，扩缩容时同步调整 HPA 的 `minReplicas` 和 `maxReplicas` |
| 镜像预热 | 到达 `spec.imagePullTime` 后，计算缺少目标镜像的节点，并创建临时 Job 执行 `crictl pull` |
| 任务清理 | 删除 `CronScale` 资源时，清理该资源注册过的 cron 任务 |

## 适用场景

这个 Operator 更适合下面几类场景：

| 场景 | 说明 |
| --- | --- |
| 业务流量有固定峰谷 | 例如每天早上提前扩容，晚上自动缩容 |
| 扩容时镜像较大 | 通过镜像预热减少 Pod 首次启动时的镜像拉取耗时 |
| 希望通过 YAML 管理规则 | 扩缩容规则可以和业务资源一起进入 GitOps 流程 |
| 部署对象是 Deployment | 当前实现以 Deployment 为目标资源 |
| 集群使用 containerd | 当前镜像预热逻辑默认访问 containerd socket |

不适合的场景：

| 场景 | 原因 |
| --- | --- |
| 流量完全不可预测 | 更适合使用 HPA、KEDA 或事件驱动扩缩容 |
| 目标资源不是 Deployment | 当前代码没有实现 StatefulSet、DaemonSet 等资源的扩缩容 |
| 节点运行时不是 containerd | 镜像预热部分需要适配实际运行时 |

## 部署模式

这个项目可以按功能拆成两种部署模式。

### 只使用定时扩缩容

如果只需要定时调整 Deployment 副本数，依赖最少，只需要：

| 依赖 | 说明 |
| --- | --- |
| Kubernetes 集群 | 运行 CRD、Controller、RBAC、Webhook |
| Operator 镜像 | 运行 controller-manager |
| 目标 Deployment | 被扩缩容的业务 Deployment |
| CronScale YAML | 声明扩容时间、缩容时间和目标副本数 |

这种模式不需要 MySQL、`cronscale-agent`、containerd socket，也不需要镜像预热 Job。

### 使用定时扩缩容 + 镜像预热

如果还要在扩容前提前拉取镜像，需要额外准备：

| 依赖 | 说明 |
| --- | --- |
| MySQL | 保存节点和镜像之间的关系 |
| cronscale-agent 镜像 | 每个节点运行一个 agent，上报节点已有镜像 |
| cronscale-task 镜像 | 临时 Job 使用，负责执行 `crictl pull` |
| containerd socket | agent 和预热 Job 访问宿主机运行时 |
| crictl 配置 | 确保 Job 能正确连接 containerd 并拉取镜像 |

推荐先跑通“只使用定时扩缩容”，确认 CRD、RBAC、Webhook 和 Deployment 扩缩容正常后，再启用镜像预热。

## 依赖说明

### 本地开发依赖

| 依赖 | 建议版本 | 用途 | 说明 |
| --- | --- | --- | --- |
| Git | 2.x+ | 拉取和提交代码 | 需要能访问 GitHub |
| Go | 1.21+ | 编译 Operator 和 agent | `go.mod` 中声明 `go 1.21` |
| Docker / Podman | 17.03+ | 构建镜像 | Makefile 默认使用 `docker`，可通过 `CONTAINER_TOOL` 覆盖 |
| kubectl | 与集群版本兼容 | 安装 CRD、部署资源、查看状态 | Makefile 默认调用 `kubectl` |
| make | 系统自带即可 | 执行构建、部署目标 | 依赖项目根目录 `Makefile` |
| kustomize | v5.3.0 | 渲染 Kubernetes YAML | Makefile 会安装到 `bin/` |
| controller-gen | v0.14.0 | 生成 CRD、RBAC、Webhook 配置 | Makefile 会安装到 `bin/` |
| golangci-lint | v1.54.2 | 代码检查 | 执行 `make lint` 时使用 |

### Kubernetes 集群依赖

| 依赖 | 建议 | 用途 |
| --- | --- | --- |
| Kubernetes | 兼容 `k8s.io/* v0.29.0` 的集群版本 | 运行 CRD、Controller、Webhook、Job、DaemonSet |
| RBAC | 开启 | Operator 需要访问 Deployment、HPA、Node、Job、CronScale 等资源 |
| Admission Webhook | 开启 | 用于校验 CronScale 创建和更新 |
| ServiceAccount | 默认由 Kustomize 创建 | Controller 在集群内访问 Kubernetes API |
| cert-manager | 可选 | 如果启用 cert-manager patch，可由 cert-manager 管理 webhook 证书 |
| Prometheus Operator | 可选 | 如果启用 `config/prometheus`，可采集 controller metrics |

### 镜像预热依赖

镜像预热不是定时扩缩容的必需能力。如果只使用定时扩缩容，可以不部署 MySQL 和 `cronscale-agent`。

| 依赖 | 是否必需 | 说明 |
| --- | --- | --- |
| MySQL | 使用镜像预热时必需 | 保存节点和镜像之间的关系 |
| cronscale-agent | 使用镜像预热时必需 | DaemonSet 方式运行在每个节点，上报节点已有镜像 |
| containerd | 使用镜像预热时必需 | agent 通过 containerd API 读取本机镜像列表 |
| `/run/containerd` | 使用镜像预热时必需 | 挂载宿主机 containerd socket |
| `/data/containerd` | 按实际集群配置 | 示例中挂载宿主机 containerd 数据目录 |
| `crictl` | 预热 Job 必需 | Job 里执行 `crictl pull <image>` |
| `/etc/crictl.yaml` | 预热 Job 按需 | 如果工具镜像不内置配置，需要从宿主机挂载 |

### Go 核心依赖

Operator 主模块的核心依赖：

| 依赖 | 版本 | 用途 |
| --- | --- | --- |
| `sigs.k8s.io/controller-runtime` | `v0.17.2` | Operator 框架，负责 manager、controller、webhook 等能力 |
| `k8s.io/api` | `v0.29.0` | Kubernetes API 类型 |
| `k8s.io/apimachinery` | `v0.29.0` | Kubernetes 元数据、runtime、schema 等基础库 |
| `k8s.io/client-go` | `v0.29.0` | 访问 Kubernetes API |
| `github.com/containerd/containerd` | `v1.7.16` | 访问 containerd 镜像信息 |
| `github.com/go-sql-driver/mysql` | `v1.8.1` | MySQL 驱动 |
| `github.com/jmoiron/sqlx` | `v1.4.0` | MySQL 访问封装 |
| `go.uber.org/zap` | `v1.26.0` | 日志 |
| `github.com/onsi/ginkgo/v2` | `v2.14.0` | 测试框架 |
| `github.com/onsi/gomega` | `v1.30.0` | 测试断言 |

agent 模块的核心依赖：

| 依赖 | 版本 | 用途 |
| --- | --- | --- |
| `github.com/containerd/containerd` | `v1.7.16` | 读取节点本地镜像 |
| `github.com/go-sql-driver/mysql` | `v1.8.1` | MySQL 驱动 |
| `github.com/jmoiron/sqlx` | `v1.4.0` | 写入节点镜像信息 |
| `github.com/robfig/cron/v3` | `v3.0.1` | agent 内部定时扫描 |
| `go.uber.org/zap` | `v1.27.0` | 日志 |

### 组件依赖关系

```text
CronScale CR
    |
    v
Controller Manager
    |
    +-- Kubernetes API: 查询/更新 Deployment、HPA、Node、Job
    |
    +-- MySQL: 查询节点镜像数据
    |
    +-- Cron: 注册扩容、缩容、镜像预热任务

cronscale-agent
    |
    +-- containerd socket: 读取节点已有镜像
    |
    +-- MySQL: 写入 nodeName 和 imageName

image-preload Job
    |
    +-- crictl: 拉取目标 Deployment 镜像
    |
    +-- containerd socket: 写入宿主机镜像缓存
```

## 架构

```text
CronScale YAML
      |
      v
Kubernetes API Server
      |
      v
CronScale Controller
      |
      +-- 注册扩容任务
      +-- 注册缩容任务
      +-- 注册镜像预热任务
      |
      v
到达 cron 时间后执行
      |
      +-- 更新 Deployment / HPA
      +-- 查询 MySQL 节点镜像数据
      +-- 创建镜像预热 Job
```

组件职责：

| 组件 | 路径 | 作用 |
| --- | --- | --- |
| CRD 类型定义 | `api/v1/cronscale_types.go` | 定义 `CronScale` 的 spec 和 status |
| Webhook 校验 | `api/v1/cronscale_webhook.go` | 校验副本数、时间字段和 Deployment 名称 |
| Controller | `internal/controller/cronscale_controller.go` | 监听资源、注册定时任务、执行扩缩容和镜像预热 |
| HPA 工具 | `pkg/hpa.go` | 查询和更新 HPA 副本上下限 |
| Kubernetes Client | `pkg/client.go` | 初始化 Kubernetes clientset |
| MySQL Client | `pkg/client.go`、`cronscale-agent/main.go` | 连接 MySQL，读写节点镜像信息 |
| cronscale-agent | `cronscale-agent/main.go` | 以 DaemonSet 方式运行，定时上报节点已有镜像 |

## 目录结构

```text
.
├── api/v1                         # CronScale API 类型和 webhook
├── cmd/main.go                    # controller-manager 入口
├── config                         # CRD、RBAC、manager、webhook、sample 等部署配置
├── cronscale-agent                # 节点镜像采集 agent
├── internal/controller            # CronScale controller 核心逻辑
├── module                         # 数据模型
├── pkg                            # Kubernetes、cron、HPA、MySQL 等工具封装
├── test                           # e2e 和测试模板
├── Dockerfile                     # Operator 镜像构建文件
├── Makefile                       # 构建、测试、部署入口
├── PROJECT                        # Kubebuilder 项目配置
└── CronScale-Operator.md          # 当前说明文档
```

## 部署前初始化参数

公开仓库不会提交真实 kubeconfig、数据库账号密码、私有镜像仓库地址或内网节点信息。部署前需要按环境补齐下面这些参数。

| 参数 | 必填 | 示例 | 配置位置 | 说明 |
| --- | --- | --- | --- | --- |
| `IMG` | 是 | `registry.example.com/cronscale-operator:v0.1.0` | `make docker-build docker-push deploy IMG=...` | Operator 镜像地址 |
| `AGENT_IMG` | 镜像预热必填 | `registry.example.com/cronscale-agent:v0.1.0` | `config/samples/Cronscale-agent.yaml` | agent DaemonSet 镜像 |
| `TASK_IMG` | 镜像预热必填 | `registry.example.com/cronscale-task:3.16` | `internal/controller/cronscale_controller.go` 或镜像预热 Job 模板 | 执行 `crictl pull` 的工具镜像 |
| `NAMESPACE` | 是 | `default` / `liuchong` | CronScale YAML、agent YAML、目标 Deployment | CronScale 和目标 Deployment 必须在同一命名空间 |
| `MYSQL_DSN` | 镜像预热必填 | `user:password@tcp(mysql:3306)/cronscale?parseTime=true` | `pkg/client.go`、`cronscale-agent/main.go` | Operator 查询镜像数据，agent 写入镜像数据 |
| `KUBECONFIG` | 本地运行必填 | `~/.kube/config` | `pkg/client.go` | 本地调试时使用；集群内运行建议使用 ServiceAccount |
| `CONTAINERD_SOCKET` | 镜像预热必填 | `/run/containerd/containerd.sock` | `pkg/client.go`、`cronscale-agent/main.go`、DaemonSet volume | 连接宿主机 containerd |
| `CONTAINERD_DATA` | 镜像预热按需 | `/data/containerd` | `config/samples/Cronscale-agent.yaml`、预热 Job volume | 挂载宿主机 containerd 数据目录 |

部署前检查清单：

| 检查项 | 命令 | 预期 |
| --- | --- | --- |
| Git 可用 | `git --version` | 能正常输出版本 |
| Go 可用 | `go version` | 版本不低于 1.21 |
| Docker 可用 | `docker version` | 能连接到 Docker daemon |
| kubectl 可用 | `kubectl version --client` | 能输出客户端版本 |
| 集群可访问 | `kubectl get node` | 能看到节点列表 |
| 当前上下文正确 | `kubectl config current-context` | 指向目标集群 |
| 镜像仓库可推送 | `docker push <image>` | 能成功推送镜像 |
| containerd 路径正确 | 在节点上确认 `/run/containerd/containerd.sock` | socket 存在 |

当前代码中需要重点替换的示例值：

| 文件 | 当前示例 | 建议 |
| --- | --- | --- |
| `pkg/client.go` | `kubeconfig := ""` | 本地调试填 kubeconfig 路径；生产建议改成集群内配置 |
| `pkg/client.go` | `dsn := ""` | 改为从 Secret / 环境变量读取 MySQL DSN |
| `cronscale-agent/main.go` | `dsn := ""` | 改为从 Secret / 环境变量读取 MySQL DSN |
| `config/samples/Cronscale-agent.yaml` | `example.com/library/cronscale-agent:1.0.2` | 替换为真实 agent 镜像 |
| `internal/controller/cronscale_controller.go` | `example.com/library/cronscale-task:3.16` | 替换为真实预热工具镜像 |
| `config/samples/*.yaml` | `namespace: liuchong` | 替换为目标业务命名空间 |

建议后续把 `dsn`、`kubeconfig`、工具镜像和 containerd 路径改成环境变量或 ConfigMap / Secret 注入，避免每个环境都重新编译镜像。

## MySQL 初始化

镜像预热能力依赖 MySQL 保存“节点 - 镜像”的关系。agent 每 5 秒扫描一次节点镜像并写入表，Operator 根据目标 Deployment 镜像查询哪些节点已经存在该镜像。

建议建表：

```sql
CREATE DATABASE IF NOT EXISTS cronscale DEFAULT CHARACTER SET utf8mb4;

CREATE TABLE IF NOT EXISTS cronscale_agent (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  node_ip VARCHAR(255) NOT NULL COMMENT '节点名称或节点标识',
  image_name VARCHAR(512) NOT NULL COMMENT '节点上已有镜像',
  update_time TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uniq_node_image (node_ip, image_name),
  KEY idx_image_name (image_name),
  KEY idx_update_time (update_time)
);
```

DSN 示例：

```text
cronscale:password@tcp(mysql.cronscale-system.svc:3306)/cronscale?charset=utf8mb4&parseTime=true&loc=Local
```

生产环境建议：

| 项目 | 建议 |
| --- | --- |
| 账号权限 | 只授予 `cronscale_agent` 表的 `SELECT`、`INSERT`、`UPDATE`、`DELETE` 权限 |
| 密码管理 | 使用 Kubernetes Secret 注入，不要写入 Git 仓库 |
| 高可用 | MySQL 地址使用稳定服务名或代理地址 |
| 数据保留 | 当前 agent 会删除 5 秒未刷新的记录，确保 agent 正常运行后再启用镜像预热 |

## 构建和部署

### 0. 拉取代码

```bash
git clone https://github.com/liusir-ht/CronScale-Operator-public.git
cd CronScale-Operator-public
```

确认本地依赖：

```bash
go version
docker version
kubectl get node
```

### 1. 构建 Operator 镜像

```bash
make docker-build docker-push IMG=registry.example.com/cronscale-operator:v0.1.0
```

参数说明：

| 参数 | 说明 |
| --- | --- |
| `IMG` | Operator 镜像完整地址 |
| `CONTAINER_TOOL` | 构建工具，默认是 `docker`，可设置为 `podman` |
| `PLATFORMS` | 使用 `make docker-buildx` 时的多架构平台列表 |

### 2. 安装 CRD

```bash
make install
```

该命令会安装 `cronscales.application.liuchong.cn`。

### 3. 部署 Operator

```bash
make deploy IMG=registry.example.com/cronscale-operator:v0.1.0
```

默认会部署到 `cronscale-operator-system` 命名空间。这个值来自 `config/default/kustomization.yaml`。

### 4. 部署 cronscale-agent

如果只需要定时扩缩容，可以跳过 agent 和 MySQL。

如果需要镜像预热，需要先构建并推送 agent 镜像。`cronscale-agent/Dockerfile` 会把已经编译好的 `cronscale-agent` 二进制打进镜像，所以需要先执行 `go build`：

```bash
cd cronscale-agent
GOOS=linux GOARCH=amd64 go build -o cronscale-agent .
docker build -t registry.example.com/cronscale-agent:v0.1.0 .
docker push registry.example.com/cronscale-agent:v0.1.0
cd ..
```

如果集群节点不是 amd64，需要把 `GOARCH` 改成实际架构，例如 `arm64`。

然后替换 `config/samples/Cronscale-agent.yaml` 中的镜像、命名空间和宿主机路径：

```yaml
containers:
  - name: cronscale-agent
    image: registry.example.com/cronscale-agent:v0.1.0
    env:
      - name: nodeName
        valueFrom:
          fieldRef:
            fieldPath: spec.nodeName
```

部署：

```bash
kubectl apply -f config/samples/Cronscale-agent.yaml
```

## CronScale 资源配置

示例：

```yaml
apiVersion: application.liuchong.cn/v1
kind: CronScale
metadata:
  name: nginx
  namespace: liuchong
  labels:
    env: test
spec:
  deploymentName: nginx
  add:
    targetReplicas: 5
    scaleTime: "0 */5 * * * ?"
  minus:
    targetReplicas: 1
    scaleTime: "0 */3 * * * ?"
  imagePullTime: "0 */1 * * * ?"
```

字段说明：

| 字段 | 是否必填 | 说明 |
| --- | --- | --- |
| `apiVersion` | 是 | 固定为 `application.liuchong.cn/v1` |
| `kind` | 是 | 固定为 `CronScale` |
| `metadata.name` | 是 | CronScale 资源名称 |
| `metadata.namespace` | 是 | 资源所在命名空间，需要和目标 Deployment 一致 |
| `spec.deploymentName` | 是 | 目标 Deployment 名称 |
| `spec.add.targetReplicas` | 是 | 扩容目标副本数，必须大于 `spec.minus.targetReplicas` |
| `spec.add.scaleTime` | 是 | 扩容时间，秒级 cron 表达式 |
| `spec.minus.targetReplicas` | 是 | 缩容目标副本数 |
| `spec.minus.scaleTime` | 是 | 缩容时间，秒级 cron 表达式 |
| `spec.imagePullTime` | 是 | 镜像预热时间，秒级 cron 表达式 |
| `spec.foo` | 否 | 示例字段，当前业务逻辑未使用 |

Webhook 当前校验规则：

| 规则 | 错误处理 |
| --- | --- |
| `add.targetReplicas` 必须大于 `minus.targetReplicas` | 拒绝创建或更新 |
| `add.scaleTime` 不能为空 | 拒绝创建或更新 |
| `add.targetReplicas` 不能为 0 | 拒绝创建或更新 |
| `minus.scaleTime` 不能为空 | 拒绝创建或更新 |
| `deploymentName` 不能为空 | 拒绝创建或更新 |
| `imagePullTime` 不能为空 | 拒绝创建或更新 |

cron 表达式使用秒级格式，示例：

| 表达式 | 含义 |
| --- | --- |
| `0 */5 * * * ?` | 每 5 分钟执行一次 |
| `0 0 9 * * ?` | 每天 09:00 执行 |
| `0 30 23 * * ?` | 每天 23:30 执行 |

## 执行逻辑

### 定时扩缩容

Operator 监听到 `CronScale` 后，会注册两个 cron 任务：

| 任务 | 来源字段 | 动作 |
| --- | --- | --- |
| 扩容任务 | `spec.add.scaleTime` | 将 Deployment 调整到 `spec.add.targetReplicas` |
| 缩容任务 | `spec.minus.scaleTime` | 将 Deployment 调整到 `spec.minus.targetReplicas` |

如果目标 Deployment 没有 HPA，Operator 直接更新 Deployment 的 Scale 子资源。

如果目标 Deployment 有同名 HPA，Operator 会先更新 HPA：

```text
targetReplicas - oldMinReplicas = delta
newMinReplicas = targetReplicas
newMaxReplicas = oldMaxReplicas + delta
```

### 镜像预热

镜像预热任务到时间后会执行下面流程：

```text
读取目标 Deployment 的第一个容器镜像
        |
        v
列出集群所有节点
        |
        v
查询 MySQL 中已有该镜像的节点
        |
        v
计算缺少镜像的节点
        |
        v
创建临时 Job 并通过 NodeAffinity 调度到这些节点
        |
        v
Job 中执行 crictl pull <deployment-image>
        |
        v
Job 完成后删除临时 Job
```

预热 Job 依赖：

| 依赖 | 说明 |
| --- | --- |
| `crictl` | 工具镜像内需要包含或挂载该命令 |
| `/run/containerd` | 访问宿主机 containerd socket |
| `/data/containerd` | 访问宿主机 containerd 数据目录 |
| `/etc/crictl.yaml` | `crictl` 运行配置 |

## 验证

查看 CRD：

```bash
kubectl get crd | grep cronscales
```

查看 Operator：

```bash
kubectl get pod -n cronscale-operator-system
kubectl logs -n cronscale-operator-system deploy/cronscale-operator-controller-manager -f
```

查看 agent：

```bash
kubectl get ds -n <namespace>
kubectl logs -n <namespace> ds/cronscale-agent -f
```

查看 CronScale：

```bash
kubectl get crs -n <namespace>
kubectl describe crs <name> -n <namespace>
```

查看扩缩容结果：

```bash
kubectl get deploy <deployment-name> -n <namespace>
kubectl get hpa <deployment-name> -n <namespace>
```

查看镜像预热 Job：

```bash
kubectl get job -n <namespace>
kubectl logs -n <namespace> job/<job-name>
```

## 常见问题

### CronScale 创建失败

| 现象 | 可能原因 | 处理方式 |
| --- | --- | --- |
| webhook 拒绝创建 | `add.targetReplicas <= minus.targetReplicas` | 调大扩容副本数或调小缩容副本数 |
| webhook 拒绝创建 | `deploymentName` 为空 | 填写目标 Deployment 名称 |
| webhook 拒绝创建 | cron 字段为空 | 填写 `add.scaleTime`、`minus.scaleTime`、`imagePullTime` |
| 无法调用 webhook | webhook Pod、Service 或证书异常 | 检查 Operator Pod、`ValidatingWebhookConfiguration` 和 webhook Service |

### 扩缩容没有生效

| 可能原因 | 处理方式 |
| --- | --- |
| CronScale 和 Deployment 不在同一命名空间 | 保持 `metadata.namespace` 与目标 Deployment namespace 一致 |
| `deploymentName` 写错 | 使用 `kubectl get deploy -n <namespace>` 确认名称 |
| cron 时间没有到达 | 临时改成更频繁的表达式验证 |
| HPA 覆盖副本数 | 查看同名 HPA 的 `minReplicas` / `maxReplicas` 是否被更新 |
| Operator 没有权限 | 检查 `config/rbac/role.yaml` 和 controller 日志 |

### 镜像预热 Job 没有创建

| 可能原因 | 处理方式 |
| --- | --- |
| 未部署 agent | 部署 `config/samples/Cronscale-agent.yaml` |
| MySQL DSN 未配置 | 配置 Operator 和 agent 的数据库连接 |
| `cronscale_agent` 表没有数据 | 查看 agent 日志和 MySQL 写入情况 |
| 节点运行时不是 containerd | 按实际运行时调整 agent 和预热 Job |
| 宿主机路径不一致 | 检查 `/run/containerd`、`/data/containerd`、`/etc/crictl.yaml` |

## 清理

```bash
kubectl delete -f config/samples/application_v1_cronscale.yaml
kubectl delete -f config/samples/Cronscale-agent.yaml
make undeploy
make uninstall
```

## 安全说明

- 不要把 kubeconfig、MySQL DSN、镜像仓库账号密码、私钥或证书提交到仓库。
- 建议使用 Kubernetes Secret 注入数据库连接串和镜像仓库凭据。
- 公开示例中的 `example.com`、空 `dsn`、空 `kubeconfig` 都需要在真实环境中替换。
- `.gitignore` 已包含 `.env`、`*.pem`、`*.key`、`*kubeconfig*` 等敏感文件模式。
