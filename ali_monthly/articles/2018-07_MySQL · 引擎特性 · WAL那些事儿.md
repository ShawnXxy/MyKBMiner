# MySQL · 引擎特性 · WAL那些事儿

**Date:** 2018/07
**Source:** http://mysql.taobao.org/monthly/2018/07/01/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 07
 ](/monthly/2018/07)

 * 当期文章

 MySQL · 引擎特性 · WAL那些事儿
* MySQL · 源码分析 · 8.0 原子DDL的实现过程续
* MongoDB · 引擎特性 · 事务实现解析
* MySQL · RocksDB · 写入逻辑的实现
* MySQL · 源码分析 · binlog crash recovery
* PgSQL · 新特征 · PG11并行Hash Join介绍
* MySQL · myrocks · clustered index特性
* MSSQL · 最佳实践 · 实例级别数据库上云RDS SQL Server
* MySQL · 最佳实践 · 一个TPC-C测试工具sqlbench使用
* PgSQL · 应用案例 · PostgreSQL flashback(闪回) 功能实现与介绍

 ## MySQL · 引擎特性 · WAL那些事儿 
 Author: 韩逸 

 ## 前言
日志先行的技术广泛应用于现代数据库中，其保证了数据库在数据不丢的情况下，进一步提高了数据库的性能。本文主要分析了WAL模块在MySQL各个版本中的演进以及在阿里云新一代数据库POLARDB中的改进。

## 基础知识
用户如果对数据库中的数据进行了修改，必须保证日志先于数据落盘。当日志落盘后，就可以给用户返回操作成功，并不需要保证当时对数据的修改也落盘。如果数据库在日志落盘前crash，那么相应的数据修改会回滚。在日志落盘后crash，会保证相应的修改不丢失。有一点要注意，虽然日志落盘后，就可以给用户返回操作成功，但是由于落盘和返回成功包之间有一个微小的时间差，所以即使用户没有收到成功消息，修改也可能已经成功了，这个时候就需要用户在数据库恢复后，通过再次查询来确定当前的状态。

在日志先行技术之前，数据库只需要把修改的数据刷回磁盘即可，用了这项技术，除了修改的数据，还需要多写一份日志，也就是磁盘写入量反而增大，但是由于日志是顺序的且往往先存在内存里然后批量往磁盘刷新，相比数据的离散写入，日志的写入开销比较小。

日志先行技术有两个问题需要工程上解决：

1. 日志刷盘问题。由于所有对数据的修改都需要写日志，当并发量很大的时候，必然会导致日志的写入量也很大，为了性能考虑，往往需要先写到一个日志缓冲区，然后再按照一定规则刷入磁盘，此外日志缓冲区大小有限，而用户会源源不断的生产日志，数据库需要不断的把缓存区中的日志刷入磁盘，缓存区才可以复用，由此可见，这里构成了一个典型的生产者和消费者模型。现代数据库必须直面这个问题，在高并发的情况下，这一定是个性能瓶颈，也一定是个锁冲突的热点。
2. 数据刷盘问题。在用户收到操作成功的时候，用户的数据不一定已经被持久化了，很有可能修改还没有落盘，这就需要数据库有一套刷数据的机制，专业术语叫做刷脏页算法。脏页(内存中被修改的但是还没落盘的数据页)在源源不断的产生，然后要持续的刷入磁盘，这里又凑成一个生产者消费者模型，影响数据库的性能。如果在脏页没被刷入磁盘，但是数据库异常crash了，这个就需要做奔溃恢复，具体的流程是，在接受用户请求之前，从checkpoint点(这个点之前的日志对应的数据页一定已经持久化到磁盘了)开始扫描日志，然后应用日志，从而把在内存中丢失的更新找回来，最后重新刷入磁盘。这里有一个很重要的点：在数据库正常启动的期间，checkpoint怎么确定，如果checkpoint做的慢了，就会导致奔溃恢复时间过长，从而影响数据库可用性，如果做的快了，会导致刷脏压力过大，甚至数据丢失。

