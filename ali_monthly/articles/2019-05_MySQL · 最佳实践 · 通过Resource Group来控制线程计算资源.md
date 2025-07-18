# MySQL · 最佳实践 · 通过Resource Group来控制线程计算资源

**Date:** 2019/05
**Source:** http://mysql.taobao.org/monthly/2019/05/05/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2019 / 05
 ](/monthly/2019/05)

 * 当期文章

 MSSQL · 最佳实践 · 挑战云计算安全的存储过程
* MySQL · 源码分析 · 聚合函数（Aggregate Function）的实现过程
* PgSQL · 最佳实践 · RDS for PostgreSQL 的逻辑订阅
* MySQL · 引擎特性 · 通过 SQL 管理 UNDO TABLESPACE
* MySQL · 最佳实践 · 通过Resource Group来控制线程计算资源
* MySQL · 引擎特性 · Skip Scan Range
* MongoDB · 应用案例 · killOp 案例详解
* MySQL · 源码分析 · LinkBuf设计与实现
* PgSQL · 应用案例 · PostgreSQL KPI分解，目标设定之 - 等比数列
* PgSQL · 应用案例 · PostgreSQL KPI 预测例子

 ## MySQL · 最佳实践 · 通过Resource Group来控制线程计算资源 
 Author: yinfeng 

 ＭySQL8.0增加了一个新功能resource group,　可以对不同的用户进行资源控制，例如对用户线程和后台系统线程给予不同的CPU优先级。

用户可以通过SQL接口创建不同的分组，这些分组可以作为sql的hit，也可以动态的绑定过去。本文主要简单介绍下用法，至于底层如何实现的，其实比较简单：创建的分组被存储到系统表中；在linux系统底层通过CPU_SET来绑定CPU，通过setpriority来设置线程的nice值

相关worklog:
[WL#9467: Resource Groups](https://dev.mysql.com/worklog/task/?id=9467)

## 创建resource group

首先系统自带两个resource group并且不可被修改

`root@(none) 05:54:22>SELECT * FROM INFORMATION_SCHEMA.RESOURCE_GROUPS\G
*************************** 1. row ***************************
RESOURCE_GROUP_NAME: USR_default
RESOURCE_GROUP_TYPE: USER
RESOURCE_GROUP_ENABLED: 1
VCPU_IDS: 0-63
THREAD_PRIORITY: 0
*************************** 2. row ***************************
RESOURCE_GROUP_NAME: SYS_default
RESOURCE_GROUP_TYPE: SYSTEM
RESOURCE_GROUP_ENABLED: 1
VCPU_IDS: 0-63
THREAD_PRIORITY: 0
2 rows in set (0.00 sec)
`

如果你想设置thread priority，可能需要使用超级账户来启动Mysqld，这是系统限制，如果以非super账户启动，只能降低而不能提升优先级。在非super启动时，thread_priority会被忽略掉并报一个warning出来。

对于类型为system的系统后台线程，cpu priority只能从-20 ~0，而普通user线程，则在0~19之间，这样就保证了系统线程的优先级肯定比用户线程高。

如果设置不在范围内，就会报错

` root@(none) 10:27:09>CREATE RESOURCE GROUP test_user_rg TYPE = USER VCPU = 0-32,48-63 THREAD_PRIORITY = -10;
 ERROR 3654 (HY000): Invalid thread priority value -10 for User resource group test_user_rg. Allowed range is [0, 19].
 我们尝试为user类线程创建一个resource group，使用0-32， 48-63号cpu, 线程优先级为10

 root@(none) 10:27:14>CREATE RESOURCE GROUP test_user_rg TYPE = USER VCPU = 0-32,48-63 THREAD_PRIORITY = 10;
 Query OK, 0 rows affected (0.01 sec)

 root@(none) 10:55:19>SELECT * FROM INFORMATION_SCHEMA.RESOURCE_GROUPS WHERE RESOURCE_GROUP_NAME = 'test_user_rg'\G
 *************************** 1. row ***************************
 RESOURCE_GROUP_NAME: test_user_rg
 RESOURCE_GROUP_TYPE: USER
 RESOURCE_GROUP_ENABLED: 1
 VCPU_IDS: 0-32,48-63
 THREAD_PRIORITY: 10
 1 row in set (0.00 sec)
 CREATE/DELETE/ALTER RESOURCE GROUP都需要RESOURCE_GROUP_ADMIN权限，具体的语法见官方文档
`

## 使用resource group
创建好后，我们该如何使用resource group呢，主要有两种方式，一种是SET RESOURCE GROUP, 一种是通过SQL HINT的方式，以下是简单的测试：

设置当前session：

` root@(none) 11:01:08>SET RESOURCE GROUP test_user_rg;
 Query OK, 0 rows affected (0.00 sec)
 也可以指定hint的方式来设置：

 root@sb1 11:07:53>select /* + RESOURCE_GROUP(test_user_rg) */ * from sbtest1 where id <10;
 还可以通过thread id来设置其他运行中的session，注意这里的thread id不是show processlist看到的id，而是通过performance_schema.threads表看到的id

 xx@performance_schema 11:30:21>SELECT THREAD_ID, TYPE FROM performance_schema.threads WHERE PROCESSLIST_ID = 26\G
 *************************** 1. row ***************************
 THREAD_ID: 71
 TYPE: FOREGROUND
 1 row in set (0.00 sec)
 xx@performance_schema 11:30:43>SET RESOURCE GROUP test_user_rg for 71;
 Query OK, 0 rows affected (0.00 sec)
`

如果你想对InnoDB的后台线程来进行设置呢 ？ 可以去查看performance_schema.threads表，例如我们对page cleaner进行优先级设置:

` xx@performance_schema 11:19:43>CREATE RESOURCE GROUP test_system_rg TYPE = SYSTEM VCPU = 49 THREAD_PRIORITY = -10;
 Query OK, 0 rows affected (0.00 sec)

 xx@performance_schema 11:24:11>SELECT THREAD_ID, TYPE FROM performance_schema.threads WHERE NAME LIKE '%page_flush_coor%'\G
 *************************** 1. row ***************************
 THREAD_ID: 13
 TYPE: BACKGROUND
 1 row in set (0.00 sec)
 xx@performance_schema 11:24:07>SET RESOURCE GROUP test_system_rg for 13;
 Query OK, 0 rows affected (0.00 sec)
`
可以看到，通过resource group，我们可以为任意的线程指定不同的计算资源。在未来我们甚至可以对这一功能进行扩展，例如某个线程的最大iops，读入数据占用buffer pool的百分比，或者对运维程序指定独立的cpu，避免干扰到正常的业务负载等等，还是有不少的想象空间的。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)