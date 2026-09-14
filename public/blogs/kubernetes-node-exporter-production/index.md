# Kubernetes 生产级部署 Node Exporter

Node Exporter 是 Prometheus 生态中用于采集 Linux 主机指标的标准组件，常用于采集 CPU、内存、磁盘、文件系统、网络、Load、TCP、VMStat 等宿主机指标。

在 Kubernetes 中，Node Exporter 最合适的部署方式是 **DaemonSet**：

```text
Kubernetes Cluster
│
├── Node-01
│   └── node-exporter
│
├── Node-02
│   └── node-exporter
│
└── Node-03
    └── node-exporter
```

保证：

```text
1 Linux Node = 1 Node Exporter Pod
```

生产环境建议直接使用 Prometheus Community 官方 Helm Chart，而不是自行维护 DaemonSet YAML。

本文采用：

```text
Helm Chart: prometheus-community/prometheus-node-exporter
Chart Version: 4.57.0
Node Exporter: 1.12.1
```

> 版本会持续更新，生产环境建议固定 Chart 版本，并在升级前进行测试。

---

## 一、整体架构

推荐的 Kubernetes 监控架构：

```text
                 上层 Prometheus
                       │
                    /federate
                       │
       ┌───────────────┼───────────────┐
       │               │               │
       ▼               ▼               ▼
 Prometheus-BJ    Prometheus-AP   Prometheus-AWS
       │               │               │
       │ K8s SD        │ K8s SD        │ K8s SD
       ▼               ▼               ▼
 node-exporter     node-exporter    node-exporter
 DaemonSet         DaemonSet        DaemonSet
       │               │               │
       ▼               ▼               ▼
 Kubernetes Node   Kubernetes Node  Kubernetes Node
```

建议：

```text
Node Exporter
    ↓
子集群 Prometheus
    ↓
Federation
    ↓
上层 Prometheus
```

不建议让上层 Prometheus 跨区域直接抓取所有 Node Exporter。

这样可以避免：

```text
跨区域网络异常
    ↓
大量 Node Exporter Target Down
    ↓
中心 Prometheus 出现大量无效告警
```

同时每个子集群仍然具有独立监控和告警能力。

---

## 二、创建 Namespace

统一将监控组件部署到：

```bash
kubectl create namespace monitoring
```

确认：

```bash
kubectl get namespace monitoring
```

---

## 三、安装 Helm

如果集群节点还没有 Helm，可以先安装 Helm。

确认版本：

```bash
helm version
```

添加 Prometheus Community Helm 仓库：

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

更新仓库：

```bash
helm repo update
```

查询 Node Exporter：

```bash
helm search repo prometheus-community/prometheus-node-exporter
```

查询所有版本：

```bash
helm search repo prometheus-community/prometheus-node-exporter --versions
```

生产环境不要直接跟随最新版升级，建议明确指定版本。

---

## 四、生产级 values.yaml

创建：

```bash
vim values-node-exporter.yaml
```

完整配置：

```yaml
image:
  registry: quay.io
  repository: prometheus/node-exporter
  tag: "v1.12.1"
  pullPolicy: IfNotPresent

service:
  enabled: true
  type: ClusterIP

  port: 9100
  targetPort: 9100
  portName: metrics

  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9100"
    prometheus.io/path: "/metrics"

hostNetwork: true

hostPID: true

hostIPC: false

hostRootFsMount:
  enabled: true
  mountPropagation: HostToContainer

nodeSelector:
  kubernetes.io/os: linux

tolerations:
  - operator: Exists

updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1

resources:
  requests:
    cpu: 20m
    memory: 32Mi

  limits:
    cpu: 200m
    memory: 128Mi

serviceAccount:
  create: true
  automountServiceAccountToken: false

securityContext:
  fsGroup: 65534
  runAsGroup: 65534
  runAsNonRoot: true
  runAsUser: 65534

containerSecurityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true

  capabilities:
    drop:
      - ALL

extraArgs:
  - --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|run/containerd/.+|var/lib/docker/.+|var/lib/kubelet/.+)($|/)
  - --collector.filesystem.fs-types-exclude=^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|iso9660|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs|erofs)$

prometheus:
  monitor:
    enabled: false

  podMonitor:
    enabled: false

revisionHistoryLimit: 5

terminationGracePeriodSeconds: 30
```

