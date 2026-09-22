# Kubernetes kubectl debug 临时容器与 Pod 副本排障指南

## 一、背景

生产环境中的业务镜像通常会尽量精简，容器内可能没有以下排障工具：

```text
curl
ping
dig
nslookup
tcpdump
ss
netstat
traceroute
```

直接修改线上镜像或重建 Deployment，不仅操作成本较高，还可能影响正在运行的业务。

Kubernetes 提供的 `kubectl debug` 可以通过以下两类方式辅助排障：

```text
1. 向正在运行的 Pod 注入临时容器（Ephemeral Container）
2. 基于原 Pod 创建一个独立副本，并在副本中修改命令或镜像
```

这两种方式需要区分：

> 临时容器模式会修改原 Pod 的 `ephemeralContainers` 字段；`--copy-to` 模式会创建一个新的 Pod，不会直接替换原 Pod。

---

## 二、脱敏后的示例变量

本文统一使用以下脱敏名称：

```text
命名空间：<namespace>
原 Pod：<source-pod>
业务容器：<app-container>
调试 Pod：<debug-pod>
调试镜像：registry.example.com/ops/netshoot:v0.13
替换镜像：ubuntu:22.04
```

执行前可以先确认 Pod 和容器名称：

```bash
kubectl -n <namespace> get pod <source-pod>

kubectl -n <namespace> get pod <source-pod> \
  -o jsonpath='{.spec.containers[*].name}{"\n"}'
```

---

## 三、方式一：向运行中的 Pod 注入临时容器

脱敏后的命令：

```bash
kubectl -n <namespace> debug -it <source-pod> \
  --image=registry.example.com/ops/netshoot:v0.13 \
  --target=<app-container>
```

### 命令作用

该命令会在原 Pod 中新增一个临时调试容器，并进入交互式终端。

```text
原 Pod
├── app-container                 原业务容器
└── debugger-xxxxx               临时调试容器
```

临时容器与业务容器位于同一个 Pod，因此默认共享 Pod 的网络命名空间，可以直接排查：

```text
Pod 内 DNS 解析
Service 连通性
本地监听端口
上游 TCP 连接
网络延迟
抓包
```

例如：

```bash
ip addr
ss -lntp
curl -v http://127.0.0.1:<port>/health
dig <service-name>.<namespace>.svc.cluster.local
tcpdump -i any -nn port <port>
```

### 参数解释

| 参数 | 含义 |
| --- | --- |
| `-n <namespace>` | 指定原 Pod 所在的命名空间 |
| `debug` | 创建 Kubernetes 调试会话 |
| `-i` | 保持标准输入打开 |
| `-t` | 分配 TTY 终端 |
| `--image` | 指定临时调试容器使用的镜像 |
| `--target` | 让临时容器以指定业务容器为进程命名空间目标 |

### `--target` 的实际含义

`--target=<app-container>` 主要用于让调试容器查看目标容器的进程。

进入临时容器后可以执行：

```bash
ps -ef
```

如果容器运行时支持该能力，输出中可以看到目标业务容器的进程。

需要注意：

> `--target` 不等于“进入目标容器”，也不会把目标容器的根文件系统自动挂载到调试容器中。

如果没有指定 `--target`，通常仍然可以利用共享的 Pod 网络命名空间排查网络，但可能无法看到业务容器进程。

---

## 四、查看已经创建的临时容器

查看 Pod 详情：

```bash
kubectl -n <namespace> describe pod <source-pod>
```

只查看临时容器定义：

```bash
kubectl -n <namespace> get pod <source-pod> \
  -o jsonpath='{.spec.ephemeralContainers[*].name}{"\n"}'
```

查看临时容器状态：

```bash
kubectl -n <namespace> get pod <source-pod> \
  -o jsonpath='{range .status.ephemeralContainerStatuses[*]}{.name}{"\t"}{.state}{"\n"}{end}'
```

重新进入仍在运行的临时容器：

```bash
kubectl -n <namespace> attach -it <source-pod> -c <debug-container>
```

---

## 五、临时容器的限制

临时容器适合快速在线诊断，但存在以下限制：

```text
1. 已经添加到 PodSpec 的临时容器不能单独删除
2. 临时容器退出后通常不能像普通容器一样重启
3. 不支持普通容器的全部字段，例如端口、探针和资源声明等能力受限
4. 是否能够看到目标容器进程取决于容器运行时对 --target 的支持
5. Pod Security、准入策略或镜像拉取权限可能阻止临时容器启动
6. 调试镜像默认不一定拥有 NET_ADMIN、SYS_PTRACE 等额外能力
```

