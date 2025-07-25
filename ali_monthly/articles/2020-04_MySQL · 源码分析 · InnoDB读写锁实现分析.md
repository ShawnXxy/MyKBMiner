# MySQL · 源码分析 · InnoDB读写锁实现分析

**Date:** 2020/04
**Source:** http://mysql.taobao.org/monthly/2020/04/02/
**Images:** 4 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2020 / 04
 ](/monthly/2020/04)

 * 当期文章

 PostgreSQL · 源码分析 · 回放分析（一）
* MySQL · 源码分析 · InnoDB读写锁实现分析
* MySQL · 最佳实践 · X-Engine并行扫描
* MySQL · 引擎特性 · 8.0 Window Functions 剖析
* MySQL · 引擎特性 · Performance_schema 内存分配
* MySQL · 引擎特性 · 手动分析InnoDB B+Tree结构
* Redis · 最佳实践 · 集群配置：Redis Cluster
* MongoDB · 引擎特性 · 大量集合启动加载优化原理
* MySQL · 引擎特性 · 8.0 Lock Manager

 ## MySQL · 源码分析 · InnoDB读写锁实现分析 
 Author: wanghu.zc 

 ## 1 背景

在InnoDB中，当多线程需要访问共享数据结构时，InnoDB使用互斥锁(mutex)和读写锁(rwlock)来同步这些并发操作。InnoDB的读写锁实现并不是对pthread rwlock的直接封装，而是基于原子操作，自旋锁和条件变量进行实现，大大减少了进入内核态进行同步操作的概率，提高了性能，和在现代多核处理器架构下的可扩展性。

本文分析了InnoDB读写锁的具体实现，所有分析基于MySQL 8.0.18代码。

## 2 锁模式

InnoDB的读写锁有三种基本模式：S（Shared），X（Excluded）和SX（Shared Excluded）。它们的锁兼容性关系如下表所示：

 S
 SX
 X

 S
 兼容
 兼容
 冲突

 SX
 兼容
 冲突
 冲突

 X
 冲突
 冲突
 冲突

### 2.1 SX锁的含义

S和X模式比较好理解是经典的读写锁两种模式。SX模式是对X模式的一种优化，它与读操作的S模式兼容，但是多个SX锁之间是冲突的。

典型的应用场景是对dict_index_t.lock冲突的优化。在过去，当插入操作会造成B+ Tree Node分裂时，使用悲观模式插入记录。此时，需要在dict_index_t.lock上加X锁，要修改的所有相关Leaf Page上加X锁，完成后开始对Branch Node进行修改，而Branch Node上不需要加任何锁。当以这种模式插入时，将阻塞所有在该 B+ Tree上的搜索操作，因为搜索操作的第一步就是在dict_index_t.lock上加S锁。

通过SX锁可以优化该场景：悲观模式的插入操作在dict_index_t.lock上加SX锁，同时在需要修改的Branch Node上加X锁，此时因为在dict_index_t.lock上加的是SX锁，就不会阻塞所有在B+ Tree上的搜索操作，把阻塞范围缩小到访问同一个Branch Node的插入和搜索操作之间。

## 3 锁状态的维护

InnoDB rw_lock_t 仅使用一个64 bit整型的lock_word就维护了绝大部分的锁状态，其取值含义如下图所示。

![](.img/5bce4256b31e_2020-04-26-wanghu-lock_word.jpg)

## 4 加解锁的实现

### 4.1 锁的重入

InnoDB的每个读写锁都可以设置是否开启可重入模式（Recursive）。当使用可重入模式时，同一个线程可以多次获得锁，只需保证加锁总次数与解锁总次数相等即可。更强大的是，可重入模式下，同一个线程可以同时获得一个读写锁的X锁和SX锁，也可以同时获得一个读写锁的SX锁和S锁，但是不能同时获得X锁和S锁。

### 4.2 加锁逻辑的实现

InnoDB读写锁实现的核心思想是避免使用pthread rwlock，而尽量使用原子操作+自旋的模式来实现加解锁，这样可以在低冲突的场景下，以尽量小的开销实现加解锁。遇到实在是冲突高的读写锁，再使用InnoDB条件变量实现等待。

下面以X锁的加锁逻辑来举例说明InnoDB读写锁加锁的实现。SX锁和S锁的加锁逻辑比较类似，对应代码可以参照阅读。X锁加锁的最终入口函数是rw_lock_x_lock_func，位于sync/sync0rw.cc中。函数签名如下：

`void rw_lock_x_lock_func(rw_lock_t *lock, ulint pass, const char *file_name, ulint line);
`

其中pass参数的含义是如果当前锁上已经有X锁或者是SX锁，是否进入可重入模式。加锁逻辑可以用下面的流程图总结。

![](.img/9ae513ba9088_2020-04-26-wanghu-x-lock.jpg)

### 4.3 解锁逻辑的实现

下面以X锁的解锁逻辑来举例说明InnoDB读写锁解锁的实现。SX锁和S锁的解锁逻辑比较类似，对应代码可以参照阅读。X锁解锁的最终入口函数是rw_lock_x_unlock_func，位于include/sync0rw.ic中。解锁逻辑可以用下面的流程图总结。

![](.img/48a6503a47ca_2020-04-26-wanghu-x-unlock.jpg)

## 5 X锁所有权的转移

InnoDB读写锁上的X锁所有权是可以在不同线程间转移的，主要用于支持Change Buffer的场景。Change Buffer是一棵全局共享的B+树，存储在系统表空间中。在读取二级索引Page的时候Change Buffer需要与二级索引Page进行合并，这时如果所有IO线程都在读取二级索引Page，将没有IO线程读取Change Buffer Page，因此Change Buffer Page的读取被放到单独的IO线程。而读取二级索引Page的时候，已经对Page加上了X锁，当在异步IO线程需要把Change Buffer合并到二级索引的Page的时候，必须在不解锁的情况下让异步线程获得Page的X锁，这就是X锁所有权转移需要实现的功能。

实现函数是rw_lock_x_lock_move_ownership，实现的逻辑也非常简单，使用CAS原子操作把读写锁的write_thread字段设置为当前线程。

`os_thread_id_t curr_thread = os_thread_get_curr_id();
...
local_thread = lock->writer_thread; 
os_compare_and_swap_thread_id(&lock->writer_thread, local_thread, curr_thread);
...
`

## 6 总结

本文分析整理了InnoDB读写锁的实现，InnoDB读写锁在兼顾性能和多核可扩展性的同时，提供了强大的功能，包括在典型的读锁和写锁的基础上增加了SX锁来优化锁冲突，可重入的锁语义以及X锁所有权的转移等等，是非常有参考意义的高性能并发同步基础代码。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)