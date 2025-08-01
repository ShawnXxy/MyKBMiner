# MySQL · 源码分析 · change master to

**Date:** 2018/05
**Source:** http://mysql.taobao.org/monthly/2018/05/09/
**Images:** 2 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 05
 ](/monthly/2018/05)

 * 当期文章

 MySQL · Community · Congratulations on MySQL 8.0 GA
* MySQL · 社区动态 · Online DDL 工具 gh-ost 支持阿里云 RDS
* MySQL · 特性分析 · MySQL 8.0 资源组 (Resource Groups)
* MySQL · 引擎分析 · InnoDB行锁分析
* PgSQL · 特性分析 · 神奇的pg_rewind
* MSSQL · 最佳实践 · 阿里云RDS SQL自动化迁移上云的一种解决方案
* MongoDB · 引擎特性 · journal 与 oplog，究竟谁先写入？
* MySQL · RocksDB · MANIFEST文件介绍
* MySQL · 源码分析 · change master to
* PgSQL · 应用案例 · 阿里云 RDS PostgreSQL 高并发特性 vs 社区版本

 ## MySQL · 源码分析 · change master to 
 Author: xijia 

 #### 重要数据结构

![image.png](.img/14d87228cab6_8085842a53ae6fac102b95a0b2a43477.png)

`
Rpl_info 的基类，保存了一些错误信息，如 IO/SQL thread last error

class Slave_reporting_capability
{

 // 获取last error
 Error const& last_error() const { return m_last_error; }

}

Master_info 、Relay_log_info 的基类，很多重要的锁和信号量都在这里

class Rpl_info : public Slave_reporting_capability
{
 /*
 为了避免死锁，需要按照以下顺序加锁
 run_lock, data_lock, relay_log.LOCK_log, relay_log.LOCK_index
 run_lock, sleep_lock
 run_lock, info_thd_lock
 
 info_thd_lock 保护对 info_thd 的操作
 读操作需要获取 info_thd_lock 或 run_lock
 写操作需要获取 info_thd_lock 和 run_lock
 
 data_lock : 保护对数据的读写
 run_lock : 保护运行状态，变量 slave_running, slave_run_id
 sleep_lock: 对slave_sleep做互斥，防止同时进入多个slave_sleep
 */
 mysql_mutex_t data_lock, run_lock, sleep_lock, info_thd_lock;

 /*
 data_cond: data_lock 保护的数据被修改时发出此信号楼，只有 Relay_log_info 会使用
 start_cond: sql thread start (start slave 先启动 io thread，后启动 sql thread，sql thread 启动代表全部启动成功)
 stop_cond: sql/io thread stop
 sleep_cond: slave被kill，目前只发现5.6中有应用，5.7 中没找到发出此信号量的代码
 */
 mysql_cond_t data_cond, start_cond, stop_cond, sleep_cond;
 
 /* 
 关联 io/sql/worker thread 的 thd
 Master_info 对应 io thread
 Relay_log_info 对应 sql thread
 Slave_worker 对应 worker thread 
 */
 THD *info_thd;

 /* 已初始化 */
 bool inited;
 
 /* 已终止slave */
 volatile bool abort_slave;
 
 /*
 当前slave运行状态，io thread 即 mi->slave_running有三种状态
 MYSQL_SLAVE_NOT_RUN 0
 MYSQL_SLAVE_RUN_NOT_CONNECT 1
 MYSQL_SLAVE_RUN_CONNECT 2 
 
 sql thread 即 mi->slave_running 只有 0 1两种状态
 */
 volatile uint slave_running;
 
 /* 用于判断slave thread 是否启动的变量，每次启动时递增1 */
 volatile ulong slave_run_id;

 /* repository's handler */
 Rpl_info_handler *handler;
 
 /* 
 唯一标识一条 info 记录的 id，标识一行或一个文件
 Master_info 和 Relay_log_info 可以通过 channel 标识唯一
 Worker_info 需要通过 {id, channel} 标识唯一
 */
 uint internal_id;

 /* 通道名，多源复制使用 */
 char channel[CHANNEL_NAME_LENGTH+1];
 
 /* 实现了两个虚函数，读写 Rpl_info */
 virtual bool read_info(Rpl_info_handler *from)= 0;
 virtual bool write_info(Rpl_info_handler *to)= 0;
}

/*
 Master_info 用户 IO thread
 主要保存以下信息：
 连接到Master的用户信息
 当前 master log name
 当前 master log offset
 一些其他控制变量
 
 Master_info 读取 master.info repository 初始化，通常是表或文件
 通过函数 mi_init_info() 进行初始化
 
 调用 flush_info() 可以将 master.info 写入磁盘，每次从master读取数据都需要刷盘
*/
class Master_info : public Rpl_info
{
 /* 前面保存了 user、host 等连接信息 */
 /* 下面是连接信息的 set/get 函数 */

 /* 和 master 的连接 */
 MYSQL* mysql;

 /* 对应的 Relay_log_info */ 
 Relay_log_info *rli;
 
 /* IO 线程复制延迟 */
 long clock_diff_with_master;
 
 /* 心跳间隔 */
 float heartbeat_period; // interface with CHANGE MASTER or master.info
 
 /* 收到心跳次数 */
 ulonglong received_heartbeats; // counter of received heartbeat events

 /* 上次心跳时间 */
 time_t last_heartbeat;

 /* 上次收到event的时间 */
 time_t last_recv_event_timestamp;

 /* 忽略复制的server_id */
 Server_ids *ignore_server_ids;

 /* master server_id */
 ulong master_id;
 
 /* FORMAT_DESCRIPTION_EVENT前的checksum 算法 */
 binary_log::enum_binlog_checksum_alg checksum_alg_before_fd;
 
 /* 初始化 Master_info, 里面调用read_info() 从 Rpl_info_handler 中读取信息 */
 int mi_init_info();
 
 /* 清理 Master_info，里面会调用 Rpl_info_handler 的 eng_info() */
 void end_info();
 
 /* Master_info 信息落盘，每次从主库读取数据后都会执行 */
 int flush_info(bool force= FALSE);
 
 /*
 从master收到的 Format_description_log_event 写在 relay log 末尾
 
 IO thread 开始时创建，IO thread 结束时销毁
 IO thread 收到一个 Format_description_log_event 时更新
 每次rotate时，IO thread 写Format_description_log_event到新relay log
 每次执行FLUSH LOGS，client 写Format_description_log_event到新relay log
 */
 Format_description_log_event *mi_description_event;

 /* 最近一个GTID，可能是未完成事务的GTID，用于事务结束时写入 Retrieved_Gtid_Set */
 Gtid last_gtid_queued;
 
 /* 用于判断事务边界 */
 Transaction_boundary_parser transaction_parser;
 
 /* 
 channel lock，以下操作需要持有写锁
 START SLAVE;
 STOP SLAVE;
 CHANGE MASTER;
 RESET SLAVE;
 end_slave();
 */
 Checkable_rwlock *m_channel_lock;
 
 /* channel 被引用的次数，只有为0时可以删除channel */
 Atomic_int32 references;
}

`

