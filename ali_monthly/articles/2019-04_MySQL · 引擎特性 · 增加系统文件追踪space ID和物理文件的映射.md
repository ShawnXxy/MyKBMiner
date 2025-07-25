# MySQL · 引擎特性 · 增加系统文件追踪space ID和物理文件的映射

**Date:** 2019/04
**Source:** http://mysql.taobao.org/monthly/2019/04/07/
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

 ## MySQL · 引擎特性 · 增加系统文件追踪space ID和物理文件的映射 
 Author: yinfeng 

 前面我们提到了MySQL5.7的几个崩溃恢复产生的性能退化 为了解决崩溃恢复的效率问题， MySQL8.0对crash recovery的逻辑进行了进一步的优化。 在之前的版本中，InnoDB通过向redo log中写入日志来追踪在一次checkpoint后修改过的表空间信息，这样就无需在crash recovery时打开所有的表空间，只需搜集哪些被影响到的表空间。而到了8.0新版本里，采用了一种全新的方式：单独创建了系统映射文件， 将space id及路径信息轮换着写到两个指定的系统文件tablespaces.open.1 and tablespaces.open.2中(ref Fil_Open::write)

实现的思路其实不复杂，就是将所有的表空间ID和对应的路径信息存储到系统文件中，在崩溃恢复时再按需打开。

## 系统文件更新
那么如何保证所有的表空间信息都一个不漏的存储到系统文件了呢 ？ 实际上他跟踪了所有的表空间文件操作，并更新内存cache中(Fil_Open::m_spaces), 如下：

`a. fil_node_open_file
 fil_system->m_open.enter();
 fil_system->m_open.log(node->space->id, node->name);
 fil_system->m_open.exit();
`
打开表空间文件后，写一条日志MLOG_FILE_OPEN， 并将表空间状态 Nodes::OPEN以及日志end lsn在内存中进行更新(Fil_Open::Nodes::load)

`b. fil_node_close_file
 fil_system->m_open.enter();
 fil_system->m_open.close(node->space->id, node->name);
 fil_system->m_open.exit();
`

关闭表空间文件后， 将缓存的表空间信息LSN重置为0，并将状态设置为CLOSED (Fil_Open::Nodes::close)

`c. fil_name_write_rename
 fil_system->m_open.enter();
 fil_system->m_open.log(space_id, new_name);
 fil_system->m_open.to_file();
 fil_system->m_open.exit();
`

在物理rename文件之前， 将新的表空间名通过MLOG_FILE_OPEN写到redo log中，记录新文件的状态到内存。

随后就将缓存的表空间信息写到系统映射文件中(Fil_Open::to_file)

`d. fil_delete_tablespace
 fil_system->m_open.enter();
 fil_system->m_open.deleted(id);
 fil_system->m_open.exit();
`

在物理删除文件之后，将对应的表空间状态设置为DELETED (Fil_Open::deleted)

`e. fil_ibd_create
 fil_system->m_open.enter();
 fil_system->m_open.open(space_id, file->name, log_get_lsn());
 fil_system->m_open.exit();
`

在物理创建表空间文件之后， 调用Fil_Open::open 将新文件的信息存储到内存中。同样的包含创建文件时的LSN

可见InnoDB在对文件进行打开，关闭，创建，删除，重命名这些操作时都进行了追踪，其中CREATE/DELETE/RENAME的cache更新均发生在记录对应的MLOG_FILE_*日志之前。

另外我们也可以看到，表空间信息不是直接写入的，而是经过zip压缩后再写的，以减少磁盘空间占用。

那么何时将缓存的信息刷到磁盘呢 ？
第一种情况是rename tablespace时，会做一次写文件
第二种情况是做checkpoint之前会去做一次flush(fil_tablespace_open_sync_to_disk), 相比第一种情况，这里先做一次清理(Fil_Open::purge -> Fil_Open::Nodes::purge)，将状态为DELETED/MISSING的无效表空间记录删除掉，再刷到磁盘

当系统正常关闭时，InnoDB会去将系统文件中的信息全部清除掉(fil_tablespace_open_clear)，因为崩溃恢复无需用到。

## 崩溃恢复
那么崩溃恢复时，如何使用该文件呢？

首先在启动时(srv_start)， 当确定了需要崩溃恢复时(recv_recovery_from_checkpoint_start)，就会去从系统映射文件中载入表空间信息到内存中(fil_tablespace_open_init_for_recovery –> Fil_Open::from_file)。

随后开始读redo log并解析, 如下堆栈:

`recv_recovery_begin
 |--> recv_scan_log_recs
 |--> recv_parse_log_recs
 |--> recv_single_rec
 |--> recv_multi_rec
`

在将redo log加入到hash table之前，会先进行判断，只有在文件中找到的表空间，才需要去apply日志。

`if (space_id == TRX_SYS_SPACE
 || fil_tablespace_lookup_for_recovery(space_id)) {

 recv_add_to_hash_table(
 type, space_id, page_no, body,
 ptr + len, old_lsn, recv_sys->recovered_lsn);

} else {

 recv_sys->missing_ids.insert(space_id);
}
`

由于系统文件不是实时flush的，因此在解析到MLOG_FILE_*类型的redo时， 也要对缓存的表空间信息进行修正(fil_tablespace_name_recover –> fil_name_process_for_recovery) ，以确保所有需要apply redo的tablespace都load到内存中。

在执行崩溃恢复时，InnoDB会按需去打开表空间文件，然后再去apply日志。（recv_apply_hashed_log_recs –> fil_tablespace_open_for_recovery），只有那些需要做崩溃恢复的文件，才会被打开。

 Note1: 本文所有代码相关的内容都是基于MySQL8.0.3，而目前版本还处于RC和快速开发的状态，不排除后面的版本逻辑，函数名等发生变化。

Note2: 主要代码在这个[commit](https://github.com/mysql/mysql-server/commit/201b2b20d110bc35ddf699754571cb0c064a3f72?spm=a2c4e.11153940.blogcont221677.14.4e0e222bbxKOjZ) 中，感兴趣的也可以自行阅读代码

Note3: 从8.0.11开始，又改成了打开全部ibd文件，但是改成了并行扫描

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)