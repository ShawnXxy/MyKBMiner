# MySQL · 原理介绍 · 再议MySQL的故障恢复

**Date:** 2018/12
**Source:** http://mysql.taobao.org/monthly/2018/12/04/
**Images:** 6 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 12
 ](/monthly/2018/12)

 * 当期文章

 Database · 原理介绍 · 数据库的事务与复制
* PgSQL · 引擎特性 · PostgreSQL Hint Bits 简介
* MSSQL · 最佳实践 · 行级别安全解决方案
* MySQL · 原理介绍 · 再议MySQL的故障恢复
* POLARDB · 引擎特性 · 物理复制解读
* Redis · 原理介绍 · 利用管道优化aofrewrite
* PgSQL · 原理介绍 · PostgreSQL行锁实现
* MySQL · RocksDB · 数据的读取(二)
* PgSQL · 应用案例 · PG 11 并行计算算法，参数，强制并行度设置
* PgSQL · 应用案例 · PostgreSQL IoT，车联网 - 实时轨迹、行程实践

 ## MySQL · 原理介绍 · 再议MySQL的故障恢复 
 Author: zhilin 

 ## MySQL的事务处理—两阶段事务提交2PC
MySQL数据库的INNODB是一款支持OLTP的存储引擎，为支持MySQL的高可用，支持跨机搭建高可用数据库集群，MySQL采用了一种简单有效的机制-基于binlog的复制，binlog是binary log的简称，实际上它是一种逻辑日志，相对InnoDB引擎的物理日志，它的数据量更小，格式也更简单，更易于跨机复制，尤其是对于网络环境不是很好的情况下，更具有天然的优势。
那么MySQL是如何协调InnoDB引擎与Binlog日志之间的关系呢？MySQL采用了两阶段事务提交(Two-Phase Commit Protocol)协议，当操作完成后，首先Prepare事务，在binlog中实际上只是fake一下，不作任何事情，而是innodb层需要将prepare写入redolog中；然后执行commit事务，首先在binlog文件中写入这些操作的binlog日志，完成后再在Innodb的redolog中写入commit日志。

![pic](.img/2c26d34b486e_201812-01.png)

注意在写binlog日志时，有个参数sync_binlog来控制何时将binlog fsync到磁盘。

* 参数为0时，并不是立即fsync文件到磁盘，而是依赖于操作系统的fsync机制；
* 参数为1时，立即fsync文件到磁盘；
* 参数大于1时，则达到指定提交次数后，统一fsync到磁盘。
因此只有当sync_binlog参数为1时，才是最安全的，当其不为1时，都存在binlog未fsync到磁盘的风险，若此时发生断电等故障，就有可能出现此事务并未刷出到磁盘，从而故障恢复时将此事务回滚的情况。

## 基于binlog的事务恢复流程
了解了MySQL关于Innodb与Binlog的两阶段提交机制后，就可以更深入去探究MySQL在故障恢复时的处理过程。
在MySQL启动时，首先会初始化存储引擎，如本例中的InnoDB引擎，然后InnoDB引擎层会读取redolog进行InnoDB层的故障恢复，回滚未prepared和commit的事务，但对于已经prepared，但未commit的事务，暂时挂起，保存到一个链表中，等待后续读取binlog日志，然后根据binlog日志再对这部分prepared的事务进行处理。
接下来，MySQL会读取最后一个binlog文件。binlog文件通常是以固定的文件名加一组连续的编号来命名的，并且将其记录到一个binlog索引文件中，因此索引文件中的最后一个binlog文件即是MySQL将要读取的最后一个binlog文件。
读取这个binlog文件时，通过文件头上是否存在标记LOG_EVENT_BINLOG_IN_USE_F，通过这个标记可以知道上次MySQL是正常关闭还是异常关闭，如果是异常关闭，则会进入故障恢复过程。
进入故障恢复过程后，会依次读取最后一个binlog文件中的所有log event，并将所有已提交事务的binlog日志中记录的xid提取出来添加到hash表中，以备后续对前述InnoDB故障恢复后遗留的Prepared事务继续处理。另外此处还要定位最后一个完整事务的位置，防止在上次系统异常关闭时有部分binlog日志未刷到磁盘上，即存在写了一半的binlog事务日志，这部分写了一半binlog日志的事务在MySQL中会按事务未提交来处理，后续会将其在存储引擎层回滚。当此文件中的内容全部读出之后，一是得到一个已提交事务的列表，另一个是最后一个完整事务的位置。
然后检查由InnoDB层得到的Prepared事务列表，若Prepared事务在从Binlog中得到的提交事务列表中，则在InnoDB层提交此事务，否则回滚此事务。
![pic](.img/ecaa69501e41_201812-02.png)
最后MySQL将最后一个完整事务位置之后的binlog清除，完成故障恢复全部过程。

