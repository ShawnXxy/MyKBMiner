# MySQL · 引擎特性 · The design of mysql8.0 redolog

**Date:** 2019/02
**Source:** http://mysql.taobao.org/monthly/2019/02/05/
**Images:** 6 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2019 / 02
 ](/monthly/2019/02)

 * 当期文章

 POLARDB · 性能优化 · 敢问路在何方 — 论B+树索引的演进方向（中）
* MySQL · 引擎特性 · Inspecting the Content of a MySQL Histogram
* Database · 原理介绍 · Snapshot Isolation 综述
* MSSQL · 最佳实践 · 数据库备份加密
* MySQL · 引擎特性 · The design of mysql8.0 redolog
* MySQL · 源码分析 · 8.0 Functional index的实现过程
* PgSQL · 源码解析 · Json — 从使用到源码
* MySQL · 最佳实践 · 如何使用C++实现 MySQL 用户定义函数
* MySQL · 最佳实践 · MySQL多队列线程池优化
* PgSQL · 应用案例 · PostgreSQL 时间线修复

 ## MySQL · 引擎特性 · The design of mysql8.0 redolog 
 Author: 暴跳 

 InnoDB 和大部分的存储引擎一样, 都是采用WAL 的方式进行写入数据,
所有的数据都先写入到redo log, 然后后续再从buffer pool 刷脏到数据页
又或者是备份恢复的时候从redo log 恢复到buffer poll, 然后在刷脏到数据页,
WAL很重要的一点是将随机写转换成了顺序写, 所以在机械磁盘时代,
顺序写的性能远远大于随机写的背景下, 充分利用了磁盘的性能. 
但是也带来一个问题, 就是任何的写入操作都必须加锁访问, 保证上一个写入操作完成以后, 才能进行下一个写入操作.
在 InnoDB 早期版本也是这样实现, 但是随着cpu 核数的增长,
这样频繁的加锁就无法发挥多核的性能, 所以在InnoDB 8.0 改成了无锁实现
这个是官方的介绍:

