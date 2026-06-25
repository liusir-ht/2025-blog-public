# MySQL 连接数查看与管理指南

## 1. 核心查询命令

在 MySQL 命令行或任何 SQL 客户端中执行以下命令，可获取不同维度的连接数信息。

| 查询目的 | SQL 命令 | 说明 |
| :--- | :--- | :--- |
| **当前活跃连接数** | `SHOW STATUS LIKE 'Threads_connected';` | 最直接的命令，返回当前打开的连接数量。 |
| **历史最大连接数** | `SHOW GLOBAL STATUS LIKE 'Max_used_connections';` | 查看自上次启动以来，曾经同时达到过的最大连接数，用于评估连接数上限设置是否合理。 |
| **所有连接的详情** | `SHOW PROCESSLIST;` | 列出当前所有连接的详细信息，包括每个连接的状态、正在执行的SQL等，方便排查问题。 |
| **查看最大连接数限制** | `SHOW VARIABLES LIKE 'max_connections';` | 查看 MySQL 允许的最大连接数（`max_connections` 参数的值）。 |

---

## 2. 补充查询命令

| 用途 | SQL 命令 | 说明 |
| :--- | :--- | :--- |
| **当前连接数（简写）** | `SELECT COUNT(*) FROM information_schema.PROCESSLIST;` | 通过系统表统计当前连接总数，效果同 `Threads_connected`。 |
| **按状态分组统计连接** | `SELECT STATE, COUNT(*) FROM information_schema.PROCESSLIST GROUP BY STATE;` | 按连接状态（如 `Sleep`、`Query`、`Locked`）分组统计，帮助定位阻塞或空闲连接过多的问题。 |
| **查看所有连接及执行时间** | `SHOW FULL PROCESSLIST;` | 显示完整SQL语句（不截断），并包含连接的执行时间等信息。 |
| **查看当前连接数（含用户维度）** | `SELECT USER, COUNT(*) FROM information_schema.PROCESSLIST GROUP BY USER;` | 按用户分组统计连接数，便于排查某个用户的连接是否异常。 |
| **查看当前连接数（含数据库维度）** | `SELECT DB, COUNT(*) FROM information_schema.PROCESSLIST GROUP BY DB;` | 按数据库分组统计连接数，便于排查某个库的连接占用情况。 |

---

## 3. 进阶技巧与解读

### 3.1 关联解读
在查看当前连接数时，可以同时查看**最大连接数限制**，评估是否快达到上限：

```sql
SELECT @@global.max_connections, @@global.Threads_connected;