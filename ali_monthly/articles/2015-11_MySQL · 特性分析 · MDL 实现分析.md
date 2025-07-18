# MySQL · 特性分析 · MDL 实现分析

**Date:** 2015/11
**Source:** http://mysql.taobao.org/monthly/2015/11/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 11
 ](/monthly/2015/11)

 * 当期文章

 MySQL · 社区见闻 · OOW 2015 总结 MySQL 篇
* MySQL · 特性分析 · Statement Digest
* PgSQL · 答疑解惑 · PostgreSQL 用户组权限管理
* MySQL · 特性分析 · MDL 实现分析
* PgSQL · 特性分析 · full page write 机制
* MySQL · 捉虫动态 · MySQL 外键异常分析
* MySQL · 答疑解惑 · MySQL 优化器 range 的代价计算
* MySQL · 捉虫动态 · ORDER/GROUP BY 导致 mysqld crash
* MySQL · TokuDB · TokuDB 中的行锁
* MySQL · 捉虫动态 · order by limit 造成优化器选择索引错误

 ## MySQL · 特性分析 · MDL 实现分析 
 Author: 襄洛 

 ## 前言

在MySQL中，DDL是不属于事务范畴的，如果事务和DDL并行执行，操作相关联的表的话，会出现各种意想不到问题，如[事务特性被破坏](bug地址)、[binlog顺序错乱](http://bugs.mysql.com/bug.php?id=989)等，为了解决类似这些问题，MySQL在5.5.3引入了MDL锁(Metadata Locking)，关于其设计思路可以参考这两个worklog：[WL#3726](http://dev.mysql.com/worklog/task/?id=3726) 和 [WL#4284](http://dev.mysql.com/worklog/task/?id=4284)。本篇从代码实现角度对MDL进行分析。

## 重要数据结构

MDL 是在 MySQL server 层实现的一个模块，通过对外接口和server层其它模块进行交互，在sql/mdl.h和sql/mdl.cc中实现。

1. `enum_mdl_type`，枚举类型，表示MDL锁的类型，目前一共9种

 ` * MDL_INTENTION_EXCLUSIVE IX // 意向X锁，只用于scope 锁
 * MDL_SHARED S // 只能读metadata，当能读写数据，如检查表是否存在时用这个锁
 * MDL_SHARED_HIGH_PRIO SH // 高优先级S锁，可以抢占X锁，只能读metadata，不能读写数据，用于填充INFORMATION_SCHEMA，或者show create table时
 * MDL_SHARED_READ SR // 可以读表数据，select语句，lock table xxx read 都用这个
 * MDL_SHARED_WRITE SW // 可以更新表数据，insert，update，delete，lock table xxx write, select for update，
 * MDL_SHARED_UPGRADABLE SU // 可升级锁，可以升级为SNW或者X锁，ALTER TABLE第一阶段会用到
 * MDL_SHARED_NO_WRITE SNW // 可升级锁，其它线程能读metadata，数据可读不能读，持锁者可以读写，可以升级成X锁，ALTER TABLE的第一阶段
 * MDL_SHARED_NO_READ_WRITE SNRW // 可升级锁，其它线程能读metadata，数据不能读写，持锁者可以读写，可以升级成X锁，LOCK TABLES xxx WRITE
 * MDL_EXCLUSIVE X // 排它锁，禁止其它线程的所有请求，CREATE/DROP/RENAME TABLE
`
2. `enum_mdl_duration`，枚举类型，表示持有MDL锁的时间

 ` * MDL_STATEMENT // 语句范围的，语句结束自动释放
 * MDL_TRANSACTION // 事务范围的，事务结束时自动释放
 * MDL_EXPLICIT // 显式锁，由lock tables xxx read 这种获取，需要通过unlock tables释放
`
3. `MDL_key`, 对MDL锁的一个标识，是个三元组：namespace + db_name + table_name

 ` * m_ptr // 字符串数组，三元组就存在这里
 - enum_mdl_namespace // 内部定义的一个枚举类型，表示加锁对象的类型
 * GLOBAL // 全局读锁，FLUSH TABLES WITH READ LOCK
 * SCHEMA // 数据库锁
 * TABLE // 表锁
 * FUNCTION // 函数锁
 * PROCEDURE // 存储过程
 * TRIGGER // 触发器
 * EVENT // event事件
 * COMMIT // 全局commit锁，FLUSH TABLES WITH READ LOCK
`
4. `MDL_request`, 线程的锁请求，这个会发送给MDL子系统，包含加锁对象（MDL_key）、加什么类型锁（enum_mdl_type）、锁持有时间（enum_mdl_duration）等信息

 ` * type // 类型是enum_mdl_type，表示锁请求的类型
 * duration // 类型是enum_mdl_duration，表示锁的持有时间
 * next_in_list // 当前线程中下一个MDL_request指针，和prev_in_list一起所有MDL_request串起来，形成双向链表
 * prev_in_list // 见上
 * ticket // 加锁成功后，MDL模块会返回一个ticket
 * key // MDL_key
`
5. `MDL_ticket`, MDL子系统内部对加锁请求或已获得锁的表示，对MDL来说非常重要，同时是`MDL_wait_for_subgraph`的子类，线程的锁等待图就通过ticket构建出来。

 ` * next_in_context // 和prev_in_context一起构造在当前context下所有的ticket双向链表
 * prev_in_context // 见上
 * next_in_lock // 和prev_in_lock一起构造当前MDL_lock的等待和持有ticket双向链表
 * prev_in_lock // 见上
 - has_pending_conflicting_lock // 当前ticket的锁类型是否和对应MDL锁的等待队列中的锁冲突
 - is_upgradable_or_exclusive // 是否是可以升级或者互斥锁
 - has_stronger_or_equal_type // 当前ticket对应的锁和指定的锁比较是否更强（如X比S强）
 - is_incompatible_when_granted // 是否能加锁
 - is_incompatible_when_waiting // 是否比等待队列中的tciket类型优先级更高
 - accept_visitor // 死锁检测用到
 - get_deadlock_weight // 拿一个死锁权重，死锁检测用
 * m_type // 锁类型
 * m_duration // 持有时间，debug 模式下有效
 * m_ctx // 指向所属context
 * m_lock // 指向请求的锁对象
`
6. `MDL_wait`，锁等待实现，当拿不到锁时就要进入等待，等待的结果也存在这里面

 ` - enum_wait_status // 锁等待退出时的状态
 * EMPTY // 初始化值
 * GRANTED // 加锁成功，拿到锁
 * VICTIM // 等待的时候，死锁检测发现死锁，当前线程选为victim，加锁失败
 * TIMEOUT // 加锁超时，加锁失败
 * KILLED // 连接被kill，加锁失败
 - timed_wait // 等待的实现，条件变量+超时
`
7. `MDL_context`，在MDL子系统中，对应一个线程，thd和MDL系统交互就通过这个类实现

 ` - try_acquire_lock // 尝试加锁，加锁失败就返回，没有死锁检测
 - acquire_lock // 加一个锁，和上面的区别是多了死锁检测
 - acquire_locks // 一次性加多个排它锁，要么成功，要么全失败
 - upgrade_shared_lock // 升级共享锁
 - clone_ticket // clone 出一个 ticket
 - release_all_locks_for_name // 把当前线程对某个对象加的所有MDL锁都释放掉
 - release_lock // 释放单个锁
 - is_lock_owner // 是否持有某个对象的锁
 - has_lock // 线程是否否在savepoint之前持有指定的锁
 - has_locks // 当前线程是否持有锁
 - set_explicit_duration_for_all_locks // 锁的时间范围都置为显式
 - set_transaction_duration_for_all_locks // 锁的时间范围都置为事务
 - set_lock_duration // 设置锁的时间范围
 - release_statement_locks // 释放所有语句时间范围的锁
 - release_transactional_locks // 释放所有事务时间范围的锁
 - rollback_to_savepoint // MDL 锁回滚到某个savepoint
 - get_deadlock_weight // 死锁时拿一个权重值，以此来判断对应线程是否要做为victim
 * m_wait // 锁等待
 * m_tickets // 指针数组，每个元素指向一个ticket链表，分别对应当前线程的语句范围锁、事务范围锁和显式锁
 * m_owner // 指向thd的指针
 * m_waiting_for // 当前线程正在等待的锁
 - find_ticket // 在当前线程ticket链表中查找一个ticket
 - release_locks_stored_before // 释放ticket链表上在某个ticket之前所有ticket
 - find_deadlock // 检测是否有死锁
 - visit_subgraph // 和死锁检测相关
`
8. `MDL_map`，MDL_key 到 MDL_lock 的一个映射，MDL模块内部用，MDL系统所有锁都放在这个Map里

 ` - init // 初始化
 - destroy
 - find_or_insert // 查找对应的MDL_lock，没有的话新建并插入
 - remove // 移除MDL_lock
 * m_partitions // MDL_map 分区
 * m_global_lock // 预先分配的全局读锁
 * m_commit_lock // 预先分配的全局commit锁
`
9. `MDL_map_partition`，为了提升MDL模块的扩展性，把原本的一个MDL_map分成多个分区，每个分区就是一个 `MDL_map_partition`

 ` - find_or_insert // 当前分区中查找对应的MDL_lock，没有的话新建并插入
 - remove // 在当前分区中移除MDL_lock
 - move_from_hash_to_lock_mutex // 锁转换，释放对分区的加锁(MDL_map_partition::m_mutex)，获取lock对象的锁(MDL_lock::m_rwlock)
 * m_mutex // 对分区对象的一个保护锁，修改当前分区要拿到这个锁
 * m_unused_locks_cache // 释放掉的锁对象的一个缓存，不用再新分配内存
`
10. `MDL_lock`，MDL锁对象，对于一个key组合（三元组），整个系统只有一个锁对象，不管请求的key是什么类型，什么时间范围

 `- Ticket_list // 一个内部嵌套类，用于表示当前MDL锁相关的ticket列表，是个list
 - add_ticket // 增加 ticket
 - remove_ticket // 移除 ticket
 - is_empty // list 是不是空的
 - clear_bit_if_not_in_list // 如果当前list中没有某种类型的ticket，就把对应的位清掉
 * m_list // 存放ticket的list
 * m_bitmap // 标识当前list中所有ticket类型对应bit位的bitmap，实例是个short类型
* key // 当前锁对应的MDL_key
* m_rwlock // 对MDL_lock锁对象的保护锁
- has_pending_conflicting_lock // 已经授权的ticket是否和等待队列中的ticket不兼容
- can_grant_lock // 能否加锁，先和等待队列进行优先级比较，然后看和已授权的锁是否兼容
- reschedule_waiters // 当持有当前锁的ticket释放或者降级时，会调用下，看等待队列里是否有ticket此时可以获取锁
- remove_ticket // 从指定队列中移出ticket
- visit_subgraph // 死锁检测相关
- needs_notification // 是否需要通知其它线程，当前ticket的锁情况
- notify_conflicting_locks // 通知其它线程，有一个高级的锁请求
- hog_lock_types_bitmap // 标识哪种锁是高级锁
* m_granted // 已经获得当前MDL锁的ticket队列
* m_waiting // 等待当前MDL锁的ticket队列
* m_hog_lock_count // 高级锁可以连接拿得锁的个数，超过这个数目就要给低级锁让路，防止低级锁饿死
* m_ref_usage // 和下面2个变量一起，为了提高锁的扩展性
* m_ref_release
* m_is_destroyed
* m_version // 用于判断锁对象是否被放入unsed队列
* m_map_part // 当前MDL锁所在的MDL_map 分区
`
11. `MDL_scoped_lock`，MDL_lock的一个子类，主要用于对schema加MDL锁，全局读锁和全局commit锁也是这种类型。
12. `MDL_object_lock`，MDL_lock的另一个子类，除了`MDL_scoped_lock`外，其它都用这个(table、fucntion等)，只有 `MDL_object_lock` 可以缓存。

总结下，上面这些类中，`MDL_key` 和 `MDL_request` 都是POD，用来保存信息的；`MDL_context`是MDL子系统和线程交互的接口，一个对象对应一个线程；`MDL_map`、`MDL_map_partition` 和 `MDL_lock` 都是MDL子系统内部实现细节，对server层其它部分不可见；`MDL_ticket` 表示线程对`MDL_lock`持有的某种锁。

MDL锁可以从不同角度进行分类：

1. namespace，如GLOBAL、SCHEMA、TABLE等；
2. 锁的持续时间，如transaction、显式等；
3. 锁的兼容性，如S、X、SH等；
4. 锁的实现类，如scope，object等；

可以看作是MDL锁的不同属性，大家不要搞乱了 :-)

