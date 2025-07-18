# MySQL · 源码阅读 · InnoDB Export/Import Tablespace解析

**Date:** 2021/02
**Source:** http://mysql.taobao.org/monthly/2021/02/02/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2021 / 02
 ](/monthly/2021/02)

 * 当期文章

 PolarDB · 特性分析 · Explain Format Tree 详解
* MySQL · 源码阅读 · InnoDB Export/Import Tablespace解析
* MySQL · 源码解析 · MySQL 8.0.23 Hypergraph Join Optimizer代码详解
* MySQL · 性能优化 · InnoDB 事务 sharded 锁系统优化
* DataBase · 社区动态 · 数据库中的表达式
* MySQL · 源码分析 · Group by优化逻辑代码分析
* MySQL · 源码阅读 · X-plugin的传输协议
* MySQL · 源码阅读 · MySQL8.0 innodb锁相关
* PolarDB · 优化改进 · 使用窗口聚合函数来将子查询解关联

 ## MySQL · 源码阅读 · InnoDB Export/Import Tablespace解析 
 Author: rixiu 

 ## 背景

InnoDB中支持Transportable Tablespace功能。也就是表空间可以从一个实例迁移到另一个实例。相比mysqldump来进行导入导出而言，速度更快，而且使用也很便捷。本篇文章将从内核实现的角度来分析表空间export/import，重点是import的原理，参照8.0代码。

## 导出表空间

表空间的导出（export）。在源实例中，执行FLUSH TABLES t FOR EXPORT。首先对表上排他锁，即停读写。其次，停purge线程；最后写脏页并从缓冲区清除。到此文件可以拷贝了。注意，这里停了purge线程，所以在表空间文件中会有未被purge的记录存在。

## 丢弃表空间

在目的实例中，进行表空间的导入（import）之前，要先丢弃表空间（discard）。ALTER TABLE t DISCARD TABLESPACE;丢弃表空间的流程主要有两个。其一，重新给表指定id（见函数row_mysql_table_id_reassign()）。这步的主要目的是让purge线程在purge与该表相关的undo记录时，打开表失败，会跳过undo记录。其二，丢弃表空间：清除缓冲区的页面，以及表空间缓存。这些操作执行完成之后，表空间文件处于不一致性状态，也不可用。

## 导入表空间

丢弃了表空间之后，将export出来的表空间文件拷贝过来，覆盖丢弃的表空间文件，就可以进行导入（import）操作。ALTER TABLE t IMPORT TABLESPACE;

导入操作首先会对配置文件（cfg）进行校验，看看是否匹配。如果cfg文件缺失，则会尝试从索引的根页中读取相关信息。接下来则是对表空间文件对处理。总体而言，有两个主要的阶段（步骤）。

### 页面转换（PageConverter）

遍历表空间的所有页面，对单个页面进行检查和转换。

#### 页面更新

针对数据页面，主要是页头或页尾。主要修改如下(见PageConverter::update_page())：

1. space id：基本上所有使用的页面头上都有space id，需要修改为新的space id。
2. index id：针对索引页，需要修改为新的index id。
3. max trx id：针对索引页，import操作会开启一个新的事务，使用该事务的trx id
4. page lsn：针对所有使用的页，import开始页面转换时，读取系统当前最新的落盘lsn(flushed_to_disk_lsn)，使用该lsn作为新的page lsn。

#### 记录更新

针对页面内的记录。主要修改如下(见PageConverter::update_records())：

1. blob ref：针对聚集索引叶子页面(leaf page)。包含blob ref的记录，更新其对应的space id。
2. trx id：针对聚集索引叶子页面。重置记录中的trx id为新的trx id。
3. roll ptr：针对聚集索引叶子页面。重置记录中的roll ptr为空（0）。
4. 删除delete marked记录：针对所有索引叶子节点（包括聚集索引和二级索引）。如果有delete mark的记录，则直接删除。注意，如果要删除的记录是页面中最后一条，则保留。因为这里都是在页面上进行操作，没有btree cursor，不能进行btree的合并操作，而完整的btree结构要求页面中至少有一条记录。这里delete marked的记录是在export表空间时，没有被purge的记录。

在页面上完成更新之后，就会修改page lsn，并刷盘。在这一轮的页面转换中，为了减小对buffer pool的影响，页面的读取和写入都没有走buffer pool，而是单独设置的一定大小的缓冲区。这算是一个优化，见fil_tablespace_iterate()。

### 索引清理(IndexPurge)

通过cursor来遍历索引叶子页面，清理上一轮转换中剩下的delete marked的记录，见IndexPurge::garbage_collect()。这一轮索引清理，针对聚集索引和二级索引。本轮清理记录，调用函数btr_cur_pessimistic_delete()，相比上一轮调用函数page_delete_rec()要重很多。官方代码中提了一个todo优化：对于上一轮中不能删除的记录，可以写undo记录，最终由后台purge线程来清理，这样可以避免全表扫描操作。

除了以上两个重要的流程之外，import空间还做了一些其他更新：

1. 更新索引根页（root page）中两个段（segment）对应的space id。见函数btr_root_adjust_on_import()。
2. 更新最大的row id，即推高当前系统row id的水位。如果聚集索引默认使用row id，则执行此操作，见函数row_import_set_sys_max_row_id()。
3. 初始化表的自增列值，见dict_table_autoinc_initialize()。

最后，还做了数据检查，主要是检查聚集索引结构是否完整，见row_import_check_corruption()。

## 小结

整个import流程完成。主要的流程是在步骤页面转换和索引清理。页面转换在页面级别对delete mark的记录进行清理；索引清理则是在游标级别对步骤#1剩下对delete mark对记录进行清理。索引清理理论上可以在后台完成。总体而言，export的时间相对很快，取决于脏页的数量。import的时间在会比较慢，并且在整个过程中，表对外不可用。

export/import流程中有一点瑕疵：export之前，如果更新blob列，则只有新的blob保存在记录中，老的blob则保存在undo记录中。export开始之后，对应的undo记录未被purge，因此老的blob对应的数据页面，未被清理回收。在import流程中，因为没有undo记录，这些数据页面就泄漏了。

以上分析只是包含了主要的步骤和环节，如有遗漏，请多包涵。有兴趣的朋友可以深入代码中了解更多细节。

## 参考

1. https://dev.mysql.com/doc/refman/8.0/en/innodb-table-import.html
2. https://developer.aliyun.com/article/59271

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)