```
/*
 主要保存以下信息:
 当前relay log
 当前relay log offset
 master log name
 与上次更新对应的主库日志序列
 sql thread 其他信息

 初始化过程和 Master_info 类似
 
 以下情况下 relay.info table/file 需要更新：
 1. relay log file rotated
 2. SQL thread stopped
 3. while processing a Xid_log_event
 4. after a Query_log_event(commit or rollback)
 5. after processing any statement written to the binary log without a transaction context.
 
 并行复制相关代码留作以后分析，本次暂不涉及
*/
class Relay_log_info : public Rpl_info
{
 /* 备份状态标志位，用于标志是否在语句中 */
 enum enum_state_flag {
 /* 在语句中 */
 IN_STMT,

 /** Flag counter. Should always be last */
 STATE_FLAGS_COUNT
 };
 
 /* 是否复制相同server_id的event，一般是false */
 bool replicate_same_server_id;
 
 /* 正在执行或最后一个执行的GTID，用来填充 performance_schema.replication_applier_status_by_worke 的 last_seen_transaction 列
 */
 Gtid_specification currently_executing_gtid;

 /* 读取下面变量时，必须受data_lock保护 */
 
 /* 当前relay log的文件描述符 */
 File cur_log_fd;
 
 /* reay_log 对象，MYSQL_BIN_LOG类留作以后分析*/
 MYSQL_BIN_LOG relay_log;
 
 /* 主要用于查询log_pos */
 LOG_INFO linfo;
 
 /*
 cur_log 指向 relay_log.get_log_file() 或者 cache_buf
 取决于relay_log_file是热日志，还是需要打开冷日志
 
 cache_buf 在打开冷日志时适用
 */
 IO_CACHE cache_buf,*cur_log;
 
 /* 标识是否正在recovery */
 bool is_relay_log_recovery;
 
 /* 下面的变量可以不加锁读 */
 
 /*
 restart slave 时需要访问临时表。
 这个变量值在 init/end 时修改
 SQL thread 只读
 */
 TABLE *save_temporary_tables;
 
 /* 对应的 Master_info */
 Master_info *mi;
 
 /* 打开临时表的数量 */
 Atomic_int32 channel_open_temp_tables;
 
 /* 保存 relay_log.get_open_count() */
 uint32 cur_log_old_open_count;
 
 /* init_info() 曾经失败，RESET SLAVE 可以修复错误 */
 bool error_on_rli_init_info;
 
 /* 这里跳过一些 Group replication 相关变量 */
 
 /* received gtid set */
 Gtid_set gtid_set;
 
 /* 标识此对象是否属于SQL线程（属于SQL线程为0） */
 bool rli_fake;
 
 /* 标识 retrieved GTID set 是否已被初始化 */
 bool gtid_retrieved_initialized;
 
 /* 上一个错误的GTID */
 Gtid last_sql_error_gtid;

 /* 日志空间限制，日志空间总量(用于sys_var:relay_log_space，控制relay log空间) */
 ulonglong log_space_limit,log_space_total;
 
 /* 是否忽略日志空间限制 */
 bool ignore_log_space_limit;

 /* 需要清理空间时，SQL线程指示IO线程rotate logs */
 bool sql_force_rotate_relay;
 
 /* 上次主库记录binlog的时间 */
 time_t last_master_timestamp;
 
 /* 上次执行event的时间 */
 time_t last_exec_event_timestamp;
 
 /* 跳过error event */
 volatile uint32 slave_skip_counter;
 
 /* 标记是否需要中断pos_wait，change master 和 reset slave时需要中断 */
 volatile ulong abort_pos_wait;
 
 /* log_space 相关信号量*/
 mysql_mutex_t log_space_lock;
 mysql_cond_t log_space_cond;

 /* 这里有一些 START SLAVE UNTIL 相关变量 */
 
 /* 重试事务次数(trans_retries是重试次数上限)，重试事务计数（retried_trans记录重试了多少次） */
 ulong trans_retries, retried_trans;
 
 /* 
 延迟复制时间
 CHANGE MASTER TO MASTER_DELAY=X.
 由data_lock保护， SQL thread 读取
 SQL thread 运行时该变量不可写
 */
 time_t sql_delay;
 
 /* sql_delay 结束时间 */
 time_t sql_delay_end;

 /* enum_state_flag 的标志位 */
 uint32 m_flags; 
}

```