## 模块初始化

整个MDL模块的初始化是在mysqld启动时进行的，初始化逻辑在 `MDL_map::init()` 中，做的事情也比较简单：

1. 初始化两个全局MDL锁，global lock 和 commit lock，两者都是类型都是`MDL_scoped_lock`；
2. 分配`metadata_locks_hash_instances`个map分区，为了解决MDL模块全局锁竞争问题，在5.6.8对MDL锁做了分区(commit)，通过`metadata_locks_hash_instances`配置指定用多少个分区，默认是8个。

## 加锁

加锁就是server的线程（thd）向MDL模块获取对应锁的ticket过程，加锁成功标志是MDL模块返回一个对应的ticket，大致逻辑如下：

1. 线程解析SQL语句，根据语义对每一个表对象设置`TABLE_LIST.mdl_request`，如对普通的select语句 `TABLE_lsit.mdl_request.type` 就是`MDL_SHARED_READ`，可以参考函数`st_select_lex::set_lock_for_tables()`；
2. 线程在打开每个表之前，会请求和这个表对应的MDL锁，通过 `thd->mdl_context.acquire_lock()` 等接口将`mdl_request`请求发给MDL模块;
3. MDL模块根据请求类型和已有的锁来判断请求能否满足，如果可以就返回一个ticket；如果不可以就等待，等待结果可以是成功（别的线程释放了阻塞的MDL锁）或者失败（超时、连接被kill或者被死锁检测选为victim）；
4. 线程根据MDL模块的返回结果，决定继续往下走还是报错退出。

 需要注意的是，MDL锁并不是对表加锁，而是在加表锁前的一个预检查，如果能拿到MDL锁，下一步加相应的表锁。

