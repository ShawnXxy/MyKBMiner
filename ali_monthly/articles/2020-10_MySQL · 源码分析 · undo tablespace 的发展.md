# MySQL · 源码分析 · undo tablespace 的发展

**Date:** 2020/10
**Source:** http://mysql.taobao.org/monthly/2020/10/02/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2020 / 10
 ](/monthly/2020/10)

 * 当期文章

 MySQL · 源码分析 · 子查询优化源码分析
* MySQL · 源码分析 · undo tablespace 的发展
* MySQL · 最佳实践 · How to read the lock information from debugger

 ## MySQL · 源码分析 · undo tablespace 的发展 
 Author: 巴彦 

 首先我们介绍一下mysql5.6 undo tablespace独立表空间的内容，然后我们看一下后续5.6 8.0，mysql对独立表空间做出了那些修改。

### mysql5.6

从mysql5.6开始支持把undo log分开配置到独立的表空间，并放到单独的文件中；这给我们带来很多便利，对于该并发的写入，我们可以把undo文件单独部署到高速存储设备上。

### 参数

1，[innodb_undo_tablespaces](http://dev.mysql.com/doc/refman/5.6/en/innodb-parameters.html#sysvar_innodb_undo_tablespaces)

用于设定undo独立表空间的个数，在install db时初始化并创建，之后便不能修改。

默认值为0，表示不独立设置undo tablespace，默认记录到ibdata中；否则创建多个undo文件；加入设定值为8，那么就会创建命名为undo001~undo08的undo tablespace文件，每个文件大小10M。

2，[innodb_undo_logs](http://dev.mysql.com/doc/refman/5.6/en/innodb-parameters.html#sysvar_innodb_undo_logs)

用于设定回滚段的个数；该变量动态可调，但是物理上的回滚段不会减少，只是控制当前使用回滚段的个数；默认值128。

3，[innodb_undo_directory](http://dev.mysql.com/doc/refman/5.6/en/innodb-parameters.html#sysvar_innodb_undo_directory)

当我们开启独立undo tablespace 独立表空间时，这个用来设定存放undo文件的目录。

#### 相关实现

在inndo启动时(**innobase_start_or_create_for_mysql**)，会调用**srv_undo_tablespaces_init**来对undo表空间进行初始化。具体流程如下：

1. 如果是新实例，并且打开了独立表空间；则会去创建undo log文件，表空间的space id从1开始，文件大小默认10M，由SRV_UNDO_TABLESPACE_SIZE_IN_PAGES来控制；并记录space id。
2. 如果不是新实例，则读取当前实例所有undo表空间的space id(trx_rseg_get_n_undo_tablespaces)
trx_rseg_get_n_undo_tablespaces 函数会首先从ibdata中读取事务系统的文件头，然后从中拿到回滚段的信息；最后找到回滚段对应的space id(trx_sysf_rseg_get_space)和page no(trx_sysf_rseg_get_page_no),最后按照space id排序返回。
3. 按照上面两步拿到的space id依次打开undo 文件(srv_undo_tablespace_open)，并且space id要保证连续；如果space id不连续或者打开undo失败，则实例启动/初始化失败。
4. 最后，如果是新实例，则对所有space header进行初始化(fsp_header_init).

### mysql5.7

mysql5.7对于undo tablespace独立表空间的改动不大；只增加了一个功能，在线truncate undo log文件。具体参数可查看官方文档：[innodb_undo_log_truncate](http://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html?spm=a2c6h.12873639.0.0.f08631711l1pPV#sysvar_innodb_undo_log_truncate) ，官方博客对[online-truncate-of-innodb-undo-tablespaces](http://mysqlserverteam.com/online-truncate-of-innodb-undo-tablespaces/?spm=a2c6h.12873639.0.0.f08631711l1pPV)的介绍。

这个功能在生产环境的某些情况下比较有用，尤其是如果有长时间未提交事务导致浪费大量undo空间的情形。

#### 相关实现

1. 新的truncate管理类。

* 新的类undo::Trunc被引入，来管理tablespace truncate的过程；具体挂载在purge_sys->undo_trunc中。

1. 标记需要truncate的undo tablespace。

 * 这个动作实际上是由purge的协调线程发起的，默认情况下每做128次purge后，会调用函数trx_purge_truncate进行清理，trx_purge_truncate的调用流程如下：

 `trx_purge_truncate()
|
->trx_purge_truncate_history()
 |
 ->trx_purge_truncate_rseg_history
 |
 ->trx_purge_mark_undo_for_truncate()
 |
 ->trx_purge_initiate_truncate()
`
* trx_purge_mark_undo_for_truncate 是标记truncate undo tablespace的入口函数，主要步骤如下。

 检查是否开启truncate参数，已经有tablespace标记为truncate。
* 检查是否可以进行安全的truncate，也就是innodb_undo_tablespaces>=2, innodb_undo_logs>=35。
* 一次遍历当前活跃的undo tablespace，看看那些tablespace可以被truncate。
* 遍历被选中的回滚段，将其设置为不可分配。
* 在标记truncate完成后，需要检查需要删除的回滚段是否是可释放的，也就是没有任何活跃的事务会应用到启动的undo log，入口函数为trx_purge_initiate_truncate，此函数的流程如下：

 检查是否有undo tablespace标记需要truncate。
* 扫描所有需要truncate回滚段，不可以有任何活跃事务使用其中undo。
* 做一次redo checkpoint。
* 清理对应的purge queue，无需继续做purge操作。
* 调用trx_undo_truncate_tablespace执行真正的truncate。
* 再做一次redo checkpoint，然后做一些清理操作即可完成。

### mysql8.0

mysql8.0对undo tablespace做了进一步的优化；不仅仅支持更多的回滚段，而且还可以动态的增删undo tablespace。具体可查看[mysql8.0 undo tablespace 官方文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-undo-tablespaces.html?spm=a2c6h.12873639.0.0.310c35daWfE2bJ)，以及 [官方博客 More Flexible Undo Tablespace Management](https://mysqlserverteam.com/mysql-8-0-2-more-flexible-undo-tablespace-management/), [官方博客 CREATE UNDO TABLESPACE](https://mysqlserverteam.com/new-in-mysql-8-0-14-create-undo-tablespace/)

#### SQL语句

1. 在安装实例时，会默认创建两个undo tablespace，可以使用如下语句查看：

`SELECT * FROM INFORMATION_SCHEMA.INNODB_TABLESPACES WHERE ROW_FORMAT = 'Undo';

SHOW GLOBAL STATUS LIKE '%UNDO_TABLESPACE%';
`

1. 可以通过如下语句来创建undo tablespace， 文件后缀必须以ibu结尾，新创建的tablespace为active状态，在创建undo tablespace时，可以使用绝对路径，也可以放在实例配置配置的undo目录下。

```
CREATE UNDO TABLESPACE myundo ADD DATAFILE 'myundo.ibu';

```

1. 如果不想使用某个undo tablespace了，可以将其设置为inactive状态，但需要保证至少有两个active的undo tablespace；这个原因是，当有一个tablespace被truncate时，还有一个tablespace可用。当被设置了inactive后事务就不会从中分配回滚段。

```
ALTER UNDO TABLESPACE myundo SET INACTIVE;

```

1. 在删除一个undo tablespace之前，需要先将其设置为inactive。

```
DROP UNDO TABLESPACE myundo;

```

### 具体实现

1. undo tablespace的创建。

 server层接口类：Sql_cmd_create_undo_tablespace
2. 为undo tablespace预留的space id。
 
 s_min_undo_space_id = 0xFFFFFFF0UL - 127 * 512
3. s_max_undo_space_id = 0xFFFFFFF0UL - 1

 innodb_create_undo_tablespace为创建undo tablespace的函数，具体流程如下：
 1. 先调用undo::get_next_available_space_num()分配一个空闲可用的space id。
2. 调用srv_undo_tablespace_create创建undo tablespace。
3. 提交变更，并把此tablespace设置为active状态。

 undo tablespace的修改

 1. server层接口类：`Sql_cmd_alter_undo_tablespace`
2. 在崩溃恢复dd后，需要调用apply_dd_undo_state将undo tablespace状态更新到内存。
3. innodb的接口是innodb_alter_undo_tablespace

 `innodb_alter_undo_tablespace()
|
->innodb_alter_undo_tablespace_active（）
|
->innodb_alter_undo_tablespace_inactive()
` 

 innodb_alter_undo_tablespace_active流程：
 
 设置dd为active。
4. 调用undo space的alter_active，如果当前tablespace没有被标记为trcaunte，则把回滚段设置为active。

 innodb_alter_undo_tablespace_inactive流程：
 1. 如果undo space为空，直接返回。
2. 判断除此tablespace之外，还有没有两个活跃的tablespace，如果没有报错返回。
3. 设置dd为inactive。
4. 设置truncate frequency为1并唤醒purge线程, 这样purge线程会更频繁的去做purge操作，加快undo space的回收。

 undo tablespace的删除

 1. Server层接口类：`Sql_cmd_drop_undo_tablespace`
2. 入口函数：innodb_drop_undo_tablespace
 
 首先判断此tablespace是否可见，以及是否是undo tablespace。
3. 如果当前tablepace格式小于2，活跃，或者正在truncate，都不能进行删除。
4. invalidate buffer pool中该tablespace的page
5. 写一条删除的ddl log，来删除这个tablespace。

 undo tablespace的truncate

 1. 由purge线程发起，入口函数:trx_purge_truncate_marked_undo()
2. 具体流程如下：

 获取MDL锁，防止space被alter/drop。
3. 调用trx_purge_truncate_marked_undo_low()来truncate，其流程如下：
 
 调用trx_undo_truncate_tablespace()来truncate，其流程如下：
 
 先用 undo::use_next_space_id来获取一个新的space id。
4. 调用fil_replace_tablespace，用新的space id来替换掉旧的space id。fil_replace_tablespace流程如下：
 
 先调用fil_delete_tablespace删除旧的tablespace。
5. 然后用之前的file_name创建新的文件。
6. 然后调用fil_space_create 创建新的tablespace， 并且把新tablespace 与文件联系起来。

 重新初始化回滚段与内存信息。

 释放MDL锁，完成操作。

 这里使用新的space id的原因是删除重建的过程中没有做checkpoint，这时如果crash，那么可能就会存在redo修改已经不存在的page。

### 小结

InndoDB的undo log从5.6开始可以存储到单独的tablespace文件中。到5.7版本支持了在线undo文件truncate，解决了undo膨胀问题。而到了8.0，对undo tablespace做了进一步的优化，每个undo tablespace可以有128个回滚段，以此来减少事务使用回滚段时的锁冲突；可以在线动态增删undo tablespace，使得undo tablespace管理更加灵活。总体来看undo tablespace是朝着更加灵活的方向发展，以后会慢慢废弃掉通过配置文件配置的方式。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)