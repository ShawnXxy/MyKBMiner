# MySQL · 答疑解惑 · MySQL 优化器 range 的代价计算

**Date:** 2015/11
**Source:** http://mysql.taobao.org/monthly/2015/11/07/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 11
 ](/monthly/2015/11)

 * 当期文章

 MySQL · 社区见闻 · OOW 2015 总结 MySQL 篇
* MySQL · 特性分析 · Statement Digest
* PgSQL · 答疑解惑 · PostgreSQL 用户组权限管理
* MySQL · 特性分析 · MDL 实现分析
* PgSQL · 特性分析 · full page write 机制
* MySQL · 捉虫动态 · MySQL 外键异常分析
* MySQL · 答疑解惑 · MySQL 优化器 range 的代价计算
* MySQL · 捉虫动态 · ORDER/GROUP BY 导致 mysqld crash
* MySQL · TokuDB · TokuDB 中的行锁
* MySQL · 捉虫动态 · order by limit 造成优化器选择索引错误

 ## MySQL · 答疑解惑 · MySQL 优化器 range 的代价计算 
 Author: 沽月 

 本文我们从一个索引选择的问题出发，来研究一下 MySQL 中 range 代价的计算过程，进而分析这种计算过程中存在的问题。

## 问题现象

**第一种情况：situation_unique_key_id**

`mysql> show create table cpa_order\G
*************************** 1. row ***************************
 Table: cpa_order
Create Table: CREATE TABLE `cpa_order` (
 `cpa_order_id` bigint(20) unsigned NOT NULL,
 ...
 `settle_date` date DEFAULT NULL COMMENT,
 `id` bigint(20) NOT NULL,
 PRIMARY KEY (`cpa_order_id`),
 UNIQUE KEY `id` (`id`),
 KEY `cpao_settle_date_id` (`settle_date`,`id`),
) ENGINE=InnoDB DEFAULT CHARSET=gbk
1 row in set (0.00 sec)

mysql> explain select * from cpa_order where settle_date='2015-11-05' and id > 15 \G
*************************** 1. row ***************************
 id: 1
 select_type: SIMPLE
 table: cpa_order
 type: ref
possible_keys: id,cpao_settle_date_id
 key: cpao_settle_date_id
 key_len: 4
 ref: const
 rows: 7
 Extra: Using index condition
1 row in set (0.00 sec)
`

SQL 语句执行过程可以看出，当 id 为 unique key 的时候，key_len= 4, 不难发现联合索引只使用了字段 cpao_settle_date_id ，而 id 并没有使用；

**第二种情况：situation_without_key_id**

`mysql> alter table cpa_order drop index id;
Query OK, 0 rows affected (0.01 sec)
Records: 0 Duplicates: 0 Warnings: 0

mysql> explain select * from cpa_order where settle_date='2015-11-05' and id > 15 \G （我们称之为 situation_without_key_id）
*************************** 1. row ***************************
 id: 1
 select_type: SIMPLE
 table: cpa_order
 type: range
possible_keys: cpao_settle_date_id
 key: cpao_settle_date_id
 key_len: 12
 ref: NULL
 rows: 3
 Extra: Using index condition
1 row in set (0.00 sec)
`

**第三种情况: situation_plain_key_id**

`mysql> explain select * from cpa_order where settle_date='2015-11-05' and id > 15 \G （我们称之为 situation_plain_key_id）
*************************** 1. row ***************************
 id: 1
 select_type: SIMPLE
 table: cpa_order
 type: range
possible_keys: cpao_settle_date_id,id
 key: cpao_settle_date_id
 key_len: 12
 ref: NULL
 rows: 3
 Extra: Using index condition
1 row in set (0.01 sec)
`

以上的两个 SQL 语句在使用索引 cpao_settle_date_id 的时候两个字段都使用到了，因此过滤性应该更好，我们将上面的3种情况分别称之为 situation_unique_key_id，situation_without_key_id，situation_plain_key_id，以方便我们分析问题。

**为什么在 id 为 unique 的时候联合索引只使用了其中的一个字段而没有字段 id ?**

## 原因分析

