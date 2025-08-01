# MySQL · 特性分析 · MySQL 5.7 外部XA Replication实现及缺陷分析

**Date:** 2017/11
**Source:** http://mysql.taobao.org/monthly/2017/11/06/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 11
 ](/monthly/2017/11)

 * 当期文章

 MySQL · 数据恢复 · undrop-for-innodb
* MySQL · 引擎特性 · DROP TABLE之binlog解析
* MSSQL · 最佳实践 · SQL Server三种常见备份
* MySQL · 最佳实践 · 什么时候该升级内存规格
* MySQL · 源码分析 · InnoDB LRU List刷脏改进之路
* MySQL · 特性分析 · MySQL 5.7 外部XA Replication实现及缺陷分析
* PgSQL · 最佳实践 · 双十一数据运营平台订单Feed数据洪流实时分析方案
* MySQL · 引擎特性 · TokuDB hot-index机制
* MySQL · 最佳实践 · 分区表基本类型
* PgSQL · 应用案例 · 流式计算与异步消息在阿里实时订单监测中的应用

 ## MySQL · 特性分析 · MySQL 5.7 外部XA Replication实现及缺陷分析 
 Author: 勉仁 

 ## MySQL 5.7 外部XA Replication实现及缺陷分析

MySQL 5.7增强了分布式事务的支持，解决了之前客户端退出或者服务器关闭后prepared的事务回滚和服务器宕机后binlog丢失的情况。

为了解决之前的问题，MySQL5.7将外部XA在binlog中的记录分成了两部分，使用两个GTID来记录。执行prepare的时候就记录一次binlog，执行commit/rollback再记录一次。由于XA是分成两部分记录，那么XA事务在binlog中就可能是交叉出现的。Slave端的SQL线程在apply的时候需要能够在这些不同事务间切换。

但MySQL XA Replication的实现只考虑了Innodb一种事务引擎的情况，当添加其他事务引擎的时候，原本的一些代码逻辑就会有问题。同时MySQL源码中也存在宕机导致主备不一致的缺陷。

## MySQL 5.7 外部XA Replication源码剖析

### Master写入

当执行 XA START ‘xid’后，内部xa_state进入XA_ACTIVE状态。

`bool Sql_cmd_xa_start::trans_xa_start(THD *thd)
{
 xid_state->set_state(XID_STATE::XA_ACTIVE);
`

第一次记录DML操作的时候，通过下面代码可以看到，对普通事务在binlog的cache中第一个event记录’BEGIN’,如果是xa_state处于XA_ACTIVE状态就记录’XA START xid’，xid为序列化后的。

`static int binlog_start_trans_and_stmt(THD *thd, Log_event *start_event)
{
 if (cache_data->is_binlog_empty())
 {
 if (is_transactional && xs->has_state(XID_STATE::XA_ACTIVE))
 {
 /*
 XA-prepare logging case.
 */
 qlen= sprintf(xa_start, "XA START %s", xs->get_xid()->serialize(buf));
 query= xa_start;
 }
 else
 {
 /*
 Regular transaction case.
 */
 query= begin;
 }

 Query_log_event qinfo(thd, query, qlen,
 is_transactional, false, true, 0, true);
 if (cache_data->write_event(thd, &qinfo))
 DBUG_RETURN(1);
`

XA END xid的执行会将xa_state设置为XA_IDLE。

`bool Sql_cmd_xa_end::trans_xa_end(THD *thd)
{
 xid_state->set_state(XID_STATE::XA_IDLE);
`

当XA PREPARE xid执行的时候，binlog_prepare会通过检查thd的xa_state是否处于XA_IDLE状态来决定是否记录binlog。如果在对应状态，就会调用MYSQL_BINLOG的commit函数，记录’XA PREPARE xid’，将之前cache的binlog写入到文件。

