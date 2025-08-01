# MySQL · 引擎特性 · InnoDB 文件系统之IO系统和内存管理

**Date:** 2016/02
**Source:** http://mysql.taobao.org/monthly/2016/02/02/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 02
 ](/monthly/2016/02)

 * 当期文章

 MySQL · 引擎特性 · InnoDB 文件系统之文件物理结构
* MySQL · 引擎特性 · InnoDB 文件系统之IO系统和内存管理
* MySQL · 特性分析 · InnoDB transaction history
* PgSQL · 会议见闻 · PgConf.Russia 2016 大会总结
* PgSQL · 答疑解惑 · PostgreSQL 9.6 并行查询实现分析
* MySQL · TokuDB · TokuDB之黑科技工具
* PgSQL · 性能优化 · PostgreSQL TPC-C极限优化玩法
* MariaDB · 版本特性 · MariaDB 的 GTID 介绍
* MySQL · 特性分析 · 线程池
* MySQL · 答疑解惑 · mysqldump tips 两则

 ## MySQL · 引擎特性 · InnoDB 文件系统之IO系统和内存管理 
 Author: 印风 

 ## 综述

在[前一篇](/monthly/2016/02/01/)我们介绍了InnoDB文件系统的物理结构，本篇我们继续介绍InnoDB文件系统的IO接口和内存管理。

为了管理磁盘文件的读写操作，InnoDB设计了一套文件IO操作接口，提供了同步IO和异步IO两种文件读写方式。针对异步IO，支持两种方式：一种是Native AIO，这需要你在编译阶段加上LibAio的Dev包，另外一种是simulated aio模式，InnoDB早期实现了一套系统来模拟异步IO，但现在Native Aio已经很成熟了，并且Simulated Aio本身存在性能问题，建议生产环境开启Native Aio模式。

对于数据读操作，通常用户线程触发的数据块请求读是同步读，如果开启了数据预读机制的话，预读的数据块则为异步读，由后台IO线程进行。其他后台线程也会触发数据读操作，例如Purge线程在无效数据清理，会读undo页和数据页；Master线程定期做ibuf merge也会读入数据页。崩溃恢复阶段也可能触发异步读来加速recover的速度。

对于数据写操作，InnoDB和大部分数据库系统一样，都是WAL模式，即先写日志，延迟写数据页。事务日志的写入通常在事务提交时触发，后台master线程也会每秒做一次redo fsync。数据页则通常由后台Page cleaner线程触发。但当buffer pool空闲block不够时，或者没做checkpoint的lsn age太长时，也会驱动刷脏操作，这两种场景由用户线程来触发。Percona Server据此做了优化来避免用户线程参与。MySQL5.7也对应做了些不一样的优化。

除了数据块操作，还有物理文件级别的操作，例如truncate、drop table、rename table等DDL操作，InnoDB需要对这些操作进行协调，目前的解法是通过特殊的flag和计数器的方式来解决。

当文件读入内存后，我们需要一种统一的方式来对数据进行管理，在启动实例时，InnoDB会按照instance分区分配多个一大块内存（在5.7里则是按照可配置的chunk size进行内存块划分），每个chunk又以UNIV_PAGE_SIZE为单位进行划分。数据读入内存时，会从buffer pool的free list中分配一个空闲block。所有的数据页都存储在一个LRU链表上，修改过的block被加到`flush_list`上，解压的数据页被放到unzip_LRU链表上。我们可以配置buffer pool为多个instance，以降低对链表的竞争开销。

在关键的地方本文注明了代码函数，建议读者边参考代码边阅读本文，本文的代码部分基于MySQL 5.7.11版本，不同的版本函数名或逻辑可能会有所不同。请读者阅读本文时尽量选择该版本的代码。

## IO子系统

本小节我们介绍下磁盘文件与内存数据的中枢，即IO子系统。InnoDB对page的磁盘操作分为读操作和写操作。

对于读操作，在将数据读入磁盘前，总是为其先预先分配好一个block，然后再去磁盘读取一个新的page，在使用这个page之前，还需要检查是否有change buffer项，并根据change buffer进行数据变更。读操作分为两种场景：普通的读page及预读操作，前者为同步读，后者为异步读

数据写操作也分为两种，一种是batch write，一种是single page write。写page默认受double write buffer保护，因此对double write buffer的写磁盘为同步写，而对数据文件的写入为异步写。

