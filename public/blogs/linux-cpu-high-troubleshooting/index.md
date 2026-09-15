# Linux 服务器 CPU 100% 排查思路指南

## 一、背景

线上服务器出现 CPU 100% 时，最忌讳的是直接重启机器或者直接 `kill -9` 进程。

这样虽然可能让服务暂时恢复，但同时也会把最有价值的故障现场清掉，后续很难继续分析根因。

生产环境建议按照下面的顺序排查：

```text
CPU 100%
  │
  ├── 1. 确认 CPU 到底是谁在忙
  │      ├── user 高
  │      ├── system 高
  │      ├── iowait 高
  │      └── steal 高
  │
  ├── 2. 找到高 CPU 进程
  │
  ├── 3. 找到高 CPU 线程
  │
  ├── 4. 分析线程栈 / perf / GC
  │
  ├── 5. 判断是否为 softirq、网络、中断、锁竞争
  │
  └── 6. 最后再执行限流、扩容、重启、kill
```

核心原则是：

> 先保留现场，再恢复服务；先定位是哪个资源维度的问题，再深入到进程、线程和代码。

---

## 二、先判断是不是 CPU 真正打满

首先执行：

```bash
uptime
```

案例输出：

```text
11:32:18 up 126 days,  4:21,  3 users,  load average: 18.76, 17.42, 13.65
```

这里的三个值分别表示：

```text
1 分钟 Load Average
5 分钟 Load Average
15 分钟 Load Average
```

假设服务器是 8 核 CPU：

```text
load < 8       基本正常
load ≈ 8       CPU 基本满载
load > 8       已经存在任务排队
load >> 8      需要重点排查
```

查看 CPU 核数：

```bash
nproc
```

案例输出：

```text
8
```

也可以：

```bash
lscpu | grep '^CPU(s):'
```

案例输出：

```text
CPU(s):                          8
```

需要注意：

**Load 高不等于 CPU 一定高。**

Linux Load Average 还会统计不可中断睡眠状态的任务，例如大量磁盘 I/O 导致的 `D` 状态进程。

---

## 三、使用 top 判断 CPU 消耗类型

执行：

```bash
top
```

案例输出：

```text
top - 11:35:01 up 126 days,  4:24,  3 users,  load average: 18.20, 17.65, 14.03
Tasks: 286 total,   4 running, 282 sleeping,   0 stopped,   0 zombie
%Cpu(s): 88.4 us, 10.2 sy,  0.0 ni,  0.4 id,  0.2 wa,  0.0 hi,  0.8 si,  0.0 st
MiB Mem :  32104.7 total,   2104.2 free,  18654.6 used,  11345.9 buff/cache
MiB Swap:   4096.0 total,   3982.0 free,    114.0 used.  12340.2 avail Mem
```

重点关注：

```text
us   用户态 CPU
sy   内核态 CPU
wa   I/O Wait
hi   硬中断
si   软中断
st   虚拟机 Steal Time
id   Idle
```

典型判断：

| 指标 | 含义 | 常见方向 |
|---|---|---|
| us 高 | 用户态计算高 | Java/Python/Go/C++ 业务进程 |
| sy 高 | 内核态高 | 系统调用、网络、内核、锁 |
| wa 高 | I/O 等待 | 磁盘、云盘、文件系统 |
| si 高 | SoftIRQ 高 | 网络包、NET_RX、NET_TX |
| st 高 | CPU 被宿主机抢占 | 云主机超卖、宿主机资源竞争 |
| id 接近 0 | CPU 基本打满 | 继续找进程 |

例如：

```text
%Cpu(s): 95.1 us,  4.2 sy,  0.0 ni,  0.3 id,  0.1 wa,  0.0 hi,  0.3 si,  0.0 st
```

说明 CPU 主要消耗在业务用户态，优先找业务进程。

---

## 四、查看每个 CPU 核心使用率

