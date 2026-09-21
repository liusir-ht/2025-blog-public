# Kubernetes NGINX Gateway Fabric 生产环境优化

> 本文用于补充 NGINX Gateway Fabric 上线后的生产调整需求，主要记录 Data Plane 扩容、腾讯云已有 CLB 复用、HTTPRoute 白名单、日志格式调整、logAgent Sidecar 接入、ConfigMap 挂载、容器权限以及时区等问题。

---

## 1. 环境说明

当前环境：

```text
Kubernetes:              1.34
NGINX Gateway Fabric:    2.7.x
Gateway API:             1.6.x
```

整体架构：

```text
                    Tencent CLB
                         │
                         ▼
                    Gateway Service
                         │
                         ▼
                NGINX Data Plane
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
          HTTPRoute A           HTTPRoute B
              │                     │
              ▼                     ▼
          Service A             Service B
```

NGINX Gateway Fabric 中需要始终区分：

```text
Control Plane
    └── NGINX Gateway Fabric Controller

Data Plane
    └── Gateway 对应的 NGINX Deployment
```

业务请求不会经过 NGF Controller，真正承载流量的是 Gateway 对应的 NGINX Data Plane。

---

## 2. Gateway Data Plane 扩容

如果 Gateway 生成的数据面 Deployment 为：

```text
nginx-public-gateway
```

临时压测或者紧急扩容可以直接执行：

```bash
kubectl scale deployment nginx-public-gateway \
  -n gateway-system \
  --replicas=4
```

查看：

```bash
kubectl get deployment nginx-public-gateway -n gateway-system
kubectl get pods -n gateway-system
```

需要明确：

```text
提高业务并发
    ↓
扩容 NGINX Data Plane

提高控制面高可用
    ↓
扩容 NGF Controller
```

真正处理 HTTP/HTTPS/WebSocket/SSE 请求的是 Data Plane。

---

## 3. NginxProxy 是什么资源

`NginxProxy` 是 NGINX Gateway Fabric 提供的 CRD，不属于 Gateway API 标准资源。

标准 Gateway API：

```text
gateway.networking.k8s.io
├── GatewayClass
├── Gateway
├── HTTPRoute
└── ReferenceGrant
```

NGF 自定义资源：

```text
gateway.nginx.org
├── NginxProxy
├── SnippetsFilter
├── SnippetsPolicy
├── ProxySettingsPolicy
└── UpstreamSettingsPolicy
```

可以简单理解：

```text
Gateway API
    ↓
控制流量怎么走

NginxProxy
    ↓
控制 NGINX Data Plane 怎么运行
```

例如：

```text
副本数
Service 类型
资源 requests/limits
Access Log 格式
Deployment Patch
Service Patch
```

都属于 NginxProxy 的职责范围。

---

## 4. 使用已有腾讯云 CLB

如果 Gateway 创建出来的 Service 使用：

```yaml
type: LoadBalancer
```

并希望复用已有腾讯云 CLB，可以使用：

```yaml
service.kubernetes.io/tke-existed-lbid: "lb-xxxxxxxx"
```

例如：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public-gateway
  namespace: gateway-system
spec:
  gatewayClassName: nginx

  infrastructure:
    annotations:
      service.kubernetes.io/tke-existed-lbid: "lb-xxxxxxxx"

  listeners:
  - name: http
    protocol: HTTP
    port: 80

  - name: https
    protocol: HTTPS
    port: 443
```

最终链路：

```text
Gateway
   │
   ▼
Gateway Service
   │
   │ tke-existed-lbid
   ▼
Existing Tencent CLB
   │
   ▼
NGINX Data Plane
```

检查：

```bash
kubectl get svc -n gateway-system -o yaml
```

重点确认：

```yaml
metadata:
  annotations:
    service.kubernetes.io/tke-existed-lbid: lb-xxxxxxxx
