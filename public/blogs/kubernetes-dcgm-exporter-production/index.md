# Kubernetes 生产环境部署 NVIDIA DCGM Exporter GPU 监控

## 1. 背景

Kubernetes GPU 集群除了 CPU、内存等基础监控外，还需要关注：

- GPU 核心利用率
- 显存使用率
- GPU 温度
- GPU 功耗
- GPU 与 Pod 的关联关系

这里使用 NVIDIA 官方 `dcgm-exporter`，通过 `ServiceMonitor` 接入 Prometheus。

整体链路：

```text
NVIDIA GPU
    ↓
DCGM
    ↓
dcgm-exporter
    ↓
ServiceMonitor
    ↓
Prometheus
    ↓
Grafana
```

---

## 2. 核心 values.yaml

```yaml
image:
  repository: nvcr.io/nvidia/k8s/dcgm-exporter
  pullPolicy: IfNotPresent

priorityClassName: system-node-critical

service:
  enable: true
  type: ClusterIP
  port: 9400
  address: ":9400"

serviceMonitor:
  enabled: true
  interval: 30s
  scrapeTimeout: 25s
  honorLabels: false

  relabelings:
    - sourceLabels:
        - __meta_kubernetes_pod_node_name
      targetLabel: node
      action: replace

    - sourceLabels:
        - __meta_kubernetes_pod_node_name
      targetLabel: Hostname
      action: replace

    - sourceLabels:
        - __address__
      regex: '(.+):[0-9]+'
      replacement: '$1'
      targetLabel: instance
      action: replace

    - sourceLabels:
        - __meta_kubernetes_endpoint_node_name
      targetLabel: nodepool
      regex: '.*(np-[a-z0-9]+).*'
      replacement: '$1'
      action: replace

nodeSelector:
  node.tke.cloud.tencent.com/accelerator-type: "gpu"

tolerations:
  - key: "nvidia.com/gpu"
    operator: "Exists"
    effect: "NoSchedule"

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

kubernetes:
  enablePodLabels: false
  enablePodUID: false
  rbac:
    create: true

securityContext:
  runAsNonRoot: false
  runAsUser: 0
  allowPrivilegeEscalation: false
  capabilities:
    add:
      - SYS_ADMIN
    drop:
      - ALL
```

---

## 3. 安装 DCGM Exporter

添加 NVIDIA Helm 仓库：

```bash
helm repo add gpu-helm-charts \
  https://nvidia.github.io/dcgm-exporter/helm-charts
```

更新 Helm Repository：

```bash
helm repo update
```

查看可用版本：

```bash
helm search repo gpu-helm-charts/dcgm-exporter
```

查看 Chart 信息：

```bash
helm show chart gpu-helm-charts/dcgm-exporter
```

安装或升级：

```bash
  helm upgrade --install dcgm-exporter \
  gpu-helm-charts/dcgm-exporter \
  --namespace monitoring \
  --create-namespace \
  --version 4.8.4 \
  -f  values.yaml \
  --wait \
  --timeout 10m
```

如果 Namespace 不存在：

```bash
kubectl create namespace monitoring
```

安装完成后检查：

```bash
helm -n monitoring list
```

```bash
kubectl -n monitoring get ds,pod,svc
```

---

## 4. 核心配置说明

### 镜像版本

没有手动指定：

```yaml
image:
  tag:
```

默认跟随 Helm Chart 的 `AppVersion`。

这样可以减少 `dcgm-exporter` 与 DCGM 版本不匹配的问题。

### 只部署到 GPU 节点

```yaml
nodeSelector:
  node.tke.cloud.tencent.com/accelerator-type: "gpu"
```

避免 dcgm-exporter 调度到普通 CPU 节点。

dcgm-exporter 本身只负责监控 GPU，不需要配置：

```yaml
nvidia.com/gpu: 1
```

否则会实际占用 Kubernetes GPU Resource。

### ServiceMonitor

```yaml
serviceMonitor:
  enabled: true
```

通过 Prometheus Operator 自动发现 dcgm-exporter。

如果 ServiceMonitor 已经创建，但是 Prometheus Targets 中没有 dcgm-exporter，优先检查：

```text
serviceMonitorSelector
```

是否可以匹配对应的 ServiceMonitor。

---

## 5. 标签处理

为了方便 Grafana 查询，额外增加：

```text
node
Hostname
instance
nodepool
```

例如 Node Name：

```text
rcs-eu-gpu-fix.np-mdao5huq.1
```

通过：

```yaml
regex: '.*(np-[a-z0-9]+).*'
replacement: '$1'
```

提取后：

```text
nodepool="np-mdao5huq"
```

Grafana 可以直接按照 NodePool 和 Hostname 过滤：

```promql
DCGM_FI_DEV_GPU_UTIL{
  nodepool=~"$NodePool",
  Hostname=~"$GPUNode"
}
```

---

## 6. 核心 PromQL

### GPU 核心利用率

```promql
avg_over_time(
  DCGM_FI_DEV_GPU_UTIL{
    nodepool=~"$NodePool",
    Hostname=~"$GPUNode"
  }[$interval]
)
```

### 显存使用率

```promql
100 *
DCGM_FI_DEV_FB_USED{
  nodepool=~"$NodePool",
  Hostname=~"$GPUNode"
}
/
(
  DCGM_FI_DEV_FB_USED{
    nodepool=~"$NodePool",
    Hostname=~"$GPUNode"
  }
  +
  DCGM_FI_DEV_FB_FREE{
    nodepool=~"$NodePool",
    Hostname=~"$GPUNode"
  }
)
```

### GPU 温度

```promql
DCGM_FI_DEV_GPU_TEMP{
  nodepool=~"$NodePool",
  Hostname=~"$GPUNode"
}
```

### GPU 功耗

```promql
DCGM_FI_DEV_POWER_USAGE{
  nodepool=~"$NodePool",
  Hostname=~"$GPUNode"
}
```

---

## 7. 生产注意点

### 控制指标基数

当前：

```yaml
kubernetes:
  enablePodLabels: false
```

建议保持关闭。

如果将大量 Kubernetes Pod Label 加入 GPU Metrics，容易增加 Prometheus Time Series 数量。

### SYS_ADMIN

```yaml
capabilities:
  add:
    - SYS_ADMIN
```

主要用于部分 DCGM Profiling 指标，例如：

```text
DCGM_FI_PROF_*
```

如果明确不需要 Profiling 指标，可以进一步评估是否移除。

---

## 8. 验证

检查 DaemonSet：

```bash
kubectl -n monitoring get ds
```

检查 Pod：

```bash
kubectl -n monitoring get pod -o wide
```

检查 ServiceMonitor：

```bash
kubectl -n monitoring get servicemonitor
```

Prometheus 中查询：

```promql
DCGM_FI_DEV_GPU_UTIL
```

确认指标中存在：

```text
Hostname
instance
nodepool
gpu
```

即可确认采集链路基本正常。

---

## 总结

这套配置主要解决：

```text
GPU 节点自动部署 dcgm-exporter
          ↓
ServiceMonitor 自动接入 Prometheus
          ↓
统一 Hostname / instance / nodepool 标签
          ↓
Grafana 根据 NodePool / Node / GPU 展示
```

最终形成：

```text
GPU → DCGM Exporter → Prometheus → Grafana
```

的 Kubernetes GPU 监控体系。

