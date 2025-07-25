# MySQL · 答疑解惑 · binlog 位点刷新策略

**Date:** 2015/05
**Source:** http://mysql.taobao.org/monthly/2015/05/10/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 05
 ](/monthly/2015/05)

 * 当期文章

 MySQL · 引擎特性 · InnoDB redo log漫游
* MySQL · 专家投稿 · MySQL数据库SYS CPU高的可能性分析
* MySQL · 捉虫动态 · 5.6 与 5.5 InnoDB 不兼容导致 crash
* MySQL · 答疑解惑 · InnoDB 预读 VS Oracle 多块读
* PgSQL · 社区动态 · 9.5 新功能BRIN索引
* MySQL · 捉虫动态 · MySQL DDL BUG
* MySQL · 答疑解惑 · set names 都做了什么
* MySQL · 捉虫动态 · 临时表操作导致主备不一致
* TokuDB · 引擎特性 · zstd压缩算法
* MySQL · 答疑解惑 · binlog 位点刷新策略

 ## MySQL · 答疑解惑 · binlog 位点刷新策略 
 Author: 沽月 

 ## 背景
MySQL 非 GTID 协议主备同步原理：
主库在执行 SQL 语句时产生binlog，在事务 commit 时将产生的binlog event写入binlog文件，备库IO线程通过 `com_binlog_dump` 用文件位置协议从主库拉取 binlog，将拉取的binlog存储到relaylog, SQL线程读取 relaylog 然后进行 apply，实现主备同步，在这个过程中有以下几个问题：

1. 主库什么时间将产生的 binlog 真正刷到文件中？
2. 备库IO线程从哪个位置读取主库的 binlog event 的？
3. 备库SQL线程如何记录执行到的 relaylog 的位点？
4. 备库IO线程何时将cache中的event 刷到relay log 文件中的?

## 问题分析

下面对这几个问题挨个解答。

**问题 1: 主库什么时间将产生的binlog 真正刷到文件中**

事务`ordered_commit` 中，会将 `thd->cache_mngr` 中的 binlog cache 写入到 binlog 文件中，但并没有执行fsync()操作，即只将文件内容写入到 OS 缓存中，详细 bt 为：

`#0 my_write
#1 0x0000000000a92f50 in inline_mysql_file_write
#2 0x0000000000a9612e in my_b_flush_io_cache
#3 0x0000000000a43466 in MYSQL_BIN_LOG::flush_cache_to_file
#4 0x0000000000a43a4d in MYSQL_BIN_LOG::ordered_commit
#5 0x0000000000a429f2 in MYSQL_BIN_LOG::commit
#6 0x000000000063d3e2 in ha_commit_trans
#7 0x00000000008adb7a in trans_commit_stmt
#8 0x00000000007e511f in mysql_execute_command
#9 0x00000000007e7e0e in mysql_parse
#10 0x00000000007dae0e in dispatch_command
#11 0x00000000007d9634 in do_command
#12 0x00000000007a046d in do_handle_one_connection
#13 0x000000000079ff75 in handle_one_connection
#14 0x0000003a00a07851 in start_thread ()
#15 0x0000003a006e767d in clone ()
`
commit 时，会判断是否将产生的 binlog flush 到文件中，即执行 fsync操作，详细bt 为：

`#0 MYSQL_BIN_LOG::sync_binlog_file
#1 0x0000000000a43c62 in MYSQL_BIN_LOG::ordered_commit
#2 0x0000000000a429f2 in MYSQL_BIN_LOG::commit
#3 0x000000000063d3e2 in ha_commit_trans
#4 0x00000000008adb7a in trans_commit_stmt
#5 0x00000000007e511f in mysql_execute_command
#6 0x00000000007e7e0e in mysql_parse
#7 0x00000000007dae0e in dispatch_command
#8 0x00000000007d9634 in do_command (thd=0x37a40160)
#9 0x00000000007a046d in do_handle_one_connection
#10 0x000000000079ff75 in handle_one_connection
#11 0x0000003a00a07851 in start_thread ()
#12 0x0000003a006e767d in clone ()
`

