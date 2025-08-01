# MySQL · 捉虫动态 · MySQL 外键异常分析

**Date:** 2015/11
**Source:** http://mysql.taobao.org/monthly/2015/11/06/
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

 ## MySQL · 捉虫动态 · MySQL 外键异常分析 
 Author: 济天 

 ## 外键约束异常现象
如下测例中，没有违反引用约束的插入失败。

`create database `a-b`;
use `a-b`;
SET FOREIGN_KEY_CHECKS=0;
create table t1(c1 int primary key, c2 int) engine=innodb;
create table t2(c1 int primary key, c2 int) engine=innodb;
alter table t2 add foreign key(c2) references `a-b`.t1(c1);
SET FOREIGN_KEY_CHECKS=1;
insert into t1 values(1,1);
select * from t1;
c1 c2
1 1
select * from t2;
c1 c2
insert into t2 values(1,1);
ERROR 23000: Cannot add or update a child row: a foreign key constraint fails (`a-b`.`t2`, CONSTRAINT `t2_ibfk_1` FOREIGN KEY (`c2`) REFERENCES `a-b`.`t1` (`c1`))
insert into t2 values(1,1); //预期应该成功实际失败了。子表插入任何数据都会报违反引用约束。
`

## 异常分析

首先我们会检查表结构是否正常

`show create table t2;
Table Create Table
t2 CREATE TABLE `t2` (
 `c1` int(11) NOT NULL,
 `c2` int(11) DEFAULT NULL,
 PRIMARY KEY (`c1`),
 KEY `c2` (`c2`),
 CONSTRAINT `t2_ibfk_1` FOREIGN KEY (`c2`) REFERENCES `a-b`.`t1` (`c1`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
`

查看 innodb_sys_foreign 表

`select * from information_schema.innodb_sys_foreign where id='a@002db/t2_ibfk_1';
+-------------------+------------+----------+--------+------+
| ID | FOR_NAME | REF_NAME | N_COLS | TYPE |
+-------------------+------------+----------+--------+------+
| a@002db/t2_ibfk_1 | a@002db/t2 | a-b/t1 | 1 | 0 |
+-------------------+------------+----------+--------+------+

select * from information_schema.innodb_sys_tables where name='a@002db/t1';
+----------+------------+------+--------+-------+-------------+------------+---------------+
| TABLE_ID | NAME | FLAG | N_COLS | SPACE | FILE_FORMAT | ROW_FORMAT | ZIP_PAGE_SIZE |
+----------+------------+------+--------+-------+-------------+------------+---------------+
| 530 | a@002db/t1 | 1 | 5 | 525 | Antelope | Compact | 0 |
+----------+------------+------+--------+-------+-------------+------------+---------------+
`

表结构正常，表面上看外键在系统表中元数据库信息正常。仔细比较发现 innodb_sys_foreign 的REF_NAME字段”a-b/t1”实际应为”a@002db/t2”。

## MySQL内部表名和库名存储格式

MySQL 内部用 my_charset_filename 字符集来表名和库名。

以下数组定义了 my_charset_filename 字符集需要转换的字符。数组下标为 ascii 值，1代表不需要转换。可以看到字母数字和下划线等不需要转换，同时字符’-‘是需要转换的， 转换函数参见`my_wc_mb_filename`。

`static char filename_safe_char[128]=
{
 1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0, /* ................ */
 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0, /* ................ */
 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0, /* !"#$%&'()*+,-./ */
 1,1,1,1,1,1,1,1,1,1,0,0,0,0,0,0, /* 0123456789:;<=>? */
 0,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1, /* @ABCDEFGHIJKLMNO */
 1,1,1,1,1,1,1,1,1,1,1,0,0,0,0,1, /* PQRSTUVWXYZ[\]^_ */
 0,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1, /* `abcdefghijklmno */
 1,1,1,1,1,1,1,1,1,1,1,0,0,0,0,0, /* pqrstuvwxyz{|}~. */
};
`

## 异常分析

由上节可知，字符’-‘作为库名或表名是需要转换的。innodb_sys_foreign 中 FOR_NAME 值是转换过的，只有 REF_NAME 未转换，而系统表 innodb_sys_tables 中存储的表名是转换后的。`dict_get_referenced_table` 根据未转换的表名 a-b/t1 去系统表 SYS_TABLES 查找会查找不到记录。于是会导致

` foreign->referenced_table==NULL
`

因此对子表的任何插入都会返回错误 DB_NO_REFERENCED_ROW，如下代码

`row_ins_check_foreign_constraint:

 if (check_ref) {
 check_table = foreign->referenced_table;
 check_index = foreign->referenced_index;
 } else {
 check_table = foreign->foreign_table;
 check_index = foreign->foreign_index;
 }