```

如果多个 Service 复用同一个 CLB，还需要额外确认腾讯云关于共享 CLB、监听器端口以及生命周期管理的限制。

---

## 5. 只针对某个 HTTPRoute 配置 IP 白名单

一个 Gateway 下可能存在多个 Route：

```text
                 public-gateway
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
      HTTPRoute A             HTTPRoute B
      需要白名单              正常公网访问
```

如果直接在 CLB 配白名单：

```text
CLB
 ↓
所有 Route 都受影响
```

因此不能满足“只限制某一个 HTTPRoute”的需求。

---

## 6. SnippetsFilter 实现 Route 级白名单

首先需要启用 snippets：

```yaml
nginxGateway:
  snippets:
    enable: true
```

创建：

```yaml
apiVersion: gateway.nginx.org/v1alpha1
kind: SnippetsFilter
metadata:
  name: route-ip-allowlist
  namespace: app-prod
spec:
  snippets:
  - context: http.server.location
    value: |
      allow 192.0.2.10/32;
      allow 198.51.100.0/24;
      deny all;
```

### context 注意

下面这种写法是错误的：

```yaml
context: http.location
```

支持的 context：

```text
main
http
http.server
http.server.location
```

针对某个 HTTPRoute 对应的 location，应使用：

```yaml
context: http.server.location
```

---

## 7. HTTPRoute 引用 SnippetsFilter

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: restricted-route
  namespace: app-prod
spec:
  parentRefs:
  - name: public-gateway
    namespace: gateway-system

  hostnames:
  - restricted.example.com

  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /

    filters:
    - type: ExtensionRef
      extensionRef:
        group: gateway.nginx.org
        kind: SnippetsFilter
        name: route-ip-allowlist

    backendRefs:
    - name: restricted-service
      port: 8080
```

效果：

```text
public-gateway
│
├── restricted.example.com
│      └── HTTPRoute
│           └── SnippetsFilter
│                ├── allow
│                └── deny all
│
└── public.example.com
       └── HTTPRoute
            └── 不受影响
```

验证：

```bash
kubectl describe snippetsfilter route-ip-allowlist -n app-prod
kubectl describe httproute restricted-route -n app-prod
```

重点检查：

```text
Accepted=True
ResolvedRefs=True
```

---

## 8. SnippetsFilter 和 SnippetsPolicy 的区别

### SnippetsFilter

作用范围：

```text
HTTPRoute / GRPCRoute rule
```

典型用途：

```text
Route 级 IP 白名单
Route 特殊 NGINX location 配置
```

### SnippetsPolicy

作用范围：

```text
Gateway
+
挂载到该 Gateway 的 Route
```

典型用途：

```text
统一日志
全局 NGINX 配置
Gateway 级配置
```

简单记：

```text
SnippetsFilter = Route 级
SnippetsPolicy = Gateway 级
```

Snippets 可以直接注入原生 NGINX 配置，应尽量作为补充能力使用，优先考虑 Gateway API 和 NGF 的一等 Policy。

---

## 9. 修改 NGINX Access Log 格式

原日志格式：

```nginx
log_format main escape=json '$status $remote_addr - - [$time_local] '
                            '"$request" $request_time $body_bytes_sent $hostname "$http_referer" '
                            '"$http_user_agent" "$http_x_forwarded_for"|Host: $http_host '
                            '|Appid: $http_x_appid '
                            '|TimeStamp: $http_x_timestamp '
                            '|X-RtmId: $http_x_rtmid '
                            '|body: $request_body';
```

NGF 中可以通过 `NginxProxy`：

```yaml
apiVersion: gateway.nginx.org/v1alpha2
kind: NginxProxy
metadata:
  name: public-gateway-proxy
  namespace: gateway-system
spec:
  logging:
    accessLog:
      escape: json
      format: '$status $remote_addr - - [$time_local] "$request" $request_time $body_bytes_sent $hostname "$http_referer" "$http_user_agent" "$http_x_forwarded_for"|Host: $http_host |Appid: $http_x_appid |TimeStamp: $http_x_timestamp |X-RtmId: $http_x_rtmid |body: $request_body'
```

