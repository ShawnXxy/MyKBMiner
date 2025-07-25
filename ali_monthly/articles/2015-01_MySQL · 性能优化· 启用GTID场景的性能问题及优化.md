# MySQL · 性能优化· 启用GTID场景的性能问题及优化

**Date:** 2015/01
**Source:** http://mysql.taobao.org/monthly/2015/01/03/
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

 ## MySQL · 性能优化· 启用GTID场景的性能问题及优化 
 Author: 

 **背景**

MySQL从5.6版本开始支持GTID特性，也就是所谓全局事务ID，在整个复制拓扑结构内，每个事务拥有自己全局唯一标识。GTID包含两个部分，一部分是实例的UUID，另一部分是实例内递增的整数。

GTID的分配包含两种方式，一种是自动分配，另外一种是显式设置session.gtid_next，下面简单介绍下这两种方式：

**自动分配**

如果没有设置session级别的变量gtid_next，所有事务都走自动分配逻辑。分配GTID发生在GROUP COMMIT的第一个阶段，也就是flush stage，大概可以描述为：

`Step 1：事务过程中，碰到第一条DML语句需要记录Binlog时，分配一段Gtid事件的cache，但不分配实际的GTID
Step 2：事务完成后，进入commit阶段，分配一个GTID并写入Step1预留的Gtid事件中，该GTID必须保证不在gtid_owned集合和gtid_executed集合中。 分配的GTID随后被加入到gtid_owned集合中。
Step 3：将Binlog 从线程cache中刷到Binlog文件中。
Step 4：将GTID加入到gtid_executed集合中。
Step 5：在完成sync stage 和commit stage后，各个会话将其使用的GTID从gtid_owned中移除。
`

**显式设置**

用户通过设置session级别变量gtid_next可以显式指定一个GTID，流程如下：

`Step 1：设置变量gtid_next，指定的GTID被加入到gtid_owned集合中。
Step 2：执行任意事务SQL，在将binlog从线程cache刷到binlog文件后，将GTID加入到gtid_executed集合中。
Step 3：在完成事务COMMIT后，从gtid_owned中移除。
`

备库SQL线程使用的就是第二种方式，因为备库在apply主库的日志时，要保证GTID是一致的，SQL线程读取到GTID事件后，就根据其中记录的GTID来设置其gtid_next变量。

**问题**

由于在实例内，GTID需要保证唯一性，因此不管是操作gtid_executed集合和gtid_owned集合，还是分配GTID，都需要加上一个大锁。我们的优化主要集中在第一种GTID分配方式。

对于GTID的分配，由于处于Group Commit的第一个阶段，由该阶段的leader线程为其follower线程分配GTID及刷Binlog，因此不会产生竞争。

而在Step 5，各个线程在完成事务提交后，各自去从gtid_owned集合中删除其使用的gtid。这时候每个线程都需要获取互斥锁，很显然，并发越高，这种竞争就越明显，我们很容易从pt-pmp输出中看到如下类似的trace：

`ha_commit_trans—&gt;MYSQL_BIN_LOG::commit—&gt;MYSQL_BIN_LOG::ordered_commit—&gt;MYSQL_BIN_LOG::finish_commit—&gt;Gtid_state::update_owned_gtids_impl—&gt;lock_sidno
`

这同时也会影响到GTID的分配阶段，导致TPS在高并发场景下的急剧下降。

**解决**

实际上对于自动分配GTID的场景，并没有必要维护gtid_owned集合。我们的修改也非常简单，在自动分配一个GTID后，直接加入到gtid_executed集合中，避免维护gtid_owned，这样事务提交时就无需去清理gtid_owned集合了，从而可以完全避免锁竞争。

当然为了保证一致性，如果分配GTID后，写入Binlog文件失败，也需要从gtid_executed集合中删除。不过这种场景非常罕见。

**性能数据**

使用sysbench，100张表，每张10w行记录，update_non_index.lua，纯内存操作，innodb_flush_log_at_trx_commit = 2，sync_binlog = 1000

`并发线程 原生 修改后
32 24500 25000
64 27900 29000
128 30800 31500
256 29700 32000
512 29300 31700
1024 27000 31000
`

从测试结果可以看到，优化前随着并发上升，性能出现下降，而优化后则能保持TPS稳定。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)