# MySQL · 捉虫动态 · Error in munmap() "Cannot allocate memory"

**Date:** 2018/01
**Source:** http://mysql.taobao.org/monthly/2018/01/05/
**Images:** 2 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 01
 ](/monthly/2018/01)

 * 当期文章

 MySQL · 引擎特性 · Group Replication内核解析之二
* MySQL · 引擎特性 · MySQL内核对读写分离的支持
* PgSQL · 内核解析 · 同步流复制实现分析
* MySQL · 捉虫动态 · UK 包含 NULL 值备库延迟分析
* MySQL · 捉虫动态 · Error in munmap() "Cannot allocate memory"
* MSSQL · 最佳实践 · 数据库备份链
* MySQL · 捉虫动态 · 字符集相关变量介绍及binlog中字符集相关缺陷分析
* PgSQL · 应用案例 · 传统分库分表(sharding)的缺陷与破解之法
* MySQL · MyRocks · MyRocks参数介绍
* PgSQL · 应用案例 · 惊天性能！单RDS PostgreSQL实例支撑 2000亿

 ## MySQL · 捉虫动态 · Error in munmap() "Cannot allocate memory" 
 Author: xijia 

 ## 前言
最近线上遇到一个问题，一个MySQL实例报错 Error in munmap(): Cannot allocate memory 造成进程异常退出

## 背景介绍

MySQL 使用 jemalloc 进行内存分配，报错的原因是 MySQL 进程的 VMA 数量大于操作系统上限

这里先介绍几个前序概念

#### 虚拟内存区域 VMA
Linux进程通过vma进行管理，每个进程都有一个结构体中维护一个vma链表，其中每个vma节点对应一段连续的进程内存。这里的连续是指在进程空间中连续，物理空间中不一定连续。如果进程申请一段内存，则内核会给进程增加vma节点

#### /proc/pid/maps
/proc/pid/maps 记录了进程的虚拟内存使用情况

举个例子，进程b.out的maps如下，每一行代表一个VMA（删除了一部分重复的行

`00400000-00401000 r-xp 00000000 fd:01 1574192 /u01/b.out
00602000-00701000 rw-p 00000000 00:00 0 [heap]
7ffff71f8000-7ffff73b0000 r-xp 00000000 fd:01 1049989 /usr/lib64/libc-2.17.so
7ffff75b6000-7ffff75bb000 rw-p 00000000 00:00 0
7ffff75bb000-7ffff75d0000 r-xp 00000000 fd:01 1052643 /usr/lib64/libgcc_s-4.8.5-20150702.so.1
7ffff77d1000-7ffff78d2000 r-xp 00000000 fd:01 1049997 /usr/lib64/libm-2.17.so
fff7ad3000 rw-p 00101000 fd:01 1049997 /usr/lib64/libm-2.17.so
7ffff7ad3000-7ffff7bbc000 r-xp 00000000 fd:01 1050280 /usr/lib64/libstdc++.so.6.0.19
dc6000 rw-p 000f1000 fd:01 1050280 /usr/lib64/libstdc++.so.6.0.19
7ffff7dc6000-7ffff7ddb000 rw-p 00000000 00:00 0
7ffff7ddb000-7ffff7dfc000 r-xp 00000000 fd:01 1049982 /usr/lib64/ld-2.17.so
7ffff7fce000-7ffff7ff4000 rw-p 00000000 00:00 0
7ffff7ff9000-7ffff7ffa000 rw-p 00000000 00:00 0
7ffff7ffa000-7ffff7ffc000 r-xp 00000000 00:00 0 [vdso]
7ffff7ffc000-7ffff7ffd000 r--p 00021000 fd:01 1049982 /usr/lib64/ld-2.17.so
7ffff7ffd000-7ffff7ffe000 rw-p 00022000 fd:01 1049982 /usr/lib64/ld-2.17.so
7ffff7ffe000-7ffff7fff000 rw-p 00000000 00:00 0
7ffffffde000-7ffffffff000 rw-p 00000000 00:00 0 [stack]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0 [vsyscall]
`

* 第一列，如00400000-00401000
 
 虚拟空间的起始和终止地址

 第二列，如rw-p
 * VMA的权限，前三位rwx分别代表可读、可写、可执行，“-”代表没有该权限；第四位p/s代表私有/共享段

 第三列，如00021000
 * 虚拟内存起始地址在文件中的偏移量，匿名映射为0

 第四列，如fd:01
 * 映射文件所属设备好，匿名映射为0

 第五列，如1049982
 * 映射文件所属节点号，匿名映射为0

 第六列，如/u01/b.out /usr/lib64/libstdc++.so.6.0.19 [stack]
 * 映射文件名，[heap]代表堆，[stack]代表栈

#### vm.max_map_count
max_map_count 是一个进程内存能拥有的VMA最大数量

当进程达到了VMA上限但又只能释放少量的内存给其他的内核进程使用时，操作系统会抛出内存不足的错误

