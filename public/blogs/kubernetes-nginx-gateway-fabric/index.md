# Kubernetes 1.34 部署 NGINX Gateway Fabric

## 1. 背景

`ingress-nginx` 退役后，如果仍然希望保留 NGINX 数据面、NGINX Access Log 以及现有的 NGINX 运维经验，可以考虑使用：

```text
Gateway API + NGINX Gateway Fabric
```

NGINX Gateway Fabric（NGF）是 NGINX 官方基于 Kubernetes Gateway API 实现的 Gateway Controller。

整体关系：

```text
Gateway API
    │
    ├── GatewayClass
    ├── Gateway
    ├── HTTPRoute
    └── ReferenceGrant
          │
          ▼
NGINX Gateway Fabric Controller
          │
          ▼
NGINX Data Plane
          │
          ▼
Kubernetes Service
          │
          ▼
Pod
```

NGF 中需要区分两个概念：

```text
Control Plane
    └── NGINX Gateway Fabric Controller

Data Plane
    └── NGINX
```

真正承载业务流量的是 NGINX Data Plane，而不是 Gateway Controller。

---

## 2. 环境版本

当前部署环境：

```text
Kubernetes:              1.34
NGINX Gateway Fabric:    2.7.0
Gateway API:             1.6.1
NGINX OSS:               1.31.4
```

NGINX Gateway Fabric 2.7.0 官方支持 Kubernetes 1.32+，因此 Kubernetes 1.34 可以直接使用。

---

## 3. Gateway API 是否需要单独安装

需要。

Gateway API 并不是安装 Kubernetes 1.34 后就默认完整提供的一组资源，需要提前安装对应 CRD。

可以先检查：

```bash
kubectl get crd | grep gateway.networking.k8s.io
```

正常情况下会看到：

```text
gatewayclasses.gateway.networking.k8s.io
gateways.gateway.networking.k8s.io
grpcroutes.gateway.networking.k8s.io
httproutes.gateway.networking.k8s.io
referencegrants.gateway.networking.k8s.io
```

如果不存在，需要先安装 Gateway API。

---

## 4. 安装 Gateway API

NGINX Gateway Fabric 2.7.0 推荐使用对应的 Gateway API Standard Channel：

```bash
kubectl kustomize \
"https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=v2.7.0" \
| kubectl apply -f -
```

检查：

```bash
kubectl get crd | grep gateway.networking.k8s.io
```

查看 Kubernetes API：

```bash
kubectl api-resources | grep -E 'Gateway|HTTPRoute'
```

核心资源应该包含：

```text
GatewayClass
Gateway
HTTPRoute
GRPCRoute
ReferenceGrant
```

生产环境建议优先使用 Standard Channel，不建议没有明确需求就直接使用 Experimental Channel。

---

## 5. 安装 NGINX Gateway Fabric

创建 Namespace：

```bash
kubectl create namespace nginx-gateway
```

NGF 推荐通过 Helm OCI Chart 安装。

首先将官方默认 values 保存到本地：

```bash
helm show values \
  oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --version 2.7.0 \
  > values-ngf-2.7.0.yaml
```

建议不要直接长期修改完整的官方 values，而是保留两份：

```text
helm/
├── values-ngf-2.7.0.yaml
└── values-prod.yaml
```

其中：

```text
values-ngf-2.7.0.yaml
    └── 保存官方默认配置，用于升级时 diff

values-prod.yaml
    └── 保存生产环境真正需要覆盖的参数
```

安装：

```bash
helm install ngf \
  oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --version 2.7.0 \
  -n nginx-gateway \
  -f values-prod.yaml \
  --wait
```

检查：

```bash
helm list -n nginx-gateway
kubectl get pods -n nginx-gateway
kubectl get deployment -n nginx-gateway
```

---

## 6. Gateway API 核心资源关系

可以简单理解为：

```text
Gateway Controller
        │
        ▼
GatewayClass
        │
        ▼
Gateway
        │
        ▼
HTTPRoute
        │
        ▼
Service
        │
        ▼
Pod
```

### Gateway Controller

Gateway Controller 是 Gateway API 的具体实现。

例如：

```text
NGINX Gateway Fabric
Envoy Gateway
Traefik
```

NGF Controller 负责监听 Gateway API 对象，并维护对应的 NGINX 数据面。

---

### GatewayClass

GatewayClass 用于声明：

> 这一类 Gateway 应该交给哪个 Controller 管理。

可以近似理解为：

```text
GatewayClass
    ≈
选择使用哪一种 Gateway Controller
```

---

### Gateway

Gateway 代表一个实际的网络入口。

主要关注：

```text
监听端口
协议
hostname
TLS
证书
允许哪些 Route 绑定
```