执行：

```bash
mpstat -P ALL 1 5
```

案例输出：

```text
11:36:01     CPU    %usr   %nice    %sys %iowait   %irq  %soft  %steal  %idle
11:36:02     all   86.91    0.00    9.33    0.12    0.00   0.74    0.00   2.90
11:36:02       0   99.00    0.00    0.00    0.00    0.00   0.00    0.00   1.00
11:36:02       1   98.00    0.00    1.00    0.00    0.00   0.00    0.00   1.00
11:36:02       2   71.00    0.00   26.00    0.00    0.00   2.00    0.00   1.00
11:36:02       3   69.00    0.00   27.00    0.00    0.00   3.00    0.00   1.00
```

如果只有单核达到 100%，而其他核心很空闲：

```text
CPU0 100%
CPU1 20%
CPU2 15%
CPU3 10%
```

可能是：

- 单线程程序；
- 某个线程死循环；
- CPU affinity 绑定；
- 单队列网络中断；
- 单线程 GC 或串行计算。

---

## 五、使用 vmstat 查看系统整体压力

执行：

```bash
vmstat 1 5
```

案例输出：

```text
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in    cs us sy id wa st
18  0 116736 614820  92440 7312024    0    0     4    18 15982 31234 89 10  1  0  0
21  0 116736 611240  92440 7312452    0    0     0    12 17210 35570 91  8  1  0  0
20  0 116736 609116  92440 7312610    0    0     0     8 16982 34992 90  9  1  0  0
```

重点关注：

```text
r   正在运行或等待 CPU 的任务数
b   不可中断睡眠任务数
us  用户态
sy  系统态
wa  I/O Wait
cs  context switch
```

假设机器只有 8 核：

```text
r = 20
```

说明 runnable queue 已经非常高，大量线程正在等待 CPU 调度。

---

## 六、找到 CPU 最高的进程

执行：

```bash
ps -eo pid,ppid,user,stat,pcpu,pmem,etime,cmd --sort=-pcpu | head -20
```

案例输出：

```text
  PID  PPID USER     STAT %CPU %MEM     ELAPSED CMD
23871     1 app      Sl   682.3 18.5    02:17:31 java -Xms8g -Xmx8g -jar app.jar
21904     1 root     S     32.1  0.3    14:20:11 /usr/bin/python3 agent.py
 1292     1 root     S      5.4  0.1   126-04:20 /usr/bin/containerd
```

这里：

```text
java %CPU = 682.3
```

表示 Java 进程大约使用了 6.8 个 CPU 核心。

Linux 中多线程进程的 CPU 百分比可以超过 100%。

---

## 七、查看进程内部哪个线程 CPU 高

假设 Java PID：

```text
23871
```

执行：

```bash
top -H -p 23871
```

案例输出：

```text
  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
24102 app       20   0   15.2g   6.0g  22160 R 99.7 19.1  35:21.81 worker-17
24111 app       20   0   15.2g   6.0g  22160 R 98.9 19.1  34:58.13 worker-19
24088 app       20   0   15.2g   6.0g  22160 R 97.3 19.1  36:04.22 worker-08
```

也可以：

```bash
ps -Lp 23871 -o pid,tid,psr,pcpu,stat,comm --sort=-pcpu | head -20
```

案例输出：

```text
  PID   TID PSR %CPU STAT COMMAND
23871 24102   0 99.4 Rl   java
23871 24111   5 98.8 Rl   java
23871 24088   2 97.1 Rl   java
```

重点得到：

```text
TID = 24102
```

---

## 八、Java 高 CPU：线程 TID 转十六进制

Java `jstack` 中线程 ID 通常以十六进制 `nid` 展示。

执行：

```bash
printf '%x\n' 24102
```

案例输出：

```text
5e26
```

也就是：

```text
24102 -> 0x5e26
```

---

## 九、使用 jstack 定位 Java 高 CPU 线程