`static int binlog_prepare(handlerton *hton, THD *thd, bool all)
{
 DBUG_RETURN(all && is_loggable_xa_prepare(thd) ?
 mysql_bin_log.commit(thd, true) : 0);

inline bool is_loggable_xa_prepare(THD *thd)
{
 return DBUG_EVALUATE_IF("simulate_commit_failure",
 false,
 thd->get_transaction()->xid_state()->
 has_state(XID_STATE::XA_IDLE));

TC_LOG::enum_result MYSQL_BIN_LOG::commit(THD *thd, bool all)
{
 if (is_loggable_xa_prepare(thd))
 {
 XID_STATE *xs= thd->get_transaction()->xid_state();
 XA_prepare_log_event end_evt(thd, xs->get_xid(), one_phase);
 err= cache_mngr->trx_cache.finalize(thd, &end_evt, xs)
 }
`

当XA COMMIT/ROLLBACK xid执行时候，调用do_binlog_xa_commit_rollback记录’XA COMMIT/ROLLBACK xid’。

`TC_LOG::enum_result MYSQL_BIN_LOG::commit(THD *thd, bool all)
{
 if (thd->lex->sql_command == SQLCOM_XA_COMMIT)
 do_binlog_xa_commit_rollback(thd, xs->get_xid(),
 true)))

int MYSQL_BIN_LOG::rollback(THD *thd, bool all)
{
 if (thd->lex->sql_command == SQLCOM_XA_ROLLBACK)
 if ((error= do_binlog_xa_commit_rollback(thd, xs->get_xid(), false)))
`

由于XA PREPARE单独记录binlog，那么binlog中的events一个xa事务就可能是分隔开的。举个例子，session1中xid为’a’的分布式事务执行xa prepare后，session2中执行并提交了xid为’z’的事务，然后xid ‘a’才提交。我们可以看到binlog events中xid ‘z’的events在’a’的prepare和commit之间。

`session1:
xa start 'a';
insert into t values(1);
xa end 'a';
xa prepare 'a';

session2:
xa start 'z';
insert into t values(2);
xa end 'z';
xa prepare 'z';
xa commit 'z';

session1:
xa commit 'a';

| mysql-bin.000008 | 250 | Gtid | 324 | 298 | SET @@SESSION.GTID_NEXT= 'uuid:9' |
| mysql-bin.000008 | 298 | Query | 324 | 385 | XA START X'61',X'',1 |
| mysql-bin.000008 | 385 | Table_map | 324 | 430 | table_id: 72 (test.t) |
| mysql-bin.000008 | 430 | Write_rows_v1 | 324 | 476 | table_id: 72 flags: STMT_END_F |
| mysql-bin.000008 | 476 | Query | 324 | 561 | XA END X'61',X'',1 |
| mysql-bin.000008 | 561 | XA_prepare | 324 | 598 | XA PREPARE X'61',X'',1 |
| mysql-bin.000008 | 598 | Gtid | 324 | 646 | SET @@SESSION.GTID_NEXT= 'uuid:10' |
| mysql-bin.000008 | 646 | Query | 324 | 733 | XA START X'7a',X'',1 |
| mysql-bin.000008 | 733 | Table_map | 324 | 778 | table_id: 72 (test.t) |
| mysql-bin.000008 | 778 | Write_rows_v1 | 324 | 824 | table_id: 72 flags: STMT_END_F |
| mysql-bin.000008 | 824 | Query | 324 | 909 | XA END X'7a',X'',1 |
| mysql-bin.000008 | 909 | XA_prepare | 324 | 946 | XA PREPARE X'7a',X'',1 |
| mysql-bin.000008 | 946 | Gtid | 324 | 994 | SET @@SESSION.GTID_NEXT= 'uuid:11' |
| mysql-bin.000008 | 994 | Query | 324 | 1082 | XA COMMIT X'7a',X'',1 |
| mysql-bin.000008 | 1082 | Gtid | 324 | 1130 | SET @@SESSION.GTID_NEXT= 'uuid:12' |
| mysql-bin.000008 | 1130 | Query | 324 | 1218 | XA COMMIT X'61',X'',1 |
`

### Slave 重放

由于XA事务在binlog中是会交叉出现的，Slave的SQL线程如果按照原本普通事务的方式重放，那么就会出现SQL线程中还存在处于prepared状态的事务，就开始处理下一个事务了，锁状态、事务状态等会错乱。所以SQL线程需要能够支持这种情况下不同事务间的切换。

SQL线程要做到能够在执行XA事务时切换到不同事务，需要做到server层保留原有xid的Transaction_ctx信息，引擎层也保留原有xid的事务信息。

