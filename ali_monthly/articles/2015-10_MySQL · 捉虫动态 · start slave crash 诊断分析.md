# MySQL · 捉虫动态 · start slave crash 诊断分析

**Date:** 2015/10
**Source:** http://mysql.taobao.org/monthly/2015/10/05/
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

 ## MySQL · 捉虫动态 · start slave crash 诊断分析 
 Author: guyue.zql 

 ## 问题现象
研发同学执行下列语句序列

`stop slave; set global slave_parallel_workers=0; start slave;
`
后程序 hang 住，不一会返回了2013 错误，即服务器连接异常中断，检查 mysqld error log, 发现在mysqld在将并行复制转化为串行复制的过程中异常 crash，其中错误信息为：

`[ERROR] Error reading slave worker configuration
[ERROR] Error creating relay log info: Failed to initialize the worker info structure.
`

crash 堆栈为：

`#0 0x000000395fc0c69c in pthread_kill () from /lib64/libpthread.so.0
#1 0x00000000006b2425 in handle_fatal_signal (sig=11)
#2 <signal handler called>
#3 remove_info (this=0x2b19b775b000)
#4 Relay_log_info::mts_finalize_recovery
#5 0x0000000000920ec3 in slave_start_workers
#6 0x0000000000921c03 in handle_slave_sql
#7 0x000000395fc07851 in start_thread () from /lib64/libpthread.so.0
#8 0x000000395f4e767d in clone () from /lib64/libc.so.6
`

结合源码仔细分析上面的错误信息不难发现，在启动 SQL 线程初始化 worker 信息的过程中失败，从而引起了空指针异常。

## 问题复现
一个简单的 start slave 就能够导致 mysqld crash？应该不会那么简单，编写 testcase，试图在本地重现以上问题，测例如下：

`--source include/master-slave.inc
--source include/not_embedded.inc
--source include/not_windows.inc
--connection slave
stop slave;
set global slave_parallel_workers= 1;
start slave;

stop slave;
set global slave_parallel_workers= 0;
start slave;

--source include/rpl_end.inc
`
运行后果然没有能够重现问题，根据研发的同学反映在中断之前有一段时间命令是 hang 住的，于是断定 hang 问题与 crash 应该存在着联系，以上测例并不能将问题重现，于是与求助研发同学进行联调，在 hang 住的时候采集到了以下关键 pt-pmp 信息：

`pthread_cond_wait,os_cond_wait(os0sync.cc:214),os_event_wait_low(os0sync.cc:610),lock_wait_suspend_thread(lock0wait.cc:323),row_mysql_handle_errors(row0mysql.cc:1040),row_search_for_mysql(row0sel.cc:5064),ha_innobase::index_read(ha_innodb.cc:7668),ha_innobase::index_first(ha_innodb.cc:8041),ha_innobase::rnd_next(ha_innodb.cc:8138),handler::ha_rnd_next(handler.cc:2812),Rpl_info_table_access::scan_info(rpl_info_table_access.cc:287),Rpl_info_table::do_check_info(rpl_info_table.cc:444),Rpl_info_factory::decide_repository(rpl_info_factory.cc:567),Rpl_info_factory::create_worker(rpl_info_factory.cc:419),Relay_log_info::mts_finalize_recovery(rpl_rli.cc:328),slave_start_workers(rpl_slave.cc:6294),handle_slave_sql(rpl_slave.cc:6523),pfs_spawn_thread(pfs.cc:1858),start_thread(libpthread.so.0),clone(libc.so.6)

pthread_cond_wait,safe_cond_wait(thr_mutex.c:240),inline_mysql_cond_wait(mysql_thread.h:1151),start_slave_thread(rpl_slave.cc:1502),start_slave_threads(rpl_slave.cc:1587),start_slave(rpl_slave.cc:9485),start_slave(rpl_slave.cc:548),start_slave_cmd(rpl_slave.cc:690),mysql_execute_command(sql_parse.cc:3576),mysql_parse(sql_parse.cc:6958),dispatch_command(sql_parse.cc:1583),do_command(sql_parse.cc:1102),do_handle_one_connection(sql_connect.cc:1006),handle_one_connection(sql_connect.cc:922),pfs_spawn_thread(pfs.cc:1858),start_thread(libpthread.so.0),clone(libc.so.6)
`

可以看到在 start slave 后，sql thread 读取 slave_worker_info 初始化 worker 的过程中遇到了锁等待问题，于是断定 relay_log_info_repository=’table’，并且有其它的线程对 slave_worker_info中的记录上了Ｘ锁，但是是哪个线程确不是很清楚，因为只看到了一个线程在访问 slave_worker_info 表，然后再一次在hang的时候执行了下列操作：

