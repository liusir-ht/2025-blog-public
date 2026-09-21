
# Prometheus 监控接入、HTTPS 联邦与标签规范化

本文衔接 kube-prometheus-stack 部署，整理从已有 Node Exporter 接入，到 HTTPS 联邦、job 修改、instance 去端口及正则解释的实际操作。

所有地址、集群名和业务名都使用示例值。上层既可以是原生 Prometheus，也可以使用 kube-prometheus-stack；下层对外 HTTPS 与下层抓取 Exporter 是两条独立链路。

---

## 一、已有 Node Exporter 安装信息

已知安装记录：

```text
Release: node-exporter
Namespace: monitoring
Revision: 1
Status: deployed
Chart: prometheus-node-exporter-4.57.0
App Version: 1.12.1
```

Revision 1 是 Helm 修订号，不是 Pod 数量。Node Exporter 通常使用 DaemonSet，实际 Pod 数取决于节点选择器、污点容忍、系统类型和调度结果。

```bash
helm get values node-exporter -n monitoring -o yaml
kubectl get ds,pod,svc -n monitoring -l app.kubernetes.io/instance=node-exporter -o wide
```

Stack 的配置保留：

```yaml
nodeExporter:
  enabled: false
```

已有独立 Release 的 values 和 Stack values 是两个文件，分别升级对应 Release，不要混用。

---

## 二、让已有 Node Exporter 创建 ServiceMonitor

先安装 Stack，确认 CRD 可用：

```bash
kubectl get crd servicemonitors.monitoring.coreos.com
```

导出已有用户配置：

```bash
helm get values node-exporter -n monitoring -o yaml > node-exporter-values.yaml
```

如果导出为 `null`，说明没有用户覆盖值，改成 YAML 对象后再编辑。将以下内容合并进现有配置，不能新建重复的顶层 `prometheus:`：

```yaml
prometheus:
  monitor:
    enabled: true
    interval: 15s
    scrapeTimeout: 10s
    relabelings:
      - action: replace
        targetLabel: job
        replacement: node-exporter
      - action: replace
        sourceLabels:
          - __address__
        regex: '(.+):[0-9]+'
        replacement: '$1'
        targetLabel: instance
```

使用原版本升级：

```bash
helm upgrade node-exporter prometheus-community/prometheus-node-exporter \
  --namespace monitoring \
  --version 4.57.0 \
  --values node-exporter-values.yaml \
  --wait --timeout 10m
```

```bash
kubectl get servicemonitor -n monitoring
kubectl get servicemonitor -n monitoring -l app.kubernetes.io/instance=node-exporter -o yaml
```

如果标签选择器没有返回结果，用实际资源名称查询，不能仅根据标签查询为空认定没有 ServiceMonitor。

下层 Prometheus 需要选择该 ServiceMonitor。本文部署配置使用：

```yaml
prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
    serviceMonitorSelector: {}
    serviceMonitorNamespaceSelector: {}
```

---

## 三、修改默认 job 的含义

上面的规则将目标标签设置为：

```text
job="node-exporter"
```

它不会修改 Helm Release 名称、Service 名称或 Prometheus 内部的 scrape pool 名。Targets 中仍可能显示 `serviceMonitor/...`，这与指标中的 `job` 不矛盾。

需要区分集群时优先使用单独的 `cluster` 标签；也可按既有规范改成 `node-exporter-cluster-a`，但需要同步检查写死 `job="node-exporter"` 的默认规则和 Grafana 面板。

```promql
up{job="node-exporter"}
```

```promql
count by (job, instance) (node_uname_info)
```

另一种办法是 `ServiceMonitor.spec.jobLabel`：它读取被选中 Service 的某个标签值作为 job，不是把 jobLabel 字段本身当作固定 job 名称。例如 Service 标签 `monitoring-job: node-exporter` 搭配 `jobLabel: monitoring-job`。

---

## 四、instance 去掉 9100 端口

```yaml
- action: replace
  sourceLabels:
    - __address__
  regex: '(.+):[0-9]+'
  replacement: '$1'
  targetLabel: instance
```

| 项目 | 修改前 | 修改后 |
|---|---|---|
| 实际抓取地址 | 10.20.0.11:9100 | 10.20.0.11:9100 |
| instance 标签 | 10.20.0.11:9100 | 10.20.0.11 |

`targetLabel` 必须为 `instance`。如果改成 `__address__`，就会修改真实抓取地址，端口也随之改变。

这些属于目标 relabelings，发生在抓取前。抓取后的 metricRelabelings 中通常已没有 `__address__`，不能照搬。

若同一 IP 上有多个端口提供同类指标，去端口后应确保 job 或其他标签仍能唯一标识目标，避免时序冲突。IPv6 地址如 `[2001:db8::1]:9100` 匹配后会保留方括号，需要裸 IPv6 时另写规则。

