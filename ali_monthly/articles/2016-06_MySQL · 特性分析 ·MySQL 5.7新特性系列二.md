# MySQL · 特性分析 ·MySQL 5.7新特性系列二

**Date:** 2016/06
**Source:** http://mysql.taobao.org/monthly/2016/06/02/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 06
 ](/monthly/2016/06)

 * 当期文章

 MySQL · 特性分析 · innodb 锁分裂继承与迁移
* MySQL · 特性分析 ·MySQL 5.7新特性系列二
* PgSQL · 实战经验 · 如何预测Freeze IO风暴
* GPDB · 特性分析· Filespace和Tablespace
* MariaDB · 新特性 · 窗口函数
* MySQL · TokuDB · checkpoint过程
* MySQL · 特性分析 · 内部临时表
* MySQL · 最佳实践 · 空间优化
* SQLServer · 最佳实践 · 数据库实现大容量插入的几种方式
* MySQL · 引擎特性 · InnoDB COUNT(*) 优化(?)

 ## MySQL · 特性分析 ·MySQL 5.7新特性系列二 
 Author: lengxiang 

 继上一期月报，MySQL5.7新特性之一介绍了一些新特性及兼容性问题后，本期继续进行学习。

### 1. 系统变量
5.7以后System and status 变量需要从performance_schema中进行获取，information_schema仍然保留了GLOBAL_STATUS，GLOBAL_VARIABLES两个表做兼容。

**[兼容性]** 

 如果希望沿用information_schema中进行查询的习惯，5.7提供了show_compatibility_56参数，设置为ON可以兼容5.7之前的用法，否则就会报错：

`ERROR 3167 (HY000): The 'INFORMATION_SCHEMA.GLOBAL_STATUS' feature is disabled; see the documentation for 'show_compatibility_56'
`
5.7.6之后，在performance_schema新增了如下的表：

`performance_schema.global_variables
performance_schema.session_variables
performance_schema.variables_by_thread
performance_schema.global_status
performance_schema.session_status
performance_schema.status_by_thread
performance_schema.status_by_account
performance_schema.status_by_host
performance_schema.status_by_user
`
5.7.9之前，需要有SELECT_ACL权限才能进行show查询，但5.7.9之后，默认这些表是不需要任何权限就可以访问了。

### 2. sys schema
新增了sys数据库，主要是performance_schema收集的信息，帮助DBA和开发人员方便诊断问题。
sys下的一共包括三种对象：1. view，2. procedure 3 function
这些对象都是基于performance_schema下的表，进行了可读性的聚合，没有真正存储数据，只存储了定义。

**[兼容性]** 

mysql_install_db可以选择–skip-sys-schema跳过安装过程， 但默认mysql_upgrade会帮你创建sys下面的对象。不存在兼容性的问题

### 3. 异常栈
5.7开始支持异常诊断栈信息，通过GET STACKED DIAGNOSTICS可以获取栈内的信息。
具体的使用方法参考：https://dev.mysql.com/doc/refman/5.7/en/diagnostics-area.html

### 4. Triggers
支持在一个table对象上建多个trigger。

### 5. Generated Columns
5.7.6开始，支持生成列，这个列可以是虚拟的列，也可以是实体存储数据的列。
比如：

`CREATE TABLE triangle (
 sidea DOUBLE,
 sideb DOUBLE,
 sidec DOUBLE AS (SQRT(sidea * sidea + sideb * sideb))
);
`

VIRTUAL： 表示这个字段是虚拟列，并不进行存储，查询的时候，通过计算得到

STORED： 需要存储空间，并且可以被索引的列

### 6. exchange partition不验证
这个是在oracle分区表上支持的功能，dba在做大表维护的时候，非常有用。

`语法： ALTER TABLE ... EXCHANGE PARTITION WITHOUT VALIDATION 
`

如果不验证，那么只有元数据信息的更改，就可以完成exchange，否则，就需要读取每一行数据进行验证，维护时间将根据这个表大小有关系。

### 7. dump线程增强
5.7.2之前，master dump线程需要持有LOCK_log锁去读取binlog然后发送到备库，而这时会阻塞client端去写入binlog。5.7.2之后，dump线程只需要持有LOCK_binlog_end_pos这个锁去读取binlog的当前的位置，来决定是否发送到备库去，这样就可以做到不阻塞任何binlog的写入。

### 8. 多源复制
多源复制可以从多个master复制到一个slave端，在数据库集群进行扩容和缩容的时候，非常有用。我们会在后面的系列单独来介绍。

### 9. 在线更改replication master
可以不用stop slave，然后在线更改replication master信息。 但这里并不是不需要slave停掉， 而是change master涉及到几个动作：

1. 如果只是更改当前relay的信息，那么只需要sql线程是不工作的就可以了，IO thread可以继续
2. 如果只是更改主库的信息，那么只需要IO线程不工作就可以了。 sql thread可以继续
3. 如果需要重新启动主库和备库的恢复信息，比如master_auto_positioin=1，那么就需要IO和sql线程都停掉。

