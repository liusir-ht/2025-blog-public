
## 一、nvidia-smi topo -m 是什么

`nvidia-smi topo -m` 是 NVIDIA 提供的 GPU 拓扑结构查看命令，用于显示服务器内各 GPU 之间、以及 GPU 与 CPU 之间的物理连接方式和通信带宽。

**核心作用：**

- 查看 GPU 之间的通信路径
- 判断 GPU 是否支持 NVLink 直连
- 查看 GPU 与 CPU 核心的 NUMA 亲和性
- 为多卡分布式训练提供通信优化依据

基本使用：

```bash
nvidia-smi topo -m
```

---

## 二、输出表格结构

运行命令后，输出分为两个部分：

### 1. GPU 间拓扑矩阵（主体）

表格的行和列均为 GPU 编号（如 `GPU0`、`GPU1`...），行列交叉处的字母表示两个 GPU 之间的连接类型。

### 2. GPU 与 CPU 亲和性信息（底部）

显示每个 GPU 对应的：

- CPU 核心亲和性（`CPU Affinity`）
- NUMA 节点亲和性（`NUMA Affinity`）
- GPU NUMA ID

---

## 三、GPU 间连接类型详解

连接类型按**通信速度从快到慢**排列如下：

| 缩写 | 全称 | 含义 | 速度等级 |
| :--- | :--- | :--- | :--- |
| **NV#** | NVLink | 通过 # 条 NVLink 直连，速度远高于 PCIe，是最高速的 GPU 间通信方式 | 极速 |
| **PIX** | Same PCIe Bridge | 位于同一 PCIe 桥片（Switch）下，只经过最多一个 PCIe 桥片，延迟极低 | 最快（PCIe 内） |
| **PXB** | Multiple PCIe Bridges | 跨多个 PCIe 桥片，但不经过 PCIe 主桥（CPU 根端口），常见于 PCIe 交换芯片级联 | 更快 |
| **PHB** | Same PCIe Host Bridge | 位于同一 PCIe 主桥下（通常同属一个 CPU 的同一组 PCIe 控制器） | 较快 |
| **NODE** | Same NUMA Node | 位于同一 NUMA 节点（同一颗物理 CPU），但连接在不同 PCIe 总线上，需经过 CPU 内部互联 | 慢 |
| **SYS** | System PCIe | 位于不同 NUMA 节点（不同 CPU），通信必须经过 QPI/UPI（CPU 间互联总线） | 最慢 |
| **X** | Self | 表示 GPU 与自身，无实际通信意义 | - |
| **N/A** | Not Applicable | 不可用或不存在 | - |

> **重要提示：**
>
> - `NV#` 中的数字（如 `NV1`、`NV2`、`NV4`）表示 NVLink 链路数量，数量越多通信带宽越高。
> - 如果输出中 GPU 间显示为 `X` 但硬件上确实有 NVLink，可能是驱动配置问题，可使用 `nvidia-smi nvlink -s` 检查链路状态。

---

## 四、GPU 与 CPU 亲和性参数

表格底部会显示类似以下内容：

```
GPU0    CPU Affinity    NUMA Affinity    GPU NUMA ID
GPU0    0-31            0                N/A
```

各字段含义：

| 字段 | 含义 | 说明 |
| :--- | :--- | :--- |
| **CPU Affinity** | CPU 核心亲和性 | 表示该 GPU 物理上离哪些 CPU 核心最近，推荐将程序线程绑定到这些核心上 |
| **NUMA Affinity** | NUMA 节点亲和性 | 表示该 GPU 所属的 NUMA 节点，对应特定的物理 CPU 和内存域 |
| **GPU NUMA ID** | GPU NUMA 标识 | 某些架构下 GPU 自身不挂载系统内存，通常显示为 `N/A` |

> **重要提示：**
>
> 在绑定 CPU 核心时，强烈建议将程序的计算线程绑定到 `CPU Affinity` 对应的核心上，避免数据在 CPU 内部绕远路，可显著降低延迟。