如果需要更高权限的网络排查，可根据集群安全策略评估：

```bash
kubectl -n <namespace> debug -it <source-pod> \
  --image=registry.example.com/ops/netshoot:v0.13 \
  --target=<app-container> \
  --profile=netadmin
```

生产环境不建议未经评估直接使用：

```text
--profile=sysadmin
```

因为该配置会授予更高的容器权限。

---

## 六、方式二：复制 Pod 并修改指定容器的启动命令

脱敏后的命令：

```bash
kubectl -n <namespace> debug <source-pod> -it \
  --copy-to=<debug-pod> \
  --container=<app-container> \
  -- sleep infinity
```

### 命令作用

该命令会基于原 Pod 创建一个名为 `<debug-pod>` 的副本，并将副本中 `<app-container>` 的启动命令修改为：

```bash
sleep infinity
```

整体关系如下：

```text
原 Pod：<source-pod>        保持运行，不被替换
副本 Pod：<debug-pod>       新建，用于独立排障
└── <app-container>         启动命令改为 sleep infinity
```

这种方式适用于：

```text
业务容器启动后立即退出
需要保留原容器的环境变量和挂载配置
需要进入容器检查配置文件
需要阻止业务进程启动
需要较长时间进行离线排障
```

### 参数解释

| 参数 | 含义 |
| --- | --- |
| `--copy-to=<debug-pod>` | 根据原 Pod 创建一个指定名称的新 Pod |
| `--container=<app-container>` | 选择副本中需要修改启动命令的容器 |
| `--` | 分隔 `kubectl debug` 参数与容器启动命令 |
| `sleep infinity` | 让容器持续运行，便于后续进入排障 |

### 特别注意

在这条命令中没有指定 `--image`，因此它不是向副本添加一个新的 Ubuntu 或 netshoot 调试容器。

它的实际行为是：

> 复制原 Pod，并修改副本中指定原容器的启动命令。

因此，原业务镜像中如果没有 Shell 或 `sleep` 命令，副本可能启动失败。

可以先检查镜像是否具备对应命令，或者改用后文的替换镜像方式。

---

## 七、进入复制出来的调试 Pod

查看 Pod 状态：

```bash
kubectl -n <namespace> get pod <debug-pod> -o wide
```

查看事件：

```bash
kubectl -n <namespace> describe pod <debug-pod>
```

进入指定容器：

```bash
kubectl -n <namespace> exec -it <debug-pod> \
  -c <app-container> -- /bin/sh
```

如果镜像中存在 Bash：

```bash
kubectl -n <namespace> exec -it <debug-pod> \
  -c <app-container> -- /bin/bash
```

---

## 八、方式三：复制 Pod 并替换指定容器镜像

脱敏后的命令：

```bash
kubectl -n <namespace> debug <source-pod> -it \
  --copy-to=<debug-pod> \
  --set-image=<app-container>=ubuntu:22.04 \
  --container=<app-container> \
  -- sleep infinity
```

### 命令作用

该命令会执行以下操作：

```text
1. 基于 <source-pod> 创建 <debug-pod>
2. 将副本中 <app-container> 的镜像替换为 ubuntu:22.04
3. 将该容器的启动命令改为 sleep infinity
4. 保留原 Pod，不直接修改线上业务 Pod
```

### 为什么建议显式添加 `--container`

原始写法如果只有：

```bash
--set-image=<app-container>=ubuntu:22.04 -- sleep infinity
```

在多容器 Pod 中，命令应用到哪个容器容易产生歧义。

建议显式指定：

```bash
--container=<app-container>
```

这样可以明确：

```text
替换镜像的容器：<app-container>
修改启动命令的容器：<app-container>
交互连接的容器：<app-container>
```

### `--set-image` 的格式

```text
--set-image=<原容器名称>=<新镜像>
```

例如：

```bash
--set-image=app=ubuntu:22.04
```

其中 `app` 必须是原 Pod 中真实存在的容器名称，而不是 Deployment 名称或 Pod 名称。

可以执行以下命令确认：

```bash
kubectl -n <namespace> get pod <source-pod> \
  -o jsonpath='{.spec.containers[*].name}{"\n"}'
```

