# MySQL · 源码分析 · binlog crash recovery

**Date:** 2018/07
**Source:** http://mysql.taobao.org/monthly/2018/07/05/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 07
 ](/monthly/2018/07)

 * 当期文章

 MySQL · 引擎特性 · WAL那些事儿
* MySQL · 源码分析 · 8.0 原子DDL的实现过程续
* MongoDB · 引擎特性 · 事务实现解析
* MySQL · RocksDB · 写入逻辑的实现
* MySQL · 源码分析 · binlog crash recovery
* PgSQL · 新特征 · PG11并行Hash Join介绍
* MySQL · myrocks · clustered index特性
* MSSQL · 最佳实践 · 实例级别数据库上云RDS SQL Server
* MySQL · 最佳实践 · 一个TPC-C测试工具sqlbench使用
* PgSQL · 应用案例 · PostgreSQL flashback(闪回) 功能实现与介绍

 ## MySQL · 源码分析 · binlog crash recovery 
 Author: 西加 

 ### 前言
本文主要介绍binlog crash recovery 的过程

假设用户使用 InnoDB 引擎，sync_binlog=1

使用 MySQL 5.7.20 版本进行分析

crash recovery 过程中，binlog 需要保证：

1. 所有已提交事务的binlog已存在
2. 所有未提交事务的binlog不存在

### 两阶段提交
MySQL 使用两阶段提交解决 binlog 和 InnoDB redo log 的一致性的问题

也就是将普通事务当做内部XA事务处理，为每个事务分配一个XID，binlog作为事务的协调者

* 阶段1：InnoDB redo log 写盘，InnoDB 事务进入 prepare 状态
* 阶段2：binlog 写盘，InooDB 事务进入 commit 状态

每个事务binlog的末尾，会记录一个 XID event，标志着事务是否提交成功，也就是说，recovery 过程中，binlog 最后一个 XID event 之后的内容都应该被 purge。

InnoDB 日志可能也需要回滚或者提交，这里就不再展开。

### binlog 文件的 crash recovery
`mysqld_main

 init_server_components
 
 MYSQL_BIN_LOG::open

 MYSQL_BIN_LOG::open_binlog
`
binlog recover 的主要过程在 MYSQL_BIN_LOG::open_binlog 中

`int MYSQL_BIN_LOG::open_binlog(const char *opt_name)
{
 
 /* 确保 index 文件初始化成功 */
 if (!my_b_inited(&index_file)) 
 {
 /* There was a failure to open the index file, can't open the binlog */
 cleanup();
 return 1;
 }
 
 /* 找到 index 中第一个 binlog */
 if ((error= find_log_pos(&log_info, NullS, true/*need_lock_index=true*/)))
 
 {
 /* 找到 index 中最后一个 binlog */
 do
 {
 strmake(log_name, log_info.log_file_name, sizeof(log_name)-1); 
 } while (!(error= find_next_log(&log_info, true/*need_lock_index=true*/)));

 /*
 打开最后一个binlog，会校验文件头的 magic number "\xfe\x62\x69\x6e"
 如果 magic number 校验失败，会直接报错退出，无法完成recovery
 如果确定最后一个binlog没有内容，可以删除binlog 文件再重试
 */
 if ((file= open_binlog_file(&log, log_name, &errmsg)) < 0)
 
 /*
 如果 binlog 没有正常关闭，mysql server 可能crash过，
 我们需要调用 MYSQL_BIN_LOG::recover：
 
 a) 找到最后一个 XID
 b) 完成最后一个事务的两阶段提交（InnoDB commit）
 c) 找到最后一个合法位点
 
 因此，我们需要遍历 binlog 文件，找到最后一个合法event集合，并 purge 无效binlog
 */
 if ((ev= Log_event::read_log_event(&log, 0, &fdle,
 opt_master_verify_checksum)) &&
 ev->get_type_code() == binary_log::FORMAT_DESCRIPTION_EVENT &&
 (ev->common_header->flags & LOG_EVENT_BINLOG_IN_USE_F ||
 DBUG_EVALUATE_IF("eval_force_bin_log_recovery", true, false)))
 {
 sql_print_information("Recovering after a crash using %s", opt_name); 
 
 /* 初始化合法位点 */ 
 valid_pos= my_b_tell(&log);
 
 /* 执行recover 过程 ，并计算出合法位点 */
 error= recover(&log, (Format_description_log_event *)ev, &valid_pos);
 }
 else
 error=0;
 
 if (valid_pos > 0){
 if (valid_pos < binlog_size)
 { 
 /* 将 valid_pos 后面的binlog purge掉 */
 if (my_chsize(file, valid_pos, 0, MYF(MY_WME)))
 }
 }
 } 
}

`

recover 函数的逻辑很简单：遍历最后一个binlog的所有 event，每次事务结尾，或者非事务event结尾更新 valid_pos(gtid event不更新)。并在一个 hash 中记录所有xid，用于引擎层 recover

