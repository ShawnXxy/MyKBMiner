# MySQL · 专家投稿 · MySQL数据库SYS CPU高的可能性分析

**Date:** 2015/05
**Source:** http://mysql.taobao.org/monthly/2015/05/02/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 05
 ](/monthly/2015/05)

 * 当期文章

 MySQL · 引擎特性 · InnoDB redo log漫游
* MySQL · 专家投稿 · MySQL数据库SYS CPU高的可能性分析
* MySQL · 捉虫动态 · 5.6 与 5.5 InnoDB 不兼容导致 crash
* MySQL · 答疑解惑 · InnoDB 预读 VS Oracle 多块读
* PgSQL · 社区动态 · 9.5 新功能BRIN索引
* MySQL · 捉虫动态 · MySQL DDL BUG
* MySQL · 答疑解惑 · set names 都做了什么
* MySQL · 捉虫动态 · 临时表操作导致主备不一致
* TokuDB · 引擎特性 · zstd压缩算法
* MySQL · 答疑解惑 · binlog 位点刷新策略

 ## MySQL · 专家投稿 · MySQL数据库SYS CPU高的可能性分析 
 Author: 高亚芳 

 ## 问题背景

我们在管理繁忙的 MySQL 数据库时，可能都有碰到 SYS CPU 高的经历：系统突然 SYS CPU 高起来，甚至比 USER CPU 高很多，这时系统 QPS、TPS 急剧下降。

SYS CPU高是什么造成的呢？主要有2种可能：

1. context switch 不高，但在内核态 spin，导致 SYS CPU 高
2. context switch 高，每秒超过 200K，有时超过1M，过多 context switch 导致 SYS CPU 高

下面我们对这两种情况逐一分析。

## context switch 不高，但在内核态 spin

MySQL 在内核态 spin，说明需要系统资源，当这个资源紧张不足时，就会在内核态 spin。

有些资源，用户进程（或线程）通过执行系统调或因中断进入内核，一般来说申请这些资源的执行时间很短，当出现资源争用时，如果采用 sleep 再唤醒机制，代价较大，因此多采用在内核态spin的策略。

例如申请内存或发生缺页中断，没有 free 内存可用时，进程或线程就可能在内核态先执行内存回收再执行内存分配，但系统内存是共享资源，分配回收时需要锁保护，当多个进程（或线程）同时回收分配内存时。就会在内核态 spin。

当 free 内存不足时，可能出现这种情况。典型症状：

1. MySQL running 高，但系统 qps、tps 下降
2. 系统 free 内存不足；或系统 free 内存充足时，但启用了 numa 内存分配策略，有的节点 free 内存很少
3. 系统 context switch 不高
4. MySQL InnoDB 的 mutex、RWlock 查不到等待信息
5. sar -B 显示有 pgscand 产生

### 分析

当系统内存不足时，MySQL 突然有大量访问，紧急需要大量内存，kswapd 在短时间内回收不了足够多的 free 内存，或 kswapd 还没有触发执行，这时 MySQL 用户线程就会在内核态执行内存回收操作，从而出现以上症状。

sar -B 输出中，pgscank 是表示内核线程 kswapd 回收内存，k意思是 kernel；pgscand是表示用户进程或线程直接回收内存，d意思是direct。

解决办法：保证系统有充足 free 内存可用，NUMA 环境要求每个节点都有足够free内存可用。

由于 Linux 系统会尽量使用 free 内存，一个运行很久的 Linux 系统，free内存通常很少，存在大量 filecache 内存，但 Linux 没有直接提供控制 filecache 占用多少的参数，那怎么能够保留足够可用的 free 内存，以应对突然内存需求呢？

对此，Linux 2.3.32+ 内核中增加一个新的参数`vm.extra_free_kbytes`，就是控制free内存的。

关于系统free内存，有2个重要参数：`vm.min_free_kbytes` 和 `vm.extra_free_kbytes`(2.6.32+)

`vm.min_free_kbytes`：系统保留给内核用的内存。

这个值决定 `/proc/zoneinfo` 中 zone 的min值。当系统 free 内存小于这个值时，kswapd 会回收内存，直到free内存达到`/proc/zoneinfo`中 high 值才停止回收；

当用户进程或线程分配内存或发生缺页中断时，free 内存少于 `vm.min_free_kbytes`，会在用户线程上下文中直接进行回收内存（pgscand）和分配内存。

`vm.extra_free_kbytes`：系统保留给应用的free内存。

这个值决定了`/proc/zoneinfo`中Normal zone的low值。当系统free内存小于`vm.min_free_kbytes + vm.extra_free_kbytes` 时，kswapd会开始回收内存，直到free内存达到 `/proc/zoneinfo` 中high值才停止回收。

这个额外的`vm.extra_free_kbytes`就是给应用突发内存需求时使用的，避免急需内存时发生pgscand或kswapd回收内存不及时。

`vm.extra_free_kbytes` 分配多大合适呢？一般能应对流量高峰时1-2秒内存需求就可以了。free内存减少后，kswapd进程会在后台回收内存的，一般512M-2G可以满足要求。

## context switch 高

有很多种情况都会导致 context switch。MySQL 中的 mutex 和 RWlock 在获取不成功后，短暂spin，还不成功，就会发生 context switch，sleep，等待唤醒。

在 MySQL中，mutex 和 RWlock导致的 context switch，一般在`show global status`,`show engine innodb mutex`,`show engine innodb status`,`performance_schema`等中会体现出来，针对不同的mutex和RWlock等待，可以采取不同的优化措施。

除了MySQL的mutex和RWlock，还发现一种情况，是MySQL外的mutex竞争导致context switch高。

典型症状：

1. MySQL running 高，但系统 qps、tps 低
2. 系统context switch很高，每秒超过200K
3. 在 MySQL 内存查不到mutex和RWlock竞争信息
4. SYS CPU 高，USER CPU 低
5. 并发执行的SQL中出现timestamp字段，MySQL的time_zone设置为system

### 分析

对于使用 timestamp 的场景，MySQL 在访问 timestamp 字段时会做时区转换，当 time_zone 设置为 system 时，MySQL 访问每一行的 timestamp 字段时，都会通过 libc 的时区函数，获取 Linux 设置的时区，在这个函数中会持有mutex，当大量并发SQL需要访问 timestamp 字段时，会出现 mutex 竞争。

MySQL 访问每一行都会做这个时区转换，转换完后释放mutex，所有等待这个 mutex 的线程全部唤醒，结果又会只有一个线程会成功持有 mutex，其余又会再次sleep，这样就会导致 context switch 非常高但 qps 很低，系统吞吐量急剧下降。

解决办法：设置time_zone=’+8:00’，这样就不会访问 Linux 系统时区，直接转换，避免了mutex问题。

另外，对于spin消耗，MySQL配置变量中的`innodb_spin_wait_delay` 和 `innodb_sync_spin_loops` 可以用于微调。

 作者介绍
高亚芳 北京理工大学计算机系毕业，IT行业老兵，目前负责数据存储基础架构工作，开源爱好者。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)