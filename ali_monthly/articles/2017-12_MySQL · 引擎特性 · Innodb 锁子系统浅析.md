# MySQL · 引擎特性 · Innodb 锁子系统浅析

**Date:** 2017/12
**Source:** http://mysql.taobao.org/monthly/2017/12/02/
**Images:** 3 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 12
 ](/monthly/2017/12)

 * 当期文章

 MySQL · 引擎特性 · InnoDB 事务系统
* MySQL · 引擎特性 · Innodb 锁子系统浅析
* MySQL · 特性分析 · LOGICAL_CLOCK 并行复制原理及实现分析
* PgSQL · 源码分析 · AutoVacuum机制之autovacuum launcher
* MSSQL · 最佳实践 · SQL Server备份策略
* MySQL · 最佳实践 · 一个“异常”的索引选择
* PgSQL · 内核开发 · 利用一致性快照迁移你的数据
* PgSQL · 应用案例 · 手机行业分析、决策系统设计-实时圈选、透视、估算
* MySQL · 最佳实践 · 如何索引JSON字段
* MySQL · myrocks · 相关tools介绍

 ## MySQL · 引擎特性 · Innodb 锁子系统浅析 
 Author: zhuyan 

 ## 锁类型
Innodb 的锁从锁粒度上大致可以分为行锁和表锁，之前接触过的[Berkeley DB](https://en.wikipedia.org/wiki/Berkeley_DB)(MySQL 5.1前的事务储存引擎，后被 Innodb 取代)只对存储格式为 Hash 的定长数据支持行锁，对于 Btree 格式的仅支持页锁，作为 KV 类型的存储引擎，[锁的类型](https://docs.oracle.com/cd/E17076_05/html/programmer_reference/lock.html)也相对简单。Innodb 根据[官方文档](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html)的描述,除了基本的共享锁和排他锁，还有意向锁，Gap锁，Next key锁等类型，最开始接触的时候确实有些眼花缭乱，关于各种锁类型的使用场景描述可以参考[早期月报](http://mysql.taobao.org/monthly/2016/01/01/)前两个小节。

在 Innodb 内部用一个 unsiged long 类型数据表示锁的类型, 如图所示，最低的 4 个 bit 表示 lock_mode, 5-8 bit 表示 lock_type, 剩下的高位 bit 表示行锁的类型。

 record_lock type
 lock_type
 lock_mode

lock_mode 描述了锁的基本类型，在代码中的定义如下：

`/* Basic lock modes */
enum lock_mode {
 LOCK_IS = 0, /* intention shared */
 LOCK_IX, /* intention exclusive */
 LOCK_S, /* shared */
 LOCK_X, /* exclusive */
 LOCK_AUTO_INC, /* locks the auto-inc counter of a table
 in an exclusive mode */
 LOCK_NONE, /* this is used elsewhere to note consistent read */
 LOCK_NUM = LOCK_NONE, /* number of lock modes */
 LOCK_NONE_UNSET = 255
};
#define LOCK_MODE_MASK 0xFUL /* mask used to extact lock type from the
 type_mode field in a lock*/
`

lock_type 占用 5-8 bit 位，目前只用了 5 和 6 位，大小为 16 和 32 ，表示 LOCK_TABLE 和 LOCK_REC，使用宏定义 `#define LOCK_TYPE_MASK 0xF0UL` 来获取值。

record_lock_type 对于 LOCK_TABLE 类型来说都是空的，对于 LOCK_REC 目前值有：

`#define LOCK_WAIT 256 /* 表示正在等待锁 */
#define LOCK_ORDINARY 0 /* 表示 next-key lock ，锁住记录本身和记录之前的 gap*/
#define LOCK_GAP 512 /* 表示锁住记录之前 gap（不锁记录本身） */
#define LOCK_REC_NOT_GAP 1024 /* 表示锁住记录本身，不锁记录前面的 gap */
#define LOCK_INSERT_INTENTION 2048 /* 插入意向锁 */
#define LOCK_CONV_BY_OTHER 4096 /* 表示锁是由其它事务创建的(比如隐式锁转换) */
`
使用位操作来设置和判断是否设置了对应的值。

## 静态数据结构
对于每个锁对象，有两个存在的纬度：一个是事务纬度，每个事务都可以获得锁结构和等待某些锁。另一个是全局纬度，所有的锁都保存在 Lock_sys->hash 哈希表中。无论是表锁还是行锁，都是用结构 lock_t 来描述：

`/** Lock struct; protected by lock_sys->mutex */
struct lock_t {
 trx_t* trx; /*!< transaction owning the
 lock */
 UT_LIST_NODE_T(lock_t)
 trx_locks; /*!< list of the locks of the
 transaction */
 ulint type_mode; /*!< lock type, mode, LOCK_GAP or
 LOCK_REC_NOT_GAP,
 LOCK_INSERT_INTENTION,
 wait flag, ORed */
 hash_node_t hash; /*!< hash chain node for a record
 lock */
 dict_index_t* index; /*!< index for a record lock */
 union {
 lock_table_t tab_lock;/*!< table lock */
 lock_rec_t rec_lock;/*!< record lock */
 } un_member; /*!< lock details */
};
`
对于每个变量的意义注释已经说的比较清楚了，其中 type_mode 就是第一小节中 lock_type | type_mode，两个锁是否冲突就是使用它们各自的 type_mode 根据锁兼容矩阵来判断的，后面会详细说。

变量 hash 是 Inodb 中构造 Hash 表需要，当锁插入到 Lock_sys->hash 中，Hash 值相同就形成链表，使用变量 hash 相连。

un_member 表示 lock_t 不是表锁就是行锁，看下行锁的结构：

`/** Record lock for a page */
struct lock_rec_t {
 ulint space; /*!< space id */
 ulint page_no; /*!< page number */
 ulint n_bits; /*!< number of bits in the lock
 bitmap; NOTE: the lock bitmap is
 placed immediately after the
 lock struct */
};
`
[space, page_no] 可以确定锁对应哪个页，参考下[上个月月报](http://mysql.taobao.org/monthly/2017/11/01/)最后两个小节，页上每行数据紧接着存放，内部使用一个 heap_no 来表示是第几行数据。因此[space, page_no, heap_no]可以唯一确定一行。Innodb 使用位图来表示锁具体锁住了那几行，在函数 lock_rec_create 中为 lock_t 分配内存空间的时候，会在对象地址后分配一段内存空间(当前行数 + 64)用来保存位图。n_bits 表示位图大小。

`/* Make lock bitmap bigger by a safety margin */
n_bits = page_dir_get_n_heap(page) + LOCK_PAGE_BITMAP_MARGIN;
n_bytes = 1 + n_bits / 8;

lock = static_cast<lock_t*>(
 mem_heap_alloc(trx->lock.lock_heap, sizeof(lock_t) + n_bytes));
`
锁创建完成后首先会插入到全局 Hash 表中，然后放到对应的事务的锁链表中。相同(space,page_no)的锁会被 Hash 到同一个桶里，使用 lock_t->hash 串成链表。

`HASH_INSERT(lock_t, hash, lock_sys->rec_hash,
 lock_rec_fold(space, page_no), lock);
....
 
UT_LIST_ADD_LAST(trx_locks, trx->lock.trx_locks, lock);
`
2016/01 月月报有一张比较直观的图：

![imag](.img/1c5a3a8b7f45_innodb_lock.png)

## 加锁分析
对于行数据的加锁是由函数 lock_rec_lock 完成，简单点来看，主要的参数是 mode(锁类型)，block(包含该行的 buffer 数据页)，heap_no(具体哪一行)。就可以确定加什么样的锁，以及在哪一行加。对于 mode 的值，来源于查询的逻辑，索引和二级索引的定义，隔离级别等等，可以参考[这篇文章](http://hedengcheng.com/?p=771),其中简单介绍了基本的语句加什么类型的锁，对于更加复杂的情况，可以设断点调试来看。

### lock fast
lock_rec_lock 首先走 lock_rec_lock_fast 逻辑，判断能否快速完成加锁。如果对应 block 上面一个锁都没有( lock_rec_get_first_on_page(block)==NULL )，那么就创建一个锁( lock_rec_create )，返回加锁成功。如果 block 上已经存在锁，满足下面代码的逻辑就返回 LOCK_REC_FAIL, 快速加锁失败。

`if (lock_rec_get_next_on_page(lock) /* 页上是否只有一个锁 */
 || lock->trx != trx /* 拥有锁的事务不是当前事务 */
 || lock->type_mode != (mode | LOCK_REC)/* 已有锁和要加的锁模式是否相同 */
 || lock_rec_get_n_bits(lock) <= heap_no) { /* 已有锁的 n_bits 是否满足 heap_no */
 status = LOCK_REC_FAIL;
}else if (!impl) {
 /* If the nth bit of the record lock is already set
 then we do not set a new lock bit, otherwise we do
 set */
 if (!lock_rec_get_nth_bit(lock, heap_no)) {
 lock_rec_set_nth_bit(lock, heap_no);
 status = LOCK_REC_SUCCESS_CREATED;
 }
`
如果上述条件都为 false，说明：

* page 上只有一个锁
* 拥有该锁的事务是当前事务
* 锁模式相同
* n_bits 也足够描述大小为 heap_no 的行

那么只需要设置一下 bitmap 就可以了（impl 表示加隐式锁，其实也就是不加锁）。

注：上述函数 lock_rec_get_first_on_page(block) 是从全局 Lock_sys->hash 中拿到第一个锁的，也就是 Hash 桶的第一个 node。

### lock slow
lock fast 逻辑失败后就会走 lock slow 逻辑，也就是上述 lock fast 判断的四个条件中有一个或多个为 true的时候。

lock slow 首先判断当前事务上是否已经加了同等级或者更强级别的锁，函数 lock_rec_has_expl，循环取出对应行上的所有锁，它们要满足以下几个条件，就认为行上有更强的锁。

1. 基本锁类型更强，就比如加了 LOCK_X 就不必要加 LOCK_S 了。lock_mode 基本锁类型之间的强弱关系使用 lock_strength_matrix 判断(lock_mode_stronger_or_eq)
 ` static const byte lock_strength_matrix[5][5] = {
 /** IS IX S X AI */
 /* IS */ { TRUE, FALSE, FALSE, FALSE, FALSE},
 /* IX */ { TRUE, TRUE, FALSE, FALSE, FALSE},
 /* S */ { TRUE, FALSE, TRUE, FALSE, FALSE},
 /* X */ { TRUE, TRUE, TRUE, TRUE, TRUE},
 /* AI */ { FALSE, FALSE, FALSE, FALSE, TRUE}
 };
`
2. 不是插入意向锁。
3. 没有等待，LOCK_WAIT 位为0
4. LOCK_REC_NOT_GAP 位为0。（没有这个标记默认就是 NEXT KEY LOCK，锁住行前面的gap）
或者 要加锁的 LOCK_REC_NOT_GAP 位为 1
或者 当前行为 PAGE_HEAD_NO_SUPREMUM, 表示上界。
5. LOCK_GAP 位为0
 或者 要加锁的 LOCK_GAP 为 1
 或者 当前行为 PAGE_HEAD_NO_SUPREMUM, 表示上界。

如果没有更强级别的锁，就要进行锁冲突判断，如果有锁冲突就需要入队列等待，并且还要进行死锁检测。冲突判断调用函数 lock_rec_other_has_conflicting，循环的拿出对应行上的每一个锁，调用 lock_rec_has_to_wait 进行冲突判断。以下描述 “锁” 表示循环拿出的每个锁，“当前锁” 表示要加的锁。

* 如果锁和当前锁是相同的事务，返回 false，不需要等待。
* 如果锁和当前锁的基本锁类型兼容，返回 false，不需要等待。兼容性根据锁的兼容矩阵判断（感觉终于和大学课本联系起来了 T-T）。兼容矩阵：
 ` static const byte lock_compatibility_matrix[5][5] = {
 /** IS IX S X AI */
 /* IS */ { TRUE, TRUE, TRUE, FALSE, TRUE},
 /* IX */ { TRUE, TRUE, FALSE, FALSE, TRUE},
 /* S */ { TRUE, FALSE, TRUE, FALSE, FALSE},
 /* X */ { FALSE, FALSE, FALSE, FALSE, FALSE},
 /* AI */ { TRUE, TRUE, FALSE, FALSE, FALSE}
 };
`
* 如果上述两条都不满足，不是相同的事务，基本锁类型也不兼容，那么满足下面任意一条，同样返回false，不需要等待，否则返回 true，需要等待。
 
 如果当前锁锁住的是 supremum 或者 LOCK_GAP 为 1 并且 LOCK_INSERT_INTENTION 为 0。因为不带 LOCK_INSERT_INTENTION 的 GAP 锁不需要等待任何东西，不同的用户可以在 gap 上持有冲突的锁。
* 如果当前锁 LOCK_INSERT_INTENTION 为 0 并且锁是 LOCK_GAP 为 1。因为行锁（LOCK_ORDINARY LOCK_REC_NOT_GAP）不需要等待一个 gap 锁。
* 如果当前锁 LOCK_GAP 为 1，锁 LOCK_REC_NOT_GAP 为 1。同样的，因为 gap 锁没有必要等待一个 LOCK_REC_NOT_GAP 锁。
* 如果锁 LOCK_INSERT_INTENTION 为 1。此处是最后一步，说明之前的条件都不满足，源码中备注描述如下：
 
 No lock request needs to wait for an insertintention lock to be removed. This is ok since our rules allow conflicting locks on gaps. This eliminates a spurious deadlock caused by a next-key lock waiting for an insert intention lock; when the insert intention lock was granted, the insert deadlocked on the waiting next-key lock.
 Also, insert intention locks do not disturb eachother.

举个简单的例子，如果一行数据上已经加了 LOCK_S | LOCK_REC_NOT_GAP, 再尝试去加 LOCK_X | LOCK_GAP，LOCK_S 和 LOCK_X 本身是冲突的，但是满足上述第 3 个条件，返回 FALSE，不需要等待。

如果行数据上没有更强级别的锁，也没有冲突的锁，并且加的不是隐式锁，就调用 lock_rec_add_to_queue。核心思想是复用锁对象，如果要加锁的行数据上当前没有其它锁等待，并且行所在的数据页上有相似的锁对象(lock_rec_find_similar_on_page)就可以直接设置对应行的 bitmap 位，表示加锁成功。如果有其它锁等待，就重新创建一个锁对象。

## 死锁检测
死锁检测的入口函数是 lock_deadlock_check_and_resolve，算法是深度优先搜索，如果在搜索过程中发现有环，就说明发生了死锁，为了避免死锁检测开销过大，如果搜索深度超过了 200（LOCK_MAX_DEPTH_IN_DEADLOCK_CHECK)也同样认为发生了死锁。

稍早版本的时候，Innodb 使用的是递归方式搜索，为了减少栈空间的开销，改为使用入栈的方式（是否还记得大学时候严蔚敏的数据结构，有两种深度搜索的方法 T-T）。MySQL 5.7 之后增加了更多面向对象的代码结构，但是实际算法并没有改变。

两个辅助数据结构：

`/** Deadlock check context. */
struct lock_deadlock_ctx_t {
 const trx_t* start; /*!< Joining transaction that is
 requesting a lock in an incompatible
 mode */

 const lock_t* wait_lock; /*!< Lock that trx wants */

 ib_uint64_t mark_start; /*!< Value of lock_mark_count at
 the start of the deadlock check. */

 ulint depth; /*!< Stack depth */

 ulint cost; /*!< Calculation steps thus far */

 ibool too_deep; /*!< TRUE if search was too deep and
 was aborted */
};

/** DFS visited node information used during deadlock checking. */
struct lock_stack_t {
 const lock_t* lock; /*!< Current lock */
 const lock_t* wait_lock; /*!< Waiting for lock */
 ulint heap_no; /*!< heap number if rec lock */
};

`
lock_stack_t 就是辅助的栈结构，使用一个 lock_stack_t 类型的数组来作为数据栈，初始化在创建 Lock_sys 的时候，大小为 LOCK_STACK_SIZE, 实际上是 srv_max_n_thread，最大的线程数。

lock_deadlock_ctx_t 中的 start 始终保持不变，是第一个请求锁的事务，如果深度搜索过程中锁对应的事务等于 start，那么就说明产生了环，发生死锁。wait_lock 表示搜索中的事务等待的锁。

举个简单的例子：
![未命名文件 (1).png](.img/bb8778e5913d_72fff8eb62e5c8927b549268bfaafb89.png)

有三个事务 A，B，C 已经获得了三行数据 1，2，3 上的 X 锁。现在事务 A 去拿数据 2 的 X 锁，阻塞等待。同样事务 B 也去拿数据 3 的 X 锁，同样阻塞等待。当事务 C 尝试去拿 数据 1 的 X 锁时，发生死锁。看下此时的死锁检测流程：

1. ctx 中的 start 初始化为 C，wait_lock 初始化为 X1（数据1上的X锁）
2. 根据 wait_lock=X1，调用函数 lock_get_first_lock 拿到加在数据 1 上的第一个锁 lock。在例子中就是事务 A 已经获得的 X1 锁。
3. 然后判断 lock 对应的事务(A)是否也在等待其它锁：lock->trx->lock.que_state == TRX_QUE_LOCK_WAIT。当前事务 A 确实在等待 X2 锁。所以为 true，把当前的 lock 入栈(lock_dead_lock_push)。
4. ctx 中的 wait_lock 更新为 lock->trx->lock.wait_lock, 也就是 X1 锁的持有者事务 A 所等待的锁 X2。
5. 同步骤 2 ，根据 wait_lock=X2, 拿到加在数据 2 上的第一个锁赋值给 lock，也就是事务 B 持有的 X2 锁。完成一次循环。
6. 再次进入循环，lock 对应的事务(B)同样在等待其它锁，所以把当前的 lock 入栈。
7. ctx 中的 wait_lock 更新为 lock->trx->lock.wait_lock, 也就是 X2 锁持有者事务 B 所等待的锁 X3。
8. 同步骤 2，根据 wait_lock=X3, 拿到加在数据 3 上的第一个锁赋值给 lock，也就是事务 C 持有的 X3 锁。完成一次循环。
9. 再次进入循环，此时 lock->trx = C = ctx->start。死锁形成。

上述例子较为简单，没有涉及到一行数据上有多个锁，也没有出栈操作，一次深度遍历就找到了死锁，实际情况会复杂点，其它分支可以参看源码理解。

### victim 选择
当发生死锁后，会选择一个代价较少的事务进行回滚操作，选择函数：lock_deadlock_select_victim(ctx)。Innodb 中的 victim 选择比较粗暴，不论死锁链条有多长，只会在 ctx->start 和 ctx->wait_lock->trx 二者中选择其一。对应上述例子，就是在事务 B 和事务 C 中选择。

具体的权重比较函数是 trx_weight_ge, 如果一个事务修改了不支持事务的表，那么认为它的权重较高，否则认为 undo log 数加持有的锁数之和较大的权重较高。

### 死锁信息分析
当发生死锁之后，会调用 lock_deadlock_notify 写入死锁信息，SHOW ENGINE INNODB STATUS 语句可以看到最近一次发生的死锁信息，因为死锁信息是默认写到 temp 目录的临时文件中，每次发生死锁都会覆盖写。如果打开 [innodb_print_all_deadlocks](https://dev.mysql.com/doc/refman/5.6/en/innodb-parameters.html#sysvar_innodb_print_all_deadlocks)可以把历史所有的死锁信息打印到 errlog 中。

关于打印出来的内容具体含义有文章已经讲的比较清楚了：[mysql lover](http://mysqllover.com/?p=411) 和 [percona](https://www.percona.com/blog/2014/10/28/how-to-deal-with-mysql-deadlocks/)。其中推荐 percona 的文章，其实发生死锁后想找出原因的话，只有死锁信息是不够的，因为 1.只显示最近两条事务的信息 2.只显示事务最近执行的一条语句。如文中推荐的做法，配合 general log 和 binlog 进行排查。

### 锁等待以及唤醒
锁的等待以及唤醒实际上是线程的等待和唤醒，调用函数 lock_wait_suspend_thread 挂起当前线程，配合 OS_EVENT 机制，实现唤醒和锁超时等功能，这块暂且不展开，后续仔细研究后单独写一篇文章。

## Test Case 实践
在完成一个锁相关 patch 的时候发现 test case 中比较诡异的点，在执行应该产生死锁的语句时，不是每次都会产生死锁，也会发生锁超时的情况。以死锁检测中描述的例子，test case 如下：

`--disable_warnings
DROP TABLE IF EXISTS t1;
--enable_warnings
create table t1(a int primary key, b int) engine=innodb;
insert into t1 values(1,1),(2,2),(3,3),(4,4),(5,5),(6,6),(7,7),(8,8),(9,9);
# ************* 3 Transactions cause deadlock. **************** #
# Hold locks
connection con1;
begin;
select * from t1 where a=1 for update;
connection con2;
begin;
select * from t1 where a=2 for update;
connection con3;
begin;
select * from t1 where a=3 for update;

# Request locks as a circle
connection con1;
insert into t1 values(11,11);
send select * from t1 where a=2 for update;
connection con2;
insert into t1 values(12,12);
send select * from t1 where a=3 for update;
connection con3;
--error ER_LOCK_DEADLOCK
select * from t1 where a=1 for update;

`
其中插入语句是为了产生 undo log，控制那一个事务会被选为 victim。上述 test case 预期产生死锁的语句有时会报锁超时，也就是没有正确发生死锁。起初以为是 victim 选择算法的原因，后来才发现是因为 send 语句，它只保证语句发出去，并不保证执行完毕，所以在最后一条 select 语句执行的时候也许前面的语句还没执行完，无法产生死锁。

使用 wait condition 语句等待下就没问题了：

`let $wait_condition=
 SELECT COUNT(*) = 2 FROM information_schema.innodb_trx
 WHERE trx_operation_state = 'starting index read' AND
 trx_state = 'LOCK WAIT';
--source include/wait_condition.inc
--error ER_LOCK_DEADLOCK
select * from t1 where a=1 for update;
`

## 总结
Innodb 的锁系统实际上是封装了一层逻辑，和行本身数据一点关系也没有，了解之前以为会像文件锁一样，锁的粒度越小，维护起来越复杂，所以开头提到的 Berkeley DB 才只有页锁，了解之后很迷惑为什么不支持行锁… 区分一下 Innodb 同步机制使用的锁和本文介绍的锁是不同的，可以参考[这篇月报](http://mysql.taobao.org/monthly/2017/01/01/), 有一个[最不可思议的死锁](http://hedengcheng.com/?p=844)问题，就是这两种锁之间切换导致的。锁系统作为事务中一个重要模块，需要配合其它模块，对于事务系统可以参考本期月报[这篇文章](http://mysql.taobao.org/monthly/2017/12/01/)。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)