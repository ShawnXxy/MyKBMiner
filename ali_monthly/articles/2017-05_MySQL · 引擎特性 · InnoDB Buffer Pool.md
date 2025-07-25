# MySQL · 引擎特性 · InnoDB Buffer Pool

**Date:** 2017/05
**Source:** http://mysql.taobao.org/monthly/2017/05/01/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 05
 ](/monthly/2017/05)

 * 当期文章

 MySQL · 引擎特性 · InnoDB Buffer Pool
* AliSQL · 特性介绍 · 动态加字段
* PgSQL · 特性分析 · 数据库崩溃恢复（上）
* MySQL · 答疑解惑 · MySQL 的那些网络超时错误
* HybridDB · 最佳实践 · HybridDB 数据合并的方法与原理
* MSSQL · 应用案例 · 构建死锁自动收集系统
* PostgreSQL · 实现分析 · PostgreSQL 10.0 并行查询和外部表的结合
* RocksDB · 特性介绍 · HashLinkList 内存表
* MySQL · myrocks · fast data load
* PgSQL · 应用案例 · "写入、共享、存储、计算" 最佳实践

 ## MySQL · 引擎特性 · InnoDB Buffer Pool 
 Author: 韩逸 

 ## 前言
用户对数据库的最基本要求就是能高效的读取和存储数据，但是读写数据都涉及到与低速的设备交互，为了弥补两者之间的速度差异，所有数据库都有缓存池，用来管理相应的数据页，提高数据库的效率，当然也因为引入了这一中间层，数据库对内存的管理变得相对比较复杂。本文主要分析MySQL Buffer Pool的相关技术以及实现原理，源码基于阿里云RDS MySQL 5.6分支，其中部分特性已经开源到AliSQL。Buffer Pool相关的源代码在buf目录下，主要包括LRU List，Flu List，Double write buffer, 预读预写，Buffer Pool预热，压缩页内存管理等模块，包括头文件和IC文件，一共两万行代码。

## 基础知识

#### Buffer Pool Instance:
大小等于innodb_buffer_pool_size/innodb_buffer_pool_instances，每个instance都有自己的锁，信号量，物理块(Buffer chunks)以及逻辑链表(下面的各种List)，即各个instance之间没有竞争关系，可以并发读取与写入。所有instance的物理块(Buffer chunks)在数据库启动的时候被分配，直到数据库关闭内存才予以释放。当innodb_buffer_pool_size小于1GB时候，innodb_buffer_pool_instances被重置为1，主要是防止有太多小的instance从而导致性能问题。每个Buffer Pool Instance有一个page hash链表，通过它，使用space_id和page_no就能快速找到已经被读入内存的数据页，而不用线性遍历LRU List去查找。注意这个hash表不是InnoDB的自适应哈希，自适应哈希是为了减少Btree的扫描，而page hash是为了避免扫描LRU List。

#### 数据页：
InnoDB中，数据管理的最小单位为页，默认是16KB，页中除了存储用户数据，还可以存储控制信息的数据。InnoDB IO子系统的读写最小单位也是页。如果对表进行了压缩，则对应的数据页称为压缩页，如果需要从压缩页中读取数据，则压缩页需要先解压，形成解压页，解压页为16KB。压缩页的大小是在建表的时候指定，目前支持16K，8K，4K，2K，1K。即使压缩页大小设为16K，在blob/varchar/text的类型中也有一定好处。假设指定的压缩页大小为4K，如果有个数据页无法被压缩到4K以下，则需要做B-tree分裂操作，这是一个比较耗时的操作。正常情况下，Buffer Pool中会把压缩和解压页都缓存起来，当Free List不够时，按照系统当前的实际负载来决定淘汰策略。如果系统瓶颈在IO上，则只驱逐解压页，压缩页依然在Buffer Pool中，否则解压页和压缩页都被驱逐。

#### Buffer Chunks:
包括两部分：数据页和数据页对应的控制体，控制体中有指针指向数据页。Buffer Chunks是最低层的物理块，在启动阶段从操作系统申请，直到数据库关闭才释放。通过遍历chunks可以访问几乎所有的数据页，有两种状态的数据页除外：没有被解压的压缩页(BUF_BLOCK_ZIP_PAGE)以及被修改过且解压页已经被驱逐的压缩页(BUF_BLOCK_ZIP_DIRTY)。此外数据页里面不一定都存的是用户数据，开始是控制信息，比如行锁，自适应哈希等。

#### 逻辑链表:
链表节点是数据页的控制体(控制体中有指针指向真正的数据页)，链表中的所有节点都有同一的属性，引入其的目的是方便管理。下面其中链表都是逻辑链表。

#### Free List:
其上的节点都是未被使用的节点，如果需要从数据库中分配新的数据页，直接从上获取即可。InnoDB需要保证Free List有足够的节点，提供给用户线程用，否则需要从FLU List或者LRU List淘汰一定的节点。InnoDB初始化后，Buffer Chunks中的所有数据页都被加入到Free List，表示所有节点都可用。

#### LRU List:
这个是InnoDB中最重要的链表。所有新读取进来的数据页都被放在上面。链表按照最近最少使用算法排序，最近最少使用的节点被放在链表末尾，如果Free List里面没有节点了，就会从中淘汰末尾的节点。LRU List还包含没有被解压的压缩页，这些压缩页刚从磁盘读取出来，还没来的及被解压。LRU List被分为两部分，默认前5/8为young list，存储经常被使用的热点page，后3/8为old list。新读入的page默认被加在old list头，只有满足一定条件后，才被移到young list上，主要是为了预读的数据页和全表扫描污染buffer pool。

