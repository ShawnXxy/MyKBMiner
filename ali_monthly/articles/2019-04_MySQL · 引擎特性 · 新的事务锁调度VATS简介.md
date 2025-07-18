# MySQL · 引擎特性 · 新的事务锁调度VATS简介

**Date:** 2019/04
**Source:** http://mysql.taobao.org/monthly/2019/04/06/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2019 / 04
 ](/monthly/2019/04)

 * 当期文章

 MySQL · 引擎特性 · 临时表那些事儿
* MSSQL · 最佳实践 · 使用SSL加密连接
* Redis · 引擎特性 · radix tree 源码解析
* MySQL · 引擎分析 · InnoDB history list 无法降到0的原因
* MySQL · 关于undo表空间的一些新变化
* MySQL · 引擎特性 · 新的事务锁调度VATS简介
* MySQL · 引擎特性 · 增加系统文件追踪space ID和物理文件的映射
* PgSQL · 应用案例 · PostgreSQL 9种索引的原理和应用场景
* PgSQL · 应用案例 · 任意字段组合查询
* PgSQL · 应用案例 · PostgreSQL 并行计算

 ## MySQL · 引擎特性 · 新的事务锁调度VATS简介 
 Author: yinfeng 

 传统的事务锁赋予方式是采用FIFS先来先服务的方式，从MySQL8.0.3开始，引入了一种新的模式CATS调度方式，全称为Contention-Aware Transaction Scheduling （或者叫做VATS, V=Variance）. 顾名思义就是能够感知到事务竞争关系来实现全局最小开销的锁调度方式。

举个简单的例子，trx1和trx2同时等待一条记录锁，按照传统的方式，谁先进入等待队列，谁将优先获得锁。但如果同时有2个事务等待trx1,10个事务等待trx2，那么从全局来看收益最大的显然是让trx2获取到行锁。

当被挂起等待的事务数超过32个时，会自动切换到新的调度方式

相关资料先列一下：

[官方博客](https://mysqlserverteam.com/contention-aware-transaction-scheduling-arriving-in-innodb-to-boost-performance/?spm=a2c4e.11153940.blogcont277682.11.43c15da8hvlVKO)

论文一：
[A Top-Down Approach to Achieving Performance Predictability in Database Systems](http://web.eecs.umich.edu/~mozafari/php/data/uploads/sigmod_2017_predictability.pdf?spm=a2c4e.11153940.blogcont277682.12.43c15da8hvlVKO&file=sigmod_2017_predictability.pdf)

论文二：
[Contention-Aware Lock Scheduling for Transactional Databases](http://web.eecs.umich.edu/~mozafari/php/data/uploads/lock-schd-report.pdf?spm=a2c4e.11153940.blogcont277682.13.43c15da8hvlVKO&file=lock-schd-report.pdf)

论文三：
[Identifying the Major Sources of Variance in Transaction Latencies: Towards More Predictable Databases](https://arxiv.org/pdf/1602.01871.pdf?spm=a2c4e.11153940.blogcont277682.14.43c15da8hvlVKO&file=1602.01871.pdf)

Release Note:

 InnoDB now uses Variance-Aware Transaction Scheduling (VATS) for scheduling the release of transaction locks when the system is highly loaded, which helps reduce lock sys wait mutex contention. Lock scheduling uses VATS when >= 32 threads are suspended in the lock wait queue.
For more information about Variance-Aware Transaction Scheduling (VATS), see Identifying the Major Sources of Variance in Transaction Latencies: Towards More Predictable Databases.

[WL#10793: InnoDB: Use CATS for scheduling lock release under high load](https://dev.mysql.com/worklog/task/?spm=a2c4e.11153940.blogcont277682.15.43c15da8KD7n8r&id=10793)

主要代码变更见这个commit: fb056f442a96114c74d291302e8c4406c8c8e1af, 或者commit log搜WL#10793关键字

这个功能的核心有两个，一个是如何去维护每个事务的权重，在代码里以trx_t::age表示，第二个是基于新的调度算法，如何去选择被调度的事务。

PS: 本文涉及函数基于MySQL8.0.3

## 何时使用VATS算法
是否使用新调度算法，需要满足如下条件(lock_use_fcfs())：

* 当前线程不是复制线程
* 并发等待线程数超过32(LOCK_VATS_THRESHOLD)

关于第二点，增加了lock_sys_t::n_waiting来追踪，在函数lock_wait_suspend_thread里递增，在lock_wait_table_release_slot里递减

## 事务权重维护
事务age的接口函数为lock_update_age 及lock_update_trx_age，在将新的事务所加入hash，或者完成一次grant操作后，都需要对事务age进行更新

先来看看函数lock_update_age是如何计算的：

* 如果当前新建的锁对象不能立刻赋予，需要等待其他锁对象时，对于已经拿到这个记录上锁的事务进行age累加，值为当前锁对象事务的trx_t::age + 1，那些等待当前新锁的事务都需要去依次更新其age;
如果当前事务无需等待，则找到那些在等待该记录锁的事务，并累加这些事务的trx_t::age+1 到当前事务
* 针对每个事务的age更新，是一个递归函数，函数接口为lock_update_trx_age
 
 将新的age值赋予给trx_t
* 如果当前事务也处于等待状态的话，则找到其等待的锁被哪些事务持有，并将age值累加上去。

 通过如上的递归流程，确保了在等待向量图中每个事务的权重被正确的更新掉

## Grant Lock
当释放掉一个锁时，需要检查是否有别的等待的锁可以获得锁，VATS调度的函数入口为lock_rec_dequeue_from_page –> lock_grant_vats

相比传统的grant方式，lock_grant_vats函数的逻辑要复杂许多：

* 将当前锁对象移除后，剩下的在同一条记录上的锁被分为两个队列：
 
 waiting队列中存储需要等待的锁对象
* granted队列存储已经获的锁对象

waiting队列会做一个排序，排序规则从comment里拷贝的，如下:

`1. If neither of them is a wait lock, the LHS one has higher priority.
2. If only one of them is a wait lock, it has lower priority.
3. If both are high priority transactions, the one with a lower seq
 number has higher priority.
4. High priority transaction has higher priority.
5. Otherwise, the one with an older transaction has higher priority.
`

简单来说，如果不考虑事务优先级，队列时按照trx_t::age进行排序。

* 然后依次遍历waiting队列，如果无需等待(lock_rec_has_to_wait_vats)， 则赋予记录锁，并将其移到哈希队列的头部
* 无论是当前释放的锁移除出锁队列，还是任一等待的事务获得了锁，都需要去更新锁等待图相关联事务权重

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)