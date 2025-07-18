# MySQL · 引擎特性· InnoDB之UNDO LOG介绍

**Date:** 2021/12
**Source:** http://mysql.taobao.org/monthly/2021/12/02/
**Images:** 13 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2021 / 12
 ](/monthly/2021/12)

 * 当期文章

 PostgreSQL · 引擎特性 · PostgreSQL 14 新特性浅析
* MySQL · 引擎特性· InnoDB之UNDO LOG介绍
* PolarDB · 引擎特性 · Nonblock add column
* PolarDB · 引擎特性 · B-tree 并发控制优化
* PostgreSQL · 引擎特性 · PostgreSQL 14 新特性浅析
* PolarDB · 引擎特性 · Nonblock add column
* MySQL · 引擎特性· InnoDB之UNDO LOG介绍
* PolarDB · 引擎特性 · B-tree 并发控制优化

 ## MySQL · 引擎特性· InnoDB之UNDO LOG介绍 
 Author: bayan.xj 

 本文基于MySQL Community 8.0.23 Version

undo log是InnoDB事务特性的重要组成部分。当对记录做增删改操作就会产生undo记录，undo记录会记录到单独的表空间中。

本文将从代码层面对undo log进行一个简单的介绍；主要从下面四个方面来介绍undo log：undo log组织形式与分配与记录，以及undo log的应用及其清理。从这四个方面出发，我们就可以基本了解undo log的整个生命周期。

### undo log的组织形式

此部分是关于Undo log的组织形式的一个介绍；主要分为两部分来对undo log的组织形式进行介绍：文件结构和内存结构。在介绍这两部分时，先从局部出发，最后再给出各个部分的联系。

### 1. 文件结构

首先，在MySQL5.6之前所有的undo log全部存储在系统表空间中(ibdata1)；但是从5.6开始也可以使用独立表空间来存储undo log。

当前版本InnoDB默认有两个undo tablespace，也可以使用`CREATE UNDO TABLESPACE`语句动态添加，最大128个；每个undo tablespace至多可以有`TRX_SYS_N_RSEGS(128)`个回滚段。

#### 1.1 Rollback Segment(回滚段)

InnoDB在undo tablespace中使用回滚段来组织undo log。同时为了保证事务的并发操作，在写undo log时不产生冲突，InnoDB使用 回滚段 来维护undo log的并发写入和持久化；而每个回滚段 又有多个undo log slot。通常通过Rollback Segment Header来管理回滚段，Rollback Segment Header通常在回滚段的第一个页，具体结构如下：

![回滚段结构](.img/43809f819633_rollback_segment_header.png)

* Max Size：参数名为 TRX_RSEG_MAX_SIZE，回滚段可以有用的最大page数。
* History Size：参数名为TRX_RSEG_HISTORY_SIZE，history list包含的page数。
* History List Base Node：参数名为TRX_RSEG_HISTORY，history list的Base Node。
* Rollback Segment FSEG Entry：参数名为TRX_RSEG_FSEG_HEADER，file segment的存放位置。
* Undo Slots Dictionary：参数名为TRX_RSEG_UNDO_SLOTS，存放活跃事务的undo header page no。

Rollback Segment Header里面最重要的两部分就是**history list**与**undo slot directory**。

其中history list把所有已经提交但还没有被purge事务的undo log串联起来，purge线程可以通过此list对没有事务使用的undo log进行purge。

每个事务在需要记录undo log时都会申请一个或两个`slot`(insert/update分开)，同时把事务的第一个undo page放入对应slot中；所以理论上InnoDB允许的最大事务并发数为128(`undo tablespace`) * 128(`Rollback Segment`) * 1024(`TRX_RSEG_UNDO_SLOTS`)。下面我们进一步介绍undo log在磁盘上如何记录。

#### 1.2 UndoPage

要想知道undo log如何记录，我们先要搞清楚一个undo page具体内容，undo page一般分两种情况：header page和normal page 。

header page除了normal page所包含的信息，还包含一些undo segment信息，后面会对undo segment进行详细介绍。

我们下面先介绍一下undo header page的详细分布。

![undo log](.img/ffb618b1975a_undo_log.png)

**undo header page**是事务需要写undo log时申请的第一个undo page；一个undo header page他同一时刻只隶属于同一个活跃事务，但是一个undo header page上面的内容可能包含多个已经提交的事务和一个活跃事务。