HTTP Header 对应变量建议使用小写：

```text
X-AppId      → $http_x_appid
X-TimeStamp  → $http_x_timestamp
X-RtmId      → $http_x_rtmid
```

为了方便排查 TTFB 和 upstream 延迟，也可以增加：

```text
$upstream_addr
$upstream_status
$upstream_connect_time
$upstream_header_time
$upstream_response_time
```

例如：

```yaml
format: '$status $remote_addr - - [$time_local] "$request" $request_time $body_bytes_sent $hostname "$http_referer" "$http_user_agent" "$http_x_forwarded_for"|Host: $http_host |Appid: $http_x_appid |TimeStamp: $http_x_timestamp |X-RtmId: $http_x_rtmid |Upstream: $upstream_addr |UpstreamStatus: $upstream_status |ConnectTime: $upstream_connect_time |HeaderTime: $upstream_header_time |UpstreamTime: $upstream_response_time'
```

### request_body 注意事项

`$request_body` 可能包含：

```text
Token
Password
用户数据
敏感业务参数
```

生产环境需要确认是否真的需要长期记录完整 Request Body。

---

## 10. NGF Access Log 默认写 stdout

### 修改资源

```text
NginxProxy
```

### 修改字段

```text
spec.logging.accessLog
```

NGF 原生 Access Log 配置示例：

```yaml
apiVersion: gateway.nginx.org/v1alpha2
kind: NginxProxy
metadata:
  name: public-gateway-proxy
  namespace: gateway-system
spec:
  logging:
    accessLog:
      escape: json
      format: '$status $remote_addr - - [$time_local] "$request" $request_time'
```

这部分控制的是 NGINX Data Plane 的标准 Access Log 格式，默认输出到：

```text
/dev/stdout
```

如果 logAgent 必须读取真实日志文件，则不能只改这里，需要组合使用：

```text
SnippetsPolicy
+
ConfigMap
+
NginxProxy
```

---

## 11. logAgent 文件采集：先看需要改哪些资源

这一部分一共涉及 3 个资源。

| 需求 | 修改资源 |
|---|---|
| 让 NGINX 写 `/data/log/nginx/access.log` | `SnippetsPolicy` |
| 提供 `/usr/local/fpnn/logAgent.conf` | `ConfigMap` |
| 创建 Volume、挂载 nginx、增加 logAgent Sidecar | `NginxProxy` |

整体关系：

```text
SnippetsPolicy
    ↓
NGINX 写文件
    ↓
/data/log/nginx/access.log
    ↓
NginxProxy 创建 shared emptyDir
    ↓
logAgent Sidecar
    ↓
ConfigMap 提供 logAgent.conf
```

最终 Pod：

```text
nginx-public-gateway Pod
│
├── nginx
│     └── /data/log/nginx/access.log
│
├── logagent
│     ├── /data/service/logAgent
│     ├── /usr/local/fpnn/logAgent.conf
│     └── /data/log/nginx/access.log
│
└── volumes
      ├── nginx-log          emptyDir
      └── log-agent-config   ConfigMap
```

---

## 12. 第一步：在 SnippetsPolicy 中增加文件 Access Log

### 修改资源

```text
SnippetsPolicy
```

### 作用对象

```text
Gateway
```

### 修改内容

让整个 Gateway 对应的 NGINX Data Plane 额外写：

```text
/data/log/nginx/access.log
```

示例：

```yaml
apiVersion: gateway.nginx.org/v1alpha1
kind: SnippetsPolicy
metadata:
  name: gateway-file-access-log
  namespace: gateway-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: public-gateway

  snippets:
  - context: http
    value: |
      log_format logagent escape=json '$status $remote_addr - - [$time_local] "$request" $request_time $body_bytes_sent $hostname "$http_referer" "$http_user_agent" "$http_x_forwarded_for"|Host: $http_host |Appid: $http_x_appid |TimeStamp: $http_x_timestamp |X-RtmId: $http_x_rtmid';
      access_log /data/log/nginx/access.log logagent;
```

