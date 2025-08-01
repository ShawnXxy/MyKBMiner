# MySQL · 捉虫动态 · 任性的 normal shutdown

**Date:** 2015/06
**Source:** http://mysql.taobao.org/monthly/2015/06/07/
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

 ## MySQL · 捉虫动态 · 任性的 normal shutdown 
 Author: 济天 

 ## 问题描述

在RDS生产环境中，一个MySQL实例莫名地被shutdown了, 日志中有如下信息：

`150525 15:30:52 [Note] User 'userxx' issued shutdown command
150525 15:30:52 [Note] /path/to/mysqld: Normal shutdown
150525 15:30:52 [Note] Stop asynchronous binlog_dump to slave (server_id: xxxxx)
150525 15:30:52 [Note] Event Scheduler: Killing the scheduler thread, thread id xxx
150525 15:30:52 [Note] Event Scheduler: Waiting for the scheduler thread to reply
150525 15:30:52 [Note] Event Scheduler: Stopped
150525 15:30:52 [Note] Event Scheduler: Purging the queue. 0 events
150525 15:30:53 [Note] User 'userxx' issued shutdown command
150525 15:31:07 [Note] Slave I/O thread exiting, read up to log 'log.xxxxx', position xxxxxx
150525 15:31:07 [Note] Error reading relay log event: slave SQL thread was killed
150525 15:31:09 [Note] User 'userxx' issued shutdown command
`

以下日志是 RDS 实例特有的日志，RDS实例会将用户的重要操作记录在错误日志中。

`150525 15:30:52 [Note] User 'userxx' issued shutdown command
`

从日志可以看出：

1. 实例是正常关闭的
2. 用户在极短的时间内执行了多次shutdown命令

## 问题分析

首先我们来查看用户userxx信息，比较奇怪的是，用户userxx为普通用户，并没有执行shutdown的权限。
第一感觉很可能是MySQL权限模块出现了bug, 导致普通用户也可以执行shutdown命令。于是在一个测试实例上，建立相同权限的同名用户，验证发现userxx确实没有权限执行shutdown命令。

进一步从源码中来分析，查找源码中所有可能执行shutdown的路径。从源码中扫描COM_SHUTDOWN 出现的地方,于是在`dispatch_command`函数中发现一处比较可疑的地方，代码如下：

` thd->set_time();
 if (!thd->is_valid_time())
 {
 /*
 If the time has got past 2038 we need to shut this server down
 We do this by making sure every command is a shutdown and we
 have enough privileges to shut the server down

 TODO: remove this when we have full 64 bit my_time_t support
 */
 thd->security_ctx->master_access|= SHUTDOWN_ACL;
 command= COM_SHUTDOWN;
 }
`

MySQL 每次执行一条命令前，会获取一个系统当前时间(`thd->set_time()`)，如果获取的时间不合法(超过2038年或小于0），那么此条命令会自动转为shutdown命令。

如果用户多个连接并发执行命令，并且获取的时间不合法，那么每个连接都会执行shutdown命令，这和我们前面看到的日志中的现象很吻合。

看来问题集中在为什么获取时间会不合法？

最可能的原因是当前主机系统时间设置超过了2038, 于是查看系统时间，然而并没有如我们所愿，系统时间是正常的。

最后我们从系统日志中发现了端倪，

`May 25 15:29:49 xxx kernel: : [4768743.131263] [<ffffffff8109bff3>] ? ktime_get+0x63/0xe0
May 25 15:29:49 xxx kernel: : [4768743.131267] [<ffffffff810726f7>] ? __do_softirq+0xb7/0x1e0
May 25 15:29:49 xxx kernel: : [4768743.131271] [<ffffffff8100c24c>] ? call_softirq+0x1c/0x30
May 25 15:29:49 xxx kernel: : [4768743.131274] [<ffffffff8100de85>] ? do_softirq+0x65/0xa0
May 25 15:29:49 xxx kernel: : [4768743.131276] [<ffffffff810724e5>] ? irq_exit+0x85/0x90
`

差不多在同一时刻系统出现较多的软中断，导致获取系统时间出现错误，即超过2038年或小于0。

## 改进

从错误日志中我们表面上看到普通用户执行了shutdown命令，这个带来了疑惑和误导。因此我们做了如下改进：

1. 此种情况下，在错误日志中打印详细的日志信息，说明shutdown是由于时间获取错误导致；
2. 增加重试机制，在第一次获取时间不合法情况下，不直接执行shutdown，而是增加重试重新获取时间，如果还是不合法，再执行shutdown。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)