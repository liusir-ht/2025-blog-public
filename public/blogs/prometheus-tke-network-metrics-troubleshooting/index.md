
# Prometheus 在腾讯云 Kubernetes 中的网络与指标缺失排查

监控接入不成功，需要区分资源发现、网络连接、原始指标、入库过滤和上层联邦五个环节。本文整理部署过程中出现的问题，并保留已确认事实与待验证假设的边界。

| 问题 | 当前证据与结论 |
|---|---|
| ServiceMonitor 跨主机采集失败 | 已报告现象；未提供完整 Target 错误或抓包，根因尚未确认 |
| 腾讯云是否部署 kube-state-metrics | kubectl top 可用，不能据此确认 kube-state-metrics |
| container_spec_memory_limit_bytes 缺失 | 已看到 `drop container_spec.*`，过滤规则会命中该指标 |
| Node Exporter instance 含 9100 | 已给出目标 relabel 方案；上线结果需实际查询确认 |

---

## 一、先按故障层次分类

| 表现 | 优先检查 |
|---|---|
| Target 根本没有出现 | Prometheus 选择器、ServiceMonitor 选择器、端口名称、RBAC |
| Target 出现但 DOWN | 具体抓取错误、网络、监听、TLS、认证 |
| Target UP 但某指标没有 | 原始端点、metricRelabelings、Exporter 版本和配置 |
| 下层有，上层没有 | Federation match[]、标签冲突、上层过滤 |
| Prometheus 有，Grafana 没有 | 数据源、时间范围、变量、旧 job/instance 条件 |

ServiceMonitor 不负责转发流量。它声明采集规则，由 Operator 生成配置，最终 Prometheus Pod 直接访问目标。

---

## 二、跨节点失败：确认实际源和目的地址

```bash
kubectl get pod -n monitoring -l app.kubernetes.io/name=prometheus -o wide
kubectl get pod -n monitoring -l app.kubernetes.io/instance=node-exporter -o wide
kubectl get nodes -o wide
kubectl get svc -n monitoring -l app.kubernetes.io/instance=node-exporter
```

在 Prometheus Targets 复制失败的实际 URL 和错误信息。再查看对应 Service 后端：

```bash
kubectl get endpointslice -n monitoring \
  -l kubernetes.io/service-name=实际Service名称 -o yaml
```

Node Exporter 使用 hostNetwork 时，后端地址通常是节点 IP。但必须查看实际 DaemonSet，不能仅凭 Chart 名断定 hostNetwork/hostPort 设置。

```bash
kubectl get ds -n monitoring -l app.kubernetes.io/instance=node-exporter -o yaml
helm get values node-exporter -n monitoring --all
```

> hostNetwork 与 hostPort 不是同一个参数；使用宿主机网络不等于必须显式声明 hostPort。

“同节点 UP、跨节点 DOWN”说明跨节点路径值得优先排查，但不能直接证明安全组是根因。

---

## 三、在 Prometheus 的网络命名空间验证

先选择真实 Prometheus Pod。若有多个实例或副本，分别测试，不要只默认选第一项。

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus
```

镜像具备 wget 时，可直接测试失败 URL：

```bash
kubectl exec -n monitoring 实际PrometheusPod -c prometheus -- \
  wget -T 5 -qO- http://目标节点IP:9100/metrics
```

也可在具备权限时添加临时调试容器，共享该 Pod 网络命名空间：

```bash
kubectl debug -n monitoring -it pod/实际PrometheusPod \
  --image=curlimages/curl --target=prometheus -- \
  curl -v --connect-timeout 3 --max-time 10 http://目标节点IP:9100/metrics
