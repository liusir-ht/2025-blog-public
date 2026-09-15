# Linux 服务器磁盘清理与 inode 排查指南

## 一、背景

服务器磁盘满是生产环境最常见的故障之一。

典型表现：

```text
No space left on device
```

但磁盘问题并不只有“文件太大”这一种情况。

常见类型包括：

```text
1. 真正的磁盘容量用满
2. 文件已经 rm，但进程仍然占用
3. inode 用满
4. Docker / containerd 占用大量空间
5. systemd journal 过大
6. 应用日志未轮转
7. /tmp 临时文件堆积
8. MySQL binlog / slow log / relay log 过大
9. Kubernetes 容器日志持续增长
10. ext4 reserved blocks
```

所以磁盘清理首先要判断：

> 到底是 block 用满，还是 inode 用满，还是删除后的文件没有真正释放。

---

## 二、先查看磁盘容量

执行：

```bash
df -h
```

案例输出：

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1       100G   97G  3.0G  98% /
/dev/vdb1       500G  231G  269G  47% /data
```

重点关注：

```text
Use%
```

例如：

```text
/dev/vda1  98%
```

说明根分区已经存在明显风险。

生产环境通常建议：

```text
< 70%      正常
70%~80%    关注
80%~90%    告警
> 90%      高风险
> 95%      建议立即处理
```

具体阈值应根据磁盘容量和业务增长速度调整。

---

## 三、判断是不是 inode 用满

执行：

```bash
df -i
```

案例输出：

```text
Filesystem       Inodes    IUsed   IFree IUse% Mounted on
/dev/vda1       6553600  6553501      99  100% /
/dev/vdb1      32768000  3021841 29746159   10% /data
```

这里：

```text
IUse% = 100%
```

说明不是磁盘容量本身用完，而是 inode 用光。

此时即使 `df -h` 还有几十 GB 空间，也可能出现：

```text
No space left on device
```

---

## 四、df 满但 du 看起来不大

这是生产环境非常典型的情况。

例如：

```bash
df -h /
```

输出：

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1       100G   95G  5.0G  95% /
```

但是：

```bash
du -sh /* 2>/dev/null
```

输出加起来只有：

```text
约 55G
```

出现：

```text
df = 95G
du = 55G
```

相差 40G。

这种情况第一优先级应该怀疑：

```text
文件被 rm 了
但是进程仍然持有文件描述符
```

Linux 只有在最后一个文件描述符关闭之后，文件占用的数据块才会真正释放。

---

## 五、查找 deleted 但仍被进程占用的文件

执行：

```bash
lsof +L1
```

或者：

```bash
lsof | grep '(deleted)'
```

案例输出：

```text
COMMAND   PID USER   FD   TYPE DEVICE    SIZE/OFF NLINK     NODE NAME
java    23871 app    12w   REG  253,1 21474836480     0 10488312 /var/log/app/access.log (deleted)
nginx    1821 root    7w   REG  253,1  8589934592     0 10491221 /var/log/nginx/access.log (deleted)
```

这里可以看出：

```text
Java 仍占用 20G
Nginx 仍占用 8G
```

虽然文件已经被删除，但是磁盘空间并没有释放。

---

## 六、使用 /proc 确认 deleted 文件

假设 PID：

```text
23871
```

执行：

```bash
ls -l /proc/23871/fd | grep deleted
```

案例输出：

```text
l-wx------ 1 app app 64 Sep 15 12:32 12 -> /var/log/app/access.log (deleted)
```

查看 fd 对应文件大小：

```bash
ls -lh /proc/23871/fd/12
```

案例输出：

```text
l-wx------ 1 app app 64 Sep 15 12:32 /proc/23871/fd/12 -> /var/log/app/access.log (deleted)
```

进一步可以：

```bash
cat /proc/23871/fdinfo/12
```

案例：

```text
pos:    21474836480
flags:  0102001
mnt_id: 31
ino:    10488312
```

`pos` 已经达到 20GB 左右，说明这个日志文件确实非常大。

---

## 七、没有 lsof 时怎么查 deleted 文件