下面对MDL模块中的主要加锁方法进行介绍。

**MDL_context::find_ticket**
这是一个shortcut方法，加锁的时候先检查当前线程是否已持有对应key的MDL锁，并且这个锁的类型不比请求的低，那么就不需要经过MDL系统再分配一个ticket出来（这个比较复杂，代价较高），直接使用已有的ticket，或者clone一个。

举个例子:

`1. begin;
2. insert into t1 values (1);
3. insert into t1 values (2);
 ...
`

在上面的语句序列中，执行语句3的时候就不需要再走一遍复杂的加锁逻辑，因为语句2已经成功拿到t1表的ticket，类型都是MDL_SHARED_WRITE，并且MDL锁时间范围也一样（transaction），这个时候直接用已有的ticket，甚至不用clone。

**MDL_context::clone_ticket**
经过检测发现可以直接使用已有的ticket，比如上面的`MDL_context::find_ticket`发现了可以复用的ticket，但是锁时间范围不一致，为了确保已经有锁释放时，不影响现在请求的，就clone一个ticket。

`1. begin;
2. insert into t1 values (1);
3. handler t1 open;
 ...
`

在上面的语句序列中，执行语句3的时候，发现有可以复用的ticket（语句2的ticket），但是handler需要的MDL锁是显式的，而语句2取得的ticket是事务时间范围的，事务完成后就会释放，为了避免handler的MDL锁被提前释放，因此单独clone一个出来用。

