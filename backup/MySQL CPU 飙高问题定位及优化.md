MySQL CPU 飙高是数据库性能故障中最常见、最棘手的故障之一，通常是由**执行效率极低的 SQL 语句**或**不合理的并发锁竞争**导致的。

排查核心思路是：**定位是哪个进程/线程消耗了 CPU，然后找出该线程正在执行的低效 SQL。**

以下是线上环境排查 MySQL CPU 飙高问题的系统化流程：

---

### 🚀 第一阶段：OS/系统层定位（确定 CPU 消耗源）

首先要确定 CPU 升高是否确实由 `mysqld` 进程引起。

1. **使用 `top` 或 `htop`：**
   **Bash**
   
   ```
   top
   ```
   
   * 观察 **`mysqld`** 进程的 `%CPU` 占用率。
2. 定位具体线程（Threads）：
   如果 mysqld 进程很高，需要进一步定位是哪个内部线程在消耗资源。
   **Bash**
   
   ```
   # 使用 top -H -p [mysqld_pid] 或 pidstat -t -p [mysqld_pid] 1
   # 查找 mysqld 进程中，哪个 LWP/TID (线程ID) 占用CPU最高。
   ```
3. 转换线程 ID：
   将高 CPU 占用的 OS 线程 ID (TID) 转换为 MySQL 内部的 PROCESSLIST\_ID 或 THREAD\_ID，以便在 MySQL 内部追踪。
   
   * **MySQL 5.7+：** 使用 `performance_schema.threads` 表进行关联。
     **SQL**
     
     ```
     SELECT *
     FROM performance_schema.threads
     WHERE THREAD_OS_ID = [OS 线程ID];
     ```

---

### 📝 第二阶段：MySQL 内部定位（找出元凶 SQL）

一旦有了 MySQL 内部的进程/线程 ID，就可以开始定位导致 CPU 升高的 SQL 语句。

#### 1. 实时查看当前活跃进程

这是最常用的命令，可以快速看到所有正在执行的 SQL 语句、执行时间和状态。

**SQL**

```
SHOW FULL PROCESSLIST;
```

**分析要点：**

* **`Id`：** 对应于上面找到的 `PROCESSLIST_ID`。
* **`Command`：** 如果是 `Query`，说明正在执行查询；如果是 `Sleep`，说明连接空闲。
* **`Time`：** 执行时间（秒）。**找出 `Time` 值大且 `Command` 为 `Query` 的语句。**
* **`State`：** 线程状态。关注如下状态：
  * **`Sending data`：** 并非一定是发送数据，可能是在进行全表/全索引扫描、大量数据处理。这是 CPU 消耗的常见状态。
  * **`Sorting result` / `Copying to tmp table`：** 通常是由于 `ORDER BY` 或 `GROUP BY` 操作没有使用索引，需要在内存或磁盘创建临时表进行排序或聚合，极度消耗 CPU。

#### 2. 分析慢查询日志（事后追溯）

如果实时查看未能捕捉到问题（因为低效查询可能执行非常快但次数极多），应该分析慢查询日志。

* **确保开启：** 确认 `slow_query_log` 为 `ON`，并设置合理的 `long_query_time`（线上环境通常设为 1 秒或更低）。
* **分析日志：** 使用 `mysqldumpslow` 或 Percona Toolkit 的 `pt-query-digest` 工具对慢查询日志文件进行分析，找出：
  * `Query time` (执行总时间) 最长的 SQL。
  * `Rows examined` (扫描行数) 最多，但 `Rows sent` (返回行数) 很少的 SQL。**扫描行数多意味着进行了大量计算和逻辑 I/O，这是 CPU 飙高的主要原因。**

#### 3. 利用 Performance Schema (MySQL 5.6+ 进阶)

Performance Schema 提供了更详细的性能指标。

**SQL**

```
-- 找出最耗时、最耗CPU的 SQL 语句 (基于语句指纹)
SELECT
    schema_name,
    digest_text,
    count_star,
    sum_timer_wait
FROM
    performance_schema.events_statements_summary_by_digest
ORDER BY
    sum_timer_wait DESC
LIMIT 10;
```

---

### 🔨 第三阶段：优化与解决

定位到“元凶 SQL”后，使用 `EXPLAIN` 命令分析其执行计划，然后进行优化。

**SQL**

```
EXPLAIN [元凶SQL语句];
```

#### 常见优化方向：

1. **索引缺失：**
   * **问题：**`EXPLAIN` 结果中 `type` 为 `ALL`（全表扫描）。
   * **解决：** 根据 `WHERE`、`ORDER BY` 或 `GROUP BY` 的字段添加或优化联合索引，避免全表扫描。
2. **函数操作：**
   * **问题：** 在索引列上使用了函数（如 `YEAR(date_col) = 2024`）或隐式类型转换，导致索引失效。
   * **解决：** 避免在索引列上使用函数，改写为范围查询（如 `date_col BETWEEN '2024-01-01' AND '2024-12-31'`）。
3. **不当的 `ORDER BY`/`GROUP BY`：**
   * **问题：**`Extra` 字段显示 `Using filesort` 或 `Using temporary`。
   * **解决：** 建立覆盖索引，让排序或分组可以直接在索引上完成，避免创建临时表。
4. **连接风暴：**
   * **现象：**`SHOW STATUS LIKE 'Threads_connected'` 很高，`Threads_running` 偶尔高。
   * **解决：** 检查应用程序连接池配置，确保连接被正确释放；限制 `max_connections` 以保护数据库。
5. **DDL 操作：**
   * **问题：** 正在执行 `ALTER TABLE` 等 DDL 操作，这些操作通常会加表锁，阻塞其他查询并可能导致 CPU 短时飙高。
   * **解决：** 尽量在业务低峰期或使用在线 DDL 工具（如 Percona 的 `pt-online-schema-change`）。