MySQL 有一个很好的东东叫 optimizer trace，它提供了 MySQL 执行计划生成的各个阶段的详细信息，其中索引部分的分析更是详细，但是由于 optimizer trace 的东西比较多，我们在分析的时候只将本文相关的内容进行展开，optimizer trace 的[详细使用](https://dev.mysql.com/doc/internals/en/optimizer-tracing.html)。

打开并使用 optimizer_trace 功能，观察situation_unique_key_id 的代价生成过程的：

`mysql> set optimizer_trace="enabled=on";
Query OK, 0 rows affected (0.00 sec)

mysql> select * from cpa_order where settle_date='2015-11-05' and id > 15 \G
3 rows in set (0.00 sec)

mysql> select * from information_schema.OPTIMIZER_TRACE\G
`

在 range 代价计算后，优化器会选择一个代价较小的 index 生成一个 read_plan 缓存起来，根据下面的代价计算过程可以看到，索引在代价计算过程中虽然是相等的，但先入为主，选择的其实是 id 这个索引。

range 部分的代价计算过程：

`"range_scan_alternatives": [
 {
 "index": "id",
 "ranges": [
 "15 < id"
 ],
 "index_dives_for_eq_ranges": true,
 "rowid_ordered": false,
 "using_mrr": false,
 "index_only": false,
 "rows": 3,
 "cost": 4.61,
 "chosen": true
 },
 {
 "index": "cpao_settle_date_id",
 "ranges": [
 "2015-11-05 <= settle_date <= 2015-11-05 AND 15 < id"
 ],
 "index_dives_for_eq_ranges": true,
 "rowid_ordered": false,
 "using_mrr": false,
 "index_only": false,
 "rows": 3,
 "cost": 4.61,
 "chosen": false,
 "cause": "cost"
 }
`

表的索引选择过程，主要是 ref & range 的索引方式的选择：

`"considered_execution_plans": [
 {
 "plan_prefix": [
 ],
 "table": "`cpa_order`",
 "best_access_path": {
 "considered_access_paths": [
 {
 "access_type": "ref",
 "index": "cpao_settle_date_id",
 "rows": 7,
 "cost": 3.4,
 "chosen": true
 },
 {
 "access_type": "range",
 "rows": 3,
 "cost": 5.21,
 "chosen": false
 }
 ]
 },
 "cost_for_plan": 3.4,
 "rows_for_plan": 7,
 "chosen": true
 }
 ]
`

可以看到优化器在比较 ref & range 的代价的时候，ref 的代价更小，所以选择的是ref，到这里我们觉得选择 ref 是“合理”的，但是当我们想到联合索引的作用时，我们应该觉得这是“不正常的”，至少这不应该是最终的索引选择方式。

**观察 situation_without_key_id 的代价及生成过程，其 optimizer_trace 如下：**

range 部分的代价计算过程：

` "analyzing_range_alternatives": {
 "range_scan_alternatives": [
 {
 "index": "cpao_settle_date_id",
 "ranges": [
 "2015-11-05 <= settle_date <= 2015-11-05 AND 15 < id"
 ],
 "index_dives_for_eq_ranges": true,
 "rowid_ordered": false,
 "using_mrr": false,
 "index_only": false,
 "rows": 3,
 "cost": 4.61,
 "chosen": true
 }
 ],
 "analyzing_roworder_intersect": {
 "usable": false,
 "cause": "too_few_roworder_scans"
 }
 },
`

表的索引选择过程：

` "considered_execution_plans": [
 {
 "plan_prefix": [
 ],
 "table": "`cpa_order`",
 "best_access_path": {
 "considered_access_paths": [
 {
 "access_type": "ref",
 "index": "cpao_settle_date_id",
 "rows": 7,
 "cost": 3.4,
 "chosen": true
 },
 {
 "access_type": "range",
 "rows": 3,
 "cost": 5.21,
 "chosen": false
 }
 ]
 },
 "cost_for_plan": 3.4,
 "rows_for_plan": 7,
 "chosen": true
 }
 ]
`

可以看到，由于 where 条件中只有 cpao_settle_date_id & id 部分，索引选择的仍是ref, 其代价的计算结果与 situation_unique_key_id 中的代价是一致的，但是在 optimizer_trace 后面发现了如下的优化：

`"attaching_conditions_to_tables": {
 "original_condition": "((`cpa_order`.`settle_date` = '2015-11-05') and (`cpa_order`.`id` > 15))",
 "attached_conditions_computation": [
 {
 "access_type_changed": {
 "table": "`cpa_order`",
 "index": "cpao_settle_date_id",
 "old_type": "ref",
 "new_type": "range",
 "cause": "uses_more_keyparts"
 }
 }
`

这里我们不难看出，在计算的结尾处优化器做了个优化，就是把 id 字段也考虑了进来，我们根据 attached_conditions_computation 的提示找到了如下代码:

` if (tab->type == JT_REF && // 1)
 !tab->ref.depend_map && // 2)
 tab->quick && // 3)
 (uint) tab->ref.key == tab->quick->index && // 4)
 tab->ref.key_length < tab->quick->max_used_key_length) // 5)
 {
 tab->type=JT_ALL;
 use_quick_range=1;
 tab->use_quick=QS_RANGE;
 tab->ref.key= -1;
 tab->ref.key_parts=0;
 }
`

结合注释，我们可以这样理解：

* ref 与 range 使用的是相同的索引；
* 当前 table 选择的索引采用的是ref；
* ref key 的使用的长度小于 range 的长度，则优先使用 range。

因此，在 situation_without_key_id 时，三个条件都满足，所以使用了 range 中的联合索引，那为什么 situation_unique_key_id 没有使用 id 呢，原因是在range 的代价计算过程中使用的是 id 这个索引，导致 unique id 这个索引与联合索引 cpao_settle_date_id 并不是同样的索引，不满足第一个条件，因此不进行优化。

有了上面的分析，我们观察 situation_plain_key_id 的代价及生成过程，situation_plain_key_id 在 range 的代价计算过程中选择的是 cpao_settle_date_id 索引，计算过程是将后者的计算结果与前者进行比较，因此即使相等，也是先入为主，其optimizer_trace如下：

range 部分的代价计算过程：

` "range_scan_alternatives": [
 {
 "index": "cpao_settle_date_id",
 "ranges": [
 "2015-11-05 <= settle_date <= 2015-11-05 AND 15 < id"
 ],
 "index_dives_for_eq_ranges": true,
 "rowid_ordered": false,
 "using_mrr": false,
 "index_only": false,
 "rows": 3,
 "cost": 4.61,
 "chosen": true
 },
 {
 "index": "id",
 "ranges": [
 "15 < id"
 ],
 "index_dives_for_eq_ranges": true,
 "rowid_ordered": false,
 "using_mrr": false,
 "index_only": false,
 "rows": 3,
 "cost": 4.61,
 "chosen": false,
 "cause": "cost"
 }
 ]
`

表的索引选择过程：

` "considered_execution_plans": [
 {
 "plan_prefix": [
 ],
 "table": "`cpa_order`",
 "best_access_path": {
 "considered_access_paths": [
 {
 "access_type": "ref",
 "index": "cpao_settle_date_id",
 "rows": 7,
 "cost": 3.4,
 "chosen": true
 },
 {
 "access_type": "range",
 "rows": 3,
 "cost": 5.21,
 "chosen": false
 }
 ]
 },
 "cost_for_plan": 3.4,
 "rows_for_plan": 7,
 "chosen": true
 }
 ]
`

结合上面的分析我们发现，ref & range 选择都是索引 cpao_settle_date_id，因此在最后的选择阶段也会进行索引的优化，与开头的问题表现相符。

## range 代价计算过程

优化器在索引选择的过程中会将where 条件、join 条件等信息进行收集，对于非等值的索引会放到 possible keys 中，进行 range 部分的代价计算，对于等值相关字段的索引会进行 ref 部分的代价计算，如果是单表，其主要过程如下：

* 调用 `get_key_scans_params` 从已知的索引中选择一个代价最小的 read_plan，利用 read_plan 生成一个读表的计划，缓存至 tab->quick 中；
* 在 `best_access_path` 中计算:
 
 全表的代价
* 如果有覆盖索引则计算覆盖索引的代价
* 如果有quick，则利用一些校验值计算上一步产生的 range 的代价
* 在 `make_join_select` 中对已经生成的执行计划进行较正，如 situation_plain_key_id 的优化部分。

多表的计算过程更为复杂，不在此描述。

## 问题解答

**为什么在 id 为 unique 的时候联合索引只使用了其中的一个字段而没有字段 id ?**

由于 situation_unique_key_id 中在计算 range 的过程中使用的是索引 id 而不是 cpao_settle_date_id，因此不符合最后优化的条件，因此只使用了 cpao_settle_date_id 的前一部分而没有使用 id，这是优化器在实现过程中的问题。

## range 代价计算过程可能引起的问题

我们已经了解了 range 代价计算的过程，可以发现可能会有以下问题：

* 当多个索引得到的代价是相同的，由于先入为主，只能缓存第一个，所以会有索引出错的问题；
* 每一次计算 range 的代价都会将缓存清空，如 order by limit 操作，这样有可能将之前的索引清空且走错索引，详情见 bug#78993。

## 小结

当执行计划出错的时候，我们可以有效的利用 optimizer_trace 来进行初步的分析，大部分还是有解的。另外由于执行计划的内容比较多，从本篇起，小编会尽量将优化器相关的东西给大家介绍一下，主要包括 optimizer_swith 的选项、含义、作用、以及在内核中是如何实现的，达到一起学习的目的。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)