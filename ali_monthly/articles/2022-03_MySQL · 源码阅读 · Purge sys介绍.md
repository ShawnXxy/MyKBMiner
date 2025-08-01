# MySQL · 源码阅读  · Purge sys介绍

**Date:** 2022/03
**Source:** http://mysql.taobao.org/monthly/2022/03/01/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2022 / 03
 ](/monthly/2022/03)

 * 当期文章

 MySQL · 源码阅读 · Purge sys介绍
* MySQL · 源码分析 · Row log分析

 ## MySQL · 源码阅读 · Purge sys介绍 
 Author: 沧何 

 ## Purge sys
参考代码8.0.23，purge系统的目的是将不被任何read可见的旧版本数据（undo数据）进行清理。它包含两类处理线程：一类是一个purge coordinator thread，另一类是innodb_purge_threads-1个purge worker threads。

* purge coordinator thread：清理删除记录，把purge任务放入队列，唤醒purge worker线程，还要truncate undolog
* purge worker threads：只用于清理删除记录，处理完purge任务后则sleep待coordinator thread唤醒

mysql在srv_sys->sys_threads数组中维护内部线程，用于暂停或重启内部线程，purge coordinator thread存入SRV_PURGE_SLOT(1)位置，purge worker threads存入2 ~ srv_sys->n_sys_threads位置。

purge sys中有个purge_queue，它是个按Rollback segements的txn_no排序的最小堆。每次通过该purge_queue获取最早提交的事务所产生的undo log，然后进行清理。

## Purge coordinator thread流程
purge coordinator thread在正常清理阶段，分批清理undo数据，每批最多清理300个undo log（具体的清理工作由purge worker thread执行）。由于清理期间依然有新事务在产生新的undo数据，清理一批后根据剩余待清理的undo log个数，来判断是要增加还是减少执行清理的worker线程个数。由于有innodb_purge_threads-1个purge worker threads，不执行清理的worker线程都处于sleep状态，要执行清理的worker线程会被purge coordinator thread唤醒，唤醒就是通过srv_sys->sys_threads数组中保存的slot->event进行的。

事务提交会将旧版本的数据放入undo segment中，其中普通表的旧版本数据放入redo rollback segment，临时表的旧版本数据放入unredo rollback segment，并将这两个rollback segment存入purge sys的purge_queue中。purge coordinator thread清理undo时，借助purge sys的purge_queue，就能确保取出的rollback segment是按照事务提交的顺序（txn_no）。

purge coordinator thread在每批清理前会先将事务系统中缓存的ReadView保存在purge_sys->view中，再按事务提交的顺序从purge sys->purge_queue中取出所有待清理的undo log，再确保待清理的undo log的trx_no不能超过purge_sys->view的m_low_limit_no（小于该值的undo log不再被该ReadView需要），然后按table_id进行分组存入map中，直到在map中存满300个undo log，如果当前rollback segment中不够，则会继续从purge sys的purge_queue中取出下一个rollback segment，重复该过程。

继续将map分组后的undo logs轮转存入清理任务purge_sys->query->thrs[i]中，例如map中包含3组undo logs，分别为table1、table2、table3，将table1的undo logs存入清理任务purge_sys->query->thrs[1]，将table2的undo logs存入清理任务purge_sys->query->thrs[2]，将table3的undo logs存入清理任务purge_sys->query->thrs[3]。再将其中前n_use_threads个清理任务存入srv_sys->tasks，并唤醒n_use_threads-1个purge worker thread，剩余一个清理任务交由purge coordinator thread来处理。

由于每批清理都只清理300个undo log，而一个rollback segment会包含多个undo log，所以需要在purge sys中记录上次清理的位置，也就是purge_sys->hdr_page_no和purge_sys->hdr_offset，这样下次清理时可以继续从上次上次清理的位置继续清理。

