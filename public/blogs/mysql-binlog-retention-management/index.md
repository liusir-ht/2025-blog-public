
# MySQL Binlog 保留策略与安全清理实践

## 1. 背景

在 MySQL 数据目录中，经常可以看到类似下面的文件：

```text
mysql-bin.000024
mysql-bin.000025
mysql-bin.000026
mysql-bin.index
```

其中：

- `mysql-bin.000026`：MySQL Binary Log（二进制日志）文件。
- `mysql-bin.index`：记录当前 MySQL 管理的 binlog 文件列表。

Binlog 主要用于：

- 主从复制 / Replication
- Point-in-Time Recovery（基于时间点恢复）
- 数据误操作恢复
- 数据变更追踪
- CDC / Canal / Debezium 等增量同步场景

如果没有配置合理的过期策略，binlog 会持续累积，最终可能导致 MySQL 数据盘被写满。

---

## 2. 查看是否开启 Binlog

```sql
SHOW VARIABLES LIKE 'log_bin';
```

正常开启时：

```text
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
+---------------+-------+
```

查看 binlog 文件前缀：

```sql
SHOW VARIABLES LIKE 'log_bin_basename';
```

查看当前已有的 binlog：

```sql
SHOW BINARY LOGS;
```

示例：

```text
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000024 | 954632841 |
| mysql-bin.000025 | 889325142 |
| mysql-bin.000026 | 536781223 |
+------------------+-----------+
```

---

## 3. 查看当前正在写哪个 Binlog

较老版本 MySQL 常用：

```sql
SHOW MASTER STATUS;
```

MySQL 8.4 可以使用：

```sql
SHOW BINARY LOG STATUS;
```

例如：

```text
File: mysql-bin.000026
Position: 183746291
```

表示当前正在写：

```text
mysql-bin.000026
```

当前正在使用的 binlog 不应该手工删除。

---

## 4. 查看 Binlog 自动过期配置

### 4.1 MySQL 5.6 / 5.7

主要使用：

```sql
SHOW VARIABLES LIKE 'expire_logs_days';
```

如果结果为：

```text
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| expire_logs_days | 0     |
+------------------+-------+
```

表示：

> Binlog 不按照 `expire_logs_days` 自动过期。

也就是说，如果没有人工执行 `PURGE BINARY LOGS`，binlog 很可能持续累积。

---

## 5. 设置 Binlog 保留 90 天

对于 MySQL 5.6 / 5.7，可以执行：

```sql
SET GLOBAL expire_logs_days = 90;
```

确认：

```sql
SHOW VARIABLES LIKE 'expire_logs_days';
```

预期：

```text
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| expire_logs_days | 90    |
+------------------+-------+
```

这表示 MySQL 后续会按照 **90 天** 的生命周期管理 binlog。

但需要注意：

> 设置为 90 天，并不等于执行 SQL 后立刻删除全部 90 天以前的文件。

自动清理通常会在 MySQL 启动、binlog rotation / flush 等时机触发。

---

## 6. 永久配置

仅执行：

```sql
SET GLOBAL expire_logs_days = 90;
```

通常只能保证当前 MySQL 实例运行期间有效。

MySQL 重启后，仍可能恢复配置文件中的值。

因此建议写入配置文件：

```ini
[mysqld]
expire_logs_days = 90
```

常见配置路径：

```text
/etc/my.cnf
/etc/mysql/my.cnf
/etc/mysql/mysql.conf.d/mysqld.cnf
```

修改后重启 MySQL：

```bash
systemctl restart mysqld
```

或者：

```bash
systemctl restart mysql
```

再次确认：

```sql
SHOW VARIABLES LIKE 'expire_logs_days';
```

---

## 7. 立即清理 90 天以前的 Binlog

如果磁盘中已经存在大量历史 binlog，仅修改 `expire_logs_days` 不一定立即释放空间。

可以手工执行：