#### FLU List:
这个链表中的所有节点都是脏页，也就是说这些数据页都被修改过，但是还没来得及被刷新到磁盘上。在FLU List上的页面一定在LRU List上，但是反之则不成立。一个数据页可能会在不同的时刻被修改多次，在数据页上记录了最老(也就是第一次)的一次修改的lsn，即oldest_modification。不同数据页有不同的oldest_modification，FLU List中的节点按照oldest_modification排序，链表尾是最小的，也就是最早被修改的数据页，当需要从FLU List中淘汰页面时候，从链表尾部开始淘汰。加入FLU List，需要使用flush_list_mutex保护，所以能保证FLU List中节点的顺序。

#### Quick List:
这个链表是阿里云RDS MySQL 5.6加入的，使用带Hint的SQL查询语句，可以把所有这个查询的用到的数据页加入到Quick List中，一旦这个语句结束，就把这个数据页淘汰，主要作用是避免LRU List被全表扫描污染。

#### Unzip LRU List:
这个链表中存储的数据页都是解压页，也就是说，这个数据页是从一个压缩页通过解压而来的。

#### Zip Clean List:
这个链表只在Debug模式下有，主要是存储没有被解压的压缩页。这些压缩页刚刚从磁盘读取出来，还没来的及被解压，一旦被解压后，就从此链表中删除，然后加入到Unzip LRU List中。

#### Zip Free:
压缩页有不同的大小，比如8K，4K，InnoDB使用了类似内存管理的伙伴系统来管理压缩页。Zip Free可以理解为由5个链表构成的一个二维数组，每个链表分别存储了对应大小的内存碎片，例如8K的链表里存储的都是8K的碎片，如果新读入一个8K的页面，首先从这个链表中查找，如果有则直接返回，如果没有则从16K的链表中分裂出两个8K的块，一个被使用，另外一个放入8K链表中。

## 核心数据结构
InnoDB Buffer Pool有三种核心的数据结构：buf_pool_t，buf_block_t，buf_page_t。

#### but_pool_t:
存储Buffer Pool Instance级别的控制信息，例如整个Buffer Pool Instance的mutex，instance_no, page_hash，old_list_pointer等。还存储了各种逻辑链表的链表根节点。Zip Free这个二维数组也在其中。

#### buf_block_t:
这个就是数据页的控制体，用来描述数据页部分的信息(大部分信息在buf_page_t中)。buf_block_t中第一字段就是buf_page_t，这个不是随意放的，是必须放在第一字段，因为只有这样buf_block_t和buf_page_t两种类型的指针可以相互转换。第二个字段是frame字段，指向真正存数据的数据页。buf_block_t还存储了Unzip LRU List链表的根节点。另外一个比较重要的字段就是block级别的mutex。

#### buf_page_t:
这个可以理解为另外一个数据页的控制体，大部分的数据页信息存在其中，例如space_id, page_no, page state, newest_modification，oldest_modification，access_time以及压缩页的所有信息等。压缩页的信息包括压缩页的大小，压缩页的数据指针(真正的压缩页数据是存储在由伙伴系统分配的数据页上)。这里需要注意一点，如果某个压缩页被解压了，解压页的数据指针是存储在buf_block_t的frame字段里。

这里介绍一下buf_page_t中的state字段，这个字段主要用来表示当前页的状态。一共有八种状态。这八种状态对初学者可能比较难理解，尤其是前三种，如果看不懂可以先跳过。

#### BUF_BLOCK_POOL_WATCH:
这种类型的page是提供给purge线程用的。InnoDB为了实现多版本，需要把之前的数据记录在undo log中，如果没有读请求再需要它，就可以通过purge线程删除。换句话说，purge线程需要知道某些数据页是否被读取，现在解法就是首先查看page hash，看看这个数据页是否已经被读入，如果没有读入，则获取(启动时候通过malloc分配，不在Buffer Chunks中)一个BUF_BLOCK_POOL_WATCH类型的哨兵数据页控制体，同时加入page_hash但是没有真正的数据(buf_blokc_t::frame为空)并把其类型置为BUF_BLOCK_ZIP_PAGE(表示已经被使用了，其他purge线程就不会用到这个控制体了)，相关函数`buf_pool_watch_set`，如果查看page hash后发现有这个数据页，只需要判断控制体在内存中的地址是否属于Buffer Chunks即可，如果是表示对应数据页已经被其他线程读入了，相关函数`buf_pool_watch_occurred`。另一方面，如果用户线程需要这个数据页，先查看page hash看看是否是BUF_BLOCK_POOL_WATCH类型的数据页，如果是则回收这个BUF_BLOCK_POOL_WATCH类型的数据页，从Free List中(即在Buffer Chunks中)分配一个空闲的控制体，填入数据。这里的核心思想就是通过控制体在内存中的地址来确定数据页是否还在被使用。

#### BUF_BLOCK_ZIP_PAGE:
当压缩页从磁盘读取出来的时候，先通过malloc分配一个临时的buf_page_t，然后从伙伴系统中分配出压缩页存储的空间，把磁盘中读取的压缩数据存入，然后把这个临时的buf_page_t标记为BUF_BLOCK_ZIP_PAGE状态(`buf_page_init_for_read`)，只有当这个压缩页被解压了，state字段才会被修改为BUF_BLOCK_FILE_PAGE，并加入LRU List和Unzip LRU List(`buf_page_get_gen`)。如果一个压缩页对应的解压页被驱逐了，但是需要保留这个压缩页且压缩页不是脏页，则这个压缩页被标记为BUF_BLOCK_ZIP_PAGE(`buf_LRU_free_page`)。所以正常情况下，处于BUF_BLOCK_ZIP_PAGE状态的不会很多。前述两种被标记为BUF_BLOCK_ZIP_PAGE的压缩页都在LRU List中。另外一个用法是，从BUF_BLOCK_POOL_WATCH类型节点中，如果被某个purge线程使用了，也会被标记为BUF_BLOCK_ZIP_PAGE。