Error in munmap(): Cannot allocate memory 就是触发了这个错误

## 问题复现
操作系统 vm.max_map_count=65530

执行以下代码，可以复现munmap无法分配内存的错误

`#include <sys/mman.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>

#define VM_MAX_MAP_COUNT (65530)
#define VM_SIZE (4096)
#define VM_CNT (VM_MAX_MAP_COUNT * 2)

static void* vma[VM_CNT];

int main(void)
{
 int i;
 for (i = 0; i < VM_CNT; i++)
 {
 vma[i] = mmap(0, VM_SIZE, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0 );
 }

 for (i = 0; i < VM_CNT; i++)
 {
 if (munmap(vma[i], VM_SIZE) != 0)
 printf("mumap() ERROR");
 }
}
`

先用 mmap 分配 65530 * 2 个虚拟内存空间

因为是连续分配的，操作系统会合并成一个VMA，下图可以看出，/proc/pid/maps 文件中多了一个VMA

另有两个已存在的VMA被修改

![image.png](.img/0206baa8748a_201801xj-01.png)

`多出来的VMA 7fffd73fc000-7ffff71f8000 共有130566个 VM_SIZE

7ffff7fef000-7ffff7ff4000(0x5000) -> 7ffff7dfc000-7ffff7ff5000(0x1f9000) 多出 500 个 VM_SIZE

7ffff7ff9000-7ffff7ffa000 -> 7ffff7ff5000-7ffff7ffa000 起始地址前移 0x4000, 多出 4 个 VM_SIZE 
`

130566 + 500 + 4 = 131070 = 65536 * 2 正好是程序中申请的内存大小

下面再用munmap每隔一个VM_SIZE释放一个VM_SIZE，将原本连续的虚拟内存空间变得不连续，这样就会形成65536个VMA，再加上本来存在的若干个VMA，超过了操作系统设定的VMA上限 65530

实际执行时，VMA数量到达65530时，再执行munmap就会报错

## jemalloc 和 glibc malloc

MySQL 使用 jemalloc 分配内存，jemalloc 默认采用mmap()/munmap()分配和释放内存，已经验证 jemalloc 在 max_map_count较小时会触发无法分配内存的异常

使用同样场景验证 glibc malloc 是否存在同样问题，glibc malloc 分配 128k 以上内存是默认使用mmap，128k以下时默认使用sbrk，所以这里把VM_SIZE 改为 129k

`#include <sys/mman.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>

#define VM_MAX_MAP_COUNT (65530)
#define VM_SIZE (129 * 1024)
#define VM_CNT (VM_MAX_MAP_COUNT * 2)

static void* vma[VM_CNT];

int main(void)
{
 int i;
 for (i = 0; i < VM_CNT; i++)
 {
 vma[i] = malloc(VM_SIZE);
 }

 for (i = 0; i < VM_CNT; i++)
 {
 free(vma[i])
 }
}
`
![image.png](.img/0206baa8748a_201801xj-01.png)

上图可以看出，除了申请新的VMA，heap空间也增长了

` 新增VMA 7ffde73e7000-7ffff71f8000 共 67044个VM_SIZE
 
 7ffff7fef000-7ffff7ff4000 -> 7ffff7e00000-7ffff7ff4000 共15个VM_SIZE
 
 [heap] 00602000-00701000 -> 00602000-20467e000 共 65531 个VM_SIZE
`

多分配出1530个VM_SIZE 这里应该是glibc 内部机制控制的，不做深入研究

重点在于申请了65531个VM_SIZE的[heap]空间，这部分空间由sbrk()申请

#### mmap() 和 sbrk()
malloc 申请小于 128k 内存时，使用 sbrk() 分配，大于 128k 默认使用 mmap()

同时，mmap() 分配内存最多65536次，超过后使用 sbrk() 分配

mmap()在虚拟地址空间中找一块空闲地址分配，分配的内存可以被随意释放

sbrk() 将指向数据段的最高地址的_edata指针上推，释放时将_edata下推；显然，sbrk()无法随意释放内存，释放一块内存时，必须把比它地址高的内存全部释放

如图，初始时 _edata 在 heap 最下方，分配A时 _edata 推到 A 下方，分配 B/ C时 _edata 推到 B/C 下方

释放 C 时，再将 _edata 上推到 B 下方，但是在释放 B 之前释放 A 只能讲 A 的内存块标记为未使用，供下次分配，不能移动 _edata

 .text

 .data

 heap

 A

 B

 C

显然，释放sbrk()申请的内存时，不会增加VMA数量，所以 glibc malloc 不会使 VMA 数量超过 max_map_count，不会触发上述问题

## 总结

问题的原因很明显，VMA 数量超过 max_map_count，调大 max_map_count 可以简单粗暴的解决这个问题

jemalloc 在大多数情况下比 glibc malloc 性能更好，但也不能适用全部场景。业务场景合适时适用 glibc malloc 也可以规避这个问题

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)