# MySQL · 捉虫动态· 设置 gtid_purged 破坏AUTO_POSITION复制协议

**Date:** 2015/01
**Source:** http://mysql.taobao.org/monthly/2015/01/08/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 01
 ](/monthly/2015/01)

 * 当期文章

 MySQL · 性能优化· Group Commit优化
* MySQL · 新增特性· DDL fast fail
* MySQL · 性能优化· 启用GTID场景的性能问题及优化
* MySQL · 捉虫动态· InnoDB自增列重复值问题
* MySQL · 优化改进· 复制性能改进过程
* MySQL · 谈古论今· key分区算法演变分析
* MySQL · 捉虫动态· mysql client crash一例
* MySQL · 捉虫动态· 设置 gtid_purged 破坏AUTO_POSITION复制协议
* MySQL · 捉虫动态· replicate filter 和 GTID 一起使用的问题
* TokuDB·特性分析· Optimize Table

 ## MySQL · 捉虫动态· 设置 gtid_purged 破坏AUTO_POSITION复制协议 
 Author: 

 **bug描述**

Oracle 最新发布的版本 5.6.22 中有这样一个关于GTID的bugfix，在主备场景下，如果我们在主库上 SET GLOBAL GTID_PURGED = "some_gtid_set"，并且 some_gtid_set 中包含了备库还没复制的事务，这个时候如果备库接上主库的话，预期结果是主库返回错误，IO线程挂掉的，但是实际上，在这种场景下主库并不报错，只是默默的把自己 binlog 中包含的gtid事务发给备库。这个bug的造成的结果是看起来复制正常，没有错误，但实际上备库已经丢事务了，主备很可能就不一致了。

**背景知识**

一） binlog GTID事件

binlog 中记录的和GTID相关的事件主要有2种，Previous_gtids_log_event 和 Gtid_log_event，前者表示之前的binlog中包含的gtid的集合，后者就是一个gtid，对应一个事务。一个 binlog 文件中只有一个 Previous_gtids_log_event，放在开头，有多个 Gtid_log_event，如下面所示

`Previous_gtids_log_event // 此 binlog 之前的所有binlog文件包含的gtid集合

Gtid_log_event // 单个gtid event
Transaction
Gtid_log_event
Transaction
.
.
.
Gtid_log_event
Transaction
`

二） 备库发送GTID集合给主库

我们知道备库的复制线程是分IO线程和SQL线程2种的，IO线程通过GTID协议或者文件位置协议拉取主库的binlog，然后记录在自己的relay log中；SQL线程通过执行realy log中的事件，把其中的操作都自己做一遍，记入本地binlog。在GTID协议下，备库向主库发送拉取请求的时候，会告知主库自己已经有的所有的GTID的集合，Retrieved_Gtid_Set + Executed_Gtid_Set，前者对应 realy log 中所有的gtid集合，表示已经拉取过的，后者对应binlog中记录有的，表示已经执行过的；主库在收到这2个总集合后，会扫描自己的binlog，找到合适的binlog然后开始发送。

三）主库如何找到要发送给备库的第一个binlog

主库将备库发送过来的总合集记为 slave_gtid_executed，然后调用 find_first_log_not_in_gtid_set(slave_gtid_executed)，这个函数的目的是从最新到最老扫描binlog文件，找到第一个含有不存在 slave_gtid_executed 这个集合的gtid的binlog。在这个扫描过程中并不需要从头到尾读binlog中所有的gtid，只需要读出 Previous_gtids_log_event ，如果Previous_gtids_log_event 不是 slave_gtid_executed的子集，就继续向前找binlog，直到找到为止。

这个查找过程总会停止的，停止条件如下：

1. 找到了这样的binlog，其Previous_gtids_log_event 是slave_gtid_executed子集
2. 在往前读binlog的时候，发现没有binlog文件了（如被purge了），但是还没找到满足条件的Previous_gtids_log_event，这个时候主库报错
3. 一直往前找，发现Previous_gtids_log_event 是空集

在条件2下，报错信息是这样的

Got fatal error 1236 from master when reading data from binary log: 'The slave is connecting using CHANGE MASTER TO MASTER_AUTO_POSITION = 1, but the master has purged binary logs containing GTIDs that the slave requires.

其实上面的条件3是条件1的特殊情况，这个bugfix针对的场景就是条件3这种，但并不是所有的符合条件3的场景都会触发这个bug，下面就分析下什么情况下才会触发bug。

**bug 分析**

假设有这样的场景，我们要用已经有MySQL实例的备份重新做一对主备实例，不管是用 xtrabackup 这种物理备份工具或者mysqldump这种逻辑备份工具，都会有2步操作，

1. 导入数据
2. SET GLOBAL GTID_PURGED ="xxxx"

步骤2是为了保证GTID的完备性，因为新实例已经导入了数据，就需要把生成这些数据的事务对应的GTID集合也设置进来。

正常的操作是主备都要做这2步的，如果我们只在主库上做了这2步，备库什么也不做，然后就直接用 GTID 协议把备库连上来，按照我们的预期这个时候是应该出错的，主备不一致，并且主库的binlog中没东西，应该报之前停止条件2报的错。但是令人大跌眼镜的是主库不报错，复制看起来是完全正常的。

为啥会这样呢，SET GLOBAL GTID_PURGED 操作会调用 mysql_bin_log.rotate_and_purge切换到一个新的binlog，并把这个GTID_PURGED 集合记入新生成的binlog的Previous_gtids_log_event，假设原有的binlog为A，新生成的为B，主库刚启动，所以A就是主库的第一个binlog，它之前啥也没有，A的Previous_gtids_log_event就是空集，并且A中也不包含任何GTID事件，否则SET GLOBAL GTID_PURGED是做不了的。按照之前的扫描逻辑，扫到A是肯定会停下来的，并且不报错。

**bug 修复**

官方的修复就是在主库扫描查找binlog之前，判断一下 gtid_purged 集合不是不比slave_gtid_executed大，如果是就报错，错误信息和条件2一样 Got fatal error 1236 from master when reading data from binary log: 'The slave is connecting using CHANGE MASTER TO MASTER_AUTO_POSITION = 1, but the master has purged binary logs containing GTIDs that the slave requires.

详细的bugfix请看revno: [6211](http://bazaar.launchpad.net/~mysql/mysql-server/5.6/revision/6211)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)