```sql
PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 90 DAY);
```

含义：

```text
删除 90 天以前的 binlog
保留最近 90 天
```

也可以指定具体时间：

```sql
PURGE BINARY LOGS BEFORE '2026-06-16 00:00:00';
```

---

## 8. 按 Binlog 文件清理

也可以指定某个 binlog：

```sql
PURGE BINARY LOGS TO 'mysql-bin.000026';
```

注意：

```text
mysql-bin.000024  删除
mysql-bin.000025  删除
mysql-bin.000026  保留
mysql-bin.000027  保留
```

也就是说：

> `PURGE BINARY LOGS TO 'mysql-bin.000026'` 删除的是 `000026` 之前的文件，不包含 `000026` 本身。

---

## 9. 不要直接 rm Binlog

不推荐：

```bash
rm -f /var/lib/mysql/mysql-bin.0000*
```

原因是 MySQL 同时通过：

```text
mysql-bin.index
```

维护 binlog 文件列表。

直接从文件系统删除，可能造成：

- `mysql-bin.index` 与实际文件不一致
- MySQL 报错
- 主从复制异常
- Point-in-Time Recovery 链路中断
- 后续 `PURGE BINARY LOGS` 行为异常

正确方式应该使用：

```sql
PURGE BINARY LOGS ...;
```

让 MySQL 自己维护 binlog 与 index。

---

# 10. MySQL 各版本差异

这是配置 binlog 生命周期时最容易踩坑的地方。

## 10.1 MySQL 5.6

主要参数：

```ini
expire_logs_days = 90
```

动态修改：

```sql
SET GLOBAL expire_logs_days = 90;
```

特点：

- 使用 `expire_logs_days`
- 单位为天
- `0` 表示不按照该参数自动清理
- 适合传统 MySQL 5.x 环境

推荐：

```ini
[mysqld]
expire_logs_days = 90
```

---

## 10.2 MySQL 5.7

MySQL 5.7 同样主要使用：

```ini
expire_logs_days = 90
```

官方也明确建议，如果存在主从复制，保留时间不能小于可能出现的最大从库延迟时间。

例如：

```text
最大可能复制延迟：7 天
binlog 保留：90 天
```

这种情况下余量比较充足。

---

## 10.3 MySQL 8.0

MySQL 8.0 推荐使用：

```text
binlog_expire_logs_seconds
```

`expire_logs_days` 已经被标记为 Deprecated。

例如保留 90 天：

```text
90 × 24 × 60 × 60 = 7776000 秒
```

配置：

```ini
[mysqld]
binlog_expire_logs_seconds = 7776000
```

动态修改：

```sql
SET GLOBAL binlog_expire_logs_seconds = 7776000;
```

查看：

```sql
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
```

MySQL 8.0 的默认过期时间为：

```text
2592000 秒
```

即：

```text
30 天
```

需要注意：MySQL 8.0 中 `expire_logs_days` 和 `binlog_expire_logs_seconds` 不应该同时设置非 0 值。

生产环境建议直接统一使用：

```ini
binlog_expire_logs_seconds = 7776000
```

---

## 10.4 MySQL 8.0.29+

MySQL 8.0.29 开始增加了：

```text
binlog_expire_logs_auto_purge
```

查看：

```sql
SHOW VARIABLES LIKE 'binlog_expire_logs_auto_purge';
```

默认：

```text
ON
```

如果配置：

```ini
binlog_expire_logs_auto_purge = OFF
```

即使配置了：

```ini
binlog_expire_logs_seconds = 7776000
```

MySQL 也不会进行自动清理。

因此排查 MySQL 8.0.29+ 时建议同时检查：

```sql
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
SHOW VARIABLES LIKE 'binlog_expire_logs_auto_purge';
```

---

## 10.5 MySQL 8.2 / 8.4

从 MySQL 8.2 开始：

```text
expire_logs_days
```

已经被移除。

执行：

