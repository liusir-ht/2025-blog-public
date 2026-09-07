


## 1. Pod 的创建流程

Pod 创建是 Kubernetes 中最核心的一条链路之一。

执行：

```bash
kubectl apply -f nginx.yaml
```

整体流程可以概括为：

```text
kubectl
   |
   v
kube-apiserver
   |
   +-- Authentication
   +-- Authorization
   +-- Admission
   |
   v
etcd
   |
   v
Scheduler
   |
   | 选择 Node
   v
kubelet
   |
   +-- 创建 PodSandbox
   +-- 配置网络
   +-- 拉取镜像
   +-- 创建 Container
   +-- 启动 Container
   |
   v
Container Runtime
```

### 1.1 API Server 接收请求

客户端首先访问 kube-apiserver。

请求通常会经过：

```text
Authentication
        |
Authorization
        |
Admission
        |
Validation
        |
etcd
```

API Server 本身并不负责选择 Node，它更像 Kubernetes 的统一控制入口。

### 1.2 写入 etcd

Pod 对象经过校验后写入 etcd。

此时 Pod 已经存在于 Kubernetes 的期望状态中，但是还没有真正运行。

例如：

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:latest
```

但是：

```yaml
spec:
  nodeName: ""
```

说明 Pod 还没有完成调度。

### 1.3 Scheduler 进行调度

Scheduler 通过 Watch/Informer 发现未调度 Pod，然后根据：

- CPU / Memory
- Resource Requests
- Taints / Tolerations
- Node Affinity
- Pod Affinity / Anti-Affinity
- Topology Spread
- GPU 等扩展资源

选择合适的 Node。

最终更新：

```yaml
spec:
  nodeName: node-01
```

### 1.4 kubelet 执行 Pod

node-01 上的 kubelet 通过 Watch API Server 发现：

```text
Pod -> node-01
```

然后开始真正创建 Pod。

### 1.5 创建 PodSandbox

kubelet 通过 CRI 调用 Container Runtime：

```text
kubelet
   |
   v
CRI
   |
   v
containerd
   |
   v
RunPodSandbox
```

PodSandbox 用于建立 Pod 的基础运行环境。

### 1.6 配置 Pod 网络

Runtime 调用 CNI：

```text
containerd
    |
    v
CNI
    |
    v
Flannel / Calico / Cilium
```

例如创建：

```text
Pod
 |
 v
veth pair
 |
 v
cni0
 |
 v
Node
```

并为 Pod 分配 IP。

### 1.7 拉取镜像

如果本地没有镜像：

```text
Node
 |
 v
Registry
 |
 +-- Manifest
 +-- Layer
 +-- Snapshot
```

完成镜像下载后创建 Container。

### 1.8 启动 Container

最终形成：

```text
Pod
 |
 +-- pause / sandbox
 |
 +-- nginx
 |
 +-- sidecar
```

kubelet 持续维护 Pod 状态，并将状态更新回 API Server。

---

## 2. Kubernetes 各个组件的作用

Kubernetes 可以简单分为 Control Plane 和 Node 两部分。

### 2.1 Control Plane

| 组件 | 作用 |
|---|---|
| kube-apiserver | Kubernetes API 入口 |
| etcd | 保存集群状态 |
| kube-scheduler | Pod 调度 |
| kube-controller-manager | 各类 Controller |
| cloud-controller-manager | 云平台相关控制逻辑 |

### 2.2 Node

| 组件 | 作用 |
|---|---|
| kubelet | Node Agent，负责 Pod 生命周期 |
| kube-proxy | Service 流量转发 |
| Container Runtime | 运行容器 |
| CNI | Pod 网络 |
| CSI | 存储 |

### 2.3 Kubernetes 的核心思想

Kubernetes 并不是简单执行用户命令，而是：

```text
用户声明 Desired State
        |
        v
API Server
        |
        v
Controller
        |
        v
不断调整 Actual State
        |
        v
最终接近 Desired State
```

这就是 Kubernetes 的：

> 声明式 API + Controller 控制循环

---

## 3. PLEG 是什么

PLEG：

> Pod Lifecycle Event Generator

它位于 kubelet 中，负责感知 Pod/Container 生命周期变化。

简单理解：

```text
kubelet 认为：
nginx 应该 Running

        |