由 `MYSQL_BIN_LOG::sync_binlog_file` 可以看出，每提交一个事务，会 fsync 一次binlog file。 当 sync_binlog != 1 的时候，每次事务提交的时候，不一定会执行 fsync 操作，binlog 的内容只是缓存在了 OS（是否会执行fsync操作，取决于OS缓存的大小），此时备库可以读到主库产生的 binlog, 在这种情况下，当主库机器挂掉时，有以下两种情况：

1. 主备同步无延迟，此时主库机器恢复后，备库接着之前的位点重新拉binlog, 但是主库由于没有fsync最后的binlog，所以会返回1236 的错误：
`MySQL error code 1236 (ER_MASTER_FATAL_ERROR_READING_BINLOG): Got fatal error %d from master when reading data from binary log: '%-.256s'`
2. 备库没有读到主库失去的binlog，此时备库无法同步主库最后的更新，备库不可用。

**问题 2: 备库IO线程从哪个位置读取主库的binlog event 的**

更新位点信息的 bt 如下：

`#0 Rpl_info_table::do_flush_info (this=0x379cbf90, force=false)
#1 0x0000000000a78270 in Rpl_info_handler::flush_info
#2 0x0000000000a773b9 in Master_info::flush_info
#3 0x0000000000a5da4b in flush_master_info
#4 0x0000000000a697eb in handle_slave_io
#5 0x0000003a00a07851 in start_thread () from /lib64/libpthread.so.0
#6 0x0000003a006e767d in clone () from /lib64/libc.so.6
`

备库通过 `master_log_info` 来记录主库的相关信息，通过参数 `sync_master_info` 来设置备库经过多少个 binlog event 来更新已经读取到的位点信息。当stop slave时，会把正常的位点更新到`master_log_info`中，此时，如果最后的位点不是commit，则在start slave后，会继续上一位点拉取 binlog，从而造成同一个事务的binlog event分布在不同的binlog file中，此时如果执行顺利则不会有问题；如果在拉这个事务的过程中，sql 线程出错中断，在并行复制下会引起分发线程停在事务中间，再次启动的时候，会从上一次分发的事务继续分发，会造成在并行复制中不可分发的情况，因此需要注意。

当 sync_master_info > 1000时，可能在第1000个binlog 拉取的时候机器出问题，此时重启后会从主库多拉999个 binlog event，造成事务在备库多次执行问题，对于没有 primary key, unique key 可能会有问题，造成主备数据不一致，最常遇到的是1062问题。

**问题3: 备库SQL线程如何记录执行到的relaylog 的位点**

同问题2一样，相关的 bt 也类似，`relay_log_info` 记录的是备库已经执行了的最后的位点，这个位点不会处于事务中间，即是每 `sync_relay_log_info` 个事务更新一下这个位点。

相关[bugs](http://bugs.mysql.com/bug.php?id=72794)
bug 原因: 备库异常 crash 后，可能造成事务在拉取过程中被重新拉取，binlog序列如下：

`begin;
table_map;
begin;
table_map;
rows_log_event;
commit;
`

在并行复制条件下，由于出现了不完整的事务，所以会造成绑定事务信息无法恢复，造成hang的情况，详情见 [bug 分析](http://bugs.mysql.com/bug.php?id=72794)。

**问题 4: 备库IO线程何时将cache中的event 刷到relay log 文件中的**

这个问题的解答和问题1类似，也是以binlog event为单位的，当然也存在着和问题1中同样的问题，在此不在赘述。

## 结语
MySQL 通过 `sync_binlog`，`sync_master_info`，`sync_relay_log_info`，`sync_relay_log` 来记录相关的位点信息，出于性能考虑以及程序本身的健壮性，引入了各式要样的bug，类似的bug在此不在列举，那么有没有更好的方法来记录这些信息呢，当然有，即GTID 协议，会在下期月报分析。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)