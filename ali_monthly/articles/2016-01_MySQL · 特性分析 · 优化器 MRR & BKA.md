# MySQL · 特性分析 · 优化器 MRR & BKA

**Date:** 2016/01
**Source:** http://mysql.taobao.org/monthly/2016/01/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 01
 ](/monthly/2016/01)

 * 当期文章

 MySQL · 引擎特性 · InnoDB 事务锁系统简介
* GPDB   · 特性分析· GreenPlum Primary/Mirror 同步机制
* MySQL · 专家投稿 · MySQL5.7 的 JSON 实现
* MySQL · 特性分析 · 优化器 MRR & BKA
* MySQL · 答疑解惑 · 物理备份死锁分析
* MySQL · TokuDB · Cachetable 的工作线程和线程池
* MySQL · 特性分析 · drop table的优化
* MySQL · 答疑解惑 · GTID不一致分析
* PgSQL · 特性分析 · Plan Hint
* MariaDB · 社区动态 · MariaDB on Power8 (下)

 ## MySQL · 特性分析 · 优化器 MRR & BKA 
 Author: 沽月 

 上一篇文章咱们对 ICP 进行了一次全面的分析，本篇文章小编继续为大家分析优化器的另外两个选项: MRR & batched_key_access(BKA) ，分析一下他们的作用、原理、相互关系、源码实现以及使用范围。

## 什么是 MRR

MRR 的全称是 Multi-Range Read Optimization，是优化器将随机 IO 转化为顺序 IO 以降低查询过程中 IO 开销的一种手段，咱们对比一下 mrr=on & mrr=off 时的执行计划：

其中表结构如下：

`mysql> show create table t1\G
*************************** 1. row ***************************
 Table: t1
Create Table: CREATE TABLE `t1` (
 `id` int(11) NOT NULL AUTO_INCREMENT,
 `a` int(11) DEFAULT NULL,
 `b` int(11) DEFAULT NULL,
 `c` int(11) DEFAULT NULL,
 PRIMARY KEY (`id`),
 KEY `mrrx` (`a`,`b`),
 KEY `xx` (`c`)
) ENGINE=MyISAM AUTO_INCREMENT=11 DEFAULT CHARSET=latin1
1 row in set (0.00 sec)
`

操作如下：

`mysql> set optimizer_switch='mrr=off';
Query OK, 0 rows affected (0.00 sec)

mysql> explain select * from test.t1 where (a between 1 and 10) and (c between 9 and 10) ;
+----+-------------+-------+-------+---------------+------+---------+------+------+------------------------------------+
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
+----+-------------+-------+-------+---------------+------+---------+------+------+------------------------------------+
| 1 | SIMPLE | t1 | range | mrrx,xx | xx | 5 | NULL | 2 | Using index condition; Using where |
+----+-------------+-------+-------+---------------+------+---------+------+------+------------------------------------+
1 row in set (0.00 sec)
`

当把 MRR 关掉的情况下，执行计划使用的是索引 xx(c)，即从索引 xx 上读取一条数据后回表，取回该主键的完整数据，当数据较多且比较分散的情况下会有比较多的随机 IO, 导致性能低下，我们将 MRR 打开，执行以下操作：

`mysql> set optimizer_switch='mrr=on';
Query OK, 0 rows affected (0.00 sec)

mysql> explain select * from test.t1 where (a between 1 and 10) and (c between 9 and 10) ;
+----+-------------+-------+-------+---------------+------+---------+------+------+-----------------------------------------------+
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
+----+-------------+-------+-------+---------------+------+---------+------+------+-----------------------------------------------+
| 1 | SIMPLE | t1 | range | mrrx,xx | xx | 5 | NULL | 2 | Using index condition; Using where; Using MRR |
+----+-------------+-------+-------+---------------+------+---------+------+------+-----------------------------------------------+
1 row in set (0.00 sec)
`

可以看到 extra 的输出中多了 “Using MRR” 信息，即使用了 MRR Optimization IO 层面进行了优化，减少 IO 方面的开销，更详细的说明可以参考[这里](http://dev.mysql.com/doc/refman/5.6/en/mrr-optimization.html)。

## MRR 原理

在不使用 MRR 时，优化器需要根据二级索引返回的记录来进行“回表”，这个过程一般会有较多的随机 IO, 使用 MRR 时，SQL 语句的执行过程是这样的：

* 优化器将二级索引查询到的记录放到一块缓冲区中；
* 如果二级索引扫描到文件的末尾或者缓冲区已满，则使用快速排序对缓冲区中的内容按照主键进行排序；
* 用户线程调用 MRR 接口取 cluster index，然后根据cluster index 取行数据；
* 当根据缓冲区中的 cluster index 取完数据，则继续调用过程 2) 3)，直至扫描结束；

