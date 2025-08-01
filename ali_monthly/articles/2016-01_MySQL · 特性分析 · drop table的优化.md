# MySQL · 特性分析 · drop table的优化

**Date:** 2016/01
**Source:** http://mysql.taobao.org/monthly/2016/01/07/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 01
 ](/monthly/2016/01)

 * 当期文章

 MySQL · 引擎特性 · InnoDB 事务锁系统简介
* GPDB   · 特性分析· GreenPlum Primary/Mirror 同步机制
* MySQL · 专家投稿 · MySQL5.7 的 JSON 实现
* MySQL · 特性分析 · 优化器 MRR & BKA
* MySQL · 答疑解惑 · 物理备份死锁分析
* MySQL · TokuDB · Cachetable 的工作线程和线程池
* MySQL · 特性分析 · drop table的优化
* MySQL · 答疑解惑 · GTID不一致分析
* PgSQL · 特性分析 · Plan Hint
* MariaDB · 社区动态 · MariaDB on Power8 (下)

 ## MySQL · 特性分析 · drop table的优化 
 Author: lengxiang 

 ## 背景

系统为了加速对象的访问，通常都会增加一层缓存，以缓解下一层IO的瓶颈，OS的page cache和数据库的buffer pool都基于此。

但对象的删除，如果同步清理对象的缓存的话，不仅大大增加了延时，同时可能因为缓存过大导致IO blooding。所以针对缓存的清理，都会采用lazy drop的优化，下面我们就来对比下percona和官方针对drop table的lazy drop 优化。

假设使用`innodb_file_per_table`为表创建独立的tablespace，在业务处理过程中，有删除表的动作，会发现`drop table`操作不仅仅持续比较长，而且在删除过程中，实例的QPS也有所降低，主要是因为在清理buffer pool过程中，持有buffer pool的mutex导致，percona server在5.1版本开始引入 `lazy drop table`来消除drop table过程中带来的影响，但也并没有完全消除，MySQL 官方在5.5.23以后也引入了 `lazy drop table` 来优化drop 操作，下面我们就来对比一下这两种方式的差异。

值得一提的是：在日常的运维中，drop的操作并非核心需求，我们也都建议DBA在 off-peak 时间去做这样的操作。

## 同步模式

在讨论lazy模式之前，我们先看看MySQL在5.5.23版本之前的处理方式即同步模式:
当要drop table的时候，会在整个操作过程中持有buffer pool的mutex，然后扫描两次LRU链表，把属于这个table的page失效掉，buffer pool中page的个数越多，持有mutex时间就会越长，对在线业务的影响也就越明显。

简短看下核心处理代码:

`fil_delete_tablespace
buf_LRU_invalidate_tablespace(
 ulint id) /*!< in: space id */
{
 ulint i;()
 for (i = 0; i < srv_buf_pool_instances; i++) {
 buf_pool_t* buf_pool;

 buf_pool = buf_pool_from_array(i);
 buf_LRU_drop_page_hash_for_tablespace(buf_pool, id);
 buf_LRU_invalidate_tablespace_buf_pool_instance(buf_pool, id);
 }
}
`

1. `buf_LRU_drop_page_hash_for_tablespace`会扫描一次LRU list，需要从adaptive hash中删除对要删除的表的page的引用；
2. `buf_LRU_invalidate_tablespace_buf_pool_instance`会扫描一次LRU list:
 如果是dirty block，需要从flush list remove掉，然后从page hash中删除，最后从LRU list中删除。

可以看到，这种同步清理掉内存结构的操作，在业务高峰期，对系统的吞吐能力会产生不小的波动。

## Percona lazy模式

percona实现了一个`lazy drop table`模式，使用参数控制：

`mysql> show global variables like '%lazy%';
+------------------------+-------+
| Variable_name | Value |
+------------------------+-------+
| innodb_lazy_drop_table | 0 |
+------------------------+-------+
`

其处理drop table的过程如下:

1. 持有buffer pool的lru list mutex锁；
2. 开始扫描LRU list中的page；
 
 如果这个page属于要删除的table的，就设置一个flag，表示这个page所在的表正在被删除

 释放lru list mutex锁；
 持有一个adaptive hash index的shared latch；
 开始扫描buffer pool中的block；
 如果这个page被AHI索引；
 1. 释放AHI 锁
2. 持有page的exclusive lock
3. 删除AHI中索引这个page的entries
4. 释放page锁
5. 持有AHI的shared lock进行下一个page的判断