这里需要记住：

```text
文件 access_log
    ↓
SnippetsPolicy
```

而不是写在 `NginxProxy.spec.logging.accessLog` 里。

应用：

```bash
kubectl apply -f snippets-policy.yaml
```

验证：

```bash
kubectl get snippetspolicy -n gateway-system
kubectl describe snippetspolicy gateway-file-access-log -n gateway-system
```

---

## 13. 第二步：创建 logAgent ConfigMap

### 修改资源

```text
ConfigMap
```

### 目标文件

```text
/usr/local/fpnn/logAgent.conf
```

示例：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: log-agent-config
  namespace: gateway-system
data:
  logAgent.conf: |
    # logAgent 配置
    # NGINX 日志文件：
    # /data/log/nginx/access.log
```

应用：

```bash
kubectl apply -f log-agent-config.yaml
```

检查：

```bash
kubectl get configmap log-agent-config \
  -n gateway-system \
  -o yaml
```

这一步只是创建配置文件，真正挂载到 Sidecar 仍然是在 `NginxProxy` 里完成。

---

## 14. 第三步：在 NginxProxy 中创建共享 Volume

### 修改资源

```text
NginxProxy
```

### 修改字段

```text
spec.kubernetes.deployment.pod.volumes
```

需要创建两个 Volume：

```text
nginx-log
    ↓
nginx 和 logAgent 共享日志目录

log-agent-config
    ↓
挂载 ConfigMap
```

示例：

```yaml
apiVersion: gateway.nginx.org/v1alpha2
kind: NginxProxy
metadata:
  name: public-gateway-proxy
  namespace: gateway-system
spec:
  kubernetes:
    deployment:
      pod:
        volumes:
        - name: nginx-log
          emptyDir: {}

        - name: log-agent-config
          configMap:
            name: log-agent-config
```

这里只是声明 Volume，还没有挂载到 Container。

---

## 15. 第四步：在 NginxProxy 中给 nginx 挂载日志目录

### 修改资源

```text
NginxProxy
```

### 修改对象

```text
NGF 已生成的 nginx Container
```

### 推荐方式

```text
StrategicMerge
```

### 修改字段

```text
spec.kubernetes.deployment.patches
```

示例：

```yaml
spec:
  kubernetes:
    deployment:
      patches:
      - type: StrategicMerge
        value:
          spec:
            template:
              spec:
                containers:
                - name: nginx
                  volumeMounts:
                  - name: nginx-log
                    mountPath: /data/log/nginx
```

效果：

```text
nginx Container
    ↓
/data/log/nginx
    ↓
nginx-log emptyDir
```

---

## 16. 第五步：在 NginxProxy 中增加 logAgent Sidecar

### 修改资源

```text
NginxProxy
```

### 修改字段

```text
spec.kubernetes.deployment.patches
```

### 推荐方式

```text
JSONPatch
```

因为当前是新增 Sidecar，而不是修改 nginx。

示例：

```yaml
spec:
  kubernetes:
    deployment:
      patches:
      - type: JSONPatch
        value:
        - op: add
          path: /spec/template/spec/containers/-
          value:
            name: logagent
            image: registry.example.com/logagent:latest

            command:
            - /bin/bash
            - -c

            args:
            - exec /data/service/logAgent /usr/local/fpnn/logAgent.conf

            securityContext:
              privileged: false
              runAsNonRoot: false
              runAsUser: 0

            volumeMounts:
            - name: nginx-log
              mountPath: /data/log/nginx

            - name: log-agent-config
              mountPath: /usr/local/fpnn/logAgent.conf
              subPath: logAgent.conf
```

关键字段：

```yaml
path: /spec/template/spec/containers/-
```

表示：

```text
向 containers 数组末尾追加一个 Container
```

最终：

```text
containers
├── nginx
└── logagent
```

---

## 17. logAgent 启动命令应该改哪里

### 修改资源

```text
NginxProxy
```

### 修改位置

```text
JSONPatch 中新增的 logagent Container
```

实际启动命令：

```bash
/bin/bash -c "/data/service/logAgent /usr/local/fpnn/logAgent.conf"
```

Kubernetes 配置：

```yaml
command:
- /bin/bash
- -c