`mysql> SELECT r.trx_id waiting_trx_id, r.trx_mysql_thread_id waiting_thread, left(r.trx_query,20) waiting_query, concat(concat(lw.lock_type, ' '), lw.lock_mode) waiting_for_lock, b.trx_id blocking_trx_id, b.trx_mysql_thread_id blocking_thread, left(b.trx_query,20) blocking_query, concat(concat(lb.lock_type, ' '), lb.lock_mode) blocking_lock FROM information_schema.innodb_lock_waits w INNER JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id INNER JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id INNER JOIN information_schema.innodb_locks lw ON lw.lock_trx_id = r.trx_id INNER JOIN information_schema.innodb_locks lb ON lb.lock_trx_id = b.trx_id\G
*************************** 1. row ***************************
 waiting_trx_id: 1313
 waiting_thread: 12
 waiting_query: NULL
waiting_for_lock: RECORD S
 blocking_trx_id: 1312
 blocking_thread: 6
 blocking_query: start slave
 blocking_lock: RECORD X
1 row in set (0.00 sec)

mysql> show processlist;
+----+-------------+-----------------+-------+---------+------+-----------------------------------------+------------------+
| Id | User | Host | db | Command | Time | State | Info |
+----+-------------+-----------------+-------+---------+------+-----------------------------------------+------------------+
| 2 | root | localhost:57292 | test | Sleep | 16 | | |
| 3 | root | localhost:57293 | test | Sleep | 16 | | |
| 6 | root | localhost:57299 | test | Query | 15 | Waiting for slave thread to start | start slave |
| 7 | root | localhost:57300 | test | Sleep | 16 | | |
| 11 | system user | | NULL | Connect | 15 | Connecting to master | |
| 12 | system user | | mysql | Connect | 15 | Waiting for the next event in relay log | |
| 13 | root | localhost | test | Query | 0 | init | show processlist |
+----+-------------+-----------------+-------+---------+------+-----------------------------------------+------------------+
7 rows in set (0.00 sec)
`

其中 information_schema.innodb_lock_waits，information_schema.innodb_locks，information_schema.innodb_trx 的信息可参考[官方文档](http://dev.mysql.com/doc/refman/5.6/en/innodb-i_s-tables.html)。

从以上信息可以清楚地看到 start slave（blocking_thread: 6）获取了 slave_worker_info 中行记录的Ｘ锁，而 sql thread 线程 (waiting_thread: 12) 在获取 S 锁失败引起锁等待，进而获取信息失败返回NULL, 进而造成了空指针引用，mysqld crash，此时 start slave 与 sql thread 的资源竟争如下：

* start slave thread 获取 slave_work_info 记录的 X 锁，等待 sql thread 已启动的信号量，然后继续运行；
* sql thread 被启动后，需要根据 slave_worker_info 为进行初始化，因此等待获取相关行的Ｓ锁;

因此，start slave thread 与 sql thread 形成了死锁，分析到了现在还有两个疑问没有解释：

* start slave 为什么会访问 slave_worker_info 且一直持有锁资源?
* 为什么本地在测试的时候加上 relay_log_info_repository 的设置还是不能重现呢？

继续研究，打开general log 后，发现有许多的 set autocommit=0 的语句，于是感觉可能是 autocommit 的问题，修改后测例如下：

`--source include/master-slave.inc
--source include/not_embedded.inc
--source include/not_windows.inc
--connection slave
stop slave;
set global relay_log_info_repository='table';
set global master_info_repository='table';
set global slave_parallel_workers= 1;
start slave;
--sleep 1

stop slave;
set autocommit=0;
set global slave_parallel_workers= 0;
start slave;

--source include/rpl_end.inc
`

运行后问题终于重现。。。

## 问题原因
有了以上的分析过程，不难看出，在 1) autocommit=0 && 2) relay_log_info_repository=’table’ && 3) 并行复制切换到串行复制时，就会触发此 bug, 原因有以下两个：

* start slave thread 在等待 sql thread 的过程中没有释放其占用的锁资源；
* sql thread 在初始化 worker 信息时没有处理获取信息失败的情况。

回到上面的问题：

* start slave 为什么会访问 slave_worker_info 且一直持有锁资源?

 答：start slave 的过程中需要初始化 worker信息，执行 crash recovery 等一系列操作，对于已经执行的不再执行，对于没有执行的继续执行，详情见`global_init_info`，`mts_recovery_groups` 等函数，在autocommit=0 的情况下，start slave thread 没有commit 或者执行完之前是不会释放锁资源的。
* 为什么本地在测试的时候加上 relay_log_info_repository 的设置还是不能重现呢？

 答：本地在测试的时候，autocommit 默认值为1，因此不会重现。
在上锁方面为什么会上写锁，可以参考：`Rpl_info_table::do_init_info` 函数；
* 影响范围
经官方验证，5.6, 5.6.26, 5.6.28, 5.7.10 均存在此 bug。

## 问题扩展
在排查问题的过程中发现 start slave 时会自动提交当前 session 之前已经操作的数据，然后开启一个新的事务，所以在使用 start slave 时要注意这个问题！

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)