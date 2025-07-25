# MySQL · 捉虫动态· InnoDB自增列重复值问题

**Date:** 2015/01
**Source:** http://mysql.taobao.org/monthly/2015/01/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 01
 ](/monthly/2015/01)

 * 当期文章

 MySQL · 性能优化· Group Commit优化
* MySQL · 新增特性· DDL fast fail
* MySQL · 性能优化· 启用GTID场景的性能问题及优化
* MySQL · 捉虫动态· InnoDB自增列重复值问题
* MySQL · 优化改进· 复制性能改进过程
* MySQL · 谈古论今· key分区算法演变分析
* MySQL · 捉虫动态· mysql client crash一例
* MySQL · 捉虫动态· 设置 gtid_purged 破坏AUTO_POSITION复制协议
* MySQL · 捉虫动态· replicate filter 和 GTID 一起使用的问题
* TokuDB·特性分析· Optimize Table

 ## MySQL · 捉虫动态· InnoDB自增列重复值问题 
 Author: 

 **问题重现**

先从问题入手，重现下这个[bug](http://bugs.mysql.com/bug.php?id=199)

`use test;
drop table if exists t1;
create table t1(id int auto_increment, a int, primary key (id)) engine=innodb;
insert into t1 values (1,2);
insert into t1 values (null,2);
insert into t1 values (null,2);
select * from t1;
+----+------+
| id | a |
+----+------+
| 1 | 2 |
| 2 | 2 |
| 3 | 2 |
+----+------+
delete from t1 where id=2;
delete from t1 where id=3;
select * from t1;
+----+------+
| id | a |
+----+------+
| 1 | 2 |
+----+------+
`

这里我们关闭mysql，再启动mysql,然后再插入一条数据

`insert into t1 values (null,2);
select * FROM T1;
+----+------+
| id | a |
+----+------+
| 1 | 2 |
+----+------+
| 2 | 2 |
+----+------+
`

我们看到插入了（2,2），而如果我没有重启，插入同样数据我们得到的应该是（4,2)。 上面的测试反映了mysqld重启后，InnoDB存储引擎的表自增id可能出现重复利用的情况。

自增id重复利用在某些场景下会出现问题。依然用上面的例子，假设t1有个历史表t1_history用来存t1表的历史数据，那么mysqld重启前,ti_history中可能已经有了（2,2）这条数据，而重启后我们又插入了（2，2），当新插入的(2,2)迁移到历史表时，会违反主键约束。

**原因分析**

InnoDB 自增列出现重复值的原因

`mysql&gt; show create table t1\G;
*************************** 1\. row ***************************
Table: t1
Create Table: CREATE TABLE `t1` (
`id` int(11) NOT NULL AUTO_INCREMENT,
`a` int(11) DEFAULT NULL,
PRIMARY KEY (`id`)
) ENGINE=innodb AUTO_INCREMENT=4 DEFAULT CHARSET=utf8
1 row in set (0.00 sec)
`

建表时可以指定 AUTO_INCREMENT值，不指定时默认为1，这个值表示当前自增列的起始值大小，如果新插入的数据没有指定自增列的值，那么自增列的值即为这个起始值。对于InnoDB表，这个值没有持久到文件中。而是存在内存中（dict_table_struct.autoinc）。那么又问，既然这个值没有持久下来，为什么我们每次插入新的值后， show create table t1看到AUTO_INCREMENT值是跟随变化的。其实show create table t1是直接从dict_table_struct.autoinc取得的（ha_innobase::update_create_info）。

知道了AUTO_INCREMENT是实时存储内存中的。那么，mysqld 重启后，从哪里得到AUTO_INCREMENT呢? 内存值肯定是丢失了。实际上mysql采用执行类似select max(id)+1 from t1;方法来得到AUTO_INCREMENT。而这种方法就是造成自增id重复的原因。

**MyISAM自增值**

MyISAM也有这个问题吗？MyISAM是没有这个问题的。myisam会将这个值实时存储在.MYI文件中（mi_state_info_write）。mysqld重起后会从.MYI中读取AUTO_INCREMENT值（mi_state_info_read)。因此，MyISAM表重启是不会出现自增id重复的问题。

**问题修复**

