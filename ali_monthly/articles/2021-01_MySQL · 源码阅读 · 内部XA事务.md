# MySQL · 源码阅读 · 内部XA事务

**Date:** 2021/01
**Source:** http://mysql.taobao.org/monthly/2021/01/02/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2021 / 01
 ](/monthly/2021/01)

 * 当期文章

 PolarDB · 源码解析 · 深度解析PolarDB的并行查询引擎
* MySQL · 源码阅读 · 内部XA事务
* PolarDB · 优化改进 · DDL的优化和演进
* Database · 最佳实践 · 内存索引指南
* Database · 最佳实践 · 高性能 Hash Join 算法实现简述
* MySQL · 源码阅读 · Innodb内存管理解析
* X-Engine · 引擎特性 · 并行DDL
* PostgreSQL · 新增特性 · PG 13 新特性

 ## MySQL · 源码阅读 · 内部XA事务 
 Author: 雁闲 

 ## 概述
MySQL是一个支持多存储引擎架构的数据库，除了早期默认的存储引擎myisam，目前使用比较多的引擎包括InnoDB，XEngine以及Rocksdb等，这些引擎都是支持事务的引擎，在数据库系统中，存储引擎支持事务基本是标配，所以其它引擎也就慢慢边缘化了。由于支持多事务引擎，为了保证事务一致性，MySQL实现了经典的XA标准，通过XA事务来保证事务的特征。binlog作为MySQL生态的一个重要组件，它记录了数据库操作的逻辑更新，并作为数据传输纽带，可以搭建复杂的MySQL集群，以及同步给下游。除了作为传输纽带，binlog还有一个角色就是XA事务的协调者，协调各个参与者(存储引擎)来实现XA事务的一致性。

## XA事务
MySQL的XA事务支持包括内部XA事务和外部XA事务。内部XA事务主要指单节点实例内部，一个事务跨多个存储引擎进行读写，那么就会产生内部XA事务；这里需要指出的是，MySQL内部每个事务都需要写binlog，并且需要保证binlog与引擎修改的一致性，因此binlog是一个特殊的参与者，所以在打开binlog的情况下，即使事务修改只涉及一个引擎，内部也会启动XA事务。外部XA事务与内部XA事务核心逻辑类似，提供给用户一套XA事务的操作命令，包括XA start， XA end，XA prepre和XA commit等，可以支持跨多个节点的XA事务。外部XA的协调者是用户的应用，参与者是MySQL节点，因此需要应用持久化协调信息，解决事务一致性问题。无论外部XA事务还是内部XA事务，存储引擎实现的prepare和commit接口都是同一条路径，本文重点介绍内部XA事务。

## 协调者
### 协调者的选择
MySQL内部XA事务，存储引擎是参与者，而协调者则有3个选项，包括binlog，TC_LOG_MMAP和TC_LOG_DUMMY。如果开启binlog，由于每个事务至少涉及一个存储引擎的修改，加上binlog，所以也会走XA事务流程。如果关闭binlog，事务修改涉及多个存储引擎，比如innodb和xengine引擎，那么内部会采用tc_log_map作为协调者。如果关闭binlog，且修改只涉及一个引擎innodb，那么实际上就不是XA事务，mysql内部为了保证接口统一，仍然使用了一个特殊的协调者TC_LOG_DUMMY，TC_LOG_DUMMY实际上什么也没做，只是做简单的转发，将server层的调用路由到引擎层调用，仅此而已。

`//协调者选择的逻辑
if (total_ha_2pc > 1 || (1 == total_ha_2pc && opt_bin_log))
{
 if (opt_bin_log)
 tc_log= &mysql_bin_log;
 else
 tc_log= &tc_log_mmap;
}
else
 tc_log= &tc_log_dummy
`
### 协调者逻辑
```
//binlog,tc_log_mmap和tc_log_dummy作为协调者的基本逻辑
binlog作为协调者：
prepare：ha_prepare_low
commit： write-binlog + ha_comit_low

tclog作为协调者：
prepare：ha_prepare_low
commit：wrtie-xid + ha_commit_low

tc_dummy作为协调者：
prepare：ha_prepare_low
commit：ha_commit_low 

//是否支持2PC，是否修改超过了1个以上的引擎
if (!trn_ctx->no_2pc(trx_scope) && (trn_ctx->rw_ha_count(trx_scope) > 1))
 error = tc_log->prepare(thd, all);

```

