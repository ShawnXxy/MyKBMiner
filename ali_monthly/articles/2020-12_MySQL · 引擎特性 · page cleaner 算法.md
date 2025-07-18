# MySQL · 引擎特性 · page cleaner 算法

**Date:** 2020/12
**Source:** http://mysql.taobao.org/monthly/2020/12/03/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2020 / 12
 ](/monthly/2020/12)

 * 当期文章

 Database · 发展前沿 · NewSQL数据库概述
* AliSQL · 内核新特性 · 2020技术总结
* MySQL · 引擎特性 · page cleaner 算法
* PolarDB · 引擎特性 · 历史库
* MySQL · 内核特性 · 统计信息的现状和发展

 ## MySQL · 引擎特性 · page cleaner 算法 
 Author: zhuyan 

 ## Page cleaner

### 刷脏流程

主要的代码和流程在参考文档 3，4 这种已经讲解的比较清楚了，一个 Coordinator 线程负责处理刷脏请求，计算刷脏的量，然后分配给几个 Worker 线程去刷不同的 Buffer Pool Instance, 完成刷脏后，Coordinator 线程进入下一轮刷脏。

Coordinator 和 Worker 之间通过 page_cleaner->slots[i]->state 来协同，page_cleaner_state_t 有四种状态，代码注释说明了状态之间的迁移。

`/** State for page cleaner array slot */
enum page_cleaner_state_t {
 /** Not requested any yet.
 Moved from FINISHED by the coordinator. */
 PAGE_CLEANER_STATE_NONE = 0,
 /** Requested but not started flushing.
 Moved from NONE by the coordinator. */
 PAGE_CLEANER_STATE_REQUESTED,
 /** Flushing is on going.
 Moved from REQUESTED by the worker. */
 PAGE_CLEANER_STATE_FLUSHING,
 /** Flushing was finished.
 Moved from FLUSHING by the worker. */
 PAGE_CLEANER_STATE_FINISHED
};
`

Coordinate 入口函数 buf_flush_page_coordinator_thread，主循环刷脏逻辑：
![image.png](/monthly/pic/202101/1600312609251-8644136c-c491-4eb3-b08a-ff3c656a2f83.png)

#### pc_sleep_if_needed
page cleaner 的循环刷脏周期是 1s，如果不足 1s 就需要 sleep，超过 1s 可能是刷脏太慢，不足 1s 可能是被其它线程唤醒的。

`/* The page_cleaner skips sleep if the server is
 idle and there are no pending IOs in the buffer pool
 and there is work to do. */
if (srv_check_activity(last_activity) /*和上次循环对比，有没有新增的 activity */
 || buf_get_n_pending_read_ios() || n_flushed == 0) { /* 有没有 pending io */
 ret_sleep = pc_sleep_if_needed(next_loop_time, sig_count);

 if (srv_shutdown_state != SRV_SHUTDOWN_NONE) {
 break;
 }
} else if (ut_time_ms() > next_loop_time) {
 ret_sleep = OS_SYNC_TIME_EXCEEDED;
} else {
 ret_sleep = 0;
}
`

#### 是否持续缓慢刷脏
错误日志里有时候会看到这样的日志：

`Page cleaner took xx ms to flush xx and evict xx pages
`
这个表示上一轮刷脏进行的比较缓慢，首先 ret_sleep == OS_SYNC_TIME_EXCEEDED, 并且本轮刷脏和上一轮刷脏超过 3s，warn_interval 控制输出日志的频率，如果持续打日志，就要看看 IO 延迟了。

#### sync flush
Sync flush 不受 io_capacity/io_capacity_max 的限制，所以会对性能产生比较大的影响。

`/* Note that the buf_flush_sync_lsn which is the maximum lsn that
 * primary must flush to disk, so if it greater than the oldest_lsn,
 * we still need to wake up page cleaner thread to flush. */
oldest_lsn = buf_pool_get_oldest_modification_lwm();
if (ret_sleep != OS_SYNC_TIME_EXCEEDED && srv_flush_sync &&
 oldest_lsn < buf_flush_sync_lsn) {

 /* Request flushing for threads */
 pc_request(ULINT_MAX, buf_flush_sync_lsn);

 /* Coordinator also treats requests */
 while (pc_flush_slot() > 0) {
 }

 pc_wait_finished(&n_flushed_lru, &n_flushed_list);
}
`
pc_request 是 Coordinate 分发的入口，有两个限制参数，page 数量或者 lsn，sync flush 只有对 lsn 的限制。 pc_flush_slot 和 pc_wait_finished 是刷脏和等待 worker 线程返回。

 TIPS: pc_ 前缀是 page cleaner 的缩写