#### BUF_BLOCK_ZIP_DIRTY:
如果一个压缩页对应的解压页被驱逐了，但是需要保留这个压缩页且压缩页是脏页，则被标记为BUF_BLOCK_ZIP_DIRTY(`buf_LRU_free_page`)，如果该压缩页又被解压了，则状态会变为BUF_BLOCK_FILE_PAGE。因此BUF_BLOCK_ZIP_DIRTY也是一个比较短暂的状态。这种类型的数据页都在Flush List中。

#### BUF_BLOCK_NOT_USED:
当链表处于Free List中，状态就为此状态。是一个能长期存在的状态。

#### BUF_BLOCK_READY_FOR_USE:
当从Free List中，获取一个空闲的数据页时，状态会从BUF_BLOCK_NOT_USED变为BUF_BLOCK_READY_FOR_USE(`buf_LRU_get_free_block`)，也是一个比较短暂的状态。处于这个状态的数据页不处于任何逻辑链表中。

#### BUF_BLOCK_FILE_PAGE:
正常被使用的数据页都是这种状态。LRU List中，大部分数据页都是这种状态。压缩页被解压后，状态也会变成BUF_BLOCK_FILE_PAGE。

#### BUF_BLOCK_MEMORY:
Buffer Pool中的数据页不仅可以存储用户数据，也可以存储一些系统信息，例如InnoDB行锁，自适应哈希索引以及压缩页的数据等，这些数据页被标记为BUF_BLOCK_MEMORY。处于这个状态的数据页不处于任何逻辑链表中

#### BUF_BLOCK_REMOVE_HASH:
当加入Free List之前，需要先把page hash移除。因此这种状态就表示此页面page hash已经被移除，但是还没被加入到Free List中，是一个比较短暂的状态。
总体来说，大部分数据页都处于BUF_BLOCK_NOT_USED(全部在Free List中)和BUF_BLOCK_FILE_PAGE(大部分处于LRU List中，LRU List中还包含除被purge线程标记的BUF_BLOCK_ZIP_PAGE状态的数据页)状态，少部分处于BUF_BLOCK_MEMORY状态，极少处于其他状态。前三种状态的数据页都不在Buffer Chunks上，对应的控制体都是临时分配的，InnoDB把他们列为invalid state(`buf_block_state_valid`)。
如果理解了这八种状态以及其之间的转换关系，那么阅读Buffer pool的代码细节就会更加游刃有余。

接下来，简单介绍一下buf_page_t中buf_fix_count和io_fix两个变量，这两个变量主要用来做并发控制，减少mutex加锁的范围。当从buffer pool读取一个数据页时候，会其加读锁，然后递增buf_page_t::buf_fix_count，同时设置buf_page_t::io_fix为BUF_IO_READ，然后即可以释放读锁。后续如果其他线程在驱逐数据页(或者刷脏)的时候，需要先检查一下这两个变量，如果buf_page_t::buf_fix_count不为零且buf_page_t::io_fix不为BUF_IO_NONE，则不允许驱逐(`buf_page_can_relocate`)。这里的技巧主要是为了减少数据页控制体上mutex的争抢，而对数据页的内容，读取的时候依然要加读锁，修改时加写锁。

## Buffer Pool内存初始化
Buffer Pool的内存初始化，主要是Buffer Chunks的内存初始化，buffer pool instance一个一个轮流初始化。核心函数为`buf_chunk_init`和`os_mem_alloc_large`
。阅读代码可以发现，目前从操作系统分配内存有两种方式，一种是通过HugeTLB的方式来分配，另外一种使用传统的mmap来分配。

#### HugeTLB:
这是一种大内存块的分配管理技术。类似数据库对数据的管理，内存也按照页来管理，默认的页大小为4KB，HugeTLB就是把页大小提高到2M或者更加多。程序传送给cpu都是虚拟内存地址，cpu必须通过快表来映射到真正的物理内存地址。快表的全集放在内存中，部分热点内存页可以放在cpu cache中，从而提高内存访问效率。假设cpu cache为100KB，每条快表占用1KB，页大小为4KB，则热点内存页为100KB/1KB=100条，覆盖100*4KB=400KB的内存数据，但是如果也默认页大小为2M，则同样大小的cpu cache，可以覆盖100*2M=200MB的内存数据，也就是说，访问200MB的数据只需要一次读取内存即可(如果映射关系没有在cache中找到，则需要先把映射关系从内存中读到cache，然后查找，最后再去读内存中需要的数据，会造成两次访问物理内存)。也就是说，使用HugeTLB这种大内存技术，可以提高快表的命中率，从而提高访问内存的性能。当然这个技术也不是银弹，内存页变大了也必定会导致更多的页内的碎片。如果需要从swap分区中加载虚拟内存，也会变慢。当然最终要的理由是，4KB大小的内存页已经被业界稳定使用很多年了，如果没有特殊的需求不需要冒这个风险。在InnoDB中，如果需要用到这项技术可以使用super-large-pages参数启动MySQL。

