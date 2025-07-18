# MySQL · 社区贡献 · AliSQL那些事儿

**Date:** 2016/09
**Source:** http://mysql.taobao.org/monthly/2016/09/01/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 09
 ](/monthly/2016/09)

 * 当期文章

 MySQL · 社区贡献 · AliSQL那些事儿
* PetaData · 架构体系 · PetaData第二代低成本存储体系
* MySQL · 社区动态 · MariaDB 10.2 前瞻
* MySQL · 特性分析 · 执行计划缓存设计与实现
* PgSQL · 最佳实践 · pg_rman源码浅析与使用
* MySQL · 捉虫状态 · bug分析两例
* PgSQL · 源码分析 · PG优化器浅析
* MongoDB · 特性分析· Sharding原理与应用
* PgSQL · 源码分析 · PG中的无锁算法和原子操作应用一则
* SQLServer · 最佳实践 · TEMPDB的设计

 ## MySQL · 社区贡献 · AliSQL那些事儿 
 Author: 印风 

 一直以来我们都在不断对我们的阿里云MySQL分支做极致的性能优化及功能扩展。我们从社区的分支，如上游版本及Percona Server上学习新的改进和功能，并引入到我们的分支中。同时我们也将我们的一些改进思路反馈到上游，让整个社区也能享受到我们的成果。

本文主要介绍下AliSQL贡献给上游MySQL5.7版本的一些跟性能相关的优化。注意这里只摘取了几个比较有意思的优化，在即将开源的AliSQL中，我们将包含更加全面、更加强悍的性能优化补丁以及更丰富的扩展功能，敬请期待 :)

## 优化row模式下的prepare statement性能

修复版本： MySQL 5.7.2

这算是一个老问题了，但不是一个通用场景优化。在一次线上业务场景中，我们发现使用Prepare Statement语句进行数据插入时，效率非常低下。我们通过pt-pmp观察到如下堆栈:

`bmove_upp,String::replace,insert_params_with_log,Prepared_statement::set_parameters,Prepared_statement::execute_loop,mysqld_stmt_execute...
`

相应的perf top输出：

`94.14% bmove_upp 
0.26% setup_one_conversion_function(THD*, Item_param*, unsigned char) 
0.26% mtr_commit 
`

通过分析代码，我们发现当打开Binlog并且当前是数据修改类型的语句时，会去将Prepare Statement中的”?”替换成真正的数据。而在拼凑的过程中又涉及到低效的字符串操作，从而导致性能下降。一个具体的例子是，如果构建一条Prepare语句进行批量的插入操作，那么这里涉及到的SQL构建开销就会非常庞大。

这里拼凑SQL的目的主要是为了记录binlog。如果使用的是ROW模式，实际上没必要进行这样的转换 (但如果slow log或者general log打开的话，依然需要拼凑SQL)。因此我们加入了对ROW模式的判断。

在优化后，我们以一个比较极端的例子进行测试，即一次INSERT 50,000条记录，相比之前的数据插入速度提升了好几倍。当然正常场景下不会有这么大的提升。

