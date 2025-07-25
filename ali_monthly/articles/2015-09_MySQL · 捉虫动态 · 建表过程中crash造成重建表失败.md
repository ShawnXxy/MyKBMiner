# MySQL · 捉虫动态 · 建表过程中crash造成重建表失败

**Date:** 2015/09
**Source:** http://mysql.taobao.org/monthly/2015/09/05/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 09
 ](/monthly/2015/09)

 * 当期文章

 MySQL · 引擎特性 · InnoDB Adaptive hash index介绍
* PgSQL · 特性分析 · clog异步提交一致性、原子操作与fsync
* MySQL · 捉虫动态 · BUG 几例
* PgSQL · 答疑解惑 · 诡异的函数返回值
* MySQL · 捉虫动态 · 建表过程中crash造成重建表失败
* PgSQL · 特性分析 · 谈谈checkpoint的调度
* MySQL · 特性分析 · 5.6 并行复制恢复实现
* MySQL · 备库优化 · relay fetch 备库优化
* MySQL · 特性分析 · 5.6并行复制事件分发机制
* MySQL · TokuDB · 文件目录谈

 ## MySQL · 捉虫动态 · 建表过程中crash造成重建表失败 
 Author: 西加 

 ## 问题描述

主库的`create table`语句传到备库，备库SQL线程执行过程中报错：

`Error 'Can't create table 'XXX.XX' (errno: -1)' on query. Default database: 'XXX'. Query: 'CREATE TABLE XX ( column_a char(32) NOT NULL, column_b int(10) DEFAULT NULL, column_c int(10) DEFAULT NULL, PRIMARY KEY (column_a), KEY expiry (column_b)) ENGINE=HEAP DEFAULT CHARSET=gbk'
`

备库 error log：

`InnoDB: Error number 17 means 'File exists'.
InnoDB: Some operating system error numbers are described at
InnoDB: http://dev.mysql.com/doc/refman/5.5/en/operating-system-error- codes.html
InnoDB: The file already exists though the corresponding table did not
InnoDB: exist in the InnoDB data dictionary. Have you moved InnoDB
InnoDB: .ibd files around without using the SQL commands
InnoDB: DISCARD TABLESPACE and IMPORT TABLESPACE, or did
InnoDB: mysqld crash in the middle of CREATE TABLE?You can
！！！InnoDB: resolve the problem by removing the file '...'
InnoDB: under the 'datadir' of MySQL.
`

从error log中可以看出，数据目录中已存在 .ibd 文件，推测是在建表过程中发生 crash。

数据目录下存在 .ibd，不存在 .frm，创建.ibd 文件的时间:

`-rw-rw---- 1 mysql mysql 65536 Sep 5 14:41 XXX.ibd
`

.ibd 文件创建时间 150905 14:41，对应时间的 error log:

`150905 14:41:58 mysqld_safe Number of processes running now: 0
150905 14:41:58 mysqld_safe mysqld restarted
`

之后也出现了和该创建失败的表相关的错误记录：

`150905 14:59:45 InnoDB: Error: table `XXX`.`XX` does not exist in the InnoDB internal
`

## 问题分析

执行如下语句，模拟建表

`create table test.t3 (id int);
`

`create table` 时，由函数`mysql_create_frm`创建 .frm 文件，`mysql_create_frm` 调用栈如下：

`#0 mysql_create_frm
#1 rea_create_table
#2 mysql_create_table_no_lock
#3 mysql_create_table
#4 mysql_execute_command
#5 mysql_parse
`

t3.frm 文件生成后，实例 crash（函数`mysql_create_frm` 执行完毕后`kill mysqld`），在数据库中`show tables`可以看到 test.t3，但是无法插入，数据目录下 t3.frm 文件依然存在。

`drop table`报错

`ERROR 1051 (42S02): Unknown table 'test.t3'
`
之后数据目录下的t3.frm不存在，show tables 无法看到t3表，可以重新创建t3表。

.ibd 文件由函数`fil_create_new_single_table_tablespace`创建，`fil_create_new_single_table_tablespace`调用栈如下：

`#0 fil_create_new_single_table_tablespace
#1 dict_build_table_def_step
#2 dict_create_table_step
#3 que_thr_step
#4 que_run_threads_low
#5 que_run_threads
#6 row_create_table_for_mysql
#7 create_table_def
#8 ha_innobase::create
#9 handler::ha_create
#10 ha_create_table
#11 rea_create_table
#12 create_table_impl
#13 mysql_create_table_no_lock
#14 mysql_create_table
#15 mysql_execute_command
#16 mysql_parse
`

t3.ibd 文件生成后，实例 crash（函数`fil_create_new_single_table_tablespace`执行完毕后`kill mysqld`），在数据库中`show tables`可以看到 test.t3，无法插入数据，在数据目录下存在文件 t3.frm 和 t3.ibd。

`drop table`依然可以移除 t3.frm 并使`show tables`无法看到 t3 表。但无法移除 t3.ibd，并在重建 t3 表时报错：

`ERROR 1813 (HY000): Tablespace for table '`test`.`t3`' exists. Please DISCARD the tablespace before IMPORT.
`

在数据目录中删除 t3.ibd ，可以正常重建 t3 表。

这个 bug 的主要原因是 MySQL 的建表过程不是原子操作。如果建表过程正在进行的时候实例 crash，可能会造成一些在实例重启后无法自动恢复的问题。就像这个问题当中的文件残留，无法通过 MySQL 客户端中的操作解决，只能手动删除文件。如果用户是远程连接数据库，又没有登录服务器操作数据文件的权限，就会影响数据库的可用性。

MySQL 5.7 的实验室版本正在设计和实现新版本的数据字典来解决这一问题。这个版本主要由以下几个特性：

* 数据字典将实现事务存储，首先利用 InnoDB 存储，其他存储引擎可能会跟进开发；
* 把分布式系统中的字典信息统一成一个整体；
* 使用统一的规则存储字典信息，给字典对象定义统一的API；
* 避免文件系统特性带来的问题。

详细信息参见[MySQL Server Blog](http://mysqlserverteam.com/a-new-data-dictionary-for-mysql/)

## 问题解决

通过问题分析，判断备库无法建表是由于在执行`create table`语句时备库实例crash，且crash时.ibd 文件已存在。用户发现表创建失败，企图重建表依然失败，之后执行了`drop table`语句，移除了.frm文件，但.ibd文件依然存在，无法重建表。
将数据目录下的.ibd文件移到其他文件夹作为备份，在备库`start slave`后建表成功，主备复制正常。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)