**MDL_context::try_acquire_lock_impl**
无等待的加锁，如果发现有冲突导致加锁失败，直接退出。会先调用`MDL_context::find_ticket`看是否有可以复用的ticket，有的话就返回成功，如果没有就看能否加锁，能加的话也返回成功，不能加也直接返回(同时返回一个ticket给调用者)。

**MDL_context::acquire_lock**
主加锁函数，调试MDL锁相关问题时，给这个函数加断点比较有效。先调用`MDL_context::try_acquire_lock_impl`，如果加锁失败就进入等待加锁逻辑：

1. 将`MDL_context::try_acquire_lock_impl`返回的ticket放进MDL_lock的等待队列；
2. 触发一次死锁检测（后面会详细介绍）；
3. 进入等待，这个时候如果我们`show processlist`就会看到”Waiting for table metadata lock”之类state。等待又分为2种：
 * 定时检查等待: 如果当前请求的锁是比较高级的（对于`MDL_object_lock`是比MDL_SHARED_NO_WRITE类型更高，对于`MDL_scoped_lock`是MDL_SHARED类型），就会每秒给其它持有当前锁的线程（并且这些连接持有的锁等级比较低）发信号，通知其释放锁，然后再检查是否锁已拿到；
* 一直等待，直到超时；
4. 检查步骤3的等待结果，可以是GRANTED（拿到锁）、VICTIM（被死锁检测算法选为受害者）、TIMEOUT（加锁超时）、KILLED（连接被kill）。拿到锁返回成功，其它返回失败。

 锁等待是靠`MDL_wait`这个类来实现的。

