# MySQL · 特性分析 ·MySQL 5.7新特性系列三

**Date:** 2016/07
**Source:** http://mysql.taobao.org/monthly/2016/07/01/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 07
 ](/monthly/2016/07)

 * 当期文章

 MySQL · 特性分析 ·MySQL 5.7新特性系列三
* MySQL · 特性分析 · 5.7 代价模型浅析
* PgSQL · 实战经验 · 分组TOP性能提升44倍
* MySQL · 源码分析 · 网络通信模块浅析
* MongoDB · 特性分析 · 索引原理
* SQLServer · 特性分析 · XML与JSON应用比较
* MySQL · 最佳实战 · 审计日志实用案例分析
* MySQL · 性能优化 · 条件下推到物化表
* MySQL · 源码分析 · Query Cache内部剖析
* MySQL · 捉虫动态 · 备库1206错误问题说明

 ## MySQL · 特性分析 ·MySQL 5.7新特性系列三 
 Author: lengxiang 

 继上两期月报，MySQL5.7新特性之一介绍了一些新特性及兼容性问题，MySQL 5.7新特性之二介绍了临时表的优化和实现。

这期我们一起来学习下undo空间管理，重点介绍truncate功能。

### 1. 背景
InnoDB存储引擎中，undo在完成事务回滚和MVCC之后，就可以purge掉了，但undo在事务执行过程中，进行的空间分配如何回收，就变成了一个问题。 我们亲历用户的小实例，因为一个大事务，导致ibdata file到800G大小。

我们先大致看下InnoDB的undo在不同的版本上的一些演进:

**MySQL 5.5的版本上**

InnoDB undo是放在系统表空间即ibdata file文件中，这样如果有比较大的事务(即需要生成大量undo的)，会撑大ibdata数据文件，
虽然空间可以重用， 但文件大小不能更改。

关于回滚段的，只有这个主要的参数，用来设置多少个rollback segment。

`mysql> show global variables like '%rollback_segment%';
+----------------------------+-------+
| Variable_name | Value |
+----------------------------+-------+
| innodb_rollback_segments | 128 |
+----------------------------+-------+
`
**MySQL 5.6的版本上**

InnoDB undo支持独立表空间， 增加如下参数：

`+-------------------------+-------+
| Variable_name | Value |
+-------------------------+-------+
| innodb_undo_directory | . |
| innodb_undo_logs | 128 |
| innodb_undo_tablespaces | 1 |
+-------------------------+-------+
`
这样，在install的时候，就会在data目录下增加undo数据文件，来组成undo独立表空间，但文件变大之后的空间回收还是成为问题。

**MySQL 5.7的版本上**

InnoDB undo在支持独立表空间的基础上，支持表空间的truncate功能，增加了如下参数：

`mysql> show global variables like '%undo%'; +--------------------------+------------+
| Variable_name | Value |
+--------------------------+------------+
| innodb_max_undo_log_size | 1073741824 |
| innodb_undo_directory | ./ |
| innodb_undo_log_truncate | OFF |
| innodb_undo_logs | 128 |
| innodb_undo_tablespaces | 3 |
+--------------------------+------------+
mysql> show global variables like '%truncate%';
+--------------------------------------+-------+
| Variable_name | Value |
+--------------------------------------+-------+
| innodb_purge_rseg_truncate_frequency | 128 |
| innodb_undo_log_truncate | OFF |
+--------------------------------------+-------+
`

InnoDB的purge线程，会根据innodb_undo_log_truncate开关的设置，和innodb_max_undo_log_size设置的文件大小阈值，以及truncate的频率来进行空间回收和rollback segment的重新初始化。

接下来我们详细看下5.7的InnoDB undo的管理：

### 2. undo表空间创建
设置innodb_undo_tablespaces的个数， 在mysql install的时候，创建指定数量的表空间。

InnoDB支持128个undo logs，这里特别说明下，从5.7开始，innodb_rollback_segments的名字改成了innodb_undo_logs，但表示的都是回滚段的个数。

