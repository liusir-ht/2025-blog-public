## 问题背景
  业务中开发了一个流式长连接（SSE）的功能，需要用户访问域名的时候，让服务端支持Websocket保障用户持续交互的能力。

> SSE（Server-Sent Events）是一种基于 HTTP 的**单向流式通信**机制：
>- 服务端持续推送数据
>- 客户端通过 `EventSource` 接收
>- 典型场景：
>  - AI 流式输出（ChatGPT 类）
>  - 实时日志
>  - 进度通知
## 环境信息

| 组件名称 | 组件版本 | 功能 |
| --- | --- | --- |
| Kubernetes | v1.32.2 | 容器编排服务 |
| Ingress-nginx |v1.13.0  | 给Ingress资源提供路由功能|


## YAML实现

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: 10m
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-next-upstream-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/websocket-services: xxx-xx
  labels:
    ingress-controller: nginx
  name: xxx-net
  namespace: xxx-prod
spec:
  ingressClassName: nginx
  rules:
  - host: xxx.xxx.net
    http:
      paths:
      - backend:
          service:
            name: xxx-xx
            port:
              number: 19090
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - xxx.xxx.net
    secretName: https
```

## 字段解释
### 基础信息

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
```

- **apiVersion**：资源版本，这里是稳定版 `v1`
- **kind**：资源类型，Ingress 表示七层（HTTP/HTTPS）路由规则

---

### metadata（元数据）

```yaml
metadata:
  annotations:
    ...
  labels:
    ingress-controller: nginx
  name: xxx-net
  namespace: xxx-prod
```

#### 1）name & namespace

- **name**：Ingress 名称（唯一标识）
- **namespace**：所属命名空间（必须和 Service 在同一个 namespace）

---

#### 2）labels

```yaml
labels:
  ingress-controller: nginx
```

- 用于资源分类或筛选
- 常用于：
  - 监控系统
  - CI/CD 管理
  - kubectl 查询过滤

---

#### 3）annotations（重点）

这是 Nginx Ingress 的核心扩展配置，用于控制反向代理行为。

##### 请求体大小限制

```yaml
nginx.ingress.kubernetes.io/proxy-body-size: 10m
```

- 最大请求体大小（如文件上传）
- 默认通常是 1MB，这里扩大到 10MB

---

##### 关闭缓冲

```yaml
nginx.ingress.kubernetes.io/proxy-buffering: "off"
```

- 禁用响应缓冲
- 适用于：
  - 流式返回（stream）
  - SSE / 实时日志
- 否则 Nginx 会先缓存再返回

---

##### 超时配置（关键）

```yaml
nginx.ingress.kubernetes.io/proxy-connect-timeout: "3600"
nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
nginx.ingress.kubernetes.io/proxy-next-upstream-timeout: "3600"
```

说明：

| 参数 | 含义 |
|------|------|
| connect-timeout | 建立连接超时时间 |
| read-timeout | 读取后端响应超时 |
| send-timeout | 发送请求到后端超时 |
| next-upstream-timeout | 切换后端重试的总时间 |

👉 全部设置为 **3600秒（1小时）**

---

##### WebSocket 支持

```yaml
nginx.ingress.kubernetes.io/websocket-services: xxx-xx
```

- 指定需要支持 WebSocket 的 Service

---

### spec（核心配置）

```yaml
spec:
  ingressClassName: nginx
```

- 指定使用的 Ingress Controller

---

### 路由规则 rules

```yaml
rules:
- host: xxx.xxx.net
  http:
    paths:
```

#### 1）host

- 访问域名：

```
https://xxx.xxx.net
```

---

#### 2）path 配置

```yaml
- path: /
  pathType: Prefix
```

##### pathType

| 类型 | 含义 |
|------|------|
| Prefix | 前缀匹配 |
| Exact | 精确匹配 |

---

#### 3）backend（后端服务）

```yaml
backend:
  service:
    name: xxx-xx
    port:
      number: 19090
```

👉 流量路径：

```
用户 → Ingress → Service → Pod
```

---

### TLS 配置

```yaml
tls:
- hosts:
  - xxx.xxx.net
  secretName: http
```

- **secretName**：证书 Secret

---

## 总结
 本质上，Ingress 对 WebSocket 的支持，就是让“短连接代理”正确服务“长连接协议”。
 
## 扩展知识
###  为什么关闭缓存针对SSE的场景特别关键呢？
 在 SSE 场景中，对响应的实时性要求很高。默认情况下，Ingress-Nginx 会对后端响应进行缓冲：先将数据写入 buffer，待积累到一定程度后再返回给客户端。这种方式类似“批处理”，可以提升吞吐率并降低网络 IO 开销。

当关闭 buffering 后，Ingress-Nginx 会变为“边收边发”的模式，后端每产生一段数据就立即转发给客户端，从而保证流式响应的实时性。