**MDL_context::acquire_locks**
一次性加多个排它MDL锁，如果其中一个加锁失败，前面已经拿到的锁也全部释放。主要用在DDL中，比如`drop table test.t1`这个DDL会一次加3个锁：

* GLOBAL，MDL_INTENTION_EXCLUSIVE
* test 库, MDL_INTENTION_EXCLUSIVE
* test.t1 表，MDL_EXCLUSIVE

**MDL_context::upgrade_shared_lock**
锁升级，从共享锁升级到互斥锁，实现方式是重新申请一个目标锁，拿到新的ticket后替换老的ticket，用在alter table和create table场景中。

如`create table test.t1(id int) engine = innodb`，会先拿test.t1的MDL_SHARED共享锁，检查表是否存在，如果不存在就把锁升级到MDL_EXCLUSIVE锁，然后开始建表。

对于`alter table test.t1 add column name varchar(10), algorithm=copy;`，alter用copy到临时的方式来做。整个过程中MDL顺序是这样的：

1. 刚开始打开表的时候，用的是 MDL_SHARED_UPGRADABLE 锁；
2. 拷贝到临时表过程中，需要升级到 MDL_SHARED_NO_WRITE 锁，这个时候其它连接可以读，不能更新；
3. 拷贝完在交换表的时候，需要升级到是MDL_EXCLUSIVE，这个时候是禁止读写的。

所以在用copy算法alter表过程中，会有2次锁升级。

**MDL_ticket::downgrade_lock**
和`MDL_context::upgrade_shared_lock`对应的锁降级，从互斥锁降级到共享锁，实现比较简单，直接把锁类型改为目标类型（不用重新申请）。

对于`alter table test.t1 add column name varchar(10), algorithm=inplace`，如果alter使用inplace算法的话，整个过程中MDL加锁顺序是这样的：

1. 和copy算法一样，刚开始打开表的时候，用的是 MDL_SHARED_UPGRADABLE 锁；
2. 在prepare前，升级到MDL_EXCLUSIVE锁；
3. 在prepare后，降级到MDL_SHARED_UPGRADABLE（其它线程可以读写）或者MDL_SHARED_NO_WRITE（其它线程只能读不能写），降级到哪种由表的引擎决定；
4. 在alter结束后，commit前，升级到MDL_EXCLUSIVE锁，然后commit。

可以看到inplace有2次锁升级，1次降级，不过在alter最耗时的阶段是有可能降级到MDL_SHARED_UPGRADABLE的，对其它线程的影响小。

**MDL_context::release_locks_stored_before**
释放线程指定ticket链表上某个ticket之前的所有ticket，每个context有3个ticket链表(statement、transaction和explicit)，分别对应当前线程持有的不同时间范围的MDL锁。而ticket在链表中的顺序和时间顺序是相反的，后插入的ticket放在链表开头，因此本函数的作用就是把某个时间点之后的ticket都释放掉，回滚MDL锁。有几个指释放MDL锁的函数都是基于此实现：

