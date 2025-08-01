# MySQL · 捉虫动态 · 字符集相关变量介绍及binlog中字符集相关缺陷分析

**Date:** 2018/01
**Source:** http://mysql.taobao.org/monthly/2018/01/07/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 01
 ](/monthly/2018/01)

 * 当期文章

 MySQL · 引擎特性 · Group Replication内核解析之二
* MySQL · 引擎特性 · MySQL内核对读写分离的支持
* PgSQL · 内核解析 · 同步流复制实现分析
* MySQL · 捉虫动态 · UK 包含 NULL 值备库延迟分析
* MySQL · 捉虫动态 · Error in munmap() "Cannot allocate memory"
* MSSQL · 最佳实践 · 数据库备份链
* MySQL · 捉虫动态 · 字符集相关变量介绍及binlog中字符集相关缺陷分析
* PgSQL · 应用案例 · 传统分库分表(sharding)的缺陷与破解之法
* MySQL · MyRocks · MyRocks参数介绍
* PgSQL · 应用案例 · 惊天性能！单RDS PostgreSQL实例支撑 2000亿

 ## MySQL · 捉虫动态 · 字符集相关变量介绍及binlog中字符集相关缺陷分析 
 Author: 勉仁 

 ## MySQL字符集相关变量介绍及binlog中字符集相关缺陷分析

MySQL支持多种字符集（character set）提供用户存储数据，同时允许用不同排序规则（collation）做比较。

本文基于MySQL5.7介绍了字符集相关变量的使用，通过例子描述了这些变量具体意义。分析了MySQL binlog中字符集相关处理的缺陷，这些缺陷会导致复制中断或者主备不一致。最后给出了修复上述缺陷的建议。

## MySQL字符集相关基础知识介绍

### character_set_system

character_set_system为元数据的字符集，即所有的元数据都使用同一个字符集。试想如果元数据采用不同字符集，INFORMATION_SCHEMA中的相关信息在不同行之间就很难展示。同时该字符集要能够支持多种语言，方便不同语言人群使用自己的语言命名database、table、column。MySQL选择UTF-8作为元数据编码，用源码固定。

`sql/mysqld.cc
int mysqld_main(int argc, char **argv)
{
 ...
 system_charset_info= &my_charset_utf8_general_ci;
}
`

```
> select @@global.character_set_system;
+-------------------------------+
| @@global.character_set_system |
+-------------------------------+
| utf8 |
+-------------------------------+

```

MySQL会将identifier转换为system_charset_info(utf8)。

`sql/sql_lex.cc
static int lex_one_token(YYSTYPE *yylval, THD *thd)
{
 case MY_LEX_IDENT:
 ...
 lip->body_utf8_append_literal
 ...
}

void Lex_input_stream::body_utf8_append_literal(THD *thd,
 const LEX_STRING *txt,
 const CHARSET_INFO *txt_cs,
 const char *end_ptr)
{
 ...
 if (!my_charset_same(txt_cs, &my_charset_utf8_general_ci))
 {
 thd->convert_string(&utf_txt,
 &my_charset_utf8_general_ci,
 txt->str, txt->length,
 txt_cs);
 }
 else
 {
 utf_txt.str= txt->str;
 utf_txt.length= txt->length;
 }
 ...
}

sql/sql_yacc.yy

IDENT_sys:
IDENT { $$= $1; }
| IDENT_QUOTED
{
 THD *thd= YYTHD;

 if (thd->charset_is_system_charset)
 {
 ...
 }
 else
 {
 if (thd->convert_string(&$$, system_charset_info,
 $1.str, $1.length, thd->charset()))
 MYSQL_YYABORT;
 }
}
;
`

### character_set_server/collation_server

当create database没有指定charset/collation就会用character_set_server/collation_server，这两个变量可以动态设置，有session/global级别。

在源码中character_set_server/collation_server实际对应一个变量，因为一个collation对应着一个charset，所以源码中只记录CHARSET_INFO结构的collation_server即可。当修改character_set_server，会选择对应charset的默认collation。对于其他同时有charset和collation的变量，源码记录也都是记录collation。