server层保留原有xid的Transaction_ctx信息是通过在prepare的时候将thd中xid的Transaction_ctx信息从transacion_cache中detach掉，创建新的保留了XA事务信息的Transaction_ctx放入transaction_cache中。

`bool Sql_cmd_xa_prepare::execute(THD *thd)
 !(st= applier_reset_xa_trans(thd)))

bool applier_reset_xa_trans(THD *thd)
 transaction_cache_detach(trn_ctx);

bool transaction_cache_detach(Transaction_ctx *transaction)
 res= create_and_insert_new_transaction(&xid, was_logged);
`

引擎层的实现并不是通过在prepare的时候创建新trx_t的来保存原有事务信息。而是在XA START的时候将原来thd中所有的engine ha_data单独保留起来，为XA事务创建新的。在XA PREPARE的时候，再将原来的reattach回来，将XA的从thd detach掉，解除XA和thd的关联。引擎层添加了新的接口replace_native_transaction_in_thd来支持上述操作。对于Slave的SQL线程，函数调用如下：

`//engine 新添加的接口
struct handlerton
{
 void (*replace_native_transaction_in_thd)(THD *thd, void *new_trx_arg, void **ptr_trx_arg);

//XA START函数调用
bool Sql_cmd_xa_start::execute(THD *thd)
{
 thd->rpl_detach_engine_ha_data();

void THD::rpl_detach_engine_ha_data()
{
 rli->detach_engine_ha_data(this);

//每个Storage engine都调用detach_native_trx
void Relay_log_info::detach_engine_ha_data(THD *thd)
{
 plugin_foreach(thd, detach_native_trx,
 MYSQL_STORAGE_ENGINE_PLUGIN, NULL);

my_bool detach_native_trx(THD *thd, plugin_ref plugin, void *unused)
{
 if (hton->replace_native_transaction_in_thd)
 hton->replace_native_transaction_in_thd(thd, NULL,
 thd_ha_data_backup(thd, hton));

//XA PREPARE函数调用
bool Sql_cmd_xa_prepare::execute(THD *thd)
{
 !(st= applier_reset_xa_trans(thd)))

bool applier_reset_xa_trans(THD *thd)
{
 attach_native_trx(thd);

//对事务涉及到的引擎调用reattach_engine_ha_data_to_thd。
static void attach_native_trx(THD *thd)
{
 if (ha_info)
 {
 for (; ha_info; ha_info= ha_info_next)
 {
 handlerton *hton= ha_info->ht();
 reattach_engine_ha_data_to_thd(thd, hton);
 ha_info_next= ha_info->next();
 ha_info->reset();
 }
 }

inline void reattach_engine_ha_data_to_thd(THD *thd, const struct handlerton *hton)
{
 if (hton->replace_native_transaction_in_thd)
 hton->replace_native_transaction_in_thd(thd, *trx_backup, NULL);
`
当XA COMMIT/ROLLBACK执行的时候，如果当前thd中没有对应的xid，就会从transaction_cache中查找对应xid的state信息，然后调用各个引擎的commit_by_xid/rollback_by_xid接口提交/回滚XA事务。

`bool Sql_cmd_xa_commit::trans_xa_commit(THD *thd)
{
 if (!xid_state->has_same_xid(m_xid))
 {
 Transaction_ctx *transaction= transaction_cache_search(m_xid);
 ha_commit_or_rollback_by_xid(thd, m_xid, !res);

static void ha_commit_or_rollback_by_xid(THD *thd, XID *xid, bool commit)
{
 plugin_foreach(NULL, commit ? xacommit_handlerton : xarollback_handlerton,
 MYSQL_STORAGE_ENGINE_PLUGIN, xid);

static my_bool xacommit_handlerton(THD *unused1, plugin_ref plugin, void *arg)
{
 if (hton->state == SHOW_OPTION_YES && hton->recover)
 hton->commit_by_xid(hton, (XID *)arg);

static my_bool xarollback_handlerton(THD *unused1, plugin_ref plugin, void *arg)
{
 if (hton->state == SHOW_OPTION_YES && hton->recover)
 hton->rollback_by_xid(hton, (XID *)arg); 
`

