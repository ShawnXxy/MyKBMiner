# MySQL · 源码分析 · MySQL replication partial transaction

**Date:** 2017/08
**Source:** http://mysql.taobao.org/monthly/2017/08/03/
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

 ## MySQL · 源码分析 · MySQL replication partial transaction 
 Author: 张远 

 ## replication 概述
目前MySQL支持的replication方式多种多样

1. 普通的master-slave 异步replication
2. 半同步的semi-sync replication
3. 支持多通道的group replication和double binlog

如果按连接协议来区分，又可以分为

1. 非GTID模式，通过binlog文件名和文件的偏移来决定replication位点信息
2. GTID模式，通过GTID信息来决定replication位点信息

如果按apply binglog的方式来区分，又可以分为

1. 串行，按binlog event顺序依次执行
2. 并行，以db, table或transaction为粒度的并行复制，以及基于group commit的LOGICAL_CLOCK并行复制

不论哪种replication, 都离不开replication最基本的组件，

1. IO thread，负责从master拉取binlog.
2. SQL thread，负责apply relay log binlog.

## replication 异常
复制过程中，由于网络或者master主机宕机，都会造成slave IO thread异常中断。
例如以下事务在复制过程中发生上述异常，

`SET GTID_NEXT; # GTID设置为ON时 
BEGIN; 
INSERT row1;
INSERT row2;
COMMIT;
`
那么备库接收的binlog可能不包含完整的事务，备库可能仅接收到BEGIN，也可能只接收到INSERT row1.

然而，当IO thread恢复后，SQL线程怎么正确处理这种异常呢？

## 异常恢复
IO thread 异常中断后，SQL线程是正常工作的，SQL执行了部分事务， 它会等待IO 线程发送新的binlog. IO thread 线程恢复后，SQL线程可以选择继续执行事务或者回滚事务重新执行事务，这是由replication协议决定的。

1. GTID模式下，设置auto_position=1时，slave会根据GTID信息，从事务起点开始，重新将事务完整binlog发给备库。此时，备库需要回滚之前的部分事务。
2. GTID模式下，设置auto_position=0或非GTID模式下，slave会根据位点信息从master续传之前的binlog。此时，备库可以继续完成之前的部分事务。

继续执行事务比较简单，但是回滚之前的部分事务就比较复杂.

分为两种情况来分析：

* 串行复制

 串行复制时，完整的事务会由SQL thread来执行，当执行到GTID_LOG_EVENT时，会发这个GTID已经分配过了，这时候就可以回滚事物。具体参考

`Gtid_log_event::do_apply_event()

 if (thd->owned_gtid.sidno)
 {
 /*
 Slave will execute this code if a previous Gtid_log_event was applied
 but the GTID wasn't consumed yet (the transaction was not committed
 nor rolled back).
 On a client session we cannot do consecutive SET GTID_NEXT without
 a COMMIT or a ROLLBACK in the middle.
 Applying this event without rolling back the current transaction may
 lead to problems, as a "BEGIN" event following this GTID will
 implicitly commit the "partial transaction" and will consume the
 GTID. If this "partial transaction" was left in the relay log by the
 IO thread restarting in the middle of a transaction, you could have
 the partial transaction being logged with the GTID on the slave,
 causing data corruption on replication.
 */
 if (thd->transaction.all.ha_list)
 {
 /* This is not an error (XA is safe), just an information */
 rli->report(INFORMATION_LEVEL, 0,
 "Rolling back unfinished transaction (no COMMIT "
 "or ROLLBACK in relay log). A probable cause is partial "
 "transaction left on relay log because of restarting IO "
 "thread with auto-positioning protocol.");
 const_cast<Relay_log_info*>(rli)->cleanup_context(thd, 1);
 }
 gtid_rollback(thd);
 }
`
* 并行复制

 并行复制有别于串行复制，binlog event由worker线程执行。按串行复制的方式来回滚事务是行不通的，因为重新发送的事务binlog并不一定会分配原来的worker来执行。因此，回滚操作需交给coordinate线程(即sql线程)来完成。
 GTID模式下，设置auto_position=1时. IO thread重连时，都会发送
ROTATE_LOG_EVENT和FORMAT_DESCRIPTION_EVENT. 并且FORMAT_DESCRIPTION_EVENT的log_pos>0. 通过非auto_position方式重连的FORMAT_DESCRIPTION_EVENT的log_pos在send之前会被置为0. SQL线程通过执行FORMAT_DESCRIPTION_EVENT且其log_pos>0来判断是否应进入回滚逻辑。而回滚是通过构造Rollback event让work来执行的。

具体参考

`exec_relay_log_event()
/*
 GTID protocol will put a FORMAT_DESCRIPTION_EVENT from the master with
 log_pos != 0 after each (re)connection if auto positioning is enabled.
 This means that the SQL thread might have already started to apply the
 current group but, as the IO thread had to reconnect, it left this
 group incomplete and will start it again from the beginning.
 So, before applying this FORMAT_DESCRIPTION_EVENT, we must let the
 worker roll back the current group and gracefully finish its work,
 before starting to apply the new (complete) copy of the group.
 */
 if (ev->get_type_code() == FORMAT_DESCRIPTION_EVENT &&
 ev->server_id != ::server_id && ev->log_pos != 0 &&
 rli->is_parallel_exec() && rli->curr_group_seen_gtid)
 {
 if (coord_handle_partial_binlogged_transaction(rli, ev))
 /*
 In the case of an error, coord_handle_partial_binlogged_transaction
 will not try to get the rli->data_lock again.
 */
 DBUG_RETURN(1);
 }
`

MySQL官方针对此问题有过多次改进，详见以下commit

666aec4a9e976bef4ddd90246c4a31dd456cbca3

3f6ed37fa218ef6a39f28adc896ac0d2f0077ddb

9e2140fc8764feeddd70c58983a8b50f52a12f18

## 异常case处理

当slave SQL线程处于部分事务异常时，按上节的逻辑，IO thread恢复后，复制是可以正常进行的。但如果IO thread如果长时间不能恢复，那么SQL apply线程会一直等待新的binlog， 并且会一直持有事务中的锁。当slave切换为master后，新master会接受用户连接处理事务，这样SQL apply线程持有的事务锁，可能阻塞用户线程的事务。这是我们不希望看到的。

此时可以通过stop slave来停止SQL apply线程，让事务回滚释放锁。

另一种更好的方案是让SQL apply 线程自动识别这种情况，并加以处理。比如，增加等待超时机制，超时后自动kill sql 线程或回滚SQL线程的部分事务。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)