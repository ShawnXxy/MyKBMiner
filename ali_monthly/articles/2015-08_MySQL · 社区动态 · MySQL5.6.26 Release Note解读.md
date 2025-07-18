# MySQL · 社区动态 · MySQL5.6.26 Release Note解读

**Date:** 2015/08
**Source:** http://mysql.taobao.org/monthly/2015/08/03/
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

 ## MySQL · 社区动态 · MySQL5.6.26 Release Note解读 
 Author: 印风 

 最近上游发布了MySQL 5.6.26版本，从Release Note来看，MySQL 5.6版本已经相当成熟，fix的bug数越来越少了。本文主要分析releae note上fix的相关bug，去除performance scheama、mac及windows平台、企业版、package相关内容。从本期开始，我们会在新版本发布时，在当月的月报上为大家做详细的版本Release Note分析。

## InnoDB storage engine

**问题描述**
在类Unix平台上，当innodb_flush_method设置为O_DIRECT时，函数`os_file_create_simple_no_error_handling_func`没有使用O_DIRECT方式打开数据文件。例如在函数`fil_node_open_file`中，可能先以函数`os_file_create_simple_no_error_handling_func`打开文件，确定文件的大小，然后关闭文件；再以`os_file_create`打开数据文件，前者使用Buffered IO，后者使用DIRECT IO。这种混合使用可能引发性能问题。

根据man手册建议：

 Applications should avoid mixing O_DIRECT and normal I/O to the same file, and especially to overlapping byte regions in handles the coherency issues in this situation, overall I/O the same file. Even when the filesystem correctly throughput is likely to be slower than using either mode of files with direct I/O to the same files.” alone. Likewise, applications should avoid mixing mmap(2)