先保存线程栈：

```bash
jstack 23871 > /tmp/jstack-23871-$(date +%F-%H%M%S).log
```

查对应线程：

```bash
grep -A 30 -B 5 'nid=0x5e26' /tmp/jstack-23871-*.log
```

案例输出：

```text
"worker-17" #137 prio=5 os_prio=0 tid=0x00007f1a5c112800 nid=0x5e26 runnable [0x00007f19bcdf9000]
   java.lang.Thread.State: RUNNABLE
        at java.util.regex.Pattern$Curly.match2(Pattern.java:4277)
        at java.util.regex.Pattern$Curly.match(Pattern.java:4232)
        at java.util.regex.Pattern$GroupHead.match(Pattern.java:4660)
        at java.util.regex.Matcher.search(Matcher.java:1248)
        at java.util.regex.Matcher.find(Matcher.java:637)
        at com.example.service.TextService.parse(TextService.java:182)
```

从这个案例就可以判断：

```text
业务线程 RUNNABLE
        ↓
大量时间消耗在正则表达式匹配
        ↓
进一步检查 TextService.parse()
```

常见高 CPU 堆栈包括：

```text
正则表达式
JSON 序列化
死循环
字符串处理
压缩/解压
加密解密
自旋锁
大量集合遍历
```

---

## 十、连续抓取多次 jstack

只抓一次线程栈可能不够。

建议连续抓 3 次：

```bash
jstack 23871 > /tmp/jstack-1.log
sleep 5
jstack 23871 > /tmp/jstack-2.log
sleep 5
jstack 23871 > /tmp/jstack-3.log
```

如果某个线程三次都停留在同一代码位置：

```text
jstack-1 -> TextService.parse:182
jstack-2 -> TextService.parse:182
jstack-3 -> TextService.parse:182
```

那么这个位置就非常可疑。

---

## 十一、Java 高 CPU 还要检查 GC

执行：

```bash
jstat -gcutil 23871 1000 10
```

案例输出：

```text
  S0     S1      E      O      M     CCS    YGC    YGCT    FGC    FGCT     GCT
  0.00   0.00   18.43  99.86  96.21  89.23    352   29.221    71  526.334  555.555
  0.00   0.00    3.21  99.91  96.21  89.23    353   29.302    72  534.781  564.083
  0.00   0.00   12.82  99.94  96.21  89.23    353   29.302    72  534.781  564.083
```

这里需要重点关注：

```text
O     Old 区使用率
FGC   Full GC 次数
FGCT  Full GC 总耗时
```

案例中：

```text
Old ≈ 99.9%
FGC 持续增长
```

说明 CPU 高很可能是 Full GC 导致。

此时需要进一步分析：

```text
内存泄漏
大对象
堆配置过小
Metaspace
晋升失败
对象创建速度过快
```

---

## 十二、非 Java 程序使用 pidstat

假设进程 PID：

```text
21904
```

执行：

```bash
pidstat -p 21904 1
```

案例输出：

```text
11:45:40      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
11:45:41     1001     21904   92.00    3.00    0.00    0.00   95.00     3  python3
11:45:42     1001     21904   94.00    2.00    0.00    0.00   96.00     3  python3
```

查看线程：

```bash
pidstat -t -p 21904 1
```

案例输出：

```text
11:46:21      UID      TGID       TID    %usr %system   %CPU  Command
11:46:22     1001     21904         -   94.00    2.00  96.00  python3
11:46:22     1001         -     21942   93.00    1.00  94.00  worker
```

说明 `TID 21942` 是主要 CPU 消耗线程。

---

## 十三、使用 perf 查看 CPU 时间花在哪里

Linux 原生程序、Go、C/C++、Python Native 扩展等场景，可以使用 `perf`。

实时查看：

```bash
perf top -p 21904
```

案例输出：

