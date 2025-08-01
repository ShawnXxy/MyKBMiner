# MySQL · 捉虫动态 · InnoDB crash

**Date:** 2017/06
**Source:** http://mysql.taobao.org/monthly/2017/06/05/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 06
 ](/monthly/2017/06)

 * 当期文章

 MySQL · 源码分析 · Tokudb序列化和反序列化过程
* PgSQL · 应用案例 · HTAP视角,数据与计算的生态融合
* MySQL · 引擎特性 · 从节点可更新机制
* PgSQL · 特性分析 · 数据库崩溃恢复（下）
* MySQL · 捉虫动态 · InnoDB crash
* MSSQL · 实现分析 · SQL Server实现审计日志的方案探索
* MySQL · 源码分析 · InnoDB Repeatable Read隔离级别之大不同
* MySQL · myrocks · MyRocks之memtable切换与刷盘
* PgSQL · 最佳实践 · 云上的数据迁移
* MySQL · 社区新闻 · MariaDB 10.2 GA

 ## MySQL · 捉虫动态 · InnoDB crash 
 Author: 冷香 

 ## 问题描述

在 MySQL 官方最新的版本 MySQL 5.6.36 版本上，我们遇到了一个非常有意思的bug，实例几乎每个小时crash一次，查看其产生的 core file，发现如下的backtrace：

`#3 <signal handler called>
#4 0x00002b65596248a5 in raise (sig=6) at ../nptl/sysdeps/unix/sysv/linux/raise.c:64
#5 0x00002b6559626085 in abort () at abort.c:92
#6 0x00000000010deabe in dict_index_is_clust (index=0x0) at storage/innobase/include/dict0dict.ic:269
#7 0x00000000010f1efb in row_merge_drop_indexes (trx=0x2b656c027b28, table=0x2b65840323e8, locked=1) at storage/innobase/row/row0merge.cc:2880
#8 0x00000000012f41ea in dict_table_remove_from_cache_low (table=0x2b65840323e8, lru_evict=1) at storage/innobase/dict/dict0dict.cc:2109
#9 0x00000000012efbdd in dict_make_room_in_cache (max_tables=400, pct_check=100) at storage/innobase/dict/dict0dict.cc:1446
#10 0x0000000001197cad in srv_master_evict_from_table_cache (pct_check=100) at storage/innobase/srv/srv0srv.cc:2012
#11 0x00000000011988ff in srv_master_do_idle_tasks () at storage/innobase/srv/srv0srv.cc:2207
#12 0x000000000119930f in srv_master_thread (arg=0x0) at storage/innobase/srv/srv0srv.cc:2355
#13 0x00002b65583f5851 in start_thread (arg=0x2b6560895700) at pthread_create.c:301
#14 0x00002b65596d967d in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:115
`

从core来看，现象也比较明确:

1. InnoDB master thread 正在淘汰 InnoDB 表数据对象 dict_table_t
2. 淘汰的 dict_table_t 对象 table->drop_abort= true，所以需要删除未完成的index
3. 当在row_merge_drop_indexes() 函数中删除索引时， 发现 table->indexes= 0, 随后就crash了

由于是master thread 后台线程触发的crash，所以并不能知道用户现场做了什么操作，以及什么时候做了什么操作而对此产生了影响。

所以，只能根据当前 core 文件中的对象 dict_table_t 的属性进行排查，来查找线索。

## InnoDB背景

#### 1. Master Thread
InnoDB有一个常驻后台 master 线程，主要做以下工作：

1. 前台用户线程lazy drop 的 table，master thread负责清理
2. merge insert buffer
3. 淘汰dict table cache
4. flush log buffer
5. make checkpoint

其中，evict table cache 的过程，会根据 server 层的一个变量 table_definition_cache 来进行淘汰，

因为 server 层会根据这个变量的设置来缓存从FRM文件中得到的数据字典定义 即table_share 对象，所以引擎层缓存超过这个设置的意义也不大。

Master线程会根据 LRU 链表即 dict_sys->table_LRU 进行淘汰，但淘汰的过程，需要保证 dict_table_t 对象不能被 handler 引用，也就是当前没有 statement 语句在操作这个表，在 dict_table_t 中，使用 table->n_ref_count 来表示有多少个handler对象在引用。

#### 2. dict_table_t的生命周期

**1. 装载**
当操纵这个表的时候，InnoDB 的 handler 对象需要引用这个 dict_table_t 对象，首先会在 dict_sys->table_hash 进行hash查找：

1. 如果存在，说明已经存在 dictionary cache 中，
2. 如果不存在，需要读取InnoDB的数据字典SYS_TABLES, SYS_INDEXES, SYS_COLUMNS等来装载dict_table_t对象

**2. 引用**
当 statement 执行的时候，会先创建 handler，然后 handler 会引用 dict_table_t 对象对象，即增加 table->n_ref_count++，因为增加了引用，会调整 dict_sys->table_LRU 的位置，保持热度。
当语句结束的时候，如果 handler close 的话，会解除 dict_table_t 对象的引用，即递减 table->n_ref_count--。

