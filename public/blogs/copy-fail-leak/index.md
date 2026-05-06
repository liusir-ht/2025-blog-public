# Copy Fail 漏洞深度解析：潜伏九年的Linux内核提权漏洞

## 一、漏洞概述

Copy Fail（CVE-2026-31431，CVSS 7.8，高危）是2026年4月底被公开披露的一个**Linux内核本地权限提升漏洞**。该漏洞影响自2017年以来的所有主流Linux发行版，攻击者可通过一个仅732字节的Python脚本，从无特权用户权限提升至root权限。

**核心危害**：攻击者获得root权限后，可完全控制目标系统，安装后门、窃取数据、横向移动。由于利用过程仅修改内存中的页缓存（page cache），**磁盘上的文件保持不变**，传统基于文件完整性校验的安全检测工具（如AIDE、Tripwire）无法察觉任何异常。

## 二、技术背景：问题的根源

### 2.1 漏洞所处的位置

Copy Fail存在于Linux内核的**加密子系统（crypto/）**，具体是`authencesn`（Authenticated Encryption with Associated Data Extended Sequence Number）模板中。这个模板主要用于IPsec协议中对扩展序列号（ESN）的支持。

### 2.2 “组合爆炸”式的成因：三个“合理设计”的致命组合

Copy Fail最值得关注的特点是：它不是单一编码错误，而是**三个各自合理、独立审查无问题的代码变更，在数年间逐步叠加，最终形成一个致命漏洞**。

| 时间 | “合理”的变更 | 安全视角的隐患 |
|------|-------------|---------------|
| **2011年** | `authencesn`算法实现被引入内核 | 算法本身正常，但设计时假设输出缓冲区与输入缓冲区完全隔离 |
| **2015年** | `splice()`零拷贝系统调用支持被引入 | `splice()`可将只读文件的页缓存引用直接传递给内核子系统，模糊了“只读”边界 |
| **2017年1月** | commit `72548b093ee3`——为提升性能，在`algif_aead`模块中引入AEAD解密时的**原地（in-place）处理优化** | 原本应该使用独立临时缓冲区的操作，改为直接在传入的页缓存上覆写 |

这三个变更的叠加，创造了一个危险的攻击面：`AF_ALG`套接字 + `splice()` + in-place加密优化 = 无特权用户可以**向任意只读文件的页缓存写入4字节数据**。

### 2.3 漏洞的本质：一个被遗忘的“临时缓冲区”

当`authencesn`算法处理数据时，它会使用一块内存区域作为**临时缓冲区（scratch space）**进行字节重排。由于2017年的in-place优化，这个临时缓冲区**直接指向了调用者传入的页缓存页面**，而这些页缓存页面原本可能属于**任何可读文件**（包括`setuid-root`二进制文件，如`/usr/bin/su`、`/usr/bin/sudo`）。

在重排过程中，算法向临时缓冲区写入4字节数据。当这4字节恰好命中某些关键位置（如`setuid`文件的特定字节偏移），该文件的**内存副本**就被篡改了——下次执行时，文件行为被改变，从而执行攻击者的恶意代码，并继承`setuid`权限（即root权限）。

## 三、漏洞影响范围

### 3.1 受影响版本

- **时间范围**：2017年1月（commit 72548b093ee3合入）至2026年4月（补丁发布）
- **内核版本**：所有在此期间发布的Linux内核
- **已修复版本**：Linux kernel 6.18.22+、6.19.12+、7.0+

### 3.2 受影响发行版

已验证可稳定利用的发行版包括：
- Ubuntu 22.04 LTS / 24.04 LTS
- Amazon Linux 2023
- RHEL 8/9/10
- SUSE Linux 15.6 / SUSE 16
- Debian 12
- Arch Linux（已修复）

### 3.3 受影响环境

| 环境类型 | 风险等级 | 说明 |
|---------|---------|------|
| **多租户Linux主机** | 🔴 高危 | 共享主机上的任意用户都可提权 |
| **Kubernetes/容器集群** | 🔴 高危 | 容器内进程通常可访问宿主机的`AF_ALG`子系统，可实现**容器逃逸**至宿主机 |
| **CI/CD运行环境** | 🔴 高危 | CI job通常运行不可信代码，成功利用后可横向突破 |
| **云SaaS平台** | 🔴 高危 | 不同租户的代码在同一节点上运行 |
| **WSL2** | 🟡 受影响 | Windows Subsystem for Linux 2同样存在漏洞 |

## 四、为什么“Copy Fail”如此危险？

### 4.1 极高的可靠性

与许多需要竞态条件（race condition）或内存地址猜测的提权漏洞不同，Copy Fail的利用是**确定性的**——一次执行即可成功提权，无需针对不同内核版本调整偏移量。

### 4.2 极低的利用门槛

- 仅需**本地无特权用户账户**
- 无需网络访问
- 无需内核调试特性
- 无需预装任何特殊工具
- 一个**732字节的Python脚本**即可完成

### 4.3 难以检测的隐蔽性

这是Copy Fail最令人不安的特性：**所有修改仅在内存中发生，磁盘上的文件从未被触碰**。