#### mmap分配：
在Linux下，多个进程需要共享一片内存，可以使用mmap来分配和绑定，所以只提供给一个MySQL进程使用也是可以的。用mmap分配的内存都是虚存，在top命令中占用VIRT这一列，而不是RES这一列，只有相应的内存被真正使用到了，才会被统计到RES中，提高内存使用率。这样是为什么常常看到MySQL一启动就被分配了很多的VIRT，而RES却是慢慢涨上来的原因。这里大家可能有个疑问，为啥不用malloc。其实查阅malloc文档，可以发现，当请求的内存数量大于MMAP_THRESHOLD(默认为128KB)时候，malloc底层就是调用了mmap。在InnoDB中，默认使用mmap来分配。
分配完了内存，`buf_chunk_init`函数中，把这片内存划分为两个部分，前一部分是数据页控制体(buf_block_t)，在阿里云RDS MySQL 5.6 release版本中，每个buf_block_t是424字节，一共有innodb_buffer_pool_size/UNIV_PAGE_SIZE个。后一部分是真正的数据页，按照UNIV_PAGE_SIZE分隔。假设page大小为16KB，则数据页控制体占的内存:数据页约等于1:38.6，也就是说如果innodb_buffer_pool_size被配置为40G，则需要额外的1G多空间来存数据页的控制体。
划分完空间后，遍历数据页控制体，设置buf_block_t::frame指针，指向真正的数据页，然后把这些数据页加入到Free List中即可。初始化完Buffer Chunks的内存，还需要初始化BUF_BLOCK_POOL_WATCH类型的数据页控制块，page hash的结构体，zip hash的结构体(所有被压缩页的伙伴系统分配走的数据页面会加入到这个哈希表中)。注意这些内存是额外分配的，不包含在Buffer Chunks中。
除了`buf_pool_init`外，建议读者参考一下`but_pool_free`这个内存释放函数，加深对Buffer Pool相关内存的理解。

## Buf_page_get函数解析
这个函数极其重要，是其他模块获取数据页的外部接口函数。如果请求的数据页已经在Buffer Pool中了，修改相应信息后，就直接返回对应数据页指针，如果Buffer Pool中没有相关数据页，则从磁盘中读取。`Buf_page_get`是一个宏定义，真正的函数为`buf_page_get_gen`，参数主要为space_id, page_no, lock_type, mode以及mtr。这里主要介绍一个mode这个参数，其表示读取的方式，目前支持六种，前三种用的比较多。

#### BUF_GET:
默认获取数据页的方式，如果数据页不在Buffer Pool中，则从磁盘读取，如果已经在Buffer Pool中，需要判断是否要把他加入到young list中以及判断是否需要进行线性预读。如果是读取则加读锁，修改则加写锁。

#### BUF_GET_IF_IN_POOL:
只在Buffer Pool中查找这个数据页，如果在则判断是否要把它加入到young list中以及判断是否需要进行线性预读。如果不在则直接返回空。加锁方式与BUF_GET类似。

#### BUF_PEEK_IF_IN_POOL:
与BUF_GET_IF_IN_POOL类似，只是即使条件满足也不把它加入到young list中也不进行线性预读。加锁方式与BUF_GET类似。

#### BUF_GET_NO_LATCH:
不管对数据页是读取还是修改，都不加锁。其他方面与BUF_GET类似。

#### BUF_GET_IF_IN_POOL_OR_WATCH:
只在Buffer Pool中查找这个数据页，如果在则判断是否要把它加入到young list中以及判断是否需要进行线性预读。如果不在则设置watch。加锁方式与BUF_GET类似。这个是要是给purge线程用。

#### BUF_GET_POSSIBLY_FREED:
这个mode与BUF_GET类似，只是允许相应的数据页在函数执行过程中被释放，主要用在估算Btree两个slot之前的数据行数。
接下来，我们简要分析一下这个函数的主要逻辑。

* 首先通过`buf_pool_get`函数依据space_id和page_no查找指定的数据页在那个Buffer Pool Instance里面。算法很简单`instance_no = (space_id << 20 + space_id + page_no >> 6) % instance_num`，也就是说先通过space_id和page_no算出一个fold value然后按照instance的个数取余数即可。这里有个小细节，page_no的第六位被砍掉，这是为了保证一个extent的数据能被缓存到同一个Buffer Pool Instance中，便于后面的预读操作。
* 接着，调用`buf_page_hash_get_low`函数在page hash中查找这个数据页是否已经被加载到对应的Buffer Pool Instance中，如果没有找到这个数据页且mode为BUF_GET_IF_IN_POOL_OR_WATCH则设置watch数据页(`buf_pool_watch_set`)，接下来，如果没有找到数据页且mode为BUF_GET_IF_IN_POOL、BUF_PEEK_IF_IN_POOL或者BUF_GET_IF_IN_POOL_OR_WATCH函数直接返回空，表示没有找到数据页。如果没有找到数据但是mode为其他，就从磁盘中同步读取(`buf_read_page`)。在读取磁盘数据之前，我们如果发现需要读取的是非压缩页，则先从Free List中获取空闲的数据页，如果Free List中已经没有了，则需要通过刷脏来释放数据页，这里的一些细节我们后续在LRU模块再分析，获取到空闲的数据页后，加入到LRU List中(`buf_page_init_for_read`)。在读取磁盘数据之前，我们如果发现需要读取的是压缩页，则临时分配一个buf_page_t用来做控制体，通过伙伴系统分配到压缩页存数据的空间，最后同样加入到LRU List中(`buf_page_init_for_read`)。做完这些后，我们就调用IO子系统的接口同步读取页面数据，如果读取数据失败，我们重试100次(`BUF_PAGE_READ_MAX_RETRIES`)然后触发断言，如果成功则判断是否要进行随机预读(随机预读相关的细节我们也在预读预写模块分析)。
* 接着，读取数据成功后，我们需要判断读取的数据页是不是压缩页，如果是的话，因为从磁盘中读取的压缩页的控制体是临时分配的，所以需要重新分配block(`buf_LRU_get_free_block`)，把临时分配的buf_page_t给释放掉，用`buf_relocate`函数替换掉，接着进行解压，解压成功后，设置state为BUF_BLOCK_FILE_PAGE，最后加入Unzip LRU List中。
* 接着，我们判断这个页是否是第一次访问，如果是则设置buf_page_t::access_time，如果不是，我们则判断其是不是在Quick List中，如果在Quick List中且当前事务不是加过Hint语句的事务，则需要把这个数据页从Quick List删除，因为这个页面被其他的语句访问到了，不应该在Quick List中了。
* 接着，如果mode不为BUF_PEEK_IF_IN_POOL，我们需要判断是否把这个数据页移到young list中，具体细节在后面LRU模块中分析。
* 接着，如果mode不为BUF_GET_NO_LATCH，我们给数据页加上读写锁。
* 最后，如果mode不为BUF_PEEK_IF_IN_POOL且这个数据页是第一次访问，则判断是否需要进行线性预读(线性预读相关的细节我们也在预读预写模块分析)。