#### 函数分析

mysql_execute_command() 当做入口开始分析，可以看出change_master需要SUPER权限

`
 case SQLCOM_CHANGE_MASTER:
 {
 if (check_global_access(thd, SUPER_ACL))
 goto error;
 res= change_master_cmd(thd);
 break;
 }
 
/*
 函数具备以下功能
 更改接收/执行日志的配置/位点
 purge relay log
 删除 worker info（并行复制使用）
*/

/* 分析 change_master 函数会略过一部分逻辑 */
int change_master(THD* thd, Master_info* mi, LEX_MASTER_INFO* lex_mi,
 bool preserve_logs)
{
 /*
 如果SQL thread 和 IO thread已经停止，并且没有指定 relay_log_pos和relay_log_file
 会purge relay log
 */
 bool need_relay_log_purge= 1;

 /* 为了修改 mysql.slave_master_info，需要无视read_only和super_read_only */
 thd->set_skip_readonly_check();
 
 /* channel 加读写锁，即将对channel做修改，函数结束时才会释放锁 */
 mi->channel_wrlock();
 
 /*
 对 mi->run_lock 和 rli->run_lock 加锁
 防止线程运行状态发生变化
 */
 lock_slave_threads(mi);

 /* 设置thread_mask，用来标识 IO/SQL thread 的运行状态 */
 init_thread_mask(&thread_mask, mi, 0);

 /* 设置auto_position=1需要IO/SQL thread 都不在运行状态，否则报错退出 */
 if (thread_mask)
 {
 if (lex_mi->auto_position != LEX_MASTER_INFO::LEX_MI_UNCHANGED)
 {
 error= ER_SLAVE_CHANNEL_MUST_STOP;
 my_error(ER_SLAVE_CHANNEL_MUST_STOP, MYF(0), mi->get_channel());
 goto err;
 }
 
 /* 如果 SQL thread 和 IO thread 没有全部停止，不能purge relay log */
 need_relay_log_purge= 0;
 }

 /*
 下面是一些错误判断，都是很明显的错误
 1. 如果设置了auto_position，同时又指定了复制位点，如 relay_log_pos，报错退出
 2. auto_position 需要 GTID_MODE != OFF
 3. IO thread 运行时不能改变 IO thread 相关配置
 4. SQL thread 运行时不能改变 SQL thread 相关配置
 5. 如果指定了master_host，那么master_host不能是空串 
 */

 /* 记录当前状态 */
 THD_STAGE_INFO(thd, stage_changing_master); 

 /* 标识停止的线程，给load_mi_and_rli_from_repositories()使用 */
 init_thread_mask(&thread_mask_stopped_threads, mi, 1);

 /* 
 从仓库加载 mi 和 rli 的配置
 只有停止状态的线程可以加载配置（SQL thread 对应 rli，IO thread 对应 mi）
 */
 if (load_mi_and_rli_from_repositories(mi, false, thread_mask_stopped_threads)) 
 {
 error= ER_MASTER_INFO;
 my_message(ER_MASTER_INFO, ER(ER_MASTER_INFO), MYF(0));
 goto err;
 }

 /* 
 修改mi相关配置，并保存老配置
 save_ 变量中保存的老配置用于打印日志
 */
 if (have_receive_option)
 {
 strmake(saved_host, mi->host, HOSTNAME_LENGTH);
 strmake(saved_bind_addr, mi->bind_addr, HOSTNAME_LENGTH);
 saved_port= mi->port;
 strmake(saved_log_name, mi->get_master_log_name(), FN_REFLEN - 1);
 saved_log_pos= mi->get_master_log_pos();

 if ((error= change_receive_options(thd, lex_mi, mi))) 
 {
 goto err;
 }
 }

 /* 打印日志，change master 的源值和目标值 */
 if (have_receive_option)
 sql_print_information("'CHANGE MASTER TO%s executed'. "
 "Previous state master_host='%s', master_port= %u, master_log_file='%s', "
 "master_log_pos= %ld, master_bind='%s'. "
 "New state master_host='%s', master_port= %u, master_log_file='%s', "
 "master_log_pos= %ld, master_bind='%s'.",
 mi->get_for_channel_str(true),
 saved_host, saved_port, saved_log_name, (ulong) saved_log_pos,
 saved_bind_addr, mi->host, mi->port, mi->get_master_log_name(),
 (ulong) mi->get_master_log_pos(), mi->bind_addr);

 /* 修改rli相关配置 */
 if (have_execute_option)
 change_execute_options(lex_mi, mi);
 
 /* 持久化master_info */
 if ((thread_mask & SLAVE_IO) == 0 && flush_master_info(mi, true))
 {
 error= ER_RELAY_LOG_INIT;
 my_error(ER_RELAY_LOG_INIT, MYF(0), "Failed to flush master info file");
 goto err;
 }
 
 if ((thread_mask & SLAVE_SQL) == 0)
 {

 /* 记录全局变量 relay_log_purge */ 
 bool save_relay_log_purge= relay_log_purge;

 if (need_relay_log_purge)
 {
 const char* errmsg= 0;

 /* purge relay logs */
 relay_log_purge= 1;
 THD_STAGE_INFO(thd, stage_purging_old_relay_logs);
 if (mi->rli->purge_relay_logs(thd,
 0 /* not only reset, but also reinit */,
 &errmsg))
 {
 error= ER_RELAY_LOG_FAIL;
 my_error(ER_RELAY_LOG_FAIL, MYF(0), errmsg);
 goto err;
 }
 }
 else
 {

 const char* msg;
 relay_log_purge= 0;

 DBUG_ASSERT(mi->rli->inited);
 
 /*初始化 relay_log_pos */
 if (mi->rli->init_relay_log_pos(mi->rli->get_group_relay_log_name(),
 mi->rli->get_group_relay_log_pos(),
 true/*we do need mi->rli->data_lock*/,
 &msg, 0))
 {
 error= ER_RELAY_LOG_INIT;
 my_error(ER_RELAY_LOG_INIT, MYF(0), msg);
 goto err;
 }
 }
 
 /* 恢复全局变量 relay_log_purge 的值 */
 relay_log_purge= save_relay_log_purge;

 /* 清理until condition */
 mi->rli->clear_until_condition();

 /* relay_log_info 持久化到磁盘 */
 if (mi->rli->flush_info(true))
 {
 error= ER_RELAY_LOG_INIT;
 my_error(ER_RELAY_LOG_INIT, MYF(0), "Failed to flush relay info file.");
 goto err;
 }

 }

/* 出错后跳到次数，释放之前申请的锁 */
err:

 unlock_slave_threads(mi);
 mi->channel_unlock();
 DBUG_RETURN(error);

}
`

#### 总结
change master 主要功能是修改 SQL 和 IO 线程的配置信息，执行时可能会purge relay log

没有特殊情况，建议指定auto_position=1，不要自己指定复制位点，避免数据丢失风险

如需对change master 做修改，需要注意在锁保护下修改变量，同时注意加锁顺序，避免死锁

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)