`static Sys_var_struct Sys_character_set_server(
 "character_set_server", "The default character set",
 SESSION_VAR(collation_server), NO_CMD_LINE,
 offsetof(CHARSET_INFO, csname), DEFAULT(&default_charset_info),
 NO_MUTEX_GUARD, IN_BINLOG, ON_CHECK(check_charset_not_null));

static Sys_var_struct Sys_collation_server(
 "collation_server", "The server default collation",
 SESSION_VAR(collation_server), NO_CMD_LINE,
 offsetof(CHARSET_INFO, name), DEFAULT(&default_charset_info),
 NO_MUTEX_GUARD, IN_BINLOG, ON_CHECK(check_collation_not_null));
`

通过下面case可以看到通过设置session中不同的character_set_server使创建database的默认charset和collation不同。

`> set character_set_server='utf8';

> create database cs_test1;

> select * from SCHEMATA where SCHEMA_NAME='cs_test1';
+--------------+-------------+----------------------------+------------------------+----------+
| CATALOG_NAME | SCHEMA_NAME | DEFAULT_CHARACTER_SET_NAME | DEFAULT_COLLATION_NAME | SQL_PATH |
+--------------+-------------+----------------------------+------------------------+----------+
| def | cs_test1 | utf8 | utf8_general_ci | NULL |
+--------------+-------------+----------------------------+------------------------+----------+

> set character_set_server='latin1';

> create database cs_test2;

> select * from SCHEMATA where SCHEMA_NAME='cs_test2';
+--------------+-------------+----------------------------+------------------------+----------+
| CATALOG_NAME | SCHEMA_NAME | DEFAULT_CHARACTER_SET_NAME | DEFAULT_COLLATION_NAME | SQL_PATH |
+--------------+-------------+----------------------------+------------------------+----------+
| def | cs_test2 | latin1 | latin1_swedish_ci | NULL |
+--------------+-------------+----------------------------+------------------------+----------+
`

### character_set_database/collation_database

该变量值session级别表示当前database的charset/collation，在后面的源码版本中该变量可能修正为只读，不建议修改该值。其global级别变量后面也会移除。

`> use cs_test1;

> select @@character_set_database;
+--------------------------+
| @@character_set_database |
+--------------------------+
| utf8 |
+--------------------------+

> use cs_test2;

> select @@character_set_database;
+--------------------------+
| @@character_set_database |
+--------------------------+
| latin1 |
+--------------------------+
`

### character_set_client

客户端发送到server的字符串使用的字符集，server会按照该变量值来解析客户端发来的语句。如果指定值和语句实际编码字符集不符就会解析出错，报语法错误或者得到非预期结果，例如下面的两个case。

`case1:实际使用utf8编码且包含中文字符，但设置character_set_client为latin1。

> set character_set_client='latin1';

> create table 字符集(c1 varchar(10));
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '­—ç¬¦é›†(c1 varchar(10))' at line 1

> set character_set_client='utf8';

> create table 字符集(c1 varchar(10));
Query OK, 0 rows affected (0.14 sec)

case2:实际使用utf8编码且包含中文字符，但设置character_set_client为gbk。

> create database cs_test;

> use cs_test;

> set character_set_client='gbk';

> create table 收费(c1 varchar(10));

> show tables;
+-------------------+
| Tables_in_cs_test |
+-------------------+
| 鏀惰垂 |
+-------------------+

> set character_set_client='utf8';

> create table 收费(c1 varchar(10));

> show tables;
+-------------------+
| Tables_in_cs_test |
+-------------------+
| 收费 |
| 鏀惰垂 |
+-------------------+
2 rows in set (0.00 sec)
`

### character_set_connection/collation_connection

没有指定字符集的常量字符串使用时的字符集，例如下面两个case。

