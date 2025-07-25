# MySQL · 特性分析 · innodb_buffer_pool_size在线修改

**Date:** 2018/03
**Source:** http://mysql.taobao.org/monthly/2018/03/06/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 03
 ](/monthly/2018/03)

 * 当期文章

 MySQL · 源码分析 · InnoDB的read view，回滚段和purge过程简介
* MySQL · 源码分析 · 原子DDL的实现过程
* MongoDB · Feature · In-place update in MongoDB
* MSSQL · 最佳实践 · 利用文件组实现冷热数据隔离备份方案
* PgSQL · 内核优化 · Hybrid DB for PG 赋能向量化执行和查询子树封装
* MySQL · 特性分析 · innodb_buffer_pool_size在线修改
* MySQL · myrocks · 事务锁分析
* PgSQL · 特性分析 · 事务ID回卷问题
* MariaDB · 源码分析 · thread pool
* PgSQL · 应用案例 · 毫秒级文本相似搜索实践一

 ## MySQL · 特性分析 · innodb_buffer_pool_size在线修改 
 Author: 勉仁 

 InnoDB Buffer Pool缓存了表数据和二级索引在内存中，提高数据库效率，因此设置innodb_buffer_pool_size到合理数值对实例性能影响很大。当size设置偏小，会导致数据库大量直接磁盘的访问，而设置过大会导致实例占用内存太多，容易发生OOM。在MySQL 5.7之前innodb_buffer_pool_size的修改需要重启实例，在5.7后支持了动态修改innodb_buffer_pool_size。本文会根据源码介绍该特性。

## innodb_buffer_pool_size 设置范围

innodb_buffer_pool_size默认值是128M，最小5M(当小于该值时会设置成5M)，最大为LLONG_MAX。当innodb_buffer_pool_instances设置大于1的时候，buffer pool size最小为1GB。同时buffer pool size需要是innodb_buffer_pool_chunk_size*innodb_buffer_pool_instances的倍数。innodb_buffer_pool_chunk_size默认为128M，最小为1M，实例启动后为只读参数。

`static MYSQL_SYSVAR_LONGLONG(buffer_pool_size, innobase_buffer_pool_size,
 PLUGIN_VAR_RQCMDARG,
 "The size of the memory buffer InnoDB uses to cache data and indexes of its tables.",
 innodb_buffer_pool_size_validate,
 innodb_buffer_pool_size_update,
 static_cast<longlong>(srv_buf_pool_def_size),//128M
 static_cast<longlong>(srv_buf_pool_min_size),//5M
 LLONG_MAX, 1024*1024L);

#define BUF_POOL_SIZE_THRESHOLD (1024 * 1024 * 1024) //1GB

static
int
innodb_buffer_pool_size_validate(
{
 ...
 //当srv_buf_pool_instances > 1，要求size不小于1GB。
 if (srv_buf_pool_instances > 1 && intbuf < BUF_POOL_SIZE_THRESHOLD) {
 buf_pool_mutex_exit_all();

 push_warning_printf(thd, Sql_condition::SL_WARNING,
 ER_WRONG_ARGUMENTS,
 "Cannot update innodb_buffer_pool_size"
 " to less than 1GB if"
 " innodb_buffer_pool_instances > 1.");
 return(1);
 }
 ...
 ulint requested_buf_pool_size
 = buf_pool_size_align(static_cast<ulint>(intbuf));
}

/** Calculate aligned buffer pool size based on srv_buf_pool_chunk_unit,
if needed.
@param[in] size size in bytes
@return aligned size */
UNIV_INLINE
ulint
buf_pool_size_align(
 ulint size)
{
 const ulint m = srv_buf_pool_instances * srv_buf_pool_chunk_unit;
 size = ut_max(size, srv_buf_pool_min_size);

 if (size % m == 0) {
 return(size);
 } else {
 return((size / m + 1) * m);
 }
}
`

## buffer pool resize流程

* 如果开启了AHI（adaptive hash index，自适应哈希索引）就关闭AHI，这里因为AHI是通过buffer pool中的B+树页构造而来。
* 如果新设定的buffer pool size小于原来的size，就需要计算需要删除的chunk数目withdraw_target。
* 遍历buffer pool instances，锁住buffer pool，收集free list中的chunk page到withdraw，直到withdraw_target或者遍历完，然后释放buffer pool锁。
* 停止加载buffer pool。
* 如果free list中没有收集到足够的chunk，则重复遍历收集，每次重复间隔时间会指数增加1s、2s、4s、8s…，以等待事务释放资源。
* 锁住buffer pool，开始增减chunk。
* 如果改变比较大，超过2倍，会重置page hash，改变桶大小。
* 释放buffer_pool,page_hash锁。
* 改变比较大时候，重新设置buffer pool大小相关的内存结构。
* 开启AHI。

