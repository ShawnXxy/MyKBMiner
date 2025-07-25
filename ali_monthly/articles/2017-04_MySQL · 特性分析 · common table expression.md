# MySQL · 特性分析 · common table expression

**Date:** 2017/04
**Source:** http://mysql.taobao.org/monthly/2017/04/05/
**Images:** 4 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 04
 ](/monthly/2017/04)

 * 当期文章

 MySQL · 源码分析 · MySQL 半同步复制数据一致性分析
* MYSQL · 新特性 · MySQL 8.0对Parser所做的改进
* MySQL · 引擎介绍 · Sphinx源码剖析（二）
* PgSQL · 特性分析 · checkpoint机制浅析
* MySQL · 特性分析 · common table expression
* PgSQL · 应用案例 · 逻辑订阅给业务架构带来了什么？
* MSSQL · 应用案例 · 基于内存优化表的列存储索引分析Web Access Log
* TokuDB · 捉虫动态 · MRR 导致查询失败
* HybridDB · 稳定性 · HybridDB如何优雅的处理Out Of Memery问题
* MySQL · 捉虫动态 · 5.7 mysql_upgrade 元数据锁等待

 ## MySQL · 特性分析 · common table expression 
 Author: 张远 

 ## common table expression

Common table expression简称CTE，由[SQL:1999标准](https://mariadb.com/kb/en/sql-99/recursive-unions/)引入，
目前支持CTE的数据库有Teradata, DB2, Firebird, Microsoft SQL Server, Oracle (with recursion since 11g release 2), PostgreSQL (since 8.4), MariaDB (since 10.2), SQLite (since 3.8.3), HyperSQL and H2 (experimental), MySQL8.0.

CTE的语法如下：

`WITH [RECURSIVE] with_query [, ...]
SELECT...

with_query:
query_name [ (column_name [,...]) ] AS (SELECT ...)
`

以下图示来自[MariaDB](https://mariadb.com/kb/en/mariadb/common-table-expressions-overview/)

Non-recursive CTEs

![screenshot.png](.img/67d3dbe4c074_c22077bc01d2f92a104e58c1dffcc95e.png)

Recursive CTEs

![screenshot.png](.img/9a285886f0ad_83b2eea5ca74f33fd950d5f4674005c8.png)

## CTE的使用

* CTE使语句更加简洁

例如以下两个语句表达的是同一语义，使用CTE比未使用CTE的嵌套查询更简洁明了。

1) 使用嵌套子查询

`SELECT MAX(txt), MIN(txt)
FROM
(
 SELECT concat(cte2.txt, cte3.txt) as txt
 FROM
 (
 SELECT CONCAT(cte1.txt,'is a ') as txt
 FROM
 (
 SELECT 'This ' as txt
 ) as cte1
 ) as cte2,
 (
 SELECT 'nice query' as txt
 UNION
 SELECT 'query that rocks'
 UNION
 SELECT 'query'
 ) as cte3
) as cte4;
`
2) 使用CTE

`WITH cte1(txt) AS (SELECT "This "),
 cte2(txt) AS (SELECT CONCAT(cte1.txt,"is a ") FROM cte1),
 cte3(txt) AS (SELECT "nice query" UNION
 SELECT "query that rocks" UNION
 SELECT "query"),
 cte4(txt) AS (SELECT concat(cte2.txt, cte3.txt) FROM cte2, cte3)
SELECT MAX(txt), MIN(txt) FROM cte4;
`

* CTE 可以进行树形查询

初始化这颗树

```
create table t1(id int, value char(10), parent_id int);
insert into t1 values(1, 'A', NULL);
insert into t1 values(2, 'B', 1);
insert into t1 values(3, 'C', 1);
insert into t1 values(4, 'D', 1);
insert into t1 values(5, 'E', 2);
insert into t1 values(6, 'F', 2);
insert into t1 values(7, 'G', 4);
insert into t1 values(8, 'H', 6);

```

1) 层序遍历

`with recursive cte as (
 select id, value, 0 as level from t1 where parent_id is null
 union all
 select t1.id, t1.value, cte.level+1 from cte join t1 on t1.parent_id=cte.id)
select * from cte;
+------+-------+-------+
| id | value | level |
+------+-------+-------+
| 1 | A | 0 |
| 2 | B | 1 |
| 3 | C | 1 |
| 4 | D | 1 |
| 5 | E | 2 |
| 6 | F | 2 |
| 7 | G | 2 |
| 8 | H | 3 |
+------+-------+-------+
`

2) 深度优先遍历

