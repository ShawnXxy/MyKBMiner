# MySQL · 捉虫动态·DROP DATABASE外键约束的GTID BUG

**Date:** 2015/03
**Source:** http://mysql.taobao.org/monthly/2015/03/06/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 03
 ](/monthly/2015/03)

 * 当期文章

 MySQL · 答疑释惑· 并发Replace into导致的死锁分析
* MySQL · 性能优化· 5.7.6 InnoDB page flush 优化
* MySQL · 捉虫动态· pid file丢失问题分析
* MySQL · 答疑释惑· using filesort VS using temporary
* MySQL · 优化限制· MySQL index_condition_pushdown
* MySQL · 捉虫动态·DROP DATABASE外键约束的GTID BUG
* MySQL · 答疑释惑· lower_case_table_names 使用问题
* PgSQL · 特性分析· Logical Decoding探索
* PgSQL · 特性分析· jsonb类型解析
* TokuDB ·引擎机制· TokuDB线程池

 ## MySQL · 捉虫动态·DROP DATABASE外键约束的GTID BUG 
 Author: 

 **背景**

MySQL的DDL没有被设计成事务操作，因此DDL操作是无法回滚的（像PgSQL把DDL也设计成事务操作，DDL就可以在执行成功后被回滚操作取消）。这就会导致如果某个DDL语句内部被拆分为多个原子的DDL调用，那么这个DDL语句就不具备中途执行失败后回滚整个DDL语句的能力，也就是说，即使语句逻辑内的某个原子DDL调用失败了，也无法回滚已经完成的那些原子DDL调用。

**问题描述**

DROP DATABASE 就是一个例子，对于MySQL而言，DROP DATABASE 并非是一个原子DDL操作，因为它是一个个删除DB下的每张表，而 DROP TABLE 操作本身是会做预检查的，无法删除就会取消删表操作返回失败，所以 DROP TABLE 才能认为是原子的DDL调用。 这就会引起一个问题，如果一个DB中的某张表DROP失败了，实际上 DROP DATABASE 作为一个整体是执行失败的，但是DB中已经有一些表被删除了，因此Binlog中会记录成多个 DROP TABLE 操作，而不是一个 DROP DATABASE 语句。 如果被删除的表的表名都不长，还是会记录成一个删除多张表的 DROP TABLE 语句（DROP TABLE tbl1, tbl2, …），但是如果表名总长度太长，MySQL会拆分为多个 DROP TABLE 语句来记录。 没有GTID的时候这似乎也不是什么大问题，但是引入GTID之后就有一个问题：每个语句只分配一个GTID。如果一个 DROP DATABASE 语句被拆分为多个 DROP TABLE 语句，Binlog中就会出现多个 DROP TABLE 事件共用一个GTID的情况！

举个例子：

`CREATE DATABASE db1;
USE db1;
CREATE TABLE t1 (id INT, name VARCHAR(20), PRIMARY KEY (`id`)) ENGINE=InnoDB;
CREATE TABLE t2 (id INT) ENGINE=InnoDB;
CREATE TABLE t3 (id INT) ENGINE=InnoDB;
CREATE TABLE t4 (id INT) ENGINE=InnoDB;
INSERT INTO t1 VALUES(1, "test"), (2, "try");
INSERT INTO t2 VALUES(1);

# 创建很多表名很长的表
let $count = 50;
while ($count) {
eval create table a_very_long_long_long_long_long_table_name_$count(id int) engine = InnoDB;
dec $count;
}

CREATE DATABASE db2;
USE db2;
CREATE TABLE t3 (id INT, num INT, ext_id INT,
CONSTRAINT t3_fk_1 FOREIGN KEY (ext_id) REFERENCES db1.t1(id)) ENGINE=InnoDB;
INSERT INTO t3 VALUES (1, 2, 2);

DROP DATABASE IF EXISTS db1;
`

这里因为 db2.t3 表引用了 db1.t1 的字段作为外键约束，所以当 db1 做 DROP DATABASE 删除到 t1 表时就报错了，但此时很多表已经被删除了。我们看Binlog中记录的内容：

