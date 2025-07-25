# MySQL · 捉虫动态 · order by limit 造成优化器选择索引错误

**Date:** 2015/11
**Source:** http://mysql.taobao.org/monthly/2015/11/10/
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

 ## MySQL · 捉虫动态 · order by limit 造成优化器选择索引错误 
 Author: xijia 

 ## 问题描述

bug 触发条件如下：

1. 优化器先选择了 where 条件中字段的索引，该索引过滤性较好；
2. SQL 中必须有 order by limit 从而引导优化器尝试使用 order by 字段上的索引进行优化，最终因代价问题没有成功。

### 复现case

**表结构**

`create table t1(
 id int auto_increment primary key,
 a int, b int, c int,
 key iabc (a, b, c),
 key ic (c)
) engine = innodb;
`

**构造数据**

`insert into t1 select null,null,null,null;
insert into t1 select null,null,null,null from t1;
insert into t1 select null,null,null,null from t1;
insert into t1 select null,null,null,null from t1;
insert into t1 select null,null,null,null from t1;
insert into t1 select null,null,null,null from t1;
update t1 set a = id / 2, b = id / 4, c = 6 - id / 8;
`

**触发SQL**

`mysql> explain select id from t1 where a<3 and b in (1, 13) and c>=3 order by c limit 2\G
*************************** 1. row ***************************
 id: 1
 select_type: SIMPLE
 table: t1
 type: index
possible_keys: iabc,ic
 key: iabc
 key_len: 15
 ref: NULL
 rows: 32
 Extra: Using where; Using index; Using filesort
`

**使用 force index 可以选择过滤性好的索引**

`mysql> explain select id from t1 force index(iabc) where a<3 and b in (1, 13) and c>=3 order by c limit 2\G
*************************** 1. row ***************************
 id: 1
 select_type: SIMPLE
 table: t1
 type: range
possible_keys: iabc
 key: iabc
 key_len: 5
 ref: NULL
 rows: 3
 Extra: Using where; Using index; Using filesort
`

## 问题分析

optimizer_trace 可以帮助分析这个问题。

SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE\G

` "range_scan_alternatives": [
 {
 "index": "iabc",
 "ranges": [
 "NULL < a < 3"
 ],
 "index_dives_for_eq_ranges": true,
 "rowid_ordered": false,
 "using_mrr": false,
 "index_only": true,
 "rows": 3,
 "cost": 1.6146,
 "chosen": true
 },
 {
 "index": "ic",
 "ranges": [
 "3 <= c"
 ],
 "index_dives_for_eq_ranges": true,
 "rowid_ordered": false,
 "using_mrr": false,
 "index_only": false,
 "rows": 17,
 "cost": 21.41,
 "chosen": false,
 "cause": "cost"
 }
 ],
`

range_scan_alternatives 计算 range_scan，各个索引的开销，从上面的结果可以看出，联合索引 iabc 开销较小，应该选择 iabc。

` "considered_execution_plans": [
 {
 "plan_prefix": [
 ],
 "table": "`t1`",
 "best_access_path": {
 "considered_access_paths": [
 {
 "access_type": "range",
 "rows": 3,
 "cost": 2.2146,
 "chosen": true
 }
 ]
 },
 "cost_for_plan": 2.2146,
 "rows_for_plan": 3,
 "chosen": true
 }
 ]
`

considered_execution_plans 表索引选择过程，access_type 是 range，rows_for_plan=3，到这里为止，执行计划还是符合预期的。

` {
 "clause_processing": {
 "clause": "ORDER BY",
 "original_clause": "`t1`.`c`",
 "items": [
 {
 "item": "`t1`.`c`"
 }
 ],
 "resulting_clause_is_simple": true,
 "resulting_clause": "`t1`.`c`"
 }
 },
 {
 "refine_plan": [
 {
 "table": "`t1`",
 "access_type": "index_scan"
 }
 ]
 },
 {
 "reconsidering_access_paths_for_index_ordering": {
 "clause": "ORDER BY",
 "index_order_summary": {
 "table": "`t1`",
 "index_provides_order": false,
 "order_direction": "undefined",
 "index": "unknown",
 "plan_changed": false
 }
 }
 }
`

clause_processing 用于简化 order by，经过 clause_processing access_type 变成 index_scan（全索引扫描，过滤性较range差），此时出现了和预期不符的结果。

因此可以推测优化器试图优化 order by 时出现了错误：

* 第一阶段，优化器选择了索引 iabc，采用 range 访问；
* 第二阶段，优化器试图进一步优化执行计划，使用 order by 的列访问，并清空了第一阶段的结果；
* 第三阶段，优化器发现使用 order by 的列访问，代价比第一阶段的结果更大，但是第一阶段结果已经被清空了，无法还原，于是选择了代价较大的访问方式（index_scan），触发了bug。

## 问题解决

1. 我们在索引优化函数`SQL_SELECT::test_quick_select` 最开始的时候保存访问计划变量（quick）；
2. 在索引没变的时候，还原这个变量；
3. 在索引发生改变的时候，删除这个变量。

在不修改 mysql 源码的情况下，可以通过 force index 强制指定索引规避这个bug。

`SQL_SELECT::test_quick_select` 调用栈如下

```
 #0 SQL_SELECT::test_quick_select
 #1 make_join_select
 #2 JOIN::optimize
 #3 mysql_execute_select
 #4 mysql_select
 #5 mysql_explain_unit
 #6 explain_query_expression
 #7 execute_sqlcom_select
 #8 mysql_execute_command
 #9 mysql_parse
 #10 dispatch_command
 #11 do_command

```

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)