#### normal flush
当系统有负载的时候，为了避免频繁刷脏影响用户，会计算出每次刷脏的 page 数量

`else if (srv_check_activity(last_activity)) {
 ulint n_to_flush;
 lsn_t lsn_limit = 0;

 /* Estimate pages from flush_list to be flushed */
 if (ret_sleep == OS_SYNC_TIME_EXCEEDED) {
 last_activity = srv_get_activity_count();
 n_to_flush =
 page_cleaner_flush_pages_recommendation(&lsn_limit, last_pages);
 } else {
 n_to_flush = 0;
 }
 
 /* Request flushing for threads */
 pc_request(n_to_flush, lsn_limit);

 /* Coordinator also treats requests */
 while (pc_flush_slot() > 0) {
 }

 pc_wait_finished(&n_flushed_lru, &n_flushed_list);
}
`

#### idle flush
系统空闲的时候不用担心刷脏影响用户线程，可以使用最大的 io_capacity 刷脏。RDS 有参数 srv_idle_flush_pct 控制刷脏比例，默认是 100%。

`} else if (ret_sleep == OS_SYNC_TIME_EXCEEDED) {
 /* no activity, slept enough */
 buf_flush_lists(PCT_IO(100), LSN_MAX, &n_flushed);
 ...
}
`