## LRU List中young list和old list的维护
当LRU List链表大于512(`BUF_LRU_OLD_MIN_LEN`)时，在逻辑上被分为两部分，前面部分存储最热的数据页，这部分链表称作young list，后面部分则存储冷数据页，这部分称作old list，一旦Free List中没有页面了，就会从冷页面中驱逐。两部分的长度由参数innodb_old_blocks_pct控制。每次加入或者驱逐一个数据页后，都要调整young list和old list的长度(`buf_LRU_old_adjust_len`)，同时引入`BUF_LRU_OLD_TOLERANCE`来防止链表调整过频繁。当LRU List链表小于512，则只有old list。
新读取进来的页面默认被放在old list头，在经过innodb_old_blocks_time后，如果再次被访问了，就挪到young list头上。一个数据页被读入Buffer Pool后，在小于innodb_old_blocks_time的时间内被访问了很多次，之后就不再被访问了，这样的数据页也很快被驱逐。这个设计认为这种数据页是不健康的，应该被驱逐。
此外，如果一个数据页已经处于young list，当它再次被访问的时候，不会无条件的移动到young list头上，只有当其处于young list长度的1/4(大约值)之后，才会被移动到young list头部，这样做的目的是减少对LRU List的修改，否则每访问一个数据页就要修改链表一次，效率会很低，因为LRU List的根本目的是保证经常被访问的数据页不会被驱逐出去，因此只需要保证这些热点数据页在头部一个可控的范围内即可。相关逻辑可以参考函数`buf_page_peek_if_too_old`。

## buf_LRU_get_free_block函数解析
这个函数以及其调用的函数可以说是整个LRU模块最重要的函数，在整个Buffer Pool模块中也有举足轻重的作用。如果能把这几个函数吃透，相信其他函数很容易就能读懂。

* 首先，如果是使用ENGINE_NO_CACHE发送过来的SQL需要读取数据，则优先从Quick List中获取(`buf_quick_lru_get_free`)。
* 接着，统计Free List和LRU List的长度，如果发现他们再Buffer Chunks占用太少的空间，则表示太多的空间被行锁，自使用哈希等内部结构给占用了，一般这些都是大事务导致的。这时候会给出报警。
* 接着，查看Free List中是否还有空闲的数据页(`buf_LRU_get_free_only`)，如果有则直接返回，否则进入下一步。大多数情况下，这一步都能找到空闲的数据页。
* 如果Free List中已经没有空闲的数据页了，则会尝试驱逐LRU List末尾的数据页。如果系统有压缩页，情况就有点复杂，InnoDB会调用`buf_LRU_evict_from_unzip_LRU`来决定是否驱逐压缩页，如果Unzip LRU List大于LRU List的十分之一或者当前InnoDB IO压力比较大，则会优先从Unzip LRU List中把解压页给驱逐，否则会从LRU List中把解压页和压缩页同时驱逐。不管走哪条路径，最后都调用了函数`buf_LRU_free_page`来执行驱逐操作，这个函数由于要处理压缩页解压页各种情况，极其复杂。大致的流程：首先判断是否是脏页，如果是则不驱逐，否则从LRU List中把链表删除，必要的话还从Unzip LRU List移走这个数据页(`buf_LRU_block_remove_hashed`)，接着如果我们选择保留压缩页，则需要重新创建一个压缩页控制体，插入LRU List中，如果是脏的压缩页还要插入到Flush List中，最后才把删除的数据页插入到Free List中(`buf_LRU_block_free_hashed_page`)。
* 如果在上一步中没有找到空闲的数据页，则需要刷脏了(`buf_flush_single_page_from_LRU`)，由于buf_LRU_get_free_block这个函数是在用户线程中调用的，所以即使要刷脏，这里也是刷一个脏页，防止刷过多的脏页阻塞用户线程。
* 如果上一步的刷脏因为数据页被其他线程读取而不能刷脏，则重新跳转到上述第二步。进行第二轮迭代，与第一轮迭代的区别是，第一轮迭代在扫描LRU List时，最多只扫描innodb_lru_scan_depth个，而在第二轮迭代开始，扫描整个LRU List。如果很不幸，这一轮还是没有找到空闲的数据页，从三轮迭代开始，在刷脏前等待10ms。
* 最终找到一个空闲页后，page的state为BUF_BLOCK_READY_FOR_USE。

