# MySQL · 引擎特性 · InnoDB mini transation

**Date:** 2017/10
**Source:** http://mysql.taobao.org/monthly/2017/10/03/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 10
 ](/monthly/2017/10)

 * 当期文章

 PgSQL · 特性分析 · MVCC机制浅析
* MySQL · 性能优化· CloudDBA SQL优化建议之统计信息获取
* MySQL · 引擎特性 · InnoDB mini transation
* MySQL · 特性介绍 · 一些流行引擎存储格式简介
* MSSQL · 架构分析 · 从SQL Server 2017发布看SQL Server架构的演变
* MySQL · 引擎介绍 · Sphinx源码剖析(三)
* PgSQL · 内核开发 · 如何管理你的 PostgreSQL 插件
* MySQL · 特性分析 · 数据一样checksum不一样
* PgSQL · 应用案例 · 经营、销售分析系统DB设计之共享充电宝
* MySQL · 捉虫动态 · 信号处理机制分析

 ## MySQL · 引擎特性 · InnoDB mini transation 
 Author: xijia 

 ## 前言
InnoDB有两个非常重要的日志，undo log 和 redo log；通过undo log可以看到数据较早版本，实现MVCC，或回滚事务等功能；redo log用来保证事务持久性

本文以一条insert语句为线索介绍 mini transaction

## mini transaction 简介
mini transation 主要用于innodb redo log 和 undo log写入，保证两种日志的ACID特性

mini-transaction遵循以下三个协议:

1. The FIX Rules
2. Write-Ahead Log
3. Force-log-at-commit

#### The FIX Rules
修改一个页需要获得该页的x-latch

访问一个页是需要获得该页的s-latch或者x-latch

持有该页的latch直到修改或者访问该页的操作完成

#### Write-Ahead Log
持久化一个数据页之前，必须先将内存中相应的日志页持久化

每个页有一个LSN,每次页修改需要维护这个LSN,当一个页需要写入到持久化设备时，要求内存中小于该页LSN的日志先写入到持久化设备中

#### Force-log-at-commit
一个事务可以同时修改了多个页，Write-AheadLog单个数据页的一致性，无法保证事务的持久性

Force -log-at-commit要求当一个事务提交时，其产生所有的mini-transaction日志必须刷到持久设备中

这样即使在页数据刷盘的时候宕机，也可以通过日志进行redo恢复

#### 代码简介

本文使用 MySQL 5.6.16 版本进行分析

mini transation 相关代码路径位于 storage/innobase/mtr/ 主要有 mtr0mtr.cc 和 mtr0log.cc 两个文件

另有部分代码在 storage/innobase/include/ 文件名以 mtr0 开头

mini transaction 的信息保存在结构体 mtr_t 中，结构体成员描述如下

 成员属性
 描述

 state
 mini transaction所处状态 MTR_ACTIVE, MTR_COMMITTING, MTR_COMMITTED

 memo
 mtr 持有锁的栈

 log
 mtr产生的日志

 inside_ibuf
 insert buffer 是否修改

 modifications
 是否修改buffer pool pages

 made_dirty
 是否产生buffer pool脏页

 n_log_recs
 log 记录数

 n_freed_pages
 释放page数

 log_mode
 日志模式，默认MTR_LOG_ALL

 start_lsn
 lsn 起始值

 end_lsn
 lsn 结束值

 magic_n
 魔术字

一个 mini transaction 从 mtr_start(mtr)开始，到 mtr_commit(mtr)结束

## 一条insert语句涉及的 mini transaction

下面涉及 mtr 的嵌套，在代码中，每个 mtr_t 对象变量名都叫 mtr，本文中为了区分不同 mtr，给不同的对象加编号

下面一般省略 mtr_t 以外的参数

第一个 mtr 从 row_ins_clust_index_entry_low 开始

`mtr_start(mtr_1) // mtr_1 贯穿整条insert语句
row_ins_clust_index_entry_low

mtr_s_lock(dict_index_get_lock(index), mtr_1) // 对index加s锁
btr_cur_search_to_nth_level
row_ins_clust_index_entry_low

mtr_memo_push(mtr_1) // buffer RW_NO_LATCH 入栈
buf_page_get_gen
btr_cur_search_to_nth_level
row_ins_clust_index_entry_low

mtr_memo_push(mtr_1) // page RW_X_LATCH 入栈
buf_page_get_gen
btr_block_get_func
btr_cur_latch_leaves
btr_cur_search_to_nth_level
row_ins_clust_index_entry_low

 mtr_start(mtr_2) // mtr_2 用于记录 undo log
 trx_undo_report_row_operation
 btr_cur_ins_lock_and_undo
 btr_cur_optimistic_insert
 row_ins_clust_index_entry_low

 mtr_start(mtr_3) // mtr_3 分配或复用一个 undo log
 trx_undo_assign_undo
 trx_undo_report_row_operation
 btr_cur_ins_lock_and_undo
 btr_cur_optimistic_insert
 row_ins_clust_index_entry_low
 
 mtr_memo_push(mtr_3) // 对复用（也可能是分配）的 undo log page 加 RW_X_LATCH 入栈
 buf_page_get_gen
 trx_undo_page_get
 trx_undo_reuse_cached // 这里先尝试复用，如果复用失败，则分配新的 undo log
 trx_undo_assign_undo
 trx_undo_report_row_operation

 trx_undo_insert_header_reuse(mtr_3) // 写 undo log header
 trx_undo_reuse_cached
 trx_undo_assign_undo
 trx_undo_report_row_operation

 trx_undo_header_add_space_for_xid(mtr_3) // 在 undo header 中预留 XID 空间
 trx_undo_reuse_cached
 trx_undo_assign_undo
 trx_undo_report_row_operation

 mtr_commit(mtr_3) // 提交 mtr_3
 trx_undo_assign_undo
 trx_undo_report_row_operation
 btr_cur_ins_lock_and_undo
 btr_cur_optimistic_insert
 row_ins_clust_index_entry_low
 
 mtr_memo_push(mtr_2) // 即将写入的 undo log page 加 RW_X_LATCH 入栈
 buf_page_get_gen
 trx_undo_report_row_operation
 btr_cur_ins_lock_and_undo
 btr_cur_optimistic_insert
 row_ins_clust_index_entry_low

 trx_undo_page_report_insert(mtr_2) // undo log 记录 insert 操作
 trx_undo_report_row_operation
 btr_cur_ins_lock_and_undo
 btr_cur_optimistic_insert
 row_ins_clust_index_entry_low

 mtr_commit(mtr_2) // 提交 mtr_2
 trx_undo_report_row_operation
 btr_cur_ins_lock_and_undo
 btr_cur_optimistic_insert
 row_ins_clust_index_entry_low
 
/*
 mtr_2 提交后开始执行 insert 操作
 page_cur_insert_rec_low 具体执行 insert 操作
 在该函数末尾调用 page_cur_insert_rec_write_log 写 redo log
*/

page_cur_insert_rec_write_log(mtr_1) // insert 操作写 redo log
page_cur_insert_rec_lowpage_cur_tuple_insert
btr_cur_optimistic_insert

mtr_commit(mtr_1) // 提交 mtr_1
row_ins_clust_index_entry_low 
`

