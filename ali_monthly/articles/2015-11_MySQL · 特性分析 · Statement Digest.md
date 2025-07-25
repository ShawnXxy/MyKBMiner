# MySQL · 特性分析 · Statement Digest

**Date:** 2015/11
**Source:** http://mysql.taobao.org/monthly/2015/11/02/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 11
 ](/monthly/2015/11)

 * 当期文章

 MySQL · 社区见闻 · OOW 2015 总结 MySQL 篇
* MySQL · 特性分析 · Statement Digest
* PgSQL · 答疑解惑 · PostgreSQL 用户组权限管理
* MySQL · 特性分析 · MDL 实现分析
* PgSQL · 特性分析 · full page write 机制
* MySQL · 捉虫动态 · MySQL 外键异常分析
* MySQL · 答疑解惑 · MySQL 优化器 range 的代价计算
* MySQL · 捉虫动态 · ORDER/GROUP BY 导致 mysqld crash
* MySQL · TokuDB · TokuDB 中的行锁
* MySQL · 捉虫动态 · order by limit 造成优化器选择索引错误

 ## MySQL · 特性分析 · Statement Digest 
 Author: lengxiang 

 ## 背景

在对数据库进行性能调优的时候，除了参数、配置的调整以外，SQL调优也是重要的手段，同时也是收益最大的一环。
当DBA对业务库进行sql调优的时候，如何做到有的放矢，投入产出受益最大？足够详细的SQL性能统计无疑是最重要的信息。

下面我们先来看下不同数据库提供的sql性能统计信息:

## Oracle的sql性能统计

Oracle可以通过直接查询v$表得到，下面的columns列表是我们常用的一些统计:

`select sql_id,
 sql_text,
 sql_fulltext,
 sharable_mem,
 persistent_mem,
 runtime_mem,
 sorts,
 fetches,
 executions,
 parse_calls,
 disk_reads,
 direct_writes,
 buffer_gets,
 application_wait_time,
 concurrency_wait_time,
 cluster_wait_time,
 user_io_wait_time,
 rows_processed,
 cpu_time,
 elapsed_time
from v$sql_area
`

这里边包括了几类统计，SQL内存使用的统计、parse的统计、物理/逻辑IO的统计、cpu时间、等待时间等时间统计。

DBA可以根据这些统计信息进行有针对性的调优:

1. CPU调优: 如果当前数据库性能CPU是瓶颈，可以通过order by cpu_time，查询出来top CPU的SQL进行调优；
2. IO调优: 可以根据buffer_gets, disk_reads，user_io_wait_time 查询top IO的SQL进行调优；
3. 锁争用: 可以根据concurrency_wait_time，cluster_wait_time查询top lock的SQL进行调优。

## MySQL的SQL性能统计

**1. 通过show profiles来查询统计信息**
在MySQL 5.6版本之前，还保留着show profiles的方式，后续版本逐步被performance_schema来替换了。
使用方法如下:

`mysql> SET profiling=1;
mysql> select 1, sleep(1);
mysql> show profile cpu, block io for query 1;
+----------------------+----------+----------+------------+--------------+---------------+
| Status | Duration | CPU_user | CPU_system | Block_ops_in | Block_ops_out |
+----------------------+----------+----------+------------+--------------+---------------+
| starting | 0.000100 | NULL | NULL | NULL | NULL |
| checking permissions | 0.000014 | NULL | NULL | NULL | NULL |
| Opening tables | 0.000024 | NULL | NULL | NULL | NULL |
| init | 0.000020 | NULL | NULL | NULL | NULL |
| optimizing | 0.000008 | NULL | NULL | NULL | NULL |
| executing | 0.000022 | NULL | NULL | NULL | NULL |
| User sleep | 1.000090 | NULL | NULL | NULL | NULL |
| end | 0.000021 | NULL | NULL | NULL | NULL |
| query end | 0.000009 | NULL | NULL | NULL | NULL |
| closing tables | 0.000010 | NULL | NULL | NULL | NULL |
| freeing items | 0.000055 | NULL | NULL | NULL | NULL |
| logging slow query | 0.000008 | NULL | NULL | NULL | NULL |
| cleaning up | 0.000012 | NULL | NULL | NULL | NULL |
+----------------------+----------+----------+------------+--------------+---------------+
13 rows in set (0.00 sec)
`

show profile的语法如下：