## 控制全表扫描不增加cache数据到Buffer Pool
全表扫描对Buffer Pool的影响比较大，即使有old list作用，但是old list默认也占Buffer Pool的3/8。因此，阿里云RDS引入新的语法ENGINE_NO_CACHE(例如：SELECT ENGINE_NO_CACHE count(*) FROM t1)。如果一个SQL语句中带了ENGINE_NO_CACHE这个关键字，则由它读入内存的数据页都放入Quick List中，当这个语句结束时，会删除它独占的数据页。同时引入两个参数。innodb_rds_trx_own_block_max这个参数控制使用Hint的每个事物最多能拥有多少个数据页，如果超过这个数据就开始驱逐自己已有的数据页，防止大事务占用过多的数据页。innodb_rds_quick_lru_limit_per_instance这个参数控制每个Buffer Pool Instance中Quick List的长度，如果超过这个长度，后续的请求都从Quick List中驱逐数据页，进而获取空闲数据页。

## 删除指定表空间所有的数据页
函数(`buf_LRU_remove_pages`)提供了三种模式，第一种(`BUF_REMOVE_ALL_NO_WRITE`)，删除Buffer Pool中所有这个类型的数据页(LRU List和Flush List)同时Flush List中的数据页也不写回数据文件，这种适合rename table和5.6表空间传输新特性，因为space_id可能会被复用，所以需要清除内存中的一切，防止后续读取到错误的数据。第二种(`BUF_REMOVE_FLUSH_NO_WRITE`)，仅仅删除Flush List中的数据页同时Flush List中的数据页也不写回数据文件，这种适合drop table，即使LRU List中还有数据页，但由于不会被访问到，所以会随着时间的推移而被驱逐出去。第三种(`BUF_REMOVE_FLUSH_WRITE`)，不删除任何链表中的数据仅仅把Flush List中的脏页都刷回磁盘，这种适合表空间关闭，例如数据库正常关闭的时候调用。这里还有一点值得一提的是，由于对逻辑链表的变动需要加锁且删除指定表空间数据页这个操作是一个大操作，容易造成其他请求被饿死，所以InnoDB做了一个小小的优化，每删除BUF_LRU_DROP_SEARCH_SIZE个数据页(默认为1024)就会释放一下Buffer Pool Instance的mutex，便于其他线程执行。

## LRU_Manager_Thread
这是一个系统线程，随着InnoDB启动而启动，作用是定期清理出空闲的数据页(数量为innodb_LRU_scan_depth)并加入到Free List中，防止用户线程去做同步刷脏影响效率。线程每隔一定时间去做BUF_FLUSH_LRU，即首先尝试从LRU中驱逐部分数据页，如果不够则进行刷脏，从Flush List中驱逐(`buf_flush_LRU_tail`)。线程执行的频率通过以下策略计算：我们设定`max_free_len = innodb_LRU_scan_depth * innodb_buf_pool_instances`，如果Free List中的数量小于max_free_len的1%，则sleep time为零，表示这个时候空闲页太少了，需要一直执行buf_flush_LRU_tail从而腾出空闲的数据页。如果Free List中的数量介于max_free_len的1%-5%，则sleep time减少50ms(默认为1000ms)，如果Free List中的数量介于max_free_len的5%-20%，则sleep time不变，如果Free List中的数量大于max_free_len的20%，则sleep time增加50ms，但是最大值不超过`rds_cleaner_max_lru_time`。这是一个自适应的算法，保证在大压力下有足够用的空闲数据页(`lru_manager_adapt_sleep_time`)。

## Hazard Pointer
在学术上，Hazard Pointer是一个指针，如果这个指针被一个线程所占有，在它释放之前，其他线程不能对他进行修改，但是在InnoDB里面，概念刚好相反，一个线程可以随时访问Hazard Pointer，但是在访问后，他需要调整指针到一个有效的值，便于其他线程使用。我们用Hazard Pointer来加速逆向的逻辑链表遍历。
先来说一下这个问题的背景，我们知道InnoDB中可能有多个线程同时作用在Flush List上进行刷脏，例如LRU_Manager_Thread和Page_Cleaner_Thread。同时，为了减少锁占用的时间，InnoDB在进行写盘的时候都会把之前占用的锁给释放掉。这两个因素叠加在一起导致同一个刷脏线程刷完一个数据页A，就需要回到Flush List末尾(因为A之前的脏页可能被其他线程给刷走了，之前的脏页可能已经不在Flush list中了)，重新扫描新的可刷盘的脏页。另一方面，数据页刷盘是异步操作，在刷盘的过程中，我们会把对应的数据页IO_FIX住，防止其他线程对这个数据页进行操作。我们假设某台机器使用了非常缓慢的机械硬盘，当前Flush List中所有页面都可以被刷盘(`buf_flush_ready_for_replace`返回true)。我们的某一个刷脏线程拿到队尾最后一个数据页，IO fixed，发送给IO线程，最后再从队尾扫描寻找可刷盘的脏页。在这次扫描中，它发现最后一个数据页(也就是刚刚发送到IO线程中的数据页)状态为IO fixed(磁盘很慢，还没处理完)所以不能刷，跳过，开始刷倒数第二个数据页，同样IO fixed，发送给IO线程，然后再次重新扫描Flush List。它又发现尾部的两个数据页都不能刷新(因为磁盘很慢，可能还没刷完)，直到扫描到倒数第三个数据页。所以，存在一种极端的情况，如果磁盘比较缓慢，刷脏算法性能会从O(N)退化成O(N*N)。
要解决这个问题，最本质的方法就是当刷完一个脏页的时候不要每次都从队尾重新扫描。我们可以使用Hazard Pointer来解决，方法如下：遍历找到一个可刷盘的数据页，在锁释放之前，调整Hazard Pointer使之指向Flush List中下一个节点，注意一定要在持有锁的情况下修改。然后释放锁，进行刷盘，刷完盘后，重新获取锁，读取Hazard Pointer并设置下一个节点，然后释放锁，进行刷盘，如此重复。当这个线程在刷盘的时候，另外一个线程需要刷盘，也是通过Hazard Pointer来获取可靠的节点，并重置下一个有效的节点。通过这种机制，保证每次读到的Hazard Pointer是一个有效的Flush List节点，即使磁盘再慢，刷脏算法效率依然是O(N)。
这个解法同样可以用到LRU List驱逐算法上，提高驱逐的效率。相应的Patch是在MySQL 5.7上首次提出的，阿里云RDS把其Port到了我们5.6的版本上，保证在大并发情况下刷脏算法的效率。

