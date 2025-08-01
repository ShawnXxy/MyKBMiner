# MySQL · 最佳实践 · RDS MySQL 8.0 语句级并发控制

**Date:** 2019/06
**Source:** http://mysql.taobao.org/monthly/2019/06/02/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2019 / 06
 ](/monthly/2019/06)

 * 当期文章

 MySQL · 引擎特性 · 安全及权限改进相关
* MySQL · 最佳实践 · RDS MySQL 8.0 语句级并发控制
* CloudDBA · 最佳实践 · Performance Insights
* PgSQL · 应用案例 · 学生为什么应该学PG
* MongoDB · 引擎特性 · 4.2 新特性解读
* PgSQL · 答疑解惑 · 垃圾回收、膨胀、多版本管理、存储引擎
* MySQL · 引擎特性 · 说说InnoDB Log System的隐藏参数
* MySQL · 引擎特性 · CHECK CONSTRAINT
* PgSQL · 应用案例 · 如何修改PostgreSQL分区表分区范围
* PgSQL · 应用案例 · 什么情况下可能表膨胀

 ## MySQL · 最佳实践 · RDS MySQL 8.0 语句级并发控制 
 Author: 

 ## 背景

为了应对突发的数据库请求流量，资源消耗过载的语句访问，SQL 访问模型的变化， 并保持 MySQL 实例持续稳定运行，AliSQL 版本设计了基于语句规则的并发控制，Statement Concurrency Control，以下简称 CCL，有效控制匹配某种规则的并发度，并提供了一组工具包（DBMS_CCL package) 方便快捷使用。

## 规则设计

CCL 规则一共定义了三个维度的特征：
1）SQL command
根据 statement 的类型，例如 ‘SELECT’, ‘UPDATE’, ‘INSERT’, ‘DELETE’;
2)  Object
根据 statement 操作的对象进行控制， 例如 TABLE，VIEW;
3）keywords
根据 statement 语句的关键字进行控制;

CCL 根据规则的定义，设计了一个系统表，mysql.concurrency_control 持久化保存 CCL rule：

### Concurrency_control

`CREATE TABLE `concurrency_control` (
 `Id` bigint(20) NOT NULL AUTO_INCREMENT,
 `Type` enum('SELECT','UPDATE','INSERT','DELETE') NOT NULL DEFAULT 'SELECT',
 `Schema_name` varchar(64) COLLATE utf8_bin DEFAULT NULL,
 `Table_name` varchar(64) COLLATE utf8_bin DEFAULT NULL,
 `Concurrency_count` bigint(20) DEFAULT NULL,
 `Keywords` text COLLATE utf8_bin,
 `State` enum('N','Y') NOT NULL DEFAULT 'Y',
 `Ordered` enum('N','Y') NOT NULL DEFAULT 'N',
 PRIMARY KEY (`Id`)
 ) /*!50100 TABLESPACE `mysql` */ ENGINE=InnoDB 
DEFAULT CHARSET=utf8 COLLATE=utf8_bin
STATS_PERSISTENT=0 COMMENT='Concurrency control'
`

### COLUMNS

* “Type”
* 用来定义 SQL command
* “Schema_name” && “Table_name”
* 用来定义 Object
* “Keywords”
* 用来定义关键字，可使用 ‘;’ 分隔符多个关键字
* “Concurrency_count”
* 用来定义并发度
* “State”
* 表示这条规则是否 active
* “Ordered”
* 表示keywords中多个关键字是否按顺序匹配

用户可以直接操作这个表来定义规则，也可以使用 DBMS_CCL 工具包来操作 CCL rule。

### 管理接口

为了便捷的管理 CCL rule，AliSQL 在 DBMS_CCL package 中定义了四个 native procedure 来管理；

**1）Add CCL rule**
dbms_ccl.add_ccl_rule(type=>, schema=>, table=>, Concurrency_count=>, keywords=>);

增加规则（包括表和内存）例如：

`1. 增加 SELECT 语句的并发度为 10；
mysql> call dbms_ccl.add_ccl_rule('SELECT', '', '', 10, '');
Query OK, 0 rows affected (0.00 sec)

2. 增加 SELECT 语句，并在语句中出现关键字 key1 的并发度为 20
mysql> call dbms_ccl.add_ccl_rule('SELECT', '', '', 20, 'key1');
Query OK, 0 rows affected (0.00 sec)

3. 增加 test.t 表的 SELECT 语句的并发读为 20；
mysql> call dbms_ccl.add_ccl_rule('SELECT', 'test', 't', 30, '');
Query OK, 0 rows affected (0.00 sec)
`

规则的匹配按照 3 > 2 > 1 的优先级顺序进行匹配。

**2）Delete CCL rule**
dbms_ccl.del_ccl_rule(rule_id=> );

删除规则（包括内存和表中）例如：

` 1. 删除 rule id = 15 的 CCL rule
 mysql> call dbms_ccl.del_ccl_rule(15);
 Query OK, 0 rows affected (0.01 sec)

 2. 如果删除的rule 不存在，语句报相应的 warning
 mysql> call dbms_ccl.del_ccl_rule(100);
 Query OK, 0 rows affected, 2 warnings (0.00 sec)

 mysql> show warnings;
 +---------+------+----------------------------------------------------+
 | Level | Code | Message |
 +---------+------+----------------------------------------------------+
 | Warning | 7514 | Concurrency control rule 100 is not found in table |
 | Warning | 7514 | Concurrency control rule 100 is not found in cache |
 +---------+------+----------------------------------------------------+
`

**3) Show CCL rule**
dbms_ccl.show_ccl_rule();

展示在内存中 active  rule 的情况，例如：