下面重点解释几个生产环境最关键的参数。

---

## 五、为什么使用 DaemonSet

Node Exporter 的目标不是采集 Pod，而是采集宿主机。

它主要采集：

```text
CPU
Memory
Load
Disk
Filesystem
Network
TCP
Kernel
VMStat
Pressure
Interrupt
Softnet
```

例如：

```text
node_cpu_seconds_total

node_memory_MemAvailable_bytes

node_load1

node_load5

node_filesystem_size_bytes

node_network_receive_bytes_total
```

因此必须保证：

```text
Node-01 -> Node Exporter
Node-02 -> Node Exporter
Node-03 -> Node Exporter
```

DaemonSet 天然适合这个场景。

当 Kubernetes 新增加 Node：

```text
Node Join
   ↓
DaemonSet Controller
   ↓
自动创建 Node Exporter Pod
```

节点删除后，对应 Pod 也会自动清理。

---

## 六、hostNetwork

配置：

```yaml
hostNetwork: true
```

Node Exporter 会直接使用宿主机网络。

例如：

```text
Node IP:

10.21.31.101
```

Node Exporter：

```text
10.21.31.101:9100
```

这样 Prometheus 可以直接通过 Node IP 抓取：

```text
http://10.21.31.101:9100/metrics
```

Node Exporter 默认端口：

```text
9100
```

---

## 七、hostPID

配置：

```yaml
hostPID: true
```

表示 Node Exporter Pod 与宿主机共享 PID Namespace。

Node Exporter 本质是：

```text
Container
   ↓
读取 Host /proc
读取 Host /sys
读取 Host filesystem
   ↓
生成主机指标
```

因此官方 Chart 默认启用了：

```yaml
hostNetwork: true
hostPID: true
hostIPC: false
```

---

## 八、挂载宿主机 RootFS

配置：

```yaml
hostRootFsMount:
  enabled: true

  mountPropagation: HostToContainer
```

宿主机：

```text
/
```

会挂载到 Node Exporter 容器：

```text
/host/root
```

Node Exporter 才能正确识别真实宿主机文件系统，而不是只看到容器自己的 RootFS。

---

## 九、为什么要过滤 Kubernetes 文件系统

这是 Kubernetes 部署 Node Exporter 非常重要的一项优化。

如果不做过滤，一个 Kubernetes Node 上可能存在大量：

```text
overlay
tmpfs
shm
proc
sysfs
cgroup
```

以及：

```text
/var/lib/kubelet/pods/...
/var/lib/docker/overlay2/...
/run/containerd/...
```

Node Exporter 会为这些挂载点生成大量：

```text
node_filesystem_size_bytes
node_filesystem_avail_bytes
node_filesystem_free_bytes
node_filesystem_files
node_filesystem_files_free
```

假设：

```text
200 Mount Points / Node
×
20 filesystem metrics
×
500 Nodes
```

就可能产生大量没有实际监控价值的 Time Series。

因此增加：

```yaml
extraArgs:
  - --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|run/containerd/.+|var/lib/docker/.+|var/lib/kubelet/.+)($|/)
```

排除：

```text
/dev
/proc
/sys
/run/containerd
/var/lib/docker
/var/lib/kubelet
```

同时排除虚拟文件系统：

```yaml
- --collector.filesystem.fs-types-exclude=^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|iso9660|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs|erofs)$
```

最终真正重点关注：

```text
/
/boot
/data
/data1
/data2
/var
```

等真实磁盘文件系统。

---

## 十、Tolerations

生产 Kubernetes 中，Control Plane、GPU Node、Infra Node 经常存在 Taint。

例如：

```bash
kubectl describe node node-01 | grep -i taint
```

可能看到：

```text
node-role.kubernetes.io/control-plane:NoSchedule
```

或者：

```text
gpu=true:NoSchedule
```

如果 Node Exporter 没有对应 Toleration，这些节点就不会部署 Node Exporter。

因此配置：

```yaml
tolerations:
  - operator: Exists
```

