# MySQL · 性能优化· InnoDB buffer pool flush策略漫谈

**Date:** 2015/02
**Source:** http://mysql.taobao.org/monthly/2015/02/01/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 02
 ](/monthly/2015/02)

 * 当期文章

 MySQL · 性能优化· InnoDB buffer pool flush策略漫谈
* MySQL · 社区动态· 5.6.23 InnoDB相关Bugfix
* PgSQL · 特性分析· Replication Slot
* PgSQL · 特性分析· pg_prewarm
* MySQL · 答疑释惑· InnoDB丢失自增值
* MySQL · 答疑释惑· 5.5 和 5.6 时间类型兼容问题
* MySQL · 捉虫动态· 变量修改导致binlog错误
* MariaDB · 特性分析· 表/表空间加密
* MariaDB · 特性分析· Per-query variables
* TokuDB · 特性分析· 日志详解

 ## MySQL · 性能优化· InnoDB buffer pool flush策略漫谈 
 Author: 

 **背景**

我们知道InnoDB使用buffer pool来缓存从磁盘读取到内存的数据页。buffer pool通常由数个内存块加上一组控制结构体对象组成。内存块的个数取决于buffer pool instance的个数，不过在5.7版本中开始默认以128M（可配置）的chunk单位分配内存块，这样做的目的是为了支持buffer pool的在线动态调整大小。

Buffer pool的每个内存块通过mmap的方式分配内存，因此你会发现，在实例启动时虚存很高，而物理内存很低。这些大片的内存块又按照16KB划分为多个frame，用于存储数据页。

虽然大多数情况下buffer pool是以16KB来存储数据页，但有一种例外：使用压缩表时，需要在内存中同时存储压缩页和解压页，对于压缩页，使用Binary buddy allocator算法来分配内存空间。例如我们读入一个8KB的压缩页，就从buffer pool中取一个16KB的block，取其中8KB，剩下的8KB放到空闲链表上；如果紧跟着另外一个4KB的压缩页读入内存，就可以从这8KB中分裂4KB，同时将剩下的4KB放到空闲链表上。

为了管理buffer pool，每个buffer pool instance 使用如下几个链表来管理：

* LRU链表包含所有读入内存的数据页；
* Flush_list包含被修改过的脏页；
* unzip_LRU包含所有解压页；
* Free list上存放当前空闲的block。

另外为了避免查询数据页时扫描LRU，还为每个buffer pool instance维护了一个page hash，通过space id 和page no可以直接找到对应的page。

一般情况下，当我们需要读入一个Page时，首先根据space id 和page no找到对应的buffer pool instance。然后查询page hash，如果page hash中没有，则表示需要从磁盘读取。在读盘前首先我们需要为即将读入内存的数据页分配一个空闲的block。当free list上存在空闲的block时，可以直接从free list上摘取；如果没有，就需要从unzip_lru 或者 lru上驱逐page。

这里需要遵循一定的原则（参考函数buf_LRU_scan_and_free_block , 5.7.5）：

1. 首先尝试从unzip_lru上驱逐解压页；
2. 如果没有，再尝试从Lru链表上驱逐Page；
3. 如果还是无法从Lru上获取到空闲block，用户线程就会参与刷脏，尝试做一次SINGLE PAGE FLUSH，单独从Lru上刷掉一个脏页，然后再重试。

Buffer pool中的page被修改后，不是立刻写入磁盘，而是由后台线程定时写入，和大多数数据库系统一样，脏页的写盘遵循日志先行WAL原则，因此在每个block上都记录了一个最近被修改时的Lsn，写数据页时需要确保当前写入日志文件的redo不低于这个Lsn。

然而基于WAL原则的刷脏策略可能带来一个问题：当数据库的写入负载过高时，产生redo log的速度极快，redo log可能很快到达同步checkpoint点。这时候需要进行刷脏来推进Lsn。由于这种行为是由用户线程在检查到redo log空间不够时触发，大量用户线程将可能陷入到这段低效的逻辑中，产生一个明显的性能拐点。

**Page Cleaner线程**

在MySQL5.6中，开启了一个独立的page cleaner线程来进行刷lru list 和flush list。默认每隔一秒运行一次，5.6版本里提供了一大堆的参数来控制page cleaner的flush行为，包括：

`innodb_adaptive_flushing_lwm， 
innodb_max_dirty_pages_pct_lwm
innodb_flushing_avg_loops
innodb_io_capacity_max
innodb_lru_scan_depth
`

这里我们不一一介绍，总的来说，如果你发现redo log推进的非常快，为了避免用户线程陷入刷脏，可以通过调大innodb_io_capacity_max来解决，该参数限制了每秒刷新的脏页上限，调大该值可以增加Page cleaner线程每秒的工作量。如果你发现你的系统中free list不足，总是需要驱逐脏页来获取空闲的block时，可以适当调大innodb_lru_scan_depth 。该参数表示从每个buffer pool instance的lru上扫描的深度，调大该值有助于多释放些空闲页，避免用户线程去做single page flush。