---

## 五、正则表达式逐项解释

```regex
(.+):[0-9]+
```

| 片段 | 作用 |
|---|---|
| `.` | 匹配一个字符；本例地址不涉及换行 |
| `+` | 前一个模式重复一次或多次 |
| `(.+)` | 将匹配内容保存为第一个捕获组 |
| `:` | 字面量冒号 |
| `[0-9]+` | 一位或多位数字 |
| `$1` | replacement 中引用第一个捕获组 |

以 `10.20.0.11:9100` 为例：

| 完整输入 | 第一个捕获组 | 末尾匹配 |
|---|---|---|
| 10.20.0.11:9100 | 10.20.0.11 | :9100 |
| node-a.example.com:9100 | node-a.example.com | :9100 |

Prometheus relabel 正则按整个字符串匹配。`.+` 是贪婪的，但也必须满足后面的 `:[0-9]+`，因此捕获的是端口分隔符之前的内容。

**修正此前解释：**“先吃完再逐字符回退”可以辅助理解匹配结果，但不是 Prometheus 使用的 RE2 的实际执行机制。RE2 不采用传统回溯引擎的实现，不能据此推断存在灾难性回溯。见 [RE2 语法说明](https://github.com/google/re2/wiki/Syntax)。

---

## 六、改用节点名称作为 instance

如果业务面板更习惯节点名，可将去端口规则替换为：

```yaml
- action: replace
  sourceLabels:
    - __meta_kubernetes_pod_node_name
  regex: '(.+)'
  replacement: '$1'
  targetLabel: instance
```

它要求发现目标具有 Pod 元数据。先在 Service Discovery 中确认存在该标签。`regex: '(.+)'` 避免空源值覆盖目标。

也可以保留 IP 为 instance，另加主机标签：

```yaml
- action: replace
  sourceLabels:
    - __meta_kubernetes_pod_node_name
  regex: '(.+)'
  replacement: '$1'
  targetLabel: namehost
```

修改标签会产生新的时序。旧数据继续保留至保留期结束，但旧序列被标记 stale 后通常不再出现在即时查询中；“TSDB 里还保留旧数据”不意味着即时查询会一直返回旧值。

---

## 七、外部宿主机的 Node Exporter

先前提到的独立 Helm Release 是集群内 DaemonSet，不应再当作外部静态主机重复接入。以下配置仅用于真正位于集群外的主机。

合并到下层 Stack values：

```yaml
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: external-node-exporter
        scheme: http
        metrics_path: /metrics
        scrape_interval: 15s
        scrape_timeout: 10s
        static_configs:
          - targets:
              - 10.30.0.11:9100
            labels:
              namehost: external-node-a
              idc: region-a
              env: production
          - targets:
              - 10.30.0.12:9100
            labels:
              namehost: external-node-b
              idc: region-a
              env: production
        relabel_configs:
          - action: replace
            source_labels: [__address__]
            regex: '(.+):[0-9]+'
            replacement: '$1'
            target_label: instance
```

注意原生 Prometheus 配置使用 `source_labels/target_label/relabel_configs`；ServiceMonitor 使用 `sourceLabels/targetLabel/relabelings`，两者不能混写。

主机频繁变化时可以采用 file_sd、HTTP SD、云服务发现或 ScrapeConfig。file_sd 需要把目标文件实际挂载进 Prometheus 容器，仅填写路径不会自动获得文件。

---

## 八、HTTPS Federation 的两条链路

| 链路 | 示例 |
|---|---|
| 下层抓 Node Exporter | http://节点IP:9100/metrics |
| 下层抓 kubelet/cAdvisor | https://节点IP:10250/metrics/cadvisor |
| 上层抓下层 Prometheus | https://prometheus-a.example.com/federate |

下层对外使用 HTTPS 不意味着所有 Exporter 都必须改 HTTPS，也不能解决下层 Pod 到其他节点 9100 的连通性问题。

TLS 可终止在现有 Ingress/Gateway，后端仍以 HTTP 访问 Prometheus Service。确保 `/federate` 原样转发、查询参数保留，证书域名正确。具体入口资源跟随现有网关管理，不必额外部署另一套入口。

---

## 九、上层原生 Prometheus 配置

将下列 job 添加到现有 `scrape_configs` 列表：

```yaml
scrape_configs:
  - job_name: federate-cluster-a
    scheme: https
    metrics_path: /federate
    honor_labels: true
    scrape_interval: 30s
    scrape_timeout: 20s
    params:
      'match[]':
        - '{job="node-exporter"}'
        - '{job="external-node-exporter"}'
        - '{__name__=~"kube_deployment_(spec_replicas|status_replicas_ready|status_replicas_available|status_replicas_unavailable)"}'
        - '{__name__="kube_pod_container_resource_limits"}'
        - '{__name__=~"container_memory_working_set_bytes|container_cpu_usage_seconds_total"}'
    static_configs:
      - targets:
          - prometheus-a.example.com:443
        labels:
          federated_cluster: cluster-a
    tls_config:
      insecure_skip_verify: false
```

域名和 job 按实际修改。`targets` 不写 `https://`，协议由 `scheme` 决定。多个 `match[]` 取并集。上层可抓聚合规则结果或必要原始指标，不需要默认抓取全部时序。Federation 是周期性抓取当前样本，不会同步下层全部历史或在故障恢复后回补历史。参见 [Federation 官方文档](https://prometheus.io/docs/prometheus/latest/federation/)。

上层配置校验：

```bash
promtool check config /etc/prometheus/prometheus.yml
```

配置有效后通过现有进程管理方式重载；只有启动了 `--web.enable-lifecycle` 时，`POST /-/reload` 才可用。

---

## 十、上层使用 kube-prometheus-stack

同一 job 放到上层 values 的以下位置，不带外层 `scrape_configs`：

```yaml
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: federate-cluster-a
        scheme: https
        metrics_path: /federate
        honor_labels: true
        scrape_interval: 30s
        scrape_timeout: 20s
        params:
          'match[]':
            - '{job="node-exporter"}'
            - '{__name__="kube_deployment_status_replicas_ready"}'
        static_configs:
          - targets:
              - prometheus-a.example.com:443
            labels:
              federated_cluster: cluster-a
        tls_config:
          insecure_skip_verify: false
```

这是最小连通性示例；需要其他指标时，沿用上一节的筛选列表。Helm 列表会整体替换，已有 additionalScrapeConfigs 必须合并到同一个列表，避免覆盖其他抓取任务。

---

## 十一、内部 CA 和认证

公网受信 CA 通常可使用系统信任库。内部 CA 应在上层 Prometheus 挂载证书：

```bash
kubectl create secret generic federation-ca -n monitoring \
  --from-file=ca.crt=./ca.crt \
  --dry-run=client -o yaml | kubectl apply -f -
```

上层 Stack values 增加：

```yaml
prometheus:
  prometheusSpec:
    secrets:
      - federation-ca
```

对应 job 的 TLS 配置替换为：

```yaml
tls_config:
  ca_file: /etc/prometheus/secrets/federation-ca/ca.crt
  server_name: prometheus-a.example.com
  insecure_skip_verify: false
```

原生 Prometheus 使用自己环境中实际挂载的 CA 路径。通过 IP 连接 HTTPS 时，server_name 用于证书名称验证和 TLS SNI，但不等于设置 HTTP Host；基于域名路由的网关优先使用正确域名和 DNS。

如果入口有 Basic Auth，把密码放入已挂载 Secret 文件，然后在 job 中配置：

```yaml
basic_auth:
  username: prometheus-federation
  password_file: /etc/prometheus/secrets/federation-auth/password
```

`federation-auth` 也需要纳入 `prometheusSpec.secrets`。临时排查可以使用 `insecure_skip_verify: true`，但生产配置应恢复证书验证。

---

## 十二、联邦验证和 externalLabels

从上层可访问的网络测试：

```bash
curl --fail --show-error -G \
  'https://prometheus-a.example.com/federate' \
  --data-urlencode 'match[]={__name__="kube_deployment_status_replicas_ready"}'
```

内部 CA 增加 `--cacert ./ca.crt`。

上层查询：

```promql
up{job="federate-cluster-a"}
```

它表示联邦 HTTP 抓取是否成功，不代表每一个下层目标都是 UP，也不保证筛选到了所需指标。

```promql
kube_deployment_status_replicas_ready{federated_cluster="cluster-a"}
```

```promql
node_uname_info{federated_cluster="cluster-a"}
```

下层 `externalLabels.cluster` 用于 Federation 等对外通信，不会自动写入下层 TSDB 的每条本地序列。因此本地 `up{cluster="cluster-a"}` 查不到，不能直接说明采集失败。对外标签还会保留样本原有同名值，避免设置冲突标签。

`honor_labels: true` 保留下层的 job、instance 等标签；发生同名冲突时，下层样本标签优先。`federated_cluster` 需约定不与下层标签冲突。

---

## 十三、验收清单

- 独立 Node Exporter 没有被第二套 DaemonSet 重复部署。
- 新 ServiceMonitor 被下层 Prometheus 选中，每个预期节点有目标。
- job 和 instance 已按预期改变，旧面板和默认规则同步适配。
- 下层所需指标存在，HTTPS `/federate` 能精确返回它们。
- 上层联邦目标正常，并能按子集群标签查询业务指标。
- 自定义证书验证通过，实际入口的认证和来源限制符合现有环境。
