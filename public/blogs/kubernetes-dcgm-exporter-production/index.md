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
    # Kubernetes 节点名
    - sourceLabels:
        - __meta_kubernetes_pod_node_name
      targetLabel: node
      action: replace

    # Grafana 使用的 Hostname
    - sourceLabels:
        - __meta_kubernetes_pod_node_name
      targetLabel: Hostname
      action: replace

    # instance 去除 :9400
    - sourceLabels:
        - __address__
      regex: '(.+):[0-9]+'
      replacement: '$1'
      targetLabel: instance
      action: replace

    # 从节点名提取 NodePool
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

## 3. 核心配置说明

### 镜像版本

没有手动指定 `image.tag`，默认跟随 Helm Chart 的 `AppVersion`。

这样可以减少 `dcgm-exporter` 和 DCGM 版本不匹配的问题。

---

### 只部署到 GPU 节点

```yaml
nodeSelector:
  node.tke.cloud.tencent.com/accelerator-type: "gpu"
```

避免 dcgm-exporter 部署到普通 CPU 节点。

dcgm-exporter 本身只是监控 GPU，因此不要配置：

```yaml
nvidia.com/gpu: 1
```

否则会实际占用 Kubernetes GPU Resource。

---

### ServiceMonitor

```yaml
serviceMonitor:
  enabled: true
```

通过 Prometheus Operator 自动发现 dcgm-exporter。

如果 ServiceMonitor 已创建，但 Prometheus 中没有 Target，优先检查：

```text
Prometheus serviceMonitorSelector
```

是否能匹配 dcgm-exporter 的 ServiceMonitor Label。

---

## 4. 标签处理

为了方便 Grafana 查询，额外增加：

```text
node
Hostname
instance
nodepool
```

例如 Kubernetes 节点：

```text
rcs-eu-gpu-fix.np-mdao5huq.1
```

经过：

```yaml
regex: '.*(np-[a-z0-9]+).*'
replacement: '$1'
```

最终得到：

```text
nodepool="np-mdao5huq"
```

这样 Grafana 就可以直接按照 NodePool 过滤 GPU：

```promql
DCGM_FI_DEV_GPU_UTIL{
  nodepool=~"$NodePool",
  Hostname=~"$GPUNode"
}
```

---

## 5. PromQL

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

## 6. 生产环境注意点

### 不建议开启所有 Pod Label

当前：

```yaml
kubernetes:
  enablePodLabels: false
```

建议保持关闭。

如果把大量 Kubernetes Pod Label 注入 GPU Metrics，容易造成 Prometheus 指标基数快速增加。

---

### SYS_ADMIN

```yaml
capabilities:
  add:
    - SYS_ADMIN
```

主要用于 DCGM Profiling 类指标。

如果后续明确不使用：

```text
DCGM_FI_PROF_*
```

可以再评估是否移除。

---

## 7. 验证

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

Prometheus 中检查：

```promql
DCGM_FI_DEV_GPU_UTIL
```

如果能够查询到 GPU 数据，并且存在：

```text
Hostname
instance
nodepool
gpu
```

这些 Label，说明采集链路正常。

---

## 总结

这套配置核心解决四个问题：

```text
GPU 节点自动部署 dcgm-exporter

ServiceMonitor 自动接入 Prometheus

统一 Hostname / instance / nodepool 标签

控制指标 Cardinality，避免 Prometheus 压力过大
```

最终形成：

```text
GPU → DCGM Exporter → Prometheus → Grafana
```

的 Kubernetes GPU 监控体系。

## Slug

```text
kubernetes-dcgm-exporter-production
```