`int MYSQL_BIN_LOG::recover(IO_CACHE *log, Format_description_log_event *fdle,
 my_off_t *valid_pos)
{

 /* 初始化 XID hash，用于记录 binlog 中的 xid */
 if (! fdle->is_valid() || 
 my_hash_init(&xids, &my_charset_bin, TC_LOG_PAGE_SIZE/3, 0,
 sizeof(my_xid), 0, 0, MYF(0),
 key_memory_binlog_recover_exec))
 goto err1;
 
 /* 依次读取 binlog event */
 while ((ev= Log_event::read_log_event(log, 0, fdle, TRUE))
 && ev->is_valid())
 {
 if (ev->get_type_code() == binary_log::QUERY_EVENT &&
 !strcmp(((Query_log_event*)ev)->query, "BEGIN"))
 /* begin 代表事务开始 */
 in_transaction= TRUE;

 if (ev->get_type_code() == binary_log::QUERY_EVENT &&
 !strcmp(((Query_log_event*)ev)->query, "COMMIT"))
 {
 DBUG_ASSERT(in_transaction == TRUE);
 /* commit 代表事务结束 */
 in_transaction= FALSE;
 }
 else if (ev->get_type_code() == binary_log::XID_EVENT)
 {
 DBUG_ASSERT(in_transaction == TRUE);
 /* xid event 代表事务结束 */
 in_transaction= FALSE;
 Xid_log_event *xev=(Xid_log_event *)ev;
 uchar *x= (uchar *) memdup_root(&mem_root, (uchar*) &xev->xid,
 sizeof(xev->xid));
 /* 记录 xid */
 if (!x || my_hash_insert(&xids, x))
 goto err2;
 }

 /*
 如果不在事务中，且不是gtid event，则更新 valid_pos
 显然，如果在事务中，最后一段 event 不是一个完整事务，pos并不合法
 */
 if (!log->error && !in_transaction &&
 !is_gtid_event(ev))
 *valid_pos= my_b_tell(log);
 }

 /*
 存储引擎recover
 所有已经记录 XID 的事务必须在存储引擎中提交
 未记录 XID 的事务必须回滚
 */
 if (total_ha_2pc > 1 && ha_recover(&xids))
 goto err2;

`

### binlog index 的 crash recovery

为了保证 binlog index 的 crash safe，MySQL 引入了一个临时文件 crash_safe_index_file

新的 binlog_file_name 写入 binlog_index_file 流程如下：

* 创建临时文件 crash_safe_index_file
* 拷贝 binlog_index_file 中的内容到 crash_safe_index_file
* 新的 binlog_file_name 写入 crash_safe_index_file
* 删除 binlog_index_file
* 重命名 crash_safe_index_file 到 binlog_index_file

这个流程保证了在任何时候crash，binlog_index_file 和 crash_safe_index_file 至少有一个可用

这样再recover 时只要判断这两个文件是否可用，如果 binlog_index_file 可用则无需特殊处理，如果binlog_index_file 不可用则重命名 crash_safe_index_file 到 binlog_index_file

binlog index 的 recover 过程主要在 bool MYSQL_BIN_LOG::open_index_file 中

显然，open_indix_file 在 open_binlog 之前

`mysqld_main

 init_server_components

 MYSQL_BIN_LOG::open_index_file

`

```

bool MYSQL_BIN_LOG::open_index_file(const char *index_file_name_arg,
 const char *log_name, bool need_lock_index)
{
 /* 拼接 index_file_name */
 fn_format(index_file_name, index_file_name_arg, mysql_data_home,
 ".index", opt); 

 /* 拼接 crash_safe_index_file_name */
 if (set_crash_safe_index_file_name(index_file_name_arg))

 /*
 recover 主要体现在这里
 检查 index_file_name 和 crash_safe_index_file_name 是否存在
 如果 index_file_name 不存在 crash_safe_index_file_name 存在，
 那么将 crash_safe_index_file_name 重命名为 index_file_name
 */
 if (my_access(index_file_name, F_OK) &&
 !my_access(crash_safe_index_file_name, F_OK) &&
 my_rename(crash_safe_index_file_name, index_file_name, MYF(MY_WME)))
 {
 sql_print_error("MYSQL_BIN_LOG::open_index_file failed to "
 "move crash_safe_index_file to index file.");
 error= true;
 goto end;
 }

}

```

新的 binlog_file_name 写入 binlog_index_file 的过程在 MYSQL_BIN_LOG::add_log_to_index

`int MYSQL_BIN_LOG::add_log_to_index(uchar* log_name,
 size_t log_name_len, bool need_lock_index)
{
 /* 创建 crash_safe_index_file */
 if (open_crash_safe_index_file())

 /* 拷贝 index_file 内容到 crash_safe_index_file */
 if (copy_file(&index_file, &crash_safe_index_file, 0))
 
 /* 写入 binlog_file_name */
 if (my_b_write(&crash_safe_index_file, log_name, log_name_len) ||
 my_b_write(&crash_safe_index_file, (uchar*) "\n", 1) ||
 flush_io_cache(&crash_safe_index_file) ||
 mysql_file_sync(crash_safe_index_file.file, MYF(MY_WME)))

 /*
 函数内部先 delete binlog_index_file 再 rename crash_safe_index_file
 如果 delete 到 rename 之间发生 crash， crash_safe_index_file 会在 recover过程中 rename 成 binlog_index_file
 */
 if (move_crash_safe_index_file_to_index_file(need_lock_index))
 
}

`
### 总结
MySQL 解决了binlog crash safe 的问题，但是 relay log 依然不保证 crash safe。

relay log 结构和 binlog 一致，可以借鉴 binlog crash safe 的方式，计算出 valid_pos，将 valid_pos之后的 event 全部purge。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)