```

调试容器需要集群支持、镜像可拉取及相应权限。单独新建的调试 Pod，即使位于同一命名空间或节点，也可能具有不同标签、ENI 和网络策略，结果只能作为辅助，不能等同原 Prometheus Pod。

同时测试同节点和跨节点目标，并记录：源 Pod IP、源 Node、目标 IP、目标端口、具体错误。

---

## 四、依据错误定位

| 错误 | 说明与下一步 |
|---|---|
| connect timeout | 检查路由、回程、安全组、ACL、节点防火墙、网络策略 |
| context deadline exceeded | 可能连接或响应阶段超时，结合 curl 分辨 |
| connection refused | 通常是无监听或主动 REJECT，不一定只是进程未启动 |
| x509 | CA、证书 SAN、域名或时间问题 |
| 401 / 403 | 认证、授权或网关访问策略 |
| 404 | 请求路径、入口路由或端点不匹配 |
| 200 但解析失败 | 可能返回 HTML 登录页、错误协议或非法指标格式 |

不能把所有超时都归因于 CNI，也不能把所有 404 都归因于云厂商限制。

---

## 五、腾讯云 CNI 与安全组排查

“腾讯云 CNI”不足以确定所有网络行为。需要进一步确认是否为 VPC-CNI/ENI、路由模式，是否启用了 Pod 安全组，以及访问节点时是否 SNAT。

目标节点观察到的源 IP 可能是 Pod IP，也可能是节点 IP。以抓包为依据决定安全组放行来源：

```bash
sudo ss -lntp 'sport = :9100'
sudo tcpdump -ni any 'tcp port 9100' -c 30
```

测试期间观察 TCP 握手：

| 目标侧抓包 | 下一步 |
|---|---|
| 看不到 SYN | 检查源端出口、VPC 路由、安全组和 ACL；仅目标抓包不能单独定根因 |
| SYN 到达但无 SYN-ACK | 检查目标监听、主机防火墙和响应路由 |
| SYN-ACK 发出，客户端未收到 | 检查回程、源端策略、非对称路由 |
| 握手成功，HTTP 慢或失败 | 检查 Exporter 处理、TLS/HTTP 配置和响应内容 |

根据实际源地址，只放行必要的来源到 TCP 9100。kubelet/cAdvisor 常用 10250，是另一条访问策略，不能只放通 9100 就认为容器指标也可抓取。

节点防火墙只读检查：

```bash
sudo iptables -nvL INPUT --line-numbers
sudo nft list ruleset
sudo firewall-cmd --state
```

按系统实际使用的防火墙工具选择，不要直接清空规则或禁用防火墙。

---

## 六、NetworkPolicy 检查注意事项

```bash
kubectl get networkpolicy -A
kubectl get networkpolicy -n monitoring -o yaml
```

检查选中 Prometheus Pod 的 Egress 策略和目标侧策略。策略支持程度及 hostNetwork 流量处理取决于 CNI。

**不要盲目新建只有 TCP 9100 的 Egress 策略。**如果此前 Pod 没有被任何出口策略选中，新策略可能首次将它置于出口隔离，导致 DNS、API Server、kubelet 和其他采集目标不通。应先检查现有策略，再按实际依赖补全允许范围。

不建议仅为了绕过问题就切换 Prometheus 或 Node Exporter 的 hostNetwork；这样会改变网络路径、端口占用或网络命名空间视角，应先定位根因。

---

## 七、如何确认已有 kube-state-metrics

```bash
kubectl get pods -A -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,IMAGE:.spec.containers[*].image' | grep -i kube-state-metrics
kubectl get deploy,svc -A | grep -i state-metrics
helm list -A
kubectl get servicemonitor -A
```

组件可能改名、托管或受权限限制；没有搜索结果只能作为线索。找到 Service 后，查看实际端口并转发，例如：

```bash
kubectl port-forward -n 实际命名空间 svc/实际Service名称 18080:8080
```

另一终端：

```bash
curl --fail --silent --show-error http://127.0.0.1:18080/metrics | grep '^kube_deployment_status_replicas_ready'
```

然后在下层 Prometheus 查询同名指标。组件存在且可访问，也不等于 Prometheus 已抓取，仍需正确的 ServiceMonitor 和选择器。

`kubectl top node` 使用 Metrics API，通常由 metrics-server 提供，与 kube-state-metrics 是不同链路。检查：

```bash
kubectl get apiservice v1beta1.metrics.k8s.io
```

已有 kube-state-metrics 能满足采集范围并稳定复用时，关闭 Stack 的重复部署：

```yaml
kubeStateMetrics:
  enabled: false
```

多个 kube-state-metrics 本身不会自动导致重复告警；重复时序取决于是否同时抓取，重复告警还取决于规则、标签和告警去重方式。

---

## 八、container_spec_memory_limit_bytes 缺失排查

这个指标由 cAdvisor 提供。先在下层查询其他容器指标：

```promql
container_memory_working_set_bytes
```

```promql
container_cpu_usage_seconds_total
```

如果都没有，检查 kubelet ServiceMonitor 是否启用了 cAdvisor：

```yaml
kubelet:
  enabled: true
  serviceMonitor:
    cAdvisor: true
```

查看实际资源：

```bash
kubectl get servicemonitor -A | grep kubelet
kubectl get servicemonitor -n monitoring 实际kubelet监控名称 -o yaml
```

在 Targets 中搜索 `/metrics/cadvisor`，不要只依赖固定 `job` 或 `metrics_path` 标签，因为这些标签可能被更改或没有保留。

---

## 九、直接查看 kubelet 原始指标

选择一个实际运行受内存限制业务容器的节点：

```bash
NODE_NAME='替换为实际节点名'
kubectl get --raw "/api/v1/nodes/${NODE_NAME}/proxy/metrics/cadvisor" \
  | grep '^container_spec_memory_limit_bytes'