同步读写操作通常由用户线程来完成，而异步读写操作则需要后台线程的协同。

举个简单的例子，假设我们向磁盘批量写数据，首先先写到double write buffer，当dblwr满了之后，一次性将dblwr中的数据同步刷到ibdata，在确保sync到dblwr后，再将这些page分别异步写到各自的文件中。注意这时候dblwr依旧未被清空，新的写Page请求会进入等待。当异步写page完成后，io helper线程会调用`buf_flush_write_complete`，将写入的Page从flush list上移除。当dblwr中的page完全写完后，在函数`buf_dblwr_update`里将dblwr清空。这时候才允许新的写请求进dblwr。

同样的，对于异步写操作，也需要IO Helper线程来检查page是否完好、merge change buffer等一系列操作。

除了数据页的写入，还包括日志异步写入线程、及ibuf后台线程。

### IO后台线程

InnoDB的IO后台线程主要包括如下几类：

* IO READ 线程：后台读线程，线程数目通过参数`innodb_read_io_threads`配置，主要处理INNODB 数据文件异步读请求，任务队列为`AIO::s_reads`，任务队列包含slot数为线程数 * 256(linux 平台)，也就是说，每个read线程最多可以pend 256个任务；
* IO WRITE 线程：后台写线程数，线程数目通过参数`innodb_write_io_threads`配置。主要处理INNODB 数据文件异步写请求，任务队列为`AIO::s_writes`，任务队列包含slot数为线程数 * 256(linux 平台)，也就是说，每个read线程最多可以pend 256个任务；
* LOG 线程：写日志线程。只有在写checkpoint信息时才会发出一次异步写请求。任务队列为`AIO::s_log`，共1个segment，包含256个slot；
* IBUF 线程：负责读入change buffer页的后台线程，任务队列为`AIO::s_ibuf`，共1个segment，包含256个slot

所有的同步写操作都是由用户线程或其他后台线程执行。上述IO线程只负责异步操作。

### 发起IO请求

入口函数：`os_aio_func`

首先对于同步读写请求（`OS_AIO_SYNC`），发起请求的线程直接调用`os_file_read_func` 或者`os_file_write_func` 去读写文件，然后返回。

对于异步请求，用户线程从对应操作类型的任务队列（`AIO::select_slot_array`）中选取一个slot，将需要读写的信息存储于其中（`AIO::reserve_slot`）:

1. 首先在任务队列数组中选择一个segment；这里根据偏移量来算segment，因此可以尽可能的将相邻的读写请求放到一起，这有利于在IO层的合并操作

 ` local_seg = (offset >> (UNIV_PAGE_SIZE_SHIFT + 6)) % m_n_segments;
`
2. 从该segment范围内选择一个空闲的slot，如果没有则等待；
3. 将对应的文件读写请求信息赋值到slot中，例如写入的目标文件，偏移量，数据等；
4. 如果这是一次IO写入操作，且使用native aio时，如果表开启了transparent compression，则对要写入的数据页先进行压缩并punch hole；如果设置了表空间加密，再对数据页进行加密；

对于Native AIO（使用linux自带的LIBAIO库），调用函数`AIO::linux_dispatch`，将IO请求分发给kernel层。

如果没有开启Native AIO，且没有设置wakeup later 标记，则会去唤醒io线程（`AIO::wake_simulated_handler_thread`），这是早期libaio还不成熟时，InnoDB在内部模拟aio实现的逻辑。

Tips：编译Native AIO需要安装libaio-dev包，并打开选项`srv_use_native_aio`。

### 处理异步AIO请求

IO线程入口函数为`io_handler_thread --> fil_aio_wait`

首先调用`os_aio_handler`来获取请求：

1. 对于Native AIO，调用函数`os_aio_linux_handle` 获取读写请求。IO线程会反复以500ms（`OS_AIO_REAP_TIMEOUT`）的超时时间通过io_getevents确认是否有任务已经完成了（`LinuxAIOHandler::collect()`），如果有读写任务完成，找到已完成任务的slot后，释放对应的槽位；
2. 对于simulated aio，调用函数`os_aio_simulated_handler` 处理读写请求，这里相比NATIVE AIO要复杂很多；
 * 如果这是异步读队列，并且`os_aio_recommend_sleep_for_read_threads`被设置，则暂时不处理，而是等待一会，让其他线程有机会将更过的IO请求发送过来。目前linear readhaed 会使用到该功能。这样可以得到更好的IO合并效果(`SimulatedAIOHandler::check_pending`)；
