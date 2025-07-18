# MySQL · 答疑解惑 · InnoDB 预读 VS Oracle 多块读

**Date:** 2015/05
**Source:** http://mysql.taobao.org/monthly/2015/05/04/
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

 ## MySQL · 答疑解惑 · InnoDB 预读 VS Oracle 多块读 
 Author: 冷香 

 ## 背景
目前，IO 仍然是数据库的性能杀手，为了提高 IO 利用率和吞吐量，不同的数据库都设计了不同的方法，本文就介绍下 InnoDB 提供的预读(read-ahead)功能，以及 Oracle 提供的多块读(multiblock-read)功能，并进行一些对比。

### InnoDB read-ahead
InnoDB 提供了两种预读的方式，一种是 Linear read ahead，由参数`innodb_read_ahead_threshold`控制，当你连续读取一个 extent 的 threshold 个 page 的时候，会触发下一个 extent 64个page的预读。另外一种是Random read-ahead，由参数`innodb_random_read_ahead`控制，当你连续读取设定的数量的page后，会触发读取这个extent的剩余page。

InnoDB 的预读功能是使用后台线程异步完成的。InnoDB启动了`innodb_read_io_threads`个后台线程，来完成IO request，并且可以使用Native AIO，在你的环境中如果安装了libaio，在MySQL实例启动的时候，查看系统日志：`InnoDB: Using Linux native AIO` 表明 InnoDB 已经使用Native AIO了。在Linear read ahead触发的时候，InnoDB通过`io_submit()`提交了下一个extent的64个pages的IO request，并由一个read IO thread完成。

### Oracle multiblock-read
当你要对堆表进行全表扫描，并需要大量IO的时候，通常在 session 级别设置`db_file_multiblock_read_count`，这样 Oracle 会在读取堆表结构的数据块的时候，一次IO读取多个数据块，大大减少了IO的次数。但这里一次合并IO请求的数据块，必须不能在buffer pool中，否则会分割IO请求。不过，在针对大表的汇总分析查找中，设置`db_file_multiblock_read_count`的效果是非常明显的。不过也要注意，不要在系统级别上设置过大的`db_file_multiblock_read_count`， 会造成buffer cache flooding。

## 场景分析

下面我们看两个非常典型的场景:

**1. 高并发，小IO的情况**
在高并发的场景下，sql响应时间主要取决于同步IO请求的时间，而InnoDB的预读通常不会触发，就算触发，更多的是预热(warmup)的效果，并不会对系统带来非常大的收益，对rt的影响也非常小。
而Oracle如果设置了`db_file_multiblock_read_count`，在这样的场景下，有可能会适得其反，因为一次同步IO请求的时间增加了。

所以在这样的场景下，InnoDB的read-ahead和Oracle的multiblock-read并不会带来太多的收益。我们看另外一个场景。

**2. 低并发，高IO吞吐**
通常，我们可能想在业务低峰期，对线上数据进行汇总查询。这时，希望能够完全使用主机的资源来完成sql的查询，在使用全表扫描的时候，InnoDB会触发read-ahead，每次提前异步读取下一个extent的page，加快读取的速度。
Oracle使用`db_file_multiblock_read_count`，一次IO读取多个block，提高读取的吞吐量。

## 问题

为什么在聚集查询的时候，Oracle的效果会比InnoDB要好？

这个问题，在针对机械盘的情况，又回到了 IOPS 和 throughput 的讨论上去了。InnoDB的read-ahead，在触发的时候，针对下一个extent，对每一个page提交了异步IO请求，也就是增加了IO request次数，虽然Native AIO和disk会有针对性合并IO，但仍然非常有限，而Oracle每次提交合并多个连续数据块的IO请求，能够更好利用disk的吞吐能力。

所以，InnoDB在针对aggregation类型的查询的时候，想要完全使用IO的吞吐能力，相比较Oracle的multiblock-read，会偏弱一点。

## 优化方法

针对InnoDB的机制，我们可以尝试几种优化方法:

1. 在session级别，提供可设置预读的触发条件，并使用多个后台线程来完成异步IO请求。因为没有减少小IO请求，作者尝试了这种方法，收益甚小；
2. 独立一个buffer pool，专门进行多块读，针对next extent，一次读取到buffer pool中，这种方式就和Oracle的multiblock-read比较类似了；
3. 终极优化方法，就是使用并行查询，Oracle在全表扫描的时候，使用`/* parallel */` hint方法启动多个进程完成查询，InnoDB的聚簇索引结构，需要逻辑分片，针对每一个分片启动一个线程完成查询。

读者如果有兴趣，可以进行一些尝试。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)