Runtime 实际：
nginx 已经退出

        |

PLEG 发现状态变化
```

### 3.1 PLEG 的作用

传统 GenericPLEG 会周期性与 Runtime 交互，例如获取：

```text
ListPodSandbox
ListContainers
Container Status
```

然后根据前后状态生成事件。

### 3.2 为什么 PLEG 很重要

如果 PLEG 长时间无法正常工作：

```text
PLEG
 |
 v
无法获取 Runtime 状态
 |
 v
kubelet 无法及时同步 Pod 状态
 |
 v
Node 状态异常
```

常见日志：

```text
PLEG is not healthy
```

因此排查 Node 异常时，需要重点关注：

```text
kubelet
containerd
PLEG
CPU
Memory
Disk IO
```

---

## 4. Kubernetes 常用优化参数

Kubernetes 参数优化不能脱离集群规模。

小集群和大规模集群的优化思路完全不同。

---

### 4.1 kube-apiserver

常见参数：

```text
--max-requests-inflight #控制读类 API 并发，默认 400
--max-mutating-requests-inflight #控制写类 API 并发，默认 200
--request-timeout #控制普通 API 请求最长处理时间，默认 1 分钟

现代 Kubernetes 默认开启 APF：
400 + 200 更应该理解为 APF 的总并发额度。
```

需要重点关注：

```text
API QPS
API Latency
Inflight Requests
Watch 数量
5xx
etcd latency
```

核心原则：

> API Server 不是越快越好，而是要保证请求压力不会把 etcd 和整个控制面拖垮。

---

### 4.2 Scheduler

常见关注：

```text
--kube-api-qps #限制客户端平均请求速率
--kube-api-burst #限制允许的突发请求数量
```

大规模集群还需要关注调度并发能力以及 Scheduler 本身 CPU / Memory 使用情况。

---

### 4.3 Controller Manager

常见参数：

```text
--kube-api-qps #限制客户端平均请求速率
--kube-api-burst #限制允许的突发请求数量
--concurrent-deployment-syncs #控制 Deployment的并发 reconcile worker 数
--concurrent-replicaset-syncs #控制ReplicaSet的并发 reconcile worker 数 
--concurrent-service-syncs #控制Service Controller的并发 reconcile worker 数 
```

并发度提高：

```text
Controller 处理速度提高
```

但是同时：

```text
API Server 压力增加
etcd 压力增加
```

所以优化目标不是单纯提高并发，而是找到控制面能够承受的平衡点。

---

### 4.4 kubelet

重点参数：

```text
--kube-api-qps
--kube-api-burst

--max-pods

--image-gc-high-threshold  # 镜像磁盘使用率达到该阈值后触发 ImageGC，开始清理未使用镜像
--image-gc-low-threshold   # ImageGC 清理目标阈值，清理会持续到镜像磁盘使用率降到该值以下

--eviction-hard            # Kubelet 硬驱逐阈值，资源达到条件后立即驱逐 Pod，例如 memory.available、nodefs.available、imagefs.available
--eviction-soft            # Kubelet 软驱逐阈值，资源达到条件并持续超过对应 grace period 后才驱逐 Pod

--pod-pids-limit           # 限制单个 Pod 可创建的最大进程数（PID 数量），用于防止 fork bomb 或单个 Pod 耗尽节点 PID
```

同时应该合理设置：

```text
systemReserved
kubeReserved
eviction
```

避免业务 Pod 把 Node 的系统资源全部消耗掉。

---

### 4.5 etcd

etcd 的优化重点通常不是简单修改某个参数，而是：

```text
磁盘 IO
磁盘延迟
CPU
Memory
Network
DB Size
Compaction
Defrag
```

其中最重要的一点：

> etcd 对磁盘延迟非常敏感。

---

## 5. Kubernetes 跨集群访问

跨集群访问常见方案主要有以下几种。

### 5.1 Ingress / Gateway

```text
Cluster A
   |
   v
Gateway / Load Balancer
   |
   v
Cluster B
```

适合 HTTP / HTTPS API。

### 5.2 LoadBalancer

通过云 LB 暴露服务：

```text
Cluster A
   |
   v
