# MySQL · 社区动态 · MySQL内存分配支持NUMA

**Date:** 2015/07
**Source:** http://mysql.taobao.org/monthly/2015/07/06/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 07
 ](/monthly/2015/07)

 * 当期文章

 MySQL · 引擎特性 · Innodb change buffer介绍
* MySQL · TokuDB · TokuDB Checkpoint机制
* PgSQL · 特性分析 · 时间线解析
* PgSQL · 功能分析 · PostGIS 在 O2O应用中的优势
* MySQL · 引擎特性 · InnoDB index lock前世今生
* MySQL · 社区动态 · MySQL内存分配支持NUMA
* MySQL · 答疑解惑 · 外键删除bug分析
* MySQL · 引擎特性 · MySQL logical read-ahead
* MySQL · 功能介绍 · binlog拉取速度的控制
* MySQL · 答疑解惑 · 浮点型的显示问题

 ## MySQL · 社区动态 · MySQL内存分配支持NUMA 
 Author: Plinux 

 **NUMA** 问题曾经一直是困扰DBA的一个大问题，早在 2010 年, 就有人给MySQL报了Bug#[57241](https://bugs.mysql.com/bug.php?id=57241), 指出了MySQL在x86系统下存在严重的 “swap insanity” 问题。在NUMA架构越来越普遍的今天，这个问题越来越严重。

## MySQL的 *swap insanity* 问题

有同学专门翻译了Jeremy Cole关于 “swap insanity” 问题的[文章](http://sohulinux.blog.sohu.com/181968823.html)，原文看[这里](http://blog.jcole.us/2010/09/28/mysql-swap-insanity-and-the-numa-architecture/)，

如果你没空看的话，这里简单描述一下，就是当你把主机大部分内存分配给InnoDB时，你会发现明明操作系统还有很多内存，但是却有很多内存被交换到了SWAP分区。

从[这里](http://ozlabs.org/~anton/junkcode/latency2001.c)可以下载到一个测试的C代码，如果你有NUMA架构的服务器，可以测试下不同分配方式的性能差异：

`sudo -s
echo 2048 > /proc/sys/vm/nr_hugepages
echo 1000000000000 > /proc/sys/kernel/shmmax

# Node local allocation
for i in `seq 0 4 127`
do
./latency2001 -a $i -c $i -l 128M
done

# Allocate on memory on CPU 0
for i in `seq 0 4 127`
do
./latency2001 -a 0 -c $i -l 128M
done
`

有两个方式可以解决这个问题：

1. 在Linux Kernel启动参数中加上numa=off（这样也会影响到其他进程使用NUMA）；
2. 在mysqld_safe脚本中加上“numactl –interleave all”来启动mysqld。

当然如果跑多实例，我也用过直接绑定mysqld进程到某个numa节点的方式，不过这要求每个实例分配的内存不超过每个NUMA节点管理的内存。脚本可以看[这里](http://www.penglixun.com/tech/database/mysql_multi_using_numactl.html)。

5年过去了，官方依然没有解决这个Bug。但好消息是，官方终于着手解决这个问题了，Stewart Smith 同学提交的Bug#[72811](https://bugs.mysql.com/bug.php?id=72811)，其Patch即将出现在MySQL 5.6.27, 5.7.9 版本之中。

## 代码层面解决NUMA问题

如果在代码层面彻底解决NUMA问题，那么我们需要解决两个问题：

1. 全局内存应该采用interleave的分配方式分散在不同的numa node上；
2. 线程内存应该采用local的分配方式分配在线程运行的numa node上。

Linux 提供了 `set_mempolicy()` 函数可以用来设置进程的内存分配策略，其中默认的MPOL_DEFAULT策略就是在当前运行的节点上分配内存，而MPOL_INTERLEAVE策略则是跨所有节点来分配内存。这个函数的说明可以看[这里](http://man7.org/linux/man-pages/man2/set_mempolicy.2.html)。

因此对于MySQL Server和InnoDB引擎都需要做修改：

1. 在`mysqld_main()`入口设置 `set_mempolicy(MPOL_INTERLEAVE, NULL, 0)` 启用全局分配方式；
2. 在MySQL启动完成之后设置`set_mempolicy(MPOL_DEFAULT, NULL, 0)` 启用本地分配方式；
3. 在InnoDB入口时设置 `set_mempolicy(MPOL_INTERLEAVE, NULL, 0)` 启用全局分配方式；
4. 在Buffer Pool分配完成时设置 `set_mempolicy(MPOL_DEFAULT, NULL, 0)` 启用本地分配方式。

MySQL 5.6.27, 5.7.9 发布之后，将会增加一个 `innodb_numa_interleave` 参数来控制这个策略。`innodb_numa_interleave` 如果打开，那么将会按上面的策略来设置内存分配方式，如果关闭或者主机不支持NUMA，那么还是按原来的方式分配。

我们一起期待新版本的发布吧，妈妈再也不用担心我的NUMA了！

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)