`srv_start_purge_threads
 v
 v
srv_purge_coordinator_thread
 |
 |-> (while !srv_purge_should_exit) 正常清理阶段
 | v
 | srv_do_purge 把多余的线程当做一个池子，只有purge跟不上更新的时候，才会去调度这些线程
 | v
 | (while !srv_purge_should_exit)
 | |
 | |-> 若trx_sys->rseg_history_len相比上次purge有增长时，或者超过了innodb_max_purge_lag，则 ++n_use_threads
 | |
 | |-> 否则说明数据库处于不活跃状态(srv_check_activity), 则 –-n_use_threads
 | |
 | |-> trx_purge
 | |
 | |-> 从trx_sys->mvcc->m_views中获取oldest的ReadView，拷贝到purge_sys->view，所有在该ReadView开启前提交的事务所产生的undo都被认为是可以清理的
 | |
 | |-> trx_purge_attach_undo_recs
 | | |
 | | |-> (for 300个待获取的undo log)
 | | | |
 | | | |-> trx_purge_fetch_next_rec 获取undo log
 | | | | |
 | | | | |-> (若purge_sys中未缓存上次清理的undo log)
 | | | | | v
 | | | | | trx_purge_choose_next_log 获取最早提交的事务所产生的undo log，然后进行清理
 | | | | | |
 | | | | | |-> TrxUndoRsegsIterator::set_next 把purge_sys->rseg_iter移到下一个有效的rseg
 | | | | | | |
 | | | | | | |-> 从purge_sys->purge_queue中取出trx_no最小的所有rsegs，存入purge_sys->rseg_iter->m_trx_undo_rsegs
 | | | | | | | 包含redo rollback segment(普通表)和noredo rollback segment(临时表)
 | | | | | | |
 | | | | | | |-> purge_sys->rseg指向m_trx_undo_rsegs中第一个rseg
 | | | | | | |
 | | | | | | |-> purge_sys->rseg_iter->m_iter 指向m_trx_undo_rsegs中下一个rseg
 | | | | | | |
 | | | | | | |-> 更新purge_sys->iter.trx_no & hdr_offset & hdr_page_no指向当前rseg上次purge到的位置
 | | | | | | |
 | | | | | | |-> purge_sys->next_stored = true
 | | | | | |
 | | | | | |-> trx_purge_read_undo_rec
 | | | | | |
 | | | | | |-> (!purge_sys->rseg->last_del_marks) 已读到该rseg的结尾
 | | | | | |
 | | | | | |-> (purge_sys->rseg->last_del_marks)
 | | | | | |
 | | | | | |-> trx_undo_get_first_rec 读取当前的undo log记录
 | | | | | |
 | | | | | |-> 更新purge_sys->offset & page_no & iter.undo_no iter.undo_rseg_space 指向当前的undo log
 | | | | |
 | | | | |-> 若purge_sys->iter.trx_no >= purge_sys->view.m_low_limit_no，则后续undo log不再清理
 | | | | |
 | | | | |-> trx_undo_build_roll_ptr 获取undo log的回滚段指针roll_ptr
 | | | | |
 | | | | |-> 获取undo log的对应事务的modifier_trx_id
 | | | | |
 | | | | |-> trx_purge_get_next_rec
 | | | | |
 | | | | |-> (若已读到该rseg的结尾) 获取下一个最早提交的事务所产生的undo log，然后进行清理
 | | | | | |
 | | | | | |-> trx_purge_rseg_get_next_history_log
 | | | | | |
 | | | | | |-> trx_purge_choose_next_log -> xxx
 | | | | |
 | | | | |-> (否则) trx_undo_page_get_next_rec 获取下一个undo log，并缓存在purge_sys中
 | | | |
 | | | |-> 按undo log的table_id进行分组存入group_by map中
 | | |
 | | |-> 从group_by map中，将table_id相同的一组undo log轮转存入清理任务purge_sys->query->thrs[i]中
 | |
 | |-> (for n_use_threads-1 个worker线程)
 | | |
 | | |-> que_fork_scheduler_round_robin 从purge_sys->query->thrs取出一个清理任务
 | | | v
 | | | thr->state = QUE_THR_RUNNING
 | | |
 | | |-> srv_que_task_enqueue_low 将该清理任务存入srv_sys->tasks
 | | v
 | | srv_release_threads 唤醒一个worker线程，它会从srv_sys->tasks中取出一个清理任务
 | |
 | |-> que_fork_scheduler_round_robin
 | |
 | |-> trx_purge_truncate 每128次purge, truncate一次history list，所以每隔一会，才看到history list长度变小
 | v
 | trx_purge_truncate_history
 | |
 | |-> (for rseg in 所有undo space中的rsegs)
 | v
 | trx_purge_truncate_rseg_history
 | v
 | trx_purge_remove_log_hdr 从rseg的history list中删除该undo log
 |
 |-> (while srv_fast_shutdown == 0 && n_pages_purged > 0) slow shutdown要避免在退出上述循环后，有新的记录加入
 | v
 | trx_purge -> xxx
 |
 |-> trx_purge(truncate=true) -> xxx 最后对history list做一次truncate，并确保所有worker线程退出
`
## Purge worker thread流程
purge worker thread被唤醒后，会从srv_sys->tasks中取出一个清理任务，清理掉其中的undo logs（具有相同的table id）。对于每个待清理的undo log，会先解析undo log内容，检查该undo log是否能进行清理，若能清理，先删除二级索引再删除clustered索引。

`srv_worker_thread
 v
(while purge_sys->state != PURGE_STATE_EXIT) worker线程总是在coordinator线程退出之后再退出
 v
srv_task_execute
 |
 |-> 持有srv_sys->tasks_mutex锁，从srv_sys->tasks中取出一个thr，然后释放tasks_mutex
 |
 |-> que_run_threads
 |
 |-> que_run_threads_low
 |
 |-> que_thr_node_step
 |
 |-> row_purge_step 从清理任务thr中取出undo logs（具有相同的table id），进行清理
 |
 |-> row_purge
 | v
 | (for undo_rec in 清理任务thr中的所有undo logs)
 | |
 | |-> row_purge_parse_undo_rec 解析undo log，检查该undo log是否能进行清理
 | |
 | |-> row_purge_record 清理undo log
 | v
 | row_purge_del_mark 先删除二级索引再删除clustered索引
 | |
 | |-> row_purge_remove_multi_sec_if_poss
 | |
 | |-> row_purge_remove_clust_if_poss
 |
 |-> row_purge_end
`

参考： 

[InnoDB事务系统](http://mysql.taobao.org/monthly/2017/12/01/) 

[InnoDB的read view，回滚段和purge过程简介](http://mysql.taobao.org/monthly/2018/03/01/zzai)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)