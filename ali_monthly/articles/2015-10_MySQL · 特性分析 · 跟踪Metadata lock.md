# MySQL · 特性分析 · 跟踪Metadata lock

**Date:** 2015/10
**Source:** http://mysql.taobao.org/monthly/2015/10/02/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 10
 ](/monthly/2015/10)

 * 当期文章

 MySQL · 引擎特性 · InnoDB 全文索引简介
* MySQL · 特性分析 · 跟踪Metadata lock
* MySQL · 答疑解惑 · 索引过滤性太差引起CPU飙高分析
* PgSQL · 特性分析 · PG主备流复制机制
* MySQL · 捉虫动态 · start slave crash 诊断分析
* MySQL · 捉虫动态 · 删除索引导致表无法打开
* PgSQL · 特性分析 · PostgreSQL Aurora方案与DEMO
* TokuDB · 捉虫动态 · CREATE DATABASE 导致crash问题
* PgSQL · 特性分析 · pg_receivexlog工具解析
* MySQL · 特性分析 · MySQL权限存储与管理

 ## MySQL · 特性分析 · 跟踪Metadata lock 
 Author: lengxiang 

 ## 背景
MySQL 从5.5.3版本，对Metadata lock进行了调整，主要是MDL锁持有的周期从语句变成了事务， 其原因主要是解决两个问题：

**问题1: 破坏事务隔离级别**
在repeatable read的隔离级别下，多次的select语句执行过程中，会因为其它session的DDL语句，而导致select语句执行的结果不相同，破坏了RR的隔离级别。

**问题2: 破坏binlog的顺序**
在对表的DML过程中，会因为其它session的DDL语句，导致binlog里的event顺序在备库执行的结果和主库不一致。

从MySQL 5.5.3开始，MDL锁的持有周期变成了事务，解决了上面提到的两个问题，但在autocommit=off的情况下，也大大增加了阻塞的可能性。DBA对于阻塞的case，处理起来又比较麻烦，原因就是MDL锁的阻塞情况没有暴露明确的信息。

从MySQL 5.7.6开始，可以通过performance schema来查询MDL锁的持有情况。

在开始介绍5.7的跟踪Metadata lock之前， 小编还想讨论一下前面提到的这两个问题，在Oracle数据库中是如何处理的。

## Oracle的处理方式
首先，Oracle只实现了两种隔离级别，即read committed和serializable，我们来看下serializable级别下，怎么来处理问题1:

先看如下的case:

`session 1: session 2:
-- create table t1(id number);
-- insert into t1 values(1);
-- commit;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE; --
TEST/TEST@ORCL>select * from t1; --
 ID
----------
 1
1 row selected.
-- alter table t1 add col number;
TEST/TEST@ORCL>select * from t1; --

 ID COL
---------- ----------
 1
-- alter table t1 add col1 number default 10;
TEST/TEST@ORCL>select * from t1; --
 ID COL COL1
---------- ---------- ----------
 1
`

可以看到，虽然session是serializable隔离级别，但并没有产生阻塞的情况，Oracle保证了session1的多次select查询的返回结果是一样的，
但t1表数据字典的变化是马上可见的，这个也是符合serializable的要求的，因为隔离级别只定义了数据的可见性，而没有定义数据字典的可见性。

那MySQL能否不要MDL锁，来达到这样的效果？

答案是否定的，因为Oracle是堆表，alter的操作只更改了数据字典，数据记录没有发生变化，纵使加了default值，也是在原记录上进行的update，完全可以使用scn号来构建一致性读版本，这样就不会产生阻塞。
而MySQL是IOT表，alter的过程进行了表重建，无法完成read view的构建。

那我们再来看问题2，Oracle的处理方式:

对于redo日志，Oracle的处理方式和InnoDB的处理方式一致，也就是当使用redo的时候，日志的写入并不和事务的提交与否有必然的关系，也不用和提交的顺序保持一致。这一点就和binlog区别开来，也就是物理日志是可以避免使用逻辑日志(binlog)带来的问题。

MySQL如果要避免这两个问题，而不引入Metadata lock，可以有以下两个思路：

1. DDL只更改数据字典，行记录的变更在原记录上进行，这样能够实现多版本，也就是我们常说的在线加字段；
2. 使用物理redo日志，避免使用binlog。

这两种都会对现有的MySQL架构带来调整，仅供参考。

下面我们回来看下对5.7 MDL的tracing。

## MySQL 5.7

首先，打开metadata locks的tracing功能。

`mysql> UPDATE performance_schema.setup_consumers SET ENABLED = 'YES' WHERE NAME = 'global_instrumentation';
Query OK, 0 rows affected (0.00 sec)
Rows matched: 1 Changed: 0 Warnings: 0
mysql> UPDATE performance_schema.setup_instruments SET ENABLED = 'YES' WHERE NAME = 'wait/lock/metadata/sql/mdl';
Query OK, 0 rows affected (0.00 sec)
Rows matched: 1 Changed: 0 Warnings: 0
`

打开两个session，一个select，一个truncate。因为MDL锁的情况，select会阻塞truncate的操作。

session 1: 操作如下：

`mysql> set session autocommit=0;
Query OK, 0 rows affected (0.00 sec)
mysql> select @@autocommit, @@tx_isolation;
+--------------+----------------+
| @@autocommit | @@tx_isolation |
+--------------+----------------+
| 0 | READ-COMMITTED |
+--------------+----------------+
1 row in set (0.00 sec)
mysql> select * from t limit 1;
+----+------+
| id | val |
+----+------+
| 1 | 1 |
+----+------+
1 row in set (0.00 sec)
`

