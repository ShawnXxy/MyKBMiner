# MySQL · 引擎特性 · Latch 持有分析

**Date:** 2020/03
**Source:** http://mysql.taobao.org/monthly/2020/03/07/
**Images:** 3 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2020 / 03
 ](/monthly/2020/03)

 * 当期文章

 MySQL · 引擎特性 · 8.0 Instant Add Column功能解析
* PgSQL · 引擎特性 · PostgreSQL 通信协议
* MySQL · 产品特性 · RDS三节点企业版的高可用体系
* AliSQL · 最佳实践 · Performance Agent
* MySQL · 内核分析 · InnoDB mutex 实现分析
* Database · 理论基础 · B link Tree
* MySQL · 引擎特性 · Latch 持有分析
* MySQL · 内核分析 · InnoDB 的统计信息
* MySQL · 引擎特性 · 排序实现
* PgSQL · 插件分析 · plProfiler

 ## MySQL · 引擎特性 · Latch 持有分析 
 Author: zanye.zjy 

 ## Introduction
mysql中`latch`没有死锁检测机制，通常指的是server层、innodb层的互斥锁和读写锁。当出现问题后，需要从现场core文件排查，下面介绍如何排查锁被谁持有了

## Mutex in Server
除了win之外都采用了`glibc`中的`pthread_mutex_t`，如server层中`LOCK_status`, `LOCK_thd_remove`等

#### 方法一：

`(gdb) p LOCK_status
$11 = {m_mutex = {__data = {__lock = 2, __count = 0, __owner = 102188, __nusers = 1, __kind = 3, __spins = 85, __list = {__prev = 0x0, __next = 0x0}},
 __size = "\002\000\000\000\000\000\000\000,\217\001\000\001\000\000\000\003\000\000\000U", '\000' <repeats 18 times>, __align = 2}, m_psi = 0x0}
`

这里的`__owner`为core中`LWP XXXX`后的值

![server_mutex_stack_example](.img/c7a386faf857_2020-03-27-zanye-server-mutex-stack-example.png)

#### 方法二：
切换到`__lll_lock_wait`这样`frame`上，对于64 bit系统：

`(gdb) p *(pthread_mutex_t*)$rdi
$12 = {__data = {__lock = 2, __count = 0, __owner = 102188, __nusers = 1, __kind = 3, __spins = 85, __list = {__prev = 0x0, __next = 0x0}},
 __size = "\002\000\000\000\000\000\000\000,\217\001\000\001\000\000\000\003\000\000\000U", '\000' <repeats 18 times>, __align = 2}
`

同样能找到`pthread_mutex`中的`owner`

## RW_lock in server
除了`win`之外都采用了`glibc`中的`pthread_rwlock_t`

`(gdb) frame 1
#1 0x0000000000ec2059 in native_rw_wrlock (rwp=0x7f5faf078298) at /home/admin/129_20200113173827294_121311408_code/rpm_workspace/include/thr_rwlock.h:101
101 /home/admin/129_20200113173827294_121311408_code/rpm_workspace/include/thr_rwlock.h: No such file or directory.
(gdb) p rwp
$13 = (native_rw_lock_t *) 0x7f5faf078298
(gdb) p *rwp
$14 = {__data = {__lock = 0, __nr_readers = 0, __readers_wakeup = 0, __writer_wakeup = 0, __nr_readers_queued = 0, __nr_writers_queued = 15, __writer = 61789, __shared = 0, __pad1 = 0, __pad2 = 0, __flags = 0},
 __size = '\000' <repeats 20 times>, "\017\000\000\000]\361", '\000' <repeats 29 times>, __align = 0}
`

* `__nr_readers`: 当前有多少个线程持有读锁
* `__nr_readers_queued`: 当前有多少个线程在等待获得读锁
* `__nr_writers_queued`: 当前有多少个线程在等待获得写锁，PS：写锁的优先级比读锁要高。即如果线程想获得读锁，当发现`__nr_writers_queued`不为`0`时，哪怕当前没有人获得写锁，也会将自己阻塞。目的是防止写锁饿死。
* `__writer`：写锁持有者的`LWP #`

