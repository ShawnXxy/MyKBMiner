# MySQL · 答疑解惑 · 索引过滤性太差引起CPU飙高分析

**Date:** 2015/10
**Source:** http://mysql.taobao.org/monthly/2015/10/03/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 10
 ](/monthly/2015/10)

 * 当期文章

 MySQL · 引擎特性 · InnoDB 全文索引简介
* MySQL · 特性分析 · 跟踪Metadata lock
* MySQL · 答疑解惑 · 索引过滤性太差引起CPU飙高分析
* PgSQL · 特性分析 · PG主备流复制机制
* MySQL · 捉虫动态 · start slave crash 诊断分析
* MySQL · 捉虫动态 · 删除索引导致表无法打开
* PgSQL · 特性分析 · PostgreSQL Aurora方案与DEMO
* TokuDB · 捉虫动态 · CREATE DATABASE 导致crash问题
* PgSQL · 特性分析 · pg_receivexlog工具解析
* MySQL · 特性分析 · MySQL权限存储与管理

 ## MySQL · 答疑解惑 · 索引过滤性太差引起CPU飙高分析 
 Author: 维度 

 ## 前言

在操作数据库系统的时候，有个常识就是在建表的时候一定要建索引。为什么要建索引呢？

这里以MySQL的InnoDB存储引擎为例，因为InnoDB会以索引的排序为基准建立B+树，这样在检索数据的时候就可以通过B+树来查找，查找算法的时间复杂度是O(logn)级别的，避免全表扫描带来的性能下降和额外资源损耗。

理论上一个表所有的字段都可以建索引，那么给哪些字段建索引效果好呢？

一个想法是给频繁在SQL的where条件中出现的字段建立索引，这样可以保证通过索引来查找数据。

有一点是经常被忽略的，那就是索引的过滤性。比如我们给一个整型字段加索引，而这个字段在几乎所有的记录上的值都是1（过滤性很差），那么我们通过这个索引来查找数据就会遍历大部分无关记录，造成浪费。

我们知道update语句也是通过索引来查找待更新的数据的，而且update会给索引查找的记录加上X锁，因此索引过滤性不好不但造成性能下降，还有可能造成锁争夺和锁等待的损耗。

下面给出一个具体的因为索引过滤性太差引起CPU飙高的case，在RDS的线上实例曾出现过类似的case。

## 场景构造

在MySQL里我们建立这样一个表：

`CREATE TABLE `sbtest1` (
 `id` int(10) unsigned NOT NULL,
 `k` int(10) unsigned NOT NULL DEFAULT '0',
 `n` int(10) unsigned NOT NULL DEFAULT '0',
 `c` char(120) NOT NULL DEFAULT '',
 `pad` char(60) NOT NULL DEFAULT '',
 PRIMARY KEY (`id`),
 KEY `k_1` (`k`)
) ENGINE=InnoDB;
`

然后我们给sbtest1加点数据，并且让索引k_1(k)的过滤性不好，表内一共10000000条数据，索引k只有2个值50,51，如下所示：

`mysql> select count(*) from sbtest1;
+----------+
| count(*) |
+----------+
| 10000000 |
+----------+
1 row in set (1.80 sec)

mysql> select distinct k from sbtest1;
+----+
| k |
+----+
| 50 |
| 51 |
+----+
2 rows in set (2.22 sec)
`

然后我们用sysbench开32个并发的update，update语句如下：

`UPDATE sbtest1 SET c='随机字符串' WHERE k=50或51 and n=随机值
`

执行show full processlist\G，可以看到这些update的状态大多处于”Searching rows for update”的状态。