```text
  31.45%  python3  libc.so.6       [.] memcpy
  21.81%  python3  libpython3.12   [.] _PyEval_EvalFrameDefault
  18.22%  python3  app.so          [.] parse_audio_frame
   8.71%  python3  libc.so.6       [.] malloc
```

这说明大量 CPU 时间消耗在：

```text
memcpy
Python Bytecode
parse_audio_frame
malloc
```

采样 30 秒：

```bash
perf record -F 99 -p 21904 -g -- sleep 30
```

查看报告：

```bash
perf report
```

---

## 十四、system CPU 高怎么排查

如果：

```text
%Cpu(s): 10 us, 85 sy, 1 id
```

说明重点不再是普通业务计算，而是内核态。

先执行：

```bash
pidstat -u 1
```

案例：

```text
12:01:10      UID       PID    %usr %system  %guest   %wait    %CPU  Command
12:01:11        0      8291    4.00   76.00    0.00    0.00   80.00  envoy
```

如果 `%system` 很高，需要考虑：

- 大量网络系统调用；
- 频繁 `read/write/send/recv`；
- 大量上下文切换；
- 锁竞争；
- page fault；
- 内核网络协议栈；
- eBPF / iptables / conntrack 等。

---

## 十五、SoftIRQ 高排查

查看：

```bash
mpstat -I ALL 1
```

或者：

```bash
cat /proc/softirqs
```

案例输出：

```text
                    CPU0       CPU1       CPU2       CPU3
NET_TX:             1240       9821       1033       1102
NET_RX:          9214471    8821932    9301022    9018221
TIMER:           4812300    4710211    4668110    4599821
SCHED:           2912001    2893411    2921102    2862311
```

如果 `NET_RX` 增长非常快，同时看到：

```text
ksoftirqd/0
ksoftirqd/1
```

CPU 很高，那么要继续排查：

```text
网卡 PPS
RSS/RPS/XPS
IRQ affinity
连接数
大量小包
iptables/conntrack
```

查看网卡流量：

```bash
sar -n DEV 1
```

案例输出：

```text
12:10:01 IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s
12:10:02 eth0    184321.0  175824.0   58231.4   49218.7
```

如果 PPS 达到几十万甚至更高，就需要重点检查网络侧。

---

## 十六、查看上下文切换

执行：

```bash
pidstat -w 1
```

案例输出：

```text
12:12:31      UID       PID   cswch/s nvcswch/s  Command
12:12:32     1001     23871   18422.0    9321.0  java
12:12:32     1001     21904     288.0      91.0  python3
```

其中：

```text
cswch/s    voluntary context switch
nvcswch/s  involuntary context switch
```

如果上下文切换异常高，需要考虑：

- 线程数量过多；
- 大量锁竞争；
- runnable 线程过多；
- CPU 不足；
- 线程池配置不合理。

---

## 十七、查看进程线程数量

执行：

```bash
ps -eLf | wc -l
```

案例：

```text
18342
```

查看单进程：

```bash
ps -Lf 23871 | wc -l
```

案例：

```text
2481
```

如果一个 Java 进程存在数千甚至上万线程，就要进一步排查线程池泄漏或线程创建异常。

---

## 十八、I/O Wait 高并不等于 CPU 性能不足

案例：

```text
%Cpu(s): 5.2 us, 4.1 sy, 0.0 ni, 12.0 id, 78.4 wa
```

这种情况 CPU 并不是在真正执行计算，而是在等待 I/O。

执行：

```bash
iostat -x 1
```

案例输出：

```text
Device            r/s     w/s   await  aqu-sz  %util
nvme0n1         821.0   944.0   48.21   12.42  99.80
```

如果：

```text
%util ≈ 100%
await 很高
```

则问题主要在存储侧。

进一步：

```bash
pidstat -d 1
```

找出哪个进程在大量读写磁盘。

---

## 十九、Steal 高：云主机宿主机争抢 CPU

