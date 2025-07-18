# MySQL · 参数故事 · innodb_additional_mem_pool_size

**Date:** 2016/04
**Source:** http://mysql.taobao.org/monthly/2016/04/01/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 04
 ](/monthly/2016/04)

 * 当期文章

 MySQL · 参数故事 · innodb_additional_mem_pool_size
* GPDB · 特性分析 · Segment事务一致性与异常处理
* GPDB · 特性分析 · Segment 修复指南
* MySQL · 捉虫动态 · 并行复制外键约束问题二
* PgSQL · 性能优化 · 如何潇洒的处理每天上百TB的数据增量
* Memcached · 最佳实践 · 热点 Key 问题解决方案
* MongoDB · 最佳实践 · 短连接Auth性能优化
* MySQL · 最佳实践 · RDS 只读实例延迟分析
* MySQL · TokuDB · TokuDB索引结构--Fractal Tree
* MySQL · TokuDB · Savepoint漫谈

 ## MySQL · 参数故事 · innodb_additional_mem_pool_size 
 Author: 西加 

 ## 参数简介

innodb_additional_mem_pool_size 是 InnoDB 用来保存数据字典信息和其他内部数据结构的内存池的大小，单位是 byte，参数默认值为8M。数据库中的表数量越多，参数值应该越大，如果 InnoDB 用完了内存池中的内存，就会从操作系统中分配内存，同时在 error log 中打入报警信息。

innodb_use_sys_malloc 配置为 ON 时，innodb_additional_mem_pool_size 失效（直接从操作系统分配内存）。

innodb_additional_mem_pool_size 和 innodb_use_sys_malloc 在 MySQL 5.7.4 中移除。

## 参数合理值预估

`./storage/innobase/handler/ha_innodb.cc:
srv_mem_pool_size = (ulint) innobase_additional_mem_pool_size;

./storage/innobase/srv/srv0srv.cc: mem_init(srv_mem_pool_size);

storage/innobase/mem/mem0dbg.cc: mem_comm_pool = mem_pool_create(size);
`

从源码中可以看出，innodb_additional_mem_pool_size 的参数值用于指定内存池 mem_comm_pool 的大小；

`storage/innobase/mem/mem0mem.cc:
 block = static_cast<mem_block_t*>(
 mem_area_alloc(&len, mem_comm_pool));
`

函数 `mem_area_alloc` 从 mem_comm_pool 内存池中分配内存；

`storage/innobase/mem/mem0pool.cc:

/* If we are using os allocator just make a simple call
to malloc */
 if (UNIV_LIKELY(srv_use_sys_malloc)) {
 return(malloc(*psize));
}

........

area = UT_LIST_GET_FIRST(pool->free_list[n]);

if (area == NULL) {
 ret = mem_pool_fill_free_list(n, pool);

 if (ret == FALSE) {
 /* Out of memory in memory pool: we try to allocate
 from the operating system with the regular malloc: */

 mem_n_threads_inside--;
 mutex_exit(&(pool->mutex));

 return(ut_malloc(size));
 }

 area = UT_LIST_GET_FIRST(pool->free_list[n]);
}
`

如果 innodb_use_sys_malloc (上述代码中的srv_use_sys_malloc) 设置为 ON，或者内存池中没有足够的内存可供分配，则直接从操作系统中分配内存。

`mem_area_alloc` 调用栈如下(use database 触发断点)

`#0 mem_area_alloc
#1 0x000000000118048d in mem_heap_create_block_func
#2 0x000000000149a390 in mem_heap_create_func
#3 0x00000000014aa6d5 in dict_load_table
#4 0x0000000001481082 in dict_table_open_on_name
#5 0x000000000109d769 in ha_innobase::open
#6 0x00000000006d5245 in handler::ha_open
#7 0x0000000000b830ae in open_table_from_share
#8 0x000000000091deee in open_table
#9 0x0000000000922eea in open_and_process_table
#10 0x000000000092492f in open_tables
#11 0x0000000000926c21 in open_normal_and_derived_tables
#12 0x0000000000a83834 in mysqld_list_fields
#13 0x00000000009f28e1 in dispatch_command
#14 0x00000000009eeb51 in do_command
#15 0x0000000000982cb6 in do_handle_one_connection
#16 0x000000000098238b in handle_one_connection
#17 0x0000000001877f91 in pfs_spawn_thread
#18 0x0000003d8c007851 in start_thread ()
#19 0x0000003d8bce767d in clone ()
`

函数 `dict_load_table` 中会为每张表分配32k的空间 ( `mem_heap_create(32000)` 实际分配32744字节空间 )，数据字典中每张表所占空间的上限是32k，具体占用空间根据列数和索引数量分配，分配完成后回收32k中未使用的空间

`storage/innobase/dict/dict0load.cc: heap = mem_heap_create(32000);
`

show engine innodb status BUFFER POOL AND MEMORY Dictionary cache

实际使用的数据字典缓存，不会超过每张表32k，实测过程中，每张表不包括索引占4K，每个索引占2k，列数对空间占用影响不大。

测试用表如下，未建索引时，1000张表占用空间4M，增加列占用空间增长不明显，每增加一个索引，占用空间增加2M，可以估测每张表占用空间4k(不含索引)，每个索引占用空间2k。

`Create Table: CREATE TABLE `1000` (
 `id` int(11) DEFAULT NULL,
 `a` varchar(255) DEFAULT NULL,
 `b` varchar(255) DEFAULT NULL,
 `c` varchar(255) DEFAULT NULL,
 `d` varchar(255) DEFAULT NULL,
 KEY `a` (`a`),
 KEY `b` (`b`),
 KEY `id` (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
`

## 引入和移除该参数的原因

早期操作系统的内存分配器性能和可伸缩性较差，并且当时没有适合多核心CPU的内存分配器。所以，InnoDB 实现了一套自己的内存分配系统，做为内存系统的参数之一，引入了`innodb_additional_mem_pool_size`。

随着多核心CPU的广泛应用和操作系统的成熟，操作系统能够提供性能更高、可伸缩性更好的内存分配器，包括 Hoard、libumem、mtmalloc、ptmalloc、tbbmalloc 和 TCMalloc 等。InnoDB 实现的内存分配器相比操作系统的内存分配器并没有明显优势，所以在之后的版本，会移除 innodb_additional_mem_pool_size 和 innodb_use_sys_malloc 两个参数，统一使用操作系统的内存分配器。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)