由于XA COMMIT/XA ROLLBACK是单独作为一部分，这部分并没有原来XA事务涉及到库、表的信息，所以XA COMMIT在Slave端当slave-parallel-type为DATABASE时是无法并发执行的，在slave端强制设置mts_accessed_dbs为OVER_MAX_DBS_IN_EVENT_MTS使其串行执行。

`bool Log_event::contains_partition_info(bool end_group_sets_max_dbs)
{
 case binary_log::QUERY_EVENT:
 {
 Query_log_event *qev= static_cast<Query_log_event*>(this);
 if ((ends_group() && end_group_sets_max_dbs) ||
 (qev->is_query_prefix_match(STRING_WITH_LEN("XA COMMIT")) ||
 qev->is_query_prefix_match(STRING_WITH_LEN("XA ROLLBACK"))))
 {
 res= true;
 qev->mts_accessed_dbs= OVER_MAX_DBS_IN_EVENT_MTS;
 }
`

## MySQL5.7 外部XA Replication实现的缺陷分析

### Prepare阶段可能导致主备不一致

MySQL中普通事务提交的时候，需要先在引擎中prepare，然后再写binlog，之后再做引擎commit。但在MySQL执行XA PREPARE的时候先写入了binlog，然后才做引擎的prepare。如果引擎在做prepare的时候失败或者服务器crash就会导致binlog和引擎不一致，主备进入不一致的状态。

在MySQL5.7中对模拟simulate_xa_failure_prepare的DEBUG情况做如下修改，使之模拟在Innodb引擎prepare的时候失败。

`--- a/sql/handler.cc
+++ b/sql/handler.cc
@@ -1460,10 +1460,12 @@ int ha_prepare(THD *thd)
 thd->status_var.ha_prepare_count++;
 if (ht->prepare)
 {
- DBUG_EXECUTE_IF("simulate_xa_failure_prepare", {
- ha_rollback_trans(thd, true);
- DBUG_RETURN(1);
- });
+ if (ht->db_type == DB_TYPE_INNODB) {
+ DBUG_EXECUTE_IF("simulate_xa_failure_prepare", {
+ ha_rollback_trans(thd, true);
+ DBUG_RETURN(1);
+ });
+ }
 if (ht->prepare(ht, thd, true))
 {
 ha_rollback_trans(thd, true);
`

然后运行下面的case，可以看到Master上的XA失败后被回滚。但由于这个时候已经写入了binlog events，导致Slave端执行了XA事务，留下一个处于prepared状态的XA事务。

`replication.test:

--disable_warnings
source include/master-slave.inc;
--enable_warnings
connection master;
CREATE TABLE ti (c1 INT) ENGINE=INNODB;
XA START 'x';
INSERT INTO ti VALUES(1);
XA END 'x';
SET @@session.debug = '+d,simulate_xa_failure_prepare';
--error ER_XA_RBROLLBACK
XA PREPARE 'x';
--echo #Master
XA RECOVER;

--sync_slave_with_master
connection slave;
--echo #Slave
XA RECOVER;

replication.result:

include/master-slave.inc
[connection master]
CREATE TABLE ti (c1 INT) ENGINE=INNODB;
XA START 'x';
INSERT INTO ti VALUES(1);
XA END 'x';
SET @@session.debug = '+d,simulate_xa_failure_prepare';
XA PREPARE 'x';
ERROR XA100: XA_RBROLLBACK: Transaction branch was rolled back
#Master
XA RECOVER;
formatID gtrid_length bqual_length data
#Slave
XA RECOVER;
formatID gtrid_length bqual_length data
1 1 0 x
`

在MySQL5.7源码中，如果在binlog和InnoDB引擎都prepare之后是不是数据就安全了呢？我们在ha_prepare函数中while循环调用完所有引擎prepare函数之后添加如下DEBUG代码，可以控制在prepare调用结束后服务器crash掉。

`--- a/sql/handler.cc
+++ b/sql/handler.cc
@@ -1479,6 +1479,7 @@ int ha_prepare(THD *thd)
 }
 ha_info= ha_info->next();
 }
+ DBUG_EXECUTE_IF("crash_after_xa_prepare", DBUG_SUICIDE(););

 DBUG_ASSERT(thd->get_transaction()->xid_state()->
 has_state(XID_STATE::XA_IDLE));
`