Cloud Load Balancer
   |
   v
Cluster B
```

### 5.3 VPC / VPN / 专线

例如：

```text
Cluster A VPC
       |
   VPN / Peering
       |
Cluster B VPC
```

打通网络后，可以直接访问对方集群暴露的地址。

### 5.4 Service Mesh

例如 Istio 多集群：

```text
Cluster A
   |
   v
Istio
   |
   v
Cluster B
```

可以实现：

- 服务发现
- mTLS
- 流量治理
- 灰度
- 熔断
- 重试

### 5.5 Global DNS / Global Load Balancing

```text
api.example.com
        |
        v
Global DNS / GSLB
        |
   +----+----+
   |         |
Cluster A  Cluster B
```

适合多地域、多活架构。

---

## 6. Kubernetes 灰度发布

灰度发布的本质：

> 不一次性把全部流量切换到新版本，而是逐步扩大新版本流量。

---

### 6.1 Deployment Rolling Update

最基础的方式：

```text
v1
 |
 v
v1 + v2
 |
 v
v2
```

适合普通应用升级。

---

### 6.2 两套 Deployment + Service

```text
             Service
             /     \
            /       \
          v1         v2
```

例如：

```text
v1: 9 Pods
v2: 1 Pod
```

可以实现近似 90/10 的容量比例。

但需要注意：

> Pod 数量比例不等于严格的流量比例。

---

### 6.3 Ingress 灰度

可以基于：

```text
Header
Cookie
Weight
IP
```

进行路由。

例如：

```text
X-Canary: true
```

进入 v2。

---

### 6.4 Service Mesh

Service Mesh 可以实现：

```text
90% -> v1
10% -> v2
```

甚至：

```text
测试用户 -> v2
普通用户 -> v1
```

适合复杂灰度场景。

---

### 6.5 蓝绿发布

```text
             Service
              |
       +------+------+
       |             |
     Blue          Green
       |             |
      v1             v2
```

发布完成后：

```text
Service
   |
   v
Green
```

蓝绿发布最大的优势：

> 回滚速度快。

---

## 7. NodeLocal DNS 解决什么问题

默认情况下：

```text
Pod
 |
 v
CoreDNS Service
 |
 v
CoreDNS Pod
```

大量 Pod 进行 DNS 查询时，会产生：

```text
DNS QPS
conntrack
kube-proxy
网络转发
CoreDNS 压力
```

NodeLocal DNSCache 会在每个 Node 上运行本地 DNS Cache。

变成：

```text
Pod
 |
 v
NodeLocal DNSCache
 |
 +-- Cache Hit -> 直接返回
 |
 +-- Cache Miss
       |
       v
    CoreDNS
```

### 7.1 解决的问题

主要解决：

1. 降低 DNS 延迟
2. 减少 CoreDNS 压力
3. 减少跨网络 DNS 查询
4. 降低 kube-proxy / Service 转发开销
5. 降低 conntrack 压力

在大规模集群中尤其有价值。

---

## 8. Flannel 网络转发流程

假设：

```text
Pod A:
10.244.1.10

Node A:
192.168.1.10

Pod B:
10.244.2.20

Node B:
192.168.1.20
```

Pod A 访问 Pod B。

### 8.1 Pod -> Node

```text
Pod A
 |
 v
veth
 |
 v
cni0
 |
 v
Node A
```

### 8.2 Node A -> Node B

Node A 判断：

```text
10.244.2.20
```

属于 Node B 的 Pod CIDR。

如果使用 VXLAN：

```text
Pod A
 |
 v
veth
 |
 v
cni0
 |
 v
flannel.1
 |
 v
VXLAN Encapsulation
 |
 v
Node B
 |
 v
flannel.1
 |
 v
cni0
 |
 v
veth
 |
 v
Pod B
```

### 8.3 Flannel Backend

常见方式：

```text
VXLAN
Host-GW
WireGuard
```

VXLAN：

> 通过封装实现跨 Node Pod 网络。

Host-GW：

> 直接利用 Node 路由转发，通常路径更简单、性能更好，但要求底层网络具备对应路由能力。

---

## 9. Pod Sandbox 隔离了什么

PodSandbox 不应该简单理解成“隔离容器”。

它主要负责建立 Pod 的基础运行环境。

典型 Pod：

```text
Pod
 |
 +-- pause
 |
 +-- nginx
 |
 +-- sidecar
