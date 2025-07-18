# MySQL · 源码分析 · SHUTDOWN过程

**Date:** 2017/08
**Source:** http://mysql.taobao.org/monthly/2017/08/09/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 08
 ](/monthly/2017/08)

 * 当期文章

 MySQL · 引擎特性 · Group Replication内核解析
* PgSQL · 特性介绍 · 列存元数据扫描介绍
* MySQL · 源码分析 · MySQL replication partial transaction
* MySQL · 特性分析 · 到底是谁执行了FTWL
* MySQL · 源码分析 · mysql认证阶段漫游
* MySQL · 源码分析 · 内存分配机制
* PgSQL · 源码分析 · PG 优化器中的pathkey与索引在排序时的使用
* MSSQL· 实现分析 · Extend Event日志文件的分析方法
* MySQL · 源码分析 · SHUTDOWN过程
* PgSQL · 应用案例 · HDB for PG特性(数据排盘与任意列高效率过滤)

 ## MySQL · 源码分析 · SHUTDOWN过程 
 Author: 西加 

 ### ORACLE 中的SHUTDOWN

MySQL SHUTDOWN LEVEL 暂时只有一种，源码中留了 LEVEL 的坑还没填

在此借用 Oracle 的 SHUTDOWN LEVEL 分析

Oracle SHUTDOWN LEVEL 共有四种：ABORT、IMMEDIATE、NORMAL、TRANSACTIONAL

##### ABORT
* 立即结束所有SQL
* 回滚未提交事务
* 断开所有用户连接
* 下次启动实例时，需要recovery

##### IMMEDIATE
* 允许正在运行的SQL执行完毕
* 回滚未提交事务
* 断开所有用户连接

##### NORMAL
* 不允许建立新连接
* 等待当前连接断开
* 下次启动实例时，不需要recovery

##### TRANSACTIONAL
* 等待事务提交或结束
* 不允许新建连接
* 事务提交或结束后断开连接

MySQL 中的 SHUTDOWN 实际相当于 Oracle 中的 SHUTDOWN IMMEDIATE，重启实例时无需recovery，但回滚事务的过程可能耗时很长

### MySQL SHUTDOWN过程分析
* mysql_shutdown 发送SHUTDOWN命令
* dispatch_command() 接受到 COM_SHUTDOWN command，调用kill_mysql()
* kill_mysql()创建 kill_server_thread
* kill_server_thread 调用 kill_server()
* kill_server()
 
 close_connections()
 
 关闭端口
* 断开连接
* 回滚事务（可能耗时很长）

 unireg_end
 * clean_up
 
 innobase_shutdown_for_mysql
* delete_pid_file

InnoDB shutdown 速度取决于参数 innodb_fast_shutdown

* 0: 最慢，需等待purge完成，change buffer merge完成
* 1: default， 不需要等待purge完成和change buffer merge完成
* 2: 不等待后台删除表完成，row_drop_tables_for_mysql_in_background 不等刷脏页，如果设置了innodb_buffer_pool_dump_at_shutdown，不需要去buffer dump.

` case COM_SHUTDOWN: // 接受到SHUTDOWN命令
 {
 if (packet_length < 1)
 { 
 my_error(ER_MALFORMED_PACKET, MYF(0));
 break;
 } 
 status_var_increment(thd->status_var.com_other);
 if (check_global_access(thd,SHUTDOWN_ACL)) // 检查权限
 break; /* purecov: inspected */
 /* 
 If the client is < 4.1.3, it is going to send us no argument; then
 packet_length is 0, packet[0] is the end 0 of the packet. Note that
 SHUTDOWN_DEFAULT is 0. If client is >= 4.1.3, the shutdown level is in
 packet[0].
 */
 enum mysql_enum_shutdown_level level; // 留的坑，default以外的LEVEL都没实现
 if (!thd->is_valid_time())
 level= SHUTDOWN_DEFAULT; 
 else 
 level= (enum mysql_enum_shutdown_level) (uchar) packet[0];
 if (level == SHUTDOWN_DEFAULT)
 level= SHUTDOWN_WAIT_ALL_BUFFERS; // soon default will be configurable
 else if (level != SHUTDOWN_WAIT_ALL_BUFFERS)
 { 
 my_error(ER_NOT_SUPPORTED_YET, MYF(0), "this shutdown level");
 break;
 } 
 DBUG_PRINT("quit",("Got shutdown command for level %u", level));
 general_log_print(thd, command, NullS); // 记录general_log
 my_eof(thd);
 kill_mysql(); // 调用kill_mysql()函数，函数内部创建 kill_server_thread 线程
 error=TRUE;
 break;
 }
 
`