```sql
SHOW VARIABLES LIKE 'expire_logs_days';
```

可能无法再得到对应参数。

应该统一使用：

```ini
[mysqld]
binlog_expire_logs_seconds = 7776000
```

MySQL 8.4 推荐：

```sql
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
SHOW VARIABLES LIKE 'binlog_expire_logs_auto_purge';
```

90 天配置：

```ini
[mysqld]
binlog_expire_logs_seconds = 7776000
binlog_expire_logs_auto_purge = ON
```

---

## 10.6 MariaDB

MariaDB 与 Oracle MySQL 的演进路线并不完全一致。

MariaDB 仍支持：

```text
expire_logs_days
```

MariaDB 10.6.1 开始还支持：

```text
binlog_expire_logs_seconds
```

并且两个参数在较新的 MariaDB 中属于关联参数。

查看版本：

```sql
SELECT VERSION();
```

然后分别检查：

```sql
SHOW VARIABLES LIKE 'expire_logs_days';
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
```

不要直接根据 Oracle MySQL 8.0/8.4 的参数规则套用到 MariaDB。

---

# 11. 版本差异汇总

| 数据库版本 | 推荐参数 | 单位 | `expire_logs_days` 状态 | 推荐 90 天配置 |
|---|---|---:|---|---|
| MySQL 5.6 | `expire_logs_days` | 天 | 支持 | `expire_logs_days=90` |
| MySQL 5.7 | `expire_logs_days` | 天 | 支持 | `expire_logs_days=90` |
| MySQL 8.0 | `binlog_expire_logs_seconds` | 秒 | Deprecated | `7776000` |
| MySQL 8.0.29+ | `binlog_expire_logs_seconds` + `binlog_expire_logs_auto_purge` | 秒 | Deprecated | `7776000 + ON` |
| MySQL 8.2 | `binlog_expire_logs_seconds` | 秒 | 已移除 | `7776000` |
| MySQL 8.4 | `binlog_expire_logs_seconds` | 秒 | 已移除 | `7776000` |
| MariaDB < 10.6.1 | `expire_logs_days` | 天 | 支持 | `expire_logs_days=90` |
| MariaDB >= 10.6.1 | 两者均可，建议按版本规范配置 | 天 / 秒 | 支持 | 按实际版本选择 |

---

# 12. 生产环境推荐配置

## MySQL 5.6 / 5.7

```ini
[mysqld]
expire_logs_days = 90
```

检查：

```sql
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'expire_logs_days';
SHOW BINARY LOGS;
```

---

## MySQL 8.0

```ini
[mysqld]
binlog_expire_logs_seconds = 7776000
```

MySQL 8.0.29+ 可以进一步确认：

```ini
binlog_expire_logs_auto_purge = ON
```

检查：

```sql
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
SHOW VARIABLES LIKE 'binlog_expire_logs_auto_purge';
SHOW BINARY LOGS;
```

---

## MySQL 8.4

推荐：

```ini
[mysqld]
binlog_expire_logs_seconds = 7776000
binlog_expire_logs_auto_purge = ON
```

不要继续配置：

```ini
expire_logs_days = 90
```

因为该参数已经被移除。

---

# 13. 主从复制环境注意事项

这是生产环境最重要的一点。

假设：

```text
Master 已生成：mysql-bin.000100
Slave 还在读取：mysql-bin.000080
```

此时如果直接清理到：

```sql
PURGE BINARY LOGS TO 'mysql-bin.000090';
```

会导致从库后续需要的 binlog 被删除。

典型报错可能类似：

```text
Could not find first log file name in binary log index file
```

因此主从环境清理前，至少需要确认：

```sql
SHOW SLAVE STATUS\G
```

较新的 MySQL 可以查看：

```sql
SHOW REPLICA STATUS\G
```

重点关注当前复制读取到的 binlog 文件和位置。

原则：