`mysql> show full processlist\G
*************************** 1. row ***************************
 Id: 2
 User: root
 Host:
 db: test
 Command: Sleep
 Time: 6
 State:
 Info: NULL
 Memory_used: 1146520
Memory_used_by_query: 8208
 Logical_read: 53
 Physical_sync_read: 2
 Physical_async_read: 0
Temp_user_table_size: 0
Temp_sort_table_size: 0
 Temp_sort_file_size: 0
*************************** 2. row ***************************
 Id: 6
 User: root
 Host:
 db: sbtest
 Command: Query
 Time: 21
 State: Searching rows for update
 Info: UPDATE sbtest1 SET c='96372750646-31206582030-89561475094-70112992370-09982266420-13264143120-70453817624-14068123856-50060327807-36562985632' WHERE k=50 and n=4951641
 Memory_used: 119840
Memory_used_by_query: 232
 Logical_read: 4935
 Physical_sync_read: 0
 Physical_async_read: 0
Temp_user_table_size: 0
Temp_sort_table_size: 0
 Temp_sort_file_size: 0
*************************** 3. row ***************************
 Id: 7
 User: root
 Host:
 db: sbtest
 Command: Query
 Time: 21
 State: Searching rows for update
 Info: UPDATE sbtest1 SET c='28921237680-50951214786-47793625883-44090170070-31354117142-11520543175-97262835853-83486109785-32721666363-10671483869' WHERE k=51 and n=5033717
 Memory_used: 119840
Memory_used_by_query: 232
 Logical_read: 4949
 Physical_sync_read: 5
 Physical_async_read: 0
Temp_user_table_size: 0
Temp_sort_table_size: 0
 Temp_sort_file_size: 0

...
`

“Searching rows for update”即MySQL正在寻找待更新的记录的状态，正常情况这个状态是非常快就结束的，但是这里却长时间处于这个状态，为什么呢？

由于表的索引过滤性太差，每个线程在查找的时候会遇到很多冲突的记录。