`/** Resize the buffer pool based on srv_buf_pool_size from
srv_buf_pool_old_size. */
void
buf_pool_resize()
{
 /* disable AHI if needed */
 btr_search_disable(true);

 /* set withdraw target */
 for (ulint i = 0; i < srv_buf_pool_instances; i++) {
 if (buf_pool->curr_size < buf_pool->old_size) {
 ...
 while (chunk < echunk) {
 withdraw_target += chunk->size;
 ++chunk;
 }
 ...
 }
 }

 /* wait for the number of blocks fit to the new size (if needed)*/
 for (ulint i = 0; i < srv_buf_pool_instances; i++) {
 buf_pool = buf_pool_from_array(i);
 if (buf_pool->curr_size < buf_pool->old_size) {

 should_retry_withdraw |=
 buf_pool_withdraw_blocks(buf_pool);
 }
 }
 ...
 if (should_retry_withdraw) {
 ib::info() << "Will retry to withdraw " << retry_interval
 << " seconds later.";
 os_thread_sleep(retry_interval * 1000000);

 if (retry_interval > 5) {
 retry_interval = 10;
 } else {
 retry_interval *= 2;
 }

 goto withdraw_retry;
 }
 ...
 /* add/delete chunks */
 for (ulint i = 0; i < srv_buf_pool_instances; ++i) {
 if (buf_pool->n_chunks_new < buf_pool->n_chunks) {
 /* delete chunks */
 /* discard withdraw list */
 }
 }
 /* reallocate buf_pool->chunks */
 if (buf_pool->n_chunks_new > buf_pool->n_chunks) {
 /* add chunks */
 }
 ...
 const bool new_size_too_diff
 = srv_buf_pool_base_size > srv_buf_pool_size * 2
 || srv_buf_pool_base_size * 2 < srv_buf_pool_size;

 /* Normalize page_hash and zip_hash,
 if the new size is too different */
}
`

## resize过程中的等待和阻塞

在支持动态修改innodb_buffer_pool_size之前，该值的修改需要修改配置项然后重启实例生效。而重启实例会导致用户连接强制断开，导致一段时间的实例不可用，如果有大事务在回滚就需要等待很长时间。

动态修改innodb_buffer_pool_size只有在收集回收块；查找持有block阻止buffer pool收集回收chunk的事务；resizing buffer pool操作时会阻塞用户写入。而这几部分操作都是内存操作，会较快完成。

如果对innodb_buffer_pool_size修改量很大，同时遇到page cleaner工作时间久，就可能导致一段时间的阻塞。例如下面一个较为极端的例子，innodb_buffer_pool_instances为1，innodb_buffer_pool_size由18GB改为5M，innodb_buffer_pool_chunk_size为1M，page cleaner loop花费近48s，导致收集回收块会花费很长时间，可以看到在测试机器上用时近48s。而这期间的写入操作也会被阻塞。

`02:54:09.798912Z 0 [Note] InnoDB: Withdrawing blocks to be shrunken.
02:54:09.798935Z 0 [Note] InnoDB: buffer pool 0 : start to withdraw the last 1151680 blocks.
02:54:57.660725Z 0 [Note] InnoDB: page_cleaner: 1000ms intended loop took 47685ms. The settings might not be optimal. (flushed=0 and evicted=0, during the time.)
02:54:57.687189Z 0 [Note] InnoDB: buffer pool 0 : withdrawing blocks. (1151680/1151680)
02:54:57.687237Z 0 [Note] InnoDB: buffer pool 0 : withdrew 1151653 blocks from free list. Tried to relocate 27 pages (1151680/1151680).
02:54:57.753014Z 0 [Note] InnoDB: buffer pool 0 : withdrawn target 1151680 blocks.

> insert into t values(10000001, 2);
Query OK, 1 row affected (9.03 sec)
`

正常不需要等待时的内存操作会很快。

`03:31:57.734231Z 0 [Note] InnoDB: Resizing buffer pool from 1887436800 to 5242880 (unit=1048576).
03:31:58.480061Z 0 [Note] InnoDB: Completed to resize buffer pool from 1887436800 to 5242880.
...
03:31:46.453250Z 10 [Note] InnoDB: Resizing buffer pool from 524288 (new size: 1887436800 bytes)
03:31:57.734231Z 0 [Note] InnoDB: Resizing buffer pool from 1887436800 to 5242880 (unit=1048576).
`

另一个方面，如果当前有事务占用大量buffer pool数据导致无法收集到足够的chunk，resize过程也会变久。下面极端测试中当执行xa rollback回滚大事务的时候，innodb_buffer_pool_chunk_size由16M改为5M，即等待了较久时间才完成回收chunk的收集。不过这段时间并不会完全阻塞用户的操作。

`> xa begin 'y';
Query OK, 0 rows affected (0.00 sec)

> update t set c2 = 2 where c2 =1;
Query OK, 999996 rows affected (8.32 sec)
Rows matched: 999996 Changed: 999996 Warnings: 0

> xa end 'y';
Query OK, 0 rows affected (0.00 sec)

> xa prepare 'y';
Query OK, 0 rows affected (0.11 sec)

> xa rollback 'y';
Query OK, 0 rows affected (10.10 sec)

InnoDB: Resizing buffer pool from 16777216 to 5242880 (unit=1048576).
InnoDB: Withdrawing blocks to be shrunken.
InnoDB: buffer pool 0 : start to withdraw the last 704 blocks.
InnoDB: buffer pool 0 : withdrew 239 blocks from free list. Tried to relocate 126 pages (689/704).
InnoDB: buffer pool 0 : withdrew 0 blocks from free list. Tried to relocate 0 pages (689/704).
...
InnoDB: Will retry to withdraw 1 seconds later.
InnoDB: buffer pool 0 : start to withdraw the last 704 blocks.
...
InnoDB: buffer pool 0 : will retry to withdraw later.
InnoDB: Will retry to withdraw 2 seconds later.
...
InnoDB: buffer pool 0 : will retry to withdraw later.
InnoDB: Will retry to withdraw 4 seconds later.
InnoDB: buffer pool 0 : start to withdraw the last 704 blocks.
...
InnoDB: Will retry to withdraw 8 seconds later.
InnoDB: buffer pool 0 : withdrawn target 704 blocks.

`

从上面可以看到innodb_buffer_pool_size的online修改相比重启对用户实例的影响降低了很多，但也最好选择业务低峰期和没有大事务操作时候进行，同时要修改MySQL配置文件，防止重启后恢复到原来的值。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)