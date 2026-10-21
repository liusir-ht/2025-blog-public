# Kubernetes 中使用 Grafana Alloy 采集日志到 Loki

## 概述

Grafana Alloy 是 Grafana Labs 推出的 OpenTelemetry Collector 发行版，支持采集、处理和转发日志、指标、链路数据。在 Kubernetes 场景下，通常以 DaemonSet 方式部署到每个节点，直接读取节点上的 `/var/log/pods` 目录，并将 Pod 日志推送至 Loki。

本文介绍一套经过实践验证的 Alloy 日志采集方案，涵盖：

- 只采集指定 namespace（`dev`、`test`、`pre`）
- 从业务日志中提取 `level` 标签
- 合并 Java 多行堆栈
- 使用业务日志时间覆盖时间戳，并正确处理东八区
- 精简和优化标签

## 一、架构与数据流

```text
K8s API（仅发现本节点 Pod 元数据）
        │  discovery.kubernetes + relabel
        ▼
计算 __path__ = /var/log/pods/*<uid>/*.log
        │  local.file_match（每 5s 同步）
        ▼
loki.source.file  ← 读 hostPath /var/log/pods
        │
        ▼
loki.process：CRI 解析 → 多行合并 → 提取 level → 时间戳覆盖
        │
        ▼
loki.write → Loki push API
```

### 设计要点

- **API 只做发现，不读日志**：通过 `discovery.kubernetes` 获取 Pod 元数据，日志内容直接从节点文件系统读取，降低 API Server 压力。
- **断点续读**：读取位置保存在 hostPath 的 `/var/lib/alloy` 下，Pod 重启后可以复用。
- **权限最小化**：ClusterRole 只授予 `pods` 的 `get`、`list`、`watch` 权限，无需 `privileged`。

## 二、前置条件

- 运行中的 Kubernetes 集群（Linux 节点）
- 已部署 Loki，并知晓 push 地址与租户 ID（如启用多租户）
- 节点上存在 `/var/log/pods`（标准 containerd/CRI-O 环境均有）
- 可拉取 Alloy 镜像（示例使用华为云 SWR 同步的 `grafana/alloy:v1.16.1`）

## 三、部署清单

以下 YAML 包含 Namespace、ServiceAccount、ClusterRole、ClusterRoleBinding、ConfigMap、DaemonSet 六类资源，可直接执行：

```bash
kubectl apply -f alloy.yaml
```

### 3.1 基础资源

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: alloy
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: alloy-logs
rules:
  - apiGroups: [""]
    resources: [pods]
    verbs: [get, list, watch]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: alloy-logs
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: alloy-logs
subjects:
  - kind: ServiceAccount
    name: alloy
    namespace: monitoring