**undo normal page**是当活跃事务产生的undo record超过undo header page容量后，单独再为此事务分配的undo page(参考函数trx_undo_add_page)；此page只隶属于一个事务，只包含undo page header不包含undo segment header。

#### 1.3 Undo Page Header

每一个undo page都要有header，其中记录了当前undo page的一些状态信息，具体内容如下：

![undo page header](.img/31bf176a86fd_undo_page_header.png)

* Undo Page Type：参数名为TRX_UNDO_PAGE_TYPE，使用该page事务的类型，包含TRX_UNDO_INSERT，TRX_UNDO_UPDATE两种。
* Latest Log Record Offset：参数名为TRX_UNDO_PAGE_START，最新事务开始记录undo log起始位置。
* Free Space Offset：参数名为TRX_UNDO_PAGE_FREE，页内空闲空间起始地址，在此之后可记录undo log。
* Undo Page List Node：参数名为TRX_UNDO_PAGE_NODE，undo page list节点，可以把同一个事务所用到的所有undo page双向串联起来。

#### 1.4 Undo Segment Header

![undo segment header](.img/737765c7d9e0_undo_segment_header.png)

* State：参数名为TRX_UNDO_STATE，undo segment的状态，TRX_UNDO_ACTIVE等
* Last Log Offset：参数名为TRX_UNDO_LAST_LOG，当前page最后一个undo log header的位置。
* Undo Segment FSEG Entry：参数名为TRX_UNDO_FSEG_HEADER，segment对应的inode的（space_id，page_no，offset等）
* Undo Segment Page List Base Node：参数名为TRX_UNDO_PAGE_LIST，undo page list的Base Node，对于同一个事务下的undo header page和undo normal page构成双向链表。

上面只是介绍了一些undo log在文件上的基本结构，下面我们继续介绍记录undo log时的文件组织。

#### 1.5 Undo Log Header

当事务开始记录undo log时，先创建一个undo log header，当update/delete事务结束后，undo log header将会被加入到hisotry list中；insert事务的undo log会被立即释放。

![undo log header](.img/44b2b0c70b2c_undo_log_header.png)

* Transaction ID：参数名为TRX_UNDO_TRX_ID，事务id（事务开始的逻辑时间）
* Transaction Number：参数名为TRX_UNDO_TRX_NO，事务no（事务结束的逻辑时间）
* Delete Marks Flags：参数名为TRX_UNDO_DEL_MARKS，如果涉及到删除记录为TRUE
* Log Start Offset：参数名为TRX_UNDO_LOG_START，事务中第一个undo record地址。
* XID Flag：参数名为TRX_UNDO_XID_EXISTS，用于XID。
* DDL Transaction Flag：参数名为TRX_UNDO_DICT_TRANS，是否是DDL事务
* Table ID if DDL Transaction：参数名为TRX_UNDO_TABLE_ID，如果是DDL事务，记录table id。
* Next Undo Log Offset：参数名为TRX_UNDO_NEXT_LOG，当前页的下一个undo log header位置
* Prev Undo Log Offset：参数名为TRX_UNDO_PREV_LOG，当前页的上一个undo log header位置
* History List Node：参数名为TRX_UNDO_HISTORY_NODE，事务结束时放入history list的节点。

有关XID的内容暂时不介绍。

有了undo log header后，我们就可以记录undo record了。

#### 1.6 Undo Record

![undo record](.img/3828a1539f40_undo_record.png)

* Previous Record Offset：存储上一条record的位置。
* Next Record Offset：存储下一条record的位置。
* Type + Extern Flag + Compilation Info：存undo record的type等信息。

可以从上面图中看出，undo record除了存储这里比较重要的几个信息，包含前后undo record位置，类型，undo no，table id等；而undo recod中具体存储的内容我们等到《undo log的分配与记录》中去介绍。

最后，我们通过下面这幅图来了解undo log在文件组织上的一个总览。

![undo log 文件结构总览](.img/ff9aa564d8f9_undo_log_disk_structure.png)

### 2. 主要内存数据结构

为了方便管理和记录undo log，在内存中有如下关键结构体对象：

* undo::Tablespace：undo tablespace内存结构体，维护undo tablespace相关信息，管理此tablespace中相关回滚段。
* trx_rseg_t：回滚段的内存结构体，用于维护回滚段相关信息。
* trx_undo_t：undo log内存结构体，用于维护undo log type等信息，便于对undo page进行维护和管理。
* purge_pq_t：purge queue，对已经提交事务的undo log进行维护和回收。
* trx_t：事务的内存结构体，对事务的信息进行管理和维护。