通过上述过程，优化器将二级索引随机的 IO 进行排序，转化为主键的有序排列，从而实现了随机 IO 到顺序 IO 的转化，提升性能。

## MRR 源码分析

首先，咱们来看一下 mrr 相对应的内存结构：

`class DsMrr_impl
{
 ...
 handler *h;
 TABLE *table; /* Always equal to h->table */
private:
 /* Secondary handler object. It is used for scanning the index */
 handler *h2;

 /* Buffer to store rowids, or (rowid, range_id) pairs */
 uchar *rowids_buf;
 uchar *rowids_buf_cur; /* Current position when reading/writing */
 uchar *rowids_buf_last; /* When reading: end of used buffer space */
 uchar *rowids_buf_end; /* End of the buffer */

 bool dsmrr_eof; /* TRUE <=> We have reached EOF when reading index tuples */

 int dsmrr_init(handler *h, RANGE_SEQ_IF *seq_funcs, void *seq_init_param,
 uint n_ranges, uint mode, HANDLER_BUFFER *buf);
 ….
 int dsmrr_fill_buffer();
 int dsmrr_next(char **range_info);
 bool get_disk_sweep_mrr_cost(uint keynr, ha_rows rows, uint flags, uint *buffer_size, Cost_estimate *cost);
 ….
}
`

简单说明：h2 指的是 MRR 使用的 second index 或主键索引, h 是指利用 h2 返回的主建来查询的句柄，rowids_buf 是 MRR 执行过程中存储有序主键的缓存区，大小由 MySQL 的变量 `read_rnd_buffer_size` 设置，下面我们结合程序的执行过程来看一下源码。

1. MRR 中有序主建的收集过程
优化器对查询语句的条件进行分析并选择合适的二级索引，并对二级索引的条件进行筛选拼装成 DYNAMIC_ARRAY ranges，在执行的时候将 ranges 传入初始化函数 `ha_myisam::multi_range_read_init` ，继而会调用 `dsmrr_fill_buffer` 函数，在`dsmrr_fill_buffer`中会使用二级索引的句柄查找符合 ranges 的数据并添加至 rowids_buf 中，在扫描结束或缓冲区满的时候会对 rowids_buf 进行快速排序，详细过程可以参考函数：`dsmrr_fill_buffer`，其调用堆栈如下：

 ` #0 DsMrr_impl::dsmrr_fill_buffer (this=0x2aab0000cf00)
 #1 0x00000000006e49dd in DsMrr_impl::dsmrr_init(...)
 #2 0x00000000017d35e4 in ha_myisam::multi_range_read_init(...)
 #3 0x0000000000d134c6 in QUICK_RANGE_SELECT::reset (this=0x2aab00014070)
 #4 0x00000000009a266f in join_init_read_record (tab=0x2aab0000f5b8)
 #5 0x000000000099d6d4 in sub_select
 #6 0x000000000099c914 in do_select (join=0x2aab000064b0)
 #7 0x00000000009982f8 in JOIN::exec (this=0x2aab000064b0)
 #8 0x0000000000a5bd7c in mysql_execute_select
 ........
`
2. MRR 中主建缓冲区的使用过程

 物理执行阶段，调用 `ha_myisam::multi_range_read_next`，在使用 MRR 的情况下会从过程1）中收集的有序主建的缓冲区取主建，然后再调用引擎层的 rnd_pos 直接找到数据，其中使用 mrr 的调用堆栈如下：

 ` #0 DsMrr_impl::dsmrr_next (this=0x2aab0000cf00, range_info=0x2aaafc03de70)
 #1 0x00000000017d3634 in ha_myisam::multi_range_read_next (this=0x2aab0000ca40, range_info=0x2aaafc03de70)
 #2 0x0000000000d138cc in QUICK_RANGE_SELECT::get_next (this=0x2aab00014070)
 #3 0x0000000000d46908 in rr_quick (info=0x2aab0000f648)
 #4 0x00000000009a2791 in join_init_read_record (tab=0x2aab0000f5b8)
 #5 0x000000000099d6d4 in sub_select (join=0x2aab000064b0, join_tab=0x2aab0000f5b8, end_of_records=false)
 #6 0x000000000099c914 in do_select (join=0x2aab000064b0)
` 

 二缓索引（h2）& 主建索引（h) 的协同是通过`rowids_buf_cur`来进行的。最初的初始化过程中，h2 会首先将数据填冲到 rowids_buf 中，如果发现缓冲区中的数据已经取完，则会继续调用 `dsmrr_fill_buffer` 往 rowids_buf 填主键并进行排序，如此反复，直至 h2 扫描至文件末尾，详情可以参考函数 `DsMrr_impl::dsmrr_next`。