1. `MDL_context::rollback_to_savepoint`，把存档点之后的所有MDL锁都释放掉；
2. `MDL_context::release_transactional_locks`，释放所有transaction和statement时间范围的MDL锁；
3. `MDL_context::release_statement_locks()`，释放所有statement时间范围的MDL锁。

## 死锁检测

MDL模块作为一个集中的资源，收到不同线程发来的锁请求，而有的锁是互斥的，不能同时满足，在这种情况下就会等待，如果线程在此之前已经拿到某些锁的话，就会形成持有-等待的状态；而又不可能要求所有线程按某一固定顺序请求锁，这样就会形成等待循环，也就是死锁，如下图所示：

线程T1持有M1，然后请求M2，但M2被线程T2持有，并且和T1的请求类型互斥，同时T2请求M1，和T1拿到的锁互斥，形成死锁。

在介绍MDL的死锁检测之前，先介绍下MDL锁的兼容矩阵。每种类型的锁各有2个兼容矩阵，granted matrix 和 waiting matrix，前者表示锁的兼容性，后者表示锁的优先级（优先级就是和等待队列的锁相比，当前锁是否能够进行加锁尝试，当前锁优先级高则可以，低则需进等待队列）。

矩阵中 ‘+’ 表示兼容，’-‘ 表示不兼容，’0’ 表示不可能存在的场景。

`MDL_scoped_lock`，支持IX，S和X锁（关于锁的缩写可以看第一节）。

1. granted matrix

 ` | Type of active |
 Request | scoped lock |
 type | IS(*) IX S X |
 ---------+------------------+
 IS | + + + + |
 IX | + + - - |
 S | + - + - |
 X | + - - - |
`
2. waiting matrix

 ` | Pending |
 Request | scoped lock |
 type | IS(*) IX S X |
 ---------+-----------------+
 IS | + + + + |
 IX | + + - - |
 S | + + + - |
 X | + + + + |
`

IS锁虽然列了出来，但是代码里并没有实现这个锁，因为IS和所有的锁类型都兼容（也可以理解为每次锁请求都默认会额外有一个IS锁）。

`MDL_object_lock`，支持S、SH、SR、SW、SU、SNW、SNRW 和 X锁。

1. granted matrix

 ` Request | Granted requests for lock |
 type | S SH SR SW SU SNW SNRW X |
 ----------+----------------------------------+
 S | + + + + + + + - |
 SH | + + + + + + + - |
 SR | + + + + + + - - |
 SW | + + + + + - - - |
 SU | + + + + - - - - |
 SNW | + + + - - - - - |
 SNRW | + + - - - - - - |
 X | - - - - - - - - |
 SU -> X | - - - - 0 0 0 0 |
 SNW -> X | - - - 0 0 0 0 0 |
 SNRW -> X | - - 0 0 0 0 0 0 |
` 

 关于’0’的情况说明下，比如对于SU锁来说其和自身是不兼容的，不可能有2个线程对同一个对象都持有SU锁，所以就不存在当一个线程进行锁升级时，另一个线程持有SU。其它’0’的情况类似。
2. waiting matrix

 ` Request | Pending requests for lock |
 type | S SH SR SW SU SNW SNRW X |
 ----------+---------------------------------+
 S | + + + + + + + - |
 SH | + + + + + + + + |
 SR | + + + + + + - - |
 SW | + + + + + - - - |
 SU | + + + + + + + - |
 SNW | + + + + + + + - |
 SNRW | + + + + + + + - |
 X | + + + + + + + + |
 SU -> X | + + + + + + + + |
 SNW -> X | + + + + + + + + |
 SNRW -> X | + + + + + + + + |
` 

 注意 SH 比 X 锁的优先级还高，正是其高优先级(high priority)的体现。

在MDL系统中，资源关系是这样的：

