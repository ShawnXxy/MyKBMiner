# MySQL · 捉虫动态 · 连接断开导致XA事务丢失

**Date:** 2015/04
**Source:** http://mysql.taobao.org/monthly/2015/04/05/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 04
 ](/monthly/2015/04)

 * 当期文章

 MySQL · 引擎特性 · InnoDB undo log 漫游
* TokuDB · 产品新闻 · RDS TokuDB小手册
* TokuDB · 特性分析 · 行锁(row-lock)与区间锁(range-lock)
* PgSQL · 社区动态 · 说一说PgSQL 9.4.1中的那些安全补丁
* MySQL · 捉虫动态 · 连接断开导致XA事务丢失
* MySQL · 捉虫动态 · GTID下slave_net_timeout值太小问题
* MySQL · 捉虫动态 · Relay log 中 GTID group 完整性检测
* MySQL · 答疑释惑 · UPDATE交换列单表和多表的区别
* MySQL · 捉虫动态 · 删被引用索引导致crash
* MySQL · 答疑释惑 · GTID下auto_position=0时数据不一致

 ## MySQL · 捉虫动态 · 连接断开导致XA事务丢失 
 Author: 印风 

 我们看到在MySQL 5.7版本里大量遗留很多年的bug都被fix掉了，bug#12161就是其中一个，该bug在2005年第一次report到Bug list上，十年之后终于在MySQL 5.7.7 第一个RC版本被fix了。

## Bug描述

当我们显式开启一个XA事务，执行操作，并完成XA PREPARE后，如果Kill session或者主动断开再重连执行XA RECOVER，之前的这个XA事务就会直接丢失掉了。

例如：

`mysql> XA BEGIN 'abc';
Query OK, 0 rows affected (0.00 sec)
 
mysql> INSERT INTO t1 VALUES (1,2,3);
Query OK, 1 row affected (0.00 sec)
 
mysql> XA END 'abc';
Query OK, 0 rows affected (0.00 sec)
 
mysql> XA PREPARE 'abc';
Query OK, 0 rows affected (0.00 sec)
 
mysql> Ctrl-C -- exit!
Aborted
 
mysql> XA RECOVER;
Empty set (0.00 sec)
`

有趣的是，如果在XA PREPARE后把实例KILL掉，是可以通过XA RECOVER恢复的：

`mysql> XA RECOVER;
+----------+--------------+--------------+------+
| formatID | gtrid_length | bqual_length | data |
+----------+--------------+--------------+------+
| 1 | 3 | 0 | abc |
+----------+--------------+--------------+------+
1 row in set (0.00 sec)
 
mysql> XA COMMIT 'abc';
Query OK, 0 rows affected (0.00 sec)
`

虽然实例异常重启可以恢复事务，但引入的另外一个问题是：事务变更的binlog丢失，导致主备数据不一致。

bug产生的原因也很简单：在退出session时，线程总是会去无条件的回滚掉自己尚未提交的事务。

## 官方修复

### 持久化
为了解决这个问题，将XA的两阶段记录到了Binlog中；

对于上文描述的序列，当执行到XA PREPARE时，记录第一阶段的binlog，如下：

`Query event : XA START X'616263',X'’,1 // 这里的'616262'即是'abc'的十六进制编码
Table_map event
Write_rows event
Query event：XA END X'616263',X'',1
XA_prepare event： XA PREPARE X'616263',X'’,1
`

这时候该XA事务同时在InnoDB层（事务处于Prepare状态，Redo持久化到磁盘）和Server层都有持久化信息。

其中XA_PREPARE事件是新引入的事件类型（内部类为XA_prepare_event），以后版本升级需要注意到这个低版本不兼容事件。

然后再执行XA COMMIT ‘abc’，产生新的事件：

`Query event：XA COMMIT X'616263',X'',1
`

如果执行XA ROLLBACK，则记录：

`Query event：XA ROLLBACK X'616263',X'',1
`

由于XA PREPARE和XA COMMIT是分开执行的，因此在这两个事件中间可能存在别的事务，备库复制线程需要处理这种情况。

为了实现XA PREPARE写binlog，对binlog_prepare进行了扩展，这里会调用mysql_bin_log.commit， 将cache中的binlog刷到文件中。

Tips：XID可以包含三个部分：gtrid, [, bqual [, format ID]]，其中gtrid是必选的，表示全局标识，bqual是分支标识，默认为空’‘，format ID是一个unsigned整型，默认值为1，在上例中，我们只指定了gtrid为’abc’，因此bqual段和format ID均为默认值。更具体的描述参考[官方文档](http://dev.mysql.com/doc/refman/5.7/en/xa-statements.html)。

### 如何恢复

当会话断开时（例如kill session或者一次干净的shutdown/restart操作），我们必须要能恢复该事务，之前的逻辑是在cleanup时，直接回滚所有的活跃事务。在新版本中，对XA PREPARE的事务做了特殊处理（THD::cleanup），如果处于Prepare状态，就将事务的in_recovery设置为TRUE，并更新到hash表transaction_cache中（transaction_cache_detach），该hash表用于维护所有XA事务。

对于非XA的活跃事务，在会话断开时，依然采用回滚策略。

当重连客户端后，我们可以直接执行 XA COMMIT ‘abc’，这时候会通过XID关键字去搜索transaction_cache并将对应的事务提交掉。

同时BINLOG的状态要保持一致，如果会话断开前的XA PREPARE没有记录Binlog， 重连后执行XA COMMIT也不应该记录。

### 备库复制

由于XA PREPARE和XA COMMIT是分开记录的，当碰到XA COMMIT时，备库采用等待之前的事务全部完成，然后再执行的方式（相当于退化到串行）。

另外，我们知道在一个正常的会话过程中，总是为其cache一个事务对象，新的事务会重用这个事务对象，避免多次分配；而XA事务的COMMIT和PREPARE是分离的，需要为XA事务单独分配事务对象。因此复制线程执行XA START时，将其拥有的事务对象临时保存起来（detach_native_trx），当执行到XA_prepare_log_event事件时，再将其恢复给复制线程，同时XA事务对象关闭read view，将is_recovered设置为TRUE（函数innodb_replace_trx_in_thd）。

随后复制线程在执行到XA COMMIT时直接根据XID找到对应的XA事务进行提交。

## 参考：
[WL#6860](http://dev.mysql.com/worklog/task/?id=6860) Binlogging XA-prepared transaction
Github：git show f4c37f7aea732763947980600c6882ec908a54a0
MySQL 5.7.7-RC

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)