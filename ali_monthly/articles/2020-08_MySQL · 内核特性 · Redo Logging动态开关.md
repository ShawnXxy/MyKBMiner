# MySQL · 内核特性 · Redo Logging动态开关

**Date:** 2020/08
**Source:** http://mysql.taobao.org/monthly/2020/08/03/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2020 / 08
 ](/monthly/2020/08)

 * 当期文章

 MySQL · 引擎特性 · truncate table在大buffer pool下的优化
* MySQL · 引擎特性 · INNODB UNDO LOG分配
* MySQL · 内核特性 · Redo Logging动态开关
* MySQL · 引擎特性 · InnoDB Buffer Page 生命周期
* MySQL · 引擎特性 · InnoDB UNDO LOG写入
* MySQL · 引擎特性 · InnoDB 数据文件简述
* Database · 案例分析 · UTF8与GBK数据库字符集

 ## MySQL · 内核特性 · Redo Logging动态开关 
 Author: zhenpin 

 ## 前言
我们都知道数据库利用write-ahead logging（WAL）的机制，来保证异常宕机后数据的持久性。即提交事务之前，不仅要更新所有事务相关的Page，也要确保所有的WAL日志都写入磁盘。在InnoDB引擎中，这个WAL就是InnoDB的redo log，一般存储在ib_logfilexxx文件中，文件数量可通过my.cnf配置。

在MySQL 8.0官方发布了新版本8.0.21中，支持了一个新特性“Redo Logging动态开关”。借助这个功能，在新实例导数据的场景下，相关事务可以跳过记录redo日志和doublewrite buffer，从而加快数据的导入速度。同时，付出的代价是短时间牺牲了数据库的ACID保障。

## 用法介绍
新增内容

* SQL语法`ALTER INSTANCE {ENABLE | DISABLE} INNODB REDO_LOG`。
* INNODB_REDO_LOG_ENABLE权限，允许执行Redo Logging动态开关的操作。
* Innodb_redo_log_enabled的status，用于显示当前Redo Logging开关状态。

操作步骤

* 创建新的MySQL实例，账号赋权
 `mysql> GRANT INNODB_REDO_LOG_ENABLE ON *.* to 'data_load_admin';
`
* 关闭redo logging
 ```
mysql> ALTER INSTANCE DISABLE INNODB REDO_LOG;

```
* 检查redo logging是否成功关闭
 ```
mysql> SHOW GLOBAL STATUS LIKE 'Innodb_redo_log_enabled';
+-------------------------+-------+
| Variable_name         | Value |
+-------------------------+-------+
| Innodb_redo_log_enabled | OFF |
+-------------------------+-------+

```
* 导数据
* 重新开启redo logging
 ```
mysql> ALTER INSTANCE ENABLE INNODB REDO_LOG;

```
* 确认redo logging状态
 ```
mysql> SHOW GLOBAL STATUS LIKE 'Innodb_redo_log_enabled';
+-------------------------+-------+
| Variable_name         | Value |
+-------------------------+-------+
| Innodb_redo_log_enabled | ON |
+-------------------------+-------+

```

## 注意事项

* 该特性仅用于新实例导数据场景，不可用于线上的生产环境；
* Redo logging关闭状态下，支持正常流程的关闭和重启实例；但在异常宕机情况下，可能会导致丢数据和页面损坏；Redo logging关闭后异常宕机的实例需要废弃重建，直接重启会有如下报错：[ERROR] [MY-013578] [InnoDB] Server was killed when Innodb Redo logging was disabled. Data files could be corrupt. You can try to restart the database with innodb_force_recovery=6.
* Redo logging关闭状态下，不支持cloning operations和redo log archiving这两个功能；
* 执行过程中不支持其他并发的ALTER INSTANCE操作；

## 代码分析

新增handler接口如下

`/**
 @brief
 Enable or Disable SE write ahead logging.

 @param[in] thd server thread handle
 @param[in] enable enable/disable redo logging

 @return true iff failed.
*/
typedef bool (*redo_log_set_state_t)(THD *thd, bool enable);

struct handlerton {
 ...
 redo_log_set_state_t redo_log_set_state;
 ...
}
`
MySQL上层链路是常见的SQL执行链路。