MyISAM选择将AUTO_INCREMENT实时存储在.MYI文件头部中。实际上.MYI头部还会实时存其他信息，也就是说写AUTO_INCREMENT只是个顺带的操作，其性能损耗可以忽略。InnoDB 表如果要解决这个问题，有两种方法。1）将AUTO_INCREMENT最大值持久到frm文件中。2）将 AUTO_INCREMENT最大值持久到聚集索引根页trx_id所在的位置。第一种方法直接写文件性能消耗较大，这是一额外的操作，而不是一个顺带的操作。我们采用第二种方案。为什么选择存储在聚集索引根页页头trx_id，页头中存储trx_id,只对二级索引页和insert buf 页头有效（MVCC)。而聚集索引根页页头trx_id这个值是没有使用的，始终保持初始值0。正好这个位置8个字节可存放自增值的值。我们每次更新AUTO_INCREMENT值时，同时将这个值修改到聚集索引根页页头trx_id的位置。 这个写操作跟真正的数据写操作一样，遵守write-ahead log原则，只不过这里只需要redo log ,而不需要undo log。因为我们不需要回滚AUTO_INCREMENT的变化（即回滚后自增列值会保留，即使insert 回滚了，AUTO_INCREMENT值不会回滚）。

因此，AUTO_INCREMENT值存储在聚集索引根页trx_id所在的位置，实际上是对内存根页的修改和多了一条redo log（量很小）,而这个redo log 的写入也是异步的，可以说是原有事务log的一个顺带操作。因此AUTO_INCREMENT值存储在聚集索引根页这个性能损耗是极小的。

修复后的性能对比，我们新增了全局参数innodb_autoinc_persistent 取值on/off； on 表示将AUTO_INCREMENT值实时存储在聚集索引根页。off则采用原有方式只存储在内存。

`./bin/sysbench --test=sysbench/tests/db/insert.lua --mysql-port=4001 --mysql-user=root \--mysql-table-engine=innodb --mysql-db=sbtest --oltp-table-size=0 --oltp-tables-count=1 \--num-threads=100 --mysql-socket=/u01/zy/sysbench/build5/run/mysql.sock --max-time=7200 --max-requests run
set global innodb_autoinc_persistent=off;
tps: 22199 rt:2.25ms
set global innodb_autoinc_persistent=on;
tps: 22003 rt:2.27ms
`

可以看出性能损耗在%1以下。

**改进**

新增参数innodb_autoinc_persistent_interval 用于控制持久化AUTO_INCREMENT值的频率。例如：innodb_autoinc_persistent_interval=100，auto_incrememt_increment=1时，即每100次insert会控制持久化一次AUTO_INCREMENT值。每次持久的值为：当前值+innodb_autoinc_persistent_interval。

测试结论

`innodb_autoinc_persistent=ON, innodb_autoinc_persistent_interval=1时性能损耗在%1以下。
innodb_autoinc_persistent=ON, innodb_autoinc_persistent_interval=100时性能损耗可以忽略。
`

**限制**

1 innodb_autoinc_persistent=on， innodb_autoinc_persistent_interval=N>1时，自增N次后持久化到聚集索引根页,每次持久的值为当前AUTO_INCREMENT+(N-1)*innodb_autoextend_increment。重启后读取持久化的AUTO_INCREMENT值会偏大，造成一些浪费但不会重复。innodb_autoinc_persistent_interval=1 每次都持久化没有这个问题。

2 如果innodb_autoinc_persistent=on，频繁设置auto_increment_increment的可能会导致持久化到聚集索引根页的值不准确。因为innodb_autoinc_persistent_interval计算没有考虑auto_increment_increment变化的情况，参看dict_table_autoinc_update_if_greater。而设置auto_increment_increment的情况极少，可以忽略。

注意：如果我们使用需要开启innodb_autoinc_persistent，应该在参数文件中指定

`innodb_autoinc_persistent= on
`

如果这样指定set global innodb_autoinc_persistent=on;重启后将不会从聚集索引根页读取AUTO_INCREMENT最大值。

疑问：对于InnoDB表，重启通过select max(id)+1 from t1得到AUTO_INCREMENT值，如果id上有索引那么这个语句使用索引查找就很快。那么，这个可以解释mysql 为什么要求自增列必须包含在索引中的原因。 如果没有指定索引，则报如下错误，

ERROR 1075 (42000): Incorrect table definition; there can be only one auto column and it must be defined as a key 而myisam表竟然也有这个要求，感觉是多余的。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)