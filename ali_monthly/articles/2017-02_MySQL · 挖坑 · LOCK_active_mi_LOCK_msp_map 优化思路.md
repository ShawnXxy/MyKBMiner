# MySQL · 挖坑 · LOCK_active_mi/LOCK_msp_map 优化思路

**Date:** 2017/02
**Source:** http://mysql.taobao.org/monthly/2017/02/03/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 02
 ](/monthly/2017/02)

 * 当期文章

 AliSQL · 开源 · Sequence Engine
* MySQL · myrocks · myrocks之备份恢复
* MySQL · 挖坑 · LOCK_active_mi/LOCK_msp_map 优化思路
* MySQL · 源码分析 · 词法分析及其性能优化
* SQL优化 · 经典案例 · 索引篇
* MySQL · 新特性分析 · CTE执行过程与实现原理
* PgSQL · 源码分析 · PG优化器物理查询优化
* SQL Server · 特性介绍 · 聚集列存储索引
* PgSQL · 应用案例 · 聚集存储 与 BRIN索引
* PgSQL · 应用案例 · GIN索引在任意组合查询中的应用

 ## MySQL · 挖坑 · LOCK_active_mi/LOCK_msp_map 优化思路 
 Author: plinux 

 ## 背景

在MySQL中Slave相关操作一直存在一把大锁——LOCK_active_mi (5.5及之前版本，以及MariaDB)，或LOCK_msp_map（5.6及之后的版本）。
在Slave操作中大家可能经常会遇到如下懵逼的操作：

1. 线程1：STOP SLAVE；有事务要回滚，一直不结束，然后LOCK_active_mi一直被这个线程持有
2. 线程2：SHOW SLAVE STATUS；拿不到LOCK_active_mi，无法执行。

SHOW SLAVE STATUS 经常作为监控脚本的语句被自动执行，然后就不停地被卡住，线程堆积，直到 too many connections。

等到了5.6引入了多源复制之后，这个问题就更严重了，LOCK_msr_map需要在访问任何通道时都被持有，因此操作两个不同的通道也可能冲突。

Percona曾经推出了SHOW SLAVE STATUS NO_BLOCK这样的语法，不加锁查看复制状态，但是，毕竟这不是根治之法，一方面查看的数据并不一定对，还可能Crash（例如查看过程中通道被删除了），并且需要专门的语法。

特别是5.6还支持了多线程复制，IO THREAD可以多个（多通道），SQL THREAD可以并行（并行复制），这种情况下，LOCK_msr_map这么大一把锁就更加显得格格不入了。

## 解决思路

我们先来分析一下，对各个Slave通道的操作到底有哪些是真的互斥。

1. 并发读写同一个通道的运行状态：
例如 mi->running，mi->info_thd 等，已有mi->run_lock保护IO线程，mi->rli->run_lock保护SQL线程。
2. 并发读写同一个通道的执行数据：
例如 mi->master_log_pos，mi->rpl_filter 等，已有mi->data_lock保护IO线程，mi->rli->data_lock保护SQL线程。
3. 并发读写同一个通道的错误码和错误消息：
例如 mi->last_error 等，已有mi->err_lock保护IO线程，mi->rli->err_lock保护SQL线程。
4. 对于多源复制，增减通道：
msr_map结构的增删改查需要保护，否则可能在遍历所有通道时有通道增加或删除，那遍历结果就不对了。这里真的需要LOCK_msr_map保护。

可见，除了msr_map的操作真的需要全局互斥以外，其他的操作其实都有Master_info内的锁可以保护，在mi内部解决矛盾就可以，根本无需全局锁。

MySQL 5.7 给了一个改进方案，是将LOCK_msr_map从mysql_mutex_t（pthread_mutex_t）改成了Checkable_rwlock。这个方案可以解决部分只读操作时可以相互并发，但是并没有解决LOCK_msr_map保护范围太广的问题。上面我们给出的STOP SLAVE卡住（wr_lock）和SHOW SLAVE STATUS执行互斥的问题就没有解决。

为了彻底解决这个问题，我们可以参考InnoDB怎么保证Buffer Pool中Page的并发性的：

1. 每当有线程正在访问Page时，将计数器（bpage->io_fix）加一，就把这个Page Pin在内存中了。
2. LRU淘汰Page时，看到io_fix还不是0，就不能从内存中清理，因为还有人在访问，必须等到0才能清除。
3. 对Page内容的操作，有Latch来保证，避免同时有人修改页面。

因此我们也可以在每个Master_info中加一个计数器（mi->users），有线程要使用mi，就将计数器加一，不用了就减一，以此来代替加锁放锁，再用一个专门的锁（sleep_lock）来保护计数器就可以了。

加锁操作成了：

`void Master_info::use()
{
 mysql_mutex_lock(&sleep_lock);
 users++;
 mysql_mutex_unlock(&sleep_lock);
}
`

放锁操作成了：

`void Master_info::release()
{
 mysql_mutex_lock(&sleep_lock);
 if (!--users && killed)
 mysql_cond_signal(&sleep_cond);
 mysql_mutex_unlock(&sleep_lock);
 DBUG_VOID_RETURN;

}
`

每次放锁时发一个信号量，让remove_mi操作能收到信号量后再执行删除Master_info的操作。

然后原本需要LOCK_msr_map保护的Master_info操作，可以缩小范围，只需要在取出mi时拿锁就可以了。

`Master_info *get_master_info(const char *connection_name)
{
 Master_info *mi;
 DBUG_ENTER("get_master_info");
 /* Protect against inserts into msr_map */
 mysql_mutex_lock(&LOCK_msr_map);
 if ((mi= msr_map.get_mi(connection_name)))

 mi->use();
 mysql_mutex_unlock(&LOCK_msr_map);
 DBUG_RETURN(mi);
}
`

再把原来需要get_mi调用的地方，全部修改为get_master_info这个调用，就可以删掉其mysql_mutex_lock(&LOCK_msr_map)加锁保护了，放锁的mysql_mutex_unlock(&LOCK_msr_map)语句全部改成mi->release()即可。这样就不存在全局锁定了。

比如启动一个通道的复制：

`if ((mi= get_master_info(lex->mi.channel)))
 { 
 res= start_slave(thd, mi, 1 /*net report */); 
 mi->release();
}
`
完全不需要 mysql_mutex_lock(&LOCK_msr_map)和mysql_mutex_unlock(&LOCK_msr_map)来包住start_slave了对不对！

但这种修改就带来了另一个问题，要删除一个Master_info的时候，可能还有线程在使用这个mi。
因此在析构函数中需要增加一个等待，让这个mi的所有调用都释放了再清理这个mi。
有了计数器这个也很容易做到，每当收到计数器减一的信号时，看一下是不是计数器到0了，到0了就说明所有使用者全部释放了，就可以正常删除了。

`void Master_info::wait_until_free()
{
 mysql_mutex_lock(&sleep_lock);
 killed= 1;
 while (users)
 mysql_cond_wait(&sleep_cond, &sleep_lock);
 mysql_mutex_unlock(&sleep_lock);
}
`

## 效果

这样改进以后，我们再来看最开始这个典型的案例：

1. STOP SLAVE执行卡住，那么会导致这个mi或者所有mi的计数器加一。
2. SHOW SLAVE STATUS执行，在这个mi或者所有mi的计数器加一。
并不涉及到相互锁定，只是此时无法删除通道而已，这也是合理的。两个线程都能愉快的执行自己的任务。

补丁我们会在之后的AliSQL开源版本中开源，敬请期待。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)