session 2: 操作如下：

`mysql> truncate table t;
`

结果看到的就是session2被阻塞， 接下来check一下performance schema的信息：

`mysql> select * from performance_schema.metadata_locks\G
*************************** 1. row ***************************
OBJECT_TYPE: TABLE
OBJECT_SCHEMA: test
OBJECT_NAME: t
OBJECT_INSTANCE_BEGIN: 140450128308592
LOCK_TYPE: SHARED_READ
LOCK_DURATION: TRANSACTION
LOCK_STATUS: GRANTED
SOURCE: sql_parse.cc:5585
OWNER_THREAD_ID: 27
OWNER_EVENT_ID: 17
*************************** 2. row ***************************
OBJECT_TYPE: GLOBAL
OBJECT_SCHEMA: NULL
OBJECT_NAME: NULL
OBJECT_INSTANCE_BEGIN: 140450195436144
LOCK_TYPE: INTENTION_EXCLUSIVE
LOCK_DURATION: STATEMENT
LOCK_STATUS: GRANTED
SOURCE: sql_base.cc:5224
OWNER_THREAD_ID: 30
OWNER_EVENT_ID: 8
*************************** 3. row ***************************
OBJECT_TYPE: SCHEMA
OBJECT_SCHEMA: test
OBJECT_NAME: NULL
OBJECT_INSTANCE_BEGIN: 140450195434272
LOCK_TYPE: INTENTION_EXCLUSIVE
LOCK_DURATION: TRANSACTION
LOCK_STATUS: GRANTED
SOURCE: sql_base.cc:5209
OWNER_THREAD_ID: 30
OWNER_EVENT_ID: 8
*************************** 4. row ***************************
OBJECT_TYPE: TABLE
OBJECT_SCHEMA: test
OBJECT_NAME: t
OBJECT_INSTANCE_BEGIN: 140450195434368
LOCK_TYPE: EXCLUSIVE
LOCK_DURATION: TRANSACTION
LOCK_STATUS: PENDING
SOURCE: sql_parse.cc:5585
OWNER_THREAD_ID: 30
OWNER_EVENT_ID: 8
*************************** 5. row ***************************
OBJECT_TYPE: TABLE
OBJECT_SCHEMA: performance_schema
OBJECT_NAME: metadata_locks
OBJECT_INSTANCE_BEGIN: 140450128262384
LOCK_TYPE: SHARED_READ
LOCK_DURATION: TRANSACTION
LOCK_STATUS: GRANTED
SOURCE: sql_parse.cc:5585
OWNER_THREAD_ID: 27
OWNER_EVENT_ID: 18
5 rows in set (0.00 sec)
`

如上所示，在t表上，持有一个SHARE_READ lock，而且还有一个EXCULSIVE lock请求是pending状态，也就是我们被阻塞的session 2。

在5.7之前，我们可以通过show processlist，来查看MDL阻塞的情况，但无法获取session 1的信息:

`mysql> SELECT OBJECT_TYPE, OBJECT_SCHEMA, OBJECT_NAME, LOCK_TYPE, LOCK_STATUS, THREAD_ID, PROCESSLIST_ID, PROCESSLIST_INFO FROM performance_schema.metadata_locks INNER JOIN performance_schema.threads ON THREAD_ID = OWNER_THREAD_ID WHERE PROCESSLIST_ID <> CONNECTION_ID();
+-------------+---------------+-------------+---------------------+-------------+-----------+----------------+------------------+
| OBJECT_TYPE | OBJECT_SCHEMA | OBJECT_NAME | LOCK_TYPE | LOCK_STATUS | THREAD_ID | PROCESSLIST_ID | PROCESSLIST_INFO |
+-------------+---------------+-------------+---------------------+-------------+-----------+----------------+------------------+
| GLOBAL | NULL | NULL | INTENTION_EXCLUSIVE | GRANTED | 30 | 8 | truncate table t |
| SCHEMA | test | NULL | INTENTION_EXCLUSIVE | GRANTED | 30 | 8 | truncate table t |
| TABLE | test | t | EXCLUSIVE | PENDING | 30 | 8 | truncate table t |
+-------------+---------------+-------------+---------------------+-------------+-----------+----------------+------------------+
3 rows in set (0.00 sec)
mysql> show processlist;
+----+------+-----------+------+---------+------+---------------------------------+------------------+
| Id | User | Host | db | Command | Time | State | Info |
+----+------+-----------+------+---------+------+---------------------------------+------------------+
| 5 | root | localhost | test | Query | 0 | starting | show processlist |
| 8 | root | localhost | test | Query | 50 | Waiting for table metadata lock | truncate table t |
+----+------+-----------+------+---------+------+---------------------------------+------------------+
2 rows in set (0.00 sec)
`

接下来当事务提交了后，释放MDL锁再查询，就看不到MDL锁的信息了。

`mysql> commit;
Query OK, 0 rows affected (0.00 sec)
mysql> SELECT OBJECT_TYPE, OBJECT_SCHEMA, OBJECT_NAME, LOCK_TYPE, LOCK_STATUS, THREAD_ID, PROCESSLIST_ID, PROCESSLIST_INFO FROM performance_schema.metadata_locks INNER JOIN performance_schema.threads ON THREAD_ID = OWNER_THREAD_ID WHERE PROCESSLIST_ID <> CONNECTION_ID();
Empty set (0.01 sec)
mysql> select * from t;
Empty set (0.00 sec)
`

MySQL 5.7可以通过performance schema来检索MDL锁阻塞情况，方便DBA来诊断问题。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)