```

Pod 中多个 Container 可以共享：

```text
Network Namespace
IPC Namespace
UTS Namespace
```

因此：

```text
nginx
10.244.1.10:80

sidecar
10.244.1.10:8080
```

它们使用同一个网络 Namespace。

### 9.1 为什么需要 pause Container

pause Container 主要承担 Pod Sandbox 的 Namespace 生命周期。

可以理解为：

```text
pause
 |
 +-- Network Namespace
 +-- IPC Namespace
 +-- UTS Namespace
```

其他 Container 加入这些 Namespace。

### 9.2 PID Namespace

Pod 是否共享 PID Namespace 取决于：

```yaml
shareProcessNamespace: true
```

默认并不是所有 Pod Container 都共享 PID Namespace。

---

## 10. 大规模节点同时拉取镜像

假设：

```text
1000 Nodes
*
10GB GPU Image
```

如果所有 Node 同时访问 Registry：

```text
1000 Nodes
      |
      v
 Registry
```

容易造成：

```text
Registry 带宽瓶颈
Registry CPU 瓶颈
网络带宽瓶颈
Pod 启动时间增加
```

### 10.1 Registry 高可用

例如：

```text
Nodes
 |
 v
Registry Cluster
 |
 v
Object Storage
```

通过多个 Registry 实例提高吞吐。

### 10.2 镜像预热

使用 DaemonSet 提前拉取：

```text
DaemonSet
   |
   +-- Node 1 -> Image
   +-- Node 2 -> Image
   +-- Node 3 -> Image
```

业务 Pod 启动时直接使用本地镜像。

### 10.3 P2P 镜像分发

例如 Dragonfly：

```text
Registry
    |
    v
Node A
 / | \
v  v  v
B  C  D
```

Node 之间共享镜像 Layer。

适合：

- 大规模节点
- 大镜像
- GPU 镜像
- 大规模批量扩容

### 10.4 镜像瘦身

例如：

```text
10GB
 |
 v
5GB
```

可以通过：

- Multi-stage Build
- Runtime Image
- 删除 Build Dependency
- 删除缓存
- 减少不必要的 CUDA 组件

降低镜像大小。

---

## 11. GPU 资源隔离

GPU 隔离需要区分三个维度：

```text
设备隔离
显存隔离
算力隔离
```

### 11.1 整卡独占

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

Pod 独占整张 GPU。

优点：

- 简单
- 稳定
- 隔离清晰

缺点：

- GPU 利用率可能较低

### 11.2 MIG

支持 MIG 的 NVIDIA GPU 可以划分：

```text
GPU
 |
 +-- MIG Instance 1
 +-- MIG Instance 2
 +-- MIG Instance 3
 +-- MIG Instance 4
```

MIG 可以提供较强的硬件资源隔离。

### 11.3 MPS / 时间片 / vGPU / qGPU

可以让多个工作负载共享 GPU：

```text
GPU
 |
 +-- Pod A
 +-- Pod B
 +-- Pod C
```

但需要注意：

```text
nvidia.com/gpu: 1
```

本身主要表示设备资源分配，并不天然意味着显存和算力都获得严格隔离。

GPU 隔离通常需要结合：

```text
NVIDIA Device Plugin
MIG
MPS
vGPU
qGPU
```

---

## 12. ImageGC

ImageGC：

> Image Garbage Collection

即 kubelet 自动清理本地无用镜像。

例如：

```text
Node
 |
 +-- app:v1
 +-- app:v2
 +-- app:v3
 +-- old:v1
 +-- old:v2
```

随着发布次数增加，本地镜像越来越多，可能导致磁盘：

```text
80%
90%
95%
100%
```

ImageGC 可以自动清理。

### 12.1 常见参数

```text
imageGCHighThresholdPercent
imageGCLowThresholdPercent
```

例如：

```text
High = 85
Low  = 80
```

表示：

```text
磁盘 > 85%
     |
     v
