# MySQL · 答疑解惑 · binlog event 中的 error code

**Date:** 2015/06
**Source:** http://mysql.taobao.org/monthly/2015/06/05/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 06
 ](/monthly/2015/06)

 * 当期文章

 MySQL · 引擎特性 · InnoDB 崩溃恢复过程
* MySQL · 捉虫动态 · 唯一键约束失效
* MySQL · 捉虫动态 · ALTER IGNORE TABLE导致主备不一致
* MySQL · 答疑解惑 · MySQL Sort 分页
* MySQL · 答疑解惑 · binlog event 中的 error code
* PgSQL · 功能分析 · Listen/Notify 功能
* MySQL · 捉虫动态 · 任性的 normal shutdown
* PgSQL · 追根究底 · WAL日志空间的意外增长
* MySQL · 社区动态 · MariaDB Role 体系
* MySQL · TokuDB · TokuDB数据文件大小计算

 ## MySQL · 答疑解惑 · binlog event 中的 error code 
 Author: 襄洛 

 ## 问题描述

RDS 有个任务叫做恢复到任意时间点，相当于一个数据时光机，可以将数据恢复到过去任意一个时间点，在用户出现误操作需要将数据找回时非常有用。这个功能主要是通过备份集恢复 + binlog回放实现，在用备份集恢复出的实例上应用 binlog 到指定时间点。

然而最近线上重放binlog时遇到了这样一个错误：

`Table xxxx already exists
`

查看对应 binlog 的，发现这是一个 CREATE VIEW 语句，而备份集恢复出来的实例上确实已经有了这个view，再往前翻看binlog，并没有发现 DROP 这个 view 的记录，倒是找到了CREATE 这个 view 的记录，仔细比较2处 CREATE VIEW 的binlog event，会发现后者多了个 error_code=1050，这个是什么错呢：

`$perror 1050
MySQL error code 1050 (ER_TABLE_EXISTS_ERROR): Table '%-.192s' already exists
`

1050 对应的错就是 Table already exists

就是说 CREATE VIEW 失败了，仍然记入 binlog 了，但是当时备库并没有这个错误中断掉。

## 复现步骤

复现非常简单，连着执行同一个create view语句即可。

`mysql> create table t1(id int, name varchar(30)) engine=innodb;
Query OK, 0 rows affected (0.02 sec)

mysql> create view t1_v as select id from t1;
Query OK, 0 rows affected (0.01 sec)

mysql> create view t1_v as select id from t1;
ERROR 1050 (42S01): Table 't1_v' already exists
`

查看binlog event

`#150614 23:15:02 server id 36302 end_log_pos 2651 CRC32 0x8f8b6c61 GTID [commit=yes]
SET @@SESSION.GTID_NEXT= '94cdda9b-a2d0-11e4-ade1-a0d3c1f20ae4:68157343'/*!*/;
# at 2651
#150614 23:15:02 server id 36302 end_log_pos 2856 CRC32 0x703fbe6d Query thread_id=21475 exec_time=0 error_code=0
SET TIMESTAMP=1434294902/*!*/;
CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`127.0.0.1` SQL SECURITY DEFINER VIEW `t1_v` AS select id from t1
/*!*/;
# at 2856
#150614 23:15:02 server id 36302 end_log_pos 2904 CRC32 0xfc2ef7cb GTID [commit=yes]
SET @@SESSION.GTID_NEXT= '94cdda9b-a2d0-11e4-ade1-a0d3c1f20ae4:68157344'/*!*/;
# at 2904
#150614 23:15:02 server id 36302 end_log_pos 3109 CRC32 0x0e807965 Query thread_id=21475 exec_time=0 error_code=1050
SET TIMESTAMP=1434294902/*!*/;
CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`127.0.0.1` SQL SECURITY DEFINER VIEW `t1_v` AS select id from t1
/*!*/;
`
可以清楚的看到，第二次 CREATE VIEW 时error_code 为 1050。

## 分析

查看 binlog 对应的代码，发现 error_code 这个字段是 `Query_log_event` 的专属，其它的如 row_event、gtid event等都没有这个字段。而备库在执行`Query_log_event` 时会检查event 的 error_code(存入expected_error），如果非0的话，就和当前SQL线程执行出错（存入actual_error）比较，看是否一致，如果一致的话就算执行成功，如果不一致的话，就再检查这个错是否能够忽略，如配置了 slave_skip_errors，代码片段如下（在`Query_log_event::do_apply_event`中）:

`/*
If we expected a non-zero error code, and we don't get the same error
code, and it should be ignored or is related to a concurrency issue.
*/
actual_error= thd->is_error() ? thd->get_stmt_da()->sql_errno() : 0;
DBUG_PRINT("info",("expected_error: %d sql_errno: %d",
expected_error, actual_error));

if ((expected_error && expected_error != actual_error &&
!concurrency_error_code(expected_error)) &&
!ignored_error_code(actual_error) &&
!ignored_error_code(expected_error))
{
rli->report(ERROR_LEVEL, 0,
"\
Query caused different errors on master and slave. \
Error on master: message (format)='%s' error code=%d ; \
Error on slave: actual message='%s', error code=%d. \
Default database: '%s'. Query: '%s'",
ER_SAFE(expected_error),
expected_error,
actual_error ? thd->get_stmt_da()->message() : "no error",
actual_error,
print_slave_db_safe(db), query_arg);
thd->is_slave_error= 1;
}
`

正常的想法应该是执行出错，就不应该记binlog，为什么会有这样的设计呢，主库错，记binlog，然后备库要求同样的错。
因为DDL是不能回滚的，如果DDL执行到一半报错，主库又不能回滚，那么应该如何通知备库它做了一半呢？就是把错记下去，期待备库也报同样的错。

挖一下黑历史，`Query_log_event` 中的 `error_code` 字段最早是在这个[commit](https://github.com/mysql/mysql-server/commit/204ae8473262f37d40f27aa35505b0492128cb7d)中加入的，目的是将主库上执行出错的信息传给备库，备库执行的时候会检测实际的出错信息和主库传过来的binlog中记录的是否是一样的，不一样就报错。

在此之前，备库对于 `Query_log_event` 执行出错是这样处理的，先检查SQL线程执行出错是不是因为表不存在，如果是的话，就单独再开个连接，从主库把不存在的表导过来(`fetch_nx_table`)，然后再重试执行失败的event，如果还有不存在的表，就再拉，再重复执行；对于其它的错就直接报错。
现在看起来是不是很奇葩，2000年的时候，MySQL还是很年青的哇 =_=

## 总结

我们在回放binlog的时候用的是mysql client，不是SQL线程，mysql client中并没有对error_cocd的处理逻辑，因此遇到执行出错就直接报错了。

所以如果脚本或者代码里有这种重放binlog逻辑的，需要注意处理这种场景。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)