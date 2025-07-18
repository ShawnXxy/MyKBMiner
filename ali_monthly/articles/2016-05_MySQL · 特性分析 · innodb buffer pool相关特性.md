# MySQL · 特性分析 · innodb buffer pool相关特性

**Date:** 2016/05
**Source:** http://mysql.taobao.org/monthly/2016/05/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 05
 ](/monthly/2016/05)

 * 当期文章

 MySQL · 引擎特性 · 基于InnoDB的物理复制实现
* MySQL · 特性分析 · MySQL 5.7新特性系列一
* PostgreSQL · 特性分析 · 逻辑结构和权限体系
* MySQL · 特性分析 · innodb buffer pool相关特性
* PG&GP · 特性分析 · 外部数据导入接口实现分析
* SQLServer · 最佳实践 · 透明数据加密在SQLServer的应用
* MySQL · TokuDB · 日志子系统和崩溃恢复过程
* MongoDB · 特性分析 · Sharded cluster架构原理
* PostgreSQL · 特性分析 · 统计信息计算方法
* MySQL · 捉虫动态 · left-join多表导致crash

 ## MySQL · 特性分析 · innodb buffer pool相关特性 
 Author: 济天 

 ## 背景
innodb buffer pool做为innodb最重要的缓存，其缓存命中率的高低会直接影响数据库的性能。因此在数据库发生变更，比如重启、主备切换实例迁移等等，innodb buffer pool 需要一段时间预热，期间数据库的性能会受到明显影响。
另外mysql 5.7以前innodb buffer pool缓存大小修改不是动态的，重启才能生效。因此innodb buffer pool的预热和innodb buffer pool大小的动态修改，对性能要求较高的应用来说是不错的特性，下面我来看看这两个特性的具体实现。

## buffer pool 预热
MySQL 5.6以后支持buffer pool预热功能。引入了以下参数, 参数具体含义参见[官方文档](http://dev.mysql.com/doc/refman/5.6/en/mysqld-option-tables.html)。

`innodb_buffer_pool_load_now
innodb_buffer_pool_dump_now
innodb_buffer_pool_load_at_startup
innodb_buffer_pool_dump_at_startup
innodb_buffer_pool_filename
`
buffer pool预热分为dump过程和load过程，均由后台线程buf_dump_thread完成。
比如用户发起set命令

`set global innodb_buffer_pool_dump_now=on;
set global innodb_buffer_pool_load_now=on;
`
set 命令会立刻返回，具体操作由buf_dump_thread来实现。

* dump 过程

 锁buf_pool
遍历LRU链表，将(space, pageno) 先收集到数组
释放锁
再将数据写入innodb_buffer_pool_filename定有的文件中
* load过程

 从文件读入数组
按（space,pageno)排序数据
依次同步读取页到buffer pool中

dump过程一般比较快，而load过程相对要慢些。

通过`Innodb_buffer_pool_dump_status`、`Innodb_buffer_pool_load_status`可查看dump/load的状态

另外5.7引入了performance_schema.events_stages_current来显示load进度，每load 32M会更新一条进度信息

`select * from performance_schema.events_stages_current;
THREAD_ID 19
EVENT_ID 1367
END_EVENT_ID NULL
EVENT_NAME stage/innodb/buffer pool load
SOURCE buf0dump.cc:619
TIMER_START 33393877311000
TIMER_END 33398961258000
TIMER_WAIT 5083947000
WORK_COMPLETED 0
WORK_ESTIMATED 1440
NESTING_EVENT_ID NULL
NESTING_EVENT_TYPE NULL
`
WORK_ESTIMATED表示总page数
WORK_COMPLETED表示当前已load page数

dump文件的数据格式如下

`#cat ib_buffer_pool |more
0,7
0,1
0,3
0,2
0,4
0,11
0,5
0,6
`

dump文件比较简单，我们可以编辑此文件来预加载指定page,比较灵活。

## buffer pool 动态调整大小
5.7 开始支持buffer pool 动态调整大小，每个`buffer_pool_instance`都由同样个数的chunk组成(chunks数组), 每个chunk内存大小为`innodb_buffer_pool_chunk_size`(实际会偏大5%，用于存放chuck中的block信息)。buffer pool以`innodb_buffer_pool_chunk_size`为单位进行动态增大和缩小。调整前后`innodb_buffer_pool_size`应一直保持是`innodb_buffer_pool_chunk_size`*`innodb_buffer_pool_instances`的倍数。

同样的buffer pool动态调整大小由后台线程`buf_resize_thread`,set命令会立即返回。通过`InnoDB_buffer_pool_resize_status`可以查看调整的运行状态。

* resize流程

 如果开启了AHI，需禁用AHI
* 如果是收缩内存
 
 计算需收缩的chunk数， 从chunks开始尾部删除指定个数的chunk.
* 锁buf_pool
* 从free_list中摘除待删chunk的page放入待删链表buf_pool->withdraw
* 如果待删chunk的page为脏页，则刷脏
* 重新加载LRU中要删除的页，从LRU中摘除，重新从free列表获取page老的page放入待删链表buf_pool->withdraw
* 释放buffer pool锁
* 如果需收缩的chunk pages没有收集全，重复2-6
* 开始resize
 
 锁住所有instance的buffer_pool，page_hash
* 收缩pool：以chunk为单位释放要收缩的内存
* 清空withdraw列表buf_pool->withdraw
* 增大pool:分配新的chunk
* 重新分配buf_pool->chunks
* 如果改变/缩小超过2倍，会重置page hash，改变桶大小
* 释放buffer_pool,page_hash锁
* 如果改变/缩小超过2倍,会重启和buffer pool大小相关的内存结构，如锁系统(lock_sys_resize)，AHI(btr_search_sys_resize), 数据字段(dict_resize)等
* 如果禁用了AHI，此时开启

由上可以看出，扩大内存比缩小内存相对容易些。缩小内存时，如果遇到有事务一直未提交且占用了待收缩的page时，导致收缩一直重试，error log会打印这种重试信息，
包含可能引用此问题的事务信息。为了避免频繁重试，每次重试的时间间隔会指数增长。

以上步骤中resize阶段buffer pool会不可用，此阶段会锁所有buffer pool, 但此阶段都是内存操作，时间比较短。收缩内存阶段耗时可能会很长，也有一定影响，但是每次都是以instance为单位进行锁定的。
总的来说，buffer pool 动态调整大小对应用的影响并不大。

* 重新加载LRU中要删除的页的影响

 search 过程中btr游标保存的page可能重新加载过，自适应哈希保存的root page也可能重新加载过, 都需要重新读取。

## 总结
buffer pool 预热 和buffer pool 动态调整大小，这两功能相辅相承的。buffer pool 动态调整大小只适用于实例在主机本地升级的情况，如果用户修改buffer pool大小，同时涉及跨机迁移，那么buffer pool 预热功能就排上用场了。
另外buffer pool 动态调整尽量在业务低锋时进行。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)