触发 ImageGC
     |
     v
清理镜像
     |
     v
降低到目标阈值
```

---

## 13. Informer

Informer 是 client-go 中非常重要的机制。

它主要解决：

> 如何高效监听 Kubernetes Object 的变化。

如果每个 Controller 都不断：

```text
GET /api/v1/pods
```

会给 API Server 带来巨大压力。

Informer 的典型结构：

```text
API Server
    |
    | List / Watch
    v
Reflector
    |
    v
DeltaFIFO
    |
    v
Indexer / Local Cache
    |
    v
Controller
```

### 13.1 Reflector

负责：

```text
List
Watch
```

API Server。

### 13.2 DeltaFIFO

保存资源变化：

```text
Added
Updated
Deleted
```

### 13.3 Indexer

提供本地缓存：

```text
Get
List
```

Controller 优先访问本地 Cache，而不是每次访问 API Server。

### 13.4 为什么大规模集群必须重视 Informer

例如：

```text
10000 Pods
100 Controllers
```

如果每个 Controller 都不断 List：

```text
API Server
     ^
     |
大量 GET/List
```

会造成控制面压力。

Informer 可以把大量重复查询转换为：

```text
一次 List
+
长期 Watch
+
本地 Cache
```

---

## 14. Operator 触发原理

Operator 可以理解为：

```text
CRD
+
Controller
+
Reconcile
```

例如定义：

```yaml
apiVersion: mysql.example.com/v1
kind: MySQL
spec:
  replicas: 3
```

用户创建 MySQL CR。

### 14.1 事件产生

```text
User
 |
 v
API Server
 |
 v
etcd
 |
 v
Watch
 |
 v
Informer
 |
 v
WorkQueue
 |
 v
Reconcile()
```

### 14.2 Reconcile

Controller 获取：

```text
Desired State
Actual State
```

进行比较。

例如：

```text
Desired:
3 Pods

Actual:
2 Pods
```

那么 Controller：

```text
创建 1 个 Pod
```

最终：

```text
Actual State
      |
      v
Desired State
```

### 14.3 Operator 的本质

Operator 并不是：

```text
CRD 创建
    |
    v
自动执行某个脚本
```

而是：

```text
Watch Event
    |
    v
Queue
    |
    v
Reconcile
    |
    v
持续让 Actual State 接近 Desired State
```

---

## 15. 大规模集群为什么使用 IPVS

Kubernetes Service 的流量转发可以使用 iptables、IPVS 等机制。

假设：

```text
10000 Services
100000 Endpoints
```

### 15.1 iptables

iptables 本质上维护大量规则：

```text
Packet
  |
  v
iptables chain
  |
  +-- rule
  +-- rule
  +-- rule
  +-- ...
```

当规则规模非常大时：

- 规则数量多
- 规则更新成本高
- 规则匹配开销增加

### 15.2 IPVS

IPVS 使用 Linux 内核的虚拟服务器能力和表结构管理后端。

例如：

```text
10.96.0.10:80
       |
       +-- 10.244.1.10:8080
       +-- 10.244.2.10:8080
       +-- 10.244.3.10:8080