if (check_table == NULL
 || check_table->ibd_file_missing
 || check_index == NULL) {

 if (!srv_read_only_mode && check_ref) {
 ……
 err = DB_NO_REFERENCED_ROW;
 }

 goto exit_func;
`

经过进一步调试分析发现，函数`innobase_get_foreign_key_info`中主表的库名和表名都没有经过转换，而是直接使用系统字符集。

回过头再看看bug的触发条件：

1. 表名或库名包含特殊字符；
2. 此表作为引用约束的主表；
3. 增加引用约束是设置了SET FOREIGN_KEY_CHECKS=0；

这里强调下第3条, 如果上面的测例中去掉了SET FOREIGN_KEY_CHECKS=0，那么结果 REF_NAME会正常转换

`SET FOREIGN_KEY_CHECKS=1;
create table t1(c1 int primary key, c2 int) engine=innodb;
create table t2(c1 int primary key, c2 int) engine=innodb;
alter table t2 add foreign key(c2) references `a-b`.t1(c1);
select * from information_schema.innodb_sys_foreign where id='a@002db/t2_ibfk_1';
+-------------------+------------+------------+--------+------+
| ID | FOR_NAME | REF_NAME | N_COLS | TYPE |
+-------------------+------------+------------+--------+------+
| a@002db/t2_ibfk_1 | a@002db/t2 | a@002db/t1 | 1 | 0 |
+-------------------+------------+------------+--------+------+
`

## online DDL 与 foreign key

MySQL 5.6 online DDL 是支持建索引的。而对于建外键索引同样也是支持的，条件是SET FOREIGN_KEY_CHECKS=0。

`ha_innobase::check_if_supported_inplace_alter：
 if ((ha_alter_info->handler_flags
 & Alter_inplace_info::ADD_FOREIGN_KEY)
 && prebuilt->trx->check_foreigns) {
 ha_alter_info->unsupported_reason = innobase_get_err_msg(
 ER_ALTER_OPERATION_NOT_SUPPORTED_REASON_FK_CHECK);
 DBUG_RETURN(HA_ALTER_INPLACE_NOT_SUPPORTED);
 }
`

SET FOREIGN_KEY_CHECKS=0时，`prebuilt->trx->check_foreigns`为false。

我们再来看出问题的函数`innobase_get_foreign_key_info`，只有online DDL的代码路径才会调用此函数：

`#0 innobase_get_foreign_key_info
#1 ha_innobase::prepare_inplace_alter_table
#2 handler::ha_prepare_inplace_alter_table
#3 mysql_inplace_alter_table
#4 mysql_alter_table
......
`

而非online DDL的路径如下，函数 `dict_scan_id` 会对表名和库名进行转换：

`#0 dict_scan_id
#1 dict_scan_table_name
#2 dict_create_foreign_constraints_low
#3 dict_create_foreign_constraints
#4 row_table_add_foreign_constraints
#5 ha_innobase::create
#6 handler::ha_create
#7 ha_create_table
#8 mysql_alter_table
......
`

## 修复

bug系统中虽然没有相关的bug信息，但从MySQL 5.6.26中我们看到官方Bug#21094069已经进行了修复，在`innobase_get_foreign_key_info`中对库名和表名进行转换。

参考commit:[1fae0d42c352908fed03e29db2b391a0d2969269](https://github.com/mysql/mysql-server/commit/1fae0d42c352908fed03e29db2b391a0d2969269)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)