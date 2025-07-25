# MySQL · 性能优化· Group Commit优化

**Date:** 2015/01
**Source:** http://mysql.taobao.org/monthly/2015/01/01/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 01
 ](/monthly/2015/01)

 * 当期文章

 MySQL · 性能优化· Group Commit优化
* MySQL · 新增特性· DDL fast fail
* MySQL · 性能优化· 启用GTID场景的性能问题及优化
* MySQL · 捉虫动态· InnoDB自增列重复值问题
* MySQL · 优化改进· 复制性能改进过程
* MySQL · 谈古论今· key分区算法演变分析
* MySQL · 捉虫动态· mysql client crash一例
* MySQL · 捉虫动态· 设置 gtid_purged 破坏AUTO_POSITION复制协议
* MySQL · 捉虫动态· replicate filter 和 GTID 一起使用的问题
* TokuDB·特性分析· Optimize Table

 ## MySQL · 性能优化· Group Commit优化 
 Author: 

 **背景**

关于Group Commit网上的资料其实已经足够多了，我这里只简单的介绍一下。

众所周知，在MySQL5.6之前的版本，由于引入了Binlog/InnoDB的XA，Binlog的写入和InnoDB commit完全串行化执行，大概的执行序列如下：

`InnoDB prepare （持有prepare_commit_mutex）；
write/sync Binlog；
InnoDB commit (写入COMMIT标记后释放prepare_commit_mutex)。
`

当sync_binlog=1时，很明显上述的第二步会成为瓶颈，而且还是持有全局大锁，这也是为什么性能会急剧下降。

很快Mariadb就提出了一个Binlog Group Commit方案，即在准备写入Binlog时，维持一个队列，最早进入队列的是leader，后来的是follower，leader为搜集到的队列中的线程依次写Binlog文件, 并commit事务。Percona 的Group Commit实现也是Port自Mariadb。不过仍在使用Percona Server5.5的朋友需要注意，该Group Commit实现可能破坏掉Semisync的行为，感兴趣的点击 [bug#1254571](https://bugs.launchpad.net/percona-server/5.5/+bug/1254571)

Oracle MySQL 在5.6版本开始也支持Binlog Group Commit，使用了和Mariadb类似的思路，但将Group Commit的过程拆分成了三个阶段：flush stage 将各个线程的binlog从cache写到文件中; sync stage 对binlog做fsync操作（如果需要的话）；commit stage 为各个线程做引擎层的事务commit。每个stage同时只有一个线程在操作。

Tips：当引入Group Commit后，sync_binlog的含义就变了，假定设为1000，表示的不是1000个事务后做一次fsync，而是1000个事务组。

Oracle MySQL的实现的优势在于三个阶段可以并发执行，从而提升效率。更进一步的理解，可以参考[这篇博客](http://mysqlmusings.blogspot.com/2012/06/binary-log-group-commit-in-mysql-56.html)

**XA Recover**

在Binlog打开的情况下，MySQL默认使用MYSQL_BIN_LOG来做XA协调者，大致流程为：

`1.扫描最后一个Binlog文件，提取其中的xid；
2.InnoDB维持了状态为Prepare的事务链表，将这些事务的xid和Binlog中记录的xid做比较，如果在Binlog中存在，则提交，否则回滚事务。
`

通过这种方式，可以让InnoDB和Binlog中的事务状态保持一致。显然只要事务在InnoDB层完成了Prepare，并且写入了Binlog，就可以从崩溃中恢复事务，这意味着我们无需在InnoDB commit时显式的write/fsync redo log。

Tips：MySQL为何只需要扫描最后一个Binlog文件呢 ？ 原因是每次在rotate到新的Binlog文件时，总是保证没有正在提交的事务，然后fsync一次InnoDB的redo log。这样就可以保证老的Binlog文件中的事务在InnoDB总是提交的。

**问题**

其实问题很简单：每个事务都要保证其Prepare的事务被write/fsync到redo log文件。尽管某个事务可能会帮助其他事务完成redo 写入，但这种行为是随机的，并且依然会产生明显的log_sys->mutex开销。

**优化**

从XA恢复的逻辑我们可以知道，只要保证InnoDB Prepare的redo日志在写Binlog前完成write/sync即可。因此我们对Group Commit的第一个stage的逻辑做了些许修改，大概描述如下：

`Step1\. InnoDB Prepare，记录当前的LSN到thd中；
Step2\. 进入Group Commit的flush stage；Leader搜集队列，同时算出队列中最大的LSN。
Step3\. 将InnoDB的redo log write/fsync到指定的LSN
Step4\. 写Binlog并进行随后的工作(sync Binlog, InnoDB commit , etc)
`

通过延迟写redo log的方式，显式的为redo log做了一次组写入，并减少了log_sys->mutex的竞争。

目前官方MySQL已经根据我们report的bug#73202锁提供的思路，对5.7.6的代码进行了优化，对应的Release Note如下：

`When using InnoDB with binary logging enabled, concurrent transactions written in the InnoDB redo log are now grouped together before synchronizing 
to disk when innodb_flush_log_at_trx_commit is set to 1, which reduces the amount of synchronization operations. This can lead to improved performance.
`

**性能数据**

简单测试了下，使用sysbench, update_non_index.lua, 100张表，每张10w行记录，innodb_flush_log_at_trx_commit=2, sync_binlog=1000，关闭Gtid

```
并发线程 原生 修改后
32 25600 27000
64 30000 35000
128 33000 39000
256 29800 38000

```

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)