### 10. Group Replication
并行复制支持按照主库组提交的形式在备库进行回放。下一个系列进行单独来介绍

**下面单独介绍一下MySQL 5.7对临时表进行的改动。**

### 1. 背景

MySQL包括两类临时表，一类是通过create temporary table创建的临时表，一类是在query过程中using temporary而创建的临时表。
5.7之前，using temporary创建的临时表，默认只能使用myisam引擎，而在5.7之后，可以选择InnoDB引擎来创建。

临时表的引擎选择使用下面的这两个参数来决定：

`mysql> show global variables like '%tmp%';
+----------------------------------+---------------------------------------+
| Variable_name | Value |
+----------------------------------+---------------------------------------+
| default_tmp_storage_engine | InnoDB |
| internal_tmp_disk_storage_engine | InnoDB |
`

### 2. 临时表空间
5.7之后，使用了独立的临时表空间来存储临时表数据，但不能是压缩表。临时表空间在实例启动的时候进行创建，shutdown的时候进行删除。

例如如下的配置：

`mysql> show global variables like '%innodb_temp%'; 
+----------------------------+-----------------------+
| Variable_name | Value |
+----------------------------+-----------------------+
| innodb_temp_data_file_path | ibtmp1:12M:autoextend |
+----------------------------+-----------------------+
`
create temporary table和using temporary table将共用这个临时表空间。

### 3. 临时表优化
临时表会伴随着大量的数据写入和读取，尤其是internal_tmp_table。所以，InnoDB专门对临时表进行了优化。

InnoDB使用如下两个标示临时表：

`dict_tf2_temporary： 表示普通临时表 
dict_tf2_intrinsic： 表示内部临时表 
`

这两个标示，会在IBD文件的segment header占用两个bit位。intrinsic一定是temproary，也就是temproary上进行的优化
完全适用于intrinsic表上。

**下面来看下具体的优化：**

### 3.1. redo
临时表在连接断开或者数据库实例关闭的时候，会进行删除，所以，临时表的数据不需要redo来保护，即recovery的过程中
不恢复临时表，只有临时表的metadata使用了redo保护，保护元数据的完整性，以便异常启动后进行清理工作。

临时表的元数据，5.7之后，使用了一个独立的表进行保存，这样就不要使用redo保护，元数据也只保存在内存中。
 但这有一个前提，必须使用共享的临时表空间，如果使用file-per-table，仍然需要持久化元数据，以便异常恢复清理。

### 3.2 undo
temporary table仍然需要语句级的回滚，所以，需要为数据生成undo。但intrinsic table不需要回滚，所以，intrinsic table
 减少了undo的生成，性能更高。

### 3.3 lock
因为临时表只有本线程可以看见，所以减少了InnoDB的加锁过程。

可以看下insert的时候，进行的分支判断：

` row_insert_for_mysql(
 const byte* mysql_rec,
 row_prebuilt_t* prebuilt)
{
 /* For intrinsic tables there a lot of restrictions that can be
 relaxed including locking of table, transaction handling, etc.
 Use direct cursor interface for inserting to intrinsic tables. */
 if (dict_table_is_intrinsic(prebuilt->table)) {
 return(row_insert_for_mysql_using_cursor(mysql_rec, prebuilt));
 } else {
 return(row_insert_for_mysql_using_ins_graph(
 mysql_rec, prebuilt));
 }
}
`
row_insert_for_mysql_using_cursor直接跳过了加锁的lock_table过程。

然后，如果是intrinsic table，就直接插入，减少了undo的生成。

如果不是，需要加lock，并生成undo信息。

`if (dict_table_is_intrinsic(index->table)) {

 index->rec_cache.rec_size = rec_size;

 *rec = page_cur_tuple_direct_insert(
 page_cursor, entry, index, n_ext, mtr);
 } else {
 /* Check locks and write to the undo log,
 if specified */
 err = btr_cur_ins_lock_and_undo(flags, cursor, entry,
 thr, mtr, &inherit);
`

插入的时候，如果是临时表。就关闭redo的生成。如下面的代码所示：

`if (dict_table_is_temporary(index->table)) {
 /* Disable REDO logging as the lifetime of temp-tables is
 limited to server or connection lifetime and so REDO
 information is not needed on restart for recovery.
 Disable locking as temp-tables are local to a connection. */

 ut_ad(flags & BTR_NO_LOCKING_FLAG);
 ut_ad(!dict_table_is_intrinsic(index->table)
 || (flags & BTR_NO_UNDO_LOG_FLAG));

 mtr.set_log_mode(MTR_LOG_NO_REDO);
 }
`

未完待续，下一个系列，我们将介绍一下undo的新特性，包括online truncated undo。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)