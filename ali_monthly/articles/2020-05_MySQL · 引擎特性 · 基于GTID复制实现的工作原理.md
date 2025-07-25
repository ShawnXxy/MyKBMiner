# MySQL · 引擎特性 · 基于GTID复制实现的工作原理

**Date:** 2020/05
**Source:** http://mysql.taobao.org/monthly/2020/05/09/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2020 / 05
 ](/monthly/2020/05)

 * 当期文章

 Database · 技术方向 · 下一代云原生数据库详解
* Database · 理论基础 · 高性能B-tree索引
* Database · 理论基础 · ARIES/IM (一)
* AliSQL · 引擎特性 · Fast Query Cache 介绍
* MySQL · 源码分析 · 8.0 · DDL的那些事
* MySQL · 内核分析 · InnoDB Buffer Pool 并发控制
* MySQL · 源码分析 · 内部 XA 和组提交
* MySQL · 插件分析 · Connection Control
* MySQL · 引擎特性 · 基于GTID复制实现的工作原理

 ## MySQL · 引擎特性 · 基于GTID复制实现的工作原理 
 Author: 刘歆 

 GTID (Global Transaction IDentifier) 是全局事务标识。它具有全局唯一性，一个事务对应一个GTID。唯一性不仅限于主服务器，GTID在所有的从服务器上也是唯一的。一个GTID在一个服务器上只执行一次，从而避免重复执行导致数据混乱或主从不一致。

在传统的复制里面，当发生故障需要主从切换时，服务器需要找到binlog和pos点，然后将其设定为新的主节点开启复制。相对来说比较麻烦，也容易出错。在MySQL 5.6里面，MySQL会通过内部机制自动匹配GTID断点，不再寻找binlog和pos点。我们只需要知道主节点的ip，端口，以及账号密码就可以自动复制。

## GTID的组成部分：

GDIT由两部分组成：GTID = source_id:transaction_id。 其中source_id是产生GTID的服务器，即是server_uuid，在第一次启动时生成（sql/mysqld.cc: generate_server_uuid()），并保存到DATADIR/auto.cnf文件里。transaction_id是序列号（sequence number），在每台MySQL服务器上都是从1开始自增长的顺序号，是事务的唯一标识。例如：3E11FA47-71CA-11E1-9E33-C80AA9429562:23 
GTID 的集合是一组GTIDs，可以用source_id+transaction_id范围表示，例如：3E11FA47-71CA-11E1-9E33-C80AA9429562:1-5 
复杂一点的：如果这组 GTIDs 来自不同的 source_id，各组 source_id 之间用逗号分隔；如果事务序号有多个范围区间，各组范围之间用冒号分隔，例如：3E11FA47-71CA-11E1-9E33-C80AA9429562:23,3E11FA47-71CA-11E1-9E33-C80AA9429562:1-5

## GTID如何产生：

GTID的生成受GTID_NEXT控制。

在主服务器上，GTID_NEXT默认值是AUTOMATIC，即在每次事务提交时自动生成GTID。它从当前已执行的GTID集合（即gtid_executed）中，找一个大于0的未使用的最小值作为下个事务GTID。同时在实际的更新事务记录之前，将GTID写入到binlog（set GTID_NEXT记录）。 
在Slave上，从binlog先读取到主库的GTID(即get GTID_NEXT记录)，而后执行的事务采用该GTID。

## GTID的工作原理：

GTID在所有主从服务器上都是不重复的。所以所有在从服务器上执行的事务都可以在bnlog找到。一旦一个事务提交了，与拥有相同GTID的后续事务都会被忽略。这样可以保证从服务器不会重复执行同一件事务。

当使用GTID时，从服务器不需要保留任何非本地数据。使用数据都可以从replicate data stream。从DBA和开发者的角度看，从服务器无保留file-offset pairs以决定如何处理主从服务器间的数据流。

## GTID的生成和使用由以下几步组成：
* 主服务器更新数据时，会在事务前产生GTID，一同记录到binlog日志中。
* binlog传送到从服务器后，被写入到本地的relay log中。从服务器读取GTID，并将其设定为自己的GTID（GTID_NEXT系统）。
* sql线程从relay log中获取GTID，然后对比从服务器端的binlog是否有记录。
* 如果有记录，说明该GTID的事务已经执行，从服务器会忽略。
* 如果没有记录，从服务器就会从relay log中执行该GTID的事务，并记录到binlog。

## GTID相关的变量

### GTID_NEXT:

SESSION级别变量，表示下一个将被使用的GTID。

* Scope : Session
* Dynamic : Yes
* Type : Enumeration
* Default Value : AUTOMATIC
* Valid Values :

`-- AUTOMATIC: 使用自动产生的下一个GTID。
-- ANONYMOUS: 事务没有GTID, 只使用 file and position 作为标识。
-- UUID:NUMBER：GTID in UUID:NUMBER format.
`

### GTID_MODE:

Log 是否使用GTID或使用anonymous。anonymous transaction用binlog file 和position来标识事务。

* Scope : Global
* Dynamic : Yes
* Type : Enumeration
* Default Value : OFF
* Valid Values
 `-- OFF：新的和复制事务都使用anonymous。
-- OFF_PERMISSIVE：新的事务都使用anonymous，而复制事务可以使用GTID或anonymous。
-- ON_PERMISSIVE：复制事务都使用anonymous，而新事务可以使用GTID或anonymous。
-- ON: 新的和复制事务都使用GTID
`

### GTID_EXECUTED:

包含已经在该实例上执行过的事务； 执行RESET MASTER 会将该变量置空; 我们还可以通过设置GTID_NEXT在执行一个空事务，来影响GTID_EXECUTED。使用 SHOW MASTER STATUS and SHOW SLAVE STATUS，其中Executed_Gtid_Set会显示GTID_EXECUTED里的GTIDs。5.7.7之前，GTID_EXECUTED可以是seesion变量。它包含当前session写入缓存的一组事务。

* Scope : Global, Session
* Dynamic : No
* Type : String

### GTID_PURGED:

已经被删除了binlog的事务，它是GTID_EXECUTED的子集，只有在GTID_EXECUTED为空时才能设置该变量，修改GTID_PURGED会同时更新GTID_EXECUTED和GTID_PURGED的值。

* Scope : Global
* Dynamic : Yes
* Type : String

### GTID_OWNED:

表示正在执行的事务的GTID以及其对应的线程ID。

* Scope : Global, Session
* Dynamic : No
* Type : String

如果GDIT_OWNED是全局变量，它包含所有当前服务器上正在使用的GTIDs和使用它们的线程IDs。这个变量主要用于多线程从服务器复制，从而可以查看一个事务是否已经被另一个线程处理。这个线程会拥有所处理事务的ownership。@@global.grid_owned会显示出GTID和它的owner。当事务处理完成，线程会释放ownership.
如果GDIT_OWNED是session变量，它包含一个seesion正在使用的GTID。这个变量对测试和debug会很有帮助。

Reference:

[https://dev.mysql.com/doc/refman/5.6/en/replication-gtids-concepts.html](https://dev.mysql.com/doc/refman/5.6/en/replication-gtids-concepts.html)
[https://dev.mysql.com/doc/refman/5.6/en/replication-options-gtids.html](https://dev.mysql.com/doc/refman/5.6/en/replication-options-gtids.html)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)