然后跑下面的testcase。可以看到即使所有引擎都prepare了，宕机重启后XA RECOVER还是还是没有能够找回之前prepare的事务。而且这个时候我们查看binlog文件可以看到binlog已经写成功，这也会导致主备不一致。很明显，应该是InnoDB引擎丢失了prepare的日志。这里是由于先调用binlog_prepare，thd->durability_property被配置为HA_IGNORE_DURABILITY。感兴趣的同学可以查看MYSQL_BIN_LOG::prepare、MYSQL_BIN_LOG::ordered_commit(THD *thd, bool all, bool skip_commit)和innobase中trx_prepare的代码，看process_flush_stage_queue和flush_logs和thd->durability_property的相关逻辑。这里不再展开详细叙述。

`replication.test:

-- source include/have_log_bin.inc
CREATE TABLE ti (c1 INT) ENGINE=INNODB;
XA START 'x';
INSERT INTO ti VALUES(1);
XA END 'x';
SET @@session.debug = '+d,crash_after_xa_prepare';
--exec echo "wait" > $MYSQLTEST_VARDIR/tmp/mysqld.1.expect
--error 2013
XA PREPARE 'x';
--source include/wait_until_disconnected.inc
--let $_expect_file_name= $MYSQLTEST_VARDIR/tmp/mysqld.1.expect
--source include/start_mysqld.inc
XA RECOVER;
show binlog events in 'mysql.000001';

replication.result:
CREATE TABLE ti (c1 INT) ENGINE=INNODB;
XA START 'x';
INSERT INTO ti VALUES(1);
XA END 'x';
SET @@session.debug = '+d,crash_after_xa_prepare';
XA PREPARE 'x';
ERROR HY000: Lost connection to MySQL server during query
# restart
XA RECOVER;
formatID gtrid_length bqual_length data
show binlog events in 'mysql.000001';
Log_name Pos Event_type Server_id End_log_pos Info
mysql.000001 4 Format_desc 1 123 Server ver: 5.7.19org-debug-log, Binlog ver: 4
mysql.000001 123 Previous_gtids 1 154
mysql.000001 154 Anonymous_Gtid 1 219 SET @@SESSION.GTID_NEXT= 'ANONYMOUS'
mysql.000001 219 Query 1 331 use `test`; CREATE TABLE ti (c1 INT) ENGINE=INNODB
mysql.000001 331 Anonymous_Gtid 1 396 SET @@SESSION.GTID_NEXT= 'ANONYMOUS'
mysql.000001 396 Query 1 483 XA START X'78',X'',1
mysql.000001 483 Table_map 1 528 table_id: 222 (test.ti)
mysql.000001 528 Write_rows 1 568 table_id: 222 flags: STMT_END_F
mysql.000001 568 Query 1 653 XA END X'78',X'',1
mysql.000001 653 XA_prepare 1 690 XA PREPARE X'78',X'',1
`

