# MySQL · 功能分析 · MySQL表定义缓存

**Date:** 2015/08
**Source:** http://mysql.taobao.org/monthly/2015/08/10/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 08
 ](/monthly/2015/08)

 * 当期文章

 MySQL · 社区动态 · InnoDB Page Compression
* PgSQL · 答疑解惑 · RDS中的PostgreSQL备库延迟原因分析
* MySQL · 社区动态 · MySQL5.6.26 Release Note解读
* PgSQL · 捉虫动态 · 执行大SQL语句提示无效的内存申请大小
* MySQL · 社区动态 · MariaDB InnoDB表空间碎片整理
* PgSQL · 答疑解惑 · 归档进程cp命令的core文件追查
* MySQL · 答疑解惑 · open file limits
* MySQL · TokuDB · 疯狂的 filenum++
* MySQL · 功能分析 · 5.6 并行复制实现分析
* MySQL · 功能分析 · MySQL表定义缓存

 ## MySQL · 功能分析 · MySQL表定义缓存 
 Author: 济天 

 ## 表定义
MySQL的表包含表名，表空间、索引、列、约束等信息，这些表的元数据我们暂且称为表定义信息。
对于InnoDB来说，MySQL在server层和engine层都有表定义信息。server层的表定义记录在frm文件中，而InnoDB层的表定义信息存储在InnoDB系统表中。例如:

` InnoDB_SYS_DATAFILES 
 InnoDB_SYS_TABLESTATS 
 InnoDB_SYS_INDEXES 
 InnoDB_SYS_FIELDS 
 InnoDB_SYS_TABLESPACES 
 InnoDB_SYS_FOREIGN_COLS
 InnoDB_SYS_FOREIGN 
 InnoDB_SYS_TABLES 
 InnoDB_SYS_COLUMNS 
`

注：以上都是memory表，它们内容是从实际系统表中获取的。实际上InnoDB系统表engine也是InnoDB类型的，数据也是以B树组织的。

在数据库每次执行sql都会访问表定义信息，如果每次都从frm文件或系统表中获取，效率会较低。因此MySQL在server层和InnoDB层都有表定义的缓存。以MySQL 5.6为例，参数table_definition_cache控制了表定义缓存中表的个数，server层和InnoDB层的表定义缓存共用此参数。

## server层表定义缓存

server层表定义为TABLE_SHARE对象，TABLE_SHARE对象有引用计数和版本信息，每次使用flush操作会递增版本信息。
server层表定义缓存由hash表和old_unused_share链表组成，通过hash表table_def_cache以表名为key缓存TABLE_SHARE对象，同时未使用的TABLE_SHARE对象通过old_unused_share链表链接。

* 获取TABLE_SHARE(`get_table_share`)
先从HASH查找，找不到再读取frm文件加载表定义信息。同时递增引用计数。
* 释放TABLE_SHARE(`release_table_share`)
递减引用计数。当引用计数为0时，如果版本发生变化，直接删除此TABLE_SHARE。

old_unused_share链表调整:

* 获取TABLE_SHARE时(`get_table_share`)
未使用的TABLE_SHARE对象被启用，须从LRU链表取出；
如果缓存总数超出table_definition_cache大小，须依次从old_unused_share链表尾部去除。
* 释放TABLE_SHARE时(`release_table_share`)
当引用计数为0时，如果版本没有发生变化，将TABLE_SHARE对象加入old_unused_share链表尾部。如果缓存总数超出table_definition_cache大小，须依次从old_unused_share链表尾部去除。
真正free TABLE_SHARE对象时，如果此对象还在old_unused_share链表中，须从其中去除。

## InnoDB层表定义缓存

InnoDB表定义为`dict_table_t`， 缓存为`dict_sys_t`，结构如下

`struct dict_sys_t{
 ...
 hash_table_t* table_hash; /*!< hash table of the tables， based
 on name */
 hash_table_t* table_id_hash; /*!< hash table of the tables， based
 on id */
 ulint size; /*!< varying space in bytes occupied
 by the data dictionary table and
 index objects */
 dict_table_t* sys_tables; /*!< SYS_TABLES table */
 dict_table_t* sys_columns; /*!< SYS_COLUMNS table */
 dict_table_t* sys_indexes; /*!< SYS_INDEXES table */
 dict_table_t* sys_fields; /*!< SYS_FIELDS table */

 UT_LIST_BASE_NODE_T(dict_table_t)
 table_LRU; /*!< List of tables that can be evicted
 from the cache */
 UT_LIST_BASE_NODE_T(dict_table_t)
 table_non_LRU; /*!< List of tables that can't be
 evicted from the cache */
};
`
主要由hash表和LRU链表组成。

* 两个hash表，分别按name和id，便于按name和id进行查找。
* table_non_LRU：
存放不放入到LRU链表的表，这些表不会从缓存中淘汰出去。那么哪些表会放入table_non_LRU链表呢？
 
 系统表，如sys_tables sys_columns sys_fields SYS_INDEXES等；
* 有引用关系的表都加入table_non_LRU(dict_foreign_add_to_cache)；
* 有全文索引的表都加入table_non_LRU(fts_optimize_add_table)；
* 便于删表，删表前对将表加入table_non_LRU，删表时加载表时保证表仍然在缓存中，例如表corrupted时。
* table_LRU
不在table_non_LRU链表中的表都加入table_LRU链表中。
* dict_table_t* sys_tables 等
常用系统表单独标识出来，每次使用时直接取出，不需要从hash表查找。
* LRU的维护
既然存在table_LRU链表，我们就需要考虑LRU的调整：

 将最近使用的表放入LRU头部（`dict_move_to_mru`）
每次按name和id查找时都会调整，参考`dict_table_open_on_name`和`dict_table_open_on_id`。
* LRU的淘汰

 淘汰哪些表
 LRU中表才可以淘汰，table_non_LRU中的表不参入淘汰。
 表引用计数必须为0(`table->n_ref_count == 0`)。
 表的索引被自适应哈希引用计数必须为0(`btr_search_t->ref_count=0`)。
* 何时淘汰
主线程控制每47（SRV_MASTER_DICT_LRU_INTERVAL）秒检查一次，只遍历一半LRU链表。
主线程空闲时检查一次，但扫所有LRU链表，清理控制缓存表个数不能超过table_definition_cache。
* 如何淘汰
从LRU尾部开始，淘汰满足条件表(`dict_make_room_in_cache`)。

注：

1. table_non_LRU没有实际作用，主要用于debug；
2. 如果有较多引用约束的表，它们不受LRU管理，参数table_definition_cache的作用会弱化。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)