## Page_Cleaner_Thread
这也是一个InnoDB的后台线程，主要负责Flush List的刷脏，避免用户线程同步刷脏页。与LRU_Manager_Thread线程相似，其也是每隔一定时间去刷一次脏页。其sleep time也是自适应的(`page_cleaner_adapt_sleep_time`)，主要由三个因素影响：当前的lsn，Flush list中的oldest_modification以及当前的同步刷脏点(`log_sys->max_modified_age_sync`，有redo log的大小和数量决定)。简单的来说，lsn - oldest_modification的差值与同步刷脏点差距越大，sleep time就越长，反之sleep time越短。此外，可以通过`rds_page_cleaner_adaptive_sleep`变量关闭自适应sleep time，这是sleep time固定为1秒。
与LRU_Manager_Thread每次固定执行清理innodb_LRU_scan_depth个数据页不同，Page_Cleaner_Thread每次执行刷的脏页数量也是自适应的，计算过程有点复杂(`page_cleaner_flush_pages_if_needed`)。其依赖当前系统中脏页的比率，日志产生的速度以及几个参数。innodb_io_capacity和innodb_max_io_capacity控制每秒刷脏页的数量，前者可以理解为一个soft limit，后者则为hard limit。innodb_max_dirty_pages_pct_lwm和innodb_max_dirty_pages_pct_lwm控制脏页比率，即InnoDB什么脏页到达多少才算多了，需要加快刷脏频率了。innodb_adaptive_flushing_lwm控制需要刷新到哪个lsn。innodb_flushing_avg_loops控制系统的反应效率，如果这个变量配置的比较大，则系统刷脏速度反应比较迟钝，表现为系统中来了很多脏页，但是刷脏依然很慢，如果这个变量配置很小，当系统中来了很多脏页后，刷脏速度在很短的时间内就可以提升上去。这个变量是为了让系统运行更加平稳，起到削峰填谷的作用。相关函数，`af_get_pct_for_dirty`和`af_get_pct_for_lsn`。

## 预读和预写
如果一个数据页被读入Buffer Pool，其周围的数据页也有很大的概率被读入内存，与其分开多次读取，还不如一次都读入内存，从而减少磁盘寻道时间。在官方的InnoDB中，预读分两种，随机预读和线性预读。

#### 随机预读:
这种预读发生在一个数据页成功读入Buffer Pool的时候(`buf_read_ahead_random`)。在一个Extent范围(1M，如果数据页大小为16KB，则为连续的64个数据页)内，如果热点数据页大于一定数量，就把整个Extend的其他所有数据页(依据page_no从低到高遍历读入)读入Buffer Pool。这里有两个问题，首先数量是多少，默认情况下，是13个数据页。接着，怎么样的页面算是热点数据页，阅读代码发现，只有在young list前1/4的数据页才算是热点数据页。读取数据时候，使用了异步IO，结合使用`OS_AIO_SIMULATED_WAKE_LATER`和`os_aio_simulated_wake_handler_threads`便于IO合并。随机预读可以通过参数innodb_random_read_ahead来控制开关。此外，`buf_page_get_gen`函数的mode参数不影响随机预读。

#### 线性预读:
这中预读只发生在一个边界的数据页(Extend中第一个数据页或者最后一个数据页)上(`buf_read_ahead_linear`)。在一个Extend范围内，如果大于一定数量(通过参数innodb_read_ahead_threshold控制，默认为56)的数据页是被顺序访问(通过判断数据页access time是否为升序或者逆序来确定)的，则把下一个Extend的所有数据页都读入Buffer Pool。读取的时候依然采用异步IO和IO合并策略。线性预读触发的条件比较苛刻，触发操作的是边界数据页同时要求其他数据页严格按照顺序访问，主要是为了解决全表扫描时的性能问题。线性预读可以通过参数`innodb_read_ahead_threshold`来控制开关。此外，当`buf_page_get_gen`函数的mode为BUF_PEEK_IF_IN_POOL时，不触发线性预读。
InnoDB中除了有预读功能，在刷脏页的时候，也能进行预写(`buf_flush_try_neighbors`)。当一个数据页需要被写入磁盘的时候，查找其前面或者后面邻居数据页是否也是脏页且可以被刷盘(没有被IOFix且在old list中)，如果可以的话，一起刷入磁盘，减少磁盘寻道时间。预写功能可以通过`innodb_flush_neighbors`参数来控制。不过在现在的SSD磁盘下，这个功能可以关闭。