kill_server() 先调用 close_connections()，再调用 unireg_end()

`static void __cdecl kill_server(int sig_ptr)
{
 ......
 close_connections();
 if (sig != MYSQL_KILL_SIGNAL &&
 sig != 0) 
 unireg_abort(1); /* purecov: inspected */
 else
 unireg_end();

`

结束线程的主要逻辑在 mysqld.cc:close_connections() 中

` static void close_connections(void)

 ......
 
 /* 下面这段代码结束监听端口 */
 /* Abort listening to new connections */
 DBUG_PRINT("quit",("Closing sockets"));
 if (!opt_disable_networking )
 {
 if (mysql_socket_getfd(base_ip_sock) != INVALID_SOCKET)
 {
 (void) mysql_socket_shutdown(base_ip_sock, SHUT_RDWR);
 (void) mysql_socket_close(base_ip_sock);
 base_ip_sock= MYSQL_INVALID_SOCKET;
 }
 if (mysql_socket_getfd(extra_ip_sock) != INVALID_SOCKET)
 {
 (void) mysql_socket_shutdown(extra_ip_sock, SHUT_RDWR);
 (void) mysql_socket_close(extra_ip_sock);
 extra_ip_sock= MYSQL_INVALID_SOCKET;
 }
 }
 
 ......

 /* 第一遍遍历线程列表 */
 sql_print_information("Giving %d client threads a chance to die gracefully",
 static_cast<int>(get_thread_count()));

 mysql_mutex_lock(&LOCK_thread_count);
 
 Thread_iterator it= global_thread_list->begin();
 for (; it != global_thread_list->end(); ++it)
 {
 THD *tmp= *it;
 DBUG_PRINT("quit",("Informing thread %ld that it's time to die",
 tmp->thread_id));
 /* We skip slave threads & scheduler on this first loop through. */
 
 /* 跳过 slave 相关线程，到 end_server() 函数内处理 */
 if (tmp->slave_thread) 
 continue;
 if (tmp->get_command() == COM_BINLOG_DUMP ||
 tmp->get_command() == COM_BINLOG_DUMP_GTID)
 {
 ++dump_thread_count;
 continue;
 }
 
 /* 先标记为 KILL 给连接一个自我了断的机会 */
 tmp->killed= THD::KILL_CONNECTION;
 
 ......
 
 }
 mysql_mutex_unlock(&LOCK_thread_count);

 Events::deinit();

 sql_print_information("Shutting down slave threads");
 /* 此处断开 slave 相关线程 */
 end_slave();
 
 /* 第二遍遍历线程列表 */
 if (dump_thread_count)
 { 
 /*
 Replication dump thread should be terminated after the clients are
 terminated. Wait for few more seconds for other sessions to end.
 */
 while (get_thread_count() > dump_thread_count && dump_thread_kill_retries)
 {
 sleep(1);
 dump_thread_kill_retries--;
 }
 mysql_mutex_lock(&LOCK_thread_count);
 for (it= global_thread_list->begin(); it != global_thread_list->end(); ++it)
 {
 THD *tmp= *it;
 DBUG_PRINT("quit",("Informing dump thread %ld that it's time to die",
 tmp->thread_id));
 if (tmp->get_command() == COM_BINLOG_DUMP ||
 tmp->get_command() == COM_BINLOG_DUMP_GTID)
 {
 /* 关闭DUMP线程 */
 tmp->killed= THD::KILL_CONNECTION;
 
 ......
 
 }
 }
 mysql_mutex_unlock(&LOCK_thread_count);
 }
 
 ......
 
 /* 第三遍遍历线程列表 */
 for (it= global_thread_list->begin(); it != global_thread_list->end(); ++it)
 {
 THD *tmp= *it;
 if (tmp->vio_ok())
 {
 if (log_warnings)
 sql_print_warning(ER_DEFAULT(ER_FORCING_CLOSE),my_progname,
 tmp->thread_id,
 (tmp->main_security_ctx.user ?
 tmp->main_security_ctx.user : ""));
 /* 关闭连接，不等待语句结束，但是要回滚未提交线程 */
 close_connection(tmp);
 }
 }
 
`