```

支持多种调度算法：

```text
rr
lc
sh
dh
```

### 15.3 对比

| 维度 | iptables | IPVS |
|---|---|---|
| 基础机制 | Netfilter Rules | IPVS Virtual Server |
| 数据结构 | Rule Chain | Table |
| 大规模 Service | 压力较大 | 更适合 |
| Endpoint 多 | 规则数量大 | 更适合 |
| 调度算法 | 相对有限 | 更丰富 |
| 规则更新 | 成本较高 | 更适合大规模场景 |

需要注意：

> 不能简单理解为“IPVS 在任何情况下都比 iptables 快”。

更准确的说法是：

> 当 Service 和 Endpoint 数量非常大时，IPVS 的表结构和内核负载均衡机制更适合大规模 Service 场景。

---

# 16. 从 SRE 角度保证 Kubernetes 集群稳定性

Kubernetes 稳定性不能只看：

```text
Pod 有没有 Running
```

SRE 更关注：

```text
可用性
性能
容量
故障恢复
变更风险
影响范围
```

可以从以下几个方面建设。

---

## 16.1 控制面稳定性

重点组件：

```text
API Server
etcd
Scheduler
Controller Manager
```

### API Server

监控：

```text
QPS
Latency
5xx
Inflight Requests
Watch
CPU
Memory
```

### etcd

监控：

```text
fsync latency
commit latency
DB size
leader changes
CPU
Memory
Disk
```

尤其关注：

> 磁盘 IO 和磁盘延迟。

---

## 16.2 Node 稳定性

重点关注：

```text
CPU
Memory
Disk
Network
PID
FD
Conntrack
```

Kubernetes 本身还需要关注：

```text
MemoryPressure
DiskPressure
PIDPressure
```

---

## 16.3 Kubelet / Runtime 稳定性

重点：

```text
PLEG
containerd
Pod Startup Latency
Image Pull Latency
Runtime API Latency
```

例如：

```text
Pod Pending
    |
    +-- Scheduler 问题
    |
    +-- Image Pull 问题
    |
    +-- CNI 问题
    |
    +-- Runtime 问题
    |
    +-- kubelet 问题
```

---

## 16.4 网络稳定性

重点：

```text
CNI
kube-proxy
IPVS / iptables
conntrack
DNS
Packet Loss
Latency
```

大规模集群通常需要考虑：

```text
NodeLocal DNS
IPVS / eBPF
conntrack 优化
CNI 优化
```

---

## 16.5 资源隔离

避免一个 Pod 把 Node 的资源全部消耗掉。

常用机制：

```text
requests
limits
QoS
PriorityClass
Taints
TopologySpread
PodAntiAffinity
```

例如：

```yaml
resources:
  requests:
    cpu: "1"
    memory: "2Gi"
  limits:
    cpu: "2"
    memory: "4Gi"
```

---

## 16.6 发布稳定性

不要：

```text
一次性 100% 发布
```

而是：

```text
5%
 |
 v
20%
 |
 v
50%
 |
 v
100%
```

每个阶段观察：

```text
Error Rate
Latency
QPS
CPU
Memory
Business Metrics
```

出现异常：

```text
自动停止
或者
自动回滚
```

---

## 16.7 容量管理

容量管理不是：

> 资源快用完了再扩容。

而应该进行容量预测。

例如：

```text
CPU 当前 70%
增长速度 5% / 月
```

预测：

```text
3 个月后
70% + 15%
= 85%
```

那么应该提前扩容。

需要关注：

```text
CPU
Memory
Pod
IP
Disk
GPU
```

---

## 16.8 故障隔离

核心原则：

> 一个故障不要扩散成整个集群故障。

可以使用：

```text
Multi-AZ
Node Pool
Taints
PriorityClass
Resource Quota
LimitRange
PodAntiAffinity
TopologySpread
PDB
```

把业务进行隔离。

---

## 16.9 多集群容灾

对于核心业务，可以：

```text
Cluster A
Cluster B
Cluster C
```

当：

```text
Cluster A
```

发生重大故障：

```text
Traffic
   |
   v
Cluster B
```

实现故障转移。

---

# 总结

如果把 Kubernetes 稳定性浓缩成一套 SRE 方法论，可以归纳成：

```text
                    Kubernetes SRE
                          |
       +------------------+------------------+
       |                  |                  |
     控制面              数据面              资源
       |                  |                  |
     API/etcd          Network/DNS        CPU/Memory/GPU
       |                  |                  |
       +------------------+------------------+
                          |
                      可观测性
                          |
             +------------+------------+
             |            |            |
          Metrics       Logs         Tracing
             |
             v
           SLI
             |
             v
           SLO
             |
             v
       Error Budget
             |
             v
       变更 / 发布 / 容灾
             |
             v
          稳定性
```

最终 SRE 的目标不是让 Kubernetes **永远不出故障**，而是：

```text
故障能够被发现
        |
        v
故障影响范围可控
        |
        v
能够快速定位
        |
        v
能够自动恢复
        |
        v
能够快速回滚
        |
        v
能够通过复盘避免再次发生
```

这才是 Kubernetes 运维从“会部署、会排障”向“稳定性工程”演进的核心。
