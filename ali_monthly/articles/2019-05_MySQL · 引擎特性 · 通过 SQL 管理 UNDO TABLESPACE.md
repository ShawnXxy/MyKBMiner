# MySQL · 引擎特性 · 通过 SQL 管理 UNDO TABLESPACE

**Date:** 2019/05
**Source:** http://mysql.taobao.org/monthly/2019/05/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2019 / 05
 ](/monthly/2019/05)

 * 当期文章

 MSSQL · 最佳实践 · 挑战云计算安全的存储过程
* MySQL · 源码分析 · 聚合函数（Aggregate Function）的实现过程
* PgSQL · 最佳实践 · RDS for PostgreSQL 的逻辑订阅
* MySQL · 引擎特性 · 通过 SQL 管理 UNDO TABLESPACE
* MySQL · 最佳实践 · 通过Resource Group来控制线程计算资源
* MySQL · 引擎特性 · Skip Scan Range
* MongoDB · 应用案例 · killOp 案例详解
* MySQL · 源码分析 · LinkBuf设计与实现
* PgSQL · 应用案例 · PostgreSQL KPI分解，目标设定之 - 等比数列
* PgSQL · 应用案例 · PostgreSQL KPI 预测例子

 ## MySQL · 引擎特性 · 通过 SQL 管理 UNDO TABLESPACE 
 Author: yinfeng 

 ## 前言
InnoDB的undo log从5.6版本开始可以存储到单独的tablespace文件中，在5.7版本支持了在线undo文件truncate，解决了长期以来的undo膨胀问题。而到了8.0版本，对Undo tablespace做了进一步的优化：在新版本中，我们可以拥有更多的回滚段(每个Undo tablespace可以有128个回滚段，而在之前的版本中所有tablespace的回滚段不允许超过128个)，减少了由于事务公用回滚段产生的锁冲突；可以在线动态的增删undo tablespace，使得undo的管理更加灵活。

在最近release的8.0.14版本中，开始支持SQL接口来创建，修改和删除 （undo space的管理不记录binlog）。可以预见未来将逐步废弃根据配置innodb_undo_tablespaces来创建undo tablespace, 通过SQL接口来创建undo tablespace将是唯一的接口。实际上在最新版本中已经将参数innodb_undo_tablespaces标记为deprecated状态，用户应尽量避免依赖该参数。

## SQL语句

### implict undo space
在安装实例时，会默认创建两个undo tablespace:

` mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_TABLESPACES WHERE ROW_FORMAT = 'Undo'\G
*************************** 1. row ***************************
SPACE: 4294967279
NAME: innodb_undo_001
FLAG: 0
ROW_FORMAT: Undo
PAGE_SIZE: 16384
ZIP_PAGE_SIZE: 0
SPACE_TYPE: Undo
FS_BLOCK_SIZE: 0
FILE_SIZE: 0
ALLOCATED_SIZE: 0
SERVER_VERSION: 8.0.15
SPACE_VERSION: 1
ENCRYPTION: N
STATE: active
*************************** 2. row ***************************
SPACE: 4294967278
NAME: innodb_undo_002
FLAG: 0
ROW_FORMAT: Undo
PAGE_SIZE: 16384
ZIP_PAGE_SIZE: 0
SPACE_TYPE: Undo
FS_BLOCK_SIZE: 0
FILE_SIZE: 0
ALLOCATED_SIZE: 0
SERVER_VERSION: 8.0.15
SPACE_VERSION: 1
ENCRYPTION: N
STATE: active
2 rows in set (0.00 sec)

mysql> SHOW GLOBAL STATUS LIKE '%UNDO_TABLESPACE%';
+----------------------------------+-------+
| Variable_name | Value |
+----------------------------------+-------+
| Innodb_undo_tablespaces_total | 2 |
| Innodb_undo_tablespaces_implicit | 2 |
| Innodb_undo_tablespaces_explicit | 0 |
| Innodb_undo_tablespaces_active | 2 |
+----------------------------------+-------+
4 rows in set (0.00 sec)
`

### 创建新的undo space
你可以通过如下语句来创建独立的undo tablespace， 文件后缀必须以ibu结尾。新创建的tablespace为active状态