```

### 3.2 ConfigMap：Alloy 配置

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alloy-config
  namespace: monitoring
data:
  config.alloy: |
    logging {
      level  = "info"
      format = "logfmt"
    }

    // 1. 发现本节点 Pod 元数据（仅 API 发现，不通过 API 读日志）
    discovery.kubernetes "pods" {
      role = "pod"
      selectors {
        role = "pod"
        field = "spec.nodeName=" + sys.env("NODE_NAME")
      }
    }

    // 2. 打标签 + 计算日志路径
    discovery.relabel "pod_logs" {
      targets = discovery.kubernetes.pods.targets

      // 只保留 dev / test / pre 三个 namespace
      rule {
        source_labels = ["__meta_kubernetes_namespace"]
        regex         = "dev|test|pre"
        action        = "keep"
      }

      rule {
        source_labels = ["__meta_kubernetes_namespace"]
        target_label  = "env"
      }
      rule {
        source_labels = ["__meta_kubernetes_pod_name"]
        target_label  = "pod"
      }
      rule {
        source_labels = ["__meta_kubernetes_pod_container_name"]
        target_label  = "container"
      }
      rule {
        source_labels = ["__meta_kubernetes_pod_node_name"]
        target_label  = "hostname"
      }
      rule {
        source_labels = ["__meta_kubernetes_pod_label_app_kubernetes_io_instance"]
        target_label  = "app"
      }

      // 计算日志文件路径
      rule {
        source_labels = ["__meta_kubernetes_pod_uid", "__meta_kubernetes_pod_container_name"]
        separator     = "/"
        target_label  = "__path__"
        replacement   = "/var/log/pods/*$1/*.log"
      }
    }

    // 3. 文件匹配
    local.file_match "pod_logs" {
      path_targets = discovery.relabel.pod_logs.output
      sync_period  = "5s"
    }

    // 4. 读取文件并 tail
    loki.source.file "pod_logs" {
      targets       = local.file_match.pod_logs.targets
      tail_from_end = false
      forward_to    = [loki.process.pod_logs.receiver]
    }

    // 5. 解析 CRI + 多行合并 + 提取 level + 时间戳覆盖
    loki.process "pod_logs" {
      stage.cri {}

      stage.multiline {
        firstline     = `^\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2},\d{3}`
        max_wait_time = "3s"
        max_lines     = 256
      }

      stage.regex {
        expression = `^(?P<app_time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2},\d{3}) TID: \S+\s+(?P<level>[A-Z]+)\s`
      }

      stage.labels {
        values = {
          level = "",
        }
      }

      stage.timestamp {
        source   = "app_time"
        format   = "2006-01-02 15:04:05,000"
        location = "Asia/Shanghai"
      }

      forward_to = [loki.write.default.receiver]
    }

    // 6. 写入 Loki
    loki.write "default" {
      endpoint {
        url       = "http://loki-pre.huajing.tech:3100/loki/api/v1/push"
        tenant_id = "huajing"
      }
      external_labels = {
        cluster = "k8s-cluster-bj-pre",
      }
    }
```

### 3.3 DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: alloy
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: alloy
  template:
    metadata:
      labels:
        app.kubernetes.io/name: alloy
    spec:
      serviceAccountName: alloy
      terminationGracePeriodSeconds: 60
      nodeSelector:
        kubernetes.io/os: linux
      tolerations:
        - operator: Exists
      containers:
        - name: alloy
          image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/grafana/alloy:v1.16.1
          imagePullPolicy: IfNotPresent
          args:
            - run
            - /etc/alloy/config.alloy
            - --storage.path=/var/lib/alloy
            - --server.http.listen-addr=0.0.0.0:12345
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: TZ
              value: Asia/Shanghai
          securityContext:
            runAsUser: 0
            runAsGroup: 0
            allowPrivilegeEscalation: false
          ports:
            - name: http
              containerPort: 12345
          readinessProbe:
            httpGet:
              path: /-/ready
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 512Mi
          volumeMounts:
            - name: config
              mountPath: /etc/alloy
              readOnly: true
            - name: pod-logs
              mountPath: /var/log/pods
              readOnly: true
            - name: data
              mountPath: /var/lib/alloy
      volumes:
        - name: config
          configMap:
            name: alloy-config
        - name: pod-logs
          hostPath:
            path: /var/log/pods
            type: Directory
        - name: data
          hostPath:
            path: /var/lib/alloy
            type: DirectoryOrCreate
