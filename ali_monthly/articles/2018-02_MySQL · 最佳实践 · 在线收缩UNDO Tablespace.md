# MySQL · 最佳实践 · 在线收缩UNDO Tablespace

**Date:** 2018/02
**Source:** http://mysql.taobao.org/monthly/2018/02/09/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 02
 ](/monthly/2018/02)

 * 当期文章

 MySQL · 源码分析 · 常用SQL语句的MDL加锁源码分析
* Influxdb · 源码分析 · Influxdb cluster实现探究
* MySQL · 源码分析 · 权限浅析
* PgSQL · 源码分析 · AutoVacuum机制之autovacuum worker
* MSSQL · 最佳实践 · 数据库恢复模式与备份的关系
* PgSQL · 最佳实践 · 利用异步 dblink 快速从 oss 装载数据
* MySQL · 源码分析 · 新连接的建立
* MySQL · 引擎特性 · INFORMATION_SCHEMA系统表的实现
* MySQL · 最佳实践 · 在线收缩UNDO Tablespace
* PgSQL · 应用案例 · 自定义并行聚合函数的原理与实践

 ## MySQL · 最佳实践 · 在线收缩UNDO Tablespace 
 Author: 

 ## 概述
Undo log一直都是事务多版本控制中的核心组件，它具有以下的核心功能

 * 交易的回退：事务在处理过程中遇到异常的时候可以rollback(撤销)所做的全部修改
* 交易的恢复：数据库实例崩溃时，将磁盘的不正确数据恢复到交易前
* 读一致性：被查询的记录有事务占用，转向回滚段找事务开始前的数据镜像

虽然Undo log是如此的重要，但在MySQL 5.6(包括5.6)之前Undo tablespace里面的undo数据文件是无法收缩的。也就是说在实例的运行过程中如果遇到有大的事务，会把undo log的文件撑的非常大。进而浪费大量的空间甚至把磁盘打爆。同时也增加了数据库物理备份的时间。在实际的工作中不止一次遇到这类问题。好在MySQL5.7中新增了一个非常有用的功能允许用户在线truncate undo log，进而是undo log文件进行收缩。

## 5.7 在线truncate undo log
必须使用独立的undo表空间,该功能主要由以下参数控制

 * innodb_undo_directory，指定单独存放undo表空间的目录，默认为.（即datadir），可以设置相对路径或者绝对路径。该参数实例初始化之后虽然不可直接改动，但是可以通过先停库，修改配置文件，然后移动undo表空间文件的方式去修改该参数；
* innodb_undo_tablespaces，指定单独存放的undo表空间个数，例如如果设置为3，则undo表空间为undo001、undo002、undo003，每个文件初始大小默认为10M。该参数我们推荐设置为大于等于3，原因下文将解释。该参数实例初始化之后不可改动；
* innodb_undo_logs，指定回滚段的个数（早期版本该参数名字是innodb_rollback_segments），默认128个。每个回滚段可同时支持1024个在线事务。这些回滚段会平均分布到各个undo表空间中。该变量可以动态调整，但是物理上的回滚段不会减少，只是会控制用到的回滚段的个数。
* innodb_undo_tablespaces>=2。因为truncate undo表空间时，该文件处于inactive状态，如果只有1个undo表空间，那么整个系统在此过程中将处于不可用状态。为了尽可能降低truncate对系统的影响，建议将该参数最少设置为3；
* innodb_undo_logs>=35（默认128）。因为在MySQL 5.7中，第一个undo log永远在系统表空间中，另外32个undo log分配给了临时表空间，即ibtmp1，至少还有2个undo log才能保证2个undo表空间中每个里面至少有1个undo log；
* innodb_max_undo_log_size，undo表空间文件超过此值即标记为可收缩，默认1G，可在线修改；
* innodb_purge_rseg_truncate_frequency,指定purge操作被唤起多少次之后才释放rollback segments。当undo表空间里面的rollback segments被释放时，undo表空间才会被truncate。由此可见，该参数越小，undo表空间被尝试truncate的频率越高。

## MySQL 5.7的undo表空间的truncate示例

（1） 首先确保如下参数被正确设置：

 * innodb_max_undo_log_size = 100M
* innodb_undo_log_truncate = ON
* innodb_undo_logs = 128
* innodb_undo_tablespaces = 3
* innodb_purge_rseg_truncate_frequency = 10

（2） 创建表：

`
mysql> create table t1( id int primary key auto_increment, name varchar(200));
Query OK, 0 rows affected (0.13 sec)

`

（3）插入测试数据

`mysql> insert into t1(name) values(repeat('a',200));
mysql> insert into t1(name) select name from t1;
mysql> insert into t1(name) select name from t1;
mysql> insert into t1(name) select name from t1;
mysql> insert into t1(name) select name from t1;
`

这时undo表空间文件大小如下，可以看到有一个undo文件已经超过了100M：

`
-rw-r----- 1 mysql mysql 13M Feb 25 17:59 undo001
-rw-r----- 1 mysql mysql 128M Feb 25 17:59 undo002
-rw-r----- 1 mysql mysql 64M Feb 25 17:59 undo003

`
此时，为了，让purge线程运行，可以运行几个delete语句：

`mysql> delete from t1 limit 1;
mysql> delete from t1 limit 1;
mysql> delete from t1 limit 1;
mysql> delete from t1 limit 1;
`

再查看undo文件大小：

`-rw-r----- 1 mysql mysql 13M Feb 25 18:05 undo001
-rw-r----- 1 mysql mysql 10M Feb 25 18:05 undo002
-rw-r----- 1 mysql mysql 64M Feb 25 18:05 undo003
`

可以看到，超过100M的undo文件已经收缩到10M了。

## 小结
在MySQL 5.7中我们有了一个有效的方法可以在数据库实例运行的过程中动态的回收undo log占用的空间。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)