我们重点对trx_rseg_t结构体的内容进行介绍。

**trx_rseg_t**主要数据成员：

* last_page_no：history list上此rseg最后一个没有被purge的page no。
* last offset：最后一个未被purge的undo log header 偏移。
* last trx no：最后一个未被purge的事务no。
* last_del_marks：最后一个未被purge的日志需要被清理。

上面四个数据可以从trx_undo_t中获取，参考trx_purge_add_update_undo_to_history函数。

* trx_ref_count：被活跃事务引用的计数器；非0时，此回滚段所在的tablespace不可以被truncate。
* update_undo_list：所有活跃的update事务的trx_undo_t对象存储在此链表。
* update_undo_cached：如果update事务提交时，此事务只使用了一个page并且此page剩余空间大于1/4放入此链表；新update事务新申请undo log时优先从此链表分配。
* insert_undo_list：所有活跃的insert事务的trx_undo_t对象存储在此链表。
* insert_undo_cached：如果insert事务提交时，此事务只使用了一个page并且此page剩余空间大于1/4放入此链表；新insert事务新申请undo log时优先从此链表分配。

下面我们通过一副关系图来介绍内存中各个关键结构体之间的关系；其中实线代表拥有该对象，虚线代表引用该对象。

![undo log 内存结构](.img/4b013f0399a7_undo_mem_structure.png)

### undo log的分配与记录

我们通过之前的介绍了解到；undo log在磁盘和内存中是如何组织的；从中了解到，回滚段不论在磁盘和内存中，都是一个非常关键的结构体；InnoDB存储引擎通过回滚段来对undo log进行组织和管理，所以首先我们需要弄清楚回滚段是如何分配与使用的，之后再阐述undo log具体是如何记录的。

### 1. 分配回滚段

当开启一个事务的时候，需要预先为事务分配一个回滚段。

首先我们将事务分为两大类：只读事务与读写事务。分别从这两大类事务来探讨如何分配回滚段的。

**只读事务**：当事务涉及到对临时表的读写时，我们需要为其分配一个回滚段对其产生的undo log record进行记录，具体调用链路如下：

`trx_assign_rseg_temp() -> get_next_temp_rseg() -> (trx_sys->tmp_rsegs)
`

trx_sys->tmp_rsegs 对应的临时文件为ibtmp1，一般来说有128个回滚段。

**读写事务**：当一个事务被判定为读写模式时，会为其分配trx_id以及回滚段，具体调用链路如下

`trx_assign_rseg_durable() -> get_next_redo_rseg()
 |
 ->get_next_redo_rseg_from_trx_sys() -> (trx_sys->rsegs)
 |
 ->get_next_redo_rseg_from_undo_spaces() -> (undo_space->rsegs())
`

当InnoDB没有配置独立undo表空间时，从trx_sys->rsegs为读写事务分配回滚段；否则则从 undo_spaces->rsegs()为其分配回滚段；InnoDB从MySQL 8.0.3开始，独立表空间个数默认值从0改为2。

trx_sys->rsegs 对应的文件为ibdata1，默认有128个回滚段。

undo_space->rsegs() 对应的文件为undo_001，undo_002…，最多可有128个undo文件，每个文件默认128个回滚段。

具体从rsegs中分配时采用round-robin方式进行分配。

### 2. 使用回滚段

当发生数据变更时，我们需要使用undo log记录下变更前的数据记录；因此需要从回滚段分配中来分配一个undo slot来供事务记录undo。

记录undo的入口函数为trx_undo_report_row_operation，其大致流程如下：

1. 判断操作的表是否为临时表；如果是临时表，为其分配临时表回滚段，否则使用普通回滚段。
2. 根据事务类型，通过trx_undo_assign_undo为其分配trx_undo_t对象；之后事务产生的undo记录在此对象中。
3. 根据事务类型，通过trx_undo_page_report_insert/modify，来记录insert/update事务产生的undo。

接着来看一下 trx_undo_assign_undo 函数流程：

1. 首先尝试通过trx_undo_reuse_cached() 来获取可用的undo log对象。
 
 对于INSERT类型的undo log，我们从rseg->insert_undo_cached链表上获取undo log对象，并将其从链表上移除；之后通过trx_undo_insert_header_reuse()重新初始化undo page头部信息。