### 执行2PC依据
TC_LOG_MMAP和binlog作为协调者本质是相同的，就是在涉及跨引擎事务时，走2PC事务提交流程，分别调用引擎的prepare接口和commit接口。协调者如何确认是否走2PC逻辑，这里主要根据事务修改是否涉及多个引擎，特殊的是，如果打开binlog，binlog也会作为参与者考虑在内，最终统计事务涉及修改的参与者是否超过1，如果超过1，则进行2PC提交流程(prepare,commit)。注意，这里有一个前提条件是涉及的修改引擎必需都支持2PC。

` struct THD_TRANS { 
 /* true is not all entries in the ht[] support 2pc */ 
 bool m_no_2pc; 
 
 /* number of engine modify */
 int m_rw_ha_count; 
 
 /* storage engines that registered in this transaction */
 Ha_trx_info *m_ha_list;
 } 
 
//统计打标，是否涉及到多个引擎的修改。
ha_check_and_coalesce_trx_read_only(bool all) {
 //统计打标
 for (ha_info = ha_list; ha_info; ha_info = ha_info->next()) {
 if (ha_info->is_trx_read_write()) ++rw_ha_count;
 
 //语句级统计
 if (!all) {
 Ha_trx_info *ha_info_all =
 &thd->get_ha_data(ha_info->ht()->slot)->ha_info[1];
 DBUG_ASSERT(ha_info != ha_info_all);
 
 /* 
 Merge read-only/read-write information about statement
 transaction to its enclosing normal transaction. Do this
 only if in a real transaction -- that is, if we know
 that ha_info_all is registered in thd->transaction.all.
 Since otherwise we only clutter the normal transaction flags.
 */
 //将语句级的读写修改，同步到事务级的读写修改
 if (ha_info_all->is_started()) /* false if autocommit. */
 ha_info_all->coalesce_trx_with(ha_info);
 } else if (rw_ha_count > 1) { 
 /* 
 It is a normal transaction, so we don't need to merge read/write
 information up, and the need for two-phase commit has been
 already established. Break the loop prematurely.
 */
 break;
 } 
 } 
}
`
## 参与者
mysql内部XA事务中，参与者主要指事务型存储引擎。mysql根据引擎是否提供了prepare接口，判断引擎是否支持2PC。引擎的prepare和commit接口有一个bool类型的参数，主要含义是这次prepare/commit是语句级别，还是事务级别。事务的2PC提交流程主要都发生在事务级别，但有一个特殊场景，就是autocommit场景下的单SQL语句，这种会触发自动提交，如果这个SQL语句的修改涉及多个引擎，也会走到2PC流程。主要逻辑如下：

`prepare逻辑：
ha_prepare(bool prepare_tx) 这里的prepare_tx由外面传递的all=true/false决定。
if (prepare_tx || (!my_core::thd_test_options(thd, OPTION_NOT_AUTOCOMMIT | OPTION_BEGIN))) {
 tx->prepare
}

commit逻辑：
ha_commit(bool commit_tx) 这里的commit_tx由外面传递的all=true/false决定。
if (commit_tx || (!my_core::thd_test_options(thd, OPTION_NOT_AUTOCOMMIT | OPTION_BEGIN))) {
 tx->commit
}
`
### XA事务存储引擎接口
```
innobase_hton->commit = innobase_commit;
innobase_hton->rollback = innobase_rollback;
innobase_hton->prepare = innobase_xa_prepare;
innobase_hton->recover = innobase_xa_recover;
innobase_hton->commit_by_xid = innobase_commit_by_xid;
innobase_hton->rollback_by_xid = innobase_rollback_by_xid;

```

## Server层与引擎层交互
从前面协调者逻辑我们了解到，MySQL内部XA事务，协调者在Server层，参与者在引擎层，因此Server层和引擎层需要有一定的通信机制来确定是否要进行2PC提交。这里主要包括两方面，一个是，事务涉及到的引擎要注册到协调者的事务列表中，二是，如果引擎有修改，要将已修改的信息通知给协调者。在MySQL中主要通过两个接口来实现，xengine_register_tx/innodbase_register_tx注册事务，handler::mark_trx_read_write标记事务读写。

### DML事务
#### 注册事务路径
server根据需要访问表进行注册事务。

`mysql_lock_tables
 lock_external
 handler::ha_external_lock
 ha_innobase::external_lock
 innobase_register_trx
 trans_register_ha
 Transaction_ctx::set_ha_trx_info

void xengine_register_tx(handlerton *const hton, THD *const thd,
 Xdb_transaction *const tx) {
 DBUG_ASSERT(tx != nullptr);
 //注册stmt的trx信息
 trans_register_ha(thd, FALSE, xengine_hton, NULL);
 
 //显示开启的事务，ddl默认将AUTOCOMMIT关掉，符合条件
 if (my_core::thd_test_options(thd, OPTION_NOT_AUTOCOMMIT | OPTION_BEGIN)) {
 tx->start_stmt();
 trans_register_ha(thd, TRUE, xengine_hton, NULL);
 }
}
`
#### 标记事务修改