上面两个问题的修复，可以参考笔者在提给社区[bug#88534](https://bugs.mysql.com/bug.php?id=88534) 中最后的解决方案。将XA PREPARE/COMMIT相关流程修改如下，对于XA事务的状态由binlog的记录决定：

`'XA PREPARE':engine->prepare, binlog->prepare.
'XA COMMIT one phase': engine->prepare, binlog->commit, engine->commit.
'XA COMMIT': binlog->commit, engine->commit (ensure engine commit before binary log rotate).
'XA ROLLBACK': binlog->rollback, engine->rollback (ensure engine rollback before binary log rotate).
`

同时因为XA事务可能在PREPARE/COMMIT在不同binlog文件中，我们添加Previous_prepared_xids_log_event binlog event放在新建binlog文件的最前面，这样我们就可以在异常重启的时候通过读取当前binlog文件知道每个XA事务的状态。

由于外部XA事务允许前后出现同名的xid，如果一个XA事务提交后，另一个同名的XA事务进入PREPARE阶段仅完成了引擎的prepare，那这个时候我们通过binlog信息就推断出后者已经是COMMIT阶段了，导致错误发生。对于这种情况，我们可以添加XA_preparing_log_event，在外部XA事务PREPARE阶段，先写该event，在做引擎的PREPARE。这样对于前述问题，我们可以推算出是同XID的新事务进入了PREPARE阶段，完成了引擎prepare，但binlog还未记录，属于prepare失败，需要回滚。

### 不支持server中使用多个事务引擎

在上面实现分析中可以看到Slave在执行XA START的时候，由于这个时候并不知道该XA事务涉及到哪些引擎，所以对所有Storage engine引擎都调用了detach_native_trx。但在XA PREPARE的时候，源码中只对XA涉及到的引擎调用了reattach_engine_ha_data_to_thd。对于引擎可插拔的MySQL来说，当server中不止一个事务引擎，这里就会存在有的引擎原thd中的trx被detach后没有被reattach。

我们可以拿支持tokudb的percona server做对应实验。对DEBUG编译的server，执行下面replication的testcase。该case对TokuDB做一个完整的XA事务后，再向Innodb写入。运行该case，slave端会产生assert_fail的错误。因为TokuDB执行XA事务时，将Innodb的ha_data放入backup，但由于Innodb没有参与该XA事务，所以并没有reattach，导致gdb可以看到assert_fail处InnoDB的ha_ptr_backup不为NULL，不符合预期。

`replication.test
--disable_warnings
source include/master-slave.inc;
--enable_warnings
connection master;
create table tk(c1 int) engine=tokudb;
create table ti(c1 int) engine=innodb;

xa start 'x';
insert into tk values(1);
xa end 'x';
xa prepare 'x';
xa commit 'x';

insert into ti values(2);

__assert_fail
thd->ha_data[ht_arg->slot].ha_ptr_backup == __null || (thd->get_transaction()->xid_state()-> has_state(XID_STATE::XA_ACTIVE))"

(gdb) p thd->ha_data[ht_arg->slot].ha_ptr_backup
$1 = (void *) 0x2b11e0401070
`

修复问题，可以在需要reattach_engine_ha_data_to_thd的代码处，对所有storage engine再次调用该操作。

### 不支持新接口的事务引擎重放新XA事务会出错

对于不支持reattach_engine_ha_data_to_thd的事务引擎实际是不支持重放MySQL5.7新XA方式生成的binlog的，但在源码中并没有合适禁止操作。这就会导致slave在apply的时候数据错乱。

继续使用支持tokudb的percona server做实验。由于TokuDB并没有实现reattach_engine_ha_data_to_thd接口，Slave在重放XA事务的时候，在TokuDB引擎中实际就在原本关联thd的trx上操作，并没有生成新的trx。这就会导致数据等信息错乱，可以看到下面的例子。session1做了一个XA事务，插入数值1，prepare后并没有提交。随后另一个session插入数值2，但在slave同步后，数值2无法查询到。在session1提交了XA事务，写入TokuDB的数值1、2才在slave端查询到。

`replication.test:

--disable_warnings
source include/master-slave.inc;
--enable_warnings
connection master;
--echo #Master
create table tk(c1 int) engine=tokudb;
xa start 'x';
insert into tk values(1);
xa end 'x';
xa prepare 'x';
connect(m, localhost, root, , test, $MASTER_MYPORT);
insert into tk values(2);
select * from tk;

--sync_slave_with_master
connection slave;
--echo #Slave
select * from tk;

connection master;
--echo #Master
xa commit 'x';
select * from tk;

--sync_slave_with_master
connection slave;
--echo #Slave
select * from tk;

connection master;
drop table tk;

replication.result:

include/master-slave.inc
[connection master]
#Master
create table tk(c1 int) engine=tokudb;
xa start 'x';
insert into tk values(1);
xa end 'x';
xa prepare 'x';
insert into tk values(2);
select * from tk;
c1
2
#Slave
select * from tk;
c1
#Master
xa commit 'x';
select * from tk;
c1
1
2
#Slave
select * from tk;
c1
1
2
drop table tk;
`

修复该问题，需要对没有实现新接口的事务引擎在执行XA时候给与合适的禁止操作，同时需要支持新XA的事务引擎要实现reattach_engine_ha_data_to_thd接口。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)