表示允许 Node Exporter 调度到存在 Taint 的节点。

Node Exporter 属于 Node 基础监控组件，一般应该覆盖所有 Linux Node。

---

## 十一、NodeSelector

配置：

```yaml
nodeSelector:
  kubernetes.io/os: linux
```

避免调度到 Windows Node。

如果集群存在混合架构：

```text
amd64
arm64
```

通常无需限制 architecture。

如果确实只想部署 AMD64：

```yaml
nodeSelector:
  kubernetes.io/os: linux
  kubernetes.io/arch: amd64
```

---

## 十二、资源限制

Node Exporter 自身资源消耗通常比较低。

建议初始配置：

```yaml
resources:
  requests:
    cpu: 20m
    memory: 32Mi

  limits:
    cpu: 200m
    memory: 128Mi
```

生产环境建议后续根据实际：

```text
container_cpu_usage_seconds_total

container_memory_working_set_bytes
```

进行调整。

不要一开始给过大的 Request，因为：

```text
Node 数量 × Request
```

会直接影响整个集群的可调度资源统计。

例如 1000 个节点：

```text
1000 × 100m CPU
=
100 Core Request
```

因此基础 DaemonSet 的资源 Request 应尽量合理。

---

## 十三、安全配置

Node Exporter 不需要 Root 用户运行。

配置：

```yaml
securityContext:
  fsGroup: 65534
  runAsGroup: 65534
  runAsNonRoot: true
  runAsUser: 65534
```

Container：

```yaml
containerSecurityContext:
  allowPrivilegeEscalation: false

  readOnlyRootFilesystem: true

  capabilities:
    drop:
      - ALL
```

实现：

```text
Non Root
+
Readonly RootFS
+
No Privilege Escalation
+
Drop Linux Capabilities
```

同时：

```yaml
serviceAccount:
  automountServiceAccountToken: false
```

Node Exporter 自身不需要调用 Kubernetes API，因此没必要把 ServiceAccount Token 挂载进 Pod。

---

## 十四、RollingUpdate

配置：

```yaml
updateStrategy:
  type: RollingUpdate

  rollingUpdate:
    maxUnavailable: 1
```

升级 Node Exporter 时：

```text
Node-01
旧 Pod -> 删除 -> 新 Pod

Node-02
旧 Pod -> 删除 -> 新 Pod

Node-03
旧 Pod -> 删除 -> 新 Pod
```

避免大批 Node Exporter 同时下线。

生产环境建议：

```text
maxUnavailable: 1
```

---

## 十五、安装 Node Exporter

执行：

```bash
helm upgrade --install node-exporter \
  prometheus-community/prometheus-node-exporter \
  --namespace monitoring \
  --version 4.57.0 \
  -f values-node-exporter.yaml
```

查看 Release：

```bash
helm list -n monitoring
```

查看：

```bash
helm status node-exporter -n monitoring
```

---

## 十六、验证 DaemonSet

查看：

```bash
kubectl get daemonset -n monitoring
```

正常情况：

```text
NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
node-exporter   20        20        20      20           20
```

重点：

```text
DESIRED
CURRENT
READY
AVAILABLE
```

应该基本一致。

如果集群有：

```text
20 个 Linux Node
```

那么应该：

```text
DESIRED = 20
READY   = 20
```

---

## 十七、查看 Pod 分布

```bash
kubectl get pod \
  -n monitoring \
  -l app.kubernetes.io/name=prometheus-node-exporter \
  -o wide
```

例如：

```text
NAME                         NODE       IP
node-exporter-2qsjh          node-01    10.21.31.101
node-exporter-6sh2p          node-02    10.21.31.102
node-exporter-x91df          node-03    10.21.31.103
```

确认每个 Node 都有一个 Node Exporter。

可以查看：

```bash
kubectl get pod -n monitoring -o wide
```

同时查看所有 Node：

```bash
kubectl get node -o wide
```

进行对比。

---

## 十八、检查 9100 端口

Node 上检查：

```bash
ss -lntp | grep 9100
```

测试：

```bash
curl http://127.0.0.1:9100/metrics
```

或者：

```bash
curl http://NODE_IP:9100/metrics
```

例如：