例如：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public-gateway
  namespace: viitor-live
spec:
  gatewayClassName: nginx

  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: "*.ilivedata.com"

    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: ilivedata-tls

    allowedRoutes:
      namespaces:
        from: Same
```

可以类比传统 NGINX：

```text
Gateway
    ≈
listen
+
server_name
+
ssl_certificate
```

---

### HTTPRoute

HTTPRoute 负责具体业务路由。

主要关注：

```text
绑定哪个 Gateway
匹配哪个域名
匹配哪个 Path/Header
转发给哪个 Service
```

例如：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: t-ilivedata
  namespace: viitor-live
spec:
  parentRefs:
  - name: public-gateway

  hostnames:
  - "t.ilivedata.com"

  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /

    backendRefs:
    - name: t-service
      port: 8080
```

可以近似理解为：

```text
HTTPRoute
    ≈
location
+
proxy_pass
```

---

## 7. 一个 Gateway 是否可以绑定多个 HTTPRoute

可以。

而且这是 Gateway API 非常典型的使用方式。

例如目前存在：

```text
t.ilivedata.com
s.ilivedata.com
```

不需要创建两个 Gateway。

可以：

```text
                       public-gateway
                   *.ilivedata.com:443
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
        HTTPRoute                   HTTPRoute
     t.ilivedata.com             s.ilivedata.com
              │                         │
              ▼                         ▼
         t-service                  s-service
```

例如：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: t-ilivedata
  namespace: viitor-live
spec:
  parentRefs:
  - name: public-gateway

  hostnames:
  - t.ilivedata.com

  rules:
  - backendRefs:
    - name: t-service
      port: 8080
```

第二个：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: s-ilivedata
  namespace: viitor-live
spec:
  parentRefs:
  - name: public-gateway

  hostnames:
  - s.ilivedata.com

  rules:
  - backendRefs:
    - name: s-service
      port: 8080
```

对应以前 ingress-nginx 的思路：

```text
以前：

一个 ingress-nginx Controller
    │
    ├── Ingress t.ilivedata.com
    └── Ingress s.ilivedata.com


现在：

一个 Gateway
    │
    ├── HTTPRoute t.ilivedata.com
    └── HTTPRoute s.ilivedata.com
```

不建议简单理解为：

```text
一个域名 = 一个 Gateway
```

通常只有需要：

```text
独立 LoadBalancer
独立 NGINX Data Plane
独立安全边界
独立公网/内网入口
```

时，才有必要拆多个 Gateway。

---

## 8. 为什么 Gateway 和 HTTPRoute 都有 hostname

两者职责不同。

### Gateway hostname

Gateway Listener 中：

```yaml
hostname: "*.ilivedata.com"
```

表示：

> 这个 Listener 允许接收哪些 hostname。

属于入口级约束。

---

### HTTPRoute hostname

HTTPRoute：

```yaml
hostnames:
- t.ilivedata.com
```

表示：

> 这个 Route 实际负责匹配哪个 Host。

属于业务路由级匹配。

最终是：

```text
Gateway Listener hostname
            ∩
HTTPRoute hostnames
            =
实际生效 hostname
```

例如：

```text
Gateway:

*.ilivedata.com


HTTPRoute A:

t.ilivedata.com


HTTPRoute B:

s.ilivedata.com
```

则：

```text
t.ilivedata.com
    ↓
HTTPRoute A


s.ilivedata.com
    ↓
HTTPRoute B
```

但如果：

```text
Gateway:
*.ilivedata.com

HTTPRoute:
api.example.com
```

两者没有 hostname 交集，这个 Route 就无法通过该 Listener 正常承载该域名。

可以简单记住：

```text
Gateway hostname
    =
入口允许哪些域名


HTTPRoute hostname
    =
具体哪个业务 Route 处理哪些域名
```

---

## 9. allowedRoutes 的 YAML 层级

`allowedRoutes` 属于单个 Listener。

所以它与：

```text
name
protocol
port
hostname
tls
```

同级。

正确：

```yaml
spec:
  gatewayClassName: nginx

  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: "*.ilivedata.com"

    tls:
      mode: Terminate
      certificateRefs:
      - name: ilivedata-tls

    allowedRoutes:
      namespaces:
        from: All
```

结构：

```text
spec
│
├── gatewayClassName
│
└── listeners
      │
      └── listener
            ├── name
            ├── protocol
            ├── port
            ├── hostname
            ├── tls
            └── allowedRoutes
```

错误的理解是：

```text
listeners
allowedRoutes
```

两者并不是同级。

---

## 10. allowedRoutes 中 Same 与 All

如果：

```yaml
allowedRoutes:
  namespaces:
    from: Same
```

代表：