MySQL中为了解决上述两个问题，采用了以下机制:

1. 当用户线程产生日志的时候，首先缓存在一个线程私有的变量(mtr)里面，只有完成某些原子操作(例如完成索引分裂或者合并等)的时候，才把日志提交到全局的日志缓存区中。全局缓存区的大小(innodb_log_file_size)可以动态配置。当线程的事务执行完后，会按照当前的配置(innodb_flush_log_at_trx_commit)决定是否需要把日志从缓冲区刷到磁盘。
2. 当把日志成功拷贝到全局日志缓冲区后，会继续把当前已经被修改过的脏页加入到一个全局的脏页链表中。这个链表有一个特性：按照最早被修改的时间排序。例如，有数据页A,B,C，数据页A早上9点被第一次修改，数据页B早上9点01分被第一次修改，数据页C早上9点02分被第一次修改，那么在这个链表上数据页A在最前，B在中间，C在最后。即使数据页A在早上9点之后又一次被修改了，他依然排在B和C之前。在数据页上，有一个字段来记录这个最早被修改的时间：oldest_modification，只不过单位不是时间，而是lsn，即从数据库初始化开始，一共写了多少个字节的日志，由于其是一个递增的值，因此可以理解为广义的时间，先写的数据，其产生的日志对应的lsn一定比后写的小。在脏页列表上的数据页，就是按照oldest_modification从小到大排序，刷脏页的时候，就从oldest_modification小的地方开始。checkpoint就是脏页列表中最小的那个oldest_modification，因为这种机制保证小于最小oldest_modification的修改都已经刷入磁盘了。这里最重要的是，脏页链表的有序性，假设这个有序性被打破了，如果数据库异常crash，就会导致数据丢失。例如，数据页ABC的oldest_modification分别为120，100，150，同时在脏页链表上的顺序依然为A，B，C，A在最前面，C在最后面。数据页A被刷入磁盘，然后checkpoint被更新为120，但是数据页B和C都还没被刷入磁盘，这个时候，数据库crash，重启后，从checkpoint为120开始扫描日志，然后恢复数据，我们会发现，数据页C的修改被恢复了，但是数据页B的修改丢失了。

在第一点中的，我们提到了私有变量mtr，这个结构除了存储修改产生的日志和脏页外，还存储了修改脏页时加的锁。在适当的时候(例如日志提交完且脏页加入到脏页链表)可以把锁给释放。

接下来，我们结合各个版本的实现，来剖析一下具体实现细节。注意，以下内容需要一点MySQL源码基础，适合MySQL内核开发者以及资深的DBA。

## MySQL 5.1版本的处理方式
5.1的版本是MySQL比较早的版本，那个时候InnoDB还是一个插件。因此设计也相对粗糙，简化后的伪代码如下：

日志进入全局缓存:

`mutex_enter(log_sys->mutex);
copy local redo log to global log buffer
mtr.start_lsn = log_sys->lsn
mtr.end_lsn = log_sys->lsn + log_len + log_block_head_or_tail_len
increase global lsn: log_sys->lsn, log_sys->buf_free
for every lock in mtr
 if (lock == share lock)
 release share lock directly
 else if (lock == exclusive lock)
 if (lock page is dirty)
 if (page.oldest_modification == 0) //This means this page is not in flush list
 page.oldest_modification = mtr.start_lsn
 add to flush list // have one flush list only
 release exclusive lock
mutex_exit(log_sys->mutex);
`

日志写入磁盘：

`mutex_enter(log_sys->mutex);
log_sys->write_lsn = log_sys->lsn;
write log to log file
mutex_exit(log_sys->mutex);
`

更新checkpoint:

`page = get_first_page(flush_list)
checkpoint_lsn = page.oldest_modification
write checkpoint_lsn to log file
`

奔溃恢复：

`read checkpoint_lsn from log file
start parse and apply redo log from checkpoint_lsn point
`

