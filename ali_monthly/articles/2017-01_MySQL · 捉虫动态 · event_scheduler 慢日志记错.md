# MySQL · 捉虫动态 · event_scheduler 慢日志记错

**Date:** 2017/01
**Source:** http://mysql.taobao.org/monthly/2017/01/05/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 01
 ](/monthly/2017/01)

 * 当期文章

 MySQL · 引擎特性 · InnoDB 同步机制
* MySQL · myrocks · myrocks index condition pushdown
* PgSQL · 案例分享 · PostgreSQL+HybridDB解决企业TP+AP混合需求
* MongoDB · 特性分析 · 网络性能优化
* MySQL · 捉虫动态 · event_scheduler 慢日志记错
* PgSQL · 引擎介绍 · 向量化执行引擎简介
* SQL Server · 特性分析 · 2012列存储索引技术
* PgSQL · 乱入拜年 · 小鸡吉吉和小象Pi吉(PostgreSQL)的鸡年传奇
* MySQL · 特性分析 · 5.7 error log 时区和系统时区不同
* TokuDB · 源码分析 · 一条query语句的执行过程

 ## MySQL · 捉虫动态 · event_scheduler 慢日志记错 
 Author: 襄洛 

 ## 问题背景

最近遇到了 event_scheduler 在记录慢日志时的一个 bug，在这里分享给大家。

为了方便描述问题，先构造一个简单的 event，如下：

`delimiter //
create event event1 on schedule every 5 second starts now() ends date_add(now(), interval 1 hour)
do begin
select sleep(1);
select * from t1;
select sleep(2);
end //
delimiter ;
`
其中的 t1 表中，有 2 条记录。

同时打开 event_scheduer 和 slow_log，并把慢日志的时间设置为 1s。

`set global event_scheduler = on;
set global slow_query_log = on;
set global long_query_time = 1;
set global log_output = 'TABLE';
`

待 event 执行段时间后，查询 slow_log 会看到如下的结果：

`+---------------------+------------+------------+-----------+-----------+---------------+------+----------------+-----------+-----------+------------------+-----------+
| start_time | user_host | query_time | lock_time | rows_sent | rows_examined | db | last_insert_id | insert_id | server_id | sql_text | thread_id |
+---------------------+------------+------------+-----------+-----------+---------------+------+----------------+-----------+-----------+------------------+-----------+
| 2017-01-14 16:15:33 | root[root] | 00:00:01 | 00:00:00 | 1 | 0 | test | 0 | 0 | 1 | select sleep(1) | 4 |
| 2017-01-14 16:15:33 | root[root] | 00:00:01 | 00:00:01 | 3 | 2 | test | 0 | 0 | 1 | select * from t1 | 4 |
| 2017-01-14 16:15:35 | root[root] | 00:00:03 | 00:00:01 | 4 | 0 | test | 0 | 0 | 1 | select sleep(2) | 4 |
| 2017-01-14 16:15:38 | root[root] | 00:00:01 | 00:00:00 | 1 | 0 | test | 0 | 0 | 1 | select sleep(1) | 5 |
| 2017-01-14 16:15:38 | root[root] | 00:00:01 | 00:00:01 | 3 | 2 | test | 0 | 0 | 1 | select * from t1 | 5 |
| 2017-01-14 16:15:40 | root[root] | 00:00:03 | 00:00:01 | 4 | 0 | test | 0 | 0 | 1 | select sleep(2) | 5 |
+---------------------+------------+------------+-----------+-----------+---------------+------+----------------+-----------+-----------+------------------+-----------+
`

可以看到，slow_log 中的 `select * from t1` 和 `select sleep(2)` 相关记录是有问题的：

1. `select * from t1` 不应该被记为慢日志，同时其中的 lock_time(应该为0) 和 rows_sent(应该为2) 都是错的；
2. `select sleep(2)` 的 query_time(应该为2)，lock_time(应该为0)，和 rows_sent(应该为1) 也都是错的。

rows_sent 记错比较好确认，query_time 和 lock_time 记错我们可以从 start_time 和 thread_id 对照确认，另外`select sleep(2)` 是没有拿锁的，不应该有等锁的时间。

## 问题分析

为了搞清楚这个问题，我们需要了解 slow_log 是怎么记的。

slow_log 是在语句执行完后记录的，因为加锁时间和返回记录数这些信息，在执行之后才知道，general_log 记录是在语句解析完执行前。

slow_log 是否记录判读的逻辑在 `log_slow_applicable()` 中：

a) 是否没有用到索引，并且 `log_queries_not_using_indexes` 打开（这种情况下还可能触发 throttle 导致不记录）
b) 被标记为慢 SQL(`thd->server_status & SERVER_QUERY_WAS_SLOW`)

我们这里只关心 b) 这种 case。

在 SQL 执行结束时，会先调用 `update_server_status()` 来判断是否是慢 SQL，逻辑如下：

`void update_server_status()
{
 ulonglong end_utime_of_query= current_utime();
 if (end_utime_of_query > utime_after_lock + variables.long_query_time)
 server_status|= SERVER_QUERY_WAS_SLOW;
}
`

utime_after_lock 表示的是拿到锁的时间点，server 层通过 `THD::set_time_after_lock()` 设置，引擎层(目前只有 InnoDB 支持)如果有锁请求等待时间的话，会累加到这个变量上，通过 `thd_storage_lock_wait()` 函数。

 因此一个 SQL 因为等锁而导致执行时间长的话，是不会记入慢 SQL 的。

`select * from t1` 语句执行结束，调用 `update_server_status()` 时，根据执行时间判断是不满足慢 SQL 的，但是因为 event 在执行前 server_status 没有重置，后面调用 `log_slow_applicable()` 时，`SERVER_QUERY_WAS_SLOW` 这个标志位还在，因此最终记到 slow_log 里了。

因此一个 event 在执行中，其中只要有一个语句是慢 SQL，那么后面所有的都会被记成慢SQL。

而其中时间记错的原因也是这样，每次执行语句前，start_utime 没有重置，而 `utime_after_lock` 会在执行 `select * from t1` 时拿锁被更新。

query_time 和 lock_time 的计算逻辑如下(`LOGGER::slow_log_print()`)：

` if (thd->start_utime)
 {
 query_utime= (current_utime - thd->start_utime);
 lock_utime= (thd->utime_after_lock - thd->start_utime);
 }
`

因此 event 中语句的 query_time 一直是增加的，lock_time 也不是 0。

 需要注意的是，start_time 记录的并不是语句开始执行的时间，而是记入 slow_log 时的时间。

rows_sent 也是因为没有重置，一直都是累加的，而 rows_examined 会在 `JOIN::exec()` 中被重置，因此记的是对的。

## 问题影响和解决

出现这种问题的前提是:

1. 用了 MySQL 的 event_scheduler，并且 event 有多个 SQL 语句；
2. 其中有一个 SQL 是慢SQL。

这个 bug 目前最新的 5.6.35/5.7.17 都受影响，官方已经确认，详见 [bug#84450](http://bugs.mysql.com/bug.php?id=84450)。

知道原因后，fix 也就比较简单了，在 event 中每个 SQL 语句执行前，把 server_status, start_utime, m_sent_rows_count 重置掉就好了。

正常的用户 SQL 的执行逻辑就是这么干的，在 `mysql_parse()` 里会调用 `THD::reset_for_next_command()`，但是 event 执行过程中并没有调用这个函数。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)