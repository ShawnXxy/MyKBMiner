# MySQL · 特性分析 · 到底是谁执行了FTWL

**Date:** 2017/08
**Source:** http://mysql.taobao.org/monthly/2017/08/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 08
 ](/monthly/2017/08)

 * 当期文章

 MySQL · 引擎特性 · Group Replication内核解析
* PgSQL · 特性介绍 · 列存元数据扫描介绍
* MySQL · 源码分析 · MySQL replication partial transaction
* MySQL · 特性分析 · 到底是谁执行了FTWL
* MySQL · 源码分析 · mysql认证阶段漫游
* MySQL · 源码分析 · 内存分配机制
* PgSQL · 源码分析 · PG 优化器中的pathkey与索引在排序时的使用
* MSSQL· 实现分析 · Extend Event日志文件的分析方法
* MySQL · 源码分析 · SHUTDOWN过程
* PgSQL · 应用案例 · HDB for PG特性(数据排盘与任意列高效率过滤)

 ## MySQL · 特性分析 · 到底是谁执行了FTWL 
 Author: 勋臣 

 ## 什么是FTWL
FTWRL是FLUSH TABLES WITH READ LOCK的简称(FTWRL)，该命令主要用于保证备份一致性备份。为了达到这个目的，它需要关闭所有表对象，因此这个命令的杀伤性很大，执行命令时容易导致库hang住。如果它在主库执行，则业务无法正常访问；如果在备库，则会导致SQL线程卡住，主备延迟。 FTWRL通过持有以下两把全局的MDL(MetaDataLock)锁：

* 全局读锁(lock_global_read_lock) 会导致所有的更新操作被堵塞
* 全局COMMIT锁(make_global_read_lock_block_commit) 会导致所有的活跃事务无法提交

FLUSH TABLES WITH READ LOCK执行后整个系统会一直处于只读状态，直到显示执行UNLOCK TABLES。这点请切记。

## 如何高效定位FTWL的执行会话
由于FTWL持有的是MDL锁，所以一旦它执行完成，你将无法以定位DML锁的方式来定位它。即在show processlist的结果和information_schema相关的表中找不到任何相关的线索。我们来看下面的一个例子：

`[test]> flush tables with read lock;
Query OK, 0 rows affected (0.06 sec)

[test]> show full processlist\G
*************************** 1. row ***************************
 Id: 10
 User: root
 Host: localhost
 db: test
 Command: Query
 Time: 0
 State: init
 Info: show full processlist
Progress: 0.000
*************************** 2. row ***************************
 Id: 11
 User: root
 Host: localhost
 db: test
 Command: Query
 Time: 743
 State: Waiting for global read lock
 Info: delete from t0
Progress: 0.000
2 rows in set (0.00 sec)

[test]> select * from information_schema.processlist\G
*************************** 1. row ***************************
 ID: 11
 USER: root
 HOST: localhost
 DB: test
 COMMAND: Query
 TIME: 954
 STATE: Waiting for global read lock
 INFO: delete from t0
 TIME_MS: 954627.587
 STAGE: 0
 MAX_STAGE: 0
 PROGRESS: 0.000
 MEMORY_USED: 67464
EXAMINED_ROWS: 0
 QUERY_ID: 1457
 INFO_BINARY: delete from t0
 TID: 8838
*************************** 2. row ***************************
 ID: 10
 USER: root
 HOST: localhost
 DB: test
 COMMAND: Query
 TIME: 0
 STATE: Filling schema table
 INFO: select * from information_schema.processlist
 TIME_MS: 0.805
 STAGE: 0
 MAX_STAGE: 0
 PROGRESS: 0.000
 MEMORY_USED: 84576
EXAMINED_ROWS: 0
 QUERY_ID: 1461
 INFO_BINARY: select * from information_schema.processlist
 TID: 8424
2 rows in set (0.02 sec)
`
从上的输出中，我们只发现了会话11 在等候一个全局读锁。但这个锁被谁持有，从这个输出里面我们找不到任何线索。我现在再来看看INNODB STATUS输出：

`...
------------
TRANSACTIONS
------------
Trx id counter 20439
Purge done for trx's n:o < 20422 undo n:o < 0 state: running but idle
History list length 176
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 0, not started
MySQL thread id 11, OS thread handle 0x7f7f5cdb8b00, query id 1457 localhost root Waiting for global read lock
delete from t0
---TRANSACTION 0, not started
MySQL thread id 10, OS thread handle 0x7f7f5ce02b00, query id 1462 localhost root init
show engine innodb status
--------
...
`
我们从引擎层也没有找到相关的线索。这个毫无疑问，在本文开始的时候就已经指出了FTWL持有的事MDL锁。
当然因为这个例子中只有两个会话，你一眼就可以看出来谁持有了全局读锁。如果是线上的环境，将会有成百上千个会话。那又怎么办呢？请继续往下看。那我们如何快速定位FTWL的锁呢？主要有下面三种方法：