2. 对于UPDATE/DELETE类型undo log，从rseg->update_undo_cached链表上获取undo log对象，并将其从链表上移除；然后通过trx_undo_header_create()创建新的undo log header。
3. 然后使用trx_undo_header_add_space_for_xid() 作用于上述undo log对象，预留XID存储空间。
4. 最后使用trx_undo_mem_init_for_reuse()初始化undo log对象相关信息。

 如果没有缓存的undo log对象，我们就需要使用trx_undo_create()从回滚段上分配一个空闲的undo slot，并分配一个undo page，对其初始化。
 将已经分配好的undo log对象放入相关的链表中（rseg->insert_undo_list或rseg->update_undo_list）。
 最后，如果这个事务时DDL操作，需要将undo_hdr_page(事务记录undo log的第一个page)中的TRX_UNDO_DICT_TRANS置为TRUE.

**undo header page结构参考之前的《undo log的组织形式》的内容**

undo log最小的并发单元为undo slot，所以undo log支持最大的并发事务为：undo tablespace 数 * 回滚段数 * undo slot数。

### 3. undo log写入

当分配完undo slot，初始化完undo log对象后，我们就可以记录真正的undo log record；undo log record也分为一下两种，insert undo log record与update undo record。

当数据库需要修改某个数据记录时，都会写入一条update undo log record；当插入一条数据记录时，会写入一条insert undo log record。

![insert_update undo log record 结构](.img/83b66b0aae6a_insert_update_undo_log_record.png)

对于insert undo log写入的入口函数为trx_undo_page_report_insert()

* **Prev record offset (2)**：本条record开始的位置。
* **Next record offset (2)**：下一条record开始的位置。
* **Type (1)**：标记undo log record的类型，此处一般为 TRX_UNDO_INSERT_REC.
* **Undo Number (1-11)**：trx->undo_no，事务的第几条undo。
* **Table ID (1-11)**：聚集索引所对应的table id。
* **Unique Fields**：唯一键值

对于update undo log写入的入口函数为trx_undo_page_report_modify()

* **Prev record offset (2)**：同上
* **Next record offset (2)**：同上
* **Type+Extern Flag+Comp Info (1)**：
 
 Type为undo log rec的类型，此处一般有三种：
 
 TRX_UNDO_DEL_MARK_REC: 标记删除操作，未修改任何列值；可能由普通删除操作产生，也有可能由修改聚集索引产生，因为修改聚集索引操作被分拆为删除+插入操作。
* TRX_UNDO_UPD_DEL_REC: 更新一个已经被删除的记录；如某个记录被删除后，在很快插入一个相同的记录；之前的记录若未被purge，就可能重用该记录所在位置。
* TRX_UNDO_UPD_EXIST_REC: 更新一个未被标记删除的记录，也就是普通更新。

 Extern Flag：是否有外部存储列，以提示purge线程去清理外部存储。
 Comp Info：更新相关信息，例如更新是否导致索引序发生变化。

 **Undo Number (1-11)**：同上
 **Table ID (1-11)**：同上
 **Info Bits (1)**：是否标记删除REC_INFO_DELETED_FLAG.
 **Data Trx ID (1-11)**：修改旧记录的事务ID。
 **Data Roll Ptr (1-11)**：旧记录的回滚指针。
 **Unique Fields**：唯一键值
 **Update Get N Fields (1-5)**: 更新的列数。
 **UPD Old Columns**：发生更新时，旧记录的内容。
 **Delete Fileds len (2)**: 删除的列数。
 **DEL Old Columns**：发生删除时，旧记录的内容。

在写入过程中，可能出现undo page空间不足的情况；当出现这种情况，我们需要通过trx_undo_erase_page_end()来清除刚刚写入的区域，然后通过trx_undo_add_page()申请一个新的undo page加入到undo page list，同时将undo->last_page_no指向新的undo page，最后重试写入。

完成undo log record的写入后，通过trx_undo_build_roll_ptr()构建新的回滚指针返回；通过回滚指针我们可以找到相关记录的undo log record，从而构建出旧版本的数据；回滚指针将会记录在聚集索引记录中。

### undo log的应用

我们通过之前的介绍已经了解到， undo log的组织方式与分配记录；那么后面我们继续介绍undo log主要的应用是什么。

undo log的应用主要有两方面：