## Double Write Buffer(dblwr)
服务器突然断电，这个时候如果数据页被写坏了(例如数据页中的目录信息被损坏)，由于InnoDB的redolog日志不是完全的物理日志，有部分是逻辑日志，因此即使奔溃恢复也无法恢复到一致的状态，只能依靠Double Write Buffer先恢复完整的数据页。Double Write Buffer主要是解决数据页半写的问题，如果文件系统能保证写数据页是一个原子操作，那么可以把这个功能关闭，这个时候每个写请求直接写到对应的表空间中。
Double Write Buffer大小默认为2M，即128个数据页。其中分为两部分，一部分留给batch write，另一部分是single page write。前者主要提供给批量刷脏的操作，后者留给用户线程发起的单页刷脏操作。batch write的大小可以由参数`innodb_doublewrite_batch_size`控制，例如假设innodb_doublewrite_batch_size配置为120，则剩下8个数据页留给single page write。
假设我们要进行批量刷脏操作，我们会首先写到内存中的Double Write Buffer(也是2M，在系统初始化中分配，不使用Buffer Chunks空间)，如果dblwr写满了，一次将其中的数据刷盘到系统表空间指定位置，注意这里是同步IO操作，在确保写入成功后，然后使用异步IO把各个数据页写回自己的表空间，由于是异步操作，所有请求下发后，函数就返回，表示写成功了(`buf_dblwr_add_to_batch`)。不过这个时候后续的写请求依然会阻塞，知道这些异步操作都成功，才清空系统表空间上的内容，后续请求才能被继续执行。这样做的目的就是，如果在异步写回数据页的时候，系统断电，发生了数据页半写，这个时候由于系统表空间中的数据页是完整的，只要从中拷贝过来就行(`buf_dblwr_init_or_load_pages`)。
异步IO请求完成后，会检查数据页的完整性以及完成change buffer相关操作，接着IO helper线程会调用`buf_flush_write_complete`函数，把数据页从Flush List删除，如果发现batch write中所有的数据页都写成了，则释放dblwr的空间。

## Buddy伙伴系统
与内存分配管理算法类似，InnoDB中的伙伴系统也是用来管理不规则大小内存分配的，主要用在压缩页的数据上。前文提到过，InnoDB中的压缩页可以有16K，8K，4K，2K，1K这五种大小，压缩页大小的单位是表，也就是说系统中可能存在很多压缩页大小不同的表。使用伙伴体统来分配和回收，能提高系统的效率。
申请空间的函数是`buf_buddy_alloc`，其首先在zip free链表中查看指定大小的块是否还存在，如果不存在则从更大的链表中分配，这回导致一些列的分裂操作。例如需要一块4K大小的内存，则先从4K链表中查找，如果有则直接返回，没有则从8K链表中查找，如果8K中还有空闲的，则把8K分成两部分，低地址的4K提供给用户，高地址的4K插入到4K的链表中，便与后续使用。如果8K中也没有空闲的了，就从16K中分配，16K首先分裂成2个8K，高地址的插入到8K链表中，低地址的8K继续分裂成2个4K，低地址的4K返回给用户，高地址的4K插入到4K的链表中。假设16K的链表中也没有空闲的了，则调用`buf_LRU_get_free_block`获取新的数据页，然后把这个数据页加入到zip hash中，同时设置state状态为BUF_BLOCK_MEMORY，表示这个数据页存储了压缩页的数据。
释放空间的函数是`buf_buddy_free`，相比于分配空间的函数，有点复杂。假设释放一个4K大小的数据块，其先把4K放回4K对应的链表，接着会查看其伙伴(释放块是低地址，则伙伴是高地址，释放块是高地址，则伙伴是低地址)是否也被释放了，如果也被释放了则合并成8K的数据块，然后继续寻找这个8K数据块的伙伴，试图合并成16K的数据块。如果发现伙伴没有被释放，函数并不会直接退出而是把这个伙伴给挪走(`buf_buddy_relocate`)，例如8K数据块的伙伴没有被释放，系统会查看8K的链表，如果有空闲的8K块，则把这个伙伴挪到这个空闲的8K上，这样就能合并成16K的数据块了，如果没有，函数才放弃合并并返回。通过这种relocate操作，内存碎片会比较少，但是涉及到内存拷贝，效率会比较低。

## Buffer Pool预热
这个也是官方5.6提供的新功能，可以把当前Buffer Pool中的数据页按照space_id和page_no dump到外部文件，当数据库重启的时候，Buffer Pool就可以直接恢复到关闭前的状态。

#### Buffer Pool Dump:
遍历所有Buffer Pool Instance的LRU List，对于其中的每个数据页，按照space_id和page_no组成一个64位的数字，写到外部文件中即可(`buf_dump`)。

#### Buffer Pool Load:
读取指定的外部文件，把所有的数据读入内存后，使用归并排序对数据排序，以64个数据页为单位进行IO合并，然后发起一次真正的读取操作。排序的作用就是便于IO合并(`buf_load`)。

## 总结
InnoDB的Buffer Pool可以认为很简单，就是LRU List和Flush List，但是InnoDB对其做了很多性能上的优化，例如减少加锁范围，page hash加速查找等，导致具体的实现细节相对比较复杂，尤其是引入压缩页这个特性后，有些核心代码变得晦涩难懂，需要读者细细琢磨。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)