InnoDB在通过索引拿到记录后，会给这些记录上X锁，同时也会请求全局的`lock_sys->mutex`和`trx_sys->mutex`，所以这里我们判断每个线程都堵在锁等待这里。（ps: 关于InnoDB加锁的逻辑，可以查看[这篇博文](http://hedengcheng.com/?p=771)）

这时候对系统用一下top命令，可以发现这个MySQL实例CPU飚的很高，我们再用perf工具看一下CPU飙高的MySQL调用堆栈是怎么样的，如下所示：

` 83.77% mysqld mysqld [.] _Z8ut_delaym
 |
 --- _Z8ut_delaym
 |
 |--99.99%-- _Z15mutex_spin_waitP10ib_mutex_tPKcm
 | |
 | |--88.88%-- _ZL20pfs_mutex_enter_funcP10ib_mutex_tPKcm.constprop.68
 | | |
 | | |--54.05%-- _ZL29lock_rec_convert_impl_to_explPK11buf_block_tPKhP12dict_index_tPKm
 | | | _Z34lock_clust_rec_read_check_and_lockmPK11buf_block_tPKhP12dict_index_tPKm9lock_modemP9que_thr_t
 | | | _ZL16sel_set_rec_lockPK11buf_block_tPKhP12dict_index_tPKmmmP9que_thr_t
 | | | _Z20row_search_for_mysqlPhmP14row_prebuilt_tmm
 | | | _ZN11ha_innobase10index_nextEPh
 | | | _ZN7handler13ha_index_nextEPh
 | | | _ZL8rr_indexP11READ_RECORD
 | | | _Z12mysql_updateP3THDP10TABLE_LISTR4ListI4ItemES6_PS4_jP8st_ordery15enum_duplicatesbPySB_
 | | | _Z21mysql_execute_commandP3THD
 | | | _Z11mysql_parseP3THDPcjP12Parser_state
 | | | _Z16dispatch_command19enum_server_commandP3THDPcj
 | | | _Z26threadpool_process_requestP3THD
 | | | _ZL11worker_mainPv
 | | | start_thread
 | | |
 | | --45.95%-- _Z15lock_rec_unlockP5trx_tPK11buf_block_tPKh9lock_mode
 | | _Z20row_unlock_for_mysqlP14row_prebuilt_tm
 | | _Z12mysql_updateP3THDP10TABLE_LISTR4ListI4ItemES6_PS4_jP8st_ordery15enum_duplicatesbPySB_
 | | _Z21mysql_execute_commandP3THD
 | | _Z11mysql_parseP3THDPcjP12Parser_state
 | | _Z16dispatch_command19enum_server_commandP3THDPcj
 | | _Z26threadpool_process_requestP3THD
 | | _ZL11worker_mainPv
 | | start_thread

`

我们看到耗CPU最高的调用函数栈是…`mutex_spin_wait`->`ut_delay`，属于锁等待的逻辑。InnoDB在这里用的是自旋锁，锁等待是通过调用ut_delay做空循环实现的，会消耗CPU。这里证明了上面的判断是对的。

在这个case里涉及到的锁有记录锁、`lock_sys->mutex`和`trx_sys->mutex`，究竟是哪个锁等待时间最长呢？我们可以用下面的方法确认一下：

`mysql> SELECT COUNT_STAR, SUM_TIMER_WAIT, AVG_TIMER_WAIT, EVENT_NAME FROM performance_schema.events_waits_summary_global_by_event_name where COUNT_STAR > 0 and EVENT_NAME like 'wait/synch/%' order by SUM_TIMER_WAIT desc limit 10;
+------------+------------------+----------------+--------------------------------------------+
| COUNT_STAR | SUM_TIMER_WAIT | AVG_TIMER_WAIT | EVENT_NAME |
+------------+------------------+----------------+--------------------------------------------+
| 36847781 | 1052968694795446 | 28575867 | wait/synch/mutex/innodb/lock_mutex |
| 8096 | 81663413514785 | 10086883818 | wait/synch/cond/threadpool/timer_cond |
| 19 | 3219754571347 | 169460766775 | wait/synch/cond/threadpool/worker_cond |
| 12318491 | 1928008466219 | 156446 | wait/synch/mutex/innodb/trx_sys_mutex |
| 36481800 | 1294486175099 | 35397 | wait/synch/mutex/innodb/trx_mutex |
| 14792965 | 459532479943 | 31027 | wait/synch/mutex/innodb/os_mutex |
| 2457971 | 62564589052 | 25346 | wait/synch/mutex/innodb/mutex_list_mutex |
| 2457939 | 62188866940 | 24909 | wait/synch/mutex/innodb/rw_lock_list_mutex |
| 201370 | 32882813144 | 163001 | wait/synch/rwlock/innodb/hash_table_locks |
| 1555 | 15321632528 | 9853039 | wait/synch/mutex/innodb/dict_sys_mutex |
+------------+------------------+----------------+--------------------------------------------+
10 rows in set (0.01 sec)
`

从上面的表可以确认，lock_mutex（在MySQL源码里对应的是`lock_sys->mutex`）的锁等待累积时间最长（SUM_TIMER_WAIT）。lock_sys表示全局的InnoDB锁系统，在源码里看到InnoDB加/解某个记录锁的时候（这个case里是X锁），同时需要维护lock_sys，这时会请求lock_sys->mutex。

在这个case里，因为在Searching rows for update的阶段频繁地加/解X锁，就会频繁请求`lock_sys->mutex`，导致`lock_sys->mutex`锁总等待时间过长，同时在等待的时候消耗了大量CPU。

当我们将索引改成过滤性好的（比如字段n），再做上述实验，就看不到那么多线程堵在”Searching rows for update”的阶段，而且实例的CPU消耗也降了很多。

## 结语

通过以上实验，我们看到索引过滤性不好可能带来灾难性的结果：语句hang住以及主机CPU耗尽。因此我们在设计表的时候，应该对业务上的数据有充分的估计，选择过滤性好的字段作为索引。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)