* 如果你用的Mysql 5.7，那么你可以使用performance_schema.metadata_locks
* 如果你用的Mysql 5.6，那么你可以使用performance_schema.events_statements_history
* 如果你用的Mysql版本比较老，那么可以使用genearal log或者一些sql审计的日志来定位

以上三种方法都是要开启的，默认情况这些方法是没有开启的。所以在工作中，我们会经常遇到这种情况。
整个库都被堵住了。数据库里出现了大量的Waiting for global read lock等待。但上面提到的三种方法又不适用于我们。所以接下来我会为大家用展示一种利用gdb去快速定位执行FTWL的会话。我们来看下面的例子：

`会话1：

flush tables with read lock;
Query OK, 0 rows affected (0.00 sec)

会话2：
mysql> delete from t; --被hang住

会话3：
mysql> show processlist;
+----+------+-----------+------+---------+------+------------------------------+------------------+
| Id | User | Host | db | Command | Time | State | Info |
+----+------+-----------+------+---------+------+------------------------------+------------------+
| 7 | root | localhost | test | Query | 227 | Waiting for global read lock | delete from t |
| 8 | root | localhost | NULL | Sleep | 215 | | NULL |
| 9 | root | localhost | NULL | Query | 0 | init | show processlist |
+----+------+-----------+------+---------+------+------------------------------+------------------+
`

由于会话1执行了FTWL,导致了会话2中的DML无法执行。接下来，我们演示如何通过gdb去定位执行了FTWL的会话。见下面的步骤

1. 找出myql的进程id， ps -ef | grep mysql

` root 7743 2366 0 05:07 ? 00:00:01 /u02/mysql/bin/mysqld 
`
2.利用gdb来跟踪mysql进程 执行 gdb -p 7743

3.在mysql把已经连接的会话保存在一个叫global_thread_list的全局变量中在这个变量中的thread有一个叫global_read_lock的变量来表示持有锁的情况。所以我们只有在gdb中找global_read_lock不为空的thread即可。所以我们在gdb中执行下面的语句

`(gdb) pset global_thread_list THD*
elem[0]: $1 = (THD *) 0x4a55de0
elem[1]: $2 = (THD *) 0x4a5cf10
elem[2]: $3 = (THD *) 0x4b24aa0
Set size = 3

`
上面的命令输出了三个会话的内存地址。接下来我们根据这些内存地址去查找每个会话各自对应的global_read_lock

4.依次在gdb中打印上面三个会话中的global_read_lock和thread_id的值

`(gdb) p ((THD *) 0x4a55de0)->global_read_lock
$4 = {
 static m_active_requests = 1, 
 m_state = Global_read_lock::GRL_NONE, 
 m_mdl_global_shared_lock = 0x0, 
 m_mdl_blocks_commits_lock = 0x0
} //这个会话的Global_read_lock为空，不是我们要找的

(gdb) p ((THD *) 0x4a5cf10)->global_read_lock
$5 = {
 static m_active_requests = 1, 
 m_state = Global_read_lock::GRL_NONE, 
 m_mdl_global_shared_lock = 0x0, 
 m_mdl_blocks_commits_lock = 0x0
} //这个会话的Global_read_lock也为空，不是我们要找的

(gdb) p ((THD *) 0x4b24aa0)->global_read_lock
$6 = {
 static m_active_requests = 1, 
 m_state = Global_read_lock::GRL_ACQUIRED_AND_BLOCKS_COMMIT, 
 m_mdl_global_shared_lock = 0x7f6034002bb0, 
 m_mdl_blocks_commits_lock = 0x7f6034002c20
} 
//这个会话的Global_read_lock不为空，GRL_ACQUIRED_AND_BLOCKS_COMMIT表示全局读锁与commit锁，这个就是我们要好的。我接下来打印出它的thread_id
p ((THD *) 0x4b24aa0)->thread_id
$7 = 8 //8号会话执行了FTWL

`

5.我们可以通过执行kill 8结束这个会话来释放全局的锁。让被堵住的会话，继续运行下去。

`在新开的mysql会话中，执行下面的语句

mysql> kill 8

以前被堵在的会话中，会看到下面的结果
mysql> delete from t;
Query OK, 0 rows affected (40 min 20.73 sec)
`

## 小结

由于FTWL持有的是MetaDataLock类型的锁，所以给我们定位问题的源头带来很大的困难。很多同学在解决类似的问题的时候，会把运行时间最长的几个会话杀掉。这种方法并不可取。因为造成拥堵的源头并没有找到。所以我给大家提供了一个利用调试工具抓取mysql内部状态变量的方法来定位这类问题的源头。希望大家喜欢。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)