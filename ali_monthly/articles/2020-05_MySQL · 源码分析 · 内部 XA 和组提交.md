# MySQL · 源码分析 · 内部 XA 和组提交

**Date:** 2020/05
**Source:** http://mysql.taobao.org/monthly/2020/05/07/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2020 / 05
 ](/monthly/2020/05)

 * 当期文章

 Database · 技术方向 · 下一代云原生数据库详解
* Database · 理论基础 · 高性能B-tree索引
* Database · 理论基础 · ARIES/IM (一)
* AliSQL · 引擎特性 · Fast Query Cache 介绍
* MySQL · 源码分析 · 8.0 · DDL的那些事
* MySQL · 内核分析 · InnoDB Buffer Pool 并发控制
* MySQL · 源码分析 · 内部 XA 和组提交
* MySQL · 插件分析 · Connection Control
* MySQL · 引擎特性 · 基于GTID复制实现的工作原理

 ## MySQL · 源码分析 · 内部 XA 和组提交 
 Author: 煜溟 

 ## XA 两阶段提交

在分布式事务处理中，全局事务（global transaction）会访问和更新多个局部数据库中的数据，如果要保证全局事务的原子性，执行全局事务 T 的所有节点必须在执行的最终结果上取得一致。X/Open 组织针对分布式事务处理而提出了 XA 规范，使用两阶段提交协议（two-phase commit protocol，2PC）来保证一个全局事务 T 要么在所有节点都提交（commit），要么在所有节点都中止。

### 提交协议

考虑一个全局事务 T 的事务协调器（transaction coordinator）是 C，当执行 T 的所有事务管理器（transaction manager）都通知 C 已经完成了执行，C 开始启动两阶段提交协议，分为 prepare 和 commit 两个阶段：

#### prepare 阶段

事务协调器 C 将一条 prepare 消息发送到执行 T 的所有节点上。当各个节点的事务管理器收到 prepare 消息时，确定是否愿意提交事务 T 中自己的部分：如果可以提交，就将所有与 T 相关的日志记录强制刷盘，并记录事务 T 的状态为 prepared，然后事务管理器返回 ready 作为应答；如果无法提交，就发送 abort 消息。

#### commit 阶段

当事务协调器 C 收到所有节点对 prepare 消息的回应后进入 commit 阶段，C 可以决定是将事务 T 进行提交还是中止，如果所有参与的节点都返回了 ready 应答，则事务 T 可以提交，否则，事务 T 需要中止。之后，协调器向所有节点发送 commit 或 abort 消息，各节点收到这个消息后，将事务最终的状态更改为 commit 或 abort，并写入日志。

### 优缺点

XA 使用两阶段提交协议的主要优点是原理简介清晰、实现方便。

主要缺点是各个节点需要阻塞等待事务协调器来决定提交或中止。如果事务协调器出现故障，那全局事务就无法获得最终的状态，各个节点可能需要持有锁并等待事务协调器的恢复，这种情况称为阻塞问题，因为事务 T 需要等待协调器恢复而被阻塞。

## MySQL 内部 XA

我们知道 MySQL 存在两个日志系统：server 层的 binlog 日志和 storage 层的事务日志（例如，InnoDB 的 redolog 日志），并且支持多个存储引擎。这样产生的问题是，如何保证事务在多个日志中的原子性，即，要么都提交，要么都中止。

在单个 MySQL 实例中，使用了内部 XA 的方式来解决上述问题，其中，server 层作为事务协调器，而多个存储引擎作为事务参与者。

### 协调器对象

在实例启动时，执行初始化函数 init_server_components 中指定了谁来作为事务协调器：

` tc_log = &tc_log_dummy;
 ...
 if (total_ha_2pc > 1 || (1 == total_ha_2pc && opt_bin_log)) {
 if (opt_bin_log)
 tc_log = &mysql_bin_log;
 else
 tc_log = &tc_log_mmap;
 }
`

TC_LOG 这个抽象类的意思是 Transaction Coordinator Log，即 XA 事务协调者日志。以 TC_LOG 为基类实现了三种不同的事务协调器子类：

