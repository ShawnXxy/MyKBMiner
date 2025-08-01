# MySQL · 源码分析 · Innodb缓冲池刷脏的多线程实现

**Date:** 2018/09
**Source:** http://mysql.taobao.org/monthly/2018/09/02/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 09
 ](/monthly/2018/09)

 * 当期文章

 MySQL · 引擎特性 · B+树并发控制机制的前世今生
* MySQL · 源码分析 · Innodb缓冲池刷脏的多线程实现
* MySQL · 引擎特性 · IO_CACHE 源码解析
* MySQL · RocksDB · Memtable flush分析
* MSSQL · 最佳实践 · 使用非对称秘钥实现列加密
* MongoDB · 引擎特性 · MongoDB索引原理
* MySQL · 案例分析 · RDS MySQL线上实例insert慢常见原因分析
* Redis · 引擎特性 · 基于 LFU 的热点 key 发现机制
* MySQL · myrocks · collation 限制
* PgSQL · 应用案例 · PostgreSQL 图像搜索实践

 ## MySQL · 源码分析 · Innodb缓冲池刷脏的多线程实现 
 Author: Yuan Zhen 

 ## 简介
为了提高性能，大多数的数据库在操作数据时都不会直接读写磁盘，而是中间经过缓冲池，将要写入磁盘的数据先写入到缓冲池里，然后在某个时刻后台线程把修改的数据刷写到磁盘上。MySQL的InnoDB引擎也使用缓冲池来缓存从磁盘读取或修改的数据页，如果当前数据库需要操作的数据集比缓冲池中的空闲页面大的话，当前缓冲池中的数据页就必须进行脏页淘汰，以便腾出足够的空闲页面供当前的查询使用。如果数据库负载太高，对于空闲页面的需求超出了page cleaner的淘汰能力，这时候是否能够快速获取空闲页面，会直接影响到数据库的处理能力。5.6版本以前，脏页的清理工作交由master线程的；Page cleaner thread是5.6.2引入的一个新线程，它实现从master线程中卸下缓冲池刷脏页的工作；为了进一步提升扩展性和刷脏效率，在5.7.4版本里引入了多个page cleaner线程，从而达到并行刷脏的效果。目前Page cleaner并未和缓冲池绑定，有一个协调线程 和 多个工作线程，协调线程本身也是工作线程。工作队列长度为缓冲池实例的个数，使用一个全局slot数组表示。
下面以MySQL 5.7的5.7.23版本为例，分析具体多线程刷脏的源码实现。

## 核心数据结构
为了支持多线程并发刷脏，新实现了以下数据结构：page_cleaner_t, page_cleaner_slot_t 和 page_cleaner_state_t。

### page_cleaner_t 结构体
这个数据结构是实现多刷脏线程的核心结构。它包含了所有刷脏线程所需要的信息，以及刷脏协调线程和刷脏工作线程之间同步所需要的同步事件。因为这个结构体是由所有的刷脏线程共用的，修改任何信息都要先获取互斥锁mutex字段；is_requested和is_finished event是分别用来唤醒工作线程和最后一个完成刷脏的工作线程通知协调线程这次的刷脏完成；n_workers表示刷脏工作线程的数目；requested用来表示刷脏协调线程是否有脏页需要写到磁盘上，若是没有的话，刷脏线程只需要对LRU列表中的页回收到空闲列表中；lsn_limit表示需要刷新到lsn的位置，页的最早修改lsn必须小于这个值，它才能被刷出到磁盘上；n_slots表示这些刷脏线程需要刷脏的缓冲池实例的个数；另外还有一个比较重要的字段slots，它用来记录刷脏线程对缓冲池刷脏的当前状态，每一个slot就是一个page_cleaner_slot_t结构; n_slots_requested/n_slots_flushing/n_slots_finished主要用在刷脏过程中记录所有刷脏线程处在各个阶段的线程数目，当一开始刷脏时协调线程会把n_slots_requested设置成当前slots的总数，也即缓冲池实例的个数，而会把n_slots_flushing和n_slots_finished清0。每当一个刷脏线程完成一个缓冲池实例的刷脏n_slots_requested会减1、n_slots_finished会加1。所有的刷脏线程完成后，n_slots_requested会为0，n_slots_finished会为slots的总数目。