从上述伪代码中可以看出，由于日志进入全局的缓存都在临界区内，不但保证了拷贝日志的完整性，也保证了脏页进入脏页链表的有序性。需要获取checkpoint_lsn时，只需从脏页链表中获取第一个数据页的oldest_modification即可。奔溃恢复也只需要从记录的checkpoint点开始扫描即可。在高并发的场景下，有很多线程需要把自己的local日志拷贝到全局缓存，会造成锁热点，另外在全局日志写入日志文件的地方，也需要加锁，进一步造成了锁的争抢。此外，每个数据库的缓存(Buffer Pool)只有一个脏页链表，性能也不高。这种方式存在于早期的InnoDB代码中，通俗易懂，但在现在的多核系统上，显然不能做到很好的扩展性。

## MySQL 5.5，5.6，5.7版本的处理方式
这三个版本是目前主流的MySQL版本，很多分支都在上面做了不少优化，但是主要的处理逻辑变化依然不大：

日志进入全局缓存:

`mutex_enter(log_sys->mutex);
copy local redo log to global log buffer
mtr.start_lsn = log_sys->lsn
mtr.end_lsn = log_sys->lsn + log_len + log_block_head_or_tail_len
increase global lsn: log_sys->lsn, log_sys->buf_free
mutex_enter(log_sys->log_flush_order_mutex);
mutex_exit(log_sys->mutex);
for every page in mtr
 if (lock == exclusive lock)
 if (page is dirty)
 if (page.oldest_modification == 0) //This means this page is not in flush list
 page.oldest_modification = mtr.start_lsn
 add to flush list according to its buffer pool instance
mutex_exit(log_sys->log_flush_order_mutex);
for every lock in mtr
 release all lock directly
`

日志写入磁盘：

`mutex_enter(log_sys->mutex);
log_sys->write_lsn = log_sys->lsn;
write log to log file
mutex_exit(log_sys->mutex);
`

更新checkpoint:

`for ervery flush list:
 page = get_first_page(curr_flush_list);
 if current_oldest_modification > page.oldest_modification
 current_oldest_modification = page.oldest_modification
checkpoint_lsn = current_oldest_modification
write checkpoint_lsn to log file
`

奔溃恢复：

`read checkpoint_lsn from log file
start parse and apply redo log from checkpoint_lsn point
`

主流的版本中最重要的一个优化是，除了log_sys->mutex外，引入了另外一把锁log_sys->log_flush_order_mutex。在脏页加入到脏页链表的操作中，不需要log_sys->mutex保护，而是需要log_sys->log_flush_order_mutex保护，这样减少了log_sys->mutex的临界区，从而减少了热点。此外，引入多个脏页链表，减少了单个链表带来的冲突。

注意，主流的分支还做了很多其他的优化，例如：

1. 引入双全局日志缓存。如果只有一个全局日志缓存，当这个日志缓存在写盘的时候，会导致后续的用户线程无法往里面拷贝日志，直到刷盘结束。有了双日志缓存，其中一个用来接收用户提交过来的日志，另外一个可以用来把之前的日志刷盘，这样用户线程不需要等待。
2. 日志自动扩展。如果发现当前需要拷贝的日志比全局的日志缓存一半还大，就会自动把全局日志缓存给扩大一倍。注意，只要扩大后，就不会再缩小了。
3. 日志对齐。早期的磁盘都是512原子写，现代的SSD磁盘大部分是4K原子写。如果小于4K的写入，会导致先把4K读取出来，然后内存中修改，最后再写下去，性能低下。但是有了日志对齐这个优化后，可以以指定大小刷日志，不够大的后面填0补齐，能提高写入效率。

这里贴一个优化后的日志写入磁盘的伪代码：