### 异步刷脏算法
在[这篇文章](https://mp.weixin.qq.com/s/i0sIfUqUUX5c_GkFTYh64Q) 中已经把刷脏算法讲解的非常清楚了，这块就把公式列一下。

`/* 总的计算公式，n_pages 是本轮尝试刷脏的量，是三个值的平均 */
#define PCT_IO(p) ((ulong)(innodb_io_capacity * ((double)(p) / 100.0)))
n_pages = (PCT_IO(pct_total) + avg_page_rate + pages_for_lsn) / 3;
if (n_pages > innodb_max_io_capacity) {
 n_pages = innodb_max_io_capacity;
}
`

#### avg_page_rate 

```
page_rate = sum_pages / time_elapsed; // 一个计算周期内的刷脏速度
avg_page_rate = (avg_page_rate + page_rate) / 2; // 平均速度

```

其中 page_rate 和 lsn_rate 都是 srv_flushing_avg_loops 秒去尝试更新一次，避免刷脏抖动太快。avg_page_rate 加入计算，也是为了平缓刷脏。

`F(avg_page_rate) = F(page_rate, srv_flushing_avg_loops);
`

#### pages_for_lsn 

```
lsn_rate = cur_lsn - prev_lsn / time_elapsed; // 一个计算周期内的lsn产生速度
lsn_avg_rate = (lsn_avg_rate + lsn_rate) / 2; // 平均速度

// lsn_avg_rate转换为脏页数
lsn_t target_lsn = oldest_lsn + lsn_avg_rate * buf_flush_lsn_scan_factor;
sum_pages_for_lsn = 计算flush list中所有小于targe_lsn的脏页数
sum_pages_for_lsn /= buf_flush_lsn_scan_factor;
pages_for_lsn = min(sum_pages_for_lsn, innodb_max_io_capacity * 2);

```

LSN 的平均产生速度包含了多少个脏页，这个参考因素可以快速 Get 到流量的变化，一定程度上增大或者减缓刷脏。

`F(pages_for_lsn) = 
F(**lsn_rate**, srv_flushing_avg_loops, buf_flush_lsn_scan_factor, innodb_max_io_capacity)
`

 Note: 这部分扫描每个 buffer pool instance 找脏页数量的时候，5.7.6 做了优化（参考文档2），每一批刷的脏页数，在各个 buffer pool instance 中根据里面脏页数量的比列分配，这样就可以做到均衡刷脏。因为各个 buffer pool instance 中的脏页比例可能是不一样的。

#### PCT_IO(pct_total) 

`pct_total = max(pct_for_dirty, pct_for_lsn);
`
因为 Redo Log 的空间是有限的，Buffer Pool 的资源是有限的，并且 Buffer Pool 中的脏页 oldest_modification_lsn 限制了 checkpoint lsn, 间接的限制了 Redo 空间的使用。所以脏页的推进会释放 buffer pool 和 redo 的可使用空间，因此在刷脏的时候也需要参考当前脏页的比例和 Redo log 的 ‘age’。

##### pct_for_dirty
`double dirty_pct = buf_get_modified_ratio_pct();
 pct_for_dirty = (dirty_pct * 100) /
(srv_max_buf_pool_modified_pct + 1)
`

除了 dirty_pct 之外，srv_max_dirty_pages_pct_lwm 和 srv_max_buf_pool_modified_pct 也影响着 pct_for_dirty 的值。具体逻辑：

`if (srv_max_dirty_pages_pct_lwm == 0) {
 /* The user has not set the option to preflush dirty
 pages as we approach the high water mark. */
 if (dirty_pct >= srv_max_buf_pool_modified_pct) {
 /* We have crossed the high water mark of dirty
 pages In this case we start flushing at 100% of
 innodb_io_capacity. */
 return (100);
 }
} else if (dirty_pct >= srv_max_dirty_pages_pct_lwm) {
 /* We should start flushing pages gradually. */
 return (static_cast<ulint>((dirty_pct * 100) /
 (srv_max_buf_pool_modified_pct + 1)));
}

return (0);
`

```
F(pct_for_dirty) = F(dirty_pct, srv_max_dirty_pages_pct_lwm, srv_max_buf_pool_modified_pct);

```

##### pct_for_lsn

```
#define PCT_IO(p) ((ulong)(srv_io_capacity * ((double)(p) / 100.0)))
age = cur_lsn > adjusted_oldest_lsn ? cur_lsn - adjusted_oldest_lsn : 0;
auto limit_for_age = log_get_max_modified_age_async();
lsn_age_factor = (age * 100) / limit_for_age;

 pct_for_lsn = (srv_max_io_capacity / srv_io_capacity) *
(lsn_age_factor * sqrt((double)lsn_age_factor)) /
 7.5

n_pages = PCT_IO(pct_for_lsn)
 = srv_io_capacity *
 (srv_max_io_capacity / srv_io_capacity) *
(lsn_age_factor * sqrt((double)lsn_age_factor)) /
 7.5 / 100
 = srv_max_io_capacity *
(lsn_age_factor * sqrt((double)lsn_age_factor)) /
 7.5 / 100

```

自适应刷脏主要影响的值是 pct_for_lsn，由开关 srv_adaptive_flushing 控制，但也不完全由开关控制。完整的逻辑，还是看代码比较直观：

`static ulint af_get_pct_for_lsn(lsn_t age) /*!< in: current age of LSN. */
{
 const lsn_t log_margin =
 log_translate_sn_to_lsn(log_free_check_margin(*log_sys));

 ut_a(log_sys->lsn_capacity_for_free_check > log_margin);

 const lsn_t log_capacity = log_sys->lsn_capacity_for_free_check - log_margin;

 lsn_t lsn_age_factor;
 lsn_t af_lwm = (srv_adaptive_flushing_lwm * log_capacity) / 100;

 if (age < af_lwm) {
 /* No adaptive flushing. */
 return (0);
 }

 auto limit_for_age = log_get_max_modified_age_async();
 ut_a(limit_for_age >= log_margin);
 limit_for_age -= log_margin;

 if (age < limit_for_age && !srv_adaptive_flushing) {
 /* We have still not reached the max_async point and
 the user has disabled adaptive flushing. */
 return (0);
 }

 /* If we are here then we know that either:
 1) User has enabled adaptive flushing
 2) User may have disabled adaptive flushing but we have reached
 max_async_age. */
 lsn_age_factor = (age * 100) / limit_for_age;

 ut_ad(srv_max_io_capacity >= srv_io_capacity);
 return (static_cast<ulint>(((srv_max_io_capacity / srv_io_capacity) *
 (lsn_age_factor * sqrt((double)lsn_age_factor))) /
 7.5));
}
`

```
F(pct_for_lsn) = F(**age**, log_capacity, srv_adaptive_flushing_lwm,
                  log_sys->max_modified_age_async, srv_adaptive_flushing, srv_max_io_capacity);

```

如果最终选择了 pct_for_lsn, 那么公式中带入会把 srv_io_capacity 约掉。

### 同步刷脏算法
同步刷脏的触发主要在 checkpoint 线程中，函数：log_consider_sync_flush

`lsn_t flush_up_to = oldest_lsn; 

/* Redo 的 age 超过 log.max_modified_age_sync 触发 sync flush */
if (current_lsn - oldest_lsn > log.max_modified_age_sync) {
 ut_a(current_lsn > log.max_modified_age_sync || in_recover_mode());

 flush_up_to = current_lsn - log.max_modified_age_sync;
}

/* 或者其他线程显示的请求到某个 LSN */
const lsn_t requested_checkpoint_lsn = log.requested_checkpoint_lsn;

if (requested_checkpoint_lsn > flush_up_to) {
 flush_up_to = requested_checkpoint_lsn;
}

if (flush_up_to > oldest_lsn) {
 log_preflush_pool_modified_pages(log, flush_up_to);
}
`

```
F(flush_up_to) = F(**age**, log.max_modified_age_sync)

```

开关控制 srv_flush_sync 在 log_preflush_pool_modified_pages 决定是否做真正的 sync_flush.

### 相关参数
* innodb_page_cleaners page cleaner 线程的数量，因为每一个 Buffer Pool Instance 同时只会有一个 pager cleaner 线程处理，所以配置的线程数不能超过 innodb_buffer_pool_instances 大小，超过就配置相同大小。
* innodb_max_dirty_pages_pct_lwm 代码中对应变量：srv_max_dirty_pages_pct_lwm， 如果系统中脏页比例超过这个值, 将会计算 pct_for_dirty 纳入到 PCT_IO(pct_total) 中。
* innodb_max_dirty_pages_pct 代码中对应变量：srv_max_buf_pool_modified_pct, 系统中最大脏页比例，和 srv_max_dirty_pages_pct_lwm 一起，影响 pct_for_dirty 的计算结果。
* innodb_adaptive_flushing_lwm 代码中对应变量：srv_adaptive_flushing_lwm，当 age (所有脏页占用的 lsn 大小) 小于 log_capacity 的srv_adaptive_flushing_lwm 比例，pc_for_lsn 为 0，也就是不启用 redo 自适应模式刷脏。
* innodb_adaptive_flushing 代码中对应变量：srv_adaptive_flushing，是否使用 redo 自适应模式刷脏，如果为 OFF, 只有 age 大于 log_sys->max_modified_age_async 才会采用 redo 自适应模式刷脏，如果为 ON, 满足 srv_adaptive_flushing_lwm 条件就采用 redo 自适应模式刷脏。
* innodb_io_capacity 代码中对应变量：srv_io_capacity，是 PCT_IO(pct_total) 的基数（但是不影响 pc_for_lsn），空闲刷脏的最大值。
 
 #define PCT_IO(p) ((ulong)(srv_io_capacity * ((double)(p) / 100.0)))
* innodb_io_capacity_max 代码中对应变量：srv_max_io_capacity, 表示系统中每次能刷的最大值。会影响 pc_for_lsn 的算法。
* innodb_flushing_avg_loops 代码中对应变量：srv_flushing_avg_loops，计算 lsn_avg_rate 和 avg_page_rate 的频率，为了让刷脏尽可能的平缓，默认 30s 更新一次。lsn_avg_rate 将会影响 pages_for_lsn 的计算，avg_page_rate 直接参数最终的 n_pages 计算。
* innodb_flush_sync 代码中对应变量：srv_flush_sync，是否触发激烈刷脏，如果是 sync_flush 的话，系统刷脏不受 srv_io_capacity 和 srv_max_io_capacity 控制，而是刷脏页到一个指定的 lsn。 checkpoint 线程会不断检测是否需要 sync_flush, 如果当前的 lsn 和 log.available_for_checkpoint_lsn 差距超过 log.max_modified_age_sync 或者有其它指定刷脏的请求（requested_checkpoint_lsn），就尝试激烈刷脏。
* innodb_lru_scan_depth Free page 不够，从 lru 中刷脏页使用，暂时不考虑刷 lru 的情况。

## 参考文档

1. 官方文档 Configuring Buffer Pool Flushing
2. 5.7.6 InnoDB page flush 优化
3. pager cleaner from 利兵
4. Innodb缓冲池刷脏的多线程实现

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)