# MySQL · 特性分析 · LOGICAL_CLOCK 并行复制原理及实现分析

**Date:** 2017/12
**Source:** http://mysql.taobao.org/monthly/2017/12/03/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 12
 ](/monthly/2017/12)

 * 当期文章

 MySQL · 引擎特性 · InnoDB 事务系统
* MySQL · 引擎特性 · Innodb 锁子系统浅析
* MySQL · 特性分析 · LOGICAL_CLOCK 并行复制原理及实现分析
* PgSQL · 源码分析 · AutoVacuum机制之autovacuum launcher
* MSSQL · 最佳实践 · SQL Server备份策略
* MySQL · 最佳实践 · 一个“异常”的索引选择
* PgSQL · 内核开发 · 利用一致性快照迁移你的数据
* PgSQL · 应用案例 · 手机行业分析、决策系统设计-实时圈选、透视、估算
* MySQL · 最佳实践 · 如何索引JSON字段
* MySQL · myrocks · 相关tools介绍

 ## MySQL · 特性分析 · LOGICAL_CLOCK 并行复制原理及实现分析 
 Author: 勉仁 

 在MySQL5.7 引入基于Logical clock的并行复制方案前，MySQL使用基于Schema的并行复制，使不同db下的DML操作可以在备库并发回放。在优化后，可以做到不同表table下并发。但是如果业务在Master端高并发写入一个库（或者优化后的表），那么slave端就会出现较大的延迟。基于schema的并行复制，Slave作为只读实例提供读取功能时候可以保证同schema下事务的因果序（Causal Consistency，本文讨论Consistency的时候均假设Slave端为只读），而无法保证不同schema间的。例如当业务关注事务执行先后顺序时候，在Master端db1写入T1，收到T1返回后，才在db2执行T2。但在Slave端可能先读取到T2的数据，才读取到T1的数据。

MySQL 5.7的LOGICAL CLOCK并行复制，解除了schema的限制，使得在主库对一个db或一张表并发执行的事务到slave端也可以并行执行。Logical Clock并行复制的实现，最初是Commit-Parent-Based方式，同一个commit parent的事务可以并发执行。但这种方式会存在可以保证没有冲突的事务不可以并发，事务一定要等到前一个commit parent group的事务全部回放完才能执行。后面优化为Lock-Based方式，做到只要事务和当前执行事务的Lock Interval都存在重叠，即保证了Master端没有锁冲突，就可以在Slave端并发执行。LOGICAL CLOCK可以保证非并发执行事务，即当一个事务T1执行完后另一个事务T2再开始执行场景下的Causal Consistency。

## LOGICAL_CLOCK Commit-Parent-Based 模式

由于在MySQL中写入是基于锁的并发控制，所以所有在Master端同时处于prepare阶段且未提交的事务就不会存在锁冲突，在Slave端执行时都可以并行执行。因此可以在所有的事务进入prepare阶段的时候标记上一个logical timestamp（实现中使用上一个提交事务的sequence_number），在Slave端同样timestamp的事务就可以并发执行。

### Master端

在SQL层实现一个全局的logical clock： commit_clock。

当事务进入prepare阶段的时候，从commit_clock获取timestamp并存储在事务中。

在transaction在引擎层提交之前，推高commit_clock。这里如果在引擎层提交之后，即释放锁后操作commit_clock，就可能出现冲突的事务拥有相同的commit-parent，所以一定要在引擎层提交前操作。

### Slave端

如果事务拥有相同的commit-parent就可以并行执行，不同commit-parent的事务，需要等前面的事务执行完毕才可以执行。

## LOGICAL_CLOCK Lock-Based模式原理及实现分析