当然这里的字符串操作太过低效也是一个优化点，使用的是String::replace函数，每次填充参数时，都要重分配buffer并拷贝参数。在[Bug#73056](http://bugs.mysql.com/bug.php?id=73056)，有人提出了改进的意见，即一次分配好内存空间，然后填进去字符串，而不是每次replace。这个问题将在MySQL 8.0被修复，但目前还不得而知怎么修的。

## 记录锁counter优化

修复版本： MySQL 5.7.5

当执行类似`SHOW ENGINE INNODB STATUS`这样的语句时，会打印出每个session持有的事务行锁的个数。其计算方式是遍历该事务持有的所有锁记录，然后算出锁的个数。而这个过程是持有全局大锁lock_sys->mutex的。这意味着如果有高频次的监控需求，就可能对性能产生影响。

修改方式也很简单，直接为每个事务添加一个计数器(trx_lock_t::n_rec_locks)来维护行锁的个数。

## MySQL组提交优化

修复版本： MySQL 5.7.6

当Binlog打开时，MySQL使用Group Commit的方式来保证性能。大约遵循如下过程：

1. 事务在引擎层进行InnoDB Prepare，即将对应的undo状态设置为Prepare，同时对redo日志进行持久化。这一步是无序的，由用户线程各自发起。
2. 某个事务开始进入binlog commit阶段，如果当前flush stage队列中没有线程，则自己作为队列头leader，否则作为follower。leader在加入队列到开始真正的写binlog之间会有一段时差，搜集到的队列长度取决于并发度。当leader带着队列开始写binlog时，其他会话将允许进入flush stage 队列，并形成新的组。
3. 随后进入sync stage及commit stage，多个链表可能被组建成更大的链表，并进行有序的提交。

我们的优化主要是在Prepare阶段，为什么在Prepare阶段要持久化redo呢？这是由MySQL的XA Recover恢复机制决定的。其内部使用InnoDB和Binlog做XA的方式来实现崩溃恢复安全：当事务处于Prepare状态时，如果binlog已经写入文件了，就把该事务提交掉，否则将其回滚。binlog日志和innodb中的undo通过一致的Xid值相互关联。

基于上述事实，只要我们保证在写binlog之前把Prepare的事务持久化了即可。我们将上述第一步写日志的动作转移到group commit的flush stage。这样做的好处是不仅降低了全局大锁log_sys->mutex的开销，也将日志写进行了显式组提交，在高并发负载下具有更好的吞吐量。

在使用sysbench, update-non-index，纯内存更新的测试场景下，我们最多获得了将近30%的性能提升。具体取决于并发度，并发越高，提升越大。

早前的另外一篇[月报](http://mysql.taobao.org/index.php?title=MySQL%E5%86%85%E6%A0%B8%E6%9C%88%E6%8A%A5_2015.01#MySQL_.C2.B7_.E6.80.A7.E8.83.BD.E4.BC.98.E5.8C.96.C2.B7_Group_Commit.E4.BC.98.E5.8C.96)也对该改进做了描述，感兴趣的自取。

## InnoDB Change Buffer优化

修复版本： MySQL 5.7.6

这是一个比较漫长的故事，故事的起因从2013年的[bug#70768](https://bugs.mysql.com/bug.php?id=70768)开始，早期有一个全局的latch数组，数组大小为64。这意味着不同的表有可能使用到同一个Latch来保护统计信息的更新，从而导致比较严重的锁冲突问题。为了解决冲突，为每个table对象创建了一个stats_latch。

然而这里引入了有个问题，InnoDB为了辅助进行记录的解析，常常会创建一个线程私有的dummy table/dummy index。对于这样的表和索引，为其创建latch是没有必要的。那么这里为什么会产生的严重的性能问题呢([bug#71708](https://bugs.mysql.com/bug.php?id=71708))? 主要有两点原因：

1. 当内存远小于数据集，并且二级索引被频繁更新时，change buffer会被触发并被高频度使用，相应的dummy table/index也会被频繁创建和销毁。
2. dummy table/index的创建和销毁本身并没有太大的开销，但挂在其上的stats_latch（以及其他相关mutex）需要去持有全局锁rw_lock_list_mutex来管理rw-lock/mutex队列。这种锁冲突在极端场景下会导致change buffer的性能极端低下，甚至还不如关闭change buffer的性能。

为了修复[bug#71708](https://bugs.mysql.com/bug.php?id=71708)，上游对stats_latch引入了延迟创建的功能，即只在第一次使用到latch时才去创建下，从而绕过了该问题。

你以为故事已经结束了吗？ 并没有！！官方的fix并没有考虑到还有表对象上的autoinc_mutex以及index->zip_pad.mutex。这种互斥锁的创建/销毁对于dummy table及dummy index而言都是没有意义的。

我们提交了[Bug#73361](https://bugs.mysql.com/bug.php?id=73361)来描述这个问题。官方对此进行了修复，对表对象上的mutex也采用延迟创建的方法，这才彻底修复了该bug。

## 提升日志并发写性能

修复版本：MySQL 5.7.13

InnoDB的日志模块大体可以描述如下：

1. 当某些操作需要日志保护时，InnoDB通过mini transaction（简称mtr），首先将日志写到本地私有cache中
2. 当操作结束时，提交mtr，将本地cache中的日志写到全局log buffer中
3. log buffer中的日志的写文件操作主要通过如下几种情况触发：后台线程定期运行; 事务提交; 刷脏页

拷贝日志到log buffer和将log buffer数据写到磁盘都需要全局大锁的保护，这意味着，在写日志时，我们无法向buffer中拷贝，从而影响了整体的吞吐量。

为了解决这个问题，我们引入了双buffer方案，假定名为buf1和buf2，并新增了一个write_mutex。

* 初始化log_sys->buf指向buf1，mtr_commit操作会将日志拷贝到buf1中。
* 当准备将Log buffer中数据写入到磁盘时，在log_sys->mutex的保护下，将buf1的最后一个block拷贝到buf2开始部位（保证日志的连续性），将log_sys->buf指向buf2，然后释放log_sys->mutex。这里mutex的加锁范围被大大的减少。此时并发线程就可以向buf2中拷贝日志了。
* 将buf1的日志写盘，文件写操作通过新的write_mutex保护。

通过轮换的使用两个Buffer，有效的提升了实例的吞吐量，尤其是在innodb_flush_log_at_trx_commit设置为2的时候，我们在纯更新场景下测试获得了接近20%的性能提升。

## 降低只读事务的锁开销

修复版本: MySQL 5.7.14

MySQL5.7对只读事务场景做了大量的优化，包括移除只读事务链表，server层thr_lock优化等等。但并不意味着只读事务就没有优化的余地了。

在测试新的只读事务(非auto-commit，显式开启事务)逻辑时，我们发现在高并发下lock_sys->mutex的锁冲突非常厉害，从performance schema中观察到：

`mysql> SELECT COUNT_STAR, SUM_TIMER_WAIT, AVG_TIMER_WAIT, EVENT_NAME FROM events_waits_summary_global_by_event_name where COUNT_STAR > 0 and EVENT_NAME like 'wait/synch/%' order by SUM_TIMER_WAIT desc limit 10;
+------------+-----------------+----------------+---------------------------------------------+
| COUNT_STAR | SUM_TIMER_WAIT | AVG_TIMER_WAIT | EVENT_NAME |
+------------+-----------------+----------------+---------------------------------------------+
| 17739300 | 172218176088930 | 9707895 | wait/synch/mutex/innodb/lock_mutex |
| 35479372 | 77340476989560 | 2179785 | wait/synch/mutex/innodb/trx_sys_mutex |
| 35465620 | 27221504947890 | 767340 | wait/synch/mutex/sql/LOCK_table_cache |
| 159575929 | 20214954245040 | 126585 | wait/synch/mutex/sql/THD::LOCK_query_plan 
`

可以看到lock_sys->mutex的平均等待时间占据最高位。大量会话堵塞在如下堆栈：

`pthread_cond_wait,os_cond_wait(os0sync.cc:214),os_event_wait_low(os0sync.cc:214),sync_array_wait_event(sync0arr.cc:424),mutex_spin_wait(sync0sync.cc:579),mutex_enter_func(sync0sync.ic:220),pfs_mutex_enter_func(sync0sync.ic:220),lock_trx_release_locks(sync0sync.ic:220),trx_commit_in_memory(trx0trx.cc:1381), ...
`

原因是当事务是显式开启的事务时，InnoDB总是无条件的去加lock_sys->mutex，并尝试释放其持有的事务锁记录。而对于只读事务而言，通常其是不会持有锁的，也就无需去持有全局大锁了。

修改的方案很简单，只要事务上不持有事务锁，就不去加lock_sys->mutex。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)