> 只允许与 Gateway 位于同一个 Namespace 的 Route 绑定。

例如：

```text
viitor-live
├── Gateway
├── HTTPRoute
└── Service
```

这种情况下非常合适。

如果：

```yaml
allowedRoutes:
  namespaces:
    from: All
```

代表：

> 允许其他 Namespace 中的 Route 绑定这个 Listener。

适合公共 Gateway：

```text
gateway-system
└── public-gateway
       │
       ├──────── viitor-live/HTTPRoute
       ├──────── auth/HTTPRoute
       └──────── nlp/HTTPRoute
```

生产环境如果只允许部分 Namespace 使用，更建议后续使用 Namespace Selector 做限制，而不是无限制 `All`。

---

## 11. Gateway 放业务 Namespace 是否可以

可以。

例如：

```text
viitor-live
│
├── Gateway
├── HTTPRoute
├── Service
└── Deployment
```

如果这个 Gateway 只服务 `viitor-live` 业务，这样设计没有问题。

并且可以使用：

```yaml
allowedRoutes:
  namespaces:
    from: Same
```

结构简单，权限边界也比较清晰。

---

## 12. 什么时候建议独立 gateway-system Namespace

如果一个 Gateway 是整个集群或者多个业务的公共入口：

```text
                    public-gateway
                          │
             ┌────────────┼────────────┐
             │            │            │
             ▼            ▼            ▼
        viitor-live      auth          nlp
        HTTPRoute      HTTPRoute    HTTPRoute
```

建议把 Gateway 独立出来：

```text
gateway-system
└── public-gateway

viitor-live
├── HTTPRoute
└── Service

auth
├── HTTPRoute
└── Service
```

这样更符合 Gateway API 的职责设计：

```text
平台/SRE
    │
    └── Gateway


业务
    │
    └── HTTPRoute
```

如果目前只有单业务 Namespace，没有必要为了形式强行拆分。

---

## 13. Gateway 与 LoadBalancer 的关系

以前 ingress-nginx 常见架构：

```text
LoadBalancer
     │
     ▼
ingress-nginx Service
     │
     ▼
ingress-nginx Pod
     │
     ▼
Service
```

NGF 可以继续保持类似入口：

```text
LoadBalancer
     │
     ▼
Gateway 对应 Service
     │
     ▼
NGINX Data Plane
     │
     ▼
Backend Service
     │
     ▼
Pod
```

需要特别注意：

```text
业务流量不会经过 NGF Controller。
```

Controller 是控制面。

真正的数据流：

```text
Client
   │
   ▼
CLB / NLB
   │
   ▼
Gateway Service
   │
   ▼
NGINX Data Plane
   │
   ▼
Backend Service
   │
   ▼
Pod
```

而控制链路：

```text
Gateway API
   │
   ▼
NGF Controller
   │
   ▼
配置 NGINX Data Plane
```

---

## 14. 推荐的当前架构

如果之前架构为：

```text
一个 CLB
   │
   ▼
一套 ingress-nginx
   │
   ├── t.ilivedata.com
   ├── s.ilivedata.com
   └── 其他业务域名
```

迁移后建议保持：

```text
一个 CLB
   │
   ▼
一个 public-gateway
   │
   ▼
一组 NGINX Data Plane
   │
   ├── HTTPRoute t.ilivedata.com
   ├── HTTPRoute s.ilivedata.com
   └── HTTPRoute ...
```

不要一开始设计成：

```text
t.ilivedata.com
    ↓
Gateway A

s.ilivedata.com
    ↓
Gateway B
```

否则很可能意味着：

```text
多个 Gateway
+
多套 NGINX Data Plane
+
多个 Service
+
可能多个 LoadBalancer
```

除非确实存在隔离需求，否则没有必要。

---

## 15. 推荐 Namespace 规划

如果当前是单业务 Gateway：

```text
nginx-gateway
    └── NGF Controller

viitor-live
    ├── Gateway
    ├── HTTPRoute
    ├── Service
    ├── Deployment
    └── TLS Secret
```

如果未来成为公共 Gateway：

```text
nginx-gateway
    └── NGF Controller

gateway-system
    ├── public-gateway
    └── TLS Secret

viitor-live
    ├── HTTPRoute
    ├── Service
    └── Deployment

auth
    ├── HTTPRoute
    ├── Service
    └── Deployment
```

---

## 16. 一套完整示例

Gateway：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: public-gateway
  namespace: viitor-live
spec:
  gatewayClassName: nginx

  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: "*.ilivedata.com"

    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: ilivedata-tls

    allowedRoutes:
      namespaces:
        from: Same
```

`t.ilivedata.com`：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: t-ilivedata
  namespace: viitor-live
spec:
  parentRefs:
  - name: public-gateway

  hostnames:
  - t.ilivedata.com

  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /

    backendRefs:
    - name: t-service
      port: 8080
```