如果：

```text
%st = 20%
```

例如：

```text
%Cpu(s): 50.2 us, 4.0 sy, 0.0 ni, 25.0 id, 0.1 wa, 0.0 hi, 0.7 si, 20.0 st
```

说明虚拟机有约 20% 的 CPU 时间被 Hypervisor 抢走。

这类问题通常不是应用自身造成，常见原因：

- 宿主机资源竞争；
- 云主机超卖；
- 突发型实例 CPU Credit 用尽；
- 云平台底层调度异常。

此时应该结合云监控和宿主机侧指标进一步确认。

---

## 二十、Kubernetes 节点 CPU 100% 排查

首先看节点：

```bash
kubectl top node
```

案例输出：

```text
NAME            CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-node-01     7800m        98%    18421Mi         58%
k8s-node-02     3210m        40%    17220Mi         54%
```

继续找 Pod：

```bash
kubectl top pod -A --sort-by=cpu
```

案例输出：

```text
NAMESPACE       NAME                         CPU(cores)   MEMORY(bytes)
production      auth-7bd7569b55-rskml       3921m        4812Mi
production      preprocess-66bbfc687-z89kn  1844m        3921Mi
kube-system     kube-proxy-x8k9m             210m         128Mi
```

查看容器：

```bash
kubectl top pod -n production auth-7bd7569b55-rskml --containers
```

案例：

```text
POD                      NAME     CPU(cores)   MEMORY(bytes)
auth-7bd7569b55-rskml    auth     3890m        4791Mi
auth-7bd7569b55-rskml    sidecar  31m          21Mi
```

说明主要 CPU 消耗来自 `auth` 容器。

宿主机可以继续：

```bash
crictl stats
```

案例：

```text
CONTAINER           NAME      CPU %     MEM        DISK
cfa0187c18f1        auth      387.12    4.7GiB     1.1GiB
```

---

## 二十一、生产环境建议保留的现场

CPU 100% 时建议立即保存：

```bash
date
uptime
nproc

 top -b -n 1 > /tmp/top-$(date +%F-%H%M%S).log
mpstat -P ALL 1 10 > /tmp/mpstat-$(date +%F-%H%M%S).log
vmstat 1 10 > /tmp/vmstat-$(date +%F-%H%M%S).log
pidstat -u -w 1 10 > /tmp/pidstat-$(date +%F-%H%M%S).log
```

Java 服务再补：

```bash
jstack <PID> > /tmp/jstack-$(date +%F-%H%M%S).log
jstat -gcutil <PID> 1000 10 > /tmp/jstat-$(date +%F-%H%M%S).log
```

如果允许使用 perf：

```bash
perf record -F 99 -p <PID> -g -- sleep 30
```

---

## 二十二、常见场景速查表

| 现象 | 常见原因 | 下一步 |
|---|---|---|
| us 90%+ | 业务计算 | ps、top -H、jstack、perf |
| sy 80%+ | 内核态 | pidstat、perf、strace、网络 |
| wa 80%+ | 存储 I/O | iostat、pidstat -d |
| si 高 | SoftIRQ | /proc/softirqs、sar -n DEV |
| st 高 | 宿主机竞争 | 云平台监控 |
| 单核 100% | 单线程 | mpstat、top -H |
| Java Old 99% | Full GC | jstat、GC log、heap dump |
| load 很高、CPU 不高 | D 状态 / I/O | ps state、iostat |
| cswch 很高 | 线程/锁竞争 | pidstat -w、线程栈 |

---

## 二十三、总结

CPU 100% 的排查可以浓缩成下面这条链路：

```text
uptime
  ↓
top / mpstat / vmstat
  ↓
判断 us / sy / wa / si / st
  ↓
ps 找进程
  ↓
top -H / pidstat 找线程
  ↓
jstack / jstat / perf
  ↓
定位代码、GC、网络、内核或存储
```