```

## 四、配置详解

### 4.1 只采集指定 namespace

在 `discovery.relabel` 中先执行 `keep`，只保留匹配的 namespace：

```alloy
rule {
  source_labels = ["__meta_kubernetes_namespace"]
  regex         = "dev|test|pre"
  action        = "keep"
}
```

`keep` 在发现阶段丢弃其余 Pod，不会进入文件匹配和读取，资源开销最小。正则默认进行全字符串匹配，不会误匹配 `dev-xxx`。如果 namespace 带前缀，可改为：

```text
huajing-dev|huajing-test|huajing-pre
```

### 4.2 标签设计

最终推送到 Loki 的标签集为：

```text
env + pod + container + hostname + app + level + stream + cluster
```

| 标签 | 来源或说明 |
| --- | --- |
| `env` | namespace 名（`dev`、`test`、`pre`） |
| `pod` | Pod 名称 |
| `container` | 容器名称 |
| `hostname` | 节点名（`spec.nodeName`） |
| `app` | Pod 的 `app.kubernetes.io/instance` 标签 |
| `level` | 从业务日志提取（如 `INFO`、`ERROR`） |
| `stream` | `stdout`/`stderr`，由 `stage.cri` 自动添加 |
| `cluster` | 由 `external_labels` 注入 |

已去掉 `job` 标签（原 namespace/container 拼接），因为它与 `env` + `container` 冗余。`level` 虽然会增加流数量，但其基数很低（通常为 4～5 个值），为方便按级别查询而保留。

### 4.3 多行合并 Java 堆栈

```alloy
stage.multiline {
  firstline     = `^\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2},\d{3}`
  max_wait_time = "3s"
  max_lines     = 256
}
```

以业务时间戳开头的行为新日志起点，后续堆栈行（如 `at org.apache...`）会追加到同一条日志中。

- `max_wait_time` 控制最长等待时间。
- `max_lines` 防止异常刷屏导致内存暴涨。

注意：该 stage 放在 `stage.cri {}` 之后，此时 CRI 前缀已剥离，可以直接匹配纯业务时间戳。

### 4.4 提取 level

```alloy
stage.regex {
  expression = `^(?P<app_time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2},\d{3}) TID: \S+\s+(?P<level>[A-Z]+)\s`
}

stage.labels {
  values = { level = "" }
}
```

对于如下日志：

```text
2026-10-20 17:10:45,701 TID: 850de... ERROR 1 --- ...
```

可以提取出 `app_time` 和 `level="ERROR"`。堆栈行不匹配正则，但合并后整条日志以第一行为准，因此 `level` 仍然正确。

### 4.5 时间戳与东八区

```alloy
stage.timestamp {
  source   = "app_time"
  format   = "2006-01-02 15:04:05,000"
  location = "Asia/Shanghai"
}
```

业务日志时间不带时区，必须显式指定 `location`。否则 Alloy 会按 UTC 解析，得到的时间比实际时间超前 8 小时，Loki 可能返回 `400 Bad Request: timestamp too new` 并丢弃数据。这是本方案最容易遇到的问题。

## 五、验证

查看 Alloy Pod 状态和日志：

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy
kubectl logs -n monitoring -l app.kubernetes.io/name=alloy --tail=50
```

在 Alloy UI（`http://<节点IP>:12345`）查看 `loki.process.pod_logs` 组件详情，确认输出标签与时间戳正确。

Loki 查询示例：

```logql
{env="test", app="mal-trade-web", level="ERROR"}
{env="dev"} |= "timeout"
```

## 六、常见问题

### 6.1 `timestamp too new` 400 错误

**原因**：`stage.timestamp` 未指定 `location`，业务时间被当成 UTC。

**解决**：增加：

```alloy
location = "Asia/Shanghai"
```

### 6.2 堆栈行没有 level 标签

**原因**：堆栈行不匹配正则，未合并时会被当作独立日志流。

**解决**：启用 `stage.multiline`，合并后整条日志以首行为准。

### 6.3 历史日志被重复推送

**原因**：`tail_from_end = false` 且 `/var/lib/alloy` 断点丢失。

**解决**：保证 hostPath 稳定；或者首次部署时临时改为 `true`，验证后再改回。

### 6.4 采集不到日志

按以下顺序排查：

1. 确认 namespace 在 `keep` 白名单内。
2. 确认 Pod 存在 `app.kubernetes.io/instance` 标签。
3. 检查 `/var/log/pods` 的挂载和读取权限。

## 七、总结

本方案以 DaemonSet + hostPath 直读节点日志为核心，通过 `discovery.relabel` 实现 namespace 白名单和标签精简，使用 `loki.process` 完成 CRI 解析、多行合并、`level` 提取与东八区时间戳覆盖，最终写入 Loki。

关键经验如下：

- API 只做发现，不通过 API 读取日志。
- 时间戳务必指定 `location = "Asia/Shanghai"`。
- 根据查询需求精简标签，控制标签基数。

