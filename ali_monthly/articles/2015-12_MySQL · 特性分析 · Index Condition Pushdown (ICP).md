# MySQL · 特性分析 · Index Condition Pushdown (ICP)

**Date:** 2015/12
**Source:** http://mysql.taobao.org/monthly/2015/12/08/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 12
 ](/monthly/2015/12)

 * 当期文章

 MySQL · 引擎特性 · InnoDB 事务子系统介绍
* PgSQL · 特性介绍 · 全文搜索介绍
* MongoDB · 捉虫动态 · Kill Hang问题排查记录
* MySQL · 参数优化 ·RDS MySQL参数调优最佳实践
* PgSQL · 特性分析 · 备库激活过程分析
* MySQL · TokuDB · 让Hot Backup更完美
* PgSQL · 答疑解惑 · 表膨胀
* MySQL · 特性分析 · Index Condition Pushdown (ICP)
* MariaDB · 社区动态 · MariaDB on Power8
* MySQL · 特性分析 · 企业版特性一览

 ## MySQL · 特性分析 · Index Condition Pushdown (ICP) 
 Author: 沽月 

 ## 前言

[上一篇文章](http://mysql.taobao.org/monthly/2015/11/07/) 提过，我们在之后的文章中会从 optimizer 的选项出发，系统的介绍 optimizer 的各个变量，包括变量的原理、作用以及源码实现等，然后再进一步的介绍优化器的工作过程（SQL 语句扁平化处理、索引选择、代价计算、多表连接顺序选择以及物理执行等内容），本期我们先看一下众所周知的 ICP，官方文档请参考[这里](https://dev.mysql.com/doc/refman/5.6/en/condition-pushdown-optimization.html)。

## ICP 测试

首先，咱们来看一下打开 ICP 与关闭 ICP 之间的性能区别，以下是测试过程：

准备数据：

`create table icp(id int, age int, name varchar(30), memo varchar(600)) engine=innodb;
alter table icp add index aind(age, name, memo);
--let $i= 100000
while ($i)
{
 --eval insert into icp values($i, 1, 'a$i', repeat('a$i', 100))
 --dec $i
}
`

PS: MySQL 有一个叫profile的东东，可以用来监视 SQL 语句在各个阶段的执行情况，咱们可以使用这个工具来观察 SQL 语句在各个阶段的运行情况，关于 profile 的详细说明可以参考[官方文档](http://dev.mysql.com/doc/refman/5.7/en/show-profile.html)。

打开 ICP 的性能测试：

`set profiling=on;
set optimizer_switch="index_condition_pushdown=on”; （default enabled）
select * from icp where age = 1 and memo like '%9999%';
mysql> show profile cpu,block io for query 7;
+----------------------+-----------+-----------+------------+--------------+---------------+
| Status | Duration | CPU_user | CPU_system | Block_ops_in | Block_ops_out |
+----------------------+-----------+-----------+------------+--------------+---------------+
| executing | 0.000009 | 0.000000 | 0.000000 | 0 | 0 |
| Sending data | 3.225383 | 3.507467 | 0.037994 | 0 | 0 |
+----------------------+-----------+-----------+------------+--------------+---------------+
mysql> show session status like '%handler%';show session status like '%handler%';
+----------------------------+--------+
| Handler_read_next | 19 |
| Handler_read_rnd_next | 30 |
+----------------------------+--------+
18 rows in set (0.00 sec)
`

关闭 ICP 的性能测试：

`mysql> set optimizer_switch="index_condition_pushdown=off”;
mysql> select * from icp where age = 1 and memo like '%9999%';
mysql> show profile cpu, block io for query 20;
+----------------------+----------+----------+------------+--------------+---------------+
| Status | Duration | CPU_user | CPU_system | Block_ops_in | Block_ops_out |
+----------------------+----------+----------+------------+--------------+---------------+
| Sending data | 15.327345 | 17.443348 | 0.165975 | 0 | 0 |
+----------------------+----------+----------+------------+--------------+---------------+
15 rows in set, 1 warning (0.00 sec)
mysql> show session status like '%handler%';
+----------------------------+--------+
| Variable_name | Value |
+----------------------------+--------+
| Handler_read_next | 100019 |
| Handler_read_rnd_next | 47 |
+----------------------------+--------+
18 rows in set (0.01 sec)
`

测试结论：由以上测试情况可以看到，在二级索引是复合索引且前面的条件过滤性较低的情况下，打开 ICP 可以有效的降低 server 层和 engine 层之间交互的次数，从而有效的降低在运行时间。

## ICP 原理

5.6 之前，在 SQL 语句的执行过程中，server 层通过 engine 的 api 获取数据，然后再进行 where_cond 的判断（具体判断逻辑在: `evaluate_join_record`），每一条数据都需要从engine层返回server层做判断。我们回顾一下上面把 ICP 关掉的测试，可以看到 `Handler_read_next` 的值陡增，其原因是第 1 个字段区分度不高，且 memo 字段无法使用索引，造成了类似 index 扫描的的情况，性能较低。

5.6 之后，在利用索引扫描的过程中，如果发现 where_cond 中含有这个 index 相关的条件，则将此条件记录在 handler 接口中，在索引扫描的过程中，只有满足索引与handler接口的条件时，才会返回到 server 层做进一步的处理，在前缀索引区分度不够，其它字段区分度高的情况下可以有效的减少 server & engine之间的开销，提升查询性能。

## ICP 源码实现

我们在上小节提到，index condition down 所用的条件是记在handler接口中的，咱们分析一下“记录”的过程是如何实现的。

首先，优化器计算代价后会生成一个 JOIN_TAB 的左支树，每一个 JOIN_TAB 包含相关表的指针、表的读取方式、访问表所包含的索引等信息，优化器会在` make_join_readinfo` 中对JOIN_TAB中表的访问方式进行相应的修正，并进一步将 where cond 中和索引相关的条件记录到 table 的句柄中，堆栈如下：

`#0 make_cond_for_index (cond=0x2b69680179e8, table=0x2b6968012100, keyno=0, other_tbls_ok=true)
#1 in push_index_cond (tab=0x2b696802aa48, keyno=0, other_tbls_ok=true, trace_obj=0x2b696413ec30)
#2 in make_join_readinfo (join=0x2b6968017db0, options=0, no_jbuf_after=4294967295)
#3 in JOIN::optimize (this=0x2b6968017db0)
#4 in mysql_execute_select (thd=0x3176760, select_lex=0x3179470, free_join=true)
`

其次， `make_cond_for_index` 是一个递归的过程，对 where_cond中的每一个条件进行判断，对满足条件的 cond 重新组合成一个新的cond，最后将新的 cond 挂在table->file 下面（table->file 指的是操作物理表的接口函数，此变量为thd下私有的，不共享，共享的是tab->table->s），详细参考`make_cond_for_index` 的详细实现，设置的堆栈如下：

`#0 ha_innobase::idx_cond_push (this=0x2b696800e810, keyno=0, idx_cond=0x2b69680179e8)
#1 0x0000000000a60a55 in push_index_cond (tab=0x2b696802aa48, keyno=0, other_tbls_ok=true, trace_obj=0x2b696413ec30)
#2 0x0000000000a6362f in make_join_readinfo (join=0x2b6968017db0, options=0, no_jbuf_after=4294967295)
#3 0x0000000000d9b8bd in JOIN::optimize (this=0x2b6968017db0
#4 0x0000000000a5b9ae in mysql_execute_select (thd=0x3176760, select_lex=0x3179470, free_join=true)
`

再次，server 层根据生成的 JOIN_TAB 读取engine层的内容，在engine读取的时候，会进行`index_condition_pushdown`的调用，即 ICP 的调用，堆栈如下：

`#0 Item_func_like::val_int (this=0x2b6978005a28)
#1 0x0000000001187b66 in innobase_index_cond (file=0x2b696800e810)
#2 0x0000000001393566 in row_search_idx_cond_check (mysql_rec=0x2b69680129f0 <incomplete sequence \361>, prebuilt=0x2b69680130f8, rec=0x2b692b56e4cf "\200", offsets=0x2b697008d450)
#3 0x0000000001397e2b in row_search_for_mysql (buf=0x2b69680129f0 <incomplete sequence \361>, mode=2, prebuilt=0x2b69680130f8, match_mode=1, direction=0)
#4 0x00000000011696b9 in ha_innobase::index_read (this=0x2b696800e810, buf=0x2b69680129f0 <incomplete sequence \361>, key_ptr=0x2b697800a660 "", key_len=5, find_flag=HA_READ_KEY_EXACT)
#5 0x00000000006ecc58 in handler::index_read_map (this=0x2b696800e810, buf=0x2b69680129f0 <incomplete sequence \361>, key=0x2b697800a660 "", keypart_map=1, find_flag=HA_READ_KEY_EXACT)
#6 0x00000000006d6bb4 in handler::ha_index_read_map (this=0x2b696800e810, buf=0x2b69680129f0 <incomplete sequence \361>, key=0x2b697800a660 "", keypart_map=1, find_flag=HA_READ_KEY_EXACT)
#7 0x00000000009a1870 in join_read_always_key (tab=0x2b697800a1b8)
#8 0x000000000099d480 in sub_select (join=0x2b6978005df0, join_tab=0x2b697800a1b8, end_of_records=false)
#9 0x000000000099c6c0 in do_select (join=0x2b6978005df0)
#10 0x00000000009980a4 in JOIN::exec (this=0x2b6978005df0)
#11 0x0000000000a5bac0 in mysql_execute_select (thd=0x32801a0, select_lex=0x3282eb0, free_join=true)
`

可见在 ICP 的判断是调用相关item的函数的，虽然同是调用 server 层的函数，但是没有 ICP 的调用需要根据主建找到记录，然后再匹配，而有了 ICP 可以省略一次主键查找数据的过程，进而提升效率。

## ICP 使用限制及问题

* 只支持 select 语句；
* 5.6 中只支持 MyISAM 与 InnoDB 引擎;
* ICP的优化策略可用于range、ref、eq_ref、ref_or_null 类型的访问数据方法；
* 不支持主建索引的 ICP；
* 当 SQL 使用覆盖索引时但只检索部分数据时，ICP 无法使用，详细的分析可以参考 bug#68554 中 Olav Sandstå的分析，代码实现部分可以参考 `make_join_readinfo`；
* 在查询的时候即使正确的使用索引的前Ｎ个字段（即遵循前缀索引的原则），还是会用到 ICP，无故的多了 ICP 相关的判断，这应该是一个退化的问题，例：

 ` mysql> explain select * from icp where age=1 and name = 'a1';
 +----+-------------+-------+------+---------------+------+---------+-------------+------+-----------------------+
 | id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
 +----+-------------+-------+------+---------------+------+---------+-------------+------+-----------------------+
 | 1 | SIMPLE | icp | ref | aind | aind | 38 | const,const | 1 | Using index condition |
 +----+-------------+-------+------+---------------+------+---------+-------------+------+-----------------------+
 1 row in set (3.26 sec)
`

PS: engine condition pushdown 是 NDB 使用的，其它引擎不支持。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)