` mysql> call dbms_ccl.show_ccl_rule();
 +------+--------+--------+-------+-------+-------+-------------------+---------+---------+----------+----------+
 | ID | TYPE | SCHEMA | TABLE | STATE | ORDER | CONCURRENCY_COUNT | MATCHED | RUNNING | WAITTING | KEYWORDS |
 +------+--------+--------+-------+-------+-------+-------------------+---------+---------+----------+----------+
 | 17 | SELECT | test | t | Y | N | 30 | 0 | 0 | 0 | |
 | 16 | SELECT | | | Y | N | 20 | 0 | 0 | 0 | key1 |
 | 18 | SELECT | | | Y | N | 10 | 0 | 0 | 0 | |
 +------+--------+--------+-------+-------+-------+-------------------+---------+---------+----------+----------+
`

除了 rule 本身的属性之外，增加了三个数字统计：

1）MATCHED
规则匹配成功次数
2）RUNNING
在此规则下，正在 run 的线程数
3）WAITING
在此规则下，正在 wait的线程数

** 4）Flush CCL rule**
dbms_ccl.flush_ccl_rule();

如果直接操作了concurrency_control table 修改规则， 不能立即生效，可以调用 flush，重新生效。例如：

` mysql> update mysql.concurrency_control set CONCURRENCY_COUNT = 15 where id = 18;
 Query OK, 1 row affected (0.00 sec)
 Rows matched: 1 Changed: 1 Warnings: 0

 mysql> call dbms_ccl.flush_ccl_rule();
 Query OK, 0 rows affected (0.00 sec)
`

### 压力测试

**测试场景**

**1）设计三条规则**

 **Rule** 1：对 sbtest1 表 应用 Object rule 控制

` call dbms_ccl.add_ccl_rule('SELECT', 'test', 'sbtest1', 3, '');
`

**Rule** 2: 对sbtest2 表 应用 keyword rule 控制

` call dbms_ccl.add_ccl_rule('SELECT', '', '', 2, 'sbtest2');
`

**Rule** 3: 对sbtest3 表 应用 SQL command 控制

` call dbms_ccl.add_ccl_rule('SELECT', '', '', 2, '');
`

**2）使用 sysbench 进行测试**

* 64 threads
* 4 tables
* select.lua

查看规则并发使用情况，可以到到 running 和 waiting 的数量：

` mysql> call dbms_ccl.show_ccl_rule();
 +------+--------+--------+---------+-------+-------+-------------------+---------+---------+----------+----------+
 | ID | TYPE | SCHEMA | TABLE | STATE | ORDER | CONCURRENCY_COUNT | MATCHED | RUNNING | WAITTING | KEYWORDS |
 +------+--------+--------+---------+-------+-------+-------------------+---------+---------+----------+----------+
 | 20 | SELECT | test | sbtest1 | Y | N | 3 | 389 | 3 | 9 | |
 | 21 | SELECT | | | Y | N | 2 | 375 | 2 | 14 | sbtest2 |
 | 22 | SELECT | | | Y | N | 2 | 519 | 2 | 34 | |
 +------+--------+--------+---------+-------+-------+-------------------+---------+---------+----------+----------+
 3 rows in set (0.00 sec)
`

查看线程运行情况： 大部分处在 Concurrency control waitting 状态。

```
 mysql> show processlist;
 +-----+-----------------+-----------------+------+---------+------+------------------------------+--------------------------------------+
 | Id | User | Host | db | Command | Time | State | Info |
 +-----+-----------------+-----------------+------+---------+------+------------------------------+--------------------------------------+
 | 72 | root | localhost:33601 | NULL | Query | 0 | starting | show processlist |
 | 171 | u1 | localhost:60120 | test | Query | 2 | Concurrency control waitting | SELECT pad FROM sbtest3 WHERE id=51 |
 | 172 | u1 | localhost:60128 | test | Query | 5 | Concurrency control waitting | SELECT pad FROM sbtest4 WHERE id=35 |
 | 174 | u1 | localhost:60385 | test | Query | 4 | Concurrency control waitting | SELECT pad FROM sbtest3 WHERE id=54 |
 | 178 | u1 | localhost:60136 | test | Query | 12 | Concurrency control waitting | SELECT pad FROM sbtest4 WHERE id=51 |
 | 179 | u1 | localhost:60149 | test | Query | 5 | Concurrency control waitting | SELECT pad FROM sbtest2 WHERE id=51 |
 | 182 | u1 | localhost:60124 | test | Query | 1 | Concurrency control waitting | SELECT pad FROM sbtest4 WHERE id=51 |
 | 183 | u1 | localhost:60371 | test | Query | 5 | User sleep | SELECT pad FROM sbtest2 WHERE id=51 |
 | 184 | u1 | localhost:60133 | test | Query | 4 | Concurrency control waitting | SELECT pad FROM sbtest3 WHERE id=51 |
 | 190 | u1 | localhost:60406 | test | Query | 5 | Concurrency control waitting | SELECT pad FROM sbtest3 WHERE id=51 |
 | 191 | u1 | localhost:60402 | test | Query | 1 | Concurrency control waitting | SELECT pad FROM sbtest4 WHERE id=51 |
 | 192 | u1 | localhost:60131 | test | Query | 2 | User sleep | SELECT pad FROM sbtest1 WHERE id=51 |

 ......

```

### 使用规则和风险

1. Concurrency_control 被设计成不产生 BINLOG，所以对于 CCL 的操作只影响当前实例。
2. 对于 DML 的并发控制，可能存在事务锁死锁的情况， 除了 CCL 提供了超时机制，
同时等待中的线程也会响应事务超时和线程 KILL 操作，以应对死锁可能。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)