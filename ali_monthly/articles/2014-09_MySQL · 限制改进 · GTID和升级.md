# MySQL · 限制改进 · GTID和升级

**Date:** 2014/09
**Source:** http://mysql.taobao.org/monthly/2014/09/02/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2014 / 09
 ](/monthly/2014/09)

 * 当期文章

 MySQL · 捉虫动态 · GTID 和 DELAYED
* MySQL · 限制改进 · GTID和升级
* MySQL · 捉虫动态 · GTID 和 binlog_checksum
* MySQL · 引擎差异·create_time in status
* MySQL · 参数故事 · thread_concurrency
* MySQL · 捉虫动态 · auto_increment
* MariaDB · 性能优化 · Extended Keys
* MariaDB · 主备复制 · CREATE OR REPLACE
* TokuDB · 参数故事 · 数据安全和性能
* TokuDB · HA方案 · TokuDB热备

 ## MySQL · 限制改进 · GTID和升级 
 Author: 

 **GTID 资料**

MySQL 5.6 引入了global transaction identifiers (GTIDs，全局事务ID)的特性，这一特性是用来解决主从复制(replication)场景下的一些问题，GTID 只存在于 binlog 中，数据库中是没有的。

要了解 GTID 的话，[官方文档](http://dev.mysql.com/doc/refman/5.6/en/replication-gtids.html)是一定要看的，另外再推荐推荐三篇 Oracle 同学写的文章(需爬墙)：

1. Failover and Flexible Replication Topologies in MySQL 5.6
2. Advanced use of Global Transaction Identifiers
3. Flexible Fail-over Policies Using MySQL and Global Transaction Identifiers
4. 有兴趣的话也可以看下 GTID 的 worklog WL#3548

**升级遇到的问题**

GTID 能很好的解决 failover 问题，做到主库切换自动化，减轻 DBA 同学的负担，但是这个前提是所有的 MySQL 实例都是 5.6，如果线上实例是 5.5 的，必须全部升到 5.6 才行，而目前官方并没有提供平滑的 5.5 升级到 5.6 GTID 的方式，中间必须要有一个实例重启过程，这是由 GTID 目前的实现方式决定的：

* 限制1. GTID 模式实例和非GTID模式实例是不能进行复制的，要求非常严格，一刀切，要么都是GTID，要么都不是
* 限制2. gtid_mode 是只读的，要改变状态必须1)关闭实例、2)修改配置文件、3) 重启实例

在这种条件要求下，我们来看下线上实例从 5.5 升级到 5.6 会有什么问题，为了保证业务不中断，升级过程一直要有实例对外提供服务，因此升级方式是创建一个新的 5.6 实例，从5.5同步数据，然后业务切换到 5.6。

为了描述方便，做如下假设：

实例A
 5.5 版本的，目前业务用的数据库
实例B
 5.6 版本的，数据迁移的目标

**迁移步骤如下**

1. 用热备份工具如 Percona XtraBackup 将 A 数据备份然后导入到B
2. B 用非 GTID 模式和 5.5 同步数据，这时用的是传统的基于文件位置的复制
3. B和A同步的差不多的时候，在 A 上设 read_only，等 B 同步完成，假设同步完后时间点为 t1
4. 关闭 B，修改参数开启GTID，重启B
5. 将业务的数据操作指向B，假设这个时间点为 t2
6. B 开始提供服务，迁移完毕

在上面的步骤中，t1 到 t2 的时间段内相当于数据库服务不可用，整个数据库停掉重启，这对线上业务来说是不可接受的，上面是用单个实例A和实例B说明问题，同样可以扩展到集群A和集群B。

gtid_mode 的取值范围除了 OFF 和 ON 这两个值外，还有 UPGRADE_STEP_1和UPGRADE_STEP_2，目前后2者并不支持，不过从名字上看应该是为了升级预留的，但是目前并没有好的升级方式。

**解决方案**

如果要想做到不重启升级，必须打破之前提到限制条件，booking.com 提供了一种方案，就是打破限制1，创造出一种特殊的模式，使实例处于GTID模式下仍然可以和非GTID的实例进行复制。 详细的方案介绍在这里 [MySQL 5.6 GTIDs: Evaluation and Online Migration](http://blog.booking.com/mysql-5.6-gtids-evaluation-and-online-migration.html)，代码的改动很小，就是让 sql/rpl_slave.cc 中的下面这段检查代码无效：

`if (mi->master_gtid_mode > gtid_mode + 1 ||
gtid_mode > mi->master_gtid_mode + 1)
{
mi->report(ERROR_LEVEL, ER_SLAVE_FATAL_ERROR,
"The slave IO thread stops because the master has "
"@@GLOBAL.GTID_MODE %s and this server has "
"@@GLOBAL.GTID_MODE %s",
gtid_mode_names[mi->master_gtid_mode],
gtid_mode_names[gtid_mode]);
DBUG_RETURN(1);
}
`
之前的升级方式是 A->B 这种拓扑，现在变为 A->C->B

实例A
 5.5 版本的，目前业务用的数据库
实例C
 5.6 版本的，一种中间状态实例，既可以和非GTID通信，又可以和GTID通信
实例B
 5.6 版本的，数据迁移的目标

**迁移步骤如下：***

用热备份工具如 Percona XtraBackup 将 A 数据备份然后导入到B
建立 A->C->B 这种复制关系，其中 A->C 之间是文件位置协议，C->B 之间是 GTID 协议
B、C和A同步的差不多的时候，在 A 上设 read_only，等B同步完成
将业务的数据操作指向B
B 开始提供服务，迁移完毕
这里为了和之前迁移目标一致，多用了一个实例C，其实这时候可以把B给去掉，还是2个实例。可以看到，引入了C后，升级过程中并没有实例重启过程，只有一个短暂的只读时间段，这个是无法避免的，即使不用GTID，也会有这个过程。

目前RDS实例升级到5.6也是用这种方式。如果是集群到集群的话，要注意一点，处于中间状态的实例C最好只有一个，因为这种实例相当于一个GTID转换器，将A中没有 GTID 的 binlog 转成包含 GTID 的 binlog，然后传给B，如果有多个实例C的话，A中同一个binlog 中的事务会转换出不同的GTID，这与 GTID 和事务一一对应的根本原则相矛盾，复制会出问题。当然，如果能保证经过不同的C的binlog事务不会重复的话就可以有多个C。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)