`/** Page cleaner structure common for all threads */
struct page_cleaner_t {
 ib_mutex_t mutex; /*!< mutex to protect whole of
 page_cleaner_t struct and
 page_cleaner_slot_t slots. */
 os_event_t is_requested; /*!< event to activate worker
 os_event_t is_finished; /*!< event to signal that all
 slots were finished. */
 volatile ulint n_workers; /*!< number of worker threads
 in existence */
 bool requested; /*!< true if requested pages
 to flush */
 lsn_t lsn_limit; /*!< upper limit of LSN to be
 flushed */
 ulint n_slots; /*!< total number of slots */
 ulint n_slots_requested;
 /*!< number of slots
 in the state
 PAGE_CLEANER_STATE_REQUESTED */
 ulint n_slots_flushing;
 /*!< number of slots
 in the state
 PAGE_CLEANER_STATE_FLUSHING */
 ulint n_slots_finished;
 /*!< number of slots
 in the state
 PAGE_CLEANER_STATE_FINISHED */
 ulint flush_time; /*!< elapsed time to flush
 requests for all slots */
 ulint flush_pass; /*!< count to finish to flush
 requests for all slots */
 page_cleaner_slot_t* slots; /*!< pointer to the slots */
 bool is_running; /*!< false if attempt
 to shutdown */
};
`

### page_cleaner_slot_t数据结构
tate 用来记录对缓冲池刷脏状态的记录，这个slot表示的缓冲池实例是否已经发起了刷脏请求（PAGE_CLEANER_STATE_REQUESTED）、是否正在刷脏（PAGE_CLEANER_STATE_FLUSHING）以及这轮的刷脏处理是否已经完成（PAGE_CLEANER_STATE_FINISHED）；n_pages_requested则记录次轮刷脏要对这个缓冲池实例刷脏的页数，在发起刷脏前由协调线程设置；而其余的各个字段都是被刷脏的工作线程返回前所设置的。n_flushed_lru和n_flushed_list 分别表示次轮刷新从LRU list刷出的页数和从flush list刷出的页数，也就是分别从函数buf_flush_LRU_list和buf_flush_do_batch返回的处理的页数；succeeded_list用来表示是否对脏页list（flush_list)刷脏成功；若是次轮要刷脏的数据页成功的放到IO的队列上则表示成功了，否则返回false；flush_lru_time和flush_list_time则分别表示刷新LRU list和flush list所用的时间；flush_lru_pass和flush_list_pass分别表示尝试对LRU list和flush list页进行刷脏的次数。当所有的刷脏线程完成后，对于每个slot的这些统计信息会统一计算到全局的page_cleaner_t结构里。

`/** Page cleaner request state for each buffer pool instance */
struct page_cleaner_slot_t {
 page_cleaner_state_t state; /*!< state of the request.
 protected by page_cleaner_t::mutex
 if the worker thread got the slot and
 set to PAGE_CLEANER_STATE_FLUSHING,
 n_flushed_lru and n_flushed_list can be
 updated only by the worker thread */
 /* This value is set during state==PAGE_CLEANER_STATE_NONE */
 ulint n_pages_requested;
 /*!< number of requested pages
 for the slot */
 /* These values are updated during state==PAGE_CLEANER_STATE_FLUSHING,
 and commited with state==PAGE_CLEANER_STATE_FINISHED.
 The consistency is protected by the 'state' */
 ulint n_flushed_lru;
 /*!< number of flushed pages
 by LRU scan flushing */
 ulint n_flushed_list;
 /*!< number of flushed pages
 by flush_list flushing */
 bool succeeded_list;
 /*!< true if flush_list flushing
 succeeded. */
 ulint flush_lru_time;
 /*!< elapsed time for LRU flushing */
 ulint flush_list_time;
 /*!< elapsed time for flush_list
 flushing */
 ulint flush_lru_pass;
 /*!< count to attempt LRU flushing */
 ulint flush_list_pass;
 /*!< count to attempt flush_list
 flushing */
};

`

## 实现刷脏多线程支持的关键函数

### 刷脏协调线程的入口函数buf_flush_page_cleaner_coordinator

buf_flush_page_cleaner_coordinator协调线程的主循环主线程以最多1s的间隔或者收到buf_flush_event事件就会触发进行一轮的刷脏。协调线程首先会调用pc_request()函数，这个函数的作用就是为每个slot代表的缓冲池实例计算要刷脏多少页，然后把每个slot的state设置PAGE_CLEANER_STATE_REQUESTED, 唤醒等待的工作线程。由于协调线程也会和工作线程一样做具体的刷脏操作，所以它在唤醒工作线程之后，会调用pc_flush_slot()，和其它的工作线程并行去做刷脏页操作。一但它做完自己的刷脏操作，就会调用pc_wait_finished()等待所有的工作线程完成刷脏操作。完成这一轮的刷脏之后，协调线程会收集一些统计信息，比如这轮刷脏所用的时间，以及对LRU和flush_list队列刷脏的页数等。然后会根据当前的负载计算应该sleep的时间、以及下次刷脏的页数，为下一轮的刷脏做准备。在主循环线程跳过与多线程刷脏不相关的部分，主循环的核心主要就集中在pc_request()、pc_flush_slot()以及pc_wait_finished()三个函数的调用上。精简后的部分代码如下：

