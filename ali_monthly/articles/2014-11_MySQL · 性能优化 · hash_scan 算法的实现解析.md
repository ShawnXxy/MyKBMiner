# MySQL · 性能优化 · hash_scan 算法的实现解析

**Date:** 2014/11
**Source:** http://mysql.taobao.org/monthly/2014/11/07/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2014 / 11
 ](/monthly/2014/11)

 * 当期文章

 MySQL · 捉虫动态 · OPTIMIZE 不存在的表
* MySQL · 捉虫动态 · SIGHUP 导致 binlog 写错
* MySQL · 5.7改进 · Recovery改进
* MySQL · 5.7特性 · 高可用支持
* MySQL · 5.7优化 · Metadata Lock子系统的优化
* MySQL · 5.7特性 · 在线Truncate undo log 表空间
* MySQL · 性能优化 · hash_scan 算法的实现解析
* TokuDB · 版本优化 · 7.5.0
* TokuDB · 引擎特性 · FAST UPDATES
* MariaDB · 性能优化 · filesort with small LIMIT optimization

 ## MySQL · 性能优化 · hash_scan 算法的实现解析 
 Author: 

 **问题描述**

首先，我们执行下面的TestCase：

`--source include/master-slave.inc
--source include/have_binlog_format_row.inc
connection slave;
set global slave_rows_search_algorithms='TABLE_SCAN';
connection master;
create table t1(id int, name varchar(20);
insert into t1 values(1,'a');
insert into t2 values(2, 'b');
......
insert into t3 values(1000, 'xxx');
delete from t1;
---source include/rpl_end.inc
`
随着 t1 数据量的增大，rpl_hash_scan.test 的执行时间会随着 t1 数据量的增大而快速的增长，因为在执行 ‘delete from t1;’ 对于t1的每一行删除操作，备库都要扫描t1,即全表扫描，如果 select count(*) from t1 = N, 则需要扫描Ｎ次 t1 表， 则读取记录数为： O(N + (N-1) + (N-2) + …. + 1) = O(N^2)，在 replication 没有引入 hash_scan，binlog_format=row时，对于无索引表，是通过 table_scan 实现的，如果一个update_rows_log_event/delete_rows_log_event 包含多行修改时，每个修改都要进行全表扫描来实现，其 stack 如下：

`#0 Rows_log_event::do_table_scan_and_update
#1 0x0000000000a3d7f7 in Rows_log_event::do_apply_event 
#2 0x0000000000a28e3a in Log_event::apply_event
#3 0x0000000000a8365f in apply_event_and_update_pos
#4 0x0000000000a84764 in exec_relay_log_event 
#5 0x0000000000a89e97 in handle_slave_sql (arg=0x1b3e030) 
#6 0x0000000000e341c3 in pfs_spawn_thread (arg=0x2b7f48004b20) 
#7 0x0000003a00a07851 in start_thread () from /lib64/libpthread.so.0
#8 0x0000003a006e767d in clone () from /lib64/libc.so.6
`
这种情况下，往往会造成备库延迟，这也是无索引表所带来的复制延迟问题。

如何解决问题：

RDS 为了解这个问题，会在每个表创建的时候检查一下表是否包含主建或者唯一建，如果没有包含，则创建一个隐式主建，此主建对用户透明，用户无感，相应的show create, select * 等操作会屏蔽隐式主建，从而可以减少无索引表带来的影响;
官方为了解决这个问题，在5.6.6 及以后版本引入参数 slave_rows_search_algorithms ，用于指示备库在 apply_binlog_event时使用的算法，有三种算法TABLE_SCAN,INDEX_SCAN,HASH_SCAN，其中table_scan与index_scan是已经存在的，本文主要研究HASH_SCAN的实现方式，关于参数slave_rows_search_algorithms的设置，详情请参考：http://dev.mysql.com/doc/refman/5.6/en/replication-options-slave.html#option_mysqld_slave-rows-search-algorithms
hash_scan 的实现方法：

简单的讲，在 apply rows_log_event时，会将 log_event 中对行的更新缓存在两个结构中，分别是：m_hash, m_distinct_key_list。 m_hash：主要用来缓存更新的行记录的起始位置，是一个hash表； m_distinct_key_list：如果有索引，则将索引的值push 到m_distinct_key_list，如果表没有索引，则不使用这个List结构； 其中预扫描整个调用过程如下： Log_event::apply_event

`Rows_log_event::do_apply_event
Rows_log_event::do_hash_scan_and_update 
Rows_log_event::do_hash_row (add entry info of changed records)
if (m_key_index < MAX_KEY) (index used instead of table scan)
Rows_log_event::add_key_to_distinct_keyset ()
`
```
当一个event 中包含多个行的更改时，会首先扫描所有的更改，将结果缓存到m_hash中，如果该表有索引，则将索引的值缓存至m_distinct_key_list List 中，如果没有，则不使用这个缓存结构，而直接进行全表扫描；

```

执行 stack 如下：

`#0 handler::ha_delete_row 
#1 0x0000000000a4192b in Delete_rows_log_event::do_exec_row 
#2 0x0000000000a3a9c8 in Rows_log_event::do_apply_row
#3 0x0000000000a3c1f4 in Rows_log_event::do_scan_and_update 
#4 0x0000000000a3c5ef in Rows_log_event::do_hash_scan_and_update 
#5 0x0000000000a3d7f7 in Rows_log_event::do_apply_event 
#6 0x0000000000a28e3a in Log_event::apply_event
#7 0x0000000000a8365f in apply_event_and_update_pos
#8 0x0000000000a84764 in exec_relay_log_event 
#9 0x0000000000a89e97 in handle_slave_sql
#10 0x0000000000e341c3 in pfs_spawn_thread
#11 0x0000003a00a07851 in start_thread () 
#12 0x0000003a006e767d in clone () 
`
执行过程说明：

`Rows_log_event::do_scan_and_update

open_record_scan()
do
next_record_scan()
if (m_key_index > MAX_KEY)
ha_rnd_next();
else
ha_index_read_map(m_key from m_distinct_key_list) 
entry= m_hash->get()
m_hash->del(entry);
do_apply_row()
while (m_hash->size > 0);
`
从执行过程上可以看出，当使用hash_scan时，只会全表扫描一次，虽然会多次遍历m_hash这个hash表，但是这个扫描是O(1),所以，代价很小，因此可以降低扫描次数，提高执行效率。

hash_scan 的一个 bug

bug详情：[http://bugs.mysql.com/bug.php?id=72788](http://bugs.mysql.com/bug.php?id=72788)

bug原因：m_distinct_key_list 中的index key 不是唯一的，所以存在着对已经删除了的记录重复删除的问题。

bug修复：[http://bazaar.launchpad.net/~mysql/mysql-server/5.7/revision/8494](http://bazaar.launchpad.net/~mysql/mysql-server/5.7/revision/8494)

问题扩展：

在没有索引的情况下，是不是把 hash_scan 打开就能提高效率，降低延迟呢？
不一定，如果每次更新操作只一条记录，此时仍然需要全表扫描，并且由于entry 的开销，应该会有后退的情况；

一个event中能包含多少条记录的更新呢？
这个和表结构以及记录的数据大小有关，一个event 的大小不会超过9000 bytes, 没有参数可以控制这个size；

hash_scan 有没有限制呢？
hash_scan 只会对更新、删除操作有效，对于binlog_format=statement 产生的 Query_log_event 或者binlog_format=row 时产生的 Write_rows_log_event 不起作用；

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)