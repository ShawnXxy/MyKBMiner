# MySQL · 答疑解惑 · GTID不一致分析

**Date:** 2016/01
**Source:** http://mysql.taobao.org/monthly/2016/01/08/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 01
 ](/monthly/2016/01)

 * 当期文章

 MySQL · 引擎特性 · InnoDB 事务锁系统简介
* GPDB   · 特性分析· GreenPlum Primary/Mirror 同步机制
* MySQL · 专家投稿 · MySQL5.7 的 JSON 实现
* MySQL · 特性分析 · 优化器 MRR & BKA
* MySQL · 答疑解惑 · 物理备份死锁分析
* MySQL · TokuDB · Cachetable 的工作线程和线程池
* MySQL · 特性分析 · drop table的优化
* MySQL · 答疑解惑 · GTID不一致分析
* PgSQL · 特性分析 · Plan Hint
* MariaDB · 社区动态 · MariaDB on Power8 (下)

 ## MySQL · 答疑解惑 · GTID不一致分析 
 Author: 济天 

 ## 背景

server A,B 为双主结构，对于 server A 当gtid_next设置为AUTOMATIC时，A上执行的事务在binlog刷盘时递增获取事务的gtid，从而保证了在binlog中属于A的gtid是连续递增的。

A的binlog在B应用时，B会通过 Executed_Gtid_Set 来记录A的binlog在B的执行情况。而A的binlog中gtid是连续的，从而

1. B未开启并行复制，B依次应用binlog，Executed_Gtid_Set中B的gtid集合应该是连续的，如`A:1-100`；
2. B开启并行复制此时，B并发应用binlog，Executed_Gtid_Set中B的gtid集合末端可能会出现不连续的情况，如`A:1-92:94-96:98-100`，
但是如果在A停止写入，B和A完全同步的情况下，Executed_Gtid_Set中B的gtid集合应该是连续的。

以下分析均基于双主结构。

## GTID不一致产生原因

**并行复制**

前面提到，并行复制时，Executed_Gtid_Set 尾部可能出现正常的空洞。这种不一致可以忽略，最终会一致的。

**change master导致gtid丢失**

change master重新指定binlog的拉取位置会导致主备gtid不一致。

考虑如下情况

主库：

`create table t1(c1 int) engine=innodb;
create view v1 as select * from t1;
`

备库：

`set sql_log_bin=0;
drop view v1;
`

主库：

`create view v1 as select * from t1;
`

执行失败但仍然会生成binlog, binlog中error_code=1050，参考之前月报[binlog event 中的 error code](http://mysql.taobao.org/monthly/2015/06/05/)。

`#160118 17:33:11 server id 2219317521 end_log_pos 172885 CRC32 0xb6f4693d Query thread_id=76714 exec_time=0 error_code=1050
SET TIMESTAMP=1453109591/*!*/;
CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`127.0.0.1` SQL SECURITY DEFINER VIEW `v1` AS select * from t1
`

备库：

`show slave status\G
Last_SQL_Error: Query caused different errors on master and slave. Error on master: message (format)='Table '%-.192s' already exists' error code=1050 ; Error on slave: actual message='no error', error code=0. Default database: 'zy'. Query: 'CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`127.0.0.1` SQL SECURITY DEFINER VIEW `v1` AS select * from t1'
`

备库create view执行成功，而与预期应出现的binlog error_code不一致导致备库复制中断。

那么如何修复此类错误呢？首先想到这个binlog事件是可以跳过的，因此尝试使用跳过空事务的方式来尝试修复。实际上，这种修复方式是不起作用的。

构造空事务来忽略相同gtid，执行start slave；后SQL线程执行过程如下

1. 首先预检查gtid是否执行，如果此gtid已执行，语句直接返回成功0;
2. 然后检查预期error_code是否为1050，而实际error_code=0，还是不一致。

因此，此类错误通过构造空事务方式无法修复。

此时就需要change master 方式指向失败事件的下一个位点。然后按位点的方式(master_auto_position=0)来拉binlog。

`stop slave;
change master to master_log_file='xxxx', master_log_pos=xxx,master_auto_position=0;
start slave;
`

至此失败已经修复，但show slave status可以看到 Executed_Gtid_Set 存在空洞，gtid已经不一致了。

此时，我们可以通过skip空事务的方式来弥补这个空洞。

下一步，我们可以选择修改为通过gtid方式来拉binlog(此步必须在弥补空洞之后）。

`stop slave;
change master to master_auto_position=1;
start slave;
`

**主机crash**

主库所在主机crash后，可能导致主库比备库少一些gtid。

binlog在写文件时先write，再sync。假设主库在write binlog之后，sync 之前，同时备库也拉取了这些未sync的binlog。此时主库宕机，主库一部分 binlog 未落盘，但这部分binlog已经传到了备库，那么备库会比主库多一些事务。因此主库重启后，重新构造 gtid_executed_set 时会比备库少一些gtid。

那些未sync的事务实际处于两阶段提交的prepare状态，重启后这些处于prepare的事务由于没有写binlog会回滚掉。

主机宕机HA切换后，新主库会比新备库多一些事务。

而实际上新主库会比新备库多一些事务应该没有影响，这些事务是用户发出了commit命令，但主机crash了，没有收到commit的回复，处于未知状态。这些未决事务可以提交也可以回滚！

对于以上情况，在binlog没有purge的情况下，结合应用我们可以根据gtid来修复主备不一致的情况，或回滚备库的修改，或者重做主库丢失的事务。

## gtid不一致的 影响

假设备库比主库少一些gtid, 而主库多出来的这些gtid已经purge了。如果用备库做的备份集来恢复出一个实例时，备库会去主库拉取缺少的那些gtid，而那些gtid已经purge了。
这就会导致臭名昭著的1236问题

`Last_IO_Errno: 1236
Last_IO_Error: Got fatal error 1236 from master when reading data from binary log: 'The slave is connecting using CHANGE MASTER TO MASTER_AUTO_POSITION = 1, but the master has purged binary logs containing GTIDs that the slave requires.'
`

根据前面的gtid不一致的分析，我们应该提前发现不一致，并及时修复，避免此类情况发生。

## 修复方法

这里总结些gtid不一致的修复方法：

1. 对于可以忽略的gtid事务，可以通过跳过gtid的方式修复；
2. 修改GTID_PURGED的方式
先reset master，再设置GTID_PURGED来修复不一致，此种方式需确保主备数据一致的情况下进行，风险较大，酌情考虑。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)