1. 事务回滚，崩溃恢复；此功能主要满足了事务的原子性，简单的说就是要么做完，要么不做。因为数据库在任何时候都可能发生宕机；包括停电，软硬件bug等。那数据库就需要保证不管发生任何情况，在重启数据库时都能恢复到一个一致性的状态；这个一致性的状态是指此时所有事务要么处于提交，要么处于未开始的状态，不应该有事务处于执行了一半的状态；所以我们可以通过undo log在数据库重启时把正在提交的事务完成提交，活跃的事务回滚，这样就保证了事务的原子性，以此来让数据库恢复到一个一致性的状态。
2. 多版本并发控制(MVCC)，此功能主要满足了事务的隔离性，简单的说就是不同活跃事务的数据互相可能是不可见的。因为如果两个活跃的事务彼此可见，那么一个事务将会看到另一个事务正在修改的数据，这样会发生数据错乱；所以我们可以借助undo log记录的历史版本数据，来恢复出对于一个事务可见的数据，来满足其读取数据的请求。

我们接下来就详细介绍上面两个功能undo log是如何实现的。

### 1. 崩溃恢复

在InnoDB因为某些原因停止运行后；重启InnoDB时，可能存在一个不一致的状态，这个时候我们就需要把MySQL恢复到一个一致的状态来保证数据库的可用性。这个恢复过程主要分下面这么几步：

1. 把最新的undo log从redo log中恢复出来，因为undo log是受redo log保护的。
2. 根据最新的undo log构建出InnoDB崩溃前的状态。
3. 回滚那些还没有提交的事务。

经过上面这三步后，InnoDB就可以恢复到一个一致的状态，并且对外提供服务。

下面我们详细的来介绍这三部分的具体过程：

#### 1.1 undo log的恢复

因为undo log受到redo log的保护，所以我们只需要根据最新的redo log就可以把undo log恢复到最新的状态；具体的调用过程如下：

`recv_recovery_from_checkpoint_start()// 从最新的一个log checkpoint开始读取redo log并应用。
 |
 -> recv_recovery_begin() // 将redo log读取到log buffer中，并将其parse到redo hash中
 |
 -> recv_scan_log_recs() // 扫描 log buffer中的redo log，并将redo hash中的redo log应用
 |
 -> recv_apply_hashed_log_recs() // 应用redo log到其对应的page上。
 |
 ->recv_apply_log_rec()->recv_recover_page()->recv_parse_or_apply_log_rec_body() -> MLOG_UNDO_INSERT… 
`

经过上述的流程之后，undo log就可以恢复到InnoDB崩溃前的最新的状态；虽然undo log已经恢复到最新的状态，但是InnoDB还没有恢复到崩溃前的最新状态；所以下一步我们就需要根据最新的undo log把InnoDB崩溃前的内存结构都恢复出来。

#### 1.2 构建InnoDB崩溃前的状态

构建InnoDB崩溃前的状态，主要是恢复崩溃前最新事务系统的状态；通过该状态我们可以知道那些事务已经提交，那些事务还未提交，以及那些事务还未开始。

我们从前面两章的介绍，回滚段不管在内存中还是在文件中都是组织undo log的重要数据结构；所以我们首先需要把回滚段的内存结构恢复出来，然后根据内存中的回滚段，把活跃的事务恢复出来。其具体过程在函数trx_sys_init_at_db_start()中实现，其大致步骤如下：

1. 通过trx_rsegs_init()扫描文件中的回滚段结构，来把rseg的内存结构恢复出来。
 
 通过trx_rseg_mem_create()把last_page_no，last_offset，last_trx_no，last_del_marks从文件中读取上来。
2. 然后通过trx_undo_lists_init()把rseg的四个链表：insert_undo_list，insert_undo_cached，update_undo_list，update_undo_cached从磁盘上恢复出来。

 在rseg内存结构恢复好之后，我们再通过trx_lists_init_at_db_start()把活跃的事务从rseg中恢复出来。
 1. 通过trx_resurrect_insert()恢复活跃的插入类型的事务。
2. 通过trx_resurrect_update()恢复活跃的更新类型的事务。

至此，我们就已经把InnoDB崩溃前的内存和文件状态都已经恢复出来了；其实这个时候InnoDB已经可以对外提供服务了，(毕竟内存和文件状态都就绪后我们也就可以保持一致性了)；那么最后一步的事务回滚就可以交给后台线程来慢慢做事务回滚，不影响主线程对外提供服务了。