` while (srv_shutdown_state == SRV_SHUTDOWN_NONE) {

 ......
 ulint n_to_flush;
 lsn_t lsn_limit = 0;

 /* Estimate pages from flush_list to be flushed */
 if (ret_sleep == OS_SYNC_TIME_EXCEEDED) {
 last_activity = srv_get_activity_count();
 n_to_flush =
 page_cleaner_flush_pages_recommendation(
 &lsn_limit, last_pages);
 } else {
 n_to_flush = 0;
 }

 /* Request flushing for threads */
 pc_request(n_to_flush, lsn_limit);

 /* Coordinator also treats requests */
 while (pc_flush_slot() > 0) {
 /* No op */
 }
 ......

 pc_wait_finished(&n_flushed_lru, &n_flushed_list);

 ......
 }

`

### 工作线程的入口函数 buf_flush_page_cleaner_worker

buf_flush_page_cleaner_worker工作线程的主循环启动后就等在page_cleaner_t的is_requested事件上，一旦协调线程通过is_requested唤醒所有等待的工作线程，工作线程就调用pc_flush_slot()函数去完成刷脏动作。

### pc_request、pc_flush_slot以及pc_wait_finished这三个核心函数的实现

request这个函数的作用主要就是为每个slot代表的缓冲池实例计算要刷脏多少页；然后把每个slot的state设置PAGE_CLEANER_STATE_REQUESTED；把n_slots_requested设置成当前slots的总数，也即缓冲池实例的个数，同时把n_slots_flushing和n_slots_finished清0，然后唤醒等待的工作线程。这个函数只会在协调线程里调用，其核心代码如下：

` mutex_enter(&page_cleaner->mutex); //由于page_cleaner是全局的，在修改之前先获取互斥锁

 page_cleaner->requested = (min_n > 0); //是否需要对flush_list进行刷脏操作，还是只需要对LRU列表刷脏
 page_cleaner->lsn_limit = lsn_limit; // 设置lsn_limit, 只有数据页的oldest_modification小于它的才会刷出去

 for (ulint i = 0; i < page_cleaner->n_slots; i++) {
 page_cleaner_slot_t* slot = &page_cleaner->slots[i];

 //为两种特殊情况设置每个slot需要刷脏的页数，当为ULINT_MAX表示服务器比较空闲，则刷脏线程可以尽可能的把当前的所有脏页都刷出去；而当为0是，表示没有脏页可刷。
 if (min_n == ULINT_MAX) {
 slot->n_pages_requested = ULINT_MAX;
 } else if (min_n == 0) {
 slot->n_pages_requested = 0;
 }

 slot->state = PAGE_CLEANER_STATE_REQUESTED; //在唤醒刷脏工作线程之前，将每个slot的状态设置成requested状态
 }

 // 协调线程在唤醒工作线程之前，设置请求要刷脏的slot个数，以及清空正在刷脏和完成刷脏的slot个数。只有当完成的刷脏个数等于总的slot个数时，才表示次轮的刷脏结束。
 page_cleaner->n_slots_requested = page_cleaner->n_slots; 
 page_cleaner->n_slots_flushing = 0;
 page_cleaner->n_slots_finished = 0;

 os_event_set(page_cleaner->is_requested);

 mutex_exit(&page_cleaner->mutex);

`

pc_flush_slot是刷脏线程真正做刷脏动作的函数，协调线程和工作线程都会调用。由于刷脏线程和slot并不是事先绑定对应的关系。所以工作线程在刷脏时首先会找到一个未被占用的slot，修改其状态，表示已被调度，然后对该slot所对应的缓冲池instance进行操作。直到所有的slot都被消费完后，才进入下一轮。通过这种方式，多个刷脏线程实现了并发刷脏缓冲池。一旦找到一个未被占用的slot，则需要把全局的page_cleaner里的n_slots_rqeusted减1、把n_slots_flushing加1，同时这个slot的状态从PAGE_CLEANER_STATE_REQUESTED状态改成PAGE_CLEANER_STATE_FLUSHING。然后分别调用buf_flush_LRU_list() 和buf_flush_do_batch() 对LRU和flush_list刷脏。刷脏结束把n_slots_flushing减1，把n_slots_finished加1，同时把这个slot的状态从PAGE_CLEANER_STATE_FLUSHING状态改成PAGE_CLEANER_STATE_FINISHED状态。同时若这个工作线程是最后一个完成的，则需要通过is_finished事件，通知协调进程所有的工作线程刷脏结束。
已删除流程无关代码代码，其核心代码如下：

