# MySQL · 新特性分析 · CTE执行过程与实现原理

**Date:** 2017/02
**Source:** http://mysql.taobao.org/monthly/2017/02/06/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 02
 ](/monthly/2017/02)

 * 当期文章

 AliSQL · 开源 · Sequence Engine
* MySQL · myrocks · myrocks之备份恢复
* MySQL · 挖坑 · LOCK_active_mi/LOCK_msp_map 优化思路
* MySQL · 源码分析 · 词法分析及其性能优化
* SQL优化 · 经典案例 · 索引篇
* MySQL · 新特性分析 · CTE执行过程与实现原理
* PgSQL · 源码分析 · PG优化器物理查询优化
* SQL Server · 特性介绍 · 聚集列存储索引
* PgSQL · 应用案例 · 聚集存储 与 BRIN索引
* PgSQL · 应用案例 · GIN索引在任意组合查询中的应用

 ## MySQL · 新特性分析 · CTE执行过程与实现原理 
 Author: 令猴 

 众所周知，Common table expression(CTE)是在大多数的关系型数据库里都存在的特性，包括ORACLE, SQLSERVER,POSTGRESQL等，唯独开源数据库老大MySQL缺失。CTE作为一个方便用户使用的功能，原本是可以利用普通的SQL语句替代的，但是对于复杂的CTE来说，要模拟出CTE的效果还是需要很大的功夫。如果考虑性能那就更是难上加难了。2013年Guilhem Bichot发表的[一篇blog](http://guilhembichot.blogspot.com/2013_11_01_archive.html)模拟了CTE的场景，
从该篇blog中可以看出，对于模拟复杂CTE的场景的难度就可见一斑。2016年9月份，Guilhem实现了MySQL自己的CTE特性，并在MySQL的lab release中进行了发布，邀请评测。本篇文章就是对这个lab release中的CTE实现过程进行一个剖析，让我们了解一下CTE在MySQL内部是如何实现的。

## 首先，我们看一下简单非递归的CTE的工作过程

`CREATE TABLE t(a int);
INSERT INTO t VALUES(1),(2);
`

下面我们尝试执行一些语句：

`mysql> WITH cte(x) as
 -> (SELECT * FROM t)
 -> SELECT * FROM cte;
+------+
| x |
+------+
| 1 |
+------+
1 row in set (0.00 sec)
`

可以看到CTE可以工作了。

`mysql> SET OPTIMIZER_SWITCH='derived_merge=off';
Query OK, 0 rows affected (0.00 sec)
为了清楚的看到OPTIMIZER的优化过程，我们先暂且关闭derived_merge特性。

mysql> EXPLAIN WITH cte(x) as
 -> (SELECT * FROM t)
 -> SELECT * FROM cte;
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
| 1 | PRIMARY | <derived2> | NULL | ALL | NULL | NULL | NULL | NULL | 2 | 100.00 | NULL |
| 2 | DERIVED | t | NULL | ALL | NULL | NULL | NULL | NULL | 1 | 100.00 | NULL |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
2 rows in set, 1 warning (0.00 sec)

mysql> show warnings; 
+-------+------+-------------------------------------------------------------------------------------------------------------------------------------+
| Level | Code | Message |
+-------+------+-------------------------------------------------------------------------------------------------------------------------------------+
| Note | 1003 | with `cte` (`x`) as (/* select#2 */ select `test`.`t`.`a` AS `a` from `test`.`t`) /* select#1 */ select `cte`.`x` AS `x` from `cte` |
+-------+------+-------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
`

从上面的EXPLAIN输出结果我们可以看到，CTE内部优化过程走的流程和subquery是一样的。下面我们打开derived_merge特性来继续看一下。

`mysql> SET OPTIMIZER_SWITCH='derived_merge=on';
Query OK, 0 rows affected (0.00 sec)

mysql> EXPLAIN WITH cte(x) as
 -> (SELECT * FROM t)
 -> SELECT * FROM cte;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
| 1 | SIMPLE | t | NULL | ALL | NULL | NULL | NULL | NULL | 1 | 100.00 | NULL |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

mysql> show warnings;
+-------+------+-------------------------------------------------------------+
| Level | Code | Message |
+-------+------+-------------------------------------------------------------+
| Note | 1003 | /* select#1 */ select `test`.`t`.`a` AS `x` from `test`.`t` |
+-------+------+-------------------------------------------------------------+
1 row in set (0.00 sec)
`
从执行计划上我们可以看出CTE已经被优化掉了，并且被merge到了subquery的上层查询。难道CTE仅仅只是subquery的一个替代？那么CTE除了递归特性（稍后介绍），与subquery的区别在哪里呢？下面我们继续看一个栗子：
为了清楚的看到区别，我们还是关闭derived_merge特性。

`mysql> SET OPTIMIZER_SWITCH='derived_merge=off';
Query OK, 0 rows affected (0.00 sec)

mysql> EXPLAIN WITH cte(x) as
 (SELECT * FROM t)
 SELECT * FROM 
 (SELECT * FROM cte) AS t1,
 (SELECT * FROM cte) AS t2;
mysql> 执行计划截取片断如下
...
 {
 "table": {
 "table_name": "t2",
 "access_type": "ALL",
 "rows_examined_per_scan": 2,
 "rows_produced_per_join": 4,
 "filtered": "100.00",
 "using_join_buffer": "Block Nested Loop",
 "cost_info": {
 "read_cost": "10.10",
 "eval_cost": "0.80",
 "prefix_cost": "21.40",
 "data_read_per_join": "64"
 },
 "used_columns": [
 "x"
 ],
 "materialized_from_subquery": {
 "using_temporary_table": true,
 "dependent": false,
 "cacheable": true,
 "query_block": {
 "select_id": 4,
 "cost_info": {
 "query_cost": "10.50"
 },
 "table": {
 "table_name": "cte",
 "access_type": "ALL",
 "rows_examined_per_scan": 2,
 "rows_produced_per_join": 2,
 "filtered": "100.00",
 "cost_info": {
 "read_cost": "10.10",
 "eval_cost": "0.40",
 "prefix_cost": "10.50",
 "data_read_per_join": "32"
 },
 "used_columns": [
 "x"
 ],
 "materialized_from_subquery": {
 "sharing_temporary_table_with": { <<注意这里临时表是共享的
 "select_id": 3
 }
 }
 }
 }
 }
 }
 }
`

我们可以看到对于CTE来说，多次利用只会被执行一次。而对于subquery来说，对于每一条query都至少会执行一次。

那么CTE是如何实现多次利用的呢？让我们看看代码： 
首先了解一下Common_table_expr这个类的定义：

`class Common_table_expr
{
public:
 // 构造函数
 Common_table_expr(MEM_ROOT *mem_root) : references(mem_root),
 recursive(false), tmp_tables(mem_root)
 {}
 // 该函数负责按照CTE的定义（包括CTE的alias，已经自定义的列名）生成一个新的临时表信息，进而替代resolve derived table过程中生成的临时表信息。 
 TABLE *clone_tmp_table(THD *thd, const char *alias);
 // 克隆第一个临时表信息来替换对Query中所有（包含递归CTE定义）对CTE的引用
 bool substitute_recursive_reference(THD *thd, SELECT_LEX *sl);
 // Query中除了CTE自身定义外对该CTE的所有引用的一个数组。
 Mem_root_array<TABLE_LIST *> references;
 /// 是否是递归CTE
 bool recursive;
 /** 
 Array中所有的临时表都是与该CTE相关的，Query中每次用到CTE都会对应生成一个临时表信息。
 但是只有第一个临时表会被存储引擎创建，其他都是共享该临时表。
 */
 List of all TABLEs pointing to the tmp table created to materialize this
 Mem_root_array<TABLE *> tmp_tables;
};
`
接下来是代码中对于CTE多次引用共享一个临时表实例的代码片断。

`bool TABLE_LIST::create_derived(THD *thd)
{
 DBUG_ENTER("TABLE_LIST::create_derived");

 SELECT_LEX_UNIT *const unit= derived_unit();

 // @todo: Be able to assert !table->is_created() as well
 DBUG_ASSERT(unit && uses_materialization() && table);

 if (!table->is_created()) // 当第2次为CTE创建临时表的时候，此时发现临时表还没有创建
 {
 Derived_refs_iterator it(table); 
 while (TABLE *t= it.get_next()) // 这里会遍历CTE表达式相关的所有已经创建的临时表
 if (t->is_created()) // 找到已经创建好的临时表
 { 
 // 直接再次打开临时表，不再重新生成一个临时表。从而达到CTE临时表被共享利用的过程。
 if (open_tmp_table(table)) 
 
 DBUG_RETURN(true);
 break;
 } 
 }
`

## 接下来，我们研究一下递归CTE

下面看一个栗子

`CREATE TABLE t(a int);
INSERT INTO t VALUES(2),(5);
`
```
mysql> WITH RECURSIVE my_cte AS 
 (SELECT a from t UNION ALL SELECT 2+a FROM my_cte WHERE a<10 ) 
 SELECT * FROM my_cte;
+------+
| a |
+------+
| 2 |
| 5 |
| 4 |
| 7 |
| 6 |
| 9 |
| 8 |
| 11 |
| 10 |
+------+
9 rows in set (15 min 54.43 sec)

```

对于递归的CTE，结构分为两个部分，一部分是SEED部分，就是不包含CTE自身的部分，作为接下来递归的初始值。另一个部分就是递归如何产生新的记录。对于上面的栗子而言：
SEED部分就是SELECT a from t；递归CTE的新纪录生成规则为SELECT 2+a FROM my_cte WHERE a<10。
对应到代码中是MySQL是如何执行的呢？首先看一个为CTE定义的执行器类结构的重要成员：

`class Recursive_executor
{
private:
 // 对应到CTE的定义部分
 SELECT_LEX_UNIT *unit;

 // 对应CTE递归的次数
 uint iteration_counter;
 ...
public：
 // 负责初始化CTE执行器并打开临时表
 bool initialize();
 // 该函数负责定位SEED部分还是CTE递归规则部分，当iteration_counter=0时，返回SEED部分，否则返回CTE递归规则部分
 SELECT_LEX *first_select() const;
 // 该函数是用来辅助执行器定位SEED部分的结尾以及CTE递归规则的结尾
 SELECT_LEX *last_select() const
 // 该函数用来判断CTE是否依旧满足递归条件，如果满足执行器便会继续执行CTE的递归部分
 bool more_iterations();
}
`

下面代码片段描述了CTE的执行过程：

`bool SELECT_LEX_UNIT::execute(THD *thd)
{
 ...

 do
 {
 for (auto sl= recursive_executor.first_select();
 sl != recursive_executor.last_select();
 sl= sl->next_select())
 {
 // 设置当前执行SEED部分或者CTE递归部分
 thd->lex->set_current_select(sl);

 // 根据LIMIT语句定义LIMIT相关执行部分
 if (set_limit(thd, sl))
 DBUG_RETURN(true);

 // 执行当前查询。这里由于不再重新打开表，所以对于临时表每次都会扫描到每次递归新产生的数据，也就是每次递归所使用到的新的SEED结果。
 sl->join->exec();
 status= sl->join->error != 0;

 // 如果包含UNION操作
 if (sl == union_distinct)
 {
 // This is UNION DISTINCT, so there should be a fake_select_lex
 DBUG_ASSERT(fake_select_lex != NULL);
 if (table->file->ha_disable_indexes(HA_KEY_SWITCH_ALL))
 DBUG_RETURN(true); /* purecov: inspected */
 table->no_keyread= 1;
 }
 if (status)
 DBUG_RETURN(true);

 if (union_result->flush())
 DBUG_RETURN(true); /* purecov: inspected */
 }
 } while (recursive_executor.more_iterations()); // 这里执行器判断是否需要继续递归

 ...
}
`

从上面的代码我们了解了CTE的具体工作过程，那么下面我们用具体的例子说明一下MySQL中CTE的执行过程。

`CREATE TABLE category(
 category_id INT AUTO_INCREMENT PRIMARY KEY,
 name VARCHAR(20) NOT NULL,
 parent INT DEFAULT NULL
);

INSERT INTO category VALUES(1,'ELECTRONICS',NULL),(2,'TELEVISIONS',1),(3,'TUBE',2),
 (4,'LCD',2),(5,'PLASMA',2),(6,'PORTABLE ELECTRONICS',1),(7,'MP3 PLAYERS',6),(8,'FLASH',7),
 (9,'CD PLAYERS',6),(10,'2 WAY RADIOS',6);
`

我们按电器种类广度遍历一下category表：

`mysql> WITH RECURSIVE cte AS
 -> (
 -> SELECT category_id, name, 0 AS depth FROM category WHERE parent IS NULL
 -> UNION ALL
 -> SELECT c.category_id, c.name, cte.depth+1 FROM category c JOIN cte ON
 -> cte.category_id=c.parent
 -> )
 -> SELECT * FROM cte ORDER BY depth;
+-------------+----------------------+-------+
| category_id | name | depth |
+-------------+----------------------+-------+
| 1 | ELECTRONICS | 0 |
| 2 | TELEVISIONS | 1 |
| 6 | PORTABLE ELECTRONICS | 1 |
| 5 | PLASMA | 2 |
| 7 | MP3 PLAYERS | 2 |
| 9 | CD PLAYERS | 2 |
| 10 | 2 WAY RADIOS | 2 |
| 3 | TUBE | 2 |
| 4 | LCD | 2 |
| 8 | FLASH | 3 |
+-------------+----------------------+-------+
10 rows in set (18.65 sec)

`
递归执行过程如下：

1. 查找parent IS NULL的第一种类别，我们可以得到ELECTRONICS
2. 接着查找parent == ELECTRONICS的第二类电器种类，可以看出我们可以得到TELEVISIONS和PORTABLE ELECTRONICS
3. 接着查找parent == TELEVISIONS 和 parent == PORTABLE ELECTRONICS，我们可以得到第三类电器分别是PLASMA，MP3 PLAYERS，CD PLAYERS，2 WAY RADIOS，TUBE，LCD
4. 接着继续查找属于第三类电器种类的产品，最后得到 FLASH。
5. 执行完毕。

综上所述，本篇文章简要的分析了MySQL Lab release中发布的CTE特性的实现方式，并对新增重点代码片段进行了介绍。希望能够帮助大家能对CTE的工作原理以及实现过程有个详细的了解。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)