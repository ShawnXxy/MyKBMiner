# MySQL · 源码阅读· MySQL 如何响应 KILL

**Date:** 2021/11
**Source:** http://mysql.taobao.org/monthly/2021/11/03/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2021 / 11
 ](/monthly/2021/11)

 * 当期文章

 PolarDB · 引擎特性· 闪回查询让历史随时可见
* MySQL · 周边工具 · MySQL InnoDB inno_space 工具介绍
* MySQL · 源码阅读· MySQL 如何响应 KILL

 ## MySQL · 源码阅读· MySQL 如何响应 KILL 
 Author: 臻成 

 ## MySQL 如何响应 kill
我们使用 MySQL 时经常会遇到需要中止一条查询的时候, 本文讨论 kill 命令的效果及其实现。

## 三种不同的 kill 命令

从 MySQL 官网上我们可以看到 kill 命令格式如下:

`KILL [CONNECTION | QUERY] processlist_id
`

首先说明要做 kill, 然后指定 kill 的级别:

* CONNECTION, kill 之后 processlist_id 对应的查询中止, 对应的连接退出, 如果没有查询, 那么连接直接退出
* QUERY, kill 之后 processlist_id 对应的查询中止, 连接保持

下面我们针对这两种情况进行一些简单的测试, 看看实际效果.

### kill query

我们先制造一个简单的可以 kill 的情况, 为了避免需要太多数据, 我们选择制造一个被锁住的情况, 然后 kill 被锁住的连接. 首先连接数据库, 使用当前连接 (之后称为 C1) 创建一张简单的表:

` -- IN C1
 CREATE TABLE `t1` (
 `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
 `k` int(10) unsigned NOT NULL DEFAULT '0',
 `c` char(120) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '',
 `pad` char(60) COLLATE utf8mb4_general_ci NOT NULL DEFAULT '',
 PRIMARY KEY (`id`),
 KEY `k_1` (`k`)
) ENGINE=InnoDB;
`

随意写入一些数据. 然后我们执行

`-- IN C1
lock tables t1 write;
`

此时再创建一个新的连接 (之后称为 C2), 执行

`-- IN C2
SELECT COUNT(*) FROM t1;
`

此时会观测到:

`+----------+-----------------+-----------+--------+---------+-------+---------------------------------+-------------------------+
| Id | User | Host | db | Command | Time | State | Info |
+----------+-----------------+-----------+--------+---------+-------+---------------------------------+-------------------------+
| 123 | AAA | localhost | XXX | Query | 0 | starting | show processlist |
| 124 | AAA | localhost | XXX | Query | 2 | Waiting for table metadata lock | select count(*) from t1 |
+----------+-----------------+-----------+--------+---------+-------+---------------------------------+-------------------------+
`
执行 kill

`-- IN C1
KILL QUERY 124;
`
此时 C2 会显示 Query execution was interrupted, 同时 C2 的连接仍然存在可以继续执行查询.

### kill connection

类似上面的操作, 只不过把 kill 操作变为

`-- IN C1
KILL CONNECTION 124;
`
或

`-- IN C1
KILL 124;
`

此时 C2 报错为 Lost connection to MySQL server during query, 124 连接也一同消失了.

### Ctrl+C

Ctrl+C 在我们做测试时经常发生, 我们使用 MySQL 客户端以可交互的方式连接 MySQL, 之后我们发出一条 SQL , 但发现一些问题, 希望停下当前的查询, 重新执行. 此时因为之前在执行查询的 session 已经被占用了, 因此 MySQL 客户端会新建立一个临时的 connection, 将 kill query 的命令发送给 MySQL 服务, 之后再回收掉临时的 connection. 这条命令在 MySQL 服务端看起来和 kill query 看起来是一样的.

## 两种不同的 kill 响应

### kill 标记

当执行 kill 命令时, 会根据命令中指定的 process id, 查找对应的连接结构体, 并设置 kill 标记, 查询执行过程中, 会在各种位置检测 THD 上的 kill 标记, 如果发现 kill 标记被设置, 那么中止执行, 并做清理退出. 省略了很多检查部分的代码, 主要的 kill 路径代码如下:

`static uint kill_one_thread(THD *thd, my_thread_id id, bool only_kill_query) {
 THD *tmp = NULL;
 uint error = ER_NO_SUCH_THREAD;
 Find_thd_with_id find_thd_with_id(id);

 // ...
 tmp = Global_THD_manager::get_instance()->find_thd(&find_thd_with_id);
 // ...
 tmp->awake(only_kill_query ? THD::KILL_QUERY : THD::KILL_CONNECTION);
 // ...
}

void THD::awake(THD::killed_state state_to_set) {
 // ...
 if (this->m_server_idle && state_to_set == KILL_QUERY) { /* nothing */
 } else {
 killed = state_to_set;
 }
 // ...
}
`

### 执行过程中响应

* 在 SELECT 查询过程中, 每读取一行就会检查一次

 `int TableScanIterator::Read() {
 int tmp;
 while ((tmp = table()->file->ha_rnd_next(m_record))) {
 /*
 ha_rnd_next can return RECORD_DELETED for MyISAM when one thread is
 reading and another deleting without locks.
 */
 if (tmp == HA_ERR_RECORD_DELETED && !thd()->killed) continue;
 return HandleError(tmp);
 }
 if (m_examined_rows != nullptr) {
 ++*m_examined_rows;
 }
 return 0;
}