`with recursive cte as (
 select id, value, 0 as level, CAST(id AS CHAR(200)) AS path from t1 where parent_id is null
 union all
 select t1.id, t1.value, cte.level+1, CONCAT(cte.path, ",", t1.id) from cte join t1 on t1.parent_id=cte.id)
select * from cte order by path;
+------+-------+-------+---------+
| id | value | level | path |
+------+-------+-------+---------+
| 1 | A | 0 | 1 |
| 2 | B | 1 | 1,2 |
| 5 | E | 2 | 1,2,5 |
| 6 | F | 2 | 1,2,6 |
| 8 | H | 3 | 1,2,6,8 |
| 3 | C | 1 | 1,3 |
| 4 | D | 1 | 1,4 |
| 7 | G | 2 | 1,4,7 |
+------+-------+-------+---------+
`

## Oracle
Oracle从9.2才开始支持CTE, 但只支持non-recursive with, 直到Oracle 11.2才完全支持CTE。但oracle 之前就支持connect by 的树形查询，recursive with 语句可以与connect by语句相互转化。 一些相互转化案例可以参考[这里](https://oracle-base.com/articles/11g/recursive-subquery-factoring-11gr2).

Oracle recursive with 语句不需要指定recursive关键字，可以自动识别是否recursive.

Oracle 还支持CTE相关的hint,

`WITH dept_count AS (
 SELECT /*+ MATERIALIZE */ deptno, COUNT(*) AS dept_count
 FROM emp
 GROUP BY deptno)
SELECT ...

WITH dept_count AS (
 SELECT /*+ INLINE */ deptno, COUNT(*) AS dept_count
 FROM emp
 GROUP BY deptno)
SELECT ...
`
“MATERIALIZE”告诉优化器产生一个全局的临时表保存结果，多次引用CTE时直接访问临时表即可。而”INLINE”则表示每次需要解析查询CTE。

## PostgreSQL

PostgreSQL从8.4开始支持CTE，PostgreSQL还扩展了CTE的功能， CTE的query中支持DML语句，例如

`create table t1 (c1 int, c2 char(10));
 insert into t1 values(1,'a'),(2,'b');
 select * from t1;
 c1 | c2
----+----
 1 | a
 2 | b

 WITH cte AS (
 UPDATE t1 SET c1= c1 * 2 where c1=1
 RETURNING *
 )
 SELECT * FROM cte; //返回更新的值
 c1 | c2
----+------------
 2 | a

 truncate table t1;
 insert into t1 values(1,'a'),(2,'b');
 WITH cte AS (
 UPDATE t1 SET c1= c1 * 2 where c1=1
 RETURNING *
 )
 SELECT * FROM t1;//返回原值
 c1 | c2
----+------------
 1 | a
 2 | b

 truncate table t1;
 insert into t1 values(1,'a'),(2,'b');
 WITH cte AS (
 DELETE FROM t1
 WHERE c1=1
 RETURNING *
 )
 SELECT * FROM cte;//返回删除的行
 c1 | c2
----+------------
 1 | a

 truncate table t1;
 insert into t1 values(1,'a'),(2,'b');
 WITH cte AS (
 DELETE FROM t1
 WHERE c1=1
 RETURNING *
 )
 SELECT * FROM t1;//返回原值
 c1 | c2
----+------------
 1 | a
 2 | b
(2 rows)

`

## MariaDB
MariaDB从10.2开始支持CTE。10.2.1 支持non-recursive CTE, 10.2.2开始支持recursive CTE。 目前的GA的版本是10.1.

## MySQL
MySQL从8.0开始支持完整的CTE。MySQL8.0还在development
阶段，RC都没有，GA还需时日。

## AliSQL

AliSQL基于mariadb10.2， port了no-recursive CTE的实现，此功能近期会上线。

以下从源码主要相关函数简要介绍其实现，

//解析识别with table引用

find_table_def_in_with_clauses

//检查依赖关系，比如不能重复定义with table名字

With_clause::check_dependencies

// 为每个引用clone一份定义

With_element::clone_parsed_spec

//替换with table指定的列名

With_element::rename_columns_of_derived_unit

此实现对于多次引用CTE，CTE会解析多次，因此此版本CTE有简化SQL的作用，但效率上没有效提高。

`select count(*) from t1 where c2 !='z';
+----------+
| count(*) |
+----------+
| 65536 |
+----------+
1 row in set (0.25 sec)

//从执行时间来看是进行了3次全表扫描
 with t as (select count(*) from t1 where c2 !='z')
 select * from t union select * from t union select * from t;
+----------+
| count(*) |
+----------+
| 65536 |
+----------+
1 row in set (0.59 sec)

 select count(*) from t1 where c2 !='z'
 union
 select count(*) from t1 where c2 !='z'
 union
 select count(*) from t1 where c2 !='z';
+----------+
| count(*) |
+----------+
| 65536 |
+----------+
1 row in set (0.57 sec)

 explain with t as (select count(*) from t1 where c2 !='z')
 -> select * from t union select * from t union select * from t;
+------+-----------------+--------------+------+---------------+------+---------+------+-------+-------------+
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
+------+-----------------+--------------+------+---------------+------+---------+------+-------+-------------+
| 1 | PRIMARY | <derived2> | ALL | NULL | NULL | NULL | NULL | 65536 | |
| 2 | SUBQUERY | t1 | ALL | NULL | NULL | NULL | NULL | 65536 | Using where |
| 3 | RECURSIVE UNION | <derived5> | ALL | NULL | NULL | NULL | NULL | 65536 | |
| 5 | SUBQUERY | t1 | ALL | NULL | NULL | NULL | NULL | 65536 | Using where |
| 4 | RECURSIVE UNION | <derived6> | ALL | NULL | NULL | NULL | NULL | 65536 | |
| 6 | SUBQUERY | t1 | ALL | NULL | NULL | NULL | NULL | 65536 | Using where |
| NULL | UNION RESULT | <union1,3,4> | ALL | NULL | NULL | NULL | NULL | NULL | |
+------+-----------------+--------------+------+---------------+------+---------+------+-------+-------------+
7 rows in set (0.00 sec)

 explain select count(*) from t1 where c2 !='z'
 union
 select count(*) from t1 where c2 !='z'
 union
 select count(*) from t1 where c2 !='z';
+------+--------------+--------------+------+---------------+------+---------+------+-------+-------------+
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
+------+--------------+--------------+------+---------------+------+---------+------+-------+-------------+
| 1 | PRIMARY | t1 | ALL | NULL | NULL | NULL | NULL | 65536 | Using where |
| 2 | UNION | t1 | ALL | NULL | NULL | NULL | NULL | 65536 | Using where |
| 3 | UNION | t1 | ALL | NULL | NULL | NULL | NULL | 65536 | Using where |
| NULL | UNION RESULT | <union1,2,3> | ALL | NULL | NULL | NULL | NULL | NULL | |
+------+--------------+--------------+------+---------------+------+---------+------+-------+-------------+
4 rows in set (0.00 sec)

`
以下是MySQL8.0 只扫描一次的执行计划

`mysql> explain select count(*) from t1 where c2 !='z' union select count(*) from t1 where c2 !='z' union select count(*) from t1 where c2 !='z';
+----+--------------+--------------+------------+------+---------------+------+---------+------+-------+----------+-----------------+
| id | select_type | table | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
+----+--------------+--------------+------------+------+---------------+------+---------+------+-------+----------+-----------------+
| 1 | PRIMARY | t1 | NULL | ALL | NULL | NULL | NULL | NULL | 62836 | 90.00 | Using where |
| 2 | UNION | t1 | NULL | ALL | NULL | NULL | NULL | NULL | 62836 | 90.00 | Using where |
| 3 | UNION | t1 | NULL | ALL | NULL | NULL | NULL | NULL | 62836 | 90.00 | Using where |
| NULL | UNION RESULT | <union1,2,3> | NULL | ALL | NULL | NULL | NULL | NULL | NULL | NULL | Using temporary |
+----+--------------+--------------+------------+------+---------------+------+---------+------+-------+----------+-----------------+
4 rows in set, 1 warning (0.00 sec)
`

以下是PostgreSQL9.4 只扫描一次的执行计划

`postgres=# explain with t as (select count(*) from t1 where c2 !='z')
postgres-# select * from t union select * from t union select * from t;
 HashAggregate (cost=391366.28..391366.31 rows=3 width=8)
 Group Key: t.count
 CTE t
 -> Aggregate (cost=391366.17..391366.18 rows=1 width=0)
 -> Seq Scan on t1 (cost=0.00..384392.81 rows=2789345 width=0)
 Filter: ((c2)::text <> 'z'::text)
 -> Append (cost=0.00..0.09 rows=3 width=8)
 -> CTE Scan on t (cost=0.00..0.02 rows=1 width=8)
 -> CTE Scan on t t_1 (cost=0.00..0.02 rows=1 width=8)
 -> CTE Scan on t t_2 (cost=0.00..0.02 rows=1 width=8)
`

AliSQL还有待改进。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)