`SET @@SESSION.GTID_NEXT= &#039;340d95b8-a699-11e4-868d-a0d3c1f20ae4:61&#039;/*!*/;
# at 12209
#150128 10:56:10 server id 1 end_log_pos 13259 CRC32 0xcf952733 Query thread_id=6 exec_time=1 error_code=0
use `db1`/*!*/;
SET TIMESTAMP=1422413770/*!*/;
DROP TABLE IF EXISTS `a_very_long_long_long_long_long_table_name_33`,`a_very_long_long_long_long_long_table_name_15`,`a_very_long_long_long_long_long_table_name_43`,`a_very_long_long_long_long_long_table_name_13`,`a_very_long_long_long_long_long_table_name_10`,`a_very_long_long_long_long_long_table_name_28`,`a_very_long_long_long_long_long_table_name_23`,`a_very_long_long_long_long_long_table_name_32`,`a_very_long_long_long_long_long_table_name_50`,`a_very_long_long_long_long_long_table_name_17`,`a_very_long_long_long_long_long_table_name_19`,`a_very_long_long_long_long_long_table_name_30`,`a_very_long_long_long_long_long_table_name_48`,`a_very_long_long_long_long_long_table_name_49`,`a_very_long_long_long_long_long_table_name_3`,`a_very_long_long_long_long_long_table_name_29`,`a_very_long_long_long_long_long_table_name_9`,`a_very_long_long_long_long_long_table_name_47`,`a_very_long_long_long_long_long_table_name_12`,`a_very_long_long_long_long_long_table_name_42`
/*!*/;
# at 13259
#150128 10:56:10 server id 1 end_log_pos 14315 CRC32 0xd91d1210 Query thread_id=6 exec_time=1 error_code=0
SET TIMESTAMP=1422413770/*!*/;
DROP TABLE IF EXISTS `a_very_long_long_long_long_long_table_name_36`,`a_very_long_long_long_long_long_table_name_1`,`a_very_long_long_long_long_long_table_name_38`,`a_very_long_long_long_long_long_table_name_24`,`a_very_long_long_long_long_long_table_name_16`,`a_very_long_long_long_long_long_table_name_34`,`a_very_long_long_long_long_long_table_name_37`,`a_very_long_long_long_long_long_table_name_6`,`a_very_long_long_long_long_long_table_name_5`,`a_very_long_long_long_long_long_table_name_40`,`t2`,`a_very_long_long_long_long_long_table_name_4`,`a_very_long_long_long_long_long_table_name_20`,`a_very_long_long_long_long_long_table_name_45`,`a_very_long_long_long_long_long_table_name_2`,`a_very_long_long_long_long_long_table_name_27`,`a_very_long_long_long_long_long_table_name_46`,`a_very_long_long_long_long_long_table_name_35`,`t3`,`a_very_long_long_long_long_long_table_name_26`,`a_very_long_long_long_long_long_table_name_8`,`a_very_long_long_long_long_long_table_name_22`
/*!*/;
# at 14315
#150128 10:56:10 server id 1 end_log_pos 14891 CRC32 0x06158e42 Query thread_id=6 exec_time=1 error_code=0
SET TIMESTAMP=1422413770/*!*/;
DROP TABLE IF EXISTS `a_very_long_long_long_long_long_table_name_44`,`a_very_long_long_long_long_long_table_name_11`,`a_very_long_long_long_long_long_table_name_25`,`a_very_long_long_long_long_long_table_name_18`,`a_very_long_long_long_long_long_table_name_7`,`a_very_long_long_long_long_long_table_name_31`,`a_very_long_long_long_long_long_table_name_21`,`a_very_long_long_long_long_long_table_name_14`,`t4`,`a_very_long_long_long_long_long_table_name_39`,`a_very_long_long_long_long_long_table_name_41`
`

3个 DROP TABLE 语句都是同一个GTID：340d95b8-a699-11e4-868d-a0d3c1f20ae4:61

这就导致备库复制报错：

`Last_SQL_Errno: 1837
Last_SQL_Error: Error &#039;When @@SESSION.GTID_NEXT is set to a GTID, you must explicitly set it to a different value after a COMMIT or ROLLBACK. Please check GTID_NEXT variable manual page for detailed explanation. Current @@SESSION.GTID_NEXT is &#039;340d95b8-a699-11e4-868d-a0d3c1f20ae4:61&#039;.&#039; on query. Default database: &#039;db1&#039;. Query: &#039;DROP TABLE IF EXISTS `a_very_long_long_long_long_long_table_name_36`,`a_very_long_long_long_long_long_table_name_1`,`a_very_long_long_long_long_long_table_name_38`,`a_very_long_long_long_long_long_table_name_24`,`a_very_long_long_long_long_long_table_name_16`,`a_very_long_long_long_long_long_table_name_34`,`a_very_long_long_long_long_long_table_name_37`,`a_very_long_long_long_long_long_table_name_6`,`a_very_long_long_long_long_long_table_name_5`,`a_very_long_long_long_long_long_table_name_40`,`t2`,`a_very_long_long_long_long_long_table_name_4`,`a_very_long_long_long_long_long_table_name_20`,`a_very_long_long_long_long_long_table_name_45`,`a_very_long_long_long_long_long_table_name_2`,`a_very_long_lon
`

**解决方案**

怎么解决这个问题呢？

1. 让MySQL支持DDL事务

2. 对DROP DATABASE操作进行预检查

第一种方案对MySQL改动太大了，完全不现实。因此我们采用了第二种方案，也间接实现了 DROP DATABASE 这个操作的原子性。 DROP DATABASE 之所以出现上面的状况，就是因为没有先检查表是否可以删除，而是走一步看一步，一个个删的时候才看能不能删除。我们对MySQL做了修正，对于DB中的每张表，在 DROP DATABASE 执行之前，都先预检查所有可能导致删除表失败的条件，如果一旦发现某张表会无法删除，就放弃整个 DROP DATABASE 操作，提示用户删除错误，让用户先自行解决问题后，再重新执行 DROP DATABASE。

例如上面例子中的情况，本来 DROP DATABASE 执行到有外键约束的表时会报错:

`ERROR 23000: Cannot delete or update a parent row: a foreign key constraint fails
`

但此时其他表已经删除了，而我们修正以后，同样的操作会报一个Error和一个Warning，并且没有真的删任何表：

`SHOW WARNINGS;
Level Code Message
Error 1217 Cannot delete or update a parent row: a foreign key constraint fails
Note 3000 There are at least one table referenced by foreign key, the first table is &#039;t1&#039;. Please drop table(s) that referenced by foreign key first!
`

这里提示了用户有表存在问题无法删除，让用户先处理掉之后，再来执行 DROP DATABASE。此时库下面所有的表都还在，一定要预检查通过才会真的删除。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)