```
handler::ha_delete_row
handler::ha_write_row
handler::ha_update_row
 handler::mark_trx_read_write

/**
 A helper function to mark a transaction read-write,
 if it is started.
*/

void handler::mark_trx_read_write() {
 Ha_trx_info *ha_info = &ha_thd()->get_ha_data(ht->slot)->ha_info[0];
 /*
 When a storage engine method is called, the transaction must
 have been started, unless it's a DDL call, for which the
 storage engine starts the transaction internally, and commits
 it internally, without registering in the ha_list.
 Unfortunately here we can't know know for sure if the engine
 has registered the transaction or not, so we must check.
 */
 if (ha_info->is_started()) {
 DBUG_ASSERT(has_transactions());
 /*
 table_share can be NULL in ha_delete_table(). See implementation
 of standalone function ha_delete_table() in sql_base.cc.
 */
 if (table_share == NULL || table_share->tmp_table == NO_TMP_TABLE) {
 /* TempTable and Heap tables don't use/support transactions. */
 ha_info->set_trx_read_write();
 }
 }
}

```

### DDL事务
对于ddl事务，由于涉及到字典的多次修改，为了避免中途提交，临时将自动提交关闭。

`/**
 Check if statement (typically DDL) needs auto-commit mode temporarily
 turned off.

 @note This is necessary to prevent InnoDB from automatically committing
 InnoDB transaction each time data-dictionary tables are closed
 after being updated.
*/
static bool sqlcom_needs_autocommit_off(const LEX *lex) {
 return (sql_command_flags[lex->sql_command] & CF_NEEDS_AUTOCOMMIT_OFF) ||
 (lex->sql_command == SQLCOM_CREATE_TABLE &&
 !(lex->create_info->options & HA_LEX_CREATE_TMP_TABLE)) ||
 (lex->sql_command == SQLCOM_DROP_TABLE && !lex->drop_temporary);
}

/*
 For statements which need this, prevent InnoDB from automatically
 committing InnoDB transaction each time data-dictionary tables are
 closed after being updated.
*/
Disable_autocommit_guard(THD *thd) {
 m_thd->variables.option_bits &= ~OPTION_AUTOCOMMIT;
 m_thd->variables.option_bits |= OPTION_NOT_AUTOCOMMIT;
}
`
#### ddl注册事务
所有dml操作，都会通过mysql_lock_tables路径来进行注册事务操作，但对于ddl，由于有些操作只涉及数据字典的修改，server层认为不涉及引擎层修改，则不会显示注册事务。xengine通过原子ddl日志和2PC支持xengine表的ddl，需要显示注册事务，通知server层。

#### ddl标记事务修改
除了主动在server层注册事务，还需要主动将事务标记为read-write，标识这个ddl中xengine引擎有修改，这样server层在统计修改的事务引擎数时，会将xengine计算在內，最后再抉择是采用1PC事务提交还是2PC事务提交。目前，实际上在handler层的所有ddl路径，都主动调用了接口mark_trx_read_write，但由于在之前，并没有将引擎注册到server，导致整个调用对部分DDL操作无效。

## 典型场景分析
这里考虑不开binlog的场景，因为开binlog情况下，任何一个事务只要有更新，加上binlog就会走内部XA事务。不开binlog场景下，如果同时启用xengine和innodb引擎，根据事务实际情况，可能会走到2PC流程。

| 1 | 场景 | 类别 | 是否走2PC流程 | 备注 | | — | — | — | — | — | 

| 2 | (one-stmt)+(modify xengine) | DML事务 | no | 隐式事务，autocommit=on，单语句自动提交事务| 

| 3 | (one-stmt)(modify xengine+innodb) | | yes | 隐式事务，autocommit=on，单语句自动提交事务 | 

| 4 | (multi-stmt)+(modify xengine) | | no | 显示事务, 结合begin/commit | 

| 5 | (multi-stmt)+(modify xengine+innodb) | | yes | 显示事务, 结合begin/commit | 

| 6 | create table | DDL事务 | yes | storage engine mark_read_write | 

| 7 | drop table | | yes | storage engine mark_read_write | 

| 8 | rename table | | yes | storage engine mark_read_write | 

| 9 | alter table online | | yes | | 

| 10 | alter table copy-offline | | yes | | 

说明，目前tc_log作为协调者，对于双引擎XA事务在部分路径存在问题。比如，对于场景3，应该走2PC流程没有走；对于场景4，不需要走2PC流程的场景反而走了2PC。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)