> Binlog 保留周期必须大于业务能够接受的最大复制延迟窗口，同时还要考虑灾备恢复要求。

---

# 14. 为什么生产环境建议保留 90 天

保留多少天没有固定答案，需要综合：

```text
磁盘容量
    ↓
每日 Binlog 增量
    ↓
主从复制最大延迟
    ↓
PITR 恢复窗口
    ↓
备份策略
    ↓
合规要求
```

例如：

```text
每天 Binlog：20 GB
保留：90 天
```

理论空间：

```text
20 GB × 90 = 1800 GB
```

即约：

```text
1.8 TB
```

所以不能只设置保留天数，还需要估算磁盘容量。

---

# 15. 估算 Binlog 占用

系统层面：

```bash
du -sh /var/lib/mysql/mysql-bin.*
```

查看各文件：

```bash
ls -lh /var/lib/mysql/mysql-bin.*
```

MySQL 内部：

```sql
SHOW BINARY LOGS;
```

可以结合业务每天产生的 binlog 大小，计算 90 天的预估容量。

---

# 16. 推荐排查流程

生产环境建议按照下面的顺序检查：

```text
SELECT VERSION()
        ↓
SHOW VARIABLES LIKE 'log_bin'
        ↓
SHOW VARIABLES LIKE 'expire_logs_days'
        ↓
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds'
        ↓
SHOW VARIABLES LIKE 'binlog_expire_logs_auto_purge'
        ↓
SHOW BINARY LOGS
        ↓
检查主从复制状态
        ↓
确认 PITR / 备份策略
        ↓
设置自动过期策略
        ↓
必要时 PURGE 历史 Binlog
```

可以一次执行：

```sql
SELECT VERSION();
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'expire_logs_days';
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
SHOW VARIABLES LIKE 'binlog_expire_logs_auto_purge';
SHOW BINARY LOGS;
```

不同版本不存在的变量返回空是正常现象。

---

# 17. 当前场景

当前查询结果：

```text
expire_logs_days = 0
```

说明当前实例使用的是支持 `expire_logs_days` 的版本，并且当前没有通过该参数设置自动过期周期。

如果确认是 MySQL 5.6 / 5.7，并计划保留 90 天，可以配置：

```sql
SET GLOBAL expire_logs_days = 90;
```

同时写入：

```ini
[mysqld]
expire_logs_days = 90
```

如果已有超过 90 天的历史 binlog，需要立即回收空间时，可以在确认复制和恢复需求后执行：

```sql
PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 90 DAY);
```

最终形成：

```text
历史 Binlog
    ↓
手工 PURGE 一次
    ↓
最近 90 天 Binlog
    ↓
expire_logs_days = 90
    ↓
后续自动滚动清理
```

---

# 18. 总结

对于 MySQL Binlog，生产环境最核心的原则是：

1. 不要直接 `rm mysql-bin.*`。
2. 使用 `PURGE BINARY LOGS` 进行手工清理。
3. MySQL 5.6 / 5.7 使用 `expire_logs_days`。
4. MySQL 8.0 推荐使用 `binlog_expire_logs_seconds`。
5. MySQL 8.2 / 8.4 已移除 `expire_logs_days`。
6. 主从复制环境清理前必须确认 Replica 已经消费对应 binlog。
7. Binlog 保留时间不仅取决于磁盘，还取决于 PITR 和灾备恢复窗口。
8. 设置 90 天不代表立即删除历史文件；需要时应手工执行一次 `PURGE BINARY LOGS`。

对于传统 MySQL 5.6 / 5.7，90 天配置可以简单理解为：

```ini
[mysqld]
expire_logs_days = 90
```

对于新版本 MySQL：

```ini
[mysqld]
binlog_expire_logs_seconds = 7776000
```

这两种配置表达的核心目标相同：

> 让 MySQL 自动维持一个可控的 Binlog 恢复窗口，避免日志无限增长导致磁盘耗尽。