case1中当设置为utf8_general_ci比较时候忽略大小写，导致’a’=’A’结果为1，如果设置为utf8_bin不忽略大小写，’a’ = ‘A’的结果就是0。

case2中当设置character_set_connection为’latin1’的时候，’你好’ = ‘我好’返回结果为1，如果设置为’utf8’，返回结果就是0。设置为’latin1’返回结果为1的原因是utf8编码的中文字符是无法转换为latin1字符的。这里MySQL就把’你好’和’我好’都转换成了’??’。

case3中character_set_connection的不同导致create table语句中column的实际default value不同。

`case1:设置collation_connection是否忽略大小写导致结果不一致。

> set collation_connection=utf8_general_ci;

> select 'a' = 'A';
+-----------+
| 'a' = 'A' |
+-----------+
| 1 |
+-----------+

> set collation_connection=utf8_bin;

> select 'a' = 'A';
+-----------+
| 'a' = 'A' |
+-----------+
| 0 |
+-----------+

case2:设置character_set_connection不同导致结果不一致。

> set character_set_connection='latin1';
Query OK, 0 rows affected (0.00 sec)

> select '你好' = '我好';
+---------------------+
| '你好' = '我好' |
+---------------------+
| 1 |
+---------------------+
1 row in set, 2 warnings (0.00 sec)

> set character_set_connection='utf8';
Query OK, 0 rows affected (0.00 sec)

> select '你好' = '我好';
+---------------------+
| '你好' = '我好' |
+---------------------+
| 0 |
+---------------------+

> set character_set_connection='latin1';

> select '你好';
+----+
| ?? |
+----+
| ?? |
+----+

case3:设置character_set_connection导致实际default value不同。

> set character_set_connection='utf8';

> create table cs_t(c1 varchar(10) default '你好')charset=utf8;

> insert into cs_t values();

> select * from cs_t;
+--------+
| c1 |
+--------+
| 你好 |
+--------+

> set character_set_connection='latin1';

> create table cs_t1(c1 varchar(10) default '你好')charset=utf8;

> insert into cs_t1 values();

> select * from cs_t1;
+------+
| c1 |
+------+
| ?? |
+------+
`

### character_set_results

查询结果和错误信息的字符集，server会把返回给客户端的结果转换为对应字符集。例如下面case，当设置character_set_results为’latin1’的时候，会导致返回的中文变成’?’。

`> set character_set_results='utf8';

> select '你好';
+--------+
| 你好 |
+--------+
| 你好 |
+--------+

> set character_set_results='latin1';

> select '你好';
+----+
| ?? |
+----+
| ?? |
+----+

> create table cs_test(c1 varchar(10)) charset=utf8;

> insert into cs_test values('你好'),('我好');

> select * from cs_test;
+------+
| c1 |
+------+
| ?? |
| ?? |
+------+

> set character_set_results='utf8';

> select * from cs_test;
+--------+
| c1 |
+--------+
| 你好 |
| 我好 |
+--------+

`

## binlog 中字符集相关缺陷

### binlog当前字符集相关实现

对于很多DDL语句，binlog都是直接记录客户端发来的字符串，对于这些语句只要记录语句执行时候的环境变量就可以在备库正确执行。binlog中Query_log_event记录了character_set_client、collation_connection和collation_server，代码如下。记录这三个变量的原因读者可以参考前面各个变量的介绍和case。

`int THD::binlog_query(THD::enum_binlog_query_type qtype, const char *query_arg,
 size_t query_len, bool is_trans, bool direct,
 bool suppress_use, int errcode)
{
 ...
 case THD::STMT_QUERY_TYPE:
 /*
 The MYSQL_BIN_LOG::write() function will set the STMT_END_F flag and
 flush the pending rows event if necessary.
 */
 {
 Query_log_event qinfo(this, query_arg, query_len, is_trans, direct,
 suppress_use, errcode);
 /*
 Binlog table maps will be irrelevant after a Query_log_event
 (they are just removed on the slave side) so after the query
 log event is written to the binary log, we pretend that no
 table maps were written.
 */
 int error= mysql_bin_log.write_event(&qinfo);
 binlog_table_maps= 0;
 DBUG_RETURN(error);
 }
 ...
}

Query_log_event::Query_log_event(THD* thd_arg, const char* query_arg,
 size_t query_length, bool using_trans,
 bool immediate, bool suppress_use,
 int errcode, bool ignore_cmd_internals)
{
 ...
 int2store(charset, thd_arg->variables.character_set_client->number);
 int2store(charset+2, thd_arg->variables.collation_connection->number);
 int2store(charset+4, thd_arg->variables.collation_server->number);
 ...
}
`