* 已经完成的slot需要及时被处理(`SimulatedAIOHandler::check_completed`，可能由上次的io合并操作完成)；
* 如果有超过2秒未被调度的请求(`SimulatedAIOHandler::select_oldest`)，则优先选择最老的slot，防止饿死，否则，找一个文件读写偏移量最小的位置的slot(`SimulatedAIOHandler::select()`)；
* 没有任何请求时进入等待状态；
* 当找到一个未完成的slot时，会尝试merge相邻的IO请求（`SimulatedAIOHandler::merge()`），并将对应的slot加入到`SimulatedAIOHandler::m_slots`数组中，最多不超过64个slot；
* 然而在5.7版本里，合并操作已经被禁止了，全部改成了一个个slot进行读写，升级到5.7的用户一定要注意这个改变，或者改为使用更好的Native AIO方式；
* 完成io后，释放slot; 并选择第一个处理完的slot作为随后优先完成的请求。

从上一步获得完成IO的slot后，调用函数`fil_node_complete_io`， 递减`node->n_pending`。对于文件写操作，需要加入到`fil_system->unflushed_spaces`链表上，表示这个文件修改过了，后续需要被sync到磁盘。

如果设置为`O_DIRECT_NO_FSYNC`的文件IO模式，则数据文件无需加入到`fil_system_t::unflushed_spaces`链表上。通常我们即时使用`O_DIRECT`的方式操作文件，也需要做一次sync，来保证文件元数据的持久化，但在某些文件系统下则没有这个必要，通常只要文件的大小这些关键元数据没发生变化，可以省略一次fsync。

最后在IO完成后，调用`buf_page_io_complete`，做page corruption检查、change buffer merge等操作；对于写操作，需要从flush list上移除block并更新double write buffer；对于LRU FLUSH产生的写操作，还会将其对应的block释放到free list上；

对于日志文件操作，调用`log_io_complete`执行一次fil_flush，并更新内存内的checkpoint信息（`log_complete_checkpoint`）。

### IO 并发控制

由于文件底层使用pwrite/pread来进行文件I/O，因此用户线程对文件普通的并发I/O操作无需加锁。但在windows平台下，则需要加锁进行读写。

对相同文件的IO操作通过大量的counter/flag来进行并发控制。

当文件处于扩展阶段时（`fil_space_extend`），将`fil_node_t::being_extended`设置为true，避免产生并发Extent，或其他关闭文件或者rename操作等。

当正在删除一个表时，会检查是否有pending的操作（`fil_check_pending_operations`）。

1. 将`fil_space_t::stop_new_ops`设置为true；
2. 检查是否有Pending的change buffer merge (`fil_space_t::n_pending_ops`)；有则等待；
3. 检查是否有pending的IO（`fil_node_t::n_pending`） 或者pending的文件flush操作（`fil_node_t::n_pending_flushes`）；有则等待。

当truncate一张表时，和drop table类似，也会调用函数`fil_check_pending_operations`，检查表上是否有pending的操作，并将`fil_space_t::is_being_truncated`设置为true。

当rename一张表时（`fil_rename_tablespace`），将文件的stop_ios标记设置为true，阻止其他线程所有的I/O操作。

当进行文件读写操作时，如果是异步读操作，发现`stop_new_ops`或者被设置了但`is_being_truncated`未被设置，会返回报错；但依然允许同步读或异步写操作(`fil_io`)。

当进行文件flush操作时，如果发现`fil_space_t::stop_new_ops`或者`fil_space_t::is_being_truncated`被设置了，则忽略该文件的flush操作 （`fil_flush_file_spaces`）。

### 文件预读

文件预读是一项在SSD普及前普通磁盘上比较常见的技术，通过预读的方式进行连续IO而非带价高昂的随机IO。InnoDB有两种预读方式：随机预读及线性预读；Facebook另外还实现了一种逻辑预读的方式。

**随机预读**
入口函数：`buf_read_ahead_random`

以64个Page为单位(这也是一个Extent的大小)，当前读入的page no所在的64个pagno 区域[ (page_no/64)*64, (page_no/64) *64 + 64]，如果最近被访问的Page数超过`BUF_READ_AHEAD_RANDOM_THRESHOLD`（通常值为13），则将其他Page也读进内存。这里采取异步读。