可以直接使用 `/proc`：

```bash
find /proc/[0-9]*/fd -lname '*deleted*' -ls 2>/dev/null
```

案例：

```text
123456 0 l-wx------ 1 app app 64 Sep 15 12:33 /proc/23871/fd/12 -> /var/log/app/access.log (deleted)
```

查看哪些进程存在 deleted fd：

```bash
for pid in /proc/[0-9]*; do
  ls -l "$pid/fd" 2>/dev/null | grep deleted >/dev/null || continue
  echo "PID=${pid##*/} CMD=$(tr '\0' ' ' < "$pid/cmdline" 2>/dev/null)"
done
```

案例输出：

```text
PID=1821 CMD=nginx: master process /usr/sbin/nginx
PID=23871 CMD=java -Xms8g -Xmx8g -jar app.jar
```

---

## 八、如何安全释放 deleted 文件空间

### 1. 最推荐：让进程重新打开日志

例如 Nginx：

```bash
kill -USR1 $(cat /run/nginx.pid)
```

Nginx 收到 `USR1` 后会重新打开日志文件。

然后执行：

```bash
df -h
```

案例：

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1       100G   66G   34G  67% /
```

说明旧文件描述符释放成功。

### 2. 普通业务服务重启

例如：

```bash
systemctl restart your-service
```

服务重新启动后 fd 会被关闭，从而释放空间。

### 3. 应急：truncate fd

如果暂时不能重启，并且确认文件只是普通日志：

```bash
truncate -s 0 /proc/23871/fd/12
```

或者：

```bash
: > /proc/23871/fd/12
```

然后：

```bash
df -h
```

需要特别注意：

> 数据库文件、binlog、WAL、数据文件、索引文件等，不要随便对 fd 执行 truncate。

这种方法更适合确认无业务数据价值的日志文件。

---

## 九、找到真正占空间最大的目录

先从根目录查：

```bash
du -xsh /* 2>/dev/null | sort -h
```

案例输出：

```text
120M    /boot
1.1G    /etc
4.7G    /usr
6.2G    /home
18G     /var
51G     /data
```

假设 `/var` 很大，继续：

```bash
du -xsh /var/* 2>/dev/null | sort -h
```

案例：

```text
180M    /var/cache
2.1G    /var/lib
14G     /var/log
```

继续：

```bash
du -xsh /var/log/* 2>/dev/null | sort -h
```

案例：

```text
500M    /var/log/messages
1.8G    /var/log/nginx
11G     /var/log/app
```

这样逐层定位，避免一上来直接对整块磁盘执行非常重的 `find`。

---

## 十、查找大文件

查大于 1GB 的文件：

```bash
find / -xdev -type f -size +1G -printf '%s %p\n' 2>/dev/null \
| sort -nr \
| head -20 \
| numfmt --field=1 --to=iec
```

案例输出：

```text
21G /var/log/app/access.log
12G /data/service/gc.log
8.0G /var/lib/docker/containers/xxx/xxx-json.log
4.5G /var/log/messages
```

也可以针对具体磁盘：

```bash
find /data -xdev -type f -size +500M -ls 2>/dev/null
```

---

## 十一、inode 满的本质

inode 满一般不是大文件导致，而是**海量小文件**导致。

比如：

```text
磁盘容量：100G
实际使用：20G
inode：100%
```

这时创建一个 1KB 文件也可能失败。

常见 inode 杀手：

- 大量小日志；
- `/tmp` 临时文件；
- 缓存目录；
- Jenkins workspace；
- npm/pnpm/node_modules；
- Python venv；
- Docker overlay2；
- containerd snapshot；
- PHP session；
- 邮件队列；
- 应用一请求一个文件；
- 海量空文件。

---

## 十二、定位 inode 使用最多的目录

推荐：

```bash
du --inodes -x -d 2 / 2>/dev/null | sort -n | tail -30
```

案例输出：

```text
  11220  /usr/share
  35120  /var/log
  98411  /var/cache
 321442  /var/lib/docker
2819221  /data/cache
3258411  /
```

这里：

```text
/data/cache = 2,819,221 inode
```

说明问题非常明显。

继续：

```bash
du --inodes -x -d 2 /data/cache 2>/dev/null | sort -n | tail -20
```

案例：

```text
122341  /data/cache/20260913
931220  /data/cache/20260914
1724410 /data/cache/20260915
```

可以判断当天缓存文件异常增长。

---

## 十三、统计目录文件数量

例如：

```bash
find /data/cache -xdev -type f | wc -l
```

案例输出：

```text
2819093
```

说明存在约 281 万个文件。

也可以分目录统计：

```bash
for i in /data/cache/*; do
  printf '%-50s ' "$i"
  find "$i" -xdev -type f 2>/dev/null | wc -l
done | sort -k2 -n
```

案例：

```text
/data/cache/20260913                              122320
/data/cache/20260914                              931201
/data/cache/20260915                             1724390
```

---

## 十四、清理 inode 时不要直接 rm 数百万文件

如果单目录有数百万文件，直接：

```bash
rm -rf /data/cache/*
```

可能导致：

- rm 进程长时间运行；
- I/O 抖动；
- metadata 压力大；
- 应用请求超时；
- load 升高。

生产环境更建议分批清理。

例如清理 7 天前文件：

```bash
find /data/cache -type f -mtime +7 -print0 | xargs -0 -r -n 1000 rm -f
```

案例输出一般为空；可以通过：

```bash
df -i
```

观察 inode 是否下降。

---

## 十五、日志文件清理

先找日志：

```bash
find /var/log -type f -size +500M -ls 2>/dev/null
```

案例：

```text
10488312 5242896 -rw-r--r-- 1 root root 5368709120 Sep 15 12:00 /var/log/app/app.log
```

如果只是普通文本日志并且确认允许清空，可以：

```bash
truncate -s 0 /var/log/app/app.log
```

不要直接：

```bash
rm /var/log/app/app.log
```

否则进程如果仍持有 fd，可能出现前面提到的 deleted-but-open 问题。

---

## 十六、配置 logrotate 避免日志再次打满

案例：

```text
/etc/logrotate.d/app
```

内容：

```text
/var/log/app/*.log {
    daily
    rotate 14
    missingok
    notifempty
    compress
    delaycompress
    copytruncate
}
```

含义：

```text
daily         每天轮转
rotate 14     保留 14 份
compress      gzip 压缩
delaycompress 延迟一个周期压缩
copytruncate  复制后清空原文件
```

对于 Nginx，更推荐通过 `USR1` 让进程重新打开日志，而不是长期依赖 `copytruncate`。

---

## 十七、systemd journal 占用过大

查看：

```bash
journalctl --disk-usage
```

案例输出：

```text
Archived and active journals take up 12.4G in the file system.
```

保留最近 7 天：

```bash
journalctl --vacuum-time=7d
```

案例：

```text
Vacuuming done, freed 8.1G of archived journals.
```

限制最大容量：

```bash
journalctl --vacuum-size=2G
```

建议长期配置 `/etc/systemd/journald.conf`：

```text
SystemMaxUse=2G
RuntimeMaxUse=512M
```

然后：

```bash
systemctl restart systemd-journald
```

---

## 十八、Docker 磁盘占用排查

执行：

```bash
docker system df
```

案例：

```text
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          48        12        39.21GB   24.31GB (62%)
Containers      21        18        5.82GB    128MB (2%)
Local Volumes   14        11        18.2GB    2.1GB (11%)
Build Cache     51        0         12.4GB    12.4GB
```

查看详细：

```bash
docker system df -v
```

清理无用镜像之前必须确认：

```bash
docker image ls
```

常见清理：

```bash
docker image prune
```

清理 Build Cache：

```bash
docker builder prune
```

生产环境不建议未经确认直接执行：

```bash
docker system prune -a
```

因为可能删除后续发布仍需要的镜像缓存。

---

## 十九、Docker 容器 json 日志过大

查找：

```bash
find /var/lib/docker/containers -name '*-json.log' -size +500M -ls 2>/dev/null
```

案例：

```text
12390112 8388608 -rw-r----- 1 root root 8589934592 Sep 15 12:20 /var/lib/docker/containers/abc/abc-json.log
```

这里日志已经达到 8GB。

应急清理：

```bash
truncate -s 0 /var/lib/docker/containers/abc/abc-json.log
```

长期应该设置 Docker 日志轮转，例如：

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "5"
  }
}
```

修改后通常需要对容器进行重建或重启才能应用新的 logging 配置。

---

## 二十、containerd / Kubernetes 节点磁盘排查

查看：

```bash
du -xsh /var/lib/containerd/* 2>/dev/null | sort -h
```

案例：

```text
420M    /var/lib/containerd/io.containerd.metadata.v1.bolt
7.2G    /var/lib/containerd/io.containerd.content.v1.content
42G     /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs
```

如果 overlayfs 很大，说明镜像层或容器 writable layer 占用明显。

可以结合：

```bash
crictl images
crictl ps -a
```

案例：

```text
IMAGE                                      TAG       IMAGE ID        SIZE
registry.example.com/app/auth              v1.2.9    a81f3b001232    3.1GB
registry.example.com/app/preprocess        v2.3.1    d1187ace2021    5.8GB
```

清理 Kubernetes 节点镜像应优先依赖 kubelet ImageGC，而不是随意手工删除 containerd 数据目录。

---

## 二十一、Kubernetes 容器日志

常见目录：

```text
/var/log/containers
/var/log/pods
```

查看：

```bash
du -xsh /var/log/pods/* 2>/dev/null | sort -h | tail -20
```

案例：

```text
621M  /var/log/pods/default_gateway-6c79884ff9-x2pmd_xxx
2.8G  /var/log/pods/production_auth-7bd7569b55-rskml_xxx
```

查看大日志：

```bash
find /var/log/pods -type f -size +500M -ls 2>/dev/null
```

这类问题应同时检查 kubelet 的日志轮转参数，例如：

```text
containerLogMaxSize
containerLogMaxFiles
```

---

## 二十二、MySQL binlog 占用大量磁盘

查看 MySQL 数据目录：

```bash
du -sh /var/lib/mysql/* 2>/dev/null | sort -h | tail
```

案例：

```text
8.0G  /var/lib/mysql/mysql-bin.000026
8.0G  /var/lib/mysql/mysql-bin.000027
8.0G  /var/lib/mysql/mysql-bin.000028
```

查看保留策略：

```sql
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
SHOW VARIABLES LIKE 'expire_logs_days';
```

案例：

```text
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| expire_logs_days | 0     |
+------------------+-------+
```

`0` 表示不会按天自动过期。

MySQL 版本不同，推荐配置项也不同，需要根据具体版本确认。

不要直接在文件系统层执行：

```bash
rm -f mysql-bin.*
```

应该通过 MySQL 自己管理 binlog，避免破坏索引和复制状态。

---

## 二十三、/tmp 临时文件清理

查看：

```bash
du -xsh /tmp/* 2>/dev/null | sort -h | tail -20
```

案例：

```text
120M  /tmp/java_pid23871
2.1G  /tmp/audio-cache
12G   /tmp/preprocess_tmp
```

找 7 天前文件：

```bash
find /tmp -xdev -type f -mtime +7 -ls 2>/dev/null | head
```

确认后再清理：

```bash
find /tmp -xdev -type f -mtime +7 -delete
```

生产环境要先确认应用是否依赖这些临时文件。

---

## 二十四、查找超过 7 天的大日志

例如：

```bash
find /var/log -xdev -type f -mtime +7 -size +100M -printf '%TY-%Tm-%Td %s %p\n' \
| sort -k2 -nr \
| numfmt --field=2 --to=iec
```

案例：

```text
2026-09-01 2.4G /var/log/app/access.log.1
2026-08-31 1.8G /var/log/app/error.log.1
2026-08-21 820M /var/log/nginx/access.log-20260821
```

---

## 二十五、ext4 Reserved Blocks

有时 `df` 看起来磁盘已经接近满，但 root 用户仍能写少量数据。

ext4 默认会保留一部分 block 给 root 和系统使用。

查看：

```bash
tune2fs -l /dev/vda1 | grep -i reserved
```

案例：

```text
Reserved block count:     1310720
Reserved GDT blocks:      1024
```

对于大容量纯数据盘，有时会适当降低 reserved block 比例。

例如：

```bash
tune2fs -m 1 /dev/vdb1
```

表示保留 1%。

注意：

> 根分区不要在不了解影响的情况下随意调整。

---

## 二十六、文件删除了为什么 df 不下降

这是最常见的几个原因：

```text
1. 进程仍持有 deleted 文件
2. 删除的是 bind mount / 其他挂载目录里的文件
3. 文件系统 snapshot 仍持有数据块
4. 容器 overlayfs 中数据仍被引用
5. du 和 df 统计方式不同
6. ext4 reserved block
```

其中第一种最常见。

排查顺序：

```bash
df -h
lsof +L1
du -xsh /* 2>/dev/null | sort -h
mount
```

---

## 二十七、磁盘清理的生产安全原则

不要看到大文件就直接：

```bash
rm -rf
```

生产环境清理前至少确认：

```text
1. 文件属于哪个进程
2. 文件是否正在写入
3. 文件是否属于数据库
4. 是否可以重新生成
5. 是否影响回滚
6. 是否还有审计需求
7. 是否存在文件描述符未关闭
8. 删除后是否真的释放空间
```

重要数据建议先：

```bash
ls -lh FILE
stat FILE
lsof FILE
```

确认后再处理。

---

## 二十八、推荐的磁盘排查顺序

遇到磁盘告警，可以直接按照以下顺序执行：

```bash
# 1. 看容量
df -h

# 2. 看 inode
df -i

# 3. 看一级目录大小
du -xsh /* 2>/dev/null | sort -h

# 4. 看 inode 分布
du --inodes -x -d 2 / 2>/dev/null | sort -n | tail -30

# 5. 查 deleted 文件
lsof +L1

# 6. 找大文件
find / -xdev -type f -size +1G -printf '%s %p\n' 2>/dev/null \
| sort -nr | head -30 | numfmt --field=1 --to=iec

# 7. 查看 journal
journalctl --disk-usage
```

如果是容器节点，再补：

```bash
docker system df

du -xsh /var/lib/docker/* 2>/dev/null | sort -h

du -xsh /var/lib/containerd/* 2>/dev/null | sort -h

du -xsh /var/log/pods/* 2>/dev/null | sort -h | tail -20
```

---

## 二十九、常见场景速查表

| 现象 | 原因 | 处理方向 |
|---|---|---|
| df 100% | block 用完 | du/find 找大文件 |
| df -i 100% | inode 用完 | 找海量小文件 |
| df 很大但 du 很小 | deleted 文件 | lsof +L1 |
| /var/log 很大 | 日志未轮转 | logrotate |
| journal 很大 | systemd 日志 | vacuum + 限额 |
| Docker 很大 | 镜像/日志/cache | docker system df |
| containerd 很大 | snapshot/image | kubelet ImageGC |
| /var/log/pods 很大 | 容器日志 | kubelet log rotation |
| mysql-bin 很大 | binlog 未过期 | MySQL 内部清理 |
| /tmp 很大 | 临时文件未清理 | find + 生命周期策略 |

---

## 三十、总结

磁盘满问题可以浓缩为下面这条排查链路：

```text
df -h
  ↓
磁盘 block 是否满？
  ↓
df -i
  ↓
inode 是否满？
  ↓
du
  ↓
哪个目录占空间？
  ↓
lsof +L1
  ↓
是否存在 deleted-but-open？
  ↓
find / docker / containerd / journal
  ↓
确认文件归属后再清理
```

磁盘清理最重要的并不是“删文件”，而是判断：

```text
为什么磁盘会增长？
哪个组件在增长？
为什么没有自动轮转？
删除后是否真正释放？
是否还会再次发生？
```