从5.7.2开始，其中32个undo logs为临时表的事务分配的，因为这部分undo不记录redo，不需要recovery，另外从33-128一共96个是redo-enabled undo。

**rollback segment的分配如下：**

`Slot-0: reserved for system-tablespace.
Slot-1....Slot-N: reserved for temp-tablespace.
Slot-N+1....Slot-127: reserved for system/undo-tablespace. */
`

其中如果是临时表的事务，需要分配两个undo logs，其中一个是non-redo undo logs；这部分用于临时表数据的回滚。
另外一个是redo-enabled undo log，是为临时表的元数据准备的，需要recovery。

而且， 其中32个rollback segment创建在临时表空间中，并且临时表空间中的回滚段在每次server start的时候，需要重建。

每一个rollback segment可以分配1024个slot，也就是可以支持96*1024个并发的事务同时， 但如果是临时表的事务，需要占用两个slot。

**InnoDB undo的空间管理简图如下：**

![undo空间管理](http://img1.tbcdn.cn/L1/461/1/a6e9323d7de7653c4aad4e4dfef66cc58d597020)

**注核心结构说明：**

**1. rseg slot**

rseg slot一共128个，保存在ibdata系统表空间中，其位置在：

` /*!< the start of the array of rollback segment specification slots */
 #define TRX_SYS_RSEGS (8 + FSEG_HEADER_SIZE) 
`
每一个slot保存着rollback segment header的位置。包括space_id + page_no，占用8个bytes。其宏定义：

`/* Rollback segment specification slot offsets */
/*-------------------------------------------------------------*/
#define TRX_SYS_RSEG_SPACE 0 /* space where the segment
 header is placed; starting with
 MySQL/InnoDB 5.1.7, this is
 UNIV_UNDEFINED if the slot is unused */
#define TRX_SYS_RSEG_PAGE_NO 4 /* page number where the segment
 header is placed; this is FIL_NULL
 if the slot is unused */

/* Size of a rollback segment specification slot */
#define TRX_SYS_RSEG_SLOT_SIZE 8
`

**2. rseg header**

 rseg header在undo表空间中，每一个rseg包括1024个undo segment slot，每一个slot保存着undo segment header的位置，包括page_no，暂用4个bytes，因为undo segment不会跨表空间，所以space_id就没有必要了。

其宏定义如下：

`/* Undo log segment slot in a rollback segment header */
/*-------------------------------------------------------------*/
#define TRX_RSEG_SLOT_PAGE_NO 0 /* Page number of the header page of
 an undo log segment */
/*-------------------------------------------------------------*/
/* Slot size */
#define TRX_RSEG_SLOT_SIZE 4
`

**3. undo segment header**

 undo segment header page即段内的第一个undo page，其中包括四个比较重要的结构：

 undo segment header
 进行段内空间的管理

 undo page header
 page内空间的管理，page的类型：FIL_PAGE_UNDO_LOG

 undo header
 包含undo record的链表，以便安装事务的反顺序，进行回滚

 undo record
 剩下的就是undo记录了。

### 3. undo段的分配
undo段的分配比较简单，其过程如下：

**首先是rollback segment的分配：**

`trx->rsegs.m_redo.rseg = trx_assign_rseg_low(
 srv_undo_logs, srv_undo_tablespaces,
 TRX_RSEG_TYPE_REDO);
`
1. 使用round-robin的方式来分配rollback segment
2. 如果有单独设置undo表空间，就不使用system表空间中的undo segment
3. 如果设置的是truncate的就不分配
4. 一旦分配了，就设置trx_ref_count，不允许truncate。

具体代码参考：

`/******************************************************************//**
Get next redo rollback segment. (Segment are assigned in round-robin fashion).
@return assigned rollback segment instance */
static
trx_rseg_t*
get_next_redo_rseg(
/*===============*/
 ulong max_undo_logs, /*!< in: maximum number of UNDO logs to use */
 ulint n_tablespaces) /*!< in: number of rollback tablespaces */
`

**其次是undo segment的创建：**

从rollback segment里边选择一个free的slot，如果没有，就会报错，通常是并发的事务太多。

错误日志如下：

`ib::warn() << "Cannot find a free slot for an undo log. Do"
 " you have too many active transactions running"
 " concurrently?";
`
如果有free，就创建一个undo的segment。

核心的代码如下：

`/***************************************************************//**
Creates a new undo log segment in file.
@return DB_SUCCESS if page creation OK possible error codes are:
DB_TOO_MANY_CONCURRENT_TRXS DB_OUT_OF_FILE_SPACE */
static 
dberr_t
trx_undo_seg_create(
/*================*/
 trx_rseg_t* rseg __attribute__((unused)),/*!< in: rollback segment */
 trx_rsegf_t* rseg_hdr,/*!< in: rollback segment header, page
 x-latched */
 ulint type, /*!< in: type of the segment: TRX_UNDO_INSERT or
 TRX_UNDO_UPDATE */
 ulint* id, /*!< out: slot index within rseg header */
 page_t** undo_page,
 /*!< out: segment header page x-latched, NULL
 if there was an error */
 mtr_t* mtr) /*!< in: mtr */

 /* fputs(type == TRX_UNDO_INSERT
 ? "Creating insert undo log segment\n"
 : "Creating update undo log segment\n", stderr); */
 slot_no = trx_rsegf_undo_find_free(rseg_hdr, mtr);

 if (slot_no == ULINT_UNDEFINED) {
 ib::warn() << "Cannot find a free slot for an undo log. Do"
 " you have too many active transactions running"
 " concurrently?";

 return(DB_TOO_MANY_CONCURRENT_TRXS);
 }
`

### 4. undo的truncate
undo的truncate主要由下面两个参数控制：innodb_purge_rseg_truncate_frequency，innodb_undo_log_truncate。

1. innodb_undo_log_truncate是开关参数。
2. innodb_purge_rseg_truncate_frequency默认128，表示purge undo轮询128次后，进行一次undo的truncate。

当设置innodb_undo_log_truncate=ON的时候， undo表空间的文件大小，如果超过了innodb_max_undo_log_size， 就会被truncate到初始大小，但有一个前提，就是表空间中的undo不再被使用。

其主要步骤如下：

1. 超过大小了之后，会被mark truncation，一次会选择一个
2. 选择的undo不能再分配新给新的事务
3. purge线程清理不再需要的rollback segment
4. 等所有的回滚段都释放了后，truncate操作，使其成为install db时的初始状态。

默认情况下， 是purge触发128次之后，进行一次rollback segment的free操作，然后如果全部free就进行一个truncate。

但mark的操作需要几个依赖条件需要满足：

1. 系统至少得有两个undo表空间，防止一个offline后，至少另外一个还能工作
2. 除了ibdata里的segment，还至少有两个segment可用
3. undo表空间的大小确实超过了设置的阈值

其核心代码参考：

`/** Iterate over all the UNDO tablespaces and check if any of the UNDO
tablespace qualifies for TRUNCATE (size > threshold).
@param[in,out] undo_trunc undo truncate tracker */
static
void
trx_purge_mark_undo_for_truncate(
 undo::Truncate* undo_trunc)
`
因为，只要你设置了truncate = on，MySQL就尽可能的帮你去truncate所有的undo表空间，所以它会循环的把undo表空间加入到mark列表中。

最后，循环所有的undo段，如果所属的表空间是marked truncate，就把这个rseg标志位不可分配，加入到trunc队列中，在purge的时候，进行free rollback segment。

**注意：**

如果是在线库，要注意影响，因为当一个undo tablespace在进行truncate的时候，不再承担undo的分配。只能由剩下的undo 表空间的rollback segment接受事务undo空间请求。

MySQL 5.7 新特性系列，下次进行group replication的分享，敬请期待。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)