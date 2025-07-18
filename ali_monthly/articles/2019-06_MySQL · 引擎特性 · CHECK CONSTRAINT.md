# MySQL · 引擎特性 · CHECK CONSTRAINT

**Date:** 2019/06
**Source:** http://mysql.taobao.org/monthly/2019/06/08/
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

 ## MySQL · 引擎特性 · CHECK CONSTRAINT 
 Author: weixiang 

 即使MySQL8.0已经GA了，官方仍然在向其中增加新的功能，比如在最新的MySQL8.0.16版本中，增加了一个众望所归的功能：CHECK CONSTRAINT，也就是说可以自动对写入的数据进行约束检查。这个特性的worklog号码为929，已经是十几年前的需求了，终于在8.0实现了。（实际上这也是标准SQL功能，像PostgreSQL, Oracle等都有这个功能）

本文简单阐述下其使用方式以及相关实现

### 如何使用

其实在之前的版本中，已经实现了标准语法 CHECK(expr), 但是实际上是被忽略掉的，在新版本中，可以在列或者表上做一些约束条件，语法如下:

1.如下是表级约束

`[CONSTRAINT [symbol]] CHECK (expr) [[NOT] ENFORCED]
`

其中symbol用来命名约束条件的唯一名字，如果没有指定的话，Mysql也会自动生成约束名，但要注意，在同一个库下面，约束名字不能重复，必须具有唯一性
expr是一个表达式，结果为bool类型
enforced是默认选定的，你也可以手动选定，表示必须满足约束条件才允许写入， 但如果选择NOT ENFORCED的话，则表示约束条件虽然创建了，但并不强制

2.如下是列级别约束, 可以在创建列的时候同时指定约束条件

`CHECK (expr)
`

3.示例:

我们举个简单的例子：

` root@test 10:23:42>CREATE TABLE t1
 -> (
 -> CHECK (c1 <> c2),
 -> c1 INT CHECK (c1 > 10),
 -> c2 INT CONSTRAINT c2_positive CHECK (c2 > 0),
 -> c3 INT CHECK (c3 < 100),
 -> CONSTRAINT c1_nonzero CHECK (c1 <> 0),
 -> CHECK (c1 > c3)
 -> );
 Query OK, 0 rows affected (0.01 sec)

 root@test 10:23:58>SHOW CREATE TABLE t1\G
 *************************** 1. row ***************************
 Table: t1
 Create Table: CREATE TABLE `t1` (
 `c1` int(11) DEFAULT NULL,
 `c2` int(11) DEFAULT NULL,
 `c3` int(11) DEFAULT NULL,
 CONSTRAINT `c1_nonzero` CHECK ((`c1` <> 0)),
 CONSTRAINT `c2_positive` CHECK ((`c2` > 0)),
 CONSTRAINT `t1_chk_1` CHECK ((`c1` <> `c2`)),
 CONSTRAINT `t1_chk_2` CHECK ((`c1` > 10)),
 CONSTRAINT `t1_chk_3` CHECK ((`c3` < 100)),
 CONSTRAINT `t1_chk_4` CHECK ((`c1` > `c3`))
 ) ENGINE=InnoDB DEFAULT CHARSET=latin1
 1 row in set (0.01 sec)

 # 违反了约束条件t1_chk_1, 即c1 != c2
 root@test 10:24:07>INSERT INTO t1 VALUES (1,1,1);
 ERROR 3819 (HY000): Check constraint 't1_chk_1' is violated.
`

既然约束名必须唯一，那如果我们把t1 rename成t2, 再新建一个t1会怎么样呢 ?

` root@test 10:25:35>rename table t1 to t2;
 Query OK, 0 rows affected (0.01 sec)

 root@test 10:25:37>show create table t2\G
 *************************** 1. row ***************************
 Table: t2
 Create Table: CREATE TABLE `t2` (
 `c1` int(11) DEFAULT NULL,
 `c2` int(11) DEFAULT NULL,
 `c3` int(11) DEFAULT NULL,
 CONSTRAINT `c1_nonzero` CHECK ((`c1` <> 0)),
 CONSTRAINT `c2_positive` CHECK ((`c2` > 0)),
 CONSTRAINT `t2_chk_1` CHECK ((`c1` <> `c2`)),
 CONSTRAINT `t2_chk_2` CHECK ((`c1` > 10)),
 CONSTRAINT `t2_chk_3` CHECK ((`c3` < 100)),
 CONSTRAINT `t2_chk_4` CHECK ((`c1` > `c3`))
 ) ENGINE=InnoDB DEFAULT CHARSET=latin1
 1 row in set (0.00 sec)

 root@test 10:25:40>create table t1 like t2;
 Query OK, 0 rows affected (0.00 sec)

 root@test 10:25:51>show create table t1;
 +-------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
 | Table | Create Table |
 +-------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
 | t1 | CREATE TABLE `t1` (
 `c1` int(11) DEFAULT NULL,
 `c2` int(11) DEFAULT NULL,
 `c3` int(11) DEFAULT NULL,
 CONSTRAINT `t1_chk_1` CHECK ((`c1` <> 0)),
 CONSTRAINT `t1_chk_2` CHECK ((`c2` > 0)),
 CONSTRAINT `t1_chk_3` CHECK ((`c1` <> `c2`)),
 CONSTRAINT `t1_chk_4` CHECK ((`c1` > 10)),
 CONSTRAINT `t1_chk_5` CHECK ((`c3` < 100)),
 CONSTRAINT `t1_chk_6` CHECK ((`c1` > `c3`))
 ) ENGINE=InnoDB DEFAULT CHARSET=latin1 |
`