* MYSQL_BIN_LOG 类：如果开启了 binlog，并且有事务引擎，则 XA 协调器为 mysql_bin_log 对象，使用 binlog 物理文件记录事务状态；
* TC_LOG_MMAP 类：如果关闭了 binlog，且存在多个事务引擎，则 XA 协调器为 tc_log_mmap 对象，使用内存数据结构来记录事务状态；
* TC_LOG_DUMMY 类：其他情况，则不需要 XA，tc_log 设置为 tc_log_dummy 对象，但是不做任何事情。

本文主要关注于如何通过内部 XA 保证 binlog 和 InnoDB redolog 的一致性，即，以 binlog 作为协调器的场景。

### 两阶段提交过程

MySQL 采用了如下的过程实现内部 XA 的两阶段提交：

1. **Prepare 阶段：**InnoDB 将回滚段设置为 prepare 状态；将 redolog 写文件并刷盘；
2. **Commit 阶段：**Binlog 写入文件；binlog 刷盘；InnoDB commit；

两阶段提交保证了事务在多个引擎和 binlog 之间的原子性，以 binlog 写入成功作为事务提交的标志，而 InnoDB 的 commit 标志并不是事务成功与否的标志。

在崩溃恢复中，是以 binlog 中的 xid 和 redolog 中的 xid 进行比较，xid 在 binlog 里存在则提交，不存在则回滚。我们来看崩溃恢复时具体的情况：

1. 在 prepare 阶段崩溃，即已经写入 redolog，在写入 binlog 之前崩溃，则会回滚；
2. 在 commit 阶段，当没有成功写入 binlog 时崩溃，也会回滚；
3. 如果已经写入 binlog，在写入 InnoDB commit 标志时崩溃，则重新写入 commit 标志，完成提交。

### 崩溃恢复过程

当 XA 控制对象为 binlog 时，MYSQL_BIN_LOG::open_binlog 实现了纯虚函数 TC_LOG::open，作用是初始化并打开 XA 协调者，进入崩溃恢复流程。

首先通过 index 文件找到最后一个 binlog 文件，因为每次在 rotate 到新的 binlog 文件时，会保证没有正在提交的事务，然后将 redolog 进行一次刷盘，这样可以保证之前的 binlog 文件中的事务在 InnoDB 总是提交的。

崩溃恢复时，InnoDB 中会存在一些 prepared 状态的事务，但是还没有进入 committed 状态。调用 binlog_recover 函数，该函数使用 binlog 作为协调者来决定这些事务哪些需要回滚，哪些需要提交。

具体的，这个函数将最后一个 binlog 中完整写入的事务 XID 添加到一个 hash，这些 XID 标志着对应的事务已经完成。实现上，遍历并解析 binlog 文件中的每个 event，遇到 XID-event 时，将其中的 xid 提取出来并加入 hash。

接下来，通过 handler 接口中的 ha_recover 函数将这个 hash 传递给 InnoDB，以此告诉 InnoDB 哪些事务需要回滚。

` /*
 Call ha_recover if and only if there is a registered engine that
 does 2PC, ...
 */
 if (total_ha_2pc > 1 && ha_recover(&xids)) goto err1;
`

在 InnoDB 拿到这个 hash 后，首先调用 innobase_xa_recover 函数得到 InnoDB 中处于 prepared 状态的 xid 集合，然后遍历其中每个 prepared 状态的事务，确定是否需要回滚：

` // recovery mode
 if (info->commit_list
 ? info->commit_list->count(x) != 0
 : tc_heuristic_recover == TC_HEURISTIC_RECOVER_COMMIT) {

 // 1. 如果 XID 在 hash 里，说明 redolog 和 binlog 都已经完成了事务的刷盘，可以提交
 hton->commit_by_xid(hton, &info->list[i].id);
 } else {
 
 // 2. 如果 XID 不在 hash 里，说明 redolog 完成刷盘，但是 binlog 还没有刷盘，2PC 没有成功，需要回滚
 hton->rollback_by_xid(hton, &info->list[i].id);
 }
`

## 组提交 group commit

### 事务提交的顺序

MySQL 的内部 XA 机制保证了单个事务在 binlog 和 InnoDB 之间的原子性，接下来我们需要考虑，在多个事务并发执行的情况下，怎么保证在 binlog 和 redolog 中的顺序一致？

#### 早期解决方法