通过上面的分析，是不是感觉 MRR 有点像二级索引与主键的 join 操作，那就是有点和 BKA 有些类似的概念了，咱们下面看一下 BKA 是如何实现的。

## BKA 原理

BKA 是指在表连接的过程中为了提升 join 性能而使用的一种 join buffer，其作用是在读取被 join 表的记录的时候使用顺序 IO，BKA 被使用的标识是执行计划的 extra 信息中会有 “Batched Key Access” 信息， 我们首先看一个例子：

`DROP TABLE t1, t2;
CREATE TABLE t1 (a int PRIMARY KEY, b int);
CREATE TABLE t2 (a int PRIMARY KEY, b int);
INSERT INTO t1 VALUES (1,2), (2,1), (3,2), (4,3), (5,6), (6,5), (7,8), (8,7), (9,10);
INSERT INTO t2 VALUES (3,0), (4,1), (6,4), (7,5);

mysql> set optimizer_switch="mrr=on,mrr_cost_based=off,batched_key_access=on";
mysql> explain SELECT * FROM t1 LEFT JOIN t2 ON t1.a = t2.a WHERE t2.b <= t1.a AND t1.a <= t1.b;
+----+-------------+-------+--------+---------------+---------+---------+-----------+------+-----------------------------------------------------+
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
+----+-------------+-------+--------+---------------+---------+---------+-----------+------+-----------------------------------------------------+
| 1 | SIMPLE | t2 | ALL | PRIMARY | NULL | NULL | NULL | 4 | Using where |
| 1 | SIMPLE | t1 | eq_ref | PRIMARY | PRIMARY | 4 | test.t2.a | 1 | Using where; Using join buffer (Batched Key Access) |
+----+-------------+-------+--------+---------------+---------+---------+-----------+------+-----------------------------------------------------+
2 rows in set (0.00 sec)
`

从以上的例子中我们可以看到，在读取表 t1 的时候使用了带 BKA 功能的 join buffer, 其中 BKA & join buffer 的关系与实现我们放在后面详解。

## BKA & MRR 之间的关系

使用 BKA 的表的 JOIN 过程如下：

1. 连接表将满足条件的记录放入JOIN_CACHE，并将两表连接的字段放入一个 DYNAMIC_ARRAY ranges 中，此过程类似于 MRR 操作的过程，且在内存中使用的是同样的结构体 DsMrr_impl；
2. 在进行表的过接过程中，会将 ranges 相关的信息传入 `DsMrr_impl::dsmrr_fill_buffer`，并进行被连接表主建的查找及排序等操作操作，这个过程比较复杂，包括需要判断使用的 key、key 是主建时的特殊操作等；
3. `JOIN_CACHE_BKA::join_matching_records` 会调用过程2中产生的有序主建，然后顺序读取数据并进入下一步的操作（`evaluate_join_record` 等）；
4. 当缓冲区的数据被读完后，会重复进行过程2，3, 直到记录被读取完。

