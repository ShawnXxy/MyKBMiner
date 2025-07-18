# MySQL · 最佳实践 · How to read the lock information from debugger

**Date:** 2020/10
**Source:** http://mysql.taobao.org/monthly/2020/10/03/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2020 / 10
 ](/monthly/2020/10)

 * 当期文章

 MySQL · 源码分析 · 子查询优化源码分析
* MySQL · 源码分析 · undo tablespace 的发展
* MySQL · 最佳实践 · How to read the lock information from debugger

 ## MySQL · 最佳实践 · How to read the lock information from debugger 
 Author: 攀峰 

 ## MySQL version (8.0)

It’s quite common for database kernel engineers to debug the lock related issue. Hence
it’s import for us to understand the lock information from debugger.

### Basic Data Structure
There are two major data structures we need to know. One is lock_t, another one is trx_t.

The main structure of lock_t is as follow:

`/** Lock struct; protected by lock_sys->mutex */
struct lock_t {

 /** True if the lock has been removed from transaction's lock
 list. For example, while the transaction is releasing lock, the
 background purge thread or someother thread may move the lock
 object to some otherplace by delete + insert. This may happen
 especially when we partitioned the lock system */ 
 bool discard;

 /** transaction owning the lock */ 
 trx_t *trx;

 /** list of the locks of the transaction */ 
 UT_LIST_NODE_T(lock_t) trx_locks;

 /** Index for a record lock */ 
 dict_index_t *index;

 /** Hash chain node for a record lock. The link node in a singly
 linked list, used by the hash table. */ 
 lock_t *hash;

 union { 
 /** Table lock */ 
 lock_table_t tab_lock; 
 /** Record lock */ 
 lock_rec_t rec_lock; 

 }; 
 
 /** The lock type and mode bit flags.
 LOCK_GAP or LOCK_REC_NOT_GAP, LOCK_INSERT_INTENTION, wait flag, ORed */ 
 uint32_t type_mode; 

 /** Timestamp when it was created. */ 
 uint64_t m_seq; 
}
`

The main strucutre of trx_t is as follow:

`struct trx_t { 
 ib_uint32_t in_depth; /*!< Track nested TrxInInnoDB count */ 
 ib_uint32_t in_innodb; /*!< if the thread is executing in the InnoDB context count > 0. */ 
 bool abort; /*!< if this flag is set then this transaction must abort when it can */ 
 trx_id_t id; /*!< transaction id */ 
 trx_id_t no; /*!< transaction serialization number: max trx id shortly before the transaction is moved to 
 COMMITTED_IN_MEMORY state. Protected by trx_sys_t::mutex when trx->in_rw_trx_list. Initially set to TRX_ID_MAX. */
 trx_state_t state; 
 trx_lock_t lock; /*!< Information about the transaction locks and state. Protected by trx->mutex or lock_sys->mutex 
 lock_pool_t rec_pool; /*!< Pre-allocated record locks */ 
 lock_pool_t table_pool; /*!< Pre-allocated table locks */ 
 ulint rec_cached; /*!< Next free rec lock in pool */ 
 ulint table_cached; /*!< Next free table lock in pool */ 
 ...... 
} 
`
As we can see, lock_t->trx will point to the corresponding transaciton. On the 
other hand, trx_t->rec_pool/table_pool will point back to the lock information.

### Real Lif Examples
Here is an live example of a lock from debugger LR 3

`(gdb) p *(ib_lock_t *)0x7f66462e4d60 
$8 = {trx = 0x7f6653f51ac0, trx_locks = {prev = 0x7f66462e4c18, next = 0x0}, 
 index = 0x7f665da63b08, 
 hash = 0x0, 
 un_member = {tab_lock = {table = 0xb00000000, locks = {prev = 0x100, next = 0x2300002000}}, 
 rec_lock = {space = 0, page_no = 11,n_bits = 256}}, 
 type_mode = 291} 
`
First of all, type_mode = 291 = 256 (LOCK_WAIT) + 32 (LOCK_REC) + 3(LOCK_X). This means we are waiting for Exclusive Record Lock.

We then check the corresponding transaction information.

`(gdb) p *((ib_lock_t *)0x7f66462e4018)->trx 
$9 = {mutex = {m_impl = {m_lock_word = 0, m_waiters = 0, m_event = 0x7f6653b43798, m_policy = {m_count = {m_spins = 0, m_waits = 0,
m_calls = 0, m_enabled = false}, 
m_id = LATCH_ID_TRX}}, 
sm_ptr = 0x7f6644c585c0}, 
in_depth = 0, 
in_innodb = 536870912, 
abort = false, 
id = 236410, 
no = 18446744073709551615, 
state = TRX_STATE_ACTIVE, 
skip_lock_inheritance = false, 
read_view = 0x0, 
trx_list = {prev = 0x7f6653f51ac0, next = 0x0}, 
no_list = {prev = 0x0, next = 0x0}, 
lock = {n_active_thrs = 0, 
que_state = TRX_QUE_RUNNING, wait_lock = 0x0, deadlock_mark = 0, was_chosen_as_deadlock_victim = false, wait_started = 0,
wait_thr = 0x0, 
rec_pool = std::vector of length 8, capacity 8 = {0x7f66462e4018, 0x7f66462e4160, 0x7f66462e42a8,
0x7f66462e43f0, 0x7f66462e4538, 0x7f66462e4680, 0x7f66462e47c8, 0x7f66462e4910}, 
table_pool = std::vector of length 8, 
capacity 8 = {0x7f664003ae18, 0x7f664003ae60, 0x7f664003aea8, 0x7f664003aef0,
0x7f664003af38, 0x7f664003af80, 0x7f664003afc8, 0x7f664003b010}, 
rec_cached = 1, 
table_cached = 6, 
dict_operation = TRX_DICT_OP_TABLE, 
`
We can see this transaciton have 1 rec lock (rec_cached) and 6 table locks (table_cached). The detail informaiton of record 
locks can be foudn in rec_pool and table locks information can be then found in table_pool. From dict_operation, we can 
see we are dropping table. We need to lock a bunch of system catalog tables. By using information from table_pool, we can 
find the corresponding system tables are: SYS_TABLES, SYS_COLUMNS, SYS_INDEXES, SYS_FIELDS, SYS_FIELDS, SYS_TABLESPACES, 
SYS_DATAFILES. By checking the informaiton from rec_pool, we can find the space id is 0, page number is 11. 

rec_lock = {space = 0, page_no = 11, n_bits = 256}}

Hence the above debugger information can be tranlated into human readable information: 
When we drop table, we hold a bunch of table locks on system catalog tables. For each system catalog table,
we then need to hold a record lock to do real update. In this case, we are waiting on a rec lock to update SYS_INDEXES’s root
page.

We can actually read even more from it. But that will be covered next time. :-)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)