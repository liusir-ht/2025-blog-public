
## 一、nvidia-smi 是什么

`nvidia-smi`（NVIDIA System Management Interface）是 NVIDIA 提供的 GPU 管理和诊断工具，常用于查看：

- GPU 状态
- 显存使用情况
- GPU 利用率
- GPU 温度
- GPU 功耗
- ECC 错误
- GPU 进程
- GPU 性能状态

基本使用：

```bash
nvidia-smi
```

---

## 二、nvidia-smi 核心输出参数

### 1. NVIDIA-SMI

表示 `nvidia-smi` 工具自身的版本。

### 2. Driver Version

当前系统安装的 NVIDIA GPU 驱动版本。

### 3. CUDA Version

表示当前 NVIDIA 驱动支持的最高 CUDA 版本。

> **注意：**
>
> 该版本表示驱动支持的 CUDA 上限，并不代表当前程序实际使用的 CUDA Runtime 版本。

### 4. GPU / Name

表示 GPU 编号和具体型号，例如：

- Tesla V100
- Tesla A100
- GeForce RTX 3090
- GeForce RTX 4090

### 5. Persistence-M

GPU 持久模式。

- `On`：驱动保持加载，可以减少 CUDA 程序启动时的初始化延迟。
- `Off`：非持久模式。

开启：

```bash
sudo nvidia-smi -pm 1
```

关闭：

```bash
sudo nvidia-smi -pm 0
```

### 6. Bus-Id

GPU 在 PCIe 总线上的物理地址，可以用于精确定位 GPU 硬件。

### 7. Disp.A

表示 GPU 是否用于显示输出。

- `Off`：通常用于计算。
- `On`：正在进行显示输出。

### 8. Volatile Uncorr. ECC

表示不可纠正的 ECC 错误数量，是判断 GPU 硬件健康状态的重要指标。

通常：

- `0`：表示没有记录到不可纠正 ECC 错误。
- `>0`：需要进一步检查硬件、温度、电源、PCIe 和系统日志。

查看 ECC：

```bash
nvidia-smi -q -d ECC
```

---

## 三、GPU 实时状态参数

### 1. Fan

风扇转速百分比。

`N/A` 表示 GPU 没有风扇，或者风扇不由驱动控制。

### 2. Temp

GPU 当前温度，单位为 `°C`。

温度过高时需要检查：

- GPU 风扇
- 服务器风道
- 散热器
- 机房环境温度
- GPU 是否长期满载

### 3. Perf

GPU 当前性能状态。

常见状态：

| 状态 | 含义 |
| --- | --- |
| `P0` | 高性能状态 |
| `P2` | 常见计算状态 |
| `P8` | 低功耗空闲状态 |

> **注意：**
>
> 对于 Tesla 等数据中心 GPU，在 CUDA 计算过程中出现 `P2` 并不代表 GPU 性能不足。

### 4. Pwr:Usage/Cap

表示：

`当前功耗 / 最大功耗`

例如：

`48W / 300W`

如果长期接近功耗上限，需要关注：

- GPU 是否降频
- GPU 散热是否正常
- 电源是否满足要求

### 5. Memory-Usage

表示：

`已使用显存 / 总显存`

例如：

`1234MiB / 16384MiB`

如果显存接近总容量，可能出现：

`CUDA out of memory`

即 CUDA OOM。

### 6. GPU-Util

表示 GPU 核心利用率，单位为 `%`。

> **注意：**
>
> 显存占用高但 `GPU-Util` 为 `0%` 不一定是异常。
>
> 例如模型已经加载到 GPU 显存，但当前没有执行计算，此时可能出现：
>
> - `Memory-Usage` 较高
> - `GPU-Util` 为 `0%`
>
> 因此不能只根据 GPU 利用率判断 GPU 是否正常。

### 7. Compute M.

表示 GPU 计算模式。

| 模式 | 说明 |
| --- | --- |
| `Default` | 多个进程可以共享 GPU |
| `Exclusive_Process` | 同一时刻限制一个进程使用 GPU |
| `Prohibited` | 禁止 CUDA/OpenCL 计算 |

设置计算模式：

```bash
# 0 - Default
# 1 - Exclusive_Process
# 2 - Prohibited

sudo nvidia-smi -i <GPU_ID> -c <模式代码>
```

例如：

```bash
sudo nvidia-smi -i 0 -c 1
```

### 8. MIG M.

MIG 即：

`Multi-Instance GPU`

部分 NVIDIA 高端 GPU，例如 A100、H100，支持通过 MIG 将一张物理 GPU 切分成多个 GPU 实例。

---

# 四、GPU 持久模式

## 4.1 Persistence Mode 的作用

持久模式可以让 NVIDIA 驱动保持加载状态，减少 CUDA 程序启动时的初始化延迟。

开启：

```bash
sudo nvidia-smi -pm 1
```

关闭：

```bash
sudo nvidia-smi -pm 0
```

查看状态：

```bash
nvidia-smi
```

查看输出中的：

`Persistence-M`

即可。

---