`mysql_parse
 mysql_execute_command
 Sql_cmd_alter_instance::execute
 // case ALTER_INSTANCE_ENABLE_INNODB_REDO
 // 或者 case ALTER_INSTANCE_DISABLE_INNODB_REDO
 Innodb_redo_log::execute
 /*
 Acquire shared backup lock to block concurrent backup. Acquire exclusive
 backup lock to block any concurrent DDL. This would also serialize any
 concurrent key rotation and other redo log enable/disable calls.
 */
 // 通过mdl锁阻止并发
 if (acquire_exclusive_backup_lock(m_thd, m_thd->variables.lock_wait_timeout,
 true) ||
 acquire_shared_backup_lock(m_thd, m_thd->variables.lock_wait_timeout)) {
 DBUG_ASSERT(m_thd->get_stmt_da()->is_error());
 return true;
 }
 hton->redo_log_set_state(m_thd, m_enable)
`
hton->redo_log_set_state在InnoDB引擎对应函数innobase_redo_set_state，最终分别调用mtr_t::s_logging.disable和mtr_t::s_logging.enable。

`static bool innobase_redo_set_state(THD *thd, bool enable) {
 if (srv_read_only_mode) {
 my_error(ER_INNODB_READ_ONLY, MYF(0));
 return (true);
 }

 int err = 0;

 if (enable) {
 err = mtr_t::s_logging.enable(thd); // 开启redo
 } else {
 err = mtr_t::s_logging.disable(thd); // 关闭redo
 }

 if (err != 0) {
 return (true);
 }

 // 设置global status
 set_srv_redo_log(enable);
 return (false);
}
`
在InnoDB引擎层的mtr模块中，新增了一个Logging子模块。该子模块有四种状态，分别的含义如下：

 ENABLED
 Redo log打开。

 ENABLED_DBLWR
 Redo log打开，所有关闭redo状态的mtr对应的page都已经刷盘，doublewrite buffer打开，但是仍有部分page走非doublewrite模式刷盘。

 ENABLED_RESTRICT
 Redo log打开，但是仍有部分关闭redo状态的mtr，且doublewrite buffer未打开。

 DISABLED
 Redo log关闭。

除了ENABLED，其他都是不crash safe的状态。其中，开启redo的状态变化为[DISABLED] -> [ENABLED_RESTRICT] -> [ENABLED_DBLWR] -> [ENABLED]，对应函数mtr::Logging::enable；关闭redo的状态变化为[ENABLED] -> [ENABLED_RESTRICT] -> [DISABLED]，对应函数mtr::Logging::disable。
同时该模块也包含一个Shards类型的m_count_nologging_mtr统计值，记录当前正在运行的关闭redo状态的mtr数量。该统计值使用shared counter类型（Shards），可以减少CPU缓存失效，起到性能优化的作用。

Redo log关闭流程（mtr::Logging::disable）

`int mtr_t::Logging::disable(THD *) {
 // 检查是否已经是DISABLED状态
 if (is_disabled()) {
 return (0);
 }

 /* Disallow archiving to start. */
 ut_ad(m_state.load() == ENABLED);
 m_state.store(ENABLED_RESTRICT);

 /* Check if redo log archiving is active. */
 // 检查是否有redo archive正在进行
 if (meb::redo_log_archive_is_active()) {
 m_state.store(ENABLED);
 my_error(ER_INNODB_REDO_ARCHIVING_ENABLED, MYF(0));
 return (ER_INNODB_REDO_ARCHIVING_ENABLED);
 }

 /* Concurrent clone is blocked by BACKUP MDL lock except when
 clone_ddl_timeout = 0. Force any existing clone to abort. */
 // 停止clone功能
 clone_mark_abort(true);
 ut_ad(!clone_check_active());

 /* Mark that it is unsafe to crash going forward. */
 // 设置redolog的m_disable和m_crash_unsafe标志位
 // 内部调用log_files_header_fill将标志位持久化
 log_persist_disable(*log_sys);

 ib::warn(ER_IB_WRN_REDO_DISABLED);
 m_state.store(DISABLED);

 clone_mark_active();

 /* Reset sync LSN if beyond current system LSN. */
 reset_buf_flush_sync_lsn();

 return (0);
}
`
Redo log打开流程（mtr::Logging::enable）