---

## 九、替换镜像后的兼容性风险

将业务容器镜像替换成 `ubuntu:22.04` 后，Pod 副本仍可能保留原容器的部分配置，例如：

```text
环境变量
VolumeMount
SecurityContext
工作目录
资源配置
ServiceAccount
调度配置
```

但新镜像不一定与原配置兼容，常见问题包括：

```text
1. 原 workingDir 在 Ubuntu 镜像中不存在
2. 原 volumeMount 路径与新镜像目录结构不匹配
3. 原安全上下文要求使用特定 UID，新镜像无法正常运行
4. 原 Pod 使用私有镜像拉取凭证，但节点无法拉取新镜像
5. Init Container 仍会执行，可能触发外部依赖或初始化逻辑
6. Sidecar 容器也会随副本启动，可能访问真实中间件或注册中心
```

因此，创建副本前建议先查看原 Pod：

```bash
kubectl -n <namespace> get pod <source-pod> -o yaml
```

---

## 十、是否需要调度到原节点

`--copy-to` 创建的 Pod 默认会重新参与调度，不保证与原 Pod 位于同一节点。

如果需要排查与节点相关的问题，例如：

```text
节点本地磁盘
本地缓存
特定 GPU
HostPath
节点网络
节点上的 Unix Socket
```

可以添加：

```bash
--same-node
```

示例：

```bash
kubectl -n <namespace> debug <source-pod> -it \
  --copy-to=<debug-pod> \
  --same-node \
  --set-image=<app-container>=ubuntu:22.04 \
  --container=<app-container> \
  -- sleep infinity
```

但即使使用 `--same-node`，仍需确认节点资源、污点、亲和性和存储挂载是否允许副本启动。

---

## 十一、避免调试副本接收线上流量

使用 `--copy-to` 创建的 Pod 通常不会默认保留原 Pod 的标签，因此一般不会被原 Service 选中。

但仍然必须检查：

```bash
kubectl -n <namespace> get pod <debug-pod> --show-labels

kubectl -n <namespace> get service -o wide
```

重点确认：

```text
调试 Pod 的标签是否匹配线上 Service selector
调试 Pod 是否会注册到服务发现系统
Sidecar 是否会主动对外建立连接
应用是否会消费真实消息队列
应用是否会执行定时任务
```

生产环境中，复制 Pod 的最大风险通常不是覆盖原 Pod，而是：

> 调试副本携带真实配置启动后，意外接收流量、消费消息或访问生产数据。

---

## 十二、不要随意使用 `--replace`

`kubectl debug` 支持：

```text
--replace
```

该参数与 `--copy-to` 一起使用时，会删除原 Pod。

生产排障通常不应使用：

```bash
kubectl debug <source-pod> \
  --copy-to=<debug-pod> \
  --replace
```

除非已经明确确认：

```text
原 Pod 可以被删除
上层控制器会正确重建
不存在本地临时数据丢失风险
不会影响线上流量
操作已经完成审批
```

---

## 十三、调试完成后的清理

### 1. 临时容器模式

已经写入 PodSpec 的临时容器不能单独删除。

退出临时容器：

```bash
exit
```

如果原 Pod 由 Deployment、StatefulSet 等控制器管理，可以在确认业务影响后，通过重建 Pod 清除临时容器记录。

不要仅为了清除临时容器记录就在生产环境随意删除业务 Pod。

### 2. Pod 副本模式

排障完成后删除副本：

```bash
kubectl -n <namespace> delete pod <debug-pod>
```

删除前再次确认目标名称，避免误删原 Pod：

```bash
kubectl -n <namespace> get pod <source-pod> <debug-pod> -o wide
```

---

## 十四、常见报错与排查

### 1. 找不到目标容器

报错可能类似：

```text
container not found
```

确认容器名称：

```bash
kubectl -n <namespace> get pod <source-pod> \
  -o jsonpath='{.spec.containers[*].name}{"\n"}'
```

### 2. 调试镜像拉取失败

查看事件：

```bash
kubectl -n <namespace> describe pod <source-pod>
```

或查看副本：

```bash
kubectl -n <namespace> describe pod <debug-pod>
```

重点检查：

```text
ImagePullBackOff
ErrImagePull
镜像地址和 Tag
imagePullSecrets
节点到镜像仓库的网络
镜像架构是否匹配节点
```