# 五、ECC 错误

## 5.1 什么是 ECC

ECC 是：

`Error Correcting Code`

即错误检查和纠正机制。

NVIDIA 数据中心 GPU 通常会通过 ECC 检测和处理显存错误。

## 5.2 Volatile Uncorr. ECC

`Volatile Uncorr. ECC` 表示当前统计周期内出现的不可纠正 ECC 错误。

正常情况下：

`Uncorrectable : 0`

如果出现：

`Uncorrectable : >0`

则需要进一步排查。

重点检查：

- GPU 温度
- GPU 散热
- 服务器电源
- PCIe 链路
- NVIDIA Driver
- 系统日志
- GPU 硬件

查看 ECC：

```bash
nvidia-smi -q -d ECC
```

也可以：

```bash
nvidia-smi -q -d ECC | grep "Uncorrectable"
```

---

# 六、GPU 性能状态 Perf

`Perf` 表示 GPU 当前性能状态。

常见状态：

| Perf | 状态 | 常见场景 |
| --- | --- | --- |
| `P0` | 高性能状态 | 高性能计算 |
| `P2` | 常见计算状态 | 深度学习训练、推理、CUDA 计算 |
| `P8` | 低功耗状态 | GPU 空闲 |

> **重点：**
>
> 对 Tesla 等数据中心 GPU 来说，计算任务运行时处于 `P2` 是正常现象。
>
> 不应该简单地认为：
>
> `P2 = GPU 性能不足`
>
> 判断 GPU 性能是否正常，需要结合实际任务吞吐量、GPU 利用率、功耗、频率和温度综合判断。

---

# 七、GPU 实时监控

## 7.1 nvidia-smi 持续刷新

每 1 秒刷新：

```bash
nvidia-smi -l 1
```

每 5 秒刷新：

```bash
nvidia-smi -l 5
```

## 7.2 使用 watch

每 2 秒刷新：

```bash
watch -n 2 nvidia-smi
```

---

# 八、GPU 进程排查

执行：

```bash
nvidia-smi
```

在输出底部的 `Processes` 区域，可以看到当前使用 GPU 的进程。

例如：

```bash
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|=============================================================================|
|    0   N/A  N/A      1234      C   python                         12000MiB |
+-----------------------------------------------------------------------------+
```

其中：

`PID = 1234`

表示进程 ID。

查看进程：

```bash
ps -ef | grep 1234
```

正常结束进程：

```bash
kill 1234
```

强制结束进程：

```bash
kill -9 1234
```

> **注意：**
>
> `kill -9` 会直接强制终止进程，可能造成任务数据丢失。
>
> 生产环境执行前需要确认该进程可以安全终止。

---

# 九、GPU 性能与功耗控制

部分 GPU 支持手动调整 GPU 时钟。

设置应用时钟：

```bash
sudo nvidia-smi -ac <内存频率>,<核心频率>
```

例如：

```bash
sudo nvidia-smi -ac <MEM_CLOCK>,<GRAPHICS_CLOCK>
```

恢复默认设置：

```bash
sudo nvidia-smi -rac
```

> **注意：**
>
> GPU 频率和功耗属于高级配置。
>
> 生产环境修改前，需要确认：
>
> - GPU 型号是否支持
> - NVIDIA Driver 是否支持
> - 当前频率组合是否有效
> - 修改后是否会导致 GPU 不稳定

---

# 十、常见 GPU 故障排查

## 10.1 GPU 无法识别

执行：

```bash
nvidia-smi
```

如果 GPU 无法识别，可以检查：

```bash
lsmod | grep nvidia
```

以及：

```bash
systemctl status nvidia-persistenced
```

重点检查：

- NVIDIA Driver 是否安装
- NVIDIA Driver 是否正常加载
- GPU 是否被操作系统识别
- PCIe 是否正常
- 驱动与 Linux Kernel 是否兼容

---

## 10.2 显存占满但 GPU 利用率很低

例如：

`Memory-Usage: 15000MiB / 16384MiB`

`GPU-Util: 0%`

不一定是 GPU 故障。

可能原因：

- 模型已经加载到 GPU
- 程序正在等待
- CPU 数据处理成为瓶颈
- CUDA Stream 等待
- 程序同步阻塞
- 数据加载速度不足

建议结合：

```bash
nvidia-smi
```

以及：

- CPU 利用率
- GPU 进程
- 程序日志
- 数据加载速度

综合判断。

---

## 10.3 GPU 温度过高

执行：

```bash
nvidia-smi
```

重点查看：

- `Temp`
- `Fan`
- `Pwr:Usage/Cap`

检查：

- GPU 风扇
- 服务器风道
- 散热器
- 机房温度
- GPU 是否长期满载
- 是否存在异常降频

---

## 10.4 ECC 错误持续增长

执行：

```bash
nvidia-smi -q -d ECC
```

重点关注：

`Uncorrectable`

如果错误持续增长，建议：