```bash
curl http://10.21.31.101:9100/metrics
```

正常会返回：

```text
# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
# TYPE node_cpu_seconds_total counter

node_cpu_seconds_total{cpu="0",mode="idle"} ...
```

---

## 十九、检查常用指标

CPU：

```text
node_cpu_seconds_total
```

Memory：

```text
node_memory_MemTotal_bytes
node_memory_MemAvailable_bytes
node_memory_MemFree_bytes
```

Load：

```text
node_load1
node_load5
node_load15
```

Filesystem：

```text
node_filesystem_size_bytes
node_filesystem_avail_bytes
node_filesystem_free_bytes
```

Disk：

```text
node_disk_read_bytes_total
node_disk_written_bytes_total
node_disk_io_time_seconds_total
```

Network：

```text
node_network_receive_bytes_total
node_network_transmit_bytes_total
node_network_receive_errs_total
node_network_transmit_errs_total
```

TCP：

```text
node_netstat_Tcp_CurrEstab
```

VMStat：

```text
node_vmstat_pgmajfault
node_vmstat_pswpin
node_vmstat_pswpout
```

---

## 二十、纯 Prometheus 如何发现 Node Exporter

如果使用的是：

```text
Prometheus Operator
```

通常使用：

```text
ServiceMonitor
PodMonitor
```

但如果部署的是纯 Prometheus：

```text
prometheus-server
```

Prometheus 本身并不识别 ServiceMonitor CRD。

因此推荐使用：

```text
Kubernetes Service Discovery
```

例如：

```yaml
scrape_configs:

  - job_name: node-exporter

    scrape_interval: 15s
    scrape_timeout: 10s

    kubernetes_sd_configs:
      - role: endpoints

        namespaces:
          names:
            - monitoring

    relabel_configs:

      - source_labels:
          - __meta_kubernetes_service_name

        regex: node-exporter-prometheus-node-exporter

        action: keep

      - source_labels:
          - __meta_kubernetes_endpoint_address_target_kind

        regex: Pod

        action: keep

      - source_labels:
          - __meta_kubernetes_pod_node_name

        target_label: instance

      - source_labels:
          - __meta_kubernetes_pod_node_name

        target_label: node
```

实际 Service 名称需要先确认：

```bash
kubectl get svc -n monitoring
```

例如：

```text
node-exporter-prometheus-node-exporter
```

---

## 二十一、instance 标签建议

Prometheus 默认可能得到：

```text
instance="10.21.31.101:9100"
```

但对于 Kubernetes Node 监控，更推荐：

```text
instance="node-01"
```

因此：

```yaml
- source_labels:
    - __meta_kubernetes_pod_node_name

  target_label: instance
```

最终：

```text
node_cpu_seconds_total{
  instance="node-01",
  job="node-exporter"
}
```

可读性更高。

Grafana：

```text
instance=node-01
```

Alertmanager：

```text
instance=node-01
```

排查问题也会更加直观。

---

## 二十二、Cluster 标签

如果存在多 Kubernetes 集群，强烈建议给每个 Prometheus 配置：

```yaml
global:

  external_labels:
    cluster: bj-prod
    idc: tencent-bj
    env: prod
```

其他集群：

```text
cluster=ap-prod

cluster=aws-us-prod

cluster=aws-sg-prod
```

最终时序：

```text
node_cpu_seconds_total{
  cluster="bj-prod",
  idc="tencent-bj",
  env="prod",
  instance="node-01",
  job="node-exporter"
}
```

这样 Federation 或 Thanos 场景都更加容易区分来源。

---

## 二十三、Scrape Interval

Node Exporter 推荐：

```yaml
scrape_interval: 15s
scrape_timeout: 10s
```

对于普通生产 Kubernetes：

```text
15s
```

基本足够。

如果节点规模：

```text
1000+
```

可以评估：

```text
30s
```

降低：

```text
Prometheus Scrape QPS
Network
Samples Ingest Rate
TSDB 压力
```

不建议 Node 级监控使用：

```text
1s
```

除非有明确的高频监控需求。

---

## 二十四、9100 安全问题

因为：

```yaml
hostNetwork: true
```

Node Exporter 实际监听：