` mysql> CREATE UNDO TABLESPACE myundo ADD DATAFILE 'myundo.ibd';
 ERROR 3121 (HY000): The ADD DATAFILE filepath must end with '.ibu'.

 mysql> CREATE UNDO TABLESPACE myundo ADD DATAFILE 'myundo.ibu';
 Query OK, 0 rows affected (0.26 sec)

 mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_TABLESPACES WHERE ROW_FORMAT = 'Undo' and NAME = 'myundo'\G
 *************************** 1. row ***************************
 SPACE: 4294967277
 NAME: myundo
 FLAG: 0
 ROW_FORMAT: Undo
 PAGE_SIZE: 16384
 ZIP_PAGE_SIZE: 0
 SPACE_TYPE: Undo
 FS_BLOCK_SIZE: 0
 FILE_SIZE: 0
 ALLOCATED_SIZE: 0
 SERVER_VERSION: 8.0.15
 SPACE_VERSION: 1
 ENCRYPTION: N
 STATE: active --> 此时状态为active
1 row in set (0.01 sec)
`

在创建undo space时，你可以使用绝对路径，也可以放在实例配置的undo目录下，但要注意一点:在崩溃恢复前undo space必须要能够被发现并打开，但这时候Innodb data dictionary还是处于不可用的状态，我们无法从其中获取准确的文件位置，只有–datadir, –innodb-home-directory, –innodb-undo-directory 和 –innodb-directories会被扫描掉，如果你放在其他地方，就可能造成找不到该tablespace, 导致实例数据不一致。

相关代码：

* Server层接口类：Sql_cmd_create_undo_tablespace
* 为undo tablespace预留的space id (但最多依然是127个undo tablespace, 每个space number会给一个范围内的space id, 默认512个id):
 
 s_min_undo_space_id = 0xFFFFFFF0UL - 127 * 512
* s_max_undo_space_id = 0xFFFFFFF0UL - 1

 InnoDB入口函数: innodb_create_undo_tablespace
 * 获取下一个可用的space id: undo::get_next_available_space_num(), 先拿到空闲的space number，再分配一个可用的space id
* srv_undo_tablespace_create: 创建undo space, 初始化回滚段并加入到全局事务系统中
* 提交变更，持久化tablespace信息后，将其设置为active状态，此后事务可以从其中分配到回滚段

### 设置inactive
如果你不想使用某个Undo tablespace，可以将其设置为inactive状态， 但需要保证至少有连个active的undo tablespace, 这个限制的原因是：当一个undo tablespace正在被truncate时，至少有一个是可用的。

当被设置为Inactive状态之后，事务就不会从其中分配回滚段。

`mysql> ALTER UNDO TABLESPACE myundo SET INACTIVE;
Query OK, 0 rows affected (0.01 sec)
`
相关代码：

* server层接口类：Sql_cmd_alter_undo_tablespace
* 在崩溃恢复data dicitonary提供服务后，需要将undo space状态更新到内存（apply_dd_undo_state()）
* innodb_alter_undo_tablespace–> innodb_alter_undo_tablespace_active
 
 设置Undo space 为active状态，并修改dd元数据

 innodb_alter_undo_tablespace –> innodb_alter_undo_tablespace_inactive
 * 当undo space状态为empty时，直接返回
* 当undo space状态为active时，需要确保至少两个active的undo space才允许操作，否则返回错误
* 设置dd state为inactive，并修改回滚段状态
* 设置truncate frequency为1并唤醒purge线程, 这样purge线程会更频繁的去做purge操作，加快undo space的回收

### 删除undo space
在删除一个undo tablespace之前，首先要把undo tablespace设置为inactive状态

` mysql> DROP UNDO TABLESPACE myundo;
 ERROR 1529 (HY000): Failed to drop UNDO TABLESPACE myundo

 mysql> ALTER UNDO TABLESPACE myundo SET INACTIVE;
 Query OK, 0 rows affected (0.01 sec)

 mysql> SELECT * FROM INFORMATION_SCHEMA.INNODB_TABLESPACES WHERE ROW_FORMAT = 'Undo' and Name = 'myundo'\G
 *************************** 1. row ***************************
 SPACE: 4294967150
 NAME: myundo
 FLAG: 0
 ROW_FORMAT: Undo
 PAGE_SIZE: 16384
 ZIP_PAGE_SIZE: 0
 SPACE_TYPE: Undo
 FS_BLOCK_SIZE: 0
 FILE_SIZE: 0
 ALLOCATED_SIZE: 0
 SERVER_VERSION: 8.0.15
 SPACE_VERSION: 1
 ENCRYPTION: N
 STATE: empty --> 此时undo space内没有任何Undo log， 已经是empty可删除状态
 1 row in set (0.00 sec)

 mysql> DROP UNDO TABLESPACE myundo;
 Query OK, 0 rows affected (0.02 sec)
`