[https://mysqlserverteam.com/mysql-8-0-new-lock-free-scalable-wal-design](https://mysqlserverteam.com/mysql-8-0-new-lock-free-scalable-wal-design)

### 5.6 版本实现

有两个操作需要获得全局的mutex, log_sys_t::mutex, log_sys_t::flush_order_mutex

1. 每一个用户连接有一个线程, 要写入数据之前必须先获得log_sys_t::mutex,
用来保证只有一个用户线程在写入log buffer
那么随着连接数的增加, 这个性能必然会受到影响
2. 同样的在把已经写入完成的redo log 加入到flush list 的时候,
为了保证只有一个用户线程从log buffer 上添加buffer 到flush list,
因此需要去获得log_sys_t::flush_order_mutex 来保证

如图:

![Imgur](.img/f7f9e23fcd93_ip6ZXZq.png)

因此在5.6 版本的实现中, 我们需要先获得log_sys_t::mutex, 然后写入buffer,
然后获得log_sys_t::flush_order_mutex, 释放log_sys_t::mutex, 然后把对应的 page
加入到flush list

所以8.0 无锁实现主要就是要去掉这两个mutex

### 8.0 无锁实现

**log_sys_t::mutex***

在去掉第一个log_sys_t::mutex 的时候, 通过在写入之前先预先分配地址,
然后在写入的时候往指定地址写入, 这样就无需抢mutex.
同样, 问题来了:
所有的线程都去获得lsn 地址的时候, 同样需要有一个mutex 来防止冲突, InnoDB
通过使用atomic 来达到无锁的实现, 即:
 const sn_t start_sn = log.sn.fetch_add(len);

在每一个线程获得了自己要写入的lsn 的位置以后, 写入自然就可以并发起来了.

那么在写入的时候, 如果位置在前面的线程未写完, 而位置靠后的已经写完了,
这个时候我该如何将Log buffer 中的内容写入到redo log,
肯定不允许写入的数据有空洞.

8.0 里面引入了log_writer 线程, log_writer 线程去检查log buffer 是否有空洞.
具体实现是引入了叫 recent_written 用来记录log buffer 是否连续, 
这个recent_written 是一个link_buf 实现, 类型于并查集. 因此最大允许并发写入的大小
就是这个recent_written 的大小

link_buf 实现如图:

![Imgur](.img/56262352e682_OM8n1OB.png)

这个后台线程在用户写入数据到recent_written buffer 的时候, 就被唤醒,
检查这个recent_written 连续的位置是否可以往前推进, 如果可以, 就往前走,
将recent_written buffer 中的内容写入到redo log

**log_sys_t::flush_order_mutex**

如果不去掉flush_order_mutex, 用户线程依然无法并发起来, 因为用户线程在写完redo
log 以后, 需要把对应的page 加入到flush list才可以退出, 而加入到flush list
需要去获得 flush_order_mutex 锁, 才能保证顺序的加入flush list.
因此也必须把flush_order_mutex 去掉.

具体做法允许把log buffer 中的对应的脏页无序的添加到flush list. 用户写完log
buffer 以后就可以把对应的 log buffer 对应的脏页添加到flush list.
而无需去抢flush_order_mutex. 这样可能出现加入到flush list 上的page lsn 是无序的,
因此在做checkpoint 的时候, 就无法保证每一个flush list 上面最头的page lsn
是最小的

InnoDB 用一个recent_closed 来记录添加到flush list 的这一段log buffer 是否连续,
那么容易得出, flush list 上page lsn - recent_closed.size() 得到的lsn
用于做checkpoint 肯定的安全的.

同样, InnoDB 后台有Log_closer 线程定期检查recent_closed 是否连续, 如果连续就把
recent_closed buffer 向前推进, 那么checkpoint 的信息也可以往前推进了

所以在8.0 的实现中, 把一个write redo log 的操作分成了几个阶段

1. 获得写入位置, 实现: 用户线程
2. 写入数据到log buffer 实现: 用户线程
3. 将log buffer 中的数据写入到 redo log 文件 实现: log writer
4. 将redo log 中的page cache flush 到磁盘 实现: log flusher
5. 将redo log 中的log buffer 对应的page 添加到flush list
6. 更新可以打checkpoint 位点信息 recent_closed 实现: log closer
7. 根据recent_closed 打checkpoint 信息 实现: log checkpointer

## 代码实现

**redo log 里面主要的内存结构**

1. log file. 也就是我们常见的ib_logfile 文件
2. log buffer, 通常的大小是64M. 用户在写入的时候先从mtr 拷贝到redo log buffer, 然后在log buffer 里面会加入相应的header/footer 信息, 然后由log buffer 刷到redo log file.
3. log recent written buffer 默认大小是4M, 这个是MySQL 8.0 加入的, 为的是提高写入时候的concurrent, 早5.6 版本的时候, 写入Log buffer 的时候是需要获得Lock, 然后顺序的写入到Log Buffer. 在8.0 的时候做了优化, 写入log buffer 的时候先reserve 空间, 然后后续的时候写入就可以并行的写入了, 也就是这一段的内容是允许有空洞的.
4. log recent closed buffer 默认大小也是4M, 这个也是MySQL 8.0 加入的, 可以理解为log recent written buffer 在这个log buffer 的最前面, log recent closed buffer 在log buffer 的最后面. 也是为了添加到flush list 的时候提供concurrent. 具体实现方式和log recent written buffer 类似. 5.6 版本的时候, 将page 添加到flush list 的时候, 必须有一个Mutex 加锁, 然后按照顺序的添加到flush list 上. 8.0 的时候运行recent closed buffer 大小的page 是并行的加入到flush list, 也就是这一段的内容是允许有空洞的.
5. log write ahead buffer 默认大小是 4k, 用于避免写入小于4k 大小数据的时候需要先将磁盘上的读取, 然后修改一部分的内容, 在写入回去.

**主要的lsn**

**log.write_lsn**

这个lsn 是到这个lsn 为止, 之前所有的data 已经从log buffer 写到log files了, 但是并没有保证这些log file 已经flush 到磁盘上了, 下面log.fushed_to_disk_lsn 指的才是已经flush 到磁盘的lsn 了.

这个值是由log writer thread 来更新

**log.buf_ready_for_write_lsn**

这个lsn 主要是由于redo log 引入的concurrent writes 才引进的, 也就是log recent written buffer. 也就是到了这个lsn 为止, 之前的log buffer 里面都不会有空洞,

这个值也是由 log writer thread 来更新

**log.flushed_to_disk_lsn**

到了这个lsn 为止, 所有的写入到redo log 的数据已经flush 到log files 上了

这个值是由log flusher thread 来更新

所以有 log.flushed_to_disk_lsn <= log.write_lsn <= log.buf_ready_for_write_lsn

**log.sn**

也就是不算上12字节的header, 4字节的checksum 以后的实际写入的字节数信息. 通常用这个log.sn 去换算获得当前的current_lsn

`*current_lsn = log_get_lsn(log);
inline lsn_t log_get_lsn(const log_t &log) {
 return (log_translate_sn_to_lsn(log.sn.load()));
}
constexpr inline lsn_t log_translate_sn_to_lsn(lsn_t sn) {
 return (sn / LOG_BLOCK_DATA_SIZE * OS_FILE_LOG_BLOCK_SIZE +
 sn % LOG_BLOCK_DATA_SIZE + LOG_BLOCK_HDR_SIZE);
}
`

以下几个lsn 跟checkpoint 相关

**log.buffer_dirty_pages_added_up_to_lsn**

到这个lsn 为止, 所有的redo log 对应的dirty page 已经添加到buffer pool 的flush list 了.

这个值其实就是recent_closed.tail()

inline lsn_t log_buffer_dirty_pages_added_up_to_lsn(const log_t &log) {
 return (log.recent_closed.tail());
}

这个值由log closer thread 来更新

**log.available_for_checkpoint_lsn**

到这个lsn 为止, 所有的redo log 对应的dirty page 已经flush 到btree 上了, 因此这里我们flush 的时候并不是顺序的flush, 所以有可能存在有空洞的情况, 因此这个lsn 的位置并不是最大的redo log 已经被flush 到btree 的位置. 而是可以作为checkpoint 的最大的位置.

这个值是由log checkpointer thread 来更新

**log.last_checkpoint_lsn**

到这个lsn 为止, 所有的btree dirty page 已经flushed 到disk了, 并且这个lsn 值已经被更新到了ib_logfile0 这个文件去了.

这个lsn 也是下一次recovery 的时候开始的地方, 因为last_checkpoint_lsn 之前的redo log 已经保证都flush 到btree 中去了. 所以比这个lsn 小的redo log 文件已经可以删除了, 因为数据已经都flush 到btree data page 中去了.

这个值是由log checkpointer thread 来更新

所以log.last_checkpoint_lsn <= log.available_for_checkpoint_lsn <= log.buf_dirty_pages_added_up_to_lsn

**为什么会有这么多的lsn?**

主要还是由于写redo log 这个过程被拆开成了多个异步的流程.

先写入到log buffer, 然后由log writer 异步写入到 redo log, 然后再由log flusher 异步进行刷新.

中间在log writer 写入到 redo log 的时候, 引入了log recent written buffer 来提高concurrent 写入性能.

同时在把这个page 加入到flush list 的时候, 也一样是为了提高并发, 增加了recent_closed buffer.

**redo log 模块后台thread**

![img](.img/4b24ef3e46a9_006tNbRwly1fxer5lbldcj30it0bc75a.jpg)

![img](.img/6116d5ac5670_006tNbRwly1fxn74ldrfvj30gi08674l.jpg)

在启动的函数 Log_start_background_threads 的时候, 会把相应的线程启动

` os_thread_create(log_checkpointer_thread_key, log_checkpointer, &log);

 os_thread_create(log_closer_thread_key, log_closer, &log);

 os_thread_create(log_writer_thread_key, log_writer, &log);

 os_thread_create(log_flusher_thread_key, log_flusher, &log);

 os_thread_create(log_write_notifier_thread_key, log_write_notifier, &log);

 os_thread_create(log_flush_notifier_thread_key, log_flush_notifier, &log);
`

这里主要有

**log_writer:**

log_writer 这个线程等在writer_event 这个os_event上, 然后判断的是 log.write_lsn.load() < ready_lsn. 这个ready_lsn 是去扫一下log buffer, 判断是否有新的连续的内存了. 这个线程主要做的事情就是不断去检查 log buffer 里面是否有连续的已经写入数据的内存 buffer, 执行的函数是 log_writer_write_buffer()=>log_files_write_buffer()=>write_blocks()=>fil_redo_io() =>shard->do_redo_io()=>os_file_write() =>…=> pwrite(m_fh, m_buf, m_n, m_offset);

这里这个io 是同步, 非direct IO.

将这部分的数据内容刷到redolog 中去, 但是不执行fsync 命令, 具体执行fsync 命令的是log_flusher.

**问题: 谁来唤醒Log_writer 这个线程?**

正常情况下. srv_flush_log_at_trx_commit == 1 的时候是没有人去唤醒这个log_writer, 这个os_event_wait_for 是在pthread_cond_timedwait 上的, 这个时间为 srv_log_writer_timeout = 10 微秒.

这个线程被唤醒以后, 执行log_writer_write_buffer() 后, 在执行Log_files_write_buffer() 函数里面 执行 notify_about_advanced_write_lsn() 函数去唤醒write_notifier_event,

同时, 在执行完成 log_writer_write_buffer() 后. 会判断srv_flush_log_at_trx_commit == 1 就去唤醒 log.flusher_event

**log_write_notifier:**

log_write_notifer 是等待在 write_notifier_event 这个os_event上, 然后判断的是 log.write_lsn.load() >= lsn, lsn 是上一次的log.write_lsn. 也就是判断Log.write_lsn 有没有增加, 如果有增加就唤醒这个log_write_notifier, 然后log_write_notifier 就去唤醒那些等待在 log.write_events[slot] 的用户thread.

**从上面可以看到, 由log_writer 执行os_event_set 唤醒**

有哪些线程等待在log.write_events上呢?

都是用户的thread 最后会等待在Log.write_events上, 用户的线程调用log_write_up_to, 最后根据

srv_flush_log_at_trx_commit 这个变量来判断是执行

!=1 log_wait_for_write(log, end_lsn); 然后等待在log.write_events[slot] 上.

const auto wait_stats =
​ os_event_wait_for(log.write_events[slot], max_spins,
​ srv_log_wait_for_write_timeout, stop_condition);

=1 log_wait_for_flush(log, end_lsn); 等待在log.flush_events[slot] 上.

const auto wait_stats =
​ os_event_wait_for(log.flush_events[slot], max_spins,
​ srv_log_wait_for_flush_timeout, stop_condition);

**log_flusher**

log_flusher 是等待在 log.flusher_event 上,

从上面可以看到一般来说, 由log_writer 执行os_event_set 唤醒

如果是 srv_flush_log_at_trx_commit == 1 的场景, 也就是我们最常见的写了事务, 必须flush 到磁盘, 才能返回的场景. 然后判断的是 last_flush_lsn < log.write_lsn.load(), 也就是上一次last_flush_lsn 比当前的write_lsn, 如果比他小, 说明有新数据写入了, 那么就可以执行flush 操作了,

如果是 srv_flush_log_at_trx_commit != 1 的场景, 也就是写了事务不需要保证redolog 刷盘的场景, 那么执行的是

` os_event_wait_time_low(log.flusher_event,
 flush_every_us - time_elapsed_us, 0);
`
也就是会定期的根据时间来唤醒, 然后执行 flusher 操作.

最后 执行完成flush 以后唤醒的是log.flush_notifier_event os_event_set(log.flush_notifier_event);

**log_flush_notifier**

和log_write_notifier 基本一样, 等待在 flush_notifier_event 上, 然后判断的是 log.flushed_to_disk_lsn.load() >= lsn, 这里lsn 是上一次的flushed_to_disk_lsn, 也就是判断flushed_to_disk_lsn 有没有增加, 如果有增加就唤醒等待在 flush_events[slot] 上面的用户线程, 跟上面一样, 也是用户线程最后会等待在flush_events 上

从上面可以看到, 有log_flusher 唤醒它

**log_closer**

log_closer 这个线程是在后台不断的去清理recent_closed 的线程, 在mtr/mtr0mtr.cc:execute() 也就是mtr commit 的时候, 会把这个mtr 修改的内容对应start_lsn, end_lsn 的内容添加到recent_closed buffer 里面, 并且在添加到recent_closed buffer 之前, 也会把相应的page 都挂到buffer pool 的flush list 里面.

和其他线程不一样的地方在于, Log_closer 并没有wait 在一个条件变量上, 只是每隔1s 的轮询而已.

而在这1s 一次的轮询里面, 一直执行的操作是 log_advance_dirty_pages_added_up_to_lsn() 这个函数类似recent_writtern 里面的 log_advance_ready_for_write_lsn(), 去这个recent_close 里面的Link_buf 里面

` /*
 * 从recent_closed.m_tail 一直往下找, 只要有连续的就串到一起, 直到
 * 找到有空洞的为止
 * 只要找到数据, 就更新m_tail 到最新的位置, 然后返回true
 * 一条数据都没有返回false
 * 注意: 在advance_tail_until 操作里面, 本身同时会进行的操作就是回收之前的空间
 * 所以执行完advance_tail_until 以后, 连续的内存就会被释放出来了
 * 下面还有validate_no_links 函数进行检查是否释放正确
 */
`

这样一直清理着recent_closed buffer, 就可以保证recent_closed buffer 一直是有空间的

log_closer thread 会一直更新着这个 log_advance_dirty_pages_added_up_to_lsn(), 这个函数里面就是一直去更新recent_close buffer 里面的 log_buffer_dirty_pages_added_up_to_lsn(), 然后在做check pointer 的时候, 会一直去检查这个log_buffer_dirty_pages_added_up_to_lsn(), 可以做check point 的lsn 必须小于这个log_buffer_dirty_pages_added_up_to_lsn(), 因为 log_buffer_dirty_pages_added_up_to_lsn 表示的是 recent close buffer 里面的其实位置, 在这个位置之前的Lsn 都已经被填满, 是连续的了, 在这个位置之后的lsn 没有这个保证.

那么是谁负责更新recent_closed 这个数组呢?
log_closed thread

**什么时候把dirty page 加入到buffer pool 的 flush list 上?**

在mtr->commit() 的时候, 就会把这个mtr 修改过的page 都加到flush list 上, 在添加到flush list 上之前, 我们会保证写入到redo log, 并且这个redo log 已经flush 了.

**log_checkpointer**

这个线程等待在 log.checkpointer_event 上, 然后判断的是10*1000, 也就是10s 的时间,

os_event_wait_time_low(log.checkpointer_event, 10 * 1000, sig_count);

os_event_wait_time_low 是等待checkpointer_event 被唤醒, 或者超时时间10s 到了, 其实就是pthread_cond_timedwait()

正常情况下都是等10s 然后log_checkpointer 被唤醒, 那么被通知到checkpointer_event 被唤醒的场景在哪里呢?

其实也是在 log_writer_write_buffer() 函数里面, 先判断

`while(1) {
 const lsn_t lsn_diff = min_next_lsn - checkpoint_lsn;

 if (lsn_diff <= log.lsn_capacity) {
 checkpoint_limited_lsn = checkpoint_lsn + log.lsn_capacity;
 break;
 }
 log_request_checkpoint(log, false);
 ...
}
// 为什么需要在log_writer 的过程加入这个逻辑, 这个逻辑是判断lsn_diff(当前这次要写入的数据的大小) 是否超过了log.lsn_capacity(redolog 的剩余容量大小), 如果比它小, 那么就可以直接进行写入操作, 就break 出去, 如果比它大, 那么说明如果这次写入写下去的话, 因为redolog 是rotate 形式的, 会把当前的redolog 给写坏, 所以必须先进行一次checkpoint, 把一部分的redolog 中的内容flush 到btree data中, 然后把这个checkpoint 点增加, 腾出空间.
// 所以我们看到如果checkpoint 做的不够及时, 会导致redolog 空间不够, 然后直接影响到线上的写入线程.

`

首先我们必须知道一个问题是, 一次transaction 修改的page 什么时候flush 下去, 我们是不知道的. 因为用户只需要写入到redo log, 并且确认redo log 已经flush 了以后, 就直接返回了. 至于什么时候从Buffer pool flush 到btree data, 这个是后台异步的, 用户也不关注的. 但是我们打checkpoint 以后, 在checkpoint 之前的redo log 应该是都可以删除的, 因此我们必须保证打的checkpoint lsn 的这个点之前的redo log 已经将对应的page flush到磁盘上了,

**那么这里的问题就是如何确定这个checkpoint lsn 点?**

在函数 log_update_available_for_checkpoint_lsn(log); 里面更新 log.available_for_checkpoint_lsn

具体的更新过程:

然后在log_request_checkpoint里面执行 log_update_available_for_checkpoint_lsn(log) =>

const lsn_t oldest_lsn = log_get_available_for_checkpoint_lsn(log);

然后执行 lsn_t lwn_lsn = buf_pool_get_oldest_modification_lwm() =>

buf_pool_get_oldest_modification_approx()

这里buf_pool_get_oldest_modification_approx() 指的是获得大概的最老的lsn 的位置, 这里是引入了recent_closed buffer 带来的一个问题, 因为引入了 recent_closed buffer 以后, 从redo log 上面的page 添加到buffer pool 的flush list 是不能保证有序的, 有可能一个flush list 上面存在的是 98 => 85 => 110 这样的情况. 因此这个函数只能获得大概的oldest_modification lsn

具体的做法就是遍历所有的buffer pool 的flush list, 然后只需要取出flush list 里面的最后一个元素(虽然因为引入了recent_closed 不能保证是最老的 lsn), 也就是最老的lsn, 然后对比8个flush_list, 最老的lsn 就是目前大概的lsn 了

然后在buf_pool_get_oldest_modification_lwm() 还是里面, 会将buf_pool_get_oldest_modification_approx() 获得的 lsn 减去recent_closed buffer 的大小, 这样得到的lsn 可以确保是可以打checkpoint 的, 但是这个lsn 不能保证是最大的可以打checkpoint 的lsn. 而且这个 lsn 不一定是指向一个记录的开始, 更多的时候是指向一个记录的中间, 因为这里会强行减去一个 recent_closed buffer 的size. 而以前在5.6 版本是能够保证这个lsn 是默认一个redo log 的record 的开始位置

最后通过 log_consider_checkpoint(log); 来确定这次是否要写这个checkpointer 信息

然后在 log_should_checkpoint() 具体的有3个条件来判断是否要做 checkpointer

最后决定要做的时候通过 log_checkpoint(log); 来写入checkpointer 的信息

在log_checkpoint() 函数里面

通过 log_determine_checkpoint_lsn() 来判断这次checkpointer 是要写入dict_lsn, 还是要写入available_for_checkpoint_lsn. 在 dict_lsn 指的是上一次DDL 相关的操作, 到dict_lsn 为止所有的metadata 相关的都已经写入到磁盘了, 这里为什么要把DDL 相关的操作和非 DDL 相关的操作分开呢?

最后通过 log_files_write_checkpoint 把checkpoint 信息写入到ib_logfile0 文件中

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)