```text
NodeIP:9100
```

因此安全组不要：

```text
0.0.0.0/0
    ↓
TCP 9100
```

建议：

```text
Prometheus Network
       ↓
 Kubernetes Node:9100
```

只允许监控网络访问。

Node Exporter 暴露的信息包括：

```text
hostname
kernel
CPU
Memory
Filesystem
Disk
Network Interface
OS
Mount Point
```

虽然一般不包含业务数据，但仍属于基础设施信息，不应该直接暴露公网。

---

## 二十五、NetworkPolicy

如果集群使用支持 NetworkPolicy 的 CNI，可以进一步限制访问来源。

需要注意：

```text
hostNetwork Pod
```

在不同 CNI 下对 NetworkPolicy 的处理可能不同。

因此不能只依赖 NetworkPolicy。

更可靠的是：

```text
Cloud Security Group
+
Host Firewall
+
NetworkPolicy
```

共同限制 9100。

---

## 二十六、Prometheus Operator 场景

如果未来切换：

```text
kube-prometheus-stack
```

可以开启：

```yaml
prometheus:

  monitor:
    enabled: true

    interval: 15s

    scrapeTimeout: 10s
```

由 Prometheus Operator 创建：

```text
ServiceMonitor
```

如果节点规模特别大，例如：

```text
1000+ Node Exporter endpoints
```

还可以评估：

```yaml
prometheus:
  podMonitor:
    enabled: true
```

官方 Chart 也特别说明，在非常大的 Node Exporter endpoint 数量下，一些环境可能更适合使用 PodMonitor。

---

## 二十七、检查 Target

Prometheus 页面：

```text
Status
  ↓
Targets
```

应该看到：

```text
node-exporter

UP
```

PromQL：

```promql
up{job="node-exporter"}
```

正常：

```text
1
```

检查 Down：

```promql
up{job="node-exporter"} == 0
```

统计节点：

```promql
count(up{job="node-exporter"})
```

---

## 二十八、基础告警

### Node Exporter Down

```yaml
- alert: NodeExporterDown

  expr: |
    up{job="node-exporter"} == 0

  for: 2m

  labels:
    severity: critical

  annotations:
    summary: "Node Exporter Down"

    detail: "Node Exporter {{ $labels.instance }} has been down for more than 2 minutes."
```

---

### CPU 使用率

```promql
100 - (
  avg by (instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  ) * 100
)
```

告警：

```yaml
- alert: NodeCPUHigh

  expr: |
    100 -
    (
      avg by (instance) (
        rate(node_cpu_seconds_total{mode="idle"}[5m])
      ) * 100
    ) > 85

  for: 10m

  labels:
    severity: warning

  annotations:
    summary: "Node CPU usage is high"

    detail: "{{ $labels.instance }} CPU usage has exceeded 85% for 10 minutes."
```

---

## 二十九、内存使用率

推荐使用：

```text
MemAvailable
```

而不是：

```text
MemFree
```

PromQL：

```promql
(
  1 -
  (
    node_memory_MemAvailable_bytes
    /
    node_memory_MemTotal_bytes
  )
) * 100
```

---

## 三十、磁盘使用率

```promql
(
  1 -
  (
    node_filesystem_avail_bytes{
      fstype!~"tmpfs|overlay"
    }
    /
    node_filesystem_size_bytes{
      fstype!~"tmpfs|overlay"
    }
  )
) * 100
```

生产环境通常：

```text
> 80% Warning

> 90% Critical
```

还应该同时监控 inode：

```promql
(
  1 -
  (
    node_filesystem_files_free
    /
    node_filesystem_files
  )
) * 100
```

否则可能出现：

```text
Disk Capacity 还有空间
但是 inode 已经耗尽
```

---

## 三十一、升级

查看当前版本：

```bash
helm list -n monitoring
```

查看可用版本：

```bash
helm search repo prometheus-community/prometheus-node-exporter --versions
```

升级前：

```bash
helm diff upgrade node-exporter \
  prometheus-community/prometheus-node-exporter \
  -n monitoring \
  -f values-node-exporter.yaml
```

如果没有 helm-diff，可以先：