`int mtr_t::Logging::enable(THD *thd) {
 if (is_enabled()) {
 return (0);
 }
 /* Allow mtrs to generate redo log. Concurrent clone and redo
 log archiving is still restricted till we reach a recoverable state. */
 ut_ad(m_state.load() == DISABLED);
 m_state.store(ENABLED_RESTRICT);

 /* 1. Wait for all no-log mtrs to finish and add dirty pages to disk.*/
 // 等待m_count_nologging_mtr计数器为0或者thd被kill
 auto err = wait_no_log_mtr(thd);
 if (err != 0) {
 m_state.store(DISABLED);
 return (err);
 }

 /* 2. Wait for dirty pages to flush by forcing checkpoint at current LSN.
 All no-logging page modification are done with the LSN when we stopped
 redo logging. We need to have one write mini-transaction after enabling redo
 to progress the system LSN and take a checkpoint. An easy way is to flush
 the max transaction ID which is generally done at TRX_SYS_TRX_ID_WRITE_MARGIN
 interval but safe to do any time. */
 trx_sys_mutex_enter();
 // 通过更新trx_id的接口生成一个mtr，目的是提供一个lsn推进的位点
 trx_sys_flush_max_trx_id();
 trx_sys_mutex_exit();

 /* It would ensure that the modified page in previous mtr and all other
 pages modified before are flushed to disk. Since there could be large
 number of left over pages from LAD operation, we still don't enable
 double-write at this stage. */
 // 不开double-write的状态checkpoint到最新的lsn
 log_make_latest_checkpoint(*log_sys);
 m_state.store(ENABLED_DBLWR);

 /* 3. Take another checkpoint after enabling double write to ensure any page
 being written without double write are already synced to disk. */
 // 再次checkpoint到最新的lsn
 log_make_latest_checkpoint(*log_sys);

 /* 4. Mark that it is safe to recover from crash. */
 // 设回m_disable和m_crash_unsafe标志位，并持久化
 log_persist_enable(*log_sys);

 ib::warn(ER_IB_WRN_REDO_ENABLED);
 m_state.store(ENABLED);

 return (0);
}
`
从以上代码我们可以看到，redo开启的过程中为了优化状态切换的性能，专门增加了ENABLED_DBLWR阶段，并在前后分别执行了一次checkpoint。
然后我们来看下关闭redo logging的行为对其他子模块的影响。Logging系统里面定义了如下几个返回bool类型的函数：

`bool dblwr_disabled() const {
 auto state = m_state.load();
 return (state == DISABLED || state == ENABLED_RESTRICT);
}
bool is_enabled() const { return (m_state.load() == ENABLED); }
bool is_disabled() const { return (m_state.load() == DISABLED); }
`
追溯这些函数的调用方发现：dblwr_disabled用于限制doublewrite buffer的写入。is_enabled用于调整adaptive flush，和阻止cloning operations和redo log archiving这两个功能。is_disabled调用的地方多一些，包含以下几个判断点：

* 调整adaptive flush的速度，加快刷脏；
* page cleaner线程正常退出时在redo header标记当前是crash-safe状态；
* 当innodb_fast_shutdown=2时，自动调整为1确保正常shutdown的时候是crash-safe的；
* 开启新的mtr的时候，调整m_count_nologging_mtr统计值，标记当前mtr为MTR_LOG_NO_REDO状态；

由于adaptive flush依据redo的lsn推进速度才决策刷盘脏页数量，因此adaptive flush的算法需要微调，这一块的逻辑可以参考`Adaptive_flush::page_recommendation`中的`set_flush_target_by_page`。

```
ulint page_recommendation(ulint last_pages_in, bool is_sync_flush) {
 ...
 /* Set page flush target based on LSN. */
 auto n_pages = skip_lsn ? 0 : set_flush_target_by_lsn(is_sync_flush);

 /* Estimate based on only dirty pages. We don't want to flush at lesser rate
 as LSN based estimate may not represent the right picture for modifications
 without redo logging - temp tables, bulk load and global redo off. */
 n_pages = set_flush_target_by_page(n_pages);
 ...
}

```

## 参考资料

* https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-21.html
* https://dev.mysql.com/doc/refman/8.0/en/innodb-redo-log.html#innodb-disable-redo-logging

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)