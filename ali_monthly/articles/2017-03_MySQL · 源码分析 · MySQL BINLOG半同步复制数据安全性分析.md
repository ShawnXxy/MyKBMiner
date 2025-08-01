# MySQL · 源码分析 · MySQL BINLOG半同步复制数据安全性分析

**Date:** 2017/03
**Source:** http://mysql.taobao.org/monthly/2017/03/07/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 03
 ](/monthly/2017/03)

 * 当期文章

 MySQL · 引擎特性 · InnoDB IO子系统
* PgSQL · 特性分析 · Write-Ahead Logging机制浅析
* MySQL · 性能优化 · MySQL常见SQL错误用法
* MSSQL · 特性分析 · 列存储技术做实时分析
* MySQL · 新特性分析 · 5.7中Derived table变形记
* MySQL · 实现分析 · 对字符集和字符序支持的实现
* MySQL · 源码分析 · MySQL BINLOG半同步复制数据安全性分析
* HybridDB · 性能优化 · Count Distinct的几种实现方式
* PgSQL · 应用案例 · PostgreSQL OLAP加速技术之向量计算
* MySQL · myrocks · myrocks监控信息

 ## MySQL · 源码分析 · MySQL BINLOG半同步复制数据安全性分析 
 Author: 荣生 

 半同步复制（semisynchronous replication）MySQL使用广泛的数据复制方案，相比于MySQL内置的异步复制它保证了数据的安 全，本文从主机在Server层提交事务开始一直到主机确认收到备机回复进行一步步解析，来看MySQL的半同步复制是怎么保证数 据安全的。本文基于MySQL 5.6源码，为了简化本文只分析DML的核心的事务处理过程，并假定事务只涉及innodb存储引擎。

## MySQL的事务提交流程

在MySQL中事务的提交Server层最后会调用函数ha_commit_trans()，该函数负责处理binlog层和存储引擎层的提交，它先调用 tc_log->prepare()在引擎层生成一个XA事务，然后再调用tc_log->commit()来提交事务，这里的tc_log是在mysqld启动时就生 成的一个MYSQL_BIN_LOG类的对象。简化后代码片断类似：

`int ha_commit_trans(THD *thd, bool all, bool ignore_global_read_lock)
{
 //...
 error= tc_log->prepare(thd, all);

 if (error || (error= tc_log->commit(thd, all)))
 {
 ha_rollback_trans(thd, all);
 error= 1;
 goto end;
 }
 //...
}
`

MYSQL_BIN_LOG::prepare()函数调用ha_prepare_low()，该函数再调用存储引擎层（这里指innodb）的prepare在存储层生成XA 事务。MYSQL_BIN_LOG::commit()先在binlog层加入一个Xid_log_event类型的日志作为XA事务在binlog层提交的标志，注意这 里并没有调用操作系统的fsync。该函数最后调用会调用MYSQL_BIN_LOG::ordered_commit()，做binlog文件的磁盘fsync和提交 到存储引擎。

MYSQL_BIN_LOG::ordered_commit()是比较重要的函数，该函数的处理步骤如下：

1. 将binlog数据刷写到文件中
2. 将当前的binlog文件名和位点注册到semisync模块中，以便后面等待备机的回复
3. 调用函数MYSQL_BIN_LOG::sync_binlog_file()将binlog文件sync到磁盘，到这里事务将不能回滚，即使mysqld崩溃了事务 也会最终提交。
4. 调用MYSQL_BIN_LOG::update_binlog_end_pos()更新binlog最后sync的位点信息，这时为备库复制服务的binlog dump线程 才可以读到这个事务，可参考Log_event::read_log_event()
5. 如果semisync模块配置了rpl_semi_sync_master_wait_point为 after_sync，那么当前Session将在这里等待备机回复再继 续。
6. ordered_commit()接下来会最终调用到 ha_commit_low()在存储引擎层提交
7. 如果rpl_semi_sync_master_wait_point参数为after_commit，当前Session就会在ordered_commit()接下来调用的 MYSQL_BIN_LOG::finish_commit()函数里等待备机的回复，