args:
- exec /data/service/logAgent /usr/local/fpnn/logAgent.conf
```

推荐使用：

```text
exec
```

让 logAgent 成为容器 PID 1，更利于处理 SIGTERM。

---

## 18. logAgent 不使用端口和探针，应该改哪里

### 修改资源

```text
NginxProxy
```

### 修改对象

```text
JSONPatch 中的 logagent Container
```

当前 logAgent 不使用：

```text
ports
readinessProbe
livenessProbe
```

因此直接不要声明这些字段即可。

只保留：

```text
image
command
args
securityContext
volumeMounts
```

---

## 19. logAgent root / privileged 应该改哪里

### 修改资源

```text
NginxProxy
```

### 修改位置

```text
JSONPatch 中的 logagent.securityContext
```

如果 logAgent 镜像默认以 root 运行：

```yaml
securityContext:
  privileged: false
  runAsNonRoot: false
  runAsUser: 0
```

含义：

```text
runAsUser: 0
    ↓
root 用户运行

runAsNonRoot: false
    ↓
允许 root

privileged: false
    ↓
不是特权容器
```

需要明确：

```text
root Container
≠
privileged Container
```

对于只读取日志、读取 ConfigMap、发送日志的 Agent，通常不需要 `privileged: true`。

---

## 20. NGINX 时区应该改哪里

### 修改资源

```text
NginxProxy
```

### 修改对象

```text
已有 nginx Container
```

### 推荐方式

```text
StrategicMerge
```

例如：

```yaml
spec:
  kubernetes:
    deployment:
      patches:
      - type: StrategicMerge
        value:
          spec:
            template:
              spec:
                containers:
                - name: nginx
                  env:
                  - name: TZ
                    value: Asia/Shanghai
```

检查：

```bash
kubectl get deploy nginx-public-gateway \
  -n gateway-system \
  -o jsonpath='{.spec.template.spec.containers[?(@.name=="nginx")].env}'
```

如果环境变量存在但时间仍是 UTC，则需要确认镜像内 tzdata / zoneinfo。

---

## 21. NGINX 时区不生效时，应该改哪里

仍然修改：

```text
NginxProxy
```

使用：

```text
StrategicMerge
```

给 Pod 增加 timezone Volume，并挂到 nginx：

```yaml
spec:
  kubernetes:
    deployment:
      patches:
      - type: StrategicMerge
        value:
          spec:
            template:
              spec:
                volumes:
                - name: timezone
                  hostPath:
                    path: /usr/share/zoneinfo/Asia/Shanghai
                    type: File

                containers:
                - name: nginx
                  volumeMounts:
                  - name: timezone
                    mountPath: /etc/localtime
                    readOnly: true
```

验证：

```bash
kubectl exec -n gateway-system \
  <gateway-pod> \
  -c nginx \
  -- date
```

---

## 22. StrategicMerge 和 JSONPatch 怎么选

| 需求 | 修改资源 | Patch 类型 |
|---|---|---|
| 修改已有 nginx Container | `NginxProxy` | `StrategicMerge` |
| 给 nginx 加 env | `NginxProxy` | `StrategicMerge` |
| 给 nginx 加 volumeMount | `NginxProxy` | `StrategicMerge` |
| 新增 logAgent Sidecar | `NginxProxy` | `JSONPatch` |
| 修改 logAgent command/args | `NginxProxy` | 跟随 JSONPatch |
| 修改 logAgent securityContext | `NginxProxy` | 跟随 JSONPatch |

简单记：

```text
修改现有 nginx
    ↓
StrategicMerge

新增 sidecar
    ↓