例如前面创建表cs_t1的case我们可以看到binlog如下。

`> set character_set_connection='latin1';
> create table cs_t1(c1 varchar(10) default '你好')charset=utf8;

SET TIMESTAMP=1516089074/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=8,@@session.collation_server=8/*!*/;
create table cs_t1(c1 varchar(10) default '你好')charset=utf8
`

### binlog字符集相关缺陷

对于Query_log_event如果记录的query仅仅是客户端的输入，上面记录字符集变量的方法没有问题。但如果query是server内部生成或者拼接成的，上面直接从thread中获取变量值得方法就可能导致错误。

例如下面的testcase，这里为便于观察和理解case没有使用mysql-test方式，后面有mysql-test。这里主库执行成功，成功创建了表t和视图’收费明细表’，但备库在创建视图的时候却报语法错误。

`用gbk编码写如下sql文本
cs_test.sql

use test;
set @@session.character_set_client=gbk;
set @@session.collation_connection=gbk_chinese_ci;
create table t(c1 int);
create view `收费明细表` as select * from t;

在主库执行
> source path/cs_test.sql;

> set character_set_results='gbk';

> use test;

> show tables;
+----------------+
| Tables_in_test |
+----------------+
| 收费明细表 |
| t |
+----------------+

备库

> show slave status\G
...
Last_SQL_Errno: 1064
Last_SQL_Error: Error 'You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '`閺鎯板瀭閺勫海绮忕悰鈺? AS select * from t' at line 1' on query. Default database: 'test'. Query: 'CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`localhost` SQL SECURITY DEFINER VIEW `鏀惰垂鏄庣粏琛╜ AS select * from t'
...

`

缺陷分析，MySQL记录create view的binlog代码如下。由前面基础知识可以知道对于db、table这些元数据MySQL会先转换为system_charset_info(utf8)。因此在下面代码中append_identifier添加的table name为utf8编码的’收费明细表’，但是views->source.str又是client端原始的gbk编码方式，binlog_query记录的是thd中的character_set_client。即binlog中的query可能是由system_charset_info和character_set_client两种编码方式组成的字符串，记录的是当前character_set_client的值。

`sql/sql_view.cc
bool mysql_create_view(THD *thd, TABLE_LIST *views,
 enum_view_create_mode mode)
{
 ...
 if (views->db && views->db[0] &&
 (thd->db().str == NULL || strcmp(views->db, thd->db().str)))
 {
 append_identifier(thd, &buff, views->db,
 views->db_length);
 buff.append('.');
 }
 append_identifier(thd, &buff, views->table_name,
 views->table_name_length);
 if (lex->view_list.elements)
 {
 List_iterator_fast<LEX_STRING> names(lex->view_list);
 LEX_STRING *name;
 int i;

 for (i= 0; (name= names++); i++)
 {
 buff.append(i ? ", " : "(");
 append_identifier(thd, &buff, name->str, name->length);
 }
 buff.append(')');
 }
 buff.append(STRING_WITH_LEN(" AS "));
 buff.append(views->source.str, views->source.length);

 int errcode= query_error_code(thd, TRUE);
 thd->add_to_binlog_accessed_dbs(views->db);
 if (thd->binlog_query(THD::STMT_QUERY_TYPE,
 buff.ptr(), buff.length(), FALSE, FALSE, FALSE, errcode))
 res= TRUE;
 ...
}
`

在MySQL源码中搜索binlog_query还可以找到多处类似的bug，可参考下面的testcase。

`--disable_warnings
--source include/master-slave.inc
--enable_warnings