### 3. `ps` 看不到业务容器进程

可能原因：

```text
容器运行时不支持 --target 的进程命名空间能力
目标容器已经退出
临时容器未成功以目标容器为 Target
安全策略限制相关能力
```

### 4. `sleep infinity` 启动失败

原业务镜像可能没有 `sleep`，或者其 `sleep` 实现不支持 `infinity`。

可以根据镜像能力尝试：

```bash
-- sh -c 'while true; do sleep 3600; done'
```

但如果镜像中连 Shell 都没有，应使用 `--set-image` 替换成调试镜像。

### 5. 权限不足

账号可能缺少以下资源权限：

```text
pods/get
pods/create
pods/ephemeralcontainers/patch
pods/attach
pods/exec
pods/delete
```

可以使用以下命令进行确认：

```bash
kubectl auth can-i patch pods/ephemeralcontainers -n <namespace>
kubectl auth can-i create pods -n <namespace>
kubectl auth can-i create pods/attach -n <namespace>
kubectl auth can-i create pods/exec -n <namespace>
```

---

## 十五、三种命令的对比

| 方式 | 是否修改原 Pod | 是否创建新 Pod | 是否替换镜像 | 主要用途 |
| --- | --- | --- | --- | --- |
| `--image` + `--target` | 会新增临时容器记录 | 否 | 否 | 在线网络、进程和连通性排查 |
| `--copy-to` + `--container` | 否 | 是 | 否 | 保留原镜像，覆盖启动命令进行排障 |
| `--copy-to` + `--set-image` | 否 | 是 | 仅替换副本中的指定容器 | 原镜像缺少 Shell 或调试工具时使用 |

---

## 十六、推荐的使用顺序

遇到运行中 Pod 故障时，可以按以下顺序选择：

```text
只需要检查网络、DNS、端口或抓包
↓
使用临时容器：--image + --target
↓
需要保留原环境并阻止业务进程启动
↓
复制 Pod：--copy-to + --container + sleep infinity
↓
原镜像没有 Shell、sleep 或排障工具
↓
复制 Pod 并替换镜像：--copy-to + --set-image
```

执行 `--copy-to` 前，额外确认：

```text
副本是否会接收线上流量
副本是否会消费真实消息
副本是否会执行定时任务
Sidecar 和 Init Container 是否会产生外部影响
是否需要使用 --same-node
排障完成后是否已经删除副本
```

---

## 十七、常用命令速查

### 注入临时网络排障容器

```bash
kubectl -n <namespace> debug -it <source-pod> \
  --image=registry.example.com/ops/netshoot:v0.13 \
  --target=<app-container>
```

### 复制 Pod 并阻止原业务进程启动

```bash
kubectl -n <namespace> debug <source-pod> -it \
  --copy-to=<debug-pod> \
  --container=<app-container> \
  -- sleep infinity
```

### 复制 Pod 并替换业务容器镜像

```bash
kubectl -n <namespace> debug <source-pod> -it \
  --copy-to=<debug-pod> \
  --set-image=<app-container>=ubuntu:22.04 \
  --container=<app-container> \
  -- sleep infinity
```

### 复制到原节点并替换镜像

```bash
kubectl -n <namespace> debug <source-pod> -it \
  --copy-to=<debug-pod> \
  --same-node \
  --set-image=<app-container>=ubuntu:22.04 \
  --container=<app-container> \
  -- sleep infinity
```

### 删除调试副本

```bash
kubectl -n <namespace> delete pod <debug-pod>
```

---

## 十八、总结

`kubectl debug` 的核心不是简单地“进入容器”，而是根据故障场景选择正确的调试模型：

```text
Ephemeral Container
适合在线、低侵入地补充排障工具

Pod Copy
适合创建与线上 Pod 配置接近的独立排障环境

Pod Copy + Set Image
适合原业务镜像过于精简、无法直接排障的场景
```

生产环境使用时最重要的安全原则是：

```text
确认 Pod、命名空间和容器名称
理解 --target、--container 和 --set-image 的区别
避免使用 --replace 误删原 Pod
确认副本不会接收流量或执行真实业务任务
排障完成后及时清理调试 Pod
```

通过这三种方式，可以在尽量不改动线上 Deployment 和业务镜像的情况下，完成 Kubernetes 容器网络、进程、配置和启动问题的定位。