以上可以看出after_sync和after_commit的主要区别是，当备机确认收到日志时，主机上的该事务是否对其他session可见， after_sync是不可见（因为在存储引擎层还没有提交），after_commit是可见。after_commit可能导致在主机上被其他事务看 见了的事务在切换到备机后又消失了，所以MySQL 5.7默认使用after_sync。

## MySQL的事务恢复流程

mysqld崩溃之后的事务恢复最终是通过MYSQL_BIN_LOG::recover()进行的，调用栈： mysqld_main() -> init_server_components() -> MYSQL_BIN_LOG::open() -> MYSQL_BIN_LOG::open_binlog() -> MYSQL_BIN_LOG::recover()。open_binlog()函数通过binlog文件头上的标志可以知道该文件在mysqld退出时没有正常关闭，然 后就调用recover()函数进行恢复。

MYSQL_BIN_LOG::recover()首先扫描binlog日志扫出在binlog里已经提交的事务加到一个commitlist里，然后调用 ha_recover()函数，该函数先调用innodb层的相关函数扫描出在innodb层已经prepare的事务，然后将在commitlist里的事务全 部提交。

从以上MySQL事务提交和恢复流程可以看出，在最终备机提交事务，必然在主机上是提交的，也就是主机的事务必然比备机更全。

## 主机和备机同步的处理流程

前文已经提到在MYSQL_BIN_LOG::ordered_commit()函数中，用户session会将要等待备机回复的事务对应的binlog文件名和位 点注册到semisync模块中，然后在向备机发送binlog的主函数里mysql_binlog_send()中，将这些事务对应的binlog event数据 包加上要求备机回复的标志，见函数ReplSemiSyncMaster::updateSyncHeader()。主机在mysqld启动时就启动了一个 ack_receiver线程，每次有新的备机连接上来，就把对应的服务线程注册到ack_receiver中，见函数 ReplSemiSyncMaster::dump_start()，ack_receiver负责接收所有备机的回复。备机在handle_slave_io()函数中读到一个 event的数据包就会检查是否有要求回复的标志，如果有则在将binlog刷到本地磁盘后向主机发送回复报文，回复的报文的内容 包含收到的binlog文件名和位点。流程大致如下：

`while (!io_slave_killed(thd,mi))
{
 // ...
 event_len= read_event(mysql, mi, &suppress_warnings);
 mi->repl_semisync_slave.slaveReadSyncHeader((const char*)mysql->net.read_pos + 1,
 event_len, &(mi->semi_ack), &event_buf,
 &event_len);
 // ...
 if (queue_event(mi, event_buf, event_len))
 {
 mi->report(ERROR_LEVEL, ER_SLAVE_RELAY_LOG_WRITE_FAILURE,
 ER(ER_SLAVE_RELAY_LOG_WRITE_FAILURE),
 "could not queue event from master");
 goto err;
 }
 // ...
 if((mi->semi_ack & SEMI_SYNC_NEED_ACK) &&
 mi->repl_semisync_slave.slaveReply(mi))
 {
 mi->report(ERROR_LEVEL, ER_SLAVE_FATAL_ERROR,
 ER(ER_SLAVE_FATAL_ERROR),
 "Failed to call 'slaveReply'");
 goto err;
 }
 // ...
 }
`

ack_receiver线程的主线程函数是Ack_receiver::run()，该函数调用poll()监听在所有已注册的slave服务线程的socket上， 接听slave的回复报文，当接收到一个回复报文后，ack_receiver会记下当前的回复报文中的binlog文件名和位点，并在自己的 注册列表中删除在这个位点之前的事务，然后通过cond_broadcast()唤醒等待备机回复的用户session线程，这些线程通过比较 自己的等待位点和ack_receiver记下的回复报文位点决定是否结束等待。

## 总结

通过以上分析可以看出在同步复制的模式上，MySQL通过非常严格的流程保证了用户Session执行完事务返回给客户端后，该事 务也必然已同步到了备机的磁盘上。同时保证了出现在备机的事务必然在主机上已经是安全提交了的，也就是在任何时刻主机 上的事务一定是大于等于备机的。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)