JSONPatch
```

---

## 23. 推荐的 NginxProxy 完整示例

下面把 logAgent 相关的 Pod 定制集中在一个资源里。

> 凡是 Pod、Container、Volume、Sidecar 相关修改，都在 `NginxProxy` 中完成。

```yaml
apiVersion: gateway.nginx.org/v1alpha2
kind: NginxProxy
metadata:
  name: public-gateway-proxy
  namespace: gateway-system
spec:

  logging:
    accessLog:
      escape: json
      format: '$status $remote_addr - - [$time_local] "$request" $request_time $body_bytes_sent $hostname "$http_referer" "$http_user_agent" "$http_x_forwarded_for"|Host: $http_host |Appid: $http_x_appid |TimeStamp: $http_x_timestamp |X-RtmId: $http_x_rtmid'

  kubernetes:
    deployment:

      pod:
        volumes:
        - name: nginx-log
          emptyDir: {}

        - name: log-agent-config
          configMap:
            name: log-agent-config

      patches:

      # 修改 NGF 已有 nginx Container
      - type: StrategicMerge
        value:
          spec:
            template:
              spec:
                containers:
                - name: nginx
                  volumeMounts:
                  - name: nginx-log
                    mountPath: /data/log/nginx

      # 新增 logAgent Sidecar
      - type: JSONPatch
        value:
        - op: add
          path: /spec/template/spec/containers/-
          value:
            name: logagent
            image: registry.example.com/logagent:latest

            command:
            - /bin/bash
            - -c

            args:
            - exec /data/service/logAgent /usr/local/fpnn/logAgent.conf

            securityContext:
              privileged: false
              runAsNonRoot: false
              runAsUser: 0

            volumeMounts:
            - name: nginx-log
              mountPath: /data/log/nginx

            - name: log-agent-config
              mountPath: /usr/local/fpnn/logAgent.conf
              subPath: logAgent.conf
```

这个 `NginxProxy` 负责：

```text
NginxProxy
├── stdout Access Log 格式
├── nginx-log emptyDir
├── log-agent-config Volume
├── nginx 日志目录挂载
└── logAgent Sidecar
```

但还有两个资源需要单独维护：

```text
SnippetsPolicy
    ↓
负责文件 access_log

ConfigMap
    ↓
负责 logAgent.conf 内容
```

---

## 24. 推荐最终文件拆分

```text
nginx-gateway/
├── gateway.yaml
├── httproute.yaml
├── nginxproxy.yaml
├── snippets-policy.yaml
└── log-agent-config.yaml
```

对应关系：

```text
gateway.yaml
    ↓
Gateway / Listener / TLS

httproute.yaml
    ↓
业务路由

nginxproxy.yaml
    ↓
Data Plane Pod / Container / Volume / logAgent Sidecar

snippets-policy.yaml
    ↓
NGINX 文件 Access Log

log-agent-config.yaml
    ↓
logAgent.conf
```

---

## 25. 一眼看懂：需求应该修改哪个资源

| 调整需求 | 修改资源 | 关键位置 |
|---|---|---|
| 修改 NGINX stdout 日志格式 | `NginxProxy` | `spec.logging.accessLog` |
| 额外写文件 Access Log | `SnippetsPolicy` | `spec.snippets` |
| 创建共享日志目录 | `NginxProxy` | `spec.kubernetes.deployment.pod.volumes` |
| 给 nginx 挂日志目录 | `NginxProxy` | `StrategicMerge` |
| 创建 logAgent 配置文件 | `ConfigMap` | `data.logAgent.conf` |
| 新增 logAgent Sidecar | `NginxProxy` | `JSONPatch` |
| 修改 logAgent 启动命令 | `NginxProxy` | JSONPatch 中 `command/args` |
| 修改 logAgent root/privileged | `NginxProxy` | JSONPatch 中 `securityContext` |
| 不配置 logAgent port/probe | `NginxProxy` | JSONPatch 中不声明 |
| 修改 nginx TZ | `NginxProxy` | `StrategicMerge` |
| 单 HTTPRoute IP 白名单 | `SnippetsFilter` | HTTPRoute `ExtensionRef` |
| 整个 Gateway NGINX 扩展 | `SnippetsPolicy` | `targetRefs -> Gateway` |

---

## 26. 推荐验证命令

检查 NginxProxy：

```bash
kubectl get nginxproxy public-gateway-proxy \
  -n gateway-system \
  -o yaml