`s.ilivedata.com`：

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: s-ilivedata
  namespace: viitor-live
spec:
  parentRefs:
  - name: public-gateway

  hostnames:
  - s.ilivedata.com

  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /

    backendRefs:
    - name: s-service
      port: 8080
```

最终：

```text
                      *.ilivedata.com
                            │
                         Gateway
                            │
                 ┌──────────┴──────────┐
                 │                     │
        t.ilivedata.com        s.ilivedata.com
                 │                     │
            HTTPRoute              HTTPRoute
                 │                     │
            t-service              s-service
                 │                     │
                Pod                   Pod
```

---

## 17. 部署后检查

检查 GatewayClass：

```bash
kubectl get gatewayclass
```

检查 Gateway：

```bash
kubectl get gateway -A
```

详细状态：

```bash
kubectl describe gateway public-gateway -n viitor-live
```

重点：

```text
Accepted=True
Programmed=True
```

检查 HTTPRoute：

```bash
kubectl get httproute -A
```

详细检查：

```bash
kubectl describe httproute t-ilivedata -n viitor-live
```

重点：

```text
Accepted=True
ResolvedRefs=True
```

如果：

```text
Accepted=False
```

重点检查：

```text
parentRefs
allowedRoutes
hostname
namespace
listener
```

如果：

```text
ResolvedRefs=False
```

重点检查：

```text
backend Service
port
跨 Namespace 引用
ReferenceGrant
```

---

## 18. 当前几个核心疑问总结

### 1. Gateway API 要不要额外安装？

需要。

NGF 是 Gateway API 的实现，而 Gateway API CRD 本身需要提前安装。

---

### 2. 一个 Gateway 可以对应多个 HTTPRoute 吗？

可以，而且推荐这么使用。

典型关系：

```text
1 Gateway
    │
    ├── N HTTPRoute
    └── N Domain
```

---

### 3. Gateway 可以直接放业务 Namespace 吗？

可以。

如果 Gateway 只服务这个业务 Namespace，这种方式非常合理。

如果未来要成为全局统一入口，则推荐迁移到独立的 `gateway-system` Namespace。

---

### 4. 为什么 Gateway 和 HTTPRoute 都有 hostname？

因为职责不同：

```text
Gateway hostname
    =
Listener 可以接收哪些域名

HTTPRoute hostname
    =
当前 Route 实际匹配哪些域名
```

最终取 hostname 交集。

---

### 5. allowedRoutes 和 listeners 是同级吗？

不是。

`allowedRoutes` 是单个 Listener 的属性：

```text
listeners
    └── listener
          └── allowedRoutes
```

所以与：

```text
name
protocol
port
hostname
tls
```

同级。

---

### 6. 是否应该一个域名创建一个 Gateway？

通常不需要。

推荐：

```text
一个统一入口 Gateway
+
多个 HTTPRoute
```

只有在需要：

```text
独立 LB
独立数据面
公网/内网隔离
安全边界隔离
资源隔离
```

时，再创建多个 Gateway。

---

## 19. 总结

Gateway API 可以用下面的方式快速理解：

```text
GatewayClass
    ↓
选择 Gateway Controller

Gateway
    ↓
定义网络入口
端口 / 协议 / TLS / hostname

HTTPRoute
    ↓
定义业务路由
host / path / header / backend

Service
    ↓
后端服务发现

Pod
```

如果从 ingress-nginx 迁移：

```text
Ingress Controller
    →
NGINX Gateway Fabric Controller


Ingress 中的入口能力
    →
Gateway


Ingress 中的业务路由
    →
HTTPRoute
```

对于原来：

```text
一个 LoadBalancer
+
一个 ingress-nginx Controller
+
多个域名
```

的架构，更推荐迁移成：

```text
一个 LoadBalancer
+
一个 Gateway
+
一组 NGINX Data Plane
+
多个 HTTPRoute
```

这样既保留了统一入口，也把平台入口配置和业务路由配置进行了拆分。

---

## 20. 参考资料

- NGINX Gateway Fabric Helm 安装：
  https://docs.nginx.com/nginx-gateway-fabric/install/helm/

- NGINX Gateway Fabric Technical Specifications：
  https://docs.nginx.com/nginx-gateway-fabric/overview/technical-specifications/

- Gateway API HTTPRoute：
  https://gateway-api.sigs.k8s.io/reference/api-types/httproute/

- Gateway API Hostnames：
  https://gateway-api.sigs.k8s.io/docs/concepts/hostnames/

- Gateway API Overview：
  https://gateway-api.sigs.k8s.io/docs/concepts/api-overview/