` SHOW PROFILE [type [, type] … ] 
 [FOR QUERY n] 
 [LIMIT row_count [OFFSET offset]] 
 
 type: 
 ALL 
 | BLOCK IO 
 | CONTEXT SWITCHES 
 | CPU 
 | IPC 
 | MEMORY 
 | PAGE FAULTS 
 | SOURCE 
 | SWAPS 
`

从结果集可以看到每一块操作的CPU时间，block IO情况。
但这种适合拿单个SQL进行分析，使用上的便捷性比较差。

**2. 通过performance_schema的digest统计**
MySQL之前的版本不支持绑定变量，导致SQL语句太多，相同业务的SQL汇总统计比较麻烦。
从MySQL 5.6开始，在performance_schema中支持了对SQL statement的digest进行统计。
performance_schema.events_statements_summary_by_digest表根据digest进行汇总统计，DBA可以直接访问这个内存表得到SQL的统计信息。

首先，需要打开performance_schema，然后系统就会自动为SQL statement生成digest，并记录统计信息。
例如:

`mysql> select 1, sleep(1);
+---+----------+
| 1 | sleep(1) |
+---+----------+
| 1 | 0 |
+---+----------+
1 row in set (1.00 sec)
mysql> select * from events_statements_summary_by_digest\G;
*************************** 1. row ***************************
 SCHEMA_NAME: performance_schema
 DIGEST: bb80cc862a205b471ce0f0ff2605a9a0
 DIGEST_TEXT: SELECT ? , `sleep` (?)
 COUNT_STAR: 1
 SUM_TIMER_WAIT: 1000577972000
 MIN_TIMER_WAIT: 1000577972000
 AVG_TIMER_WAIT: 1000577972000
 MAX_TIMER_WAIT: 1000577972000
 SUM_LOCK_TIME: 0
 SUM_ERRORS: 0
 SUM_WARNINGS: 0
 SUM_ROWS_AFFECTED: 0
 SUM_ROWS_SENT: 1
 SUM_ROWS_EXAMINED: 0
SUM_CREATED_TMP_DISK_TABLES: 0
 SUM_CREATED_TMP_TABLES: 0
 SUM_SELECT_FULL_JOIN: 0
SUM_SELECT_FULL_RANGE_JOIN: 0
 SUM_SELECT_RANGE: 0
 SUM_SELECT_RANGE_CHECK: 0
 SUM_SELECT_SCAN: 0
 SUM_SORT_MERGE_PASSES: 0
 SUM_SORT_RANGE: 0
 SUM_SORT_ROWS: 0
 SUM_SORT_SCAN: 0
 SUM_NO_INDEX_USED: 0
 SUM_NO_GOOD_INDEX_USED: 0
 FIRST_SEEN: 2015-11-06 11:15:52
 LAST_SEEN: 2015-11-06 11:15:52
2 rows in set (0.00 sec)
`

DBA可用通过这个表的统计信息，来有的放矢的进行SQL调优。

然而performance_schema中digest的限制:

1. 必须打开performance_schema的所有功能才行，但performance_schema全部打开会对性能产生一些影响；
2. events_statements_summary_by_digest 表默认有200个的最大限制，但从MySQL 5.5开始，可以通过调整performance_schema_digests_size来修改。但如果表满了的话，新来的digest的统计信息，会被全部汇总到一个digest=NULL的记录中；
3. 对于每一个SQL statement，performance_schema会生成一个最大1024bytes的digest_text，超过的会被截断。

针对以上的一些限制，MySQL5.7最新GA的版本，进行了一些改进。

另外，针对限制2 Percona有一些工具pt-query-digest, 并建立一些digest历史表，进行分析，有兴趣的可以使用尝试一下。可以参考: [mysql-query-digest-with-performance-schema](https://www.percona.com/blog/2015/10/13/mysql-query-digest-with-performance-schema/)

## MySQL 5.7 performance_schema digest的改进

1. SQL statement digest生成功能不必和performance_schema绑定，digest的功能的源代码主要是这两个文件：PFS_digest.cc和PFS_digest.h。这两个文件从存储引擎目录storage/perfschema/ 移到了server目录sql/下；
2. 从MySQL 5.7.6开始，digest的最大长度由固定的1024bytes，变成了可变大小，由参数performance_schema_max_sql_text_length在系统启动的时候初始化。

## 总结

用户可以针对performance_schema提供的digest的功能，根据需求进行一些开发和扩展，比如定期历史保存、建立SQL性能基线、或者更进一步如果能修改源码，可以为digest增加更多的merics等。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)