在 MySQL 5.6 版本之前，使用 prepare_commit_mutex 对整个 2PC 过程进行加锁，只有当上一个事务 commit 后释放锁，下个事务才可以进行 prepare 操作，这样完全串行化的执行保证了顺序一致。

存在的问题是，prepare_commit_mutex 的锁机制会严重影响高并发时的性能，在每个事务执行过程中， 都会至少调用 3 次刷盘操作（写 redolog，写 binlog，写 commit），多个小 IO 是非常低效的方式。

### 组提交

为了提高并发性能，肯定要细化锁粒度。MySQL 5.6 引入了 binlog 的组提交（group commit）功能，prepare 阶段不变，只针对 commit 阶段，将 commit 阶段拆分为三个过程：

1. flush stage：多个线程按进入的顺序将 binlog 从 cache 写入文件（不刷盘）；
2. sync stage：对 binlog 文件做 fsync 操作（多个线程的 binlog 合并一次刷盘）；
3. commit stage：各个线程按顺序做 InnoDB commit 操作。

其中，每个阶段有 lock 进行保护，因此保证了事务写入的顺序。

实现方法是，在每个 stage 设置一个队列，第一个进入该队列的线程会成为 leader，后续进入的线程会阻塞直至完成提交。leader 线程会领导队列中的所有线程执行该 stage 的任务，并带领所有 follower 进入到下一个 stage 去执行，当遇到下一个 stage 为非空队列时，leader 会变成 follower 注册到此队列中。

这种组提交的优势在于锁的粒度减小，三个阶段可以并发执行，从而提升效率。

#### 5.7 组提交优化：

延迟写 redo 到 group commit 阶段

MySQL 5.6 的组提交逻辑中，每个事务各自做 prepare 并写 redo log，只有到了 commit 阶段才进入组提交，因此每个事务的 redolog sync 操作成为性能瓶颈。

在 5.7 版本中，修改了组提交的 flush 阶段，在 prepare 阶段不再让线程各自执行 flush redolog 操作，而是推迟到组提交的 flush 阶段，flush stage 修改成如下逻辑：

1. 收集组提交队列，得到 leader 线程，其余 follower 线程进入阻塞；
2. leader 调用 ha_flush_logs 做一次 redo write/sync，即，一次将所有线程的 redolog 刷盘；
3. 将队列中 thd 的所有 binlog cache 写到 binlog 文件中。

这个优化是将 redolog 的刷盘延迟到了 binlog group commit 的 flush stage 之中，sync binlog 之前。通过延迟写 redolog 的方式，为 redolog 做了一次组写入，这样 binlog 和 redolog 都进行了优化。