#### 1.3 事务回滚

事务需要回滚主要有两种情况：

1. 事务发生异常：如发生在崩溃恢复时；其活跃事务虽然被恢复出来，但是无法继续，需要将其回滚。
2. 事务被显式回滚：如用户打开一个事务，执行完某些操作后需要将其回滚。

那么在回滚时，我们就需要借助undo log中的旧数据来把事务恢复到之前的状态；其入口函数为row_undo_step()；

其操作就是通过undo log来读取旧的数据记录，然后做逆向操作；主要分为下面这么几类：

1. 对于标记删除的记录清理删除标记。
2. 对于in-place更新，将数据更新为老版本。
3. 对于插入操作，删除聚集索引记录和二级索引记录。
 
 先通过row_undo_ins_remove_sec_rec()删除二级索引记录。
4. 再通过row_undo_ins_remove_clust_rec()删除聚集索引记录。

### 2. 多版本并发控制(MVCC)

多版本并发控制简单的说就是当前事务只能看见已经提交的数据记录，看不到正在修改的数据记录。所以我们只要弄清楚那些事务对于当前事务是已经提交的，那些事务对于当前事务是活跃的。

为了实现上述的功能我们先介绍几个比较关键的概念：

**trx::id**：事务开始的逻辑时间，也叫事务ID，在事务开始时通过trx_start_low()分配。

**trx::no**：事务结束的逻辑时间，在事务结束的时候通过trx_commit_low()分配。

**trx_sys::rw_trx_ids**：当前活跃的事务ID；事务在开始时ID会被添加至此数据结构中；事务提交时ID会被从此数据结构中删除。

在构造一条数据记录时，我们除了在数据记录中添加用户自主添加的数据列，系统还会自动分配一些系统列，具体包括：

**DATA_TRX_ID**：修改过此行数据记录的最新事务ID。

**DATA_ROLL_PTR**：指向这条数据记录的上一个版本的指针，上一版数据在undo log中。

通过上面几个数据结构的介绍，我们大概了解了一些基本概念；但是这对于一个事务来判断那些事务对它是已经提交的，那些事务对它是活跃的还是远远不够的；所以我们接下来介绍MVCC中最重要的一个数据结构：视图，也就是read view。

#### 2.1 视图

每一个事务在读取数据时都会被分配一个视图，通过视图就可以来判断其他事务对数据记录的可见性。下面我们来具体介绍一下视图是如何运作的。

**分配**：主要通过trx_assign_read_view()来给一个事务分配视图；在事务的隔离级别是Consistent Snapshot 或 Read Repeatable时，事务开始时会给其分配；其他情况下当事务需要读取数据时将会给其分配一个视图。

**回收**：事务结束时，会通过view_close()对其视图进行回收。

几个关键数据结构：

1. m_low_limit_id：高水位，分配时取trx_sys::max_trx_id，也就是取当前还没有被分配的事务ID。
2. m_up_limit_id：低水位，如果m_ids不为空，取其最小值，否则取trx_sys::max_trx_id。
3. m_ids：在此视图初始化时，通过copy_trx_ids()从trx_sys::rw_trx_ids拷贝一份活跃事务ID(不包含当前事务ID)。

那么有了上面这些数据我们就可以判断那些事务对于此视图是活跃的，那些事务对于此视图是已经提交的。

**那些事务对于此视图是活跃的：**

1. trx_id > read_view::m_low_limit_id
2. read_view::m_up_limit_id < trx_id < read_view::m_low_limit_id，并且 trx_id 属于 trx_t::read_view::m_ids

如果给定一个trx_id满足上面两个条件其中之一，那么这个事务对于此视图就是活跃的。

**那些事务对于此视图是已经提交的：**

1. trx id < read_view::m_up_limit_id
2. read_view::m_up_limit_id < trx id < read_view::m_low_limit_id，并且 trx id 不属于 trx_t::read_view::m_ids

如果给定一个trx_id满足上面两个条件其中之一，那么这个事务对于此视图就是已经提交的。

#### 2.2 数据可见性

通过上面的介绍，那么一个事务就可以通过此事务的视图来对数据记录判断可见性了。

具体是通过ReadView::changes_visible()来判断可见性的，具体如下：

假设一个事务为T，trx_id为记录R中的DATA_TRX_ID：