随机预读受参数`innodb_random_read_ahead`控制

**线性预读**
入口函数：`buf_read_ahead_linear`

所谓线性预读，就是在读入一个新的page时，和随机预读类似的64个连续page范围内，默认从低到高Page no，如果最近连续被访问的page数超过`innodb_read_ahead_threshold`，则将该Extent之后的其他page也读取进来。

**逻辑预读**
由于表可能存在碎片空间，因此很可能对于诸如全表扫描这样的场景，连续读取的page并不是物理连续的，线性预读不能解决这样的问题，另外一次读取一个Extent对于需要全表扫描的负载并不足够。因此Facebook引入了逻辑预读。

其大致思路为，扫描聚集索引，搜集叶子节点号，然后根据叶子节点的page no (可以从非叶子节点获取)顺序异步读入一定量的page。

由于Innodb Aio一次只支持提交一个page读请求，虽然Kernel层本身会做读请求合并，但那显然效率不够高。他们对此做了修改，使INNODB可以支持一次提交（`io_submit`）多个aio请求。

入口函数：`row_search_for_mysql --> row_read_ahead_logical`

具体参阅[这篇博文](http://planet.mysql.com/entry/?id=516236)

或者webscalesql上的几个commit：

`git show 2d61329446a08f85c89a4119317ae85baacf2bbb // 合并多个AIO请求，对所有的预读逻辑（上述三种）采用这种方式
git show 9f52bfd2222403f841fe5fcbedd1333f78a70a4b // 逻辑预读的主要代码逻辑
git show 64b68e07430b50f6bff5ed67374b336623db24b6 // 防止事务在多个表上读取操作时预读带来的影响
`

### 日志填充写入

由于现代磁盘通常的block size都是大于512字节的，例如一般是4096字节，为了避免 “read-on-write” 问题，在5.7版本里添加了一个参数`innodb_log_write_ahead_size`，你可以通过配置该参数，在写入redo log时，将写入区域配置到block size对齐的字节数。

在代码里的实现，就是在写入redo log 文件之前，为尾部字节填充0（参考函数`log_write_up_to`）。

Tips：所谓READ-ON-WRITE问题，就是当修改的字节不足一个block时，需要将整个block读进内存，修改对应的位置，然后再写进去；如果我们以block为单位来写入的话，直接完整覆盖写入即可。

## buffer pool 内存管理

InnoDB buffer pool从5.6到5.7版本发生了很大的变化。首先是分配方式上不同，其次实现了更好的刷脏效率。对buffer pool上的各个链表的管理也更加高效。

### buffer pool初始化

在5.7之前的版本中，一个buffer pool instance拥有一个chunk，每个chunk的大小为buffer pool size / instance个数。

而到了5.7版本中，每个instance可能划分成多个chunk，每个chunk的大小是可定义的，默认为127MB。因此一个buffer pool instance可能包含多个chunk内存块。这么做的目的是为了实现在线调整buffer pool大小([WL#6117](http://dev.mysql.com/worklog/task/?id=6117))，buffer pool增加或减少必须以chunk为基本单位进行。

在5.7里有个问题值得关注，即buffer pool size会根据instances * chunk size向上对齐，举个简单的例子，假设你配置了64个instance, chunk size为默认128MB，就需要以8GB进行对齐，这意味着如果你配置了9GB的buffer pool，实际使用的会是16GB。所以**尽量不要配置太多的buffer pool instance**。

### buffer pool 链表及管理对象

出于不同的目的，每个buffer pool instance上都维持了多个链表，可以根据space id及page no找到对应的instance(`buf_pool_get`)。

一些关键的结构对象及描述如下表所示：

 name
 desc

 buf_pool_t::page_hash
 page_hash用于存储已经或正在读入内存的page。根据<space_id, page_no>快速查找。当不在page hash时，才会去尝试从文件读取

 buf_pool_t::LRU
 LRU上维持了所有从磁盘读入的数据页，该LRU上又在链表尾部开始大约3/8处将链表划分为两部分，新读入的page被加入到这个位置；当我们设置了innodb_old_blocks_time，若两次访问page的时间超过该阀值，则将其挪动到LRU头部；这就避免了类似一次性的全表扫描操作导致buffer pool污染

 buf_pool_t::free
 存储了当前空闲可分配的block

 buf_pool_t::flush_list
 存储了被修改过的page，根据oldest_modification（即载入内存后第一次修改该page时的Redo LSN）排序

 buf_pool_t::flush_rbt
 在崩溃恢复阶段在flush list上建立的红黑数，用于将apply redo后的page快速的插入到flush list上，以保证其有序

 buf_pool_t::unzip_LRU
 压缩表上解压后的page被存储到unzip_LRU。 buf_block_t::frame存储解压后的数据，buf_block_t::page->zip.data指向原始压缩数据。

 buf_pool_t::zip_free[BUF_BUDDY_SIZES_MAX]
 用于管理压缩页产生的空闲碎片page。压缩页占用的内存采用buddy allocator算法进行分配。

### buffer pool 并发控制

除了不同的用户线程会并发操作buffer pool外，还有后台线程也会对buffer pool进行操作。InnoDB通过读写锁、buf fix计数、io fix标记来进行并发控制。

**读写并发控制**
通常当我们读取到一个page时，会对其加block S锁，并递增`buf_page_t::buf_fix_count`，直到mtr commit时才会恢复。而如果读page的目的是为了进行修改，则会加X锁。

当一个page准备flush到磁盘时(`buf_flush_page`)，如果当前Page正在被访问，其`buf_fix_count`不为0时，就忽略flush该page，以减少获取block上SX Lock的昂贵代价。

**并发读控制**
当多个线程请求相同的page时，如果page不在内存，是否可能引发对同一个page的文件IO ？答案是不会。

从函数`buf_page_init_for_read`我们可以看到，在准备读入一个page前，会做如下工作：

1. 分配一个空闲block；
2. `buf_pool_mutex_enter`；
3. 持有page_hash x lock；
4. 检查page_hash中是否已被读入，如果是，表示另外一个线程已经完成了io，则忽略本次io请求，退出；
5. 持有`block->mutex`，对block进行初始化，并加入到page hash中；
6. 设置IO FIX为`BUF_IO_READ`；
7. 释放hash lock；
8. 将block加入到LRU上；
9. 持有block s lock；
10. 完成IO后，释放s lock；

当另外一个线程也想请求相同page时，首先如果看到page hash中已经有对应的block了，说明page已经或正在被读入buffer pool，如果`io_fix`为`BUF_IO_READ`，说明正在进行IO，就通过加X锁的方式做一次sync（`buf_wait_for_read`），确保IO完成。

请求Page通常还需要加S或X锁，而IO期间也是持有block x锁的，如果成功获取了锁，说明IO肯定完成了。

### Page驱逐及刷脏

当buffer pool中的free list不足时，为了获取一个空闲block，通常会触发page驱逐操作(`buf_LRU_free_from_unzip_LRU_list`)。

首先由于压缩页在内存中可能存在两份拷贝：压缩页和解压页；InnoDB根据最近的IO情况和数据解压技术来判定实例是处于IO-BOUND还是CPU-BOUND（`buf_LRU_evict_from_unzip_LRU`）。如果是IO-BOUND的话，就尝试从unzip_lru上释放一个block出来(`buf_LRU_free_from_unzip_LRU_list`)，而压缩页依旧保存在内存中。

其次再考虑从`buf_pool_t::LRU`链表上释放block，如果有可替换的page(`buf_flush_ready_for_replace`)时，则将其释放掉，并加入到free list上；对于压缩表，压缩页和解压页在这里都会被同时驱逐。

当无法从LRU上获得一个可替换的Page时，说明当前Buffer pool可能存在大量脏页，这时候会触发single page flush(`buf_flush_single_page_from_LRU`)，即用户线程主动去刷一个脏页并替换掉。这是个慢操作，尤其是如果并发很高的时候，可能观察到系统的性能急剧下降。在RDS MySQL中，我们开启了一个后台线程， 能够自动根据当前Free List的长度来主动做flush，避免用户线程陷入其中。

除了single page flush外，在MySQL 5.7版本里还引入了多个page cleaner线程，根据一定的启发式算法，可以定期且高效的的做page flush操作。

本文对此不展开讨论，感兴趣的可以阅读我之前的月报：

1. MySQL · 性能优化· 5.7.6 InnoDB page flush 优化
2. MySQL · 性能优化· InnoDB buffer pool flush策略漫谈

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)