相比较同步模式，Percona的lazy drop table在扫描lru list过程中，只set了一个flag，随后在lru正常的淘汰过程中或者flush dirty block的时候如果碰到这中block，直接就做删除处理了，这也就是lazy的核心。

其核心代码如下：

`buf_LRU_mark_space_was_deleted(
 ulint id) /*!< in: space id */
{
 ulint i;

/* 这一部分代码就是持有lru链表mutex，进行第一步，第二步操作。*/
 for (i = 0; i < srv_buf_pool_instances; i++) {
 mutex_enter(&buf_pool->LRU_list_mutex);
 while (bpage != NULL) {
 if (buf_page_get_space(bpage) == id)
 bpage->space_was_being_deleted = TRUE;
 }
 mutex_exit(&buf_pool->LRU_list_mutex);

/* 这里扫描的是buf_pool中的chunk，也就是启动的时候，根据buffer pool的大小预分配好的blocks，不能更改，
 所以并不需要持有buffer pool mutex，或者lru list mutex。
*/
 btr_search_s_lock_all();
 chunk = buf_pool->chunks;
 for (j = buf_pool->n_chunks; j--; chunk++) {
 buf_block_t* block = chunk->blocks;
 for (k = chunk->size; k--; block++) {
 if (buf_block_get_state(block)
 != BUF_BLOCK_FILE_PAGE
 || !block->index
 || buf_page_get_space(&block->page) != id) {
 continue;
 }
/* 这里把AHI的锁释放掉了，但在btr_search_drop_page_hash_index中会持有AHI的lock对AHI结构进行变更。*/
 btr_search_s_unlock_all();
 rw_lock_x_lock(&block->lock);
 btr_search_drop_page_hash_index(block, NULL);
 rw_lock_x_unlock(&block->lock);

 btr_search_s_lock_all();
 }
 }
 btr_search_s_unlock_all();
 }
}
`

## MySQL lazy模式

在MySQL 5.5.23以后的版本，也实现了一个`lazy drop table`的方式，和percona的方式有所区别，下面来看一下具体的过程：

1. 持有`buffer pool mutex`；
2. 持有buffer pool中的`flush list mutex`；
3. 开始扫描flush list；
 
 如果dirty page属于drop table，那么就直接从flush list中remove掉；
4. 如果删除的page个数超过了`#define BUF_LRU_DROP_SEARCH_SIZE 1024` 这个数目的话，释放`buffer pool mutex`，`flush list mutex`，释放cpu资源；
 * 释放`flush list mutex`；
* 释放`buffer pool mutex`；
* 强制通过pthread_yield进行一次OS context switch，释放剩余的cpu时间片；
5. 重新持有`buffer pool mutex`；
6. 重新持有`flush list mutext`；

 释放`flush list mutex`；
 释放`buffer pool mutex`；

相比较percona的lazy方式，这里扫描的是dirty block，在LRU list中进行淘汰的时候，就不再判断当前fil_space是否存在的问题了，因为不牵涉到写入。

这里边有两个相关的bug，[bug#51325](http://bugs.mysql.com/bug.php?id=51325)、[bug#64284](http://bugs.mysql.com/bug.php?id=64284)，有兴趣可以参考一下。

其核心的代码如下：

`buf_LRU_flush_or_remove_pages(id, BUF_REMOVE_FLUSH_NO_WRITE, 0);

buf_pool_mutex_enter(buf_pool);

err = buf_flush_or_remove_pages(buf_pool, id, flush, trx);
......
buf_pool_mutex_exit(buf_pool);

/* BUF_REMOVE_FLUSH_NO_WRITE：意思表示，只对dirty block进行remove操作，不做写入。
`

## 对比

从上面的percona和oracle的MySQL版本比较来看，percona是持有了`LRU list mutex`和`AHI lock`，而MySQL官方版本是持有了`buffer pool mutex`和`flush list mutex`，从锁的保护范围来看，`buffer pool mutex`直观上瓶颈会比较明显，但具体还要跟表的大小、dirty block的比例来看，如果dirty block比较少的话，官方版本并不扫描LRU list，所以可能持有的时间并不会太久。

Percona的开发人员还针对这两个不同版本进行了Benchmarks， 大家可以看下他们测试出来的结果:

这个图是 MySQL 官方版本测试在系统压力下，进行频繁drop table的系统抖动:

这个图是 Percona 版本测试在系统压力下，进行频繁drop table的系统抖动:

但对于这样的测试，小编想说，哪个DBA/开发人员这么变态，要这么频繁的drop table -_-||

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)