```bash
helm plugin install https://github.com/databus23/helm-diff
```

升级：

```bash
helm upgrade node-exporter \
  prometheus-community/prometheus-node-exporter \
  --namespace monitoring \
  --version <NEW_CHART_VERSION> \
  -f values-node-exporter.yaml
```

观察：

```bash
kubectl rollout status daemonset \
  -n monitoring \
  -l app.kubernetes.io/name=prometheus-node-exporter
```

---

## 三十二、回滚

查看历史：

```bash
helm history node-exporter -n monitoring
```

例如：

```text
REVISION
1
2
3
```

回滚：

```bash
helm rollback node-exporter 2 -n monitoring
```

检查：

```bash
kubectl get pod -n monitoring -o wide
```

---

## 三十三、卸载

```bash
helm uninstall node-exporter -n monitoring
```

检查：

```bash
kubectl get all -n monitoring
```

---

## 三十四、常见问题

### 1. 部分 Node 没有 Node Exporter

检查：

```bash
kubectl get node
```

查看 Taint：

```bash
kubectl describe node NODE_NAME | grep -i taint
```

查看 DaemonSet：

```bash
kubectl describe daemonset node-exporter -n monitoring
```

常见原因：

```text
Taint / Toleration

NodeSelector

NodeAffinity

资源不足

PodSecurity
```

---

### 2. Prometheus Target Down

测试：

```bash
curl http://NODE_IP:9100/metrics
```

如果不通：

```bash
ss -lntp | grep 9100
```

再检查：

```text
Security Group
iptables/nftables
NetworkPolicy
CNI
Routing
```

---

### 3. filesystem 指标太多

查询：

```promql
count by (instance) (
  node_filesystem_size_bytes
)
```

如果非常多，检查：

```text
mount-points-exclude
fs-types-exclude
```

是否生效。

查看 Node Exporter 参数：

```bash
kubectl get pod \
  -n monitoring \
  -l app.kubernetes.io/name=prometheus-node-exporter \
  -o yaml
```

---

### 4. Node Exporter CPU 占用异常

检查：

```bash
kubectl top pod -n monitoring
```

重点排查 Collector：

```text
filesystem
diskstats
netdev
systemd
processes
slabinfo
```

不要随意打开高成本 Collector。

Node Exporter 默认 Collector 一般已经足够覆盖生产基础监控。

---

## 三十五、生产环境最终建议

生产环境 Node Exporter 建议至少满足：

```text
DaemonSet 部署

Linux Node 全覆盖

hostNetwork

hostPID

Host RootFS Mount

Filesystem Filter

Tolerations

Resource Requests / Limits

Non Root

Readonly RootFS

Disable ServiceAccount Token

RollingUpdate

固定 Helm Chart Version

限制 TCP 9100 来源

Prometheus Kubernetes Service Discovery

统一 instance=nodeName

增加 cluster / idc / env 标签

15s Scrape Interval

Node Exporter Down 告警
```

完整链路建议：

```text
Kubernetes Node
      │
      ▼
Node Exporter
      │
      │ :9100
      ▼
子集群 Prometheus
      │
      ├── Recording Rules
      │
      ├── Alert Rules
      │
      └── Local Alertmanager
      │
      ▼
Federation / Thanos
      │
      ▼
中心监控
```

Node Exporter 解决的是：

```text
Host Metrics
```

完整 Kubernetes 基础监控通常还需要：

```text
Node Exporter
+
kube-state-metrics
+
Kubelet / cAdvisor
+
Prometheus
+
Alertmanager
+
Grafana
```

其中：

```text
Node Exporter
    -> Node / OS

kube-state-metrics
    -> Kubernetes Object State

Kubelet / cAdvisor
    -> Pod / Container Resource

Prometheus
    -> Collection + TSDB + PromQL

Alertmanager
    -> Alert Routing

Grafana
    -> Visualization
```

这样才能组成完整的 Kubernetes 基础可观测体系。

---

## 参考

- Prometheus Node Exporter  
  https://github.com/prometheus/node_exporter

- Prometheus Community Helm Charts  
  https://github.com/prometheus-community/helm-charts

- prometheus-node-exporter Helm Chart  
  https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-node-exporter