```

检查 SnippetsPolicy：

```bash
kubectl get snippetspolicy gateway-file-access-log \
  -n gateway-system \
  -o yaml
```

检查 ConfigMap：

```bash
kubectl get configmap log-agent-config \
  -n gateway-system \
  -o yaml
```

检查 Data Plane 容器：

```bash
kubectl get deploy nginx-public-gateway \
  -n gateway-system \
  -o jsonpath='{.spec.template.spec.containers[*].name}'
```

预期：

```text
nginx logagent
```

查看 logAgent 最终配置：

```bash
kubectl get deploy nginx-public-gateway \
  -n gateway-system \
  -o json \
| jq '.spec.template.spec.containers[] | select(.name=="logagent")'
```

查看文件日志：

```bash
kubectl exec -it \
  -n gateway-system \
  <gateway-pod> \
  -c nginx \
  -- tail -f /data/log/nginx/access.log
```

查看 logAgent 配置：

```bash
kubectl exec -it \
  -n gateway-system \
  <gateway-pod> \
  -c logagent \
  -- cat /usr/local/fpnn/logAgent.conf
```

---

## 27. 生产环境注意事项

### Snippets 权限

Snippets 可以直接注入 NGINX 配置，应限制普通业务账号对 `SnippetsFilter`、`SnippetsPolicy` 的写权限。

### 日志重复

如果同时开启：

```text
stdout access log
+
file access log
```

同一请求会出现两份日志，需要关注日志量、存储成本和重复采集。

### Request Body

不要默认长期记录：

```text
$request_body
```

需要确认是否包含 Token、密码、隐私信息或其他敏感字段。

### Sidecar 扩容

Gateway Data Plane 扩容后：

```text
NGINX Pod x N
=
logAgent x N
```

因此也要评估 logAgent CPU、Memory、日志出口带宽和日志平台写入压力。

### emptyDir

```text
Pod 删除
    ↓
emptyDir 删除
```

logAgent 应及时发送日志，不能把 `emptyDir` 当持久化日志目录。

---

## 28. 总结

NGINX Gateway Fabric 上线后，很多 ingress-nginx 时代通过 ConfigMap、Annotation、直接改 Controller Deployment 完成的事情，会拆分到不同资源。

最重要的是先判断“我要改什么”：

```text
改入口
    ↓
Gateway

改路由
    ↓
HTTPRoute

改 NGINX Data Plane / Pod / Container
    ↓
NginxProxy

改单 Route 的 NGINX 行为
    ↓
SnippetsFilter

改整个 Gateway 的 NGINX 原生配置
    ↓
SnippetsPolicy

提供 logAgent.conf
    ↓
ConfigMap
```

对于当前 logAgent 场景：

```text
SnippetsPolicy
    ↓
让 NGINX 写文件

ConfigMap
    ↓
提供 logAgent.conf

NginxProxy
    ↓
创建 Volume
挂载 nginx
新增 logAgent Sidecar
配置 command / securityContext
```

---

## 29. 参考资料

- NGINX Gateway Fabric  
  https://docs.nginx.com/nginx-gateway-fabric/

- Data Plane Configuration  
  https://docs.nginx.com/nginx-gateway-fabric/how-to/data-plane-configuration/

- Snippets  
  https://docs.nginx.com/nginx-gateway-fabric/traffic-management/snippets/

- API Reference  
  https://docs.nginx.com/nginx-gateway-fabric/reference/api/

- Deploy Gateway Data Plane  
  https://docs.nginx.com/nginx-gateway-fabric/install/deploy-data-plane/

- 腾讯云 TKE Service 使用已有 CLB  
  https://cloud.tencent.com/document/product/457/45491