至此 insert 语句执行结束后

一条 insert 是一个单语句事务，事务提交时也会涉及 mini transaction

提交事务时，第一个 mtr 从 trx_prepare 开始

`mtr_start(mtr_4) // mtr_4 用于 prepare transaction
trx_prepare
trx_prepare_for_mysql
innobase_xa_prepare
ha_prepare_low
MYSQL_BIN_LOG::prepare
ha_commit_trans
trans_commit_stmt
mysql_execute_command

mtr_memo_push(mtr_4) // undo page 加 RW_X_LATCH 入栈
buf_page_get_gen
trx_undo_page_get
trx_undo_set_state_at_prepare
trx_prepare

mlog_write_ulint(seg_hdr + TRX_UNDO_STATE, undo->state, MLOG_2BYTES, mtr_4) 写入TRX_UNDO_STATE
trx_undo_set_state_at_prepare
trx_prepare

mlog_write_ulint(undo_header + TRX_UNDO_XID_EXISTS, TRUE, MLOG_1BYTE, mtr_4) 写入 TRX_UNDO_XID_EXISTS
trx_undo_set_state_at_prepare
trx_prepare

trx_undo_write_xid(undo_header, &undo->xid, mtr_4) undo 写入 xid
trx_undo_set_state_at_prepare
trx_prepare

mtr_commit(mtr_4) // 提交 mtr_4
trx_prepare

mtr_start(mtr_5) // mtr_5 用于 commit transaction
trx_commit
trx_commit_for_mysql
innobase_commit_low
innobase_commit
ha_commit_low
MYSQL_BIN_LOG::process_commit_stage_queue
MYSQL_BIN_LOG::ordered_commit
MYSQL_BIN_LOG::commit
ha_commit_trans
trans_commit_stmt
mysql_execute_command

mtr_memo_push(mtr_5) // undo page 加 RW_X_LATCH 入栈
buf_page_get_gen
trx_undo_page_get
trx_undo_set_state_at_finish
trx_write_serialisation_history
trx_commit_low
trx_commit

trx_undo_set_state_at_finish(mtr_5) // set undo state， 这里是 TRX_UNDO_CACHED
trx_write_serialisation_history
trx_commit_low
trx_commit

mtr_memo_push(mtr_5) // 系统表空间 transaction system header page 加 RW_X_LATCH 入栈
buf_page_get_gen
trx_sysf_get
trx_sys_update_mysql_binlog_offset
trx_write_serialisation_history
trx_commit_low
trx_commit

trx_sys_update_mysql_binlog_offset // 更新偏移量信息到系统表空间
trx_write_serialisation_history
trx_commit_low
trx_commit

mtr_commit(mtr_5) // 提交 mtr_5
trx_commit_low
trx_commit
`
至此 insert 语句涉及的 mini transaction 全部结束

## 总结

上面可以看到加锁、写日志到 mlog 等操作在 mini transaction 过程中进行

解锁、把日志刷盘等操作全部在 mtr_commit 中进行，和事务类似

mini transaction 没有回滚操作， 因为只有在 mtr_commit 才将修改落盘，如果宕机，内存丢失，无需回滚；如果落盘过程中宕机，崩溃恢复时可以看出落盘过程不完整，丢弃这部分修改

mtr_commit 主要包含以下步骤

1. mlog 中日志刷盘
2. 释放 mtr 持有的锁，锁信息保存在 memo 中，以栈形式保存，后加的锁先释放
3. 清理 mtr 申请的内存空间，memo 和 log
4. mtr—>state 设置为 MTR_COMMITTED

上面的步骤 1. 中，日志刷盘策略和 innodb_flush_log_at_trx_commit 有关

* 当设置该值为1时，每次事务提交都要做一次fsync，这是最安全的配置，即使宕机也不会丢失事务
* 当设置为2时，则在事务提交时只做write操作，只保证写到系统的page cache，因此实例crash不会丢失事务，但宕机则可能丢失事务
* 当设置为0时，事务提交不会触发redo写操作，而是留给后台线程每秒一次的刷盘操作，因此实例crash将最多丢失1秒钟内的事务

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)