为了更好的理解组提交的过程，可以参考这篇文章中的图解：[[图解MySQL]MySQL组提交(group commit)](https://yq.aliyun.com/articles/617776)

### 代码分析

在 MySQL 8.0 中，binlog 组提交逻辑的主要函数是 MYSQL_BIN_LOG::ordered_commit ，此时引擎层事务已经 prepare，但是还没有写 redolog，并发情况下多个线程将不断涌入这个函数中。

ordered_commit 函数明确地分为了三个阶段，组提交过程中，每个阶段的进入都要调用 MYSQL_BIN_LOG::change_stage 函数。

首先，将当前线程加入 stage 对应的 queue，如果队列为空，则当前线程成为这个 stage 的 leader 线程，负责整个 queue 的执行，如果队列非空，则当前线程进入阻塞状态，等待 commit 完成再被唤醒。

change_stage 的流程是：线程先入队，在释放上一阶段的 lock，最后申请下一阶段的 lock。这样保证了每个时刻，每个 stage 都只有一个线程在执行，从而保证了线程的顺序性。反之，如果先释放上一个 stage lock，再申请入队，后面的线程就可能赶上来，同时申请入队，从而无法保证顺序性。

`bool MYSQL_BIN_LOG::change_stage(THD *thd MY_ATTRIBUTE((unused)),
 Stage_manager::StageID stage, THD *queue,
 mysql_mutex_t *leave_mutex,
 mysql_mutex_t *enter_mutex) {
 // 入队，并释放上一个 stage lock
 // 这里 stage leader 的选举通过 leave_mutex 保证
 // Follower 线程会等待直到被 Leader 线程唤醒，然后返回 true
 if (!stage_manager.enroll_for(stage, queue, leave_mutex)) {
 DBUG_ASSERT(!thd_get_cache_mngr(thd)->dbug_any_finalized());
 DBUG_RETURN(true);
 }
 ...
 
 // leader 申请 stage lock，不会有多个线程同时申请
 // 因为，只有每个 stage queue 的 leader 会申请 stage lock
 // 第一个执行到这里的线程是 leader，后续线程都会在上面的 enroll_for 中等待
 if (need_lock_enter_mutex)
 mysql_mutex_lock(enter_mutex);
}
`

具体入队操作在函数 Stage_manager::enroll_for 中：

`bool Stage_manager::enroll_for(StageID stage, THD *thd,
 mysql_mutex_t *stage_mutex) {
 
 // 参数 queue 可能是一个由 leader 带领的链表
 
 // If the queue was empty: we're the leader for this batch
 // 主要执行链表操作
 bool leader = m_queue[stage].append(thd);

 // 释放上一个 stage 的lock
 if (stage_mutex && need_unlock_stage_mutex) mysql_mutex_unlock(stage_mutex);

 /*
 If the queue was not empty, we're a follower and wait for the
 leader to process the queue. If we were holding a mutex, we have
 to release it before going to sleep.
 */
 if (!leader) {
 mysql_mutex_lock(&m_lock_done);
 // m_cond_done 条件变量，用于接收 signal
 // thd->tx_commit_pending 判断提交是否成功
 while (thd->tx_commit_pending) mysql_cond_wait(&m_cond_done, &m_lock_done);
 mysql_mutex_unlock(&m_lock_done);
 }
 return leader;
}
`

#### Flush 阶段

change_stage 后 leader 线程进入到 flush 阶段，leader 线程获得 LOCK_log 锁，然后执行 MYSQL_BIN_LOG::process_flush_stage_queue 函数：

`int MYSQL_BIN_LOG::process_flush_stage_queue(my_off_t *total_bytes_var,
 bool *rotate_var,
 THD **out_queue_var) {
 my_off_t total_bytes = 0;

 // leader 线程在这里取出了当前的 flush queue，将 flush queue 重置为空
 // 这个时刻之后进入 ordered_commit 的第一个线程会在 change_stage 里面成为 leader
 // 但是会在 change_stage 里等待当前线程释放 flush 阶段的 lock
 // 因此，当前执行 flush 的时候，新的 flush queue 中会不断积累多个 follower thd
 THD *first_seen = stage_manager.fetch_queue_for(Stage_manager::FLUSH_STAGE);

 // redo log 批量刷盘 
 // log_buffer_flush_to_disk 将 innodb 中 prepared 状态的事务刷入 redolog
 // 即，这些事务已经填充了 mtr，并已经申请 logbuffer 的位置了
 // 通知 log_writer 线程和 log_flusher 线程将 redolog 刷到指定 LSN
 ha_flush_logs(true);
 
 // binlog 批量刷盘 
 /* Flush thread caches to binary log. */
 for (THD *head = first_seen; head; head = head->next_to_commit) {
 // 队列中每一个 thd 都进行 cache 刷盘
 // 每个线程有两个 binlog cache，分别对应事务型 event 和非事务型 event
 std::pair<int, my_off_t> result = flush_thread_caches(head);
 // 更新总共的写入bytes
 total_bytes += result.second;
 }

 *out_queue_var = first_seen;
 *total_bytes_var = total_bytes;

 // 如果 binlog 文件超过了 max_size，则准备 rotate binlog，设置 rotate_var=true
 if (total_bytes > 0 &&
 (m_binlog_file->get_real_file_size() >= (my_off_t)max_size ||
 DBUG_EVALUATE_IF("simulate_max_binlog_size", true, false)))
 *rotate_var = true;
}
`

如果在这一步完成后数据库崩溃，由于协调者 binlog 中不保证有该组事务的记录，所以 MySQL 可能会在重启后回滚该组事务。

#### Sync 阶段

flush 阶段的 leader 线程带着一个链表进入 sync 阶段的 change_stage 函数，可能成为 sync leader，也可能成为 follower，因为上一个进入 sync stage 的线程，可能还在等更之前的 sync 线程释放 lock，从而在 sync 队列里堆积，这里相当于多个 flush queue 组成了一个 sync queue。

` // 每次执行到这里，说明 group leader 进来了，即一个新的group 
 // sync_counter 是之前进入这里，但是没 sync 的次数，不包括这一次
 // get_sync_period() = sync_binlog 表示几个 group 提交一次，而不是几个 thd 提交一次

 // if 判断逻辑：
 // 1. 如果 (sync_counter + 1 >= get_sync_period())，说明这次会执行 sync
 // 那么，稍等一会，更多的 thd 进入到 sync queue，再一同提交
 // 2. 如果这次不执行 sync，没有必要等待
 //
 // 特殊情况：
 // 1. sync_binlog=0：每次 sync 都要等待，增加组内 thd 个数
 // 2. sync_binlog=1：每次 sync 都要等待，因为每次都要提交

 if (!flush_error && (sync_counter + 1 >= get_sync_period()))
 stage_manager.wait_count_or_timeout(
 opt_binlog_group_commit_sync_no_delay_count,
 opt_binlog_group_commit_sync_delay, Stage_manager::SYNC_STAGE);

 // leader 线程在这里取出了当前的 sync queue
 // 当前 queue sync 的时候，新的 sync queue 中会积累多个 flush queue
 // 可以预料，没到达 sync_period 的时候，当前线程快速通过 sync stage
 // 新的 sync queue 比较短就会被取出
 // 如果到达了 sync_period，新的 sync queue 就会积压更多的 flush queue
 final_queue = stage_manager.fetch_queue_for(Stage_manager::SYNC_STAGE);

 if (flush_error == 0 && total_bytes > 0) {
 // 每调用一次 sync 把 sync_counter +1
 // 如果 sync_counter 没到达 sync_period 直接进入 commit stage
 std::pair<bool, bool> result = sync_binlog_file(false);
 sync_error = result.first;
 }
`

如果在这一步完成后数据库崩溃，由于协调者 binlog 中已经有了事务记录，MySQL 会在重启后通过 flush 阶段中 redolog 刷盘的数据继续进行事务的提交。

#### Commit 阶段

依次将 redolog 中已经 prepare 的事务在引擎层提交，commit 阶段不用刷盘，因为 flush 阶段中的 redolog 刷盘已经足够保证数据库崩溃时的数据安全了。

commit 阶段队列的作用是承接 sync 阶段的事务，完成最后的引擎提交，使得 sync 可以尽早的处理下一组事务，最大化组提交的效率。

```
 // opt_binlog_order_commits 是否由 leader 一起做 commit
 if (opt_binlog_order_commits &&
 (sync_error == 0 || binlog_error_action != ABORT_SERVER)) {

 // 由commit leader对队列中的所有thd进行commit
 if (change_stage(thd, Stage_manager::COMMIT_STAGE, final_queue,
 leave_mutex_before_commit_stage, &LOCK_commit)) {
 DBUG_RETURN(finish_commit(thd));
 }
 THD *commit_queue =
 stage_manager.fetch_queue_for(Stage_manager::COMMIT_STAGE);
 
 // 对 queue 中每个线程执行 ha_commit_low，完成事务提交
 process_commit_stage_queue(thd, commit_queue);
 mysql_mutex_unlock(&LOCK_commit);
 } else {
 // 如果不进行 order commit，那么 sync leader 还没有 change stage
 // 需要我们手动释放 sync lock
 if (leave_mutex_before_commit_stage)
 mysql_mutex_unlock(leave_mutex_before_commit_stage);
 }

 // 通知队列中所有等待的线程
 // 通过 thd->tx_commit_pending 标志来通知 thd
 // follower 线程被唤醒后调用 finish_commit
 // 如果发现事务没有提交，会调用 ha_commit_low, 此时就不能保证 commit 的顺序了。
 stage_manager.signal_done(final_queue);

 (void)finish_commit(thd);
 
 // do_rotate 标志位在 flush 阶段被设置
 if (DBUG_EVALUATE_IF("force_rotate", 1, 0) ||
 (do_rotate && thd->commit_error == THD::CE_NONE &&
 !is_rotating_caused_by_incident)) {
 bool check_purge = false;
 mysql_mutex_lock(&LOCK_log);
 // 进行 binlog rotate 操作
 int error = rotate(false, &check_purge);
 mysql_mutex_unlock(&LOCK_log);
 }

```

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)