# MySQL · 引擎特性 · 说说InnoDB Log System的隐藏参数

**Date:** 2019/06
**Source:** http://mysql.taobao.org/monthly/2019/06/07/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2019 / 06
 ](/monthly/2019/06)

 * 当期文章

 MySQL · 引擎特性 · 安全及权限改进相关
* MySQL · 最佳实践 · RDS MySQL 8.0 语句级并发控制
* CloudDBA · 最佳实践 · Performance Insights
* PgSQL · 应用案例 · 学生为什么应该学PG
* MongoDB · 引擎特性 · 4.2 新特性解读
* PgSQL · 答疑解惑 · 垃圾回收、膨胀、多版本管理、存储引擎
* MySQL · 引擎特性 · 说说InnoDB Log System的隐藏参数
* MySQL · 引擎特性 · CHECK CONSTRAINT
* PgSQL · 应用案例 · 如何修改PostgreSQL分区表分区范围
* PgSQL · 应用案例 · 什么情况下可能表膨胀

 ## MySQL · 引擎特性 · 说说InnoDB Log System的隐藏参数 
 Author: weixiang 

 InnoDB在设计lock-free的log system时，除了已有的参数外，还通过宏控制隐藏了一些参数，如果你使用源码编译时，打开cmake选项-DENABLE_EXPERIMENT_SYSVARS=1, 就可以看到这些参数了。本文主要简单的过一下这些隐藏的参数所代表的含义

A.
innodb_log_write_events
innodb_log_flush_events
两者的含义类似，表示用来唤醒等待log write/flush的event的个数，默认值都是2048
比如你要等待的位置在lsnA，那么计算的slot为:
slot = (lsnA - 1) /OS_FILE_LOG_BLOCK_SIZE & (innodb_log_write/flush_events - 1)
 这意味着：如果事务的commit log的end lsn落在相同block里，他们可能产生event的竞争
 当然如果不在同一个block的时候，如果调大参数，就可以减少竞争，但也会有无效的唤醒
 唤醒操作通常由后台线程log_write_notifier 或者log_flush_notifier异步来做，但如果推进的log write/flush还不足一个block的话，那就log_writter/flusher
 自己去唤醒了。

B.
 innodb_log_recent_written_size, 默认1MB
 表示recent_written这个link_buf的大小，其实控制了并发往log buffer中同时拷贝的事务日志量，向前由新的日志加入，后面由log writer通过写日志向前推进，如果写的慢的话，那这个link_buf很可能用满，用户线程就得spin等待。再慢io的系统上，我们可以稍微调大这个参数

innodb_Log_recent_closed_size, 默认2MB
 表示recent closed这个link_buf的大小，也是维护可以并发往flush list上插入脏页的并罚度，如果插入脏页速度慢，或者lin_buf没有及时合并推进，就会spin wait

` 简单说下link_buf, 这本质上是一个数组，但使用无锁的使用方式来维护lsn的推进，比如获得一个lsn开始和结束，那就
 通过设置buf[start_lsn] = end_lsn的类似方式来维护lsn链，基于lsn是连续值的事实，最终必然不会出现空洞，所以在演化的过程中，可以从尾部
 推进连续的lsn，头部插入新的值.
 如果新插入的值超过了尾部，表示buf满了，就需要spin wait了
`

C.
 innodb_log_wait_for_write_spin_delay， 
 innodb_log_wait_for_write_timeout

从8.0版本开始用户线程不再自己去写redo，而是等待后台线程去写，这两个变量控制了spin以及condition wait的timeout时间，当spin一段时间还没推进到某个想要的lsn点时，就会进入condition wait

另外两个变量
 innodb_log_wait_for_flush_spin_delay
 innodb_log_wait_for_flush_timeout
 含义类似，但是是等待log flush到某个指定lsn

注意在实际计算过程中，最大spin次数，会考虑到cpu利用率，以及另外两个参数:
 innodb_log_spin_cpu_abs_lwm
 innodb_log_spin_cpu_pct_hwm

如果是等待flush操作的话，还收到参数innodb_log_wait_for_flush_spin_hwm限制，该参数控制了等待flush的时间上限，如果平均等待flush的时间超过了这个上限的话, 就没必要去spin，而是直接进入condition wait

关于spin次数的计算方式在函数`log_max_spins_when_waiting_in_user_thread`中”:

函数的参数即为配置项innodb_log_wait_for_write_spin_delay或innodb_log_wait_for_flush_spin_delay值

` static inline uint64_t log_max_spins_when_waiting_in_user_thread(
 uint64_t min_non_zero_value) {
 uint64_t max_spins;

 /* Get current cpu usage. */
 const double cpu = srv_cpu_usage.utime_pct;

 /* Get high-watermark - when cpu usage is higher, don't spin! */
 const uint32_t hwm = srv_log_spin_cpu_pct_hwm;

 if (srv_cpu_usage.utime_abs < srv_log_spin_cpu_abs_lwm || cpu >= hwm) {
 /* Don't spin because either cpu usage is too high or it's
 almost idle so no reason to bother. */
 max_spins = 0;

 } else if (cpu >= hwm / 2) {
 /* When cpu usage is more than 50% of the hwm, use the minimum allowed
 number of spin rounds, not to increase cpu usage too much (risky). */
 max_spins = min_non_zero_value;

 } else {
 /* When cpu usage is less than 50% of the hwm, choose maximum spin rounds
 in range [minimum, 10*minimum]. Smaller usage of cpu is, more spin rounds
 might be used. */
 const double r = 1.0 * (hwm / 2 - cpu) / (hwm / 2);

 max_spins =
 static_cast<uint64_t>(min_non_zero_value + r * min_non_zero_value * 9);
 }

 return (max_spins);
 }
`

D. 以下几个参数是后台线程等待任务时spin及condition wait timeout的值
log_writer线程：
innodb_log_writer_spin_delay,
 innodb_log_writer_timeout

log_flusher线程：
 innodb_ log_flusher_spin_delay
 innodb_log_flusher_timeout

log_write_notifier线程：
 innodb_ log_write_notifier_spin_delay
 innodb_log_write_notifier_timeout

log_flush_notifier线程
 innodb_log_flush_notifier_spin_delay
 innodb_log_flush_notifier_timeout

log_closer线程（用于推进recent_closed这个link_buf的专用线程）
 innodb_log_closer_spin_delay
 innodb_log_closer_timeout

E
 innodb_ log_write_max_size
 表示允许一个write操作最大的字节数，默认为4kb， 这个是在推进recent_written这个link buf时计算的，个人认为这个限制太小了，可以适当调大这个参数。（然而8.0的最大写入限制还受到innodb_log_write_ahead_size限制，两者得综合起来看）

F
 innodb_log_checkpoint_every
 默认1000毫秒（1秒），表示至少每隔这么长时间log_checkpointer线程会去尝试做一次checkpoint. 当然是否做checkpoint还受到其他因素的影响，具体见函数`log_should_checkpoint`:
 ` 
 a) more than 1s elapsed since last checkpoint
 b) checkpoint age is greater than max_checkpoint_age_async
 c) it was requested to have greater checkpoint_lsn,
 and oldest_lsn allows to satisfy the request
 `

G. 参考：
 MySQL8.0.16源代码

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)