为了提升扩展性和刷脏效率，在5.7.4版本里引入了多个page cleaner线程，从而达到并行刷脏的效果。目前Page cleaner并未和buffer pool绑定，其模型为一个协调线程 + 多个工作线程，协调线程本身也是工作线程。因此如果innodb_page_cleaners设置为4，那么就是一个协调线程，加3个工作线程，工作方式为生产者-消费者。工作队列长度为buffer pool instance的个数，使用一个全局slot数组表示。

协调线程在决定了需要flush的page数和lsn_limit后，会设置slot数组，将其中每个slot的状态设置为PAGE_CLEANER_STATE_REQUESTED, 并设置目标page数及lsn_limit，然后唤醒工作线程 (pc_request)

工作线程被唤醒后，从slot数组中取一个未被占用的slot，修改其状态，表示已被调度，然后对该slot所对应的buffer pool instance进行操作。直到所有的slot都被消费完后，才进入下一轮。通过这种方式，多个page cleaner线程实现了并发flush buffer pool，从而提升flush dirty page/lru的效率。

**MySQL5.7的InnoDB flush策略优化**

在之前版本中，因为可能同时有多个线程操作buffer pool刷page （在刷脏时会释放buffer pool mutex），每次刷完一个page后需要回溯到链表尾部，使得扫描bp链表的时间复杂度最差为O（N*N）。

在5.6版本中针对Flush list的扫描做了一定的修复，使用一个指针来记录当前正在flush的page，待flush操作完成后，再看一下这个指针有没有被别的线程修改掉，如果被修改了，就回溯到链表尾部，否则无需回溯。但这个修复并不完整，在最差的情况下，时间复杂度依旧不理想。

因此在5.7版本中对这个问题进行了彻底的修复，使用多个名为hazard pointer的指针，在需要扫描LIST时，存储下一个即将扫描的目标page，根据不同的目的分为几类：

* flush_hp: 用作批量刷FLUSH LIST
* lru_hp: 用作批量刷LRU LIST
* lru_scan_itr: 用于从LRU链表上驱逐一个可替换的page，总是从上一次扫描结束的位置开始，而不是LRU尾部
* single_scan_itr: 当buffer pool中没有空闲block时，用户线程会从FLUSH LIST上单独驱逐一个可替换的page 或者 flush一个脏页，总是从上一次扫描结束的位置开始，而不是LRU尾部。

后两类的hp都是由用户线程在尝试获取空闲block时调用，只有在推进到某个buf_page_t::old被设置成true的page (大约从Lru链表尾部起至总长度的八分之三位置的page)时， 再将指针重置到Lru尾部。

这些指针在初始化buffer pool时分配，每个buffer pool instance都拥有自己的hp指针。当某个线程对buffer pool中的page进行操作时，例如需要从LRU中移除Page时，如果当前的page被设置为hp，就要将hp更新为当前Page的前一个page。当完成当前page的flush操作后，直接使用hp中存储的page指针进行下一轮flush。

**社区优化**

一如既往的，Percona Server在5.6版本中针对buffer pool flush做了不少的优化，主要的修改包括如下几点：

* 优化刷LRU流程buf_flush_LRU_tail 
该函数由page cleaner线程调用。
* 原生的逻辑：依次flush 每个buffer pool instance，每次扫描的深度通过参数innodb_lru_scan_depth来配置。而在每个instance内，又分成多个chunk来调用；
* 修改后的逻辑为：每次flush一个buffer pool的LRU时，只刷一个chunk，然后再下一个instance，刷完所有instnace后，再回到前面再刷一个chunk。简而言之，把集中的flush操作进行了分散，其目的是分散压力，避免对某个instance的集中操作，给予其他线程更多访问buffer pool的机会。
* 允许设定刷LRU/FLUSH LIST的超时时间，防止flush操作时间过长导致别的线程（例如尝试做single page flush的用户线程）stall住；当到达超时时间时,page cleaner线程退出flush。
* 避免用户线程参与刷buffer pool 
当用户线程参与刷buffer pool时，由于线程数的不可控，将产生严重的竞争开销，例如free list不足时做single page flush，以及在redo空间不足时，做dirty page flush，都会严重影响性能。Percona Server允许选择让page cleaner线程来做这些工作，用户线程只需要等待即可。出于效率考虑，用户还可以设置page cleaner线程的cpu调度优先级。 
另外在Page cleaner线程经过优化后，可以知道系统当前处于同步刷新状态，可以去做更激烈的刷脏(furious flush)，用户线程参与到其中，可能只会起到反作用。
* 允许设置page cleaner线程，purge线程，io线程，master线程的CPU调度优先级，并优先获得InnoDB的mutex。
* 使用新的独立后台线程来刷buffer pool的LRU链表，将这部分工作负担从page cleaner线程剥离。 
实际上就是直接转移刷LRU的代码到独立线程了。从之前Percona的版本来看，都是在不断的强化后台线程，让用户线程少参与到刷脏/checkpoint这类耗时操作中。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)