如果有线程持有写锁，通过`__writer`很容易找到该线程；如果有线程持有了读锁，持有读锁的线程和位置可能有多个，则可以尝试通过下述方法进行排查：

`$ gdb <binary> <coredump> -ex "thread apply all bt" -ex "quit" > core.bt
$ pt-pmp core.bt > pt-pmp.log
`

在`pt-pmp.log`中，排除：

1. 出现频次高于`__nr_readers`的堆栈
2. 阻塞在获取该锁的写锁的所有线程
3. 带有`poll()`、`epoll_wait`的堆栈
4. 带有`pthread_cond_wait`的堆栈持有该读锁的可能性也比较低

由于持有读锁的线程和位置可能有多个，排查读锁持有者需要根据具体情况分析。

## RW_lock in Innodb
innodb层的读写锁，如`dict_operation_lock`、`btr_search_latches`，`checkpoint_lock`等

`(gdb) p *dict_operation_lock
$16 = {lock_word = -2, waiters = 1, recursive = true, sx_recursive = 0, writer_is_wait_ex = false, writer_thread = 140042102085376, event = 0x7f5faf05aab8, wait_ex_event = 0x7f5faf05ab58,
 cfile_name = 0x162c6d8 "/home/admin/129_20200113173827294_121311408_code/rpm_workspace/storage/innobase/dict/dict0dict.cc",
 last_s_file_name = 0x1619240 "/home/admin/129_20200113173827294_121311408_code/rpm_workspace/storage/innobase/row/row0undo.cc",
 last_x_file_name = 0x1614968 "/home/admin/129_20200113173827294_121311408_code/rpm_workspace/storage/innobase/row/row0mysql.cc", cline = 1186, is_block_lock = 0, last_s_line = 322, last_x_line = 4290, count_os_wait = 20559,
 list = {prev = 0x7f5faea79150, next = 0x7f5faea87428}, pfs_psi = 0x0}
`

* 当`lock_word = X_LOCK_DECR`时，意味着当前锁没有被任何人持有
* 当`X_LOCK_HALF_DECR < lock_word < X_LOCK_DECR`，意味着当前有一个或多个线程持有读锁
* 当`0 < lock_word <= X_LOCK_HALF_DECR`时，意味着当前有一个线程持有`SX`锁，有0个（`lock_word = X_LOCK_HALF_DECR`）或多个线程（`lock_word < X_LOCK_HALF_DECR`）持有读锁
* 当`lock_word = 0`时表示没有线程持有读锁，下一个写锁已经加上（并已获得）
* 当`lock_word < 0`是表示有线程持有一个或多个读锁，下一个写锁已经预定（仍未获得，在等待读锁释放）

1. 这里`SX`锁是一种介于`X`锁和`S`锁的锁，它阻塞`X`、`SX`锁，但不阻塞`S`锁
2. 为了更好理解`lock_word`的含义，下面简单介绍`rw_lock_t`获取写锁的操作

```
// lock_word 的初始值，意味着最多允许0x20000000个读锁同时持有
#define X_LOCK_DECR 0x20000000
// 当上SX锁时，会尝试将lock_word减少X_LOCK_HALF_DECR
#define X_LOCK_HALF_DECR 0x10000000

rw_lock_x_lock_low(rw_lock_t* lock, ulint pass, const char* file_name, ulint line) {

 // 如果lock_word>X_LOCK_HALF_DECR，尝试将lock_word减少X_LOCK_DECR
 // 如果成功，则至少预定自己为下一个写锁的持有者，返回true，否则返回false
 if (rw_lock_lock_word_decr(lock, X_LOCK_DECR, X_LOCK_HALF_DECR)) {
 
 // 预定自己为下一个写锁持有者，此时lock_word<=0，last_x_file_name:last_x_line 为上一个写锁持有者的上锁位置
 // 将自己的线程标识写入writer_thread，
 rw_lock_set_writer_id_and_recursion_flag(lock, !pass);)

 // 如果lock_word<0，说明有线程持有读锁，必须等待读锁释放
 // 阻塞直到 lock_word==0, 
 rw_lock_x_lock_wait(lock, pass, 0, file_name, line);

 } else {
 ......
 }
 
 // 成功获得写锁，last_x_file_name:last_x_line指向加锁的位置
 lock->last_x_file_name = file_name;
 lock->last_x_line = (unsigned int) line;

 return true;
}

```

