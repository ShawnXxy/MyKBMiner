# MySQL · 备库优化 ·  relay fetch 备库优化

**Date:** 2015/09
**Source:** http://mysql.taobao.org/monthly/2015/09/08/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 09
 ](/monthly/2015/09)

 * 当期文章

 MySQL · 引擎特性 · InnoDB Adaptive hash index介绍
* PgSQL · 特性分析 · clog异步提交一致性、原子操作与fsync
* MySQL · 捉虫动态 · BUG 几例
* PgSQL · 答疑解惑 · 诡异的函数返回值
* MySQL · 捉虫动态 · 建表过程中crash造成重建表失败
* PgSQL · 特性分析 · 谈谈checkpoint的调度
* MySQL · 特性分析 · 5.6 并行复制恢复实现
* MySQL · 备库优化 · relay fetch 备库优化
* MySQL · 特性分析 · 5.6并行复制事件分发机制
* MySQL · TokuDB · 文件目录谈

 ## MySQL · 备库优化 · relay fetch 备库优化 
 Author: 沽月 

 ## 业务背景

MySQL 主备通过 binlog 实现数据同步的功能，主库将生成的 binlog 通过 binlog send 线程发送到备库，备库通过应用这些 binlog 来更新数据，实现主备数据一致，其应用 binlog 的读取操作与更新操作的堆栈分别如下。

读取操作：

`#0 row_search_for_mysql
#1 0x0000000000c200c2 in ha_innobase::index_read
#2 0x0000000000c21c57 in ha_innobase::rnd_pos
#3 0x000000000090c5d3 in handler::rnd_pos_by_record
#4 0x0000000000a574c3 in Rows_log_event::find_row
#5 0x0000000000a589da in Delete_rows_log_event::do_exec_row
#6 0x0000000000a50dcc in Rows_log_event::do_apply_event
#7 0x00000000005d0bb8 in Log_event::apply_event
#8 0x00000000005b9782 in apply_event_and_update_pos
...
`

更新操作：

`#0 row_update_for_mysql
#1 0x0000000000c1f466 in ha_innobase::delete_row
#2 0x000000000090b64a in handler::ha_delete_row
#3 0x0000000000a58a4b in Delete_rows_log_event::do_exec_row
#4 0x0000000000a50dcc in Rows_log_event::do_apply_event
#5 0x00000000005d0bb8 in Log_event::apply_event
#6 0x00000000005b9782 in apply_event_and_update_pos
...
`

* 由堆栈可以看出，sql 线程首先将数据从磁盘加载到内存，然后调用引擎层的接口执行相应的操作，当iops 及 buffer pool 较小时，读磁盘需要较多的时间，容易造成主备延迟问题；
* 当系统重启后，需要对系统进行预热，提高 buffer pool 的命中率，因此需要提供有效的方法来对系统进行预热；

综上，我们需要一种可以在 DML 操作之前将数据从磁盘加载到内存的功能，以实现数据库的快速操作。

## 解决方法

我们需要找到一种将数据加载到内存的方法，但又不对数据进行修改，需要满足以下的条件：

* 在库上更新的数据应该在备库操作之前被加载到内存中；
* 对于重启的mysqld实例，应该将启动之前所用的数据页加载到内存中；
* 加载操作对数据本身不进行修改，类似于select 语句。

因此，我们可以在mysqld启动时启动额外的线程对 relay log 进行特殊处理，以达到数据加载的目的。

## 设计思路 & 使用方法

RDS MySQL 利用 relay log 来解决上述两个问题，当系统启动后，可以在后台开启一个独立于SQL thread之外的线程将 relay log 相关的数据从磁盘加载到内存中，从而使备库在查找数据的时候直接利用buffer pool，而不需要从磁盘中进行加载，同理，使用这种方法也可以解决系统预热的问题。

当启动后，如果发现延迟且 buffer pool 命中率较低时，可以启用 relay fetch thread, 具体语法为：

`启动 relay_fetch_thread: start slave relay_fetch_thread;
停止 relay_fetch_thread: stop slave relay_fetch_thread;
`

relay fetch thread 读取relay log, 并将要执行的数据从磁盘上加载到内存中，所以只能对包含数据部分的 log_event 进行操作，对 Query_log_event，Write_rows_log_event 是无法进行预读的，前者是因为Query_log_event 只是SQL语句，不包含具体的数据信息；后者则是event中没有的数据，所以不需要进行加载，另外为了防止 buffer pool 中读取的 page 被 evict 出去，我们需要对两种情况进行分别处理：

1. relay fetch thread 不能领先 sql thread 过多，如果领先过多的 relay log files，当 buffer pool 较小时，新加载进来的数据页会将老的数据页从内存中 evict 出去，对 sql thread 的命中率会有直接的影响；
2. 当 sql thread 领先 relay fetch thread 时，此时 relay fetch thread 不需要将已执行完的 relay log 加载到内存，继续加载不仅会有命中率的问题，同时会造成 CPU 不必要的资源浪费。

因此，relay fetch thread 与 sql thread 应该相差的距离不太远，我们的策略是 relay fetch thread 与 sql thread 应该在同一个 relay log 上，具体策略如下：

1. 如果 relay fetch thread 领先, 则当 relay fetch thread 读完一个文件后要等待 sql thread，直到 sql thread 应用完此relay log 再继续加载；
2. 如果 sql thread 领先，则会通知 relay fetch thread 跳过当前执行的文件并用 sql thread 的位点来初始化自己将要执行的起点；

relay fetch thread 执行过程的伪码如下：

```
handle_slave_relay_fetch
{
 init_thd_and_rli();
 while (!relay_fetch_killed(eli))
 {
 ev= Log_event::read_log_event(&rli->relay_log_buf, 0, rli->relay_log.description_event_for_relay_fetch);
 if (ev == NULL) 
 { 
 deal with situations like hot_log, relay log purged, eof of relay log etc.
 }
 else
 {
 switch(ev->get_type_code())
 {
 case QUERY_EVENT:
 deal with begin, commit 
 break;

 case XID_EVENT:
 deal with xid(commit)
 break;

 case TABLE_MAP_EVENT:
 init table info for rows log event
 break;

 case UPDATE_ROWS_EVENT:
 case DELETE_ROWS_EVENT:
 find_row();
 break;

 case FORMAT_DESCRIPTION_EVENT:
 init description_event_for_relay_fetch for reading binlog event;
 default:
 break;
 }
 delete ev;
 }
 }
}

```

## 实现过程中注意的细节

* 由于 relay fetch thread 在加载数据的过程中会对记录进行加锁，所以在遇到begin, commit 的事件时，需要释放在读取过程中获取的所有锁资源，否则有可能会引起 sql 线程锁超时错误；
* 由于 relay fetch thread 的位点是使用 sql thread 的位点进行初始化的，所以需要处理 relay log 不是完整事务的情况；
* 释放 relay fetch thread 在执行过程中使用到的内存，否则会有内存问题；
* 在 relay fetch thread 执行的过程中需要特别注意 log_lock、run_lock 等锁问题，以避免备库的死锁；
* 需要对 relay log 的purge进行特殊处理；
* 如果是系统预热的功能，则需要对 relay fetch thread 与 sql thread 的领先策略进行调整。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)