# MySQL ·  最佳实践  ·  一次InnoDB死锁Bug排查

**Date:** 2022/02
**Source:** http://mysql.taobao.org/monthly/2022/02/01/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2022 / 02
 ](/monthly/2022/02)

 * 当期文章

 MySQL · 最佳实践 · 一次InnoDB死锁Bug排查
* DataBase · 理论基础 · HTAP列存引擎探秘
* DataBase · 理论基础 · 热点优化 (SIGMOD'21 Paper 解读)

 ## MySQL · 最佳实践 · 一次InnoDB死锁Bug排查 
 Author: 破明 

 ## 背景

某天收到一封读者的邮件，询问我一个关于 InnoDB 死锁的问题, 他在 MySQL 5.7 可以复现这个问题, MySQL 8.0.22 却无法复现, 他询问其死锁的原因. 经过一系列的排查，我后来发现是 InnoDB 内部实现的一个 Bug，目前这个 Bug 已经在 8.0.18 版本进行了修复, 所以也可以通过 8.0.17 vs 8.0.18 来验证这个问题.

整个 SQL 流程如下:

`/* 1. 表结构 */
CREATE TABLE t (
 id BIGINT UNSIGNED NOT NULL PRIMARY KEY COMMENT 'id, 无实际意义',
 account_id VARCHAR (64) NOT NULL COMMENT '用户id，不同app下的account_id可能重复',
 type TINYINT UNSIGNED NOT NULL COMMENT '余额类型 1:可用余额',
 balance BIGINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '余额',
 state INT UNSIGNED NOT NULL DEFAULT 1 COMMENT '账户状态 1:NORMAL; 2:FROZE',
 UNIQUE KEY uk_account (account_id, type)
)ENGINE = INNODB DEFAULT CHARSET utf8mb4
COMMENT '测试';

/* 2. 其中 UNIQUE INDEX 为 uk_account(account_id, type) */

/* 3. 插入数据 */
insert into t values(1,'1',1,100,1);
insert into t values(2,'2',1,100,1);
insert into t values(3,'3',1,100,1);
insert into t values(4,'4',1,100,1);
insert into t values(5,'5',1,100,1);

/* 4. 查询所有数据. */
select * from t;

/* 5. 执行以下 SQL, 注意事务隔离级别为 (RR) */
t1-1：(session1)
begin; /* 显式开启事务, 排除 autocommit 的影响. */
select * from t where account_id = '1' and type =1 for update;

t2: (session2)
begin;
select * from t where account_id = '1' and type =1 for update;

t1-2: (session1)
update t set state = 2 where account_id = '1';
`

我们按照顺序执行分别在 MySQL 8.0.17 和 MySQL 8.0.18 执行，可以看到在 8.0.17 版本事务 t2 因为死锁检测而被视为`victim_trx`进行了回滚，而 8.0.18 却不会回滚事务 t2.

基于 MySQL 8.0.17

`MySQL [sbtest]> select * from t where account_id = '1' and type =1 for update;
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
`

## 分析流程

我们基于问题版本 8.0.17 来分析 Bug 的真正原因.

### SQL 分析

通过表结构我们可以看到整个表有两个索引, PRIMARY INDEX 和 UNIQUE INDEX uk_account. 因为是死锁问题, 所以我们要逐条分析 SQL 语句加的 record lock 分别是什么:

* t1-1
t1-1 是一条 SELECT FROM UPDATE 的语句, 而`account_id`和`type`是一组唯一索引字段, 所以只需要加一个主键索引的 X record lock 和唯一索引 uk_account 的 X record lock.
* t2
t2 语句与 t1-1 相同, 加锁一致，也是一个主键索引的 X record lock 和 唯一索引 uk_account 的 X record lock.
* t1-2
t1-2 注意 t1-2 的查询条件只有where account_id = ‘1’, 这与 t1-1 的查询条件是不同的, 所以在 RR 隔离级别下，为了避免出现可能的幻读, 这需要加一个 Next-key lock, 另外需要对 record (2,’2’,1,100,1) 加一个 GAP lock, 防止在此之前的插入造成幻读.

### 锁信息验证

为了验证我们对于 SQL 的分析, 我们可以通过`set global innodb_status_output_locks = on;` 打开锁状态输出, 然后`show engine innodb status\G`, 来查看锁信息, 这里我们为了验证分析正确，跳过执行 t2 语句, 因为 t2 的加锁类型一定是与 t1-1 一致的:

`> begin;
> select * from t where account_id = '1' and type =1 for update;
> update t set state = 2 where account_id = '1';
> set global innodb_status_output_locks = on; /* 暂不提交，以查看事务锁的持有信息. */
> show engine innodb status\G

/* 事务 t1 持有 5 个 lock, 4 个 row lock, 1 个 table lock (暂时忽略). */
---TRANSACTION 2068, ACTIVE 16 sec
5 lock struct(s), heap size 1200, 4 row lock(s)
MySQL thread id 9, OS thread handle 140737025267456, query id 74 127.0.0.1 myadmin starting
show engine innodb status

/* 下列为 table lock (笔者注). */
TABLE LOCK table `sbtest`.`t` trx id 2068 lock mode IX

/* 下列为 Record: 1,'1',1,100,1 的 X record lock 基于索引 uk_account (笔者注). */
RECORD LOCKS space id 2 page no 5 n bits 72 index uk_account of table `sbtest`.`t` trx id 2068 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 1; hex 31; asc 1;;
 1: len 1; hex 01; asc ;;
 2: len 8; hex 0000000000000001; asc ;;

/* 下列为 Record: 1,'1',1,100,1 的 X record lock 基于主键索引 (笔者注). */
RECORD LOCKS space id 2 page no 4 n bits 72 index PRIMARY of table `sbtest`.`t` trx id 2068 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 8; hex 0000000000000001; asc ;;
 1: len 6; hex 000000000809; asc ;;
 2: len 7; hex 01000001160151; asc Q;;
 3: len 1; hex 31; asc 1;;
 4: len 1; hex 01; asc ;;
 5: len 8; hex 0000000000000064; asc d;;
 6: len 4; hex 00000002; asc ;;

/* 下列为 Record: 1,'1',1,100,1 的 Next-key record lock 基于索引 uk_account. (笔者注). */
RECORD LOCKS space id 2 page no 5 n bits 72 index uk_account of table `sbtest`.`t` trx id 2068 lock_mode X
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 1; hex 31; asc 1;;
 1: len 1; hex 01; asc ;;
 2: len 8; hex 0000000000000001; asc ;;

/* 下列为 Record: 2,'2',1,100,1 的 GAP lock 基于索引 uk_account. (笔者注). */
RECORD LOCKS space id 2 page no 5 n bits 72 index uk_account of table `sbtest`.`t` trx id 2068 lock_mode X locks gap before rec
Record lock, heap no 3 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 1; hex 32; asc 2;;
 1: len 1; hex 01; asc ;;
 2: len 8; hex 0000000000000002; asc ;;
`

通过`show engine innodb status\G`我们可以看到当前的事务的锁持有信息, 事务 t1 分别执行 t1-1 和 t1-2 语句后持有的锁分别有:

1. 一个主键索引的 X record lock.
2. 一个 UNIQUE INDEX 的 X record lock.
3. 一个 Next-key record lock, InnoDB 为了明确 Next-key lock 和普通的 record lock 的区别，分别用不同的 mode 来区分:

* 普通的 X record lock:

 lock_mode X locks rec but not gap. [mode: 1027 LOCK_REC_NOT_GAP
 LOCK_REC
 LOCK_X]

 Next-key lock:

 * lock_mode X [mode: 35 LOCK_X
 LOCK_REC
 LOCK_ORDINARY]

1. 一个 GAP record lock

### 死锁原因排查

既然 t2 被死锁检测回滚, 我们就需要检查当时是什么锁关系导致了死锁.

官方在 8.0.18 版本对死锁检测进行了优化, 将原先的死锁检测机制[ MySQL 死锁检测源码分析](https://leviathan.vip/2020/02/02/mysql-deadlock-check/) 交给了 background thread 来处理, 具体的 Patch 链接: [MySQL-8.0.18 死锁检测优化](https://github.com/mysql/mysql-server/commit/3859219875b62154b921e8c6078c751198071b9c). 具体的思路是将当前事务系统的 lock 信息打一份快照, 由这份快照判断是否存在回环, 假如存在死锁即唤醒等待事务.

而在 8.0.17 版本依然采用旧的死锁检测方法, 具体细节可以参考这篇文章: [MySQL 死锁检测源码分析](https://leviathan.vip/2020/02/02/mysql-deadlock-check/): 每次申请 lock 失败进入 wait 状态后触发一下死锁检测, 所以我们通过 gdb 调试的方法来梳理当时的锁依赖关系, 当我们执行完成 t1-1, 继而执行 t2 后, 事务 t2 进入了 wait 状态，当执行 t1-2 后 t2 回滚，说明触发 t2 回滚的死锁检测是由 t1-2 发起的, 我们 break 在死锁检测的路径上，然后 print 整个锁信息 (代码基于 8.0.17):

我们设置断点在死锁检测的路径上，因为可以明确是 t1-2 的死锁检测触发了 t2 的回滚，所以我们可以明确哪次 break 是我们想要的断点位置.

`(gdb) b storage/innobase/lock/lock0lock.cc:7125

/* 1. 按顺序执行 t1-1, t2, t1-2 直到 t1-2 触发死锁检测. */
/* 2. DeadlockChecker 会设置一个 m_start 即发起死锁检测的 trx 和 m_wait_lock 即 m_start 等待的 lock. */

(gdb) p m_start /* 事务t1 */
$9 = (const trx_t *) 0x7fffe506cc78

(gdb) p *m_wait_lock /* 事务 t1 尝试申请的 lock, 目前在等待状态. */
$53 = {trx = 0x7fffe506cc78, trx_locks = {prev = 0x7fffe00101f8, next = 0x0}, index = 0x7ff8740971f8, hash = 0x0, {tab_lock = {
 table = 0x500000002, locks = {prev = 0x48, next = 0x0}}, rec_lock = {space = 2, page_no = 5, n_bits = 72}},
 m_psi_internal_thread_id = 43, m_psi_event_id = 66, type_mode = 291, m_seq = 72}

/* 注意 m_wait_lock 的 type_mode */
(gdb) p /t 291
$13 = 100100011 /* 即 LOCK_X | LOCK_REC | LOCK_WAIT

(gdb) p m_wait_lock->index->name
$22 = {m_name = 0x7ff874097538 "uk_account"} /* 可以确认是等待在唯一索引 uk_account 的 record lock. */

/* 3. s 进入函数 DeadlockChecker::search(), 会首先 get_first_lock(&heap_no) 即从 m_wait_lock 对应的 record 上的 heap_no 从 lock_sys->rec_hash 找到第一个 lock. */

const lock_t *lock = get_first_lock(&heap_no);

(gdb) p lock
$15 = (const ib_lock_t *) 0x7fffe0010098

(gdb) p *lock
$14 = {trx = 0x7fffe506cc78, trx_locks = {prev = 0x7fffe0010cb8, next = 0x7fffe00101f8}, index = 0x7ff8740971f8, hash = 0x7fffe0010358, {
 tab_lock = {table = 0x500000002, locks = {prev = 0x48, next = 0x0}}, rec_lock = {space = 2, page_no = 5, n_bits = 72}},
 m_psi_internal_thread_id = 43, m_psi_event_id = 50, type_mode = 1059, m_seq = 54}

/* record 上的第一个 record 锁是 0x7fffe0010098, 它属于我们目前发起的事务 m_start 即 0x7fffe506cc78.
 type_mode 是 1059 即 LOCK_X | LOCK_REC | LOCK_REC_NOT_GAP. */

/* 我们通过 lock->hash 迭代来查看整个 record 上的锁排列信息. */
(gdb) p *(ib_lock_t*)0x7fffe0010098 /* 第一个 record lock */
$48 = {trx = 0x7fffe506cc78, trx_locks = {prev = 0x7fffe0010cb8, next = 0x7fffe00101f8}, index = 0x7ff8740971f8, hash = 0x7fffe0011918, {
 tab_lock = {table = 0x500000002, locks = {prev = 0x48, next = 0x0}}, rec_lock = {space = 2, page_no = 5, n_bits = 72}},
 m_psi_internal_thread_id = 43, m_psi_event_id = 65, type_mode = 1059, m_seq = 69}

(gdb) p *(ib_lock_t*)0x7fffe0011918 /* 第二个 record lock. */
$49 = {trx = 0x7fffe506d090, trx_locks = {prev = 0x7fffe00124c8, next = 0x0}, index = 0x7ff8740971f8, hash = 0x7fffe0010358, {tab_lock = {
 table = 0x500000002, locks = {prev = 0x48, next = 0x0}}, rec_lock = {space = 2, page_no = 5, n_bits = 72}},
 m_psi_internal_thread_id = 44, m_psi_event_id = 36, type_mode = 1315, m_seq = 71}

(gdb) p *(ib_lock_t*)0x7fffe0010358 /* 第三个 record lock. */
$50 = {trx = 0x7fffe506cc78, trx_locks = {prev = 0x7fffe00101f8, next = 0x0}, index = 0x7ff8740971f8, hash = 0x0, {tab_lock = {
 table = 0x500000002, locks = {prev = 0x48, next = 0x0}}, rec_lock = {space = 2, page_no = 5, n_bits = 72}},
 m_psi_internal_thread_id = 43, m_psi_event_id = 66, type_mode = 291, m_seq = 72}

/* 当前事务与 record 的关系如下:
发起死锁检测是 trx 是: 0x7fffe506cc78, trx 0x7fffe506cc78 等待的 lock 是 0x7fffe0010358.

lock: 0x7fffe0010098 所属的 trx: 0x7fffe506cc78 type_mode: 1059 LOCK_X | LOCK_REC | LOCK_REC_NOT_GAP 即持有 X record lock.
lock: 0x7fffe0011918 所属的 trx: 0x7fffe506d090 type_mode: 1315 LOCK_X | LOCK_WAIT | LOCK_REC | LOCK_REC_NOT_GAP 即等待 X record lock.
lock: 0x7fffe0010358 所属的 trx: 0x7fffe506cc78 type_mode: 291 LOCK_X | LOCK_REC | LOCK_WAIT 即等待 X record lock. */

/* 死锁检测流程结束后选择的 victim_trx 是: */
(gdb) p victim_trx
$58 = (const trx_t *) 0x7fffe506d090
`

通过上述分析我们可以得出结论 t1 事务的 t1-2 语句触发了死锁检测，选择的 victim_trx 是事务 t2, 我们需要明确以下几个问题:

1. 发起死锁检测的原因是因为事务 t1 无法立即获得 X record lock.
2. 事务 t1 认为可能会发生的死锁原因是因为在整个 lock 的等待关系中存在一个环, 即 t1 不 commit 提交事务, t2 事务也无法获取 X record lock, 从而导致 t1-2 的 UPDATE 语句也无法获得 X record lock 组成 Next-key record lock, 即使 t1 已经持有了 X record lock.

### 解决方案

根据最近的 Release Note, 我二分验证 8.0.16 - 8.0.22 的版本, 发现在 8.0.17 存在问题, 8.0.18 不存在, 所以根据现象我仔细查看了 8.0.18 的 [Release Note](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-18.html), 发现了疑似这个现象的 Bugfix:

 * InnoDB: A deadlock was possible when a transaction tries to upgrade a record lock to a next key lock. (Bug #23755664, Bug #82127)

根据 Bug ID, 可以通过 Github 的 MySQL 提交记录来查找这个 [Patch](https://github.com/mysql/mysql-server/commit/85927b60bc658ddfffcc3aeab15d7553d163b0be):

 Bug #23755664 DEADLOCK WITH 3 CONCURRENT DELETES BY UNIQUE KEY

 PROBLEM:
A deadlock was possible when a transaction tried to “upgrade” an already held Record Lock to Next Key Lock.

 SOLUTION:
This patch is based on observations that:
(1) a Next Key Lock is equivalent to Record Lock combined with Gap Lock
(2) a GAP Lock never has to wait for any other lock
In case we request a Next Key Lock, we check if we already own a Record Lock of equal or stronger mode,
and if so, then we either upgrade it to Next Key Lock, or if it is not possible (because the single lock_t
struct is shared by more than one row) we change the requested lock type to GAP Lock, which we either already
have, or can be granted immediately.
(I don’t consider Insert Intention Locks a Gap Lock in above statements).

 Reviewed-by: Debarun Banerjee [debarun.banerjee@oracle.com](mailto:debarun.banerjee@oracle.com)
RB:19879

经过验证确实是这个 Patch 修复了这个死锁的问题.

## Patch 分析

这个 Patch 具体的原理是当尝试获取 Next-key record lock 时，不再与旧的逻辑一样，旧的逻辑是先直接尝试申请 Next-key lock, 现在改为先判断当前 trx 是否持有 X record lock, 假如持有就复用这个 X record lock, 从而直接申请 GAP lock, 以达到 Next-key Lock 的效果.

所以在我们上面的例子中，申请 Next-key record lock 时跳过申请 X record lock, 就不会进入等待队列，也不会产生死锁的回环.

`/* 使用 8.0.26 最新版分析 lock 持有情况.
 * 执行 t1：
 select * from t where account_id = '1' and type =1 for update;
 update t set state = 2 where account_id = '1';
 show engine innodb status\G; */

show engine innodb status
TABLE LOCK table `sbtest`.`t` trx id 1323 lock mode IX

/* 下列为 Record: 1,'1',1,100,1 的 X record lock 基于索引 uk_account. (笔者注). */
RECORD LOCKS space id 2 page no 5 n bits 72 index uk_account of table `sbtest`.`t` trx id 1323 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 1; hex 31; asc 1;;
 1: len 1; hex 01; asc ;;
 2: len 8; hex 0000000000000001; asc ;;

/* 下列为 Record: 1,'1',1,100,1 的 X record lock 基于主键索引. (笔者注). */
RECORD LOCKS space id 2 page no 4 n bits 72 index PRIMARY of table `sbtest`.`t` trx id 1323 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 7; compact format; info bits 0
 0: len 8; hex 0000000000000001; asc ;;
 1: len 6; hex 00000000052b; asc +;;
 2: len 7; hex 020000011b0110; asc ;;
 3: len 1; hex 31; asc 1;;
 4: len 1; hex 01; asc ;;
 5: len 8; hex 0000000000000064; asc d;;
 6: len 4; hex 00000002; asc ;;

/* 下列为 Record: 1,'1',1,100,1 的 GAP record lock 基于索引 uk_account, 与上面的 uk_account 的 X lock 组成 Next-key Lock. (笔者注). */
RECORD LOCKS space id 2 page no 5 n bits 72 index uk_account of table `sbtest`.`t` trx id 1323 lock_mode X locks gap before rec
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 1; hex 31; asc 1;;
 1: len 1; hex 01; asc ;;
 2: len 8; hex 0000000000000001; asc ;;

/* 下列为 Record: 2,'2',1,100,1 的 GAP record lock 基于索引 uk_account. (笔者注: lock 的类型与上列一致, InnoDB 在这里对相同类型的 lock 做了省略). */
Record lock, heap no 3 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 1; hex 32; asc 2;;
 1: len 1; hex 01; asc ;;
 2: len 8; hex 0000000000000002; asc ;
`

## 总结

根据例子我们分析了一个 InnoDB 的死锁场景, 以及 Bug 产生的原因. 通过 gdb 调试的方式分析 InnoDB 的死锁原因，最主要任务就是梳理整个锁的等待依赖关系, 这能帮助我们更直观的分析真正的原因. 这是一个 X record lock “升级” 至 Next-key record lock 的 Bug, 官方在 8.0.18 已经修复了这个存在了几年的问题.

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)