1. trx_id > read_view::m_low_limit_id，T 不可见 R
2. trx_id < read_view::m_up_limit_id，T 可见 R
3. read_view::m_up_limit_id =< trx_id <= read_view::m_low_limit_id时，如果trx_id 属于 trx_t::read_view::m_ids 时，T 不可见 R。否则可见 R

如果T对R不可见，就需要R中的DATA_ROLL_PTR来构造出上一个数据页版本，直至记录可见。

我们通过下面一个例子来说明可见性。

![read view 数据可见性](.img/bb70b894e608_read_view_visable.png)

### undo log 的清理

我们通过之前的系列文章已经了解到undo log在磁盘和内存中是如何组织的；undo log是如何分配的；以及undo log是如何使用的。那么undo log会一直记录下去么？当然不是，有些undo log如果没用的话是会被回收清理的。

那么下面这将会介绍那些undo log可以清理，以及undo log是怎么进行清理的。

### 1. 几个关键的数据结构

在介绍undo log 清理之前，先介绍几个关键的数据结构；这几个数据结构对于undo log的清理实现是至关重要的。

**trx_sys->serialisation_list**： 里面存放的是正在提交的事务，按照`trx_t::no`有序的排列；事务会在开始提交时通过 `trx_serialisation_number_get()` 添加至该数据结构，事务结束提交时通过`trx_erase_lists()`将该事务从该数据结构中移除。

**read_view::m_low_limit_no**：拥有该`read_view`的对象，对于`trx_t::no`小于`read_view::m_low_limit_no`的undo log都不在需要；该变量的取值时`trx_sys->serialisation_list`中最早的一个事务的`trx_t::no`；因为`trx_sys->serialisation_list`内有序存放的正在提交的事务，如果一个事务的`trx_t::no`比该数值还小，那么这个事务一定已经提交了。

**TRX_RSEG_HISTORY与TRX_UNDO_HISTORY_NODE**：这两个值我们之前在《undolog的组织形式》里简单介绍过，这两个值共同将回滚段中的history list组织起来；在事务提交时，如果是update/delete类型的undo log，将其`undo log header`以头插法的方式通过`trx_purge_add_update_undo_to_history()`加入到该回滚段的history list中，如果是insert类型的undo log其空间会被当场释放，这是因为insert记录没有旧的版本；因此history list中的undo log header是以`trx_t::no`降序排列的，这里需要注意一下：**history list里面的节点是undo log header**。下面我们通过一幅图来具体说明下磁盘上history list的结构。

![history list磁盘结构](.img/1ed3ab69f4dc_history_list_disk_structure.png)

### 2. 那些undo log可以清理？

对于一个事务来说，早于`read_view::m_low_limit_no`的undo log都不需要访问了；那么如果存在一个read view，其`read_view::m_low_limit_no`比所有read view的`m_low_limit_no`都要小，那么小于此`read_view::m_low_limit_no`的undo log就不在被所有活跃事务所需要了，那么这些undo log就可以清理了。

在read_view初始化时，会使用头插法通过`view_open()`插入到一个全局视图链表(`MVCC::m_views`)中，在事务结束时通过`view_close()`会从全局视图链表中将此read view移除；因为是顺序插入，所以此链表中最后一个还没有close的视图就可以看做是最老的一个视图；小于此视图的undo log可以被清理，一般将此视图赋值给`purge_sys::view`。

**现在我们已经可以决定那些undo log是可以被清理的，那么下一步我们还需要找到具体那些undo log可以清理。**

在事务提交时，此事务对应的回滚段会通过`trx_serialisation_number_get()`加入到`purge_sys::purge_queue`中。

`purge_sys::purge_queue`是一个以回滚段中第一个提交事务的`trx_t::no`为key的优先级队列。

如此一来，从`purge_sys::purge_queue`取出的回滚段中一定包含最老提交的事务，将此事务的`trx_t::no`与`purge_sys::view`对比，即可判断出此事务相关的undo log是否可以被清理。

`purge_sys::purge_queue`的详细信息如下图：

![purge queue结构](.img/91b9c66a5358_purge_queue.png)

### 3. undo log怎么清理？

解决了那些undo log可以清理的问题后，下面接着继续看undo log怎么进行清理的问题。

当放入history list的undo log且不会再被访问时，需要进行清理操作，另外数据页上面的标记删除的操作也需要清理掉，有一些purge线程负责这些操作，去入口函数为`srv_do_purge() -> trx_purge()`，其大致流程如下：

