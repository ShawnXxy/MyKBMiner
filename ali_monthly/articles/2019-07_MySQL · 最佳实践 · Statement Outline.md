# MySQL · 最佳实践 · Statement Outline

**Date:** 2019/07
**Source:** http://mysql.taobao.org/monthly/2019/07/01/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2019 / 07
 ](/monthly/2019/07)

 * 当期文章

 MySQL · 最佳实践 · Statement Outline
* PgSQL · 新特性解读 · undo log 存储接口（上）
* MySQL · 引擎特性 · Buffer Pool 漫谈
* MongoDB · 引擎特性 · oplog 查询优化
* PgSQL · 最佳实践 · pg_cron 内核分析及用法简介
* MySQL · 引擎特性 · CTE(Common Table Expressions)
* Database · 理论基础 · Mass Tree
* MySQL · 源码分析 · `slow log` 与`CSV`引擎
* PgSQL · 应用案例 · 使用SQL查询数据库日志
* PgSQL · 应用案例 · PostgreSQL psql的元素周期表

 ## MySQL · 最佳实践 · Statement Outline 
 Author: lengxiang 

 ## 背景

在生产环境，MySQL 数据库实例运行过程中，一些 SQL 语句会发生执行计划的变化，导致增加了数据库稳定性的风险， 这里边有几个因素和场景，比如：随着表数据量的变化，以及统计信息的自动收集，CBO optimizer 计算得到了一个cost 更低的 plan， 又或者 表结构发生了变化，增加和删减了某些索引，或者在实例升级迁移等过程中，MySQL 自身优化器行为和算法发生了变化等。 为了能够在线应对和干预 业务 SQL 语句的执行计划， AliSQL 设计了一套利用 MySQL optimizer/index hint 来稳定执行计划的方法，称为 Statement outline，并提供了 一组管理接口方便使用（DBMS_OUTLN package）， 并在 RDS MySQL 8.0 产品上公开使用。

## Outline 设计

AliSQL 8.0 outline 支持 MySQL 8.0 官方支持的所有 hint 类型，主要分为两大类：

1 Optimizer Hint

