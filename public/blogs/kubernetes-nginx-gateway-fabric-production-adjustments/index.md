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
  name: office-ip-whitelist
  namespace: app-prod
spec:
  snippets:
  - context: http.server.location
    value: |
      allow xx.0.113.10/32;
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
  name: private-api
  namespace: app-prod
spec:
  parentRefs:
  - name: public-gateway
    namespace: gateway-system

  hostnames:
  - private.example.com

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
        name: office-ip-whitelist

    backendRefs:
    - name: private-service
      port: 8080
```

效果：

```text
public-gateway
│
├── private.example.com
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
kubectl describe snippetsfilter office-ip-whitelist -n app-prod
kubectl describe httproute private-api -n app-prod
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

NGF：

```yaml
spec:
  logging:
    accessLog:
```

当前日志目的地固定为：

```text
/dev/stdout
```

不能直接通过 `logging.accessLog` 改成：

```text
/data/log/nginx/access.log
```

如果日志 Agent 可以采 Kubernetes 容器日志，优先采 stdout。

但当前场景中 logAgent 必须指定 NGINX 日志文件，因此需要额外输出文件日志。

---

## 11. logAgent 必须读取文件日志

推荐结构：

```text
NGINX Data Plane Pod
│
├── nginx
│     └── /data/log/nginx/access.log
│
├── logagent
│     └── 读取 /data/log/nginx/access.log
│
└── emptyDir
      └── /data/log/nginx
```

优点：

```text
Gateway 扩容时 logAgent 自动一起扩容
每个 Pod 只采自己的 NGINX 日志
不存在多个 Pod 同写一个文件的问题
不依赖固定 Node
```

---

## 12. SnippetsPolicy 额外输出文件日志

可以额外定义一个文件 access log：

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

最终：

```text
NGINX
├── /dev/stdout
└── /data/log/nginx/access.log
```

即使 stdout 和文件日志格式完全相同，也建议文件日志单独维护自己的 `log_format` 名称，不依赖 NGF 内部生成的名称。

---

## 13. NGINX 和 logAgent 共享日志目录

推荐：

```yaml
spec:
  kubernetes:
    deployment:
      pod:
        volumes:
        - name: nginx-log
          emptyDir: {}
```

NGINX 和 logAgent 同时挂载：

```text
/data/log/nginx
```

需要注意：

```text
emptyDir 生命周期 = Pod 生命周期
```

Pod 删除后文件消失，所以 logAgent 应及时把日志发送出去。

---

## 14. logAgent ConfigMap

ConfigMap：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: log-agent-config
  namespace: gateway-system
data:
  logAgent.conf: |
    # logAgent 配置
    # 日志文件：
    # /data/log/nginx/access.log
```

挂载建议使用 `subPath`：

```yaml
volumeMounts:
- name: log-agent-config
  mountPath: /usr/local/fpnn/logAgent.conf
  subPath: logAgent.conf
```

这样不会覆盖整个：

```text
/usr/local/fpnn
```

目录。

---

## 15. 使用 JSONPatch 增加 logAgent Sidecar

对于已经存在的 NGINX Data Plane：

```text
containers:
- nginx
```

需要追加：

```text
- logagent
```

建议使用 `JSONPatch`：

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

启动效果：

```bash
/bin/bash -c "exec /data/service/logAgent /usr/local/fpnn/logAgent.conf"
```

使用 `exec` 可以让 logAgent 成为容器 PID 1，更利于 SIGTERM 处理。

---

## 16. logAgent 不需要 Port 和 Probe

当前 logAgent：

```text
不提供 HTTP/TCP Service
不使用 readinessProbe
不使用 livenessProbe
```

因此不需要：

```yaml
ports:
livenessProbe:
readinessProbe:
```

只保留：

```text
image
command
args
volumeMounts
securityContext
```

如果 logAgent 崩溃并直接退出：

```text
PID 1 exit
   ↓
Container Exit
   ↓
Kubelet 根据 restartPolicy 重启
```

如果程序可能出现“进程还存在但内部卡死”，没有 livenessProbe 时 Kubernetes 无法主动判断，需要结合程序特性评估。

---

## 17. root 用户和 privileged 容器的区别

曾遇到：

```text
container has runAsNonRoot and image will run as root
```

说明 logAgent 镜像默认以 root 运行，但 Pod/容器上下文要求非 root。

这和：

```yaml
privileged: false
```

不是一回事。

如果镜像当前必须 root：

```yaml
securityContext:
  privileged: false
  runAsNonRoot: false
  runAsUser: 0
```

表示：

```text
UID = 0
但容器仍然不是 privileged
```

两者区别：

```text
runAsUser: 0
    =
容器内部 root 用户

privileged: true
    =
大幅解除容器隔离和 Capability 限制
```

对于只读取日志文件、读取 ConfigMap 并发送日志的 Agent，一般不需要：

```yaml
privileged: true
```

后续如果程序支持普通 UID，建议继续收紧为非 root。

---

## 18. 挂载 /etc/localtime 设置 UTC+8

如果镜像内 `TZ` 不生效，可以将 Node 的：

```text
/usr/share/zoneinfo/Asia/Shanghai
```

挂到 NGINX Container：

```text
/etc/localtime
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
kubectl exec -n gateway-system   <gateway-pod>   -c nginx   -- date
```

预期时区：

```text
CST / +0800
```

NGINX `$time_local` 则会类似：

```text
21/Sep/2026:16:30:00 +0800
```

这种方法依赖 Node 上存在：

```text
/usr/share/zoneinfo/Asia/Shanghai
```

如果希望完全不依赖宿主机，长期更规范的方式是制作包含 tzdata 的自定义镜像。

---

## 19. StrategicMerge 和 JSONPatch 如何选择

NGF 的 NginxProxy Patch 支持：

```text
StrategicMerge
Merge
JSONPatch
```

### 修改已有 nginx Container

优先：

```text
StrategicMerge
```

例如：

```yaml
containers:
- name: nginx
  env:
  - name: TZ
    value: Asia/Shanghai