---

## 五、实际应用场景

### 1. 多卡 AI 训练优化

根据拓扑矩阵设置 NCCL 环境变量：

| 拓扑类型 | 推荐策略 |
| :--- | :--- |
| 显示 `NV#` | 优先使用 NVLink 通信，NCCL 默认会自动启用 |
| 显示 `PIX`/`PHB` | 启用 P2P 通信收益巨大，保持 NCCL 默认配置即可 |
| 显示 `SYS` | 跨 NUMA 节点通信带宽受限，可考虑设置 `NCCL_P2P_DISABLE=1` 或调整 GPU 顺序 |

### 2. GPU 绑核优化

根据 `CPU Affinity` 列，使用以下命令绑定进程：

```bash
# 使用 numactl（推荐，同时绑核和绑内存）
numactl --cpunodebind=0 --membind=0 python train.py

# 使用 taskset（只绑核心）
taskset -c 0-31 python train.py
```

### 3. 排查通信瓶颈

如果多卡训练时通信速度异常，运行 `nvidia-smi topo -m` 可快速确认是否为跨 CPU 插槽布线导致的物理限制（`SYS` 连接）。

---

## 六、常用命令速查

| 操作 | 命令 |
| :--- | :--- |
| 查看 GPU 拓扑结构 | `nvidia-smi topo -m` |
| 查看 NVLink 链路状态 | `nvidia-smi nvlink -s` |
| 查看 NVLink 详细状态 | `nvidia-smi nvlink -d` |
| 查看系统 NUMA 布局 | `numactl --hardware` |
| 查看 NUMA 节点详细信息 | `lstopo-no-graphics` |
| 查看进程 NUMA 内存分配 | `numastat -p <PID>` |

---

## 七、常见问题排查

### 1. GPU 间显示 SYS 但期望是 NVLink

可能原因：

- GPU 未通过 NVLink 桥接器物理连接
- 驱动未正确识别 NVLink
- NVLink 被禁用

**排查步骤：**

```bash
# 检查 NVLink 状态
nvidia-smi nvlink -s

# 检查详细链路信息
nvidia-smi nvlink -d
```

### 2. GPU 显示 X 或 N/A

- `X`：表示 GPU 与自身，属于正常显示
- `N/A`：表示该连接不存在或不可用，属于正常情况

### 3. CPU Affinity 显示所有核心

如果 `CPU Affinity` 显示全部 CPU 核心（如 `0-127`），说明 GPU 与所有 CPU 核心距离相同（常见于单 CPU 主板或特殊拓扑），此时绑核影响较小。

---

## 八、核心总结

`nvidia-smi topo -m` 是 GPU 服务器多卡通信性能调优的核心工具。

日常最值得关注的要点包括：

1. **GPU 间连接类型**
   - 优先使用 `NV#` 和 `PIX` 连接的 GPU 组队
   - 尽量避免跨 `SYS` 连接做频繁数据交换

2. **CPU 亲和性**
   - 将程序线程绑定到 `CPU Affinity` 对应的核心
   - 使用 `numactl` 同时绑定 CPU 和内存到正确的 NUMA 节点

3. **NVLink 状态**
   - 确认 NVLink 是否正常启用
   - `NV#` 中的数字越大，通信带宽越高

4. **NCCL 策略调整**
   - 根据拓扑设置 `NCCL_P2P_DISABLE`、`NCCL_IB_DISABLE` 等环境变量
   - 跨 NUMA 节点通信时可考虑调整 GPU 顺序

> **核心原则：**
>
> 多卡训练的性能不仅取决于 GPU 算力，还高度依赖 GPU 间的通信带宽和延迟。
>
> 应结合 **拓扑连接类型 + CPU 亲和性 + NVLink 状态 + NCCL 配置** 进行综合优化，才能充分发挥多卡并行计算的优势。