`mutex_enter(log_sys->write_mutex);
check if other thead has done write for us
mutex_enter(log_sys->mutex);
calculate the range log need to be write
switch log buffer so that user threads can still copy log during writing
mutex_exit(log_sys->mutex);
align log to specified size if needed
write log to log file 
log_sys->write_lsn = log_sys->lsn;
mutex_exit(log_sys->write_mutex);
`
可以看到log_sys->mutex被进一步缩小。往日志文件里面写日志的阶段已经不许要log_sys->mutex保护了。
有了以上的优化，MySQL的日志子系统在大多数场景下不会达到瓶颈。但是，用户线程往全局日志缓存拷贝日志以及脏页加入脏页链表这两个操作，依然是基于锁机制的，很难发挥出多核系统的性能。

## MySQL 8.0版本的处理方式
之前的版本虽然做了很多优化，但是没有真正做到lock free，在高并发下，可以看到很多锁冲突。官方因此在这块下了大力气，彻头彻尾的大改了一番。

详细细节可以参考上个月[这篇月报](http://mysql.taobao.org/monthly/2018/06/01/)。

这里再简单概括一下。

在日志写入阶段，通过atomic变量分配保留空间，由于atomic变量增长是个原子操作，所以这一步不要加锁。分配完空间后，就可以拷贝日志，由于上一步中空间已经被预留，所以多线程可以同时进行拷贝，而不会导致日志有重叠。但是不能保证拷贝完成的先后顺序，有可能先拷贝的，后完成，所以需要有一种机制来保证某个点之前的日志已经都拷贝到全局日志缓存了。这里，官方就引入了一种新的lock free数据结构Link_buf，它是一个数组，用来标记拷贝完成的情况。每个用户线程完成拷贝后，就在那个数组中标记一下，然后后台再开一个线程来计算是否有连续的块完成拷贝了，完成了就可以把这些日志刷到磁盘。

在脏页插入脏页链表这一块，官方也提出了一种有趣的算法，它也是基于新的lock free数据结构Link_buf。基本思想是，脏页链表的有序性可以被部分的打破，也就是说，在一定范围内可以无序，但是整体还是有序的。这个无序程度是受控的。假设脏页链表第一个数据页的oldest_modification为A, 在之前的版本中，这个脏页链表后续的page的oldest_modification都严格大于等于A，也就是不存在一个数据页比第一个数据页还老。在MySQL 8.0中，后续的page的oldest_modification并不是严格大于等于A，可以比A小，但是必须大于等于A-L，这个L可以理解为无序度，是一个定值。那么问题来了，如果脏页链表顺序乱了，那么checkpoint怎么确定，或者说是，奔溃恢复后，从哪个checkpoint_lsn开始扫描日志才能保证数据不丢。官方给出的解法是，checkpoint依然由脏页链表中第一个数据页的oldest_modification的确定，但是奔溃恢复从checkpoint_lsn-L开始扫描(有可能这个值不是一个mtr的边界，因此需要调整)。

所以可以看到，官方通过link_buf这个数据结构很巧妙的解决了局部日志往全局日志拷贝的问题以及脏页插入脏页链表的问题。由于都是lock free算法，因此扩展性会比较好。

但是，从实际测试的情况来看，似乎是因为用了太多的条件变量event，在我们的测试中没有达到官方标称的性能。后续我们会进一步分析原因。

## POLARDB FOR MYSQL的处理方式
POLARDB作为阿里云下一代关系型云数据库，我们自然在InnoDB日志子系统做了很多优化，其中也包含了上述的领域。这里可以简单介绍一下我们的思路:

每个buffer pool instance都额外增加了一把读写锁(rw_locks)，主要用来控制对全局日志缓存的访问。
此外还引入两个存储脏页信息的集合，我们这里简称in-flight set和ready-to-process set。主要用来临时存储脏页信息。

日志进入全局缓存:

`release all share locks holded by this mtr's page
acquire log_buf s-locks for all buf_pool instances for which we have dirty pages
reserver enough space on log_buf via increasing atomit variables //Just like MySQL 8.0
copy local log to global log buffer
add all pages dirtied by this mtr to in-flight set
release all exclusive locks holded by this mtr's page
release log_buf s-locks for all buf_pool instances

`

日志写入磁盘：

`mutex_enter(log_sys->write_mutex)
check if other thead has done write for us
mutex_enter(log_sys->mutex)
acquire log_buf x-locks for all buf_pool instances
update log_sys->lsn to newest
switch log buffer so that user threads can still copy log during writing
mutex_exit(log_sys->mutex)
release log_buf x-locks for all buf_pool instances
align log to specified size if needed
write log to log file 
log_sys->write_lsn = log_sys->lsn;
mutex_exit(log_write_mutex)
`

刷脏线程(每个buffer pool instance)：

`acquire log_buf x-locks for specific buffer pool instance
toggle in-flight set with ready-to-process set. Only this thread will toggle between these two.
release log_buf x-locks for specific buffer pool instance
for each page in ready-to-process
 add page to flush list
do normal flush page operations
`

更新checkpoint:

`for ervery flush list:
 acquire log_buf x-locks for specific buffer pool instance
 ready_to_process_lsn = minimum oldest_modification in ready-to-process set
 flush_list_lsn = get_first_page(curr_flush_list).oldest_modification
 min_lsn = min(ready_to_process_lsn, flush_list_lsn)
 release log_buf x-locks for specific buffer pool instance
 if current_oldest_modification > min_lsn
 current_oldest_modification = min_lsn
checkpoint_lsn = current_oldest_modification
write checkpoint_lsn to log file
`

奔溃恢复：

`read checkpoint_lsn from log file
start parse and apply redo log from checkpoint_lsn point
`

在局部日志拷贝入全局日志这块，与官方MySQL 8.0类似，首先利用atomic变量的原子增长来分配空间，但是MySQL 8.0是使用link_buf来保证拷贝完成，而在POLARDB中，我们使用读写锁的机制，即在拷贝之前加上读锁，拷贝完才释放读锁，而在日志写入磁盘前，首先尝试加上写锁，利用写锁和读锁互斥的特性，保证在获取写锁时所有读锁都释放，即所有拷贝操作都完成。

在脏页进入脏页链表这块，官方MySQL允许脏页链表有一定的无序度(也是通过link_buf保证)，然后通过在奔溃恢复的时候从checkpoint_lsn-L开始扫描的机制，来保证数据的一致性。在POLARDB中，我们解决办法是，把脏页临时加入到一个集合，在刷脏线程工作前再按顺序加入脏页链表，通过获取写锁来保证在加入脏页链表前，整个集合是完整的。换句话说，假设这个脏页集合最小的oldest_modification为A，那么可以保证没有加入脏页集合的脏页的oldest_modification都大于等于A。

从脏页集合加入到脏页链表的操作，我们没有加锁，所以在更行checkpoint的时候，我们需要使用min(ready_to_process_lsn, flush_list_lsn)来作为checkpoint_lsn。在奔溃恢复的时候，直接从checkpoint_lsn扫描即可。

此外，我们在POLARDB上，还做了额外的优化:

1. 提前释放page的共享锁。如果一个数据页被加了共享锁，说明没有被修改，只是被读取而已，我们可以提前释放掉，这有助于减少热点数据页的锁冲突。
2. 在日志进入全局缓存时，我们没有及时更新log_sys->lsn，而是先更新另外一个变量，当在日志写入磁盘前，即获取log_buf写锁后，然后在更新log_sys->lsn。主要是为了减少冲突。

最后我们测试了一下性能，在non_index_updates的全内存高并发测试下，性能有10%的提高。

`Upstream 5.6.40: 71K
MySQL-8.0: 132K
POLARDB (master): 162K
POLARDB(master + mtr_optimize): 178K
`
当然，这不是我们最高的性能，可以小小透露一下，通过对事务子系统的优化，我们可以达到200K的性能。

更多更好用的功能都在路上，欢迎使用POLARDB！

## 总结
日志子系统是关系型数据库不可获取的模块，也是数据库内核开发者非常感兴趣的模块，本文结合代码分析了MySQL不同版本的WAL机制的实现，希望对大家有所帮助。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)