由上面的分析可以看出，BKA将有序主建投递到存储引擎是通过 MRR 的接口的调用来实现的（`DsMrr_impl::dsmrr_next`），所以BKA 依赖 MRR，如果要使用BKA, MRR 是需要打开的，另外 `batched_key_access` 是默认关闭的，如果要使用，需要打开此选项。
BKA 的详细说明可参考[这里](https://dev.mysql.com/doc/refman/5.6/en/bnl-bka-optimization.html)。

## BKA 源码实现

表之间的连接操作是通过 JOIN_CACHE 来做的，5.6 目前实现了 BNL, BKA (JOIN_CACHE_BKA & JOIN_CACHE_BKA_UNIQUE) 两种表连接的优化方式，其中 BKA 就是其中减少随机 IO 的一种方式，BKA内存中对应的结构是 JOIN_CACHE_BKA，咱们首先看一下多表 JOIN 之间的过程；

1. 优化器生成的执行计划是由一个 JOIN_TAB 的左支树组成，每个 JOIN_TAB 包含了相关的表、使用的索引、语句中包含的条件等信息；
2. 进入物理执行计划后，会对每一个表进行读数据，然后进入 `evaluate_join_record`, 当发现满足条件的记录时，则会将该记录添加到下一个JOIN_TAB 中的JOIN_CACHE 中，其堆栈如下：

 ` #0 JOIN_CACHE::put_record (this=0x2aab00019d20)
 #1 0x000000000099d29c in sub_select_op (join=0x2aab00016268, join_tab=0x2aab00018ed8, end_of_records=false)
 #2 0x000000000099ee1c in evaluate_join_record (join=0x2aab00016268, join_tab=0x2aab00018bd8)
 #3 0x000000000099d984 in sub_select (join=0x2aab00016268, join_tab=0x2aab00018bd8, end_of_records=false)
 #4 0x000000000099c914 in do_select (join=0x2aab00016268)
 #5 0x00000000009982f8 in JOIN::exec (this=0x2aab00016268)
 #6 0x0000000000a5bd7c in mysql_execute_select (thd=0x314d690, select_lex=0x31503a8, free_join=true)
`
3. 当缓冲区满或者读到文件的末尾时，会调用下一个JOIN_TAB 中 `JOIN_CACHE::join_records` 方法（BKA 使用时 JOIN_CACHE 为 JOIN_CACHE_BKA），然后会进入 MRR 的相关逻辑，其完整的堆栈为：

 ` #0 DsMrr_impl::dsmrr_fill_buffer (this=0x2aab000128e0)
 #1 0x00000000006e49dd in DsMrr_impl::dsmrr_init
 #2 0x00000000017d35e4 in ha_myisam::multi_range_read_init
 #3 0x0000000000d838aa in JOIN_CACHE_BKA::init_join_matching_records (this=0x2aab00019d20, seq_funcs=0x2aaafc03dd80, ranges=4)
 #4 0x0000000000d8335c in JOIN_CACHE_BKA::join_matching_records (this=0x2aab00019d20, skip_last=false)
 #5 0x0000000000d812e8 in JOIN_CACHE::join_records (this=0x2aab00019d20, skip_last=false)
 #6 0x0000000000d86ed3 in JOIN_CACHE::end_send (this=0x2aab00019d20)
 #7 0x000000000099d0d1 in sub_select_op (join=0x2aab00016268, join_tab=0x2aab00018ed8, end_of_records=true)
 #8 0x000000000099d3c4 in sub_select (join=0x2aab00016268, join_tab=0x2aab00018bd8, end_of_records=true) at
 #9 0x000000000099c97d in do_select (join=0x2aab00016268)
 #10 0x00000000009982f8 in JOIN::exec (this=0x2aab00016268)
 #11 0x0000000000a5bd7c in mysql_execute_select
`
4. `dsmrr_fill_buffer` 的过程相对复杂，需要首先取出两表相连接的字段的索引，如果没有索引，则会使用主建并直接读取，如果使用了索引，则需要从上一个JOIN_TAB中将索引的信息读出来并从 join_cache 的 buffer 中取出该索引的数据，然后再进行回表，查找主建、排序等操作，其堆栈如下：

 ` #0 JOIN_CACHE_BKA::get_next_key (this=0x2aab00019d20, key=0x2aab0001e178)
 #1 0x0000000000d82f83 in bka_range_seq_next (rseq=0x2aab00019d20, range=0x2aab0001e178)
 #2 0x00000000006e3cac in handler::multi_range_read_next (this=0x2aab0001e020, range_info=0x2aaafc03dc10)
 #3 0x00000000006e5466 in DsMrr_impl::dsmrr_fill_buffer (this=0x2aab000128e0)
 #4 0x00000000006e49dd in DsMrr_impl::dsmrr_init (…)
 #5 0x00000000017d35e4 in ha_myisam::multi_range_read_init (…)
 #6 0x0000000000d838aa in JOIN_CACHE_BKA::init_join_matching_records (this=0x2aab00019d20, seq_funcs=0x2aaafc03dd80, ranges=4)
`

此过程只是两个表的使用 BKA 时的过程，当是多表时，过程将更为复杂。

## 小结

本篇文章中我们详细的介绍了 MRR、BKA 以及 MRR & BKA 之间的关系等内容，测试用例都是在mrr_cost_based=OFF 的情况下进行的，因为SQL 语句是否使用 MRR 优化依赖于其代价的大小，优化器的代价计算是一个比较复杂的过程，无论是 MRR 还是 BKA 都只是优化器进行优化的方法，当其发现优化后的代价过高时就会不使用该项优化，因此在使用 MRR 相关的优化时，尽量设置 mrr_cost_based=ON，毕竟大多数情况下优化器是对的。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)