```

该请求通过 API Server 代理访问 kubelet，能验证原始端点；它不等价于 Prometheus Pod 直连 kubelet 的网络测试。

- 原始端点有、下层没有：检查目标状态与 metricRelabelings。
- 原始端点也没有：再检查 kubelet/cAdvisor 版本、运行时和指标暴露设置。
- 403：检查当前身份的 `nodes/proxy` 权限。
- 404：检查节点名、路径和端点支持，不据此直接认定腾讯云限制。

---

## 十、已确认的默认过滤规则

此次实际看到：

```yaml
- action: drop
  regex: container_spec.*
  sourceLabels:
    - __name__
```

其中 `__name__` 是指标名。该规则会命中 `container_spec_memory_limit_bytes`、`container_spec_cpu_quota` 等指标，在 Prometheus 入库前丢弃。

Chart 配置中的对应位置通常是：

```yaml
kubelet:
  serviceMonitor:
    cAdvisorMetricRelabelings:
      - action: drop
        sourceLabels: [__name__]
        regex: container_spec.*
```

渲染后的 ServiceMonitor 字段则是 `spec.endpoints[].metricRelabelings`，不会保留 Helm values 的 `cAdvisorMetricRelabelings` 字段名。

默认说明指出，这组容器规格指标与 kube-state-metrics 有重叠，因此过滤以减少冗余。它不代表所有指标完全等价，也不代表 kubelet 停止生成。参考 [Chart 默认值](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/values.yaml)。

---

## 十一、为什么过滤，以及能节省什么

| 数据 | 来源与含义 |
|---|---|
| container_spec_memory_limit_bytes | cAdvisor 观察的运行时/cgroup 内存限制 |
| kube_pod_container_resource_limits | Kubernetes API 中声明的容器资源限制 |

日常容量和配置监控通常只需要使用量加声明的 limits；需要排查声明值与实际运行时限制差异时，两者都有价值。不能笼统说一个永远比另一个准确。

减少这些入库样本能降低 Head、WAL、TSDB 和后续远程传输负担。数值即使不变，周期性抓取仍会生成样本。

**metricRelabelings 在抓取响应之后执行。**因此它不会阻止 cAdvisor 生成指标，也不会节省 kubelet 到下层 Prometheus 的该部分 HTTP 响应流量；只有指标不入库后，上层联邦或 remote_write 才不会再传这些样本。

---

## 十二、保留其他默认过滤，恢复所需指标

先导出当前实际版本的默认 values：

```bash
helm show values prometheus-community/kube-prometheus-stack \
  --version "$STACK_CHART_VERSION" > stack-default-values.yaml
```

推荐把完整 `kubelet.serviceMonitor.cAdvisorMetricRelabelings` 列表复制到自己的 values，保留其他过滤，删除 `regex: container_spec.*` 这一条。这样会恢复整个 container_spec 指标族。

如果只需要内存 limit，可以把原规则替换为下列两条，其余规则原样保留：

```yaml
# 第一步：把需要保留的指标名临时映射为空，其他指标名保持不变。
- action: replace
  sourceLabels: [__name__]
  regex: '(container_spec_memory_limit_bytes)|(.+)'
  replacement: '$2'
  targetLabel: __tmp_spec_filter

# 第二步：仍然丢弃其他 container_spec 指标。
- action: drop
  sourceLabels: [__tmp_spec_filter]
  regex: 'container_spec.*'

- action: labeldrop
  regex: '__tmp_spec_filter'
```

精确指标命中第一个分支时 `$2` 为空，因此不会被第二步删除；其他 container_spec 指标命中第二个分支，仍被 drop。必须替换原 drop 规则，不能仅追加到它后面；已经丢弃的样本无法被后续规则恢复。

> 上述仅为替换片段。Helm 数组整体替换，不会逐条合并，必须带上要保留的完整默认列表和现有自定义规则。

另一种简化方式：

```yaml
kubelet:
  serviceMonitor:
    cAdvisorMetricRelabelings: []
```

这会移除该列表中的全部默认过滤，恢复更多无关指标，不适合作为只恢复一个指标的默认方案。实际是否完全移除还要检查渲染结果及其他附加规则。

---

## 十三、升级与恢复验证

```bash
helm template monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --version "$STACK_CHART_VERSION" \
  -f stack-values.yaml > stack-rendered.yaml

helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --version "$STACK_CHART_VERSION" \
  -f stack-values.yaml --wait --timeout 15m