close_connection() 中调用 THD::disconnect() 断开连接
连接断开后开始回滚事务

`bool do_command(THD *thd)
{
 ......
 packet_length= my_net_read(net); // thd->disconnect() 后此处直接返回
 ...... 
}

void do_handle_one_connection(THD *thd_arg)
{
 ......
 while (thd_is_connection_alive(thd))
 {
 if (do_command(thd)) //do_command 返回 error，跳出循环
 break;
 }
 end_connection(thd);
 
end_thread:
 close_connection(thd);
 /* 此处调用one_thread_per_connection_end() */
 if (MYSQL_CALLBACK_ELSE(thd->scheduler, end_thread, (thd, 1), 0))
 return; // Probably no-threads

 ......
}

`
事务回滚调用链

`trans_rollback(THD*) ()
THD::cleanup() ()
THD::release_resources() ()
one_thread_per_connection_end(THD*, bool) ()
do_handle_one_connection(THD*) ()
handle_one_connection ()
`

unireg_end 调用 clean_up()

`void clean_up(bool print_message)
{
 /* 这里是一些释放内存和锁的操作 */ 
 ......
 
 /*
 这里调用 innobase_shutdown_for_mysql
 purge all (innodb_fast_shutdown = 0)
 merge change buffer (innodb_fast_shutdown = 0）
 flush dirty page (innodb_fast_shutdown = 0,1)
 flush log buffer
 都在这里面做 
 */
 plugin_shutdown();
 
 /* 这里是一些释放内存和锁的操作 */
 ......
 
 /* 
 删除 pid 文件，删除后 mysqld_safe不会重启 mysqld，
 不然会认为 mysqld crash，尝试重启
 */
 delete_pid_file(MYF(0));
 
 /* 这里是一些释放内存和锁的操作 */
 ......
 
`

#### innodb shutdown 分析
innodb shutdown 的主要操作在 logs_empty_and_mark_files_at_shutdown() 中

* 等待后台线程结束
 
 srv_error_monitor_thread
* srv_lock_timeout_thread
* srv_monitor_thread
* buf_dump_thread
* dict_stats_thread

 等待所有事物结束 trx_sys_any_active_transactions
 等待后台线程结束
 * worker threads: srv_worker_thread
* master thread: srv_master_thread
* purge thread: srv_purge_coordinator_thread

 等待 buf_flush_lru_manager_thread 结束
 等待 buf_flush_page_cleaner_thread 结束
 等待 Pending checkpoint_writes, Pending log flush writes 结束
 等待 buffer pool pending io 结束
 if (innodb_fast_shutdown == 2)
 * flush log buffer 后 return

 log_make_checkpoint_at
 * flush buffer pool
* write checkpoint

 将 lsn 落盘 fil_write_flushed_lsn_to_data_files()
 关闭所有文件

logs_empty_and_mark_files_at_shutdown() 结束后，innobase_shutdown_for_mysql() 再做一些资源清理工作即结束 shutdown 过程

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)