**3. 缓存**
因为 server 层存在 table open cache，受 table_open_cache 参数设置影响，所以，当 statement 结束的时候，并不会立即 close opened table，相应的 InnoDB 的 handler 也不会立即关闭，这样就保持了 table->n_ref_count 引用数。

**4. 淘汰**
Master thread 每一秒钟都会轮询 dict_sys->table_LRU, 当 table->n_ref_count == 0, 进行淘汰dict_table_t 对象， 保留的数量受参数table_definition_cache控制。

#### 3. table->drop_aborted

按照 InnoDB online DDL 的定义，在 DDL 的过程中，如果任务失败，会把 table->drop_aborted 设置成 true，随后，会回滚掉当前的操作，因为是online操作，在中间时刻不阻塞 DML， 所以这里会产生两种情况：

1. 如果当前没有 statement 操作这个表，那当前在回滚的时候，就把这个 DDL 给直接回滚掉了
2. 如果当前有 statement 在操作这个表，那就会把 table->drop_aborted 设置成TRUE，进行 lazy drop 回滚。

根据代码的路径，lazy drop的会在以下场景发生：

1. dict_table_close()
 也就是当最后一个 statement 引用 dict_table_t 使用完了之后，即 table->n_ref_count == 0 时，这个线程负责清理掉未完成的 DDL
2. 下一个 DDL
也就是当下一个 DDL 操作的时候，如果发现 table->drop_aborted 为 true，那么也会负责清理这个未完成的 DDL

## 复现过程

从上面的 InnoDB 背景介绍来看，我们已经 cover 了这个 crash 相关的概念和内容，下面我们就来看复现过程：

**1. 从core文件看 table->drop_aborted= true**

所以我们断定一定存在失败的 DDL 语句，随后通过审计日志，我们发现：

` alter table t add unique key(col1);
 ERROR 1062 (23000): Duplicate entry '2' for key 'col1'
`
**2.回滚**
因为用户操作的时候，没有回滚掉这个 online add unique key 操作，所以我们断定在 alter 操作的时候，同时有 DML 语句在。

根据这两点我们构思了如下的case：

**环境准备：**

`create table t(id int primary key, col1 int) engine=innodb;
insert into t values(1, 2);
insert into t values(2, 2);
`

Session 1:

` // 需要再执行rollback之前，session 2进行insert，递增table->n_ref_count
 alter table t add unique key(col1);
`
Session 2:

`// 需要等待alter操作完成之后，insert才去完成，继而递减table->n_ref_count
insert into t values(3, 2);
`
Session 3:

`// close所有打开的表，使 table->n_ref_count == 0;
flush tables
// 创建1000张表，这样t表就会率先淘汰出去
let $loop=1000;
while($loop)
{
 eval create table t_$loop(id int)engine=innodb;
 dec $loop;
}
`

为了复现这个case，我们添加了两个sleep函数在代码中，参考如下：

`diff --git a/storage/innobase/handler/ha_innodb.cc b/storage/innobase/handler/ha_innodb.cc
index 41c767a..bfd7102 100644
--- a/storage/innobase/handler/ha_innodb.cc
+++ b/storage/innobase/handler/ha_innodb.cc
@@ -6779,6 +6779,7 @@ no_commit:

 build_template(true);
 }
+ os_thread_sleep(5000000);

 innobase_srv_conc_enter_innodb(prebuilt->trx);

diff --git a/storage/innobase/handler/handler0alter.cc b/storage/innobase/handler/handler0alter.cc
index e772208..dea7696 100644
--- a/storage/innobase/handler/handler0alter.cc
+++ b/storage/innobase/handler/handler0alter.cc
@@ -4138,6 +4138,7 @@ rollback_inplace_alter_table(
 (almost) nothing has been or needs to be done. */
 goto func_exit;
 }
+ os_thread_sleep(2000000);

 row_mysql_lock_data_dictionary(ctx->trx);
`

这样我们就复现出来这个 crash， 同样，我们在 MySQL 5.7.18，以及还没有 release 的8.0版本上发现了存在相同的问题。

## 问题原因和修复方法

**1. 问题原因**
 从代码的设计上，table->drop_aborted设置成TRUE，会在两种场景下进行lazy drop，即上面提到的：

1. dict_table_close 即 dict_table_t 对象 n_ref_count 引用数降成0
2. 下一个 DDL 的时候

而这个 lazy drop 却是在 master thread 要淘汰 dict_table_t 的时候。 因为淘汰的条件需要 n_ref_count == 0, 所以一定发生过dict_table_close() 了。

那问题的原因就明确了： 在 dict_table_close 把 n_ref_count 降成0的时候，没有完成 lazy drop 回滚。

**2. 修复方法**
知道了问题的原因，修复方法很简单，我们发现 dict_table_close() 函数存在一些逻辑错误， 

我们将会在Aliyun RDS版本和我们的开源版本AliSQL上进行修复，敬请关注。

同时也可以关注，我们提交给官方和MariaDB的进度：
https://bugs.mysql.com/bug.php?id=86607
https://jira.mariadb.org/browse/MDEV-13051

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)