(Bug #21113036, Bug #76627)

**解决**
在函数`os_file_create_simple_no_error_handling_func` 中禁止OS Cache（函数`os_file_set_nocache`）

**补丁**
[b4daac21f52ced96c11632b83445111c0acede56](https://github.com/mysql/mysql-server/commit/b4daac21f52ced96c11632b83445111c0acede56)

**问题描述**
在将一个脏页从非压缩page拷贝到压缩页后，在写page到文件时(`buf_flush_write_block_low`)，在设置压缩页的修改LSN之前先调用了函数`page_zip_verify_checksum`，由于此时压缩页上的LSN为0，而计算出来的checksum也可能为0，此时`page_zip_verify_checksum`认为要尝试写入一个空page，返回false，导致断言失败(Bug #21086723)。

**解决**
先设置LSN，再调用`page_zip_verify_checksum`。

**补丁**
[5b6041b2c7cbee8a1d917631d3a051122b8c4f8d](https://github.com/mysql/mysql-server/commit/5b6041b2c7cbee8a1d917631d3a051122b8c4f8d)

**问题描述**
当以如下序列执行时，实例会crash:

`create database `b`;
use b;
create table `#mysql50#q.q` select 1;
drop database `b`;
`

在创建表时，发现非法的表名，表名被reset成一个空字符串，传递到引擎层就是”dbname/”， 而引擎层的数据词典定义中，是通过“dbname/tablename”这样的字符串来定位的，这就违反了数据词典的约定。 随后如果执行drop database, 会去遍历以db名作为前缀的数据词典项，触发crash。PS：即使重启实例，drop database，也无法执行清理操作，用户线程会不停的在drop db的逻辑里loop(Bug #19929435)。

**解决**
在引擎层拒绝创建空的表名。

**补丁**
[8fd710e06024a890e08e35009da541194ca0e5a4](https://github.com/mysql/mysql-server/commit/8fd710e06024a890e08e35009da541194ca0e5a4)

**问题描述**
在函数`innobase_get_foreign_key_info`中，需要根据子表中存储的父表表名去打开父表，但子表上是根据系统字符集system_charset_info存储的，而innodb是使用my_charset_filename存储表名和库名，因此如果包含父表包含特殊字符，就会造成无法打开父表，导致报错。(Bug #21094069)

**解决**
将系统字符集的表名和库名转换成my_charset_filename格式（tablename_to_filename）。

**补丁**
[1fae0d42c352908fed03e29db2b391a0d2969269](https://github.com/mysql/mysql-server/commit/1fae0d42c352908fed03e29db2b391a0d2969269)

**问题描述**

* 当一个IO后台线程为了做ibuf merge，需要读入对应数据文件的bitmap page时(check 函数`buf_page_io_complete` –> `ibuf_merge_or_delete_for_page`),读取方式为同步读, `space->n_pending_ops`递增；
* 另外一个用户线程准备删除对应的tablespace，因此将`space->stop_new_ops`设置为true，并等待直到`space->n_pending_ops`为0（`fil_check_pending_operations`）；
* 后台线程尝试读入ibuf bitmap page，但由于在`fil_io`函数中，如果发现`space->stop_new_ops`被设置，所有的读操作都被拒绝，直接返回DB_TABLESPACE_DELETED错误，但在函数`ibuf_merge_or_delete_for_page`中总是认为ibuf bitmap page被成功读入内存，后面直接引用这个page（实际上是空指针），可能会导致实例crash。

**解决**
在进行fil_io时，如果表空间正在被删除(`space->stop_new_ops`被设置为true），不允许异步读操作，但允许写操作和同步读操作。

**补丁**
[3ba4563a757e07c3052c780b63e2626c78ca5c47](https://github.com/mysql/mysql-server/commit/3ba4563a757e07c3052c780b63e2626c78ca5c47)

**问题描述**
当表上的索引存在前缀索引时(prefix index)，对表进行export，再import tablespace可能会失败，并报Schema mismatch错误，错误码为ER_TABLE_SCHEMA_MISMATCH。test case见bug#76877 (Bug #20977779, Bug #76877)。原因是cfg文件和表的索引定义相匹配时逻辑错误，例如如下表：

`CREATE TABLE t1 (c1 VARCHAR(128), PRIMARY KEY (c1(16))) ENGINE=InnoDB;
`

在索引对象中定义了4个列：(c1, prefix_len=16), (DB_TRX_ID), (DB_ROLL_PTR)，(c1, prefix_len=0)。
cfg和表索引对象相比较时，其实两者是一样的，但cfg在取列时，如果存在相同列名的，总是取第一个，如上例，在比较第四个列的schema是否一致时，取的实际上是第一个，从而产生报错。

参考函数：`row_import::match_index_columns` ((Bug #20977779, Bug #76877))。

**解决**
一个列一个列的依次校验。

**补丁：**
[db23392bac27ad3e84319229ee3db9921b734abd](https://github.com/mysql/mysql-server/commit/db23392bac27ad3e84319229ee3db9921b734abd)

**问题描述**
考虑如下场景：

1. purge线程读取一个undo ，解析出对应的记录 (`row_purge` —> `row_purge_parse_undo_rec`)；
2. 先 purge 二级索引(`row_purge_remove_sec_if_poss`)，再purge聚集索引(`row_purge_remove_clust_if_poss`)；
3. 当 purge 二级索引页时，需要检查二级索引记录是否可以被物理purge掉(`row_purge_remove_sec_if_poss_leaf`)。

参考函数：`row_purge_poss_sec`

`can_delete = !row_purge_reposition_pcur(BTR_SEARCH_LEAF, node, &mtr)
 || !row_vers_old_has_index_entry(TRUE,
 btr_pcur_get_rec(&node->pcur),
 &mtr, index, entry,
 node->roll_ptr, node->trx_id);
`

`row_purge_reposition_pcur`定位到聚集索引上，`node->found_clust`设置为true，定位到clust index上的cursor存储在node->pour上。

* 然后再检查二级索引记录是否被标记删除了，(`row_purge_remove_sec_if_poss_leaf` —> `red_get_deleted_flag`)，如果没有被标记删除，则报warning。

但是步骤3中，即时二级索引没有被标记删除，在函数`row_purge_poss_sec`也返回了true，这是因为重新定位cursor的逻辑错误。

函数`row_purge_reposition_pcur`:

`if (node->found_clust) {
 ibool found;
 found = btr_pcur_restore_position(mode, &node->pcur, mtr);
 return(found);
} else {
 node->found_clust = row_search_on_row_ref(
 &node->pcur, mode, node->table, node->ref, mtr);
 if (node->found_clust) {
 btr_pcur_store_position(&node->pcur, mtr);
 }
}

return(node->found_clust);
`

考虑如下序列：

1. purge Index1时，根据node->ref找到对应的聚集索引记录，node->found_clust设置为true，当前cursor存到node->pour中；
2. 其他用户线程操作了聚集索引页，导致在purge index2时，restore position可能失败，因此返回false；
3. 随后purge index2，发现node->found_clust为true，依旧用上次restore的position来作restore，依然失败；在函数`row_purge_reposition_pcur`返回false就认为对应的聚集索引不存在，然后就去尝试删除二级索引记录；但注意这次想purge的二级索引记录可能是一个新鲜插入的记录，并没有被delete mark，我们实际上需要根据node->ref重新定位。

**解决**
在函数`row_purge_reposition_pcur`中，若是restore cursor失败，需要重置node->found_clust为false (Bug #19138298, Bug #70214, Bug #21126772, Bug #21065746)

**补丁**
[982a157c71667040838def7a00d951ffc55eccbc](https://github.com/mysql/mysql-server/commit/982a157c71667040838def7a00d951ffc55eccbc)
[4b8304a9a41c8382d18e084608c33e5c27bec311](https://github.com/mysql/mysql-server/commit/4b8304a9a41c8382d18e084608c33e5c27bec311)
[e59914034ab695035c3fe48f046a96bb98d53044](https://github.com/mysql/mysql-server/commit/e59914034ab695035c3fe48f046a96bb98d53044)
[92b4683d59c066f099be1d283c7d61b00caeedb2](https://github.com/mysql/mysql-server/commit/92b4683d59c066f099be1d283c7d61b00caeedb2)

## InnoDB 全文索引

**问题描述**
尝试为表上rebuild 全文索引，但表上已经有损坏的索引时，会触发assert。(Bug #20637494)

**解决**
抛出错误，提示用户先删掉损坏的索引。返回错误码为ER_INNODB_INDEX_CORRUPT。

**补丁**
[4395ad1755c3ed86c4210f76001a76eb0a69b553](https://github.com/mysql/mysql-server/commit/4395ad1755c3ed86c4210f76001a76eb0a69b553)
[3bdb4573e9b25357eea2421647263216c36367cb](https://github.com/mysql/mysql-server/commit/3bdb4573e9b25357eea2421647263216c36367cb)

**问题描述**
构建full-text的表上存在隐藏的FTS_DOC_ID和唯一索引FTS_DOC_ID_INDEX（FTS_DOC_ID），当删除全文索引时，对应的隐藏列并没有删除，但在当前的逻辑中，如果存在FTS_DOC_ID，则不允许ONLINE DDL(Bug #20590013, Bug #76012)。

**解决**
当表上只有FTS_DOC_ID_INDEX和FTS_DOC_ID 但没有定义全文索引时，允许ONLINE DDL。这些隐藏列直到全表rebuild时才被删除。

**补丁**
[5610e5354a8be6609b2fc2a37902961be26af3cf](https://github.com/mysql/mysql-server/commit/5610e5354a8be6609b2fc2a37902961be26af3cf)

## InnoDB API/Memcached

**问题描述**
`ib_cursor_moveto` 函数没有判断构建的tuple的列个数是否小于索引列个数，而是直接用索引列的个数来做遍历，可能导致段错误(Bug #21121197, Bug #77083)。

**解决**
加上对应的判断。

**补丁：**
[d511b503353c1588e90907f59b947e31796c1fc1](https://github.com/mysql/mysql-server/commit/d511b503353c1588e90907f59b947e31796c1fc1)

**问题描述**
`ib_table_truncate`函数中，当truncate失败时，没有正确的释放事务对象，可能导致shutdown hang住。

**解决:**
总是释放事务对象。

**补丁:**
[aeef8dc2c7af8be4f8ac91be6963e5252e8a9d3f](https://github.com/mysql/mysql-server/commit/aeef8dc2c7af8be4f8ac91be6963e5252e8a9d3f)
[e0e1f02d97f54252c1e6ea386dc029560c9f7d08](https://github.com/mysql/mysql-server/commit/e0e1f02d97f54252c1e6ea386dc029560c9f7d08)

**问题描述**
`ib_open_table_by_id`函数中，已经加了`dict_sys->mutex`锁，但该函数中调用`dict_table_open_on_id`传递的第二个参数为FALSE，认为没有持有mutex，属于基本的逻辑错误(Bug #21121084, Bug #77100)。

**解决**
调整传参。

**补丁**
[a2353c5d7ff6430e853de435d007ac64d91fd17d](https://github.com/mysql/mysql-server/commit/a2353c5d7ff6430e853de435d007ac64d91fd17d)

上面几个bug看起来都是非常“低级”的代码缺陷，这也侧面证明了InnoDB API接口在推出后社区用的人实在太少了，这三个Bug都是facebook的工程师提出的，很好奇他们会利用InnoDB API做些什么。

**问题描述**
InnoDB memcached plugin在处理unsigned NOT NULL类型时没有正确处理，导致返回的数据错误。

* 对于unsigned类型，对应的IB_COL_UNSIGNED = 2
* 对于NOT NULL类型，对应的IB_COL_NOT_NULL = 1

但是代码里很多地方都使用类似`m_col->attr == IB_COL_UNSIGNED`，导致大量的逻辑错误(Bug #20535517, Bug #75864)。

**解决**
修改成`m_col->attr & IB_COL_UNSIGNED`。

**补丁：**
[6ff8d5d2940b9c9079e07641b2beb12e8dd84b38](https://github.com/mysql/mysql-server/commit/6ff8d5d2940b9c9079e07641b2beb12e8dd84b38)

## 复制

**问题描述**
当使用多线程复制时，执行STOP SLAVE需要等待所有的worker线程完成其各自的工作队列中的事务。如果Pending的事务很多，可能要等待很长时间才能完成STOP SLAVE，另外在STOP SLAVE的过程中，是无法SHOW SLAVE STATUS的，一种比较常见的场景就是大量的监控程序SQL堵塞堆积(Bug #75525, Bug #20369401)。

**解决**

解决方案是先找到任意worker线程中最新的commit的事务，确定一个上限位点，所有的worker线程执行到这个位置停止，剩下的事务暂时不执行。具体的：

1. 执行STOP SLAVE，coordinator线程首先将所有worker线程的状态设置成STOP（`slave_stop_workers(rli, &mts_inited)`），并更新`rli->max_updated_index`为最新的已经执行（或正在执行）的事务的group index(`set_max_updated_index_on_stop`)；
2. 所有worker的工作队列中索引序号小于等于 `rli->max_updated_index` 的事务都需要被执行完，否则worker状态设置为STOP_ACCEPTED，表示已经完成了max_updated_index 之前的事务，可以退出(`set_max_updated_index_on_stop`)；
3. coordinator线程等待所有worker线程退出，并做一次checkpoint(`slave_stop_workers` –> `mts_checkpoint_routine`)。

但是上述方案并不能解决正在执行的大事务过慢的问题。

**补丁**
[37f2e969bd36a7455e81ea2350685707bc859866](https://github.com/mysql/mysql-server/commit/37f2e969bd36a7455e81ea2350685707bc859866)

**问题描述**
 MySQL使用InnoDB + binlog做XA的方式来进行crash recovery，但在之前的版本中如果写Binlog到磁盘发生了错误，group commit的逻辑并没有感知到这个错误，而是继续在引擎层提交事务，备库没有接收到对应的Binlog，导致主备数据不一致 (Bug #76795, Bug #20938915)。

**解决**
从MySQL 5.6.22版本开始，引入了一个新参数binlog_error_action (5.6.20及21版本叫做binlogging_impossible_mode)，若设置为ABORT_SERVER，则在发生binlog写入错误时直接让实例退出，避免引发更大的错误；若设置为IGNORE_ERROR，则忽略本次写入失败，同时禁止Binlog记录，需要重启才能让binlog再次开启。
为了主备数据的强一致性，通常应该将binlog_error_action设置为ABORT_SERVER，这样在打开文件、rotate新文件、从IO Cache写binlog到文件出现磁盘错误时，都会退出实例。

**补丁**
[3b6b4bf8c5d1bfada58678acebafdf6f813c2dfe](https://github.com/mysql/mysql-server/commit/3b6b4bf8c5d1bfada58678acebafdf6f813c2dfe)

**问题描述**
relay_log_recovery参数打开时，备库在重启时就可以根据SQL线程执行到的位置重新拉binlog，这可以有效处理备库发生机器宕机导致relay log文件损坏的情况，无需人工去change master，在之前版本中，如果使用了多线程复制，是无法开启该特性的，在启动实例时会报如下错误：

 relay-log-recovery cannot be executed when the slave was stopped with an error or killed in MTS mode

实际上，如果开启了GTID，就无需关心各个worker线程之间的gap，通过备库的GTID集合充拉relay log即可(Bug #73397, Bug #19316063)。

**解决**
在重启recovery时检查是否开启了GTID。

**补丁**
[fce558959bd0e5af1ae6aac3d8573db00c271dfd](https://github.com/mysql/mysql-server/commit/fce558959bd0e5af1ae6aac3d8573db00c271dfd)

**问题描述**
当两台备库错误的配置了相同的server_uuid，并指向同一个主库时，备库的IO线程会被频繁的断开并尝试重连。而在备库来看，并没有足够的信息提示产生重连的原因。

**解决**
这种场景下，主库会生产一个错误信息传递到备库，当备库接受到这样的错误信息时不再尝试重连。(Bug #72581, Bug #18731252)

**补丁**
[751a3da76dfd66b92395f90f11fce6bd890c9db5](https://github.com/mysql/mysql-server/commit/751a3da76dfd66b92395f90f11fce6bd890c9db5)

## 分区表

**问题描述**
bug被隐藏，无test case，对应release note:

 Partitioning: In certain cases, ALTER TABLE … REBUILD PARTITION was not handled correctly when executed on a locked table. (Bug #75677, Bug #20437706)

**解决**
参考commit log.

**补丁**
[6b0e6683416dc6f8274a460bd2512e7b037ec75f](https://github.com/mysql/mysql-server/commit/6b0e6683416dc6f8274a460bd2512e7b037ec75f)

## 优化器
小编对优化器模块代码理解不深，感兴趣的同学可以自行阅读对应的bug report及commit log，手动尝试复现bug。

**问题描述**

 While calculating the cost for doing semjoin_dupsweedout strategy inner_fnout is calculated wrongly when max_outer_fanout becomes 0. This causes mysql server to exit later (Bug #21184091)

**解决**

 Calculate the inner_fanout only when max_outer_fanout is > 0. Else there is no need to recalculate inner_fanout w.r.t max_outer_fanout.

**补丁**
[bfba2338902a81927d116c30eaa1245eaea025c8](https://github.com/mysql/mysql-server/commit/bfba2338902a81927d116c30eaa1245eaea025c8)

**问题描述**

 GROUP BY or ORDER BY on a CHAR(0) NOT NULL column could lead to a server exit. (Bug #19660891)
ASSERTION `PARAM.SORT_LENGTH != 0’ FAILED IN SQL/FILESORT.CC:361

**解决**
参考commit log.

**补丁：**
[60c6920509516a1e05b855799479a59c27803191](https://github.com/mysql/mysql-server/commit/60c6920509516a1e05b855799479a59c27803191)
[b62c5daa646434290c9b2d1c9b162487cb8edf04](https://github.com/mysql/mysql-server/commit/b62c5daa646434290c9b2d1c9b162487cb8edf04)

**问题描述**

 When choosing join order, the optimizer could incorrectly calculate the cost of a table scan and choose a table scan over a more efficient eq_ref join. (Bug #71584, Bug #18194196)

**解决**
参考commit log.

**补丁：**
[7a36c155ea3f484799c213a5be5a3deb464251dc](https://github.com/mysql/mysql-server/commit/7a36c155ea3f484799c213a5be5a3deb464251dc)

## 其他

**问题描述**
MySQL String库下的字符串处理问题，在`cs_values`函数中，对字符串长度的处理存在缺陷，可能导致内存损坏(Bug #20359808)。

**解决**
调整长度判断。

**补丁**
[1cdd3b832ae32d3c236869954f0c7a8a851ed94a](https://github.com/mysql/mysql-server/commit/1cdd3b832ae32d3c236869954f0c7a8a851ed94a)

**问题描述**
当会话断开或者执行类似change user时，session status会merge到全局status中（`add_to_status(&global_status_var, &status_var)`），但没有立刻对thd的status_var做reset，这时候另外一个session去查询global status时，会重复把这些session的status值加到全局。

**解决**
在`THD::change_user`、`THD::release_resources`函数中累加到全局status后，重置session的status。

**补丁**
[c8243dd36047debb76134344d761e48f0cedf78e](https://github.com/mysql/mysql-server/commit/c8243dd36047debb76134344d761e48f0cedf78e)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)