- ✅ 文件哈希（SHA256/MD5）验证不会发现任何变化
- ✅ `stat`文件修改时间（mtime）不变
- ✅ 系统重启后，所有恶意修改自动消失（但攻击者的root权限早已落地）

这意味着：
- 事后取证几乎不可能确定漏洞是否被利用过
- 传统的“完整性监控”完全失效
- 攻击者留下的唯一痕迹，是进程执行链中的异常行为

### 4.4 极低的检测可见性

利用过程只使用**完全合法且正常的系统调用**：`socket()`（AF_ALG）、`splice()`、`sendmsg()`。这些系统调用在日常操作中极为常见，传统的基于规则的IDS/EDR很难将其与正常行为区分开来。

### 4.5 PoC迅速公开

漏洞披露后24小时内，GitHub上已出现多份可用利用代码。微软在PoC公开后观察到**有限的在野利用**，CISA已将该漏洞列入已知被利用漏洞目录（KEV catalog）。

## 五、与其他著名Linux提权漏洞的对比

| 漏洞 | 漏洞类型 | 写入量 | 稳定性 | 检测难度 |
|------|---------|-------|--------|---------|
| **Dirty Cow** (CVE-2016-5195) | 竞态条件 | 任意 | 不稳定 | 中 |
| **Dirty Pipe** (CVE-2022-0847) | 页缓存损坏 | 任意 | 高 | 中 |
| **Copy Fail** (CVE-2026-31431) | **逻辑缺陷（无需竞态）** | **4字节（可控）** | **100%** | **极高** |

Copy Fail与Dirty Pipe的主要区别在于：Dirty Pipe利用的是**pipe缓冲区标志位**的缺陷，而Copy Fail是**加密子系统内部逻辑的缺陷**，根本原因完全不同。Dirty Cow作为“祖爷爷”级漏洞，仍需要复杂的竞态条件利用，成功率远低于Copy Fail。

## 六、修复与缓解措施

### 6.1 官方修复

- 补丁已合入Linux主线：commit `a664bf3d603d`
- 稳定内核版本：6.18.22、6.19.12、7.0

### 6.2 操作建议

**优先级（高 → 低）**：
1. **立即更新内核**：`sudo apt update && sudo apt upgrade`（Ubuntu/Debian）或对应发行版的包管理器更新
2. **无法立即更新时，封锁攻击路径**：
   ```bash
   # 禁止加载 algif_aead 模块
   echo "install algif_aead /bin/false" | sudo tee /etc/modprobe.d/disable-algif_aead.conf
   sudo rmmod algif_aead 2>/dev/null

## 尝试复现

### 环境介绍
| 组件名称|组件版本|作用|
|--|--|--|
|Ubuntu|24.04|开源的Linux操作系统|
|Python| 3.12| 生态丰富的程序语言|

### 查看系统的基本信息
```
# 查看当前内核版本
root@xxxx:/home/test-user/copy-fail-CVE-2026-31431#  uname -a
Linux VM-37-57-ubuntu 6.8.0-51-generic #52-Ubuntu SMP PREEMPT_DYNAMIC Thu Dec  5 13:09:44 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
root@xxxx:/home/test-user/copy-fail-CVE-2026-31431#  uname -r
6.8.0-51-generic

# 检查 algif_aead 模块是否已加载
root@xxxx:/home/test-user/copy-fail-CVE-2026-31431# lsmod | grep algif_aead
algif_aead             16384  0
af_alg                 32768  1 algif_aead

root@xxxxx:/home/testuser/copy-fail-CVE-2026-31431# dpkg -l kmod | grep '^ii'
ii  kmod           31+20240202-2ubuntu7 amd64        tools for managing Linux kernel modules

# 检查 AEAD 接口是否可用（结果为 y 或 m 表示受影响）
zgrep CONFIG_CRYPTO_USER_API_AEAD /proc/config.gz 2>/dev/null
```

### 复现核心步骤
#### 创建普通用户
```
useradd test-user
```

#### 下载复现代码(把代码放到普通用户的家目录下面)
```
git clone https://github.com/theori-io/copy-fail-CVE-2026-31431.git
cd copy-fail-CVE-2026-31431
```

#### 使用普通用户复现
```
root@xxxx:/home/test-user/copy-fail-CVE-2026-31431# chown  -R test-user:test-user  /home/test-user/
root@xxxx:/home/test-user/copy-fail-CVE-2026-31431# su - test-user
$ python3 copy_fail_exp.py  #执行提权脚本
# id 
uid=0(root) gid=1004(test-user) groups=1004(test-user) #已经是root用户了
# stat /root/nohup.out
  File: nohup.out
  Size: 7651      	Blocks: 16         IO Block: 4096   regular file
Device: 253,2	Inode: 562777      Links: 1
Access: (0600/-rw-------)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-05-06 16:05:44.946026497 +0800
Modify: 2026-03-03 15:26:01.528773651 +0800
Change: 2026-03-03 15:26:01.528773651 +0800
 Birth: 2026-01-19 16:11:39.672126018 +0800
# cat /root/nohup.out | head -10
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
Traceback (most recent call last):
  File "/usr/lib/python3.12/socketserver.py", line 692, in process_request_thread
    self.finish_request(request, client_address)
  File "/usr/lib/python3.12/http/server.py", line 1311, in finish_request

```