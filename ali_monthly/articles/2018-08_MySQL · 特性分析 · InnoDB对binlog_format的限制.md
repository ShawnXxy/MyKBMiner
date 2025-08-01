# MySQL · 特性分析 · InnoDB对binlog_format的限制

**Date:** 2018/08
**Source:** http://mysql.taobao.org/monthly/2018/08/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 08
 ](/monthly/2018/08)

 * 当期文章

 MySQL · 引擎特性 · 主库 binlog 概览
* MySQL · RocksDB · Write Prepared Policy
* MSSQL · 最佳实践 · 使用对称秘钥实现列加密
* MySQL · 特性分析 · InnoDB对binlog_format的限制
* MongoDB · 引擎特性 · sharding chunk 分裂与迁移详解
* PgSQL · 源码分析 · PostgreSQL物理备份内部原理
* MySQL · 源码分析 · 连接与认证过程
* MySQL · RocksDB · MemTable的写入逻辑
* PgSQL · 最佳实践 · Greenplum RoaringBitmap多阶段聚合
* PgSQL · 应用案例 · 高并发空间位置更新、多属性KNN搜索并测

 ## MySQL · 特性分析 · InnoDB对binlog_format的限制 
 Author: 勉仁 

 ## 前言

我们都知道binlog_format为STATEMENT在一些场景下能够节省IO、加快同步速度，但是对于InnoDB这种事务引擎，在READ-COMMITTED、READ-UNCOMMITTED隔离级别或者参数innodb_locks_unsafe_for_binlog为ON时，禁止binlog_format=statement下的写入，同时对于binlog_format=mixed这种对于非事务引擎、其他隔离级别默认写statement格式的模式也只会记录row格式。

## 示例

`> select @@tx_isolation;
+----------------+
| @@tx_isolation |
+----------------+
| READ-COMMITTED |
+----------------+

> create table t(c1 int) engine=innodb;

> set binlog_format=statement;

> insert into t values(1);
ERROR 1665 (HY000): Cannot execute statement: impossible to write to binary log since BINLOG_FORMAT = STATEMENT and at least one table uses a storage engine limited to row-based logging. InnoDB is limited to row-logging when transaction isolation level is READ COMMITTED or READ UNCOMMITTED.

> set binlog_format='mixed';

> show binlog events in 'mysql-bin.002044'\G
*************************** 3. row ***************************
 Log_name: mysql-bin.002044
 Pos: 287
 Event_type: Gtid
 Server_id: 3249401818
End_log_pos: 335
 Info: SET @@SESSION.GTID_NEXT= 'ed0eab2f-dfb0-11e7-8ad8-a0d3c1f20ae4:9375'
*************************** 4. row ***************************
 Log_name: mysql-bin.002044
 Pos: 335
 Event_type: Query
 Server_id: 3249401818
End_log_pos: 407
 Info: BEGIN
*************************** 5. row ***************************
 Log_name: mysql-bin.002044
 Pos: 407
 Event_type: Table_map
 Server_id: 3249401818
End_log_pos: 452
 Info: table_id: 124 (test.t)
*************************** 6. row ***************************
 Log_name: mysql-bin.002044
 Pos: 452
 Event_type: Write_rows_v1
 Server_id: 3249401818
End_log_pos: 498
 Info: table_id: 124 flags: STMT_END_F
*************************** 7. row ***************************
 Log_name: mysql-bin.002044
 Pos: 498
 Event_type: Xid
 Server_id: 3249401818
End_log_pos: 529
 Info: COMMIT /* xid=18422 */
`

## 分析
为什么READ-COMMITTED(RC)、READ-UNCOMMITTED下无法使用statement格式binlog？这是因为语句在事务中执行时，能够看到其他事务提交或者正在写入的数据。事务提交后binlog写入在备库回放时候，其看到的数据会与主库写入时候不对应。例如MySQL [Bug23051](https://bugs.mysql.com/bug.php?id=23051)中的例子：当master session在事务中做update的时候满足条件的只有行(10,2)，然后master1 session将行(20,1)更新为(20,2)提交，然后前面的master sesssion提交对行(10,2)的更新。如果记录为Statement，在slave回放的时候，master1 session中的更新由于先提交会先回放，将行(20,1)更新为（20,2)。随后回放master session的语句UPDATE t1 SET a=11 where b=2;语句就会将更新(10,2)和(20,2)两行为(11,2)。这就导致主库行为(11, 2), (20,2)，slave端为(11,2), (11, 2)。

`-- source include/master-slave.inc
-- source include/have_innodb.inc

connection master;
CREATE TABLE `t1` (
 `a` int(11) DEFAULT NULL,
 `b` int(11) DEFAULT NULL,
 KEY `a` (`a`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;

insert into t1 values(10,2),(20,1);
show create table t1;

connection master;
set session transaction isolation level read committed;

set autocommit=0;
UPDATE t1 SET a=11 where b=2;

connection master1;
set session transaction isolation level read committed;
set autocommit=0;
UPDATE t1 SET b=2 where b=1;
COMMIT;

connection master;
COMMIT;

select * from t1;
show binlog events;
sync_slave_with_master;
show create table t1;
select * from t1;

connection master;
drop table t1;
sync_slave_with_master;
`

上面是通过一个具体的例子说明。本质原因是RC事务隔离级别并不满足事务串行化执行要求，没有解决不可重复和幻象读。

那么对于Repetable-Read和Serializable隔离级别就没关系么？这是因为对于RR和Serializable，会保证可重复读，在执行更新时候除了锁定对应行还会在可能插入满足条件行的时候加GAP Lock。上述case更新时，Master session更新b =2的行时，会把所有行和范围都锁住，这样master1在更新的时候就需要等待。从隔离级别的角度看Serializable满足事务的串行化，因此binlog串行记录事务statement格式是可以的。同时InnoDB的RR隔离级别实际已经解决了不可重复读和幻象读，满足了ANSI SQL标准的事务隔离性要求。

READ-COMMITTED、READ-UNCOMMITTED的binlog_format限制可以说对于所有事务引擎都适用。

对于InnoDB RR和Serializable隔离级别下就一定能保证binlog记录Statement格式么？Innodb存在参数innodb_locks_unsafe_for_binlog控制GAP Lock，该参数默认为OFF，即RR级别及以上除了行锁还会加GAP Lock。但如果该参数设置为ON，对于当前读就不会加GAP Lock，即在RR隔离级别下需要加Next-key lock的当前读蜕化为READ-COMMITTED，上述场景中Master1 session就可以更新成功。

## 总结
所以对于线上业务，如果使用InnoDB等事务引擎，除非保证RR及以上隔离级别的写入，一定不要设置为binlog_format为STATEMENT，否则业务就无法写入了。而对于binlog_format为Mixed模式，RR隔离级别以下这些事务引擎也一定写入的是ROW event。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)