再回到上述的例子：

* `lock_word=-2`，说明这里有两个线程持有了读锁，从`last_s_file_name` : `last_s_line` 可以看到加读锁的位置；
* 同时，下一个写锁已经预定，预定者由`writer_thread`指明；
* 但是，`last_x_file_name` : `last_x_line` 并不是预订者的位置，因为此时写锁还没有真正持有
* `writer_thread`指明了持有或即将持有写锁的线程id，将其转成16进制可以在堆栈中搜出：

![innodb_rw_lock_stack_example](.img/39156e7ef71f_2020-03-27-zanye-innodb-rw-lock-stack-example.png)

另外：

* 如果拿不到锁，线程会尝试自旋一段时间，如果自旋后还是拿不到锁，则让出处理器
* 自旋的时间由`innodb`参数`innodb_sync_spin_loops`、`innodb_spin_wait_delay`决定
* 如果发现所有的拿锁的线程都处于自旋状态，则可以尝试减少`innodb_sync_spin_loops`、`innodb_spin_wait_delay`

### Mutex in Innodb
`innodb`层最常见的`mutex` `latch`为`PolicyMutex<TTASEventMutex<GenericPolicy>`，这种锁和`rw_lock_t`一样是spin锁，当拿不到锁时会尝试自旋一段时间:

`spin_and_try_lock(...)
{
 ...
 for (;;) {
 // 尝试自旋，自旋的时间同样由由`innodb_sync_spin_loops`、`innodb_spin_wait_delay`决定
 is_free(max_spins, max_delay, n_spins) {
 if (try_lock()) {
 break;
 } else {
 ...
 }
 } else {
 max_spins = n_spins + step;
 }
 os_thread_yield();
 ...
 }
 ...
}

`

这种锁一般持有时间很短，在`innodb`上采用`atomic`来实现，目前没有好的办法排查加这种锁的线程和位置，但是`core`文件仍然提供了许多有用的信息：

`(gdb) p *this
$19 = {m_impl = {m_lock_word = 0, m_waiters = 0, m_event = 0x7f5faea51358, m_policy = {m_count = {m_spins = 0, m_waits = 0, m_calls = 0, m_enabled = false}, m_id = LATCH_ID_FLUSH_LIST}}, m_ptr = 0x0}
`

`m_lock_word`对应值的含义：

`/** Mutex is free */
 MUTEX_STATE_UNLOCKED = 0
 
 /** Mutex is acquired by some thread. */
 MUTEX_STATE_LOCKED = 1
 
 /** Mutex is contended and there are threads waiting on the lock. */
 MUTEX_STATE_WAITERS = 2
`

另外`m_waiters = 0`并不意味着目前没有等锁的线程，如果拿该锁的线程都处于自旋状态，`m_waiters`仍然等于`0`

如果有线程持有该锁，想要排查，同样可以用`pt-pmp`排查：

1. 排除堆栈重复次数超过`1`次的所有线程
2. 排除阻塞在获取该锁的所有线程
3. 排除带有`poll()`、`epoll_wait`的堆栈
4. 带有`pthread_cond_wait`的堆栈持有该锁的可能性也比较低
5. 阻塞在`__lll_lock_wait`的线程持有该锁的可能性比较低，持有innodb层mutex锁的线程阻塞在server层锁的可能性比较低

持有该锁的堆栈只可能出现`1`次，排查持有者需要根据具体情况分析

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)