根据作用域（query block）和 hint 对象，又分为：Global level hint，Table/Index level hint,Join order hint等等。
 详细信息参考：[https://dev.mysql.com/doc/refman/8.0/en/optimizer-hints.html](https://dev.mysql.com/doc/refman/8.0/en/optimizer-hints.html)

2 Index Hint

主要根据 index hint 的类型 （USE, FORCE, IGNORE）和 scope  （FOR JOIN, FOR ORDER BY,FOR GROUP BY）进行分类。
 详细语法参考：[https://dev.mysql.com/doc/refman/8.0/en/index-hints.html](https://dev.mysql.com/doc/refman/8.0/en/index-hints.html)

为了表示和抽象这些 Hint，并能够持久化 outline，AliSQL 8.0 增加了一个系统表 mysql.outline，其结构如下：

**MYSQL.OUTLINE**

`CREATE TABLE `mysql`.`outline` (
 `Id` bigint(20) NOT NULL AUTO_INCREMENT,
 `Schema_name` varchar(64) COLLATE utf8_bin DEFAULT NULL,
 `Digest` varchar(64) COLLATE utf8_bin NOT NULL,
 `Digest_text` longtext COLLATE utf8_bin,
 `Type` enum('IGNORE INDEX','USE INDEX','FORCE INDEX','OPTIMIZER') CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
 `Scope` enum('','FOR JOIN','FOR ORDER BY','FOR GROUP BY') CHARACTER SET utf8 COLLATE utf8_general_ci DEFAULT '',
 `State` enum('N','Y') CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL DEFAULT 'Y',
 `Position` bigint(20) NOT NULL,
 `Hint` text COLLATE utf8_bin NOT NULL,
 PRIMARY KEY (`Id`)
 ) /*!50100 TABLESPACE `mysql` */ ENGINE=InnoDB
DEFAULT CHARSET=utf8 COLLATE=utf8_bin STATS_PERSISTENT=0 COMMENT='Statement outline'
`

**Columns**

1 **Digest/Digest_text**

Outline 根据语句的特征进行匹配，这个特征就是 Digest text，根据这个 Digest Text 进行 hash 计算得到一个 64 字节的 hash 字符串。例如：

`Statement query: select * from t1 where id = 1

根据计算得到的Digest 和 Digest text 分别是：

Digest ： 36bebc61fce7e32b93926aec3fdd790dad5d895107e2d8d3848d1c60b74bcde6

Digest_text: SELECT * FROM `t1` WHERE `id` = ? 
`

当语句 parse 完之后， 会根据 [schema + digest] 作为 hash key，进行查询匹配的 Outline。

2 **Type**

所有的 optimizer hint 的 type 统一为 OPTIMIZER.
Index hint 分为三类， 分别是：

* USE INDEX
* FORCE INDEX
* IGNORE INDEX

3 ** Scope**

 scope 只针对 Index hint 而言，分为四类：

* FOR GROUP BY
* FOR ORDER BY
* FOR JOIN

 如果是 空串，表示 ALL

4 **Position**

Position 非常关键：
Optimizer hint 中，position 表示 Query Block， 因为所有的 optimizer hint 必须作用到 Query Block上，这里判断比较简单， 因为 Optimizer hint 只支持 这几类关键字：

`SELECT /*+ ... */ ...
INSERT /*+ ... */ ...
REPLACE /*+ ... */ ...
UPDATE /*+ ... */ ...
DELETE /*+ ... */ ...
`
所以，position 从 1 开始，hint 作用在语句的第几个关键字锚点上，就是几。

Index hint 中， position 表示 table position， 也是从1开始，hint作用在第几个 table 锚点上，就是几。

5 **Hint**

在 Index hint 中， 这里表示的是 索引名字的列表， 比如 “ind_1, ind_2”在 Optimizer hint 中， 这里表示的就是完整的 hint 字符串，比如：“/*+ MAX_EXECUTION_TIME(1000) */”

## 用户接口

为了更方便的管理 Statement outline，AliSQL 设计了一个 DBMS_OUTLN package 来进行管理，并提供了 5 个native procedure 接口：

`DBMS_OUTLN.add_index_outline(); 增加 index hint
DBMS_OUTLN.add_optimizer_outline(); 增加 optimizer hint
DBMS_OUTLN.preview_outline(); 预览某一个 SQL 语句命中 outline 的情况
DBMS_OUTLN.show_outline(); 展示内存中可用的所有 outline 及命中情况
DBMS_OUTLN.del_outline(); 删除内存和持久化表中的 outline
DBMS_OUTLN.flush_outline(); 刷新所有的 outline，从 mysql.outline 表中重新 load
`

为了方便的介绍 DBMS_OUTLN 的使用，这里使用一些测试表：

` CREATE TABLE `t1` (
 `id` int(11) NOT NULL AUTO_INCREMENT,
 `col1` int(11) DEFAULT NULL,
 `col2` varchar(100) DEFAULT NULL,
 PRIMARY KEY (`id`),
 KEY `ind_1` (`col1`),
 KEY `ind_2` (`col2`)
 ) ENGINE=InnoDB DEFAULT CHARSET=utf8;

 CREATE TABLE `t2` (
 `id` int(11) NOT NULL AUTO_INCREMENT,
 `col1` int(11) DEFAULT NULL,
 `col2` varchar(100) DEFAULT NULL,
 PRIMARY KEY (`id`),
 KEY `ind_1` (`col1`),
 KEY `ind_2` (`col2`)
 ) ENGINE=InnoDB DEFAULT CHARSET=utf8;

`

### ADD_INDEX_OUTLINE

#### 语法和参数

```
 CALL DBMS_OUTLN.add_index_outline(schema=>, digest=>, position=>, type=>,
 hint=>, scope=>, query=>);

说明：
digest 和 query 可以选择其一， 如果填写了原始query语句，这个 proc 会计算 digest 和 digest text。

```

#### 测试 case 1

测试语句

`select * from t1 where t1.col1 =1 and t1.col2 ='xpchild';
`
   使用 ind_1 的索引

`call dbms_outln.add_index_outline('outline_db', '', 1, 'USE INDEX', 'ind_1', '',
 "select * from t1 where t1.col1 =1 and t1.col2 ='xpchild'");

查看 outline：

mysql> call dbms_outln.show_outline();
+------+------------+------------------------------------------------------------------+-----------+-------+------+-------+------+----------+------------------------------------------------------------------+
| ID | SCHEMA | DIGEST | TYPE | SCOPE | POS | HINT | HIT | OVERFLOW | DIGEST_TEXT |
+------+------------+------------------------------------------------------------------+-----------+-------+------+-------+------+----------+------------------------------------------------------------------+
| 30 | outline_db | b4369611be7ab2d27c85897632576a04bc08f50b928a1d735b62d0a140628c4c | USE INDEX | | 1 | ind_1 | 0 | 0 | SELECT * FROM `t1` WHERE `t1` . `col1` = ? AND `t1` . `col2` = ? |
 +------+------------+------------------------------------------------------------------+-----------+-------+------+-------+------+----------+------------------------------------------------------------------+
1 row in set (0.00 sec)
`

验证 Outline：

验证 Outline 是否其效果，可以有两种方法：

1. dbms_outln.preview_outline() 进行预览
2. 直接使用 explain 进行查看。

` mysql> call dbms_outln.preview_outline('outline_db', "select * from t1 where t1.col1 =1 and t1.col2 ='xpchild'");
 +------------+------------------------------------------------------------------+------------+------------+-------+---------------------+
 | SCHEMA | DIGEST | BLOCK_TYPE | BLOCK_NAME | BLOCK | HINT |
 +------------+------------------------------------------------------------------+------------+------------+-------+---------------------+
 | outline_db | b4369611be7ab2d27c85897632576a04bc08f50b928a1d735b62d0a140628c4c | TABLE | t1 | 1 | USE INDEX (`ind_1`) |
 +------------+------------------------------------------------------------------+------------+------------+-------+---------------------+
 1 row in set (0.01 sec)

 mysql> explain select * from t1 where t1.col1 =1 and t1.col2 ='xpchild';
 +----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-------------+
 | id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
 +----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-------------+
 | 1 | SIMPLE | t1 | NULL | ref | ind_1 | ind_1 | 5 | const | 1 | 100.00 | Using where |
 +----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-------------+
 1 row in set, 1 warning (0.00 sec)

 mysql> show warnings;
 +-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
 | Level | Code | Message |
 +-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
 | Note | 1003 | /* select#1 */ select `outline_db`.`t1`.`id` AS `id`,`outline_db`.`t1`.`col1` AS `col1`,`outline_db`.`t1`.`col2` AS `col2` from `outline_db`.`t1` USE INDEX (`ind_1`) where ((`outline_db`.`t1`.`col1` = 1) and (`outline_db`.`t1`.`col2` = 'xpchild')) |
 +-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
 1 row in set (0.00 sec)
`

#### 测试 case 2
测试语句：

` select * from t1, t2 where t1.col1 = t2.col1 and t2.col2 ='xpchild'
`

测试使用 t2 表的 ind_2 索引：

` call dbms_outln.add_index_outline('outline_db', '', 2, 'USE INDEX', 'ind_2', '',
 "select * from t1, t2 where t1.col1 = t2.col1 and t2.col2 ='xpchild'");

`
 验证 Outline：

` mysql> explain select * from t1, t2 where t1.col1 = t2.col1 and t2.col2 ='xpchild';
 +----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-------------+
 | id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
 +----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-------------+
 | 1 | SIMPLE | t1 | NULL | ALL | ind_1 | NULL | NULL | NULL | 1 | 100.00 | NULL |
 | 1 | SIMPLE | t2 | NULL | ref | ind_2 | ind_2 | 303 | const | 1 | 100.00 | Using where |
 +----+-------------+-------+------------+------+---------------+-------+---------+-------+------+----------+-------------+
 2 rows in set, 1 warning (0.01 sec)

 mysql> show warnings;
 +-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
 | Level | Code | Message |
 +-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
 | Note | 1003 | /* select#1 */ select `outline_db`.`t1`.`id` AS `id`,`outline_db`.`t1`.`col1` AS `col1`,`outline_db`.`t1`.`col2` AS `col2`,`outline_db`.`t2`.`id` AS `id`,`outline_db`.`t2`.`col1` AS `col1`,`outline_db`.`t2`.`col2` AS `col2` from `outline_db`.`t1` join `outline_db`.`t2` USE INDEX (`ind_2`) where ((`outline_db`.`t2`.`col1` = `outline_db`.`t1`.`col1`) and (`outline_db`.`t2`.`col2` = 'xpchild')) |
 +-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
 1 row in set (0.00 sec)
`

### ADD_OPTIMIZER_OUTLINE
#### 语法和参数

```
 CALL DBMS_OUTLN.add_optimizer_outline(schema=>, digest=>, query_block=>
 hint=>, query=>);

 说明：digest 和 query 同样可以填其一，或者都填入。proc 会自动计算digest 和 digest text。

```

#### 测试 case 1

增加全局 MAX_EXECUTION_TIME / SET VAR optimizer hint;

` CALL DBMS_OUTLN.add_optimizer_outline("outline_db", '', 1, '/*+ MAX_EXECUTION_TIME(1000) */',
 "select * from t1 where id = 1");
 CALL DBMS_OUTLN.add_optimizer_outline("outline_db", '', 1, '/*+ SET_VAR(foreign_key_checks=OFF) */',
 "select * from t1 where id = 1");
`
验证 Outline：

`mysql> explain select * from t1 where id = 1;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------+
| id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------+
| 1 | SIMPLE | NULL | NULL | NULL | NULL | NULL | NULL | NULL | NULL | NULL | no matching row in const table |
 +----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+--------------------------------+
1 row in set, 1 warning (0.01 sec)

mysql> show warnings;
+-------+------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Level | Code | Message |
+-------+------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Note | 1003 | /* select#1 */ select /*+ MAX_EXECUTION_TIME(1000) SET_VAR(foreign_key_checks='OFF') */ NULL AS `id`,NULL AS `col1`,NULL AS `col2` from `outline_db`.`t1` where multiple equal(1, NULL) |
+-------+------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
`

#### 测试 case 2
 测试多表关联查询： Nested-Loop join processing

` CALL DBMS_OUTLN.add_optimizer_outline('outline_db', '', 1, '/*+ BNL(t1,t2) */',
 "select t1.id, t2.id from t1,t2");
`

验证Outline：

`mysql> explain select t1.id, t2.id from t1,t2;
+----+-------------+-------+------------+-------+---------------+-------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+-------+---------+------+------+----------+----------------------------------------------------+
| 1 | SIMPLE | t1 | NULL | index | NULL | ind_1 | 5 | NULL | 1 | 100.00 | Using index |
| 1 | SIMPLE | t2 | NULL | index | NULL | ind_1 | 5 | NULL | 1 | 100.00 | Using index; Using join buffer (Block Nested Loop) |
 +----+-------------+-------+------------+-------+---------------+-------+---------+------+------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.01 sec)

mysql> show warnings;
+-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Level | Code | Message |
+-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Note | 1003 | /* select#1 */ select /*+ BNL(`t1`@`select#1`) BNL(`t2`@`select#1`) */ `outline_db`.`t1`.`id` AS `id`,`outline_db`.`t2`.`id` AS `id` from `outline_db`.`t1` join `outline_db`.`t2` |
+-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
`

#### 测试 case 3
测试 subquery 中带有 query block name 的情况

`CALL DBMS_OUTLN.add_optimizer_outline('outline_db', '', 2, ' /*+ QB_NAME(subq1) */', 
 "SELECT * FROM t1 WHERE t1.col1 IN (SELECT col1 FROM t2)");

CALL DBMS_OUTLN.add_optimizer_outline('outline_db', '', 1, '/*+ SEMIJOIN(@subq1 MATERIALIZATION, DUPSWEEDOUT) */ ',
 "SELECT * FROM t1 WHERE t1.col1 IN (SELECT col1 FROM t2)");

`
验证 Outline：

` mysql> explain SELECT * FROM t1 WHERE t1.col1 IN (SELECT col1 FROM t2);
 +----+--------------+-------------+------------+--------+---------------+------------+---------+--------------------+------+----------+-------------+
 | id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
 +----+--------------+-------------+------------+--------+---------------+------------+---------+--------------------+------+----------+-------------+
 | 1 | SIMPLE | t1 | NULL | ALL | ind_1 | NULL | NULL | NULL | 1 | 100.00 | Using where |
 | 1 | SIMPLE | <subquery2> | NULL | eq_ref | <auto_key> | <auto_key> | 5 | outline_db.t1.col1 | 1 | 100.00 | NULL |
 | 2 | MATERIALIZED | t2 | NULL | index | ind_1 | ind_1 | 5 | NULL | 1 | 100.00 | Using index |
 +----+--------------+-------------+------------+--------+---------------+------------+---------+--------------------+------+----------+-------------+
 3 rows in set, 1 warning (0.00 sec)

 mysql> show warnings;
 +-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
 | Level | Code | Message |
 +-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
 | Note | 1003 | /* select#1 */ select /*+ SEMIJOIN(@`subq1` MATERIALIZATION, DUPSWEEDOUT) */ `outline_db`.`t1`.`id` AS `id`,`outline_db`.`t1`.`col1` AS `col1`,`outline_db`.`t1`.`col2` AS `col2` from `outline_db`.`t1` semi join (`outline_db`.`t2`) where (`<subquery2>`.`col1` = `outline_db`.`t1`.`col1`) |
 +-------+------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
 1 row in set (0.00 sec)
`

### PREVIEW_OUTLINE
dbms_outln.preview_outline() 用于使用具体 SQL 语句，查看匹配 Outline 的情况，用于手动验证。其语法和参数：

` CALL DBMS_OUTLN.preview_outline(schema=>, query=>);
`

例如：

` mysql> call dbms_outln.preview_outline('outline_db', "select * from t1 where t1.col1 =1 and t1.col2 ='xpchild'");
 +------------+------------------------------------------------------------------+------------+------------+-------+---------------------+
 | SCHEMA | DIGEST | BLOCK_TYPE | BLOCK_NAME | BLOCK | HINT |
 +------------+------------------------------------------------------------------+------------+------------+-------+---------------------+
 | outline_db | b4369611be7ab2d27c85897632576a04bc08f50b928a1d735b62d0a140628c4c | TABLE | t1 | 1 | USE INDEX (`ind_1`) |
 +------------+------------------------------------------------------------------+------------+------------+-------+---------------------+
 1 row in set (0.00 sec)
`

### SHOW_OUTLINE
dbms_outln.show_outline 展示 outline 在内存 cache 中 命中的情况，里边有两个字段：HIT：                 说明 这个outline 命中的次数OVERFLOW：     说明 这个 outline hint 没有找到 query block 或者 相应的 table 的次数

例如：

` mysql> call dbms_outln.show_outline();
 +------+------------+------------------------------------------------------------------+-----------+-------+------+-------------------------------------------------------+------+----------+-------------------------------------------------------------------------------------+
 | ID | SCHEMA | DIGEST | TYPE | SCOPE | POS | HINT | HIT | OVERFLOW | DIGEST_TEXT |
 +------+------------+------------------------------------------------------------------+-----------+-------+------+-------------------------------------------------------+------+----------+-------------------------------------------------------------------------------------+
 | 33 | outline_db | 36bebc61fce7e32b93926aec3fdd790dad5d895107e2d8d3848d1c60b74bcde6 | OPTIMIZER | | 1 | /*+ SET_VAR(foreign_key_checks=OFF) */ | 1 | 0 | SELECT * FROM `t1` WHERE `id` = ? |
 | 32 | outline_db | 36bebc61fce7e32b93926aec3fdd790dad5d895107e2d8d3848d1c60b74bcde6 | OPTIMIZER | | 1 | /*+ MAX_EXECUTION_TIME(1000) */ | 2 | 0 | SELECT * FROM `t1` WHERE `id` = ? |
 | 34 | outline_db | d4dcef634a4a664518e5fb8a21c6ce9b79fccb44b773e86431eb67840975b649 | OPTIMIZER | | 1 | /*+ BNL(t1,t2) */ | 1 | 0 | SELECT `t1` . `id` , `t2` . `id` FROM `t1` , `t2` |
 | 35 | outline_db | 5a726a609b6fbfb76bb8f9d2a24af913a2b9d07f015f2ee1f6f2d12dfad72e6f | OPTIMIZER | | 2 | /*+ QB_NAME(subq1) */ | 2 | 0 | SELECT * FROM `t1` WHERE `t1` . `col1` IN ( SELECT `col1` FROM `t2` ) |
 | 36 | outline_db | 5a726a609b6fbfb76bb8f9d2a24af913a2b9d07f015f2ee1f6f2d12dfad72e6f | OPTIMIZER | | 1 | /*+ SEMIJOIN(@subq1 MATERIALIZATION, DUPSWEEDOUT) */ | 2 | 0 | SELECT * FROM `t1` WHERE `t1` . `col1` IN ( SELECT `col1` FROM `t2` ) |
 | 30 | outline_db | b4369611be7ab2d27c85897632576a04bc08f50b928a1d735b62d0a140628c4c | USE INDEX | | 1 | ind_1 | 3 | 0 | SELECT * FROM `t1` WHERE `t1` . `col1` = ? AND `t1` . `col2` = ? |
 | 31 | outline_db | 33c71541754093f78a1f2108795cfb45f8b15ec5d6bff76884f4461fb7f33419 | USE INDEX | | 2 | ind_2 | 1 | 0 | SELECT * FROM `t1` , `t2` WHERE `t1` . `col1` = `t2` . `col1` AND `t2` . `col2` = ? |
 +------+------------+------------------------------------------------------------------+-----------+-------+------+-------------------------------------------------------+------+----------+-------------------------------------------------------------------------------------+
 7 rows in set (0.00 sec)
`

### DEL_OUTLINE
dbms_outln.del_outline() 可以删除内存和表中的某一条 outline。

语法和参数如下：

` CALL DBMS_OUTLN.del_outline(outline_id=>);
`

例如：

` mysql> call dbms_outln.del_outline(1000);
 Query OK, 0 rows affected, 2 warnings (0.00 sec)

 mysql> show warnings;
 +---------+------+----------------------------------------------+
 | Level | Code | Message |
 +---------+------+----------------------------------------------+
 | Warning | 7521 | Statement outline 1000 is not found in table |
 | Warning | 7521 | Statement outline 1000 is not found in cache |
 +---------+------+----------------------------------------------+
 2 rows in set (0.00 sec)
`

### FLUSH_OUTLINE
dbms_outln.flush_outline() 支持清理 cache 中 outline，并从 mysql.outline 表中重新 load。如果用户直接修改表来加载 outline，需要调用 flush 到 cache 中。

例如：

```
 mysql> call dbms_outln.flush_outline(); 
 Query OK, 0 rows affected (0.01 sec)

```

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)