kubectl get servicemonitor -n monitoring 实际kubelet监控名称 -o yaml
```

检查 `/metrics/cadvisor` 对应 endpoint 下的 `metricRelabelings`。等待配置重载和至少一个成功抓取周期，再在下层查询：

```promql
count by (job) (container_spec_memory_limit_bytes)
```

之前丢弃的历史不会恢复。若上层仍无数据，在既有 Federation 的 match[] 列表中增加：

```yaml
- '{__name__="container_spec_memory_limit_bytes"}'
```

---

## 十四、直接用 kube-state-metrics 的 limits 计算使用率

先查询内存限制：

```promql
kube_pod_container_resource_limits{resource="memory",unit="byte"}
```

在单个下层集群中，计算每个普通业务容器的工作集占内存 limit 百分比：

```promql
100 *
max by (namespace, pod, container) (
  container_memory_working_set_bytes{
    namespace!="", pod!="", container!="", container!="POD"
  }
)
/
max by (namespace, pod, container) (
  kube_pod_container_resource_limits{resource="memory",unit="byte"} > 0
)
```

这里使用 max 对同一容器的重复采集副本做防御性去重，避免直接 sum 把同一容器重复加总。仍需排查并消除不必要的重复抓取。

在上层多集群查询时，两边聚合维度都必须加 `federated_cluster`，或你实际统一的集群标签，不能把不同集群的同名 Pod 合并。

CPU limit 使用率：

```promql
100 *
max by (namespace, pod, container) (
  rate(container_cpu_usage_seconds_total{
    namespace!="", pod!="", container!="", container!="POD"
  }[5m])
)
/
max by (namespace, pod, container) (
  kube_pod_container_resource_limits{resource="cpu",unit="core"} > 0
)
```

CPU 公式假设使用的是每容器总 CPU 序列；若暴露 cpu 核维度，需要先按实际标签确认并汇总各核，再对重复采集源去重。没有声明 limit 的容器不会返回使用率。工作集占 limit 的百分比是监控口径，并不完全等同于 OOM 判定条件。

---

## 十五、上下层指标不一致的排查

依次检查：

1. 下层直接查询目标指标。
2. HTTPS `/federate` 按指标名精确查询。
3. 上层 Federation Target 的错误信息和 UP 状态。
4. 上层 job 中 match[] 是否匹配下层最终标签。
5. 上层 metric_relabel_configs 是否再次删除指标。
6. Grafana 是否仍使用旧的 job、含端口 instance 或错误数据源。

```bash
curl --fail --show-error -G \
  'https://prometheus-a.example.com/federate' \
  --data-urlencode 'match[]={__name__="container_spec_memory_limit_bytes"}'
```

如果下层的实际 `job` 被重写，或者原始业务标签落在 `exported_job` 中，联邦筛选要按下层实际存储标签填写。不要照搬一个与实际标签不匹配的正则。

---

## 十六、问题回顾与修正记录

| 原疑问或容易误解之处 | 明确结论 |
|---|---|
| ServiceMonitor 不能跨主机吗 | 可以，前提是目标可发现、可连接且认证正确 |
| 同节点通、跨节点不通一定是安全组吗 | 不是，需结合错误、源地址和双向抓包定位 |
| 下层用了 HTTPS 就要把 Node Exporter 改 HTTPS 吗 | 不需要，两条独立链路 |
| kubectl top 正常说明安装了 kube-state-metrics 吗 | 不能，只能说明 Metrics API 可用 |
| 默认 drop 意味着 kubelet 没有这个指标吗 | 不意味着，过滤发生在抓取后入库前 |
| 恢复过滤是否补回历史 | 不会，只影响后续采样 |
| externalLabels 是否出现在本地全部序列 | 不会自动加入本地每条序列 |
| 更改 instance 会改变抓取端口吗 | 只改 instance 不会；改 __address__ 才会影响实际地址 |
| 旧标签数据还在磁盘，即时查询也一直显示吗 | 不会，stale 后通常不再出现在即时结果中 |
| RE2 贪婪匹配等于回溯算法吗 | 不等于，先前回退描述仅可用于理解结果 |

---

## 十七、参考资料

- [参考博客：Kubernetes 生产级部署 Node Exporter](https://www.liuchong.tech/blog/kubernetes-node-exporter-production)
- [kube-prometheus-stack 配置](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/values.yaml)
- [Prometheus 配置与 relabel](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [Prometheus Operator API](https://prometheus-operator.dev/docs/api-reference/api/)
- [kube-state-metrics Pod 指标](https://github.com/kubernetes/kube-state-metrics/blob/main/docs/metrics/workload/pod-metrics.md)
- [cAdvisor Prometheus 指标](https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md)