int IndexRangeScanIterator::Read() {
 int tmp;
 while ((tmp = m_quick->get_next())) {
 if (thd()->killed || (tmp != HA_ERR_RECORD_DELETED)) {
 return HandleError(tmp);
 }
 }

 if (m_examined_rows != nullptr) {
 ++*m_examined_rows;
 }
 return 0;
}
`
* 在进行表复制操作的 ALTER TABLE 命令进行时, 每读取源表的一些行, 就会进行检查, 如果发现被 kill, 那么会停止复制并删除创建的临时表

 `static int copy_data_between_tables(/* 很多参数 ... */) {
 // ...
 to->file->extra(HA_EXTRA_BEGIN_ALTER_COPY);

 while (!(error = info->Read())) {
 if (thd->killed) {
 thd->send_kill_message();
 error = 1;
 break;
 }
 // ... copy data
 }
 // ... clean up
`
* 在 UPDATE 和 DELETE 操作中, 在读取数据或者进行任何更新删除时都会检查, 如果开启了事务, 那么事务会在 kill 之后回滚

 `void delete_from_single_table(THD *) {
 // ...
 while (!(error = info->Read()) && !thd->killed) {
 DBUG_ASSERT(!thd->is_error());
 thd->inc_examined_row_count(1);
 // ...
 }
 // ...
}

void update_single_table(THD *) {
 // ...
 while (!(error = info->Read()) && !thd->killed) {
 DBUG_ASSERT(!thd->is_error());
 thd->inc_examined_row_count(1);
 // ...
 }
 // ...
}
`
* kill GET_LOCK 命令会使其返回 NULL
* 对于已经加锁的线程, 响应 kill 之后会释放已经得到的锁, 数据库中有相当多的锁, 这部分其实是一个很可能遇到的情况, MySQL 有专门的处理机制, 在下面单独讨论实现

### 等待中响应

如果查询此时等待在某个 condition_variable 上, 那么短时间内可能很难唤醒, 如果出现了死锁的情况, 那么就更不可能唤醒了, 因此, kill 实现了针对等待的特殊响应, 其主要思路是

* 在某个查询进入等待状态之前, 在 THD 上记录下当前查询等待的 condition_variable 对象及其对应的 mutex

 ` void enter_cond(mysql_cond_t *cond, mysql_mutex_t *mutex,
 const PSI_stage_info *stage, PSI_stage_info *old_stage,
 const char *src_function, const char *src_file,
 int src_line) {
 DBUG_ENTER("THD::enter_cond");
 mysql_mutex_assert_owner(mutex);
 current_mutex = mutex;
 current_cond = cond;
 enter_stage(stage, old_stage, src_function, src_file, src_line);
 DBUG_VOID_RETURN;
}
`
* 在等待的条件上增加对 thd->killed 状态的判断, 即检测到 killed 时退出等待

 `longlong Item_func_sleep::val_int() {
 THD *thd = current_thd;
 Interruptible_wait timed_cond(thd);
 mysql_cond_t cond;
 // ...
 timeout = args[0]->val_real();
 mysql_cond_init(key_item_func_sleep_cond, &cond);
 mysql_mutex_lock(&LOCK_item_func_sleep);

 thd->ENTER_COND(&cond, &LOCK_item_func_sleep, &stage_user_sleep, NULL);

 error = 0;
 thd_wait_begin(thd, THD_WAIT_SLEEP);
 while (!thd->killed) {
 error = timed_cond.wait(&cond, &LOCK_item_func_sleep);
 if (is_timeout(error)) break;
 error = 0;
 }
 thd_wait_end(thd);
 mysql_mutex_unlock(&LOCK_item_func_sleep);
 thd->EXIT_COND(NULL);

 mysql_cond_destroy(&cond);

 return (error == 0); // Return 1 killed
}

`
* kill 发生时, 使用 THD 记录的 condition_variable 进行 pthread_cond_signal, 唤醒等待者, 等待者醒来会检测 kill 标记, 发现已被 kill 从而快速退出, 这段代码也位于 THD::awake 中.

 `void THD::awake(THD::killed_state state_to_set) {
 // ...
 /* Broadcast a condition to kick the target if it is waiting on it. */
 if (is_killable) {
 mysql_mutex_lock(&LOCK_current_cond);
 if (current_cond.load() && current_mutex.load()) {
 DBUG_EXECUTE_IF("before_dump_thread_acquires_current_mutex", {
 const char act[] =
 "now signal dump_thread_signal wait_for go_dump_thread";
 DBUG_ASSERT(!debug_sync_set_action(current_thd, STRING_WITH_LEN(act)));
 };);
 mysql_mutex_lock(current_mutex);
 mysql_cond_broadcast(current_cond);
 mysql_mutex_unlock(current_mutex);
 }
 mysql_mutex_unlock(&LOCK_current_cond);
 }
 DBUG_VOID_RETURN;
}
`

## 为什么有时候无法 kill

有时候 kill 命令发出了, show processlist 可以看到 session 状态已经变为 Killed 状态了, 但是查询仍在执行, 这多数情况是由于这个 session 的查询未运行到检查点, 没有发现自己已经被 kill 了, 这个查询可能在引擎层内部执行比较复杂的工作, 也可能在读写临时表进行繁重的 IO. 此时一般只能继续等待程序运行到检查点, 发现 kill 状态后, 就会退出了.

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)