1. 通过`trx_sys->mvcc->clone_oldest_view()`获取最老的视图复制给`purge_sys::view`，方便之后真正purge undo log时判断其是否不会再被访问到了。
2. 通过`trx_purge_attach_undo_recs()`获取需要被purge的undo log，其大致流程如下：

 通过`trx_purge_fetch_next_rec()`循环获取可以被purge的undo log，默认一次最多获取300个undo log record，可以通过`innodb_purge_batch_size`来调整。
3. 循环获取可以被purge的undo log record大致流程如下：
 
 从`purge_sys::purge_queue`取出第一个回滚段，从其history list上读取最老还未被purge的事务的undo log header。
4. 从此undo log header依次读取undo log record。
5. 读取完毕后，重新统计此回滚段最老还未被purge的事务的位点，然后重新放入`purge_sys::purge_queue`；最后回到第一步。

 将这些undo log分发给purge工作线程，purge工作线程的入口函数为`row_purge_step()->row_purge()->row_purge_record()`。

 这里purge undo log record时主要分为两种情况：清理`TRX_UNDO_DEL_MARK_REC`记录或者清理`TRX_UNDO_UPD_EXIST_REC`记录

 1. 清理`TRX_UNDO_DEL_MARK_REC`类型的记录，需要通过`row_purge_del_mark()`将所有的聚集索引与二级索引记录都清除掉。
2. 清理`TRX_UNDO_UPD_EXIST_REC`类型的记录，需要通过`row_purge_upd_exist_or_extern()`将旧的二级索引清理掉。

 通过`trx_purge_truncate()`来对history list进行清理，其大致流程如下：

 1. 遍历所有回滚段，并通过`trx_purge_truncate_rseg_history()`对回滚段中的history list进行清理，其大致流程如下：
 
 将history list最后一个事务的undo log header读取出来。
2. 判断此undo log是否已经被purge，如果已经被purge则继续；如果没有被purge则退出。
3. 将此事务所有的undo log释放，并从history list上删除；会到第一步。

 在这之后，如果发现某些undo tablespace空间占用过大，被标记需要通过`trx_purge_truncate_marked_undo()`进行对其truncate，其大致流程如下：
 1. 创建一个undo_trunc.log的标记文件，来表明当前undo tablespace正在进行truncate；这是为了保证在truncate中间发生重启时可以顺利重建此undo tablespace。
2. 通过`trx_undo_truncate_tablespace()`接口来对其文件做真正的truncate。
3. 删除undo_trunc.log标记文件，表明undo tablespace的truncate已经完成。

注意：当一个undo tablespace被标记为需要truncate时，不会再有事务从此undo tablespace分配回滚段，而且进行truncate时必须保证该undo tablespace上所有的undo log都已经被purge。

### 4. 最后

通过上面的介绍，我们知道了undo log那些记录可以被清理以及是怎么清理的，但是清理undo log过程中还有很多繁杂的细节；比如清理索引时涉及到对B树的操作，以及旧版本数据的构建，XA事务，BLOB等等；这类内容暂时略过后面有机会继续介绍。

### 参考内容

 [MySQL 8.0.23’s source code](https://github.com/mysql/mysql-server/tree/mysql-8.0.23)

 [MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/)

 [MySQL Server Team Blog](https://dev.mysql.com/blog-archive/)

 [庖丁解InnoDB之Undo LOG](https://zhuanlan.zhihu.com/p/427911093)

 [数据库故障恢复机制的前世今生](https://zhuanlan.zhihu.com/p/54981906)

 [浅析数据库并发控制机制](https://zhuanlan.zhihu.com/p/45339550)

 [Jeremy Cole’s github](https://github.com/jeremycole)

 [A little fun with InnoDB multi-versioning](https://blog.jcole.us/2014/04/16/a-little-fun-with-innodb-multi-versioning/)

 [The basics of the InnoDB undo logging and history system](https://blog.jcole.us/2014/04/16/the-basics-of-the-innodb-undo-logging-and-history-system/)

 [MySQL · 引擎特性 · InnoDB undo log 漫游](https://developer.aliyun.com/article/50853)

 [InnoDB：undo log（1）](https://zhuanlan.zhihu.com/p/165457904)

 [InnoDB：undo log（2）](https://zhuanlan.zhihu.com/p/263038786)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)