Commit-Parent-Based 模式，用事务commit的点将clock分隔成了多个intervals。在同一个time interval中进入prepare状态的事务可以被并发。例如下面这个例子（引自[WL#7165](https://dev.mysql.com/worklog/task/?id=7165)）：

`Trx1 ------------P----------C-------------------------------->
 |
Trx2 ----------------P------+---C---------------------------->
 | |
Trx3 -------------------P---+---+-----C---------------------->
 | | |
Trx4 -----------------------+-P-+-----+----C----------------->
 | | | |
Trx5 -----------------------+---+-P---+----+---C------------->
 | | | | |
Trx6 -----------------------+---+---P-+----+---+---C---------->
 | | | | | |
Trx7 -----------------------+---+-----+----+---+-P-+--C------->
 | | | | | | |
`

每一个水平线代表一个事务。时间从左到右。P表示prepare阶段读取commit-parent的时间点。C表示事务提交前增加全局counter的时间点。垂直线表示每个提交划分出的time interval。

从上图可以看到因为Trx5和Trx6的commit-parent都是Trx2提交点，所以可以并行执行。但是Commit-Parent-Based模式下Trx4和Trx5不可以并行执行，因为Trx4的commit-parent是Trx1的提交点。Trx6和Trx7也不可以并行执行，Trx7的commit-parent是Trx5的提交点。但Trx4和Trx5有一段时间同时持有各自的所有锁，Trx6和Trx7也是，即它们之间并不存在冲突，是可以并发执行的。

针对上面的情况，为了进一步增加复制性能，MySQL将LOGICAL_CLOCK优化为Lock-Based模式，使同时hold住各自所有锁的事务可以在slave端并发执行。

### Master端

* 添加全局的事务计数器产生事务timestamp和记录当前最大事务timestamp的clock。

 `class MYSQL_BIN_LOG: public TC_LOG
{
 ...
 public:
 /* Committed transactions timestamp */
 Logical_clock max_committed_transaction;
 /* "Prepared" transactions timestamp */
 Logical_clock transaction_counter;
 ...
}
`
* 对每个事务定义其lock interval，并记录到binlog中。

 在每个transaction中添加下面两个member。

 `class Transaction_ctx
{
 ...
 int64 last_committed;
 int64 sequence_number;
 ...
}
` 

 其中last_committed表示事务lock interval的起始点，是所有锁都获得时候的max-commited-timestamp。由于在一个事务执行过程中，数据库无法知道当前的锁是否为最后一个，在实际实现的时候，对每次DML操作都更新一次last_committed。

 `static int binlog_prepare(handlerton *hton, THD *thd, bool all)
{
 ...
 if (!all)//DML操作
 {
 Logical_clock& clock= mysql_bin_log.max_committed_transaction;
 thd->get_transaction()->
 store_commit_parent(clock.get_timestamp());//更新transaction中的last_committed
 sql_print_information("stmt prepare");
 }
 ...
}

class Transaction_ctx
{
 ...
 void store_commit_parent(int64 last_arg)
 {
 last_committed= last_arg;
 }
 ...
}
` 

 sequence_number为lock interval的结束点。从理论上在最后更新last_committed后，引擎层commit前的一个时刻即可，满足这一条件的情况下时间点越靠后越能获得更大lock interval，后面在Slave执行也就能获得更大并发度。由于我们需要把该信息记录到binlog中，所以实现中在flush binlog cache到binlog文件中的时候记录。而且当前的MySQL5.7已经disable掉了设置GTID_MODE为OFF的功能，会强制记录GTID_EVENT。这样事务的last_committed和sequence_number记录在事务开头的Gtid_log_event中。

 `int
binlog_cache_data::flush(THD *thd, my_off_t *bytes_written, bool *wrote_xid)
{
 ...
 if (flags.finalized)
 {
 trn_ctx->sequence_number= mysql_bin_log.transaction_counter.step();//获取sequence_number

 if (!error)
 if ((error= mysql_bin_log.write_gtid(thd, this, &writer)))//记录Gtid_log_event
 ...
}

bool MYSQL_BIN_LOG::write_gtid(THD *thd, binlog_cache_data *cache_data,
 Binlog_event_writer *writer)
{
 ...
 Transaction_ctx *trn_ctx= thd->get_transaction();
 Logical_clock& clock= mysql_bin_log.max_committed_transaction;

 DBUG_ASSERT(trn_ctx->sequence_number > clock.get_offset());

 int64 relative_sequence_number= trn_ctx->sequence_number - clock.get_offset(); 
 int64 relative_last_committed=
 trn_ctx->last_committed <= clock.get_offset() ?
 SEQ_UNINIT : trn_ctx->last_committed - clock.get_offset();
 ...
 Gtid_log_event gtid_event(thd, cache_data->is_trx_cache(),
 relative_last_committed, relative_sequence_number,//Gtid_log_event中记录relative_last_committed和relative_sequence_number
 cache_data->may_have_sbr_stmts());
 ...
}

` 

 同时可以看到记录在Gtid_log_event(即binlog file中)的sequence_number和last_committed使用的是相对当前binlog文件的clock的值。即每个binlog file中事务的last_commited起始值为0，sequence_number为1。由于binlog切换后，需要等待上一个文件的事务执行完，所以这里记录相对值并不会引起冲突事务并发执行。这样做一个明显的好处是由于server在每次启动的时候都会生成新的binlog文件，max_committed_transaction和transaction_counter不需要持久化。
* 更新max_committed_transaction。

 max_committed_transaction的更新一定要在引擎层commit（即锁释放）之前，如果之后更新，释放的锁被其他事务获取到并且获取到last_committed小于该事务的sequence_number，就会导致有锁冲突的事务lock interval却发生重叠。

 `void
MYSQL_BIN_LOG::process_commit_stage_queue(THD *thd, THD *first)
{
 ...
 if (head->get_transaction()->sequence_number != SEQ_UNINIT)
 update_max_committed(head);
 ...
 if (head->get_transaction()->m_flags.commit_low)
 {
 if (ha_commit_low(head, all, false))
 head->commit_error= THD::CE_COMMIT_ERROR;
 ...

}
`

### Slave端

当事务的lock interval存在重叠，即代表他们的锁没有冲突，可以并发执行。下图中L代表lock interval的开始，C代表lock interval的结束。

`- 可并发执行:
 Trx1 -----L---------C------------>
 Trx2 ----------L---------C------->

- 不可并发执行:
 Trx1 -----L----C----------------->
 Trx2 ---------------L----C------->
`

slave端在并行回放时候，worker的分发逻辑在函数Slave_worker *Log_event::get_slave_worker(Relay_log_info *rli)中，MySQL5.7中添加了schedule_next_event函数来决定是否分配下一个event到worker线程。对于DATABASE并行回放该函数实现为空。

`bool schedule_next_event(Log_event* ev, Relay_log_info* rli)
{
 ...
 error= rli->current_mts_submode->schedule_next_event(rli, ev);
 ...
}

int
Mts_submode_database::schedule_next_event(Relay_log_info *rli, Log_event *ev)
{
 /*nothing to do here*/
 return 0;
}
`

Mts_submode_logical_clock的相关实现如下。

在Mts_submode_logical_clock中存储了回放事务中已经提交事务timestamp(sequence_number)的low-water-mark lwm。low-water-mark表示该事务已经提交，同时该事务之前的事务都已经提交。

`class Mts_submode_logical_clock: public Mts_submode
{
 ...
 /* "instant" value of committed transactions low-water-mark */
 longlong last_lwm_timestamp;
 ...
 longlong last_committed;
 longlong sequence_number;
`

在Mts_submode_logical_clock的schedule_next_event函数实现中会检查当前事务是否和正在执行的事务冲突，如果当前事务的last_committed比last_lwm_timestamp大，同时该事务前面还有其他事务执行，coordinator就会等待，直到确认没有冲突事务或者前面的事务已经执行完，才返回。这里last_committed等于last_lwm_timestamp的时候，实际这两个值拥有事务的lock interval是没有重叠的，也可能有冲突。在前面lock-interval介绍中，这种情况是前面一个事务执行结束，后面一个事务获取到last_committed为前面一个的sequence_number的情况，他们的lock interval没有重叠。但由于last_lwm_timestamp更新表示事务已经提交，所以等于的时候，该事务也可以执行。

`int
Mts_submode_logical_clock::schedule_next_event(Relay_log_info* rli,
 Log_event *ev)
{
 ...
 switch (ev->get_type_code())
 {
 case binary_log::GTID_LOG_EVENT:
 case binary_log::ANONYMOUS_GTID_LOG_EVENT:
 // TODO: control continuity
 ptr_group->sequence_number= sequence_number=
 static_cast<Gtid_log_event*>(ev)->sequence_number;
 ptr_group->last_committed= last_committed=
 static_cast<Gtid_log_event*>(ev)->last_committed;
 break;

 default:

 sequence_number= last_committed= SEQ_UNINIT;

 break;
 }
 ...
 if (!is_new_group)
 {
 longlong lwm_estimate= estimate_lwm_timestamp();
 if (!clock_leq(last_committed, lwm_estimate) && //如果last_committed > lwm_estimate
 rli->gaq->assigned_group_index != rli->gaq->entry) //当前事务前面还有执行的事务
 {
 ...
 if (wait_for_last_committed_trx(rli, last_committed, lwm_estimate))
 ...
 }
 ...
 }
}

@return true when a "<=" b,
 false otherwise
*/
static bool clock_leq(longlong a, longlong b)
{
if (a == SEQ_UNINIT)
 return true;
else if (b == SEQ_UNINIT)
 return false;
else
 return a <= b;
}

bool Mts_submode_logical_clock::
wait_for_last_committed_trx(Relay_log_info* rli,
 longlong last_committed_arg,
 longlong lwm_estimate_arg)
{
 ...
 my_atomic_store64(&min_waited_timestamp, last_committed_arg);//设置min_waited_timestamp
 ...
 if ((!rli->info_thd->killed && !is_error) &&
 !clock_leq(last_committed_arg, get_lwm_timestamp(rli, true)))//真实获取lwm并检查当前是否有冲突事务
 {

 //循环等待直到没有冲突事务
 do
 {
 mysql_cond_wait(&rli->logical_clock_cond, &rli->mts_gaq_LOCK);
 }
 while ((!rli->info_thd->killed && !is_error) &&
 !clock_leq(last_committed_arg, estimate_lwm_timestamp())); 
 ... 
 }
}
`

上面循环等待的时候，会等待logical_clock_cond条件然后做检查。该条件的唤醒逻辑是：当回放事务结束，如果存在等待的事务，即检查min_waited_timestamp和当前curr_lwm(lwm同时会被更新)，如果min_waited_timestamp小于等于curr_lwm，则唤醒等待的coordinator线程。

`void Slave_worker::slave_worker_ends_group(Log_event* ev, int error)
{
 ...
 if (mts_submode->min_waited_timestamp != SEQ_UNINIT)
 {
 longlong curr_lwm= mts_submode->get_lwm_timestamp(c_rli, true);//获取并更新当前lwm。

 if (mts_submode->clock_leq(mts_submode->min_waited_timestamp, curr_lwm))
 {
 /*
 There's a transaction that depends on the current.
 */
 mysql_cond_signal(&c_rli->logical_clock_cond);//唤醒等待的coordinator线程
 }
 }
 ...
}
`

## LOGICAL_CLOCK Consistency的分析

无论是Commit-Parent-Based还是Lock-Based，Master端一个事务T1和其commit后才开始的事务T2在Slave端都不会被并发回放，T2一定会等T1执行结束才开始回放。因此LOGICAL_CLOCK并发方式在Slave端只读时候的上述场景中能够保证Causal Consistency。但如果事务T2只是等待事务T1执行commit成功后再执行commit操作，那么事务T1和T2在Slave端的执行顺序就无法得到保证，用户在Slave端读取可能先读到T2再读到T1的提交。这种场景就无法满足Causal Consistency。

## slave_preserve_commit_order的简要介绍

我们在前面的介绍中了解到，当slave_parallel_type为DATABASE和LOGICAL_CLOCK的时候，在Slave端的读取操作都存在场景无法满足Causal Consistency，都可能存在Slave端并行回放时候事务顺序发生变化。复制进行中时业务方可能会在某一时刻观察到Slave的GTID_EXECUTED有空洞。那如果业务需要完整的保证Causal Consistency呢，除了使用单线程复制，是否可以在并发回放的情况下满足这一需求？

MySQL提供了slave_preserve_commit_order，使LOGICAL_CLOCK的并发执行时候满足Causal Consistency，实际获得Sequential Consistency。这里Sequential Consistency除了满足之前分析的客户端事务T1、T2先后执行操作的场景外，还满足即使T1\T2均并发执行的时候，第三个客户端在主库观察到T1先于T2发生，在备库也会观察到T1先于T2发生，即在备库获得和主库完全一致的执行顺序。

slave_preserve_commit_order实现的关键是添加了Commit_order_manager类，开启该参数会在获取worker时候向Commit_order_manager注册事务。

`Slave_worker *
Mts_submode_logical_clock::get_least_occupied_worker(Relay_log_info *rli,
 Slave_worker_array *ws,
 Log_event * ev)
{
 ...
 if (rli->get_commit_order_manager() != NULL && worker != NULL)
 rli->get_commit_order_manager()->register_trx(worker);
 ...
}

void Commit_order_manager::register_trx(Slave_worker *worker)
{
 ...
 queue_push(worker->id);
 ...
}
`

在事务进入FLUSH_STAGE前， 会等待前面的事务都进入FLUSH_STAGE。

`int MYSQL_BIN_LOG::ordered_commit(THD *thd, bool all, bool skip_commit)
{
 ...
 if (has_commit_order_manager(thd))
 {
 Slave_worker *worker= dynamic_cast<Slave_worker *>(thd->rli_slave);
 Commit_order_manager *mngr= worker->get_commit_order_manager();

 if (mngr->wait_for_its_turn(worker, all)) //等待前面的事务都进入FLUSH\_STAGE
 {
 thd->commit_error= THD::CE_COMMIT_ERROR;
 DBUG_RETURN(thd->commit_error);
 }

 if (change_stage(thd, Stage_manager::FLUSH_STAGE, thd, NULL, &LOCK_log))
 DBUG_RETURN(finish_commit(thd));
 }
 ...
}

bool Commit_order_manager::wait_for_its_turn(Slave_worker *worker,
 bool all)
{
 ...
 mysql_cond_t *cond= &m_workers[worker->id].cond;
 ...
 while (queue_front() != worker->id)
 {
 ...
 mysql_cond_wait(cond, &m_mutex);//等待condition
 }
... 
}
`

当该事务进入FLUSH_STAGE后，会通知下一个事务的worker可以进入FLUSH_STAGE。

`bool
Stage_manager::enroll_for(StageID stage, THD *thd, mysql_mutex_t *stage_mutex)
{
 bool leader= m_queue[stage].append(thd);
 if (stage == FLUSH_STAGE && has_commit_order_manager(thd))
 {
 Slave_worker *worker= dynamic_cast<Slave_worker *>(thd->rli_slave);
 Commit_order_manager *mngr= worker->get_commit_order_manager();

 mngr->unregister_trx(worker);
 }
 ...
}

void Commit_order_manager::unregister_trx(Slave_worker *worker)
{
 ...
 queue_pop();//退出队列
 if (!queue_empty())
 mysql_cond_signal(&m_workers[queue_front()].cond);//唤醒下一个
 ...
}
`

在保证binlog flush的顺序后，通过binlog_order_commit即可获取同样的提交顺序。

## 浅谈LOGICAL_CLOCK依然存在的不足

LOGICAL_CLOCK为了准确性和实现的需要，其lock interval实际实现获得的区间比理论值窄，会导致原本一些可以并发执行的事务在Slave中没有并发执行。当使用级联复制的时候，这会后面层级的Slave并发度会越来越小。

实际很多业务中，虽然事务没有Lock Interval重叠，但这些事务操作的往往是不同的数据行，也不会有锁冲突，是可以并发执行，但LOGICAL_CLOCK的实现无法使这部分事务得到并发回放。

虽然有上述不足，LOGICAL_CLOCK的复制方式在有多客户端写入同样database的场景中相比DATABASE能够获得很大的复制性能提升，实际场景中很多业务的写入也都是在一个database下。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)