# case1:创建gbk编码中文名视图

create table t(c1 int);
SET @@session.character_set_client=gbk;
set @@session.collation_connection=gbk_chinese_ci;
set @@session.collation_server=utf8_general_ci;
create view `收费明细` as select * from t;
drop view `收费明细`;
show tables;

--sync_slave_with_master

connection slave;
show tables;

connection master;
drop table t;

# case2:创建gbk编码中文名视图，且view body中包含中文
connection master;
SET @@session.character_set_client=gbk;
create table 视图(c1 int);
create view 视图信息 as select * from 视图;
drop view 视图信息;

# case3: drop table 语句会是generated by server.
drop table 视图;
--sync_slave_with_master

# case4:内存表，重启后再次访问时会生成delete from tableName语句.
connection master;
SET @@session.character_set_client=utf8;
set @@session.collation_connection=utf8_general_ci;
set @@session.collation_server=utf8_general_ci;
create table `收费明细表`(c1 int) engine=memory;
create view tv as select * from `收费明细表`;
--connection slave
-- source include/stop_slave.inc

--let $rpl_server_number= 1
--source include/rpl_restart_server.inc
# access memory table after restarting server cause binlog 'delete from tableName'
connection master;
SET @@session.character_set_client=gbk;
set @@session.collation_connection=gbk_chinese_ci;
set @@session.collation_server=utf8_general_ci;
select * from tv;

--connection slave
-- source include/start_slave.inc
connection master;
--sync_slave_with_master
connection slave;

# case5:character_set_client为gbk时中文名的procedure

connection master;
delimiter $$;
create procedure 收费明细()
begin
 select 'hello world';
end $$
delimiter ;$$
drop procedure `收费明细`;

connection master;
SET @@session.character_set_client=utf8;
set @@session.collation_connection=utf8_general_ci;
set @@session.collation_server=utf8_general_ci;
drop view tv;
drop table `收费明细表`;
--sync_slave_with_master

connection slave;
show tables;

# case6: 不同环境变量下create table like/as 表中有中文default value的

set character_set_client = utf8;
set character_set_connection = utf8;
set character_set_database = utf8;
set character_set_results = utf8;
set character_set_server = utf8;

CREATE TABLE `t1` (
 `id` int(11) NOT NULL,
 `orderType` char(6) NOT NULL DEFAULT '已创建',
 PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

create temporary table `tm` (c1 varchar(10) default '你好');

show create table t1;

## switch client charset
set character_set_client = latin1;
set character_set_connection = latin1;
set collation_server = utf8_bin;
CREATE TABLE t2 SELECT * FROM t1;
create table t3 like tm;
show create table t2;
show create table t3;

--sync_slave_with_master

connection slave;
show tables;
set character_set_client = utf8;
set character_set_connection = utf8;
set character_set_database = utf8;
set character_set_results = utf8;
set character_set_server = utf8;
show create table t1;
show create table t2;
show create table t3;

connection master;
drop table t1;
drop table t2;
drop table t3;

--sync_slave_with_master

--source include/rpl_end.inc
`

### 修复方法

对于create view/create procedure等一个query包含两种编码的可以将system_charset_info的部分转换为thread中的character_set_client。这里的转换需要考虑当前thread中character_set_client不支持utf8字符的问题，当转换失败需要报错，否则主备会不一致。

对于完全由server生成的query，例如delete from和drop table语句，其query实际可以理解为system_charset_info，这种语句可以直接使binlog中character_set_client部分记录system_charset_info，而不是thread中的变量值。

该bug在MariaDB中也存在，可以见[MDEV-14249](https://jira.mariadb.org/browse/MDEV-14249?page=com.atlassian.jira.plugin.system.issuetabpanels%3Aall-tabpanel)，参考链接中的fix diff或者MariaDB的修复。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)