`
 for (i = 0; i < page_cleaner->n_slots; i++) { //由于slot和刷脏线程不是事先定好的一一对应关系，所以在每个工作线程开始要 先找到一个未被处理的slot
 slot = &page_cleaner->slots[i];

 if (slot->state == PAGE_CLEANER_STATE_REQUESTED) {
 break;
 }
 }

 buf_pool_t* buf_pool = buf_pool_from_array(i); // 根据找到的slot，对应其缓冲池的实例

 page_cleaner->n_slots_requested--; // 表明这个slot开始被处理，将未被处理的slot数减1
 page_cleaner->n_slots_flushing++; //这个slot开始刷脏，将flushing加1
 slot->state = PAGE_CLEANER_STATE_FLUSHING; // 把这个slot的状态设置为flushing状态

 if (page_cleaner->n_slots_requested == 0) { //若是所有的slot都处理了，则清楚is_requested的通知标志
 os_event_reset(page_cleaner->is_requested);
 }

 /* Flush pages from end of LRU if required */
 slot->n_flushed_lru = buf_flush_LRU_list(buf_pool); // 开始刷LRU队列

 /* Flush pages from flush_list if required */
 if (page_cleaner->requested) { // 刷flush_list队列
 slot->succeeded_list = buf_flush_do_batch(
 buf_pool, BUF_FLUSH_LIST,
 slot->n_pages_requested,
 page_cleaner->lsn_limit,
 &slot->n_flushed_list);
 } else {
 slot->n_flushed_list = 0;
 slot->succeeded_list = true;
 }

 page_cleaner->n_slots_flushing--; // 刷脏工作线程完成次轮刷脏后，将flushing减1
 page_cleaner->n_slots_finished++; //刷脏工作线程完成次轮刷脏后，将完成的slot加一
 slot->state = PAGE_CLEANER_STATE_FINISHED; // 设置此slot的状态为FINISHED

 if (page_cleaner->n_slots_requested == 0
 && page_cleaner->n_slots_flushing == 0) {
 os_event_set(page_cleaner->is_finished); // 当所有的工作线程都完成了刷脏，要通知协调进程，本轮刷脏完成
 }
`

pc_wait_finished函数的主要由协调线程调用，它主要用来收集每个工作线程分别对LRU和flush_list列表刷脏的页数。以及为每个slot清0次轮请求刷脏的页数和重置它的状态为NONE。

` os_event_wait(page_cleaner->is_finished); // 协调线程通知工作线程和完成自己的刷脏任务之后，要等在is_finished事件上，知道最后一个完成的工作线程会set这个事件唤醒协调线程

 mutex_enter(&page_cleaner->mutex);

 for (ulint i = 0; i < page_cleaner->n_slots; i++) { 
 page_cleaner_slot_t* slot = &page_cleaner->slots[i];

 ut_ad(slot->state == PAGE_CLEANER_STATE_FINISHED);

 // 统计每个slot分别通过LRU和flush_list队列刷出去的页数
 *n_flushed_lru += slot->n_flushed_lru;
 *n_flushed_list += slot->n_flushed_list;
 all_succeeded &= slot->succeeded_list;

 // 把所有slot的状态设置为NONE
 slot->state = PAGE_CLEANER_STATE_NONE;

 //为每个slot清除请求刷脏的页数
 slot->n_pages_requested = 0; 
 }

 // 清零完成的slot刷脏个数，为下一轮刷脏重新统计做准备
 page_cleaner->n_slots_finished = 0; 

 // 清除is_finished事件的通知标志
 os_event_reset(page_cleaner->is_finished);

 mutex_exit(&page_cleaner->mutex);
`

## 总结
在MySQL 5.7中，Innodb通过定义page_cleaner_t, page_cleaner_slot_t 和 page_cleaner_state_t等数据结构，以及pc_request、pc_flush_slot和pc_wait_finished等函数实现了多线程的刷脏，提高了刷脏的效率，尽可能的避免用户线程参与刷脏。

## 参考

[MySQL · 引擎特性 · InnoDB Buffer Pool](http://mysql.taobao.org/monthly/2017/05/01/)

[MySQL · 性能优化· 5.7.6 InnoDB page flush 优化](http://mysql.taobao.org/monthly/2015/03/02/)

[MySQL · 源码分析 · InnoDB LRU List刷脏改进之路](http://mysql.taobao.org/monthly/2017/11/05/)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)