1. 线程和锁的关系通过ticket建立；
2. 每个线程有3个ticket链表，分别对应当前持有的statement锁、transaction锁和显式锁，放在 `MDL_context::m_tickets`中；对于当前线程正在等待的锁只有一个，用`MDL_context::m_waiting_for`表示；
3. 每个MDL锁有2个ticket链表，分别对应已经获得锁的线程（`MDL_lock::m_granted`）和等待锁的线程（`MDL_lock::m_waiting`）；
4. 线程的ticket链表和MDL锁的ticket链表一起构成了MDL系统的等待关系图，死锁检测就是搜索这张图，看是否有环路。

为了描述的简洁，我们将线程和MDL锁的ticket链表都简化为1个，如下图2矩阵的，横线表示线程的链表，纵向表示MDL锁的链表，有色彩的交点表示一个ticket，橘黄色表示连接已经拿到锁，青色表示正在等待的锁，图中MDL上锁的类型不兼容，形成持有等待回路——死锁。

下面介绍下死锁检测中的函数。

**MDL_context::find_deadlock**
这个是死锁检测的入口，线程在`MDL_context::acquire_lock`尝试拿锁失败，进入等待之前，会调用这个函数进行一次死锁检测。

函数会行循环检测，直到发现没有死锁（每轮检测会去掉等待图中一条边，但不保证能解决死锁，所以需要循环），或者当前线程被选为victim才退出。

**MDL_context::visit_subgraph**
看当前线程是否有锁等待`MDL_context::m_waiting_for`，有的话就沿着ticket搜下去，没有就退出。

**MDL_ticket::accept_visitor**
这个方法看起来没有什么实际内容，只是简单调用`MDL_lock::visit_subgraph`，其实可以看作是搜索视角的转换，从 `MDL_context` 经过 `MDL_ticket` 进入到 `MDL_lock`，代码逻辑显得比较清晰。

**MDL_lock::visit_subgraph**
这个是死锁检测核心逻辑：

1. 先给搜索深度加1，然后判断是否超过最大搜索深度（MAX_SEARCH_DEPTH= 32），超过就无条件认为有死锁，退出；
2. 遍历当前锁的ticket链表，看ticket对应的线程是否和死锁检测的发起线程是同一个，如果是则说明有回路，退出（相当于做了一层的广度搜索）；
3. 从头开始遍历当前锁的ticket链表，对每个ticket对应的线程，递归调用`MDL_context::visit_subgraph`（深度搜索）。

整个死锁检测逻辑是一个加了深度限制的深搜，中间同时多了一层广搜。

**Deadlock_detection_visitor** 是死锁检测中重要的辅助类，主要负责：

1. 记录死锁检测的起始线程；
2. 记录被选做victim的线程；
3. 在检测到死锁，深搜一层层退出的时候，会依次检查回路上各线程的死锁权重，选择权重最小的做为最终的victim（权重由锁的类型决定）。

## global read lock

相信FTWRL（FLUSH TABLES WITH READ LOCK）这个命令很多人都用过，比如备份时为了获取SQL线程执行位点或binlog位点，这个命令的目的是阻止新的更新进来和已有事务的提交。就这个命令主要靠MDL锁来实现，这里用到了2个MDL锁，namespace分别为`MDL_key::GLOBAL`和`MDL_key::COMMIT`，这2个锁在整个MDL系统中都是全局唯一的，都是`MDL_scoped_lock`类型。

执行 FTWRL 的线程会请求这2个锁的MDL_SHARED锁，并且是显式的。在所有更新数据的代码路径里，除了必须的锁外，还会额外请求`MDL_key::GLOBAL`锁的MDL_INTENTION_EXCLUSIVE锁；在事务提交前，会先请求`MDL_key::COMMIT`锁的MDL_INTENTION_EXCLUSIVE锁。对于scope锁来说，IX锁和S锁是不兼容的（参考granted matrix），所以更新和事务提交都被FTWRL挡到了。

Percona Server 实现的相对于 FTWRL 轻量级的backup锁也是基于MDL实现的，其对MDL_key的 namespace 额外扩展了2个，`MDL_key::BACKUP`和`MDL_key::BINLOG`，对应的2个锁也是全局唯一的，感兴趣的同学可以了解下[backup locks](https://www.percona.com/doc/percona-server/5.6/management/backup_locks.html)。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)