即使状态为inactive的，但要保证如下几点才能被删除：

* 没有任何事务需要看到其中的老版本数据，也就是说所有在该事务之前开启的read view必须全部关闭
* 所有使用该undo tablespace的事务必须全部提交或回滚掉
* purge线程需要将其中的Undo log全部清理掉

如果undo tablespace非空，在drop时，会返回错误码HA_ERR_TABLESPACE_IS_NOT_EMPTY. 所以在设置为inactive到真正可以删除可能存在时间差，我们可以通过监控INFORMATION_SCHEMA.INNODB_TABLESPACES中的undo space状态是否为empty来判定是否可以删除。 Note:系统创建的Undo space不允许被删除

相关代码:

* Server层接口类：Sql_cmd_drop_undo_tablespace
* InnoDB 入口函数： innodb_drop_undo_tablespace
 
 当undo space状态不为emtpy时或者是系统创建的Undo space时，不允许删除
* invalidate buffer pool中该space的page
* 从内存中删除，记录ddl log
* 事务提交后，执行post ddl (Log_DDL::replay_delete_space_log)
 
 真正物理删除文件
* 标记对应的space num为未使用状态

### undo truncation
当参数innodb_undo_log_truncate打开时，所有隐式和显式创建的Undo tablespace都会在满足一定条件时被purge线程truncate掉. 当参数关闭时，则只有将Undo tablespace设置为Inactive状态时才会去truncate tablespace。 因此如果你想自己控制undo truncation, 可以关闭参数，在监控undo tablespace的大小，通过SET INACTIVE触发truncation, 再通过SET ACTIVE激活undo space。

相关代码：

* 由purge线程发起，入口函数:trx_purge_truncate_marked_undo()
* 需要获取MDL锁，来保护space不被alter/drop
* 通过flush_observer flush当前space的page
* trx_purge_truncate_marked_undo_low
 
 trx_undo_truncate_tablespace:
 
 为当前space分配一个新的space id: undo::use_next_space_id(space_num)
* fil_replace_tablespace: 删除当前undo space，重建文件并设置为新的space id
* 重新初始化回滚段和内存信息
* 根据新的space id，将所有变更刷到磁盘
* 如果是用户创建的undo space，将状态设置为empty，否则设置为active状态
* 更新DD

为何需要新的space id ? 这是因为在删除重建文件的过程中我们没有做checkpoint,这时候如果crash掉，有些redo log可能需要修改一些已经不存在的page，导致崩溃恢复时候([ref: bug93170](https://bugs.mysql.com/bug.php?spm=a2c4e.11153940.blogcont689955.11.7be727f7BMhh8t&id=93170))

## Reference
[1. WL#9508: InnoDB: Support CREATE/ALTER/DROP UNDO TABLESPACE](https://dev.mysql.com/worklog/task/?spm=a2c4e.11153940.blogcont689955.12.7be727f7BMhh8t&id=9508)
[2. WL#9507: InnoDB: Make the number of undo tablespaces and rollback segments dynamic](https://dev.mysql.com/worklog/task/?spm=a2c4e.11153940.blogcont689955.13.7be727f7BMhh8t&id=9507)
[3. 主要代码](https://github.com/mysql/mysql-server/commit/d285c74c714a9f464267d1ea1ebd94e215fda0a5?spm=a2c4e.11153940.blogcont689955.14.7be727f7BMhh8t)
[4. 官方文档](https://dev.mysql.com/doc/refman/8.0/en/innodb-undo-tablespaces.html?spm=a2c4e.11153940.blogcont689955.15.7be727f7BMhh8t)
[5. MySQL8.0 · 引擎特性 · 关于undo表空间的一些新变化](https://yq.aliyun.com/articles/341036?spm=a2c4e.11153940.blogcont689955.16.7be727f7BMhh8t)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)