可以看到t1 rename成t2后，其中自动生成的约束名被更改成t2_前缀的。而创建的t2 like t1表， 依然完整的继承了原来t1表的约束条件，但命名全部改成了t1_前缀的。

你也可以通过alter table的方式增加或删除约束：

` root@test 10:45:08>ALTER TABLE t1 ADD CONSTRAINT check ((c1 + c2 + c3) > 1000);
 Query OK, 0 rows affected (0.02 sec)
 Records: 0 Duplicates: 0 Warnings: 0

 root@test 10:45:39>INSERT INTO t1 values (1999, 50, 90);
 Query OK, 1 row affected (0.00 sec)

 root@test 10:46:23>INSERT INTO t1 values (100, 50, 90);
 ERROR 3819 (HY000): Check constraint 't1_chk_7' is violated.

 root@test 10:46:53>ALTER TABLE t1 DROP CHECK t1_chk_7;
 Query OK, 0 rows affected (0.01 sec)
 Records: 0 Duplicates: 0 Warnings: 0

 root@test 10:47:17>INSERT INTO t1 values (100, 50, 90);
 Query OK, 1 row affected (0.00 sec)
`

InnoDB新增了一个data dictionary表check_constrains, 你可以从information_schema表下面查询：

` root@(none) 10:54:05>SELECT * FROM INFORMATION_SCHEMA.CHECK_CONSTRAINTS;
 +--------------------+-------------------+-----------------+----------------+
 | CONSTRAINT_CATALOG | CONSTRAINT_SCHEMA | CONSTRAINT_NAME | CHECK_CLAUSE |
 +--------------------+-------------------+-----------------+----------------+
 | def | test | t2_chk_1 | (`c1` <> `c2`) |
 | def | test | t2_chk_2 | (`c1` > 10) |
 | def | test | c2_positive | (`c2` > 0) |
 | def | test | t2_chk_3 | (`c3` < 100) |
 | def | test | c1_nonzero | (`c1` <> 0) |
 | def | test | t2_chk_4 | (`c1` > `c3`) |
 | def | test | t1_chk_1 | (`c1` <> 0) |
 | def | test | t1_chk_2 | (`c2` > 0) |
 | def | test | t1_chk_3 | (`c1` <> `c2`) |
 | def | test | t1_chk_4 | (`c1` > 10) |
 | def | test | t1_chk_5 | (`c3` < 100) |
 | def | test | t1_chk_6 | (`c1` > `c3`) |
 +--------------------+-------------------+-----------------+----------------+
12 rows in set (0.00 sec)
`

### 相关实现

1.新增代码文件

sql/sql_check_constraint.cc

sql/dd/impl/system_views/check_constraints.cc

sql/dd/impl/types/check_constraint_impl.cc

2.表达式定义及存储

InnoDB新增了一个数据词典表mysql.check_constraints用来存储所有的约束条件，表的定义在文件`sql/dd/impl/tables/check_constraints.cc`中, 相关堆栈

` mysql_execute_command
 |-> Sql_cmd_create_table::execute 
 |-> mysql_create_table
 |-> prepare_check_constraints_for_create
 |--> generate_check_constraint_name //自动生成constraint名字
 |-> mysql_create_table_no_lock -> create_table_impl -> rea_create_base_table 
 |-> dd::cache::Dictionary_client::store
 .....
 |-> dd::Collection<dd::Check_constraint*>::store_items
 |-> d::Check_constraint_impl::store // 存储到数据词典表中
`

3.载入内存及显示

先存储到dd::Table中，当打开table share时，拷贝到TABLE_SHARE::check_constraint_share_list

` open_table
 |-> get_table_share_with_discover
 |-> get_table_share
 |-> dd::cache::Dictionary_client::acquire //去dd获取uncached的表上的定义，存储到dd:Table中
 |-> open_table_def // 构建table share表定义
 |--> fill_check_constraints_from_dd // 将约束条件拷贝到table share中
`

在每次实例化线程可操作的TABLE对象时，再从table share中读取，存储到TABLE::table_check_constraint_list中
参考函数：`open_table_from_share`

4.检查约束

每次插入或修改数据，都需要检查对应的约束条件
参考函数: `invoke_table_check_constraints`

### 参考文档

[MySQL 8.0.16 Introducing CHECK constraint](https://mysqlserverteam.com/mysql-8-0-16-introducing-check-constraint/)

[WL#929: CHECK constraints](https://dev.mysql.com/worklog/task/?id=929)

[官方文档](https://dev.mysql.com/doc/refman/8.0/en/create-table-check-constraints.html)

[相关代码](https://github.com/mysql/mysql-server/commit/4d7d5165f92f676d011814a0d8e6d0f70c5325fd)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)