```

它可以按照 Container `name` 合并。

### 追加 logAgent Sidecar

优先：

```text
JSONPatch
```

例如：

```yaml
- op: add
  path: /spec/template/spec/containers/-
```

表示：

```text
向 containers 数组末尾新增一个容器
```

不会修改 NGF 已生成的 nginx 容器。

---

## 20. 当前推荐的完整架构

```text
                     Tencent CLB
                          │
                          ▼
                     Gateway
                          │
                          ▼
                 Gateway Service
                          │
                          ▼
             nginx-public-gateway Pod
             ┌─────────────────────────┐
             │                         │
             │  nginx                  │
Request ───► │    │                    │
             │    │ access_log         │
             │    ▼                    │
             │ /data/log/nginx/        │
             │ access.log              │
             │    │                    │
             │    ▼                    │
             │ logagent                │
             │    │                    │
             │    ▼                    │
             │ 日志平台                │
             │                         │
             └─────────────────────────┘
```

职责划分：

```text
Gateway / HTTPRoute
    ↓
流量入口和路由

NginxProxy
    ↓
Data Plane Deployment / Service / Logging

SnippetsFilter
    ↓
单个 Route 的 NGINX 扩展

SnippetsPolicy
    ↓
Gateway 级 NGINX 扩展

JSONPatch
    ↓
追加 logAgent Sidecar

StrategicMerge
    ↓
修改已有 NGINX Container

emptyDir
    ↓
共享文件日志

ConfigMap
    ↓
提供 logAgent.conf
```

---

## 21. 推荐验证命令

查看 Gateway：

```bash
kubectl get gateway -A
```

查看 NginxProxy：

```bash
kubectl get nginxproxy -A
```

查看 Snippets：

```bash
kubectl get snippetsfilter -A
kubectl get snippetspolicy -A
```

查看 Data Plane：

```bash
kubectl get deploy nginx-public-gateway   -n gateway-system   -o yaml
```

查看容器：

```bash
kubectl get deploy nginx-public-gateway   -n gateway-system   -o jsonpath='{.spec.template.spec.containers[*].name}'
```

预期：

```text
nginx logagent
```

查看 logAgent：

```bash
kubectl get deploy nginx-public-gateway   -n gateway-system   -o json | jq '.spec.template.spec.containers[] | select(.name=="logagent")'
```

查看 NGINX 文件日志：

```bash
kubectl exec -it   -n gateway-system   <gateway-pod>   -c nginx   -- ls -lh /data/log/nginx/
```

查看 logAgent 配置：

```bash
kubectl exec -it   -n gateway-system   <gateway-pod>   -c logagent   -- cat /usr/local/fpnn/logAgent.conf
```

查看时区：

```bash
kubectl exec -it   -n gateway-system   <gateway-pod>   -c nginx   -- date
```

---

## 22. 生产环境注意事项

### Snippets 权限

Snippets 可以直接注入 NGINX 配置，应限制：

```text
create
update
patch
delete
```

相关 RBAC 权限，避免普通业务账号任意修改。

### 日志重复

如果同时开启：

```text
stdout access log
+
file access log
```

同一个请求会出现两份日志，需要关注：

```text
日志存储量
采集成本
重复采集
```

### Request Body

不要默认长期记录：

```text
$request_body
```

应先确认是否包含：

```text
Token
Password
隐私信息
敏感业务字段
```

### Sidecar 扩容

Gateway Data Plane 扩容后：

```text
NGINX Pod x N
=
logAgent x N
```

因此扩容时也需要评估：

```text
logAgent CPU
logAgent Memory
日志出口带宽
日志平台写入压力
```

### emptyDir

```text
Pod 删除
    ↓
emptyDir 删除
```

日志 Agent 必须具备及时上传能力。

---

## 23. 总结

NGINX Gateway Fabric 上线后，很多 ingress-nginx 时代通过：

```text
ConfigMap
Annotation
直接修改 Controller Deployment
```

完成的配置，会逐渐拆分成：

```text
Gateway API
    ↓
标准入口与路由

NginxProxy
    ↓
数据面配置

NGF Policy / Filter
    ↓
NGINX 扩展

Deployment Patch
    ↓
Kubernetes 原生高级定制
```

当前生产实践可以总结为：

```text
提高并发
    ↓
扩 NGINX Data Plane

复用腾讯云 CLB
    ↓
Service Annotation

只限制单个 HTTPRoute
    ↓
SnippetsFilter

Gateway 级统一配置
    ↓
SnippetsPolicy

Access Log 格式
    ↓
NginxProxy logging

logAgent 文件采集
    ↓
SnippetsPolicy + emptyDir + Sidecar

追加 Sidecar
    ↓
JSONPatch

修改 nginx Container
    ↓
StrategicMerge

logAgent 镜像必须 root
    ↓
runAsUser: 0
privileged: false

NGINX 日志 UTC+8
    ↓
先检查 TZ
必要时挂载 /etc/localtime
```

这样既可以继续保留 NGINX 的日志与排障体系，也能把入口逐步迁移到 Gateway API。

---

## 24. 参考资料

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