## 基于binlog的两阶段提交对高可用复制解决方案的影响
MySQL最常见的高可用解决方案就是基于binlog复制来完成的，通过将master的binlog复制到slave上，然后在slave上重放，从而达到master与slave上数据一致的效果。
正常情况下，这个方案简单、易用，基本满足大部分用户的高可用需求，但在一些特殊情况下，这个方案还是存在一些不足，可能会导致master与slave存在数据不一致的情况。
如果master与slave之间采用异步模式进行binlog复制，显然就会存在部分binlog未复制到slave的情况。为提高可用性，MySQL支持semi-sync模式，也就是当master在提交事务之前，保证binlog已经复制到slave，并且收到slave回复的ACK后，master再将事务提交。Semi-sync模式的复制机制虽然已经极大提高了可用性，但是在极端情况下还是存在master与slave数据不一致的风险，甚至数据丢失的风险。
考虑一下master出现故障后无法立即恢复的情况，为保障应用的持续性，需要将slave切换为master。若在故障发生前，master恰好有事务正准备提交，并且binlog日志已经刷到磁盘，但在将binlog复制给slave过程中master故障了，备机未收到或只收到部分binlog日志，若此时slave切换为master，显然这些未收到或只收到部分binlog日志的事务是无法重现的，也就是这部分事务是丢失的。理论上应用层并未得到事务提交的反馈，即使事务不存在也不是什么问题。问题是若此时用户查询这些事务不存在，准备重做这些事务，更糟的事情发生了，新的mater也发生故障了，并且无法立即恢复。万幸的是原来的master可以恢复工作了，直接作为master就好了，但问题出现了，用户已经在重做这些事务了，但这些事务在这个master已经存在了，原因如前述的基于binlog的事务恢复。这就好比之前买东西已经付钱给商家200块了，结果人家说没收到，我一查账户，也没少钱，那就再付一次吧，结果杯具了，付了400块给人家？？？
![pic](.img/8b2cc35e6b99_201812-03.png)
如上图所示：若T2的binlog日志尚未复制到slave时，master故障，原slave切换为master，而原master重启恢复后成为新的slave，如下图所示：
![pic](.img/b8802f30a07a_201812-04.png)

## 高可用必杀技–基于RAFT的多副本集群
基于RAFT协议的多副本架构，每一条数据都会被复制多份，通过多副本来增加系统的可用性，防止单副本失效而导致数据不可用。多副本之间基于RAFT协议来实现数据的一致性，只有数据存在于半数以上副本方可认为数据有效，而无效的副本数据系统会自动修复，从而确保系统只会提供一个统一的一致性视图。
![pic](.img/e94b9e35f37d_201812-05.png)
阿里云的MySQL金融版就是基于RAFT的多副本集群，从根本上彻底解决了多副本集群故障切换后的数据不一致的问题，从而实现RPO等于0的目标，相比传统的主备集群有以下优势：

* 消除master故障后由于切换master导致的数据不一致；
* 提供更高可用性，提供链路冗余，防止两节点主备集群中主备链路不稳定导致的主机hang；
* 不低于两节点主备集群的性能；
* 节点故障或网络故障后自动切换master，响应及时；
* 管理透明，用户无需额外管理及学习成本；

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)