1. 停止相关计算任务。
2. 检查 GPU 温度。
3. 检查服务器电源。
4. 检查系统日志。
5. 检查 PCIe 状态。
6. 重启服务器后继续观察。
7. 如果问题仍然存在，联系硬件供应商。

---

# 十一、GPU 健康检查

## 11.1 基础检查

```bash
nvidia-smi
```

确认：

- GPU 可以正常识别
- GPU 型号正确
- Driver Version 正确
- CUDA Version 符合预期
- 温度正常
- 功耗正常
- 显存使用正常
- GPU 利用率符合当前任务

## 11.2 ECC 检查

```bash
nvidia-smi -q -d ECC | grep "Uncorrectable"
```

正常示例：

`Uncorrectable : 0`

## 11.3 详细 GPU 检查

```bash
nvidia-smi -q
```

## 11.4 CUDA 计算检查

如果安装了 CUDA Samples，可以运行：

```bash
./deviceQuery
```

如果 GPU 可以正常识别，并且 CUDA 示例程序能够正常执行，则说明 CUDA 和 GPU 基础计算环境基本正常。

---

# 十二、日常 GPU 巡检重点

日常巡检建议重点关注以下指标：

| 指标 | 主要用途 |
| --- | --- |
| `Temp` | 判断 GPU 温度和散热 |
| `GPU-Util` | 判断 GPU 是否处于计算负载 |
| `Memory-Usage` | 判断显存是否存在不足风险 |
| `Pwr:Usage/Cap` | 判断 GPU 功耗是否接近限制 |
| `Perf` | 判断 GPU 当前性能状态 |
| `ECC` | 判断是否存在硬件错误 |
| `Processes` | 定位占用 GPU 的进程 |

---

# 十三、常用命令速查

| 操作 | 命令 |
| --- | --- |
| 查看 GPU 基本状态 | `nvidia-smi` |
| 每秒刷新 GPU 状态 | `nvidia-smi -l 1` |
| 每 2 秒刷新 GPU 状态 | `watch -n 2 nvidia-smi` |
| 查看详细 GPU 信息 | `nvidia-smi -q` |
| 查看 ECC 信息 | `nvidia-smi -q -d ECC` |
| 查看不可纠正 ECC | `nvidia-smi -q -d ECC \| grep "Uncorrectable"` |
| 开启持久模式 | `sudo nvidia-smi -pm 1` |
| 关闭持久模式 | `sudo nvidia-smi -pm 0` |
| 设置计算模式 | `sudo nvidia-smi -i <GPU_ID> -c <模式代码>` |
| 查看进程 | `ps -ef \| grep <PID>` |
| 正常结束进程 | `kill <PID>` |
| 强制结束进程 | `kill -9 <PID>` |
| 设置 GPU 时钟 | `sudo nvidia-smi -ac <内存频率>,<核心频率>` |
| 恢复默认设置 | `sudo nvidia-smi -rac` |

---

# 十四、GPU 故障判断原则

判断 GPU 是否异常时，不要只看一个指标。

建议综合分析：

`GPU 利用率 + 显存使用率 + 温度 + 功耗 + Perf + ECC + 进程信息 + 实际任务吞吐量`

通过这些指标，可以快速判断：

- GPU 是否被正确识别
- GPU 是否正在计算
- 是否存在显存不足
- 是否存在温度异常
- 是否存在功耗限制
- 是否存在异常进程
- 是否存在硬件 ECC 错误
- 是否存在 GPU 性能瓶颈

---

# 十五、推荐的日常巡检流程

## 第一步：查看 GPU 状态

```bash
nvidia-smi
```

## 第二步：检查 ECC

```bash
nvidia-smi -q -d ECC | grep "Uncorrectable"
```

## 第三步：持续观察

```bash
nvidia-smi -l 1
```

## 第四步：发现异常进程时定位 PID

```bash
ps -ef | grep <PID>
```

## 第五步：检查详细 GPU 信息

```bash
nvidia-smi -q
```

## 第六步：必要时运行 CUDA 测试

```bash
./deviceQuery
```

---

# 十六、核心总结

`nvidia-smi` 是 NVIDIA GPU 服务器日常运维和故障排查最重要的基础工具之一。

日常最值得关注的指标主要包括：

1. **GPU 温度**
   - 判断散热是否正常。

2. **GPU 利用率**
   - 判断 GPU 是否真正处于计算状态。

3. **显存使用率**
   - 判断是否存在显存不足风险。

4. **GPU 功耗**
   - 判断是否接近功耗限制。

5. **Perf**
   - 判断当前 GPU 性能状态。

6. **ECC 错误**
   - 判断 GPU 是否存在潜在硬件问题。

7. **GPU Processes**
   - 定位具体的 GPU 使用进程。

> **核心原则：**
>
> 不要只通过单一指标判断 GPU 是否异常。
>
> 应结合 **GPU 利用率 + 显存使用率 + 温度 + 功耗 + Perf + ECC + 进程信息 + 实际任务吞吐量** 进行综合判断。
>
> 只有综合分析这些指标，才能更加准确地定位 GPU 的性能瓶颈、资源问题和硬件故障。
