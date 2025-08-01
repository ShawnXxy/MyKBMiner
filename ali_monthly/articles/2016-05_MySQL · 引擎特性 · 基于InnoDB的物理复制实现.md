# MySQL · 引擎特性 · 基于InnoDB的物理复制实现

**Date:** 2016/05
**Source:** http://mysql.taobao.org/monthly/2016/05/01/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 05
 ](/monthly/2016/05)

 * 当期文章

 MySQL · 引擎特性 · 基于InnoDB的物理复制实现
* MySQL · 特性分析 · MySQL 5.7新特性系列一
* PostgreSQL · 特性分析 · 逻辑结构和权限体系
* MySQL · 特性分析 · innodb buffer pool相关特性
* PG&GP · 特性分析 · 外部数据导入接口实现分析
* SQLServer · 最佳实践 · 透明数据加密在SQLServer的应用
* MySQL · TokuDB · 日志子系统和崩溃恢复过程
* MongoDB · 特性分析 · Sharded cluster架构原理
* PostgreSQL · 特性分析 · 统计信息计算方法
* MySQL · 捉虫动态 · left-join多表导致crash

 ## MySQL · 引擎特性 · 基于InnoDB的物理复制实现 
 Author: 印风 

 最近有幸前去美国参加Percona Live 2016会议并分享了我们最近在MySQL复制上所做的工作，也就是基于InnoDB的物理复制。会后很多小伙伴私信我说[分享的PPT](https://www.percona.com/live/data-performance-conference-2016/sessions/physical-replication-based-innodb)太内核了，不太容易理解。因此本文主要针对分享的内容进行展开描述，希望能对大家有所帮助。

## 背景知识

在开始之前，你需要对InnoDB的事务系统有个基本的认识。如果您不了解，可以参考我之前的几篇关于InnoDB的文章，包括InnoDB的[事务子系统](http://mysql.taobao.org/monthly/2015/12/01/)，[事务锁](http://mysql.taobao.org/monthly/2016/01/01/)，[redo log](http://mysql.taobao.org/monthly/2015/05/01/)，[undo log](http://mysql.taobao.org/monthly/2015/04/01/)，以及[崩溃恢复逻辑](http://mysql.taobao.org/monthly/2015/06/01/)。在这里我们简单的概述一下几个基本的概念：

**事务ID**：一个自增的序列号，每次开启一个读写事务（或者事务从只读转换成读写模式）时分配并递增，每更新256次后持久化到Ibdata的事务系统页中。每个读写事务都必须保证拥有的ID是唯一的。

**Read View**: 用于一致性读的snapshot，InnoDB里称为视图；在需要一致性读时开启一个视图，记录当时的事务状态快照，包括当时活跃的事务ID以及事务ID的上下水位值，以此用于判断数据的可见性。

**Redo Log**：用于记录对物理文件的修改，所有对InnoDB物理文件的修改都需要通过Redo保护起来，这样才能从崩溃中恢复。

**Mini Transaction(mtr)**：是InnoDB中修改物理块的最小原子操作单位，同时也负责生产本地的redo日志，并在提交mtr时将redo日志拷贝到全局log buffer中。

**LSN**: 一个一直在递增的日志序列号，在InnoDB中代表了从实例安装到当前已经产生的日志总量。可以通过LSN计算出其在日志文件中的位置。每个block在写盘时，其最近一次修改的LSN也会记入其中，这样在崩溃恢复时，无需Apply该LSN之前的日志。

**Undo Log**: 用于存储记录被修改之前的旧版本，如果被多次修改，则会产生一个版本链。保留旧版本的目的是用于可重复读。通过结合Undo和视图控制来实现InnoDB的MVCC。

**Binary Log**: 构建在存储引擎之上的统一的日志格式；有两种存储方式，一种是记录执行的SQL，另外一种是记录修改的行记录。Binlog本质上是一种逻辑日志，因此能够适用所有的存储引擎，并进行数据复制。

## 原生复制的优缺点

MySQL的每条读写事务都需要维持两份日志，一份是redo log，一份是binary log。MySQL使用两阶段提交协议，只有当redo 和binlog都写入磁盘时，事务才算真正的持久化了。如果只写入redo，未写入binlog，这样的事务在崩溃恢复时需要回滚掉。MySQL通过XID来关联InnoDB的事务和binlog。

MySQL的原生事务日志复制有一些显著的优点：
首先，相比InnoDB的redo log而言，Binary Log更加可读，有成熟的配套工具来进行解析；由于记录了行级别的更改。我们可以通过解析binlog，转换成DML语句来将数据变更同步到异构数据库。另外一种典型的做法是使用Binlog来失效构建在前端的cache。事实上，基于Binlog的数据流服务在阿里内部使用的非常广泛，也是最重要的基础设施之一。

其次由于Binary log是一种统一的日志格式，你可以在主备上使用不同的存储引擎，例如当你需要测试某种新的存储引擎时，你可以搭建一个备库，将所有表alter到新引擎，然后开启数据复制进行观察。

此外基于Binary Log你还可以构建起非常复杂的复制拓扑结构，尤其是在引入了GTID之后，这种优势尤为明显: 如果设计妥当，你可以实现相当复杂的复制结构。甚至可以做到多点写入。总体使用起来非常灵活。

然而，也正是这种日志架构可能会带来一些问题：首先MySQL需要记录两份日志：redo及binlog，只有当两份日志都fsync到磁盘，我们才能认为事务是持久化的，而众所周知，fsync是一种开销非常昂贵的操作。更多的日志写入还增加了磁盘IO压力。这两点都会影响到响应时间和吞吐量。

Binlog复制还会带来复制延迟的问题。我们知道只有主库事务提交后，日志才会写入到binlog文件并传递到备库，这意味着备库至少延迟一个事务的执行时间。另外有些操作例如DDL，大事务等等，由于在备库需要继续保持事务完整性，这些执行时间很长的操作会长时间占用某个worker线程，而协调线程会碰到复制同步点，导致后续的任务无法分发到其他空闲的worker线程。

MySQL是原生复制是MySQL生态的一个非常重要的组成部分。官方也在积极的改进其特性，例如MySQL5.7在这一块就有非常显著的改进。

## Why Phsyical Replication

既然原生复制这么成熟，优点这么多，为什么我们还要考虑基于物理日志的复制呢？

首先最重要的原因就是性能！当我们事先了物理复制后，就可以关闭binlog和gtid，大大减少了数据写盘量。这种情况下，最多只需要一次fsync既可以将事务持久化到磁盘。实例整体的吞吐量和响应时间都得到了非常大的提升。

另外，通过物理复制，我们能获得更加理想的物理复制性能。事务在执行过程中产生的redo log只要写到文件中，就会被传送到备库。这意味着我们可以同时在主备库上执行事务，而无需等待主库上执行完成。我们可以基于(space_id, page_no)来进行并发apply，同一个page上的变更也可以做到合并写操作，相比传统复制，具有更好的并发性。最重要的是，基于物理变更的复制，可以最大程度保证主备的数据总是一致的。

当然物理复制不是银弹，当启用该特性后，我们将只能支持InnoDB存储引擎；我们也很难去设计多点写复制拓扑。物理复制无法取代原生复制，而是应对特定的场景，例如需求高并发DML性能的场景。

因此在正式开始前，我们设置了这些前提：1.主库上不应该有任何限制； 2.备库上只允许执行查询操作，不允许通过用户接口对数据产生任何的变更。

下文默认MySQL已包含如下特性：

* 没有只读事务链表，并且不为只读事务分配事务ID
* 使用全局事务ID数组来构建read view快照
* 所有MySQL库下的系统表都使用InnoDB存储引擎

## High Level Architecture

### 复制架构

这里复制的基础架构和原生复制类似，但代码是完全独立的。如下图所示：
![14627529803838](http://img4.tbcdn.cn/L1/461/1/3cbc0eb465f326ce6ca01d73e5d1ea2d796e2346)

首先，我们在备库上配置好连接后，执行START INNODB SLAVE，备库上会开启一个io线程，同时InnoDB层启动一个Log Apply协调线程以及多个worker线程。

IO线程建立和主库的连接，并发送一个dump请求，请求的内容包括：
master_uuid: 最近备库上日志最初产生所在的实例的server_uuid
start_lsn: 开始复制的点

在主库上，一个log_dump线程被创建，先检查dump请求是否是合法的，如果合法，就去从本地的ib_logfile中读取日志，并发送到备库。

备库IO线程在接受到日志后，将其拷贝到InnoDB的Log Buffer中，然后调用log_write_up_to将其写入到本地的ib_logfile文件中。

Log Apply协调线程被唤醒，从文件中读取日志进行解析，并根据fold(space id ,page no)% (n_workers + 1)进行分发，系统表空间的变更存放到sys hash中，用户表空间的变更存储到user hash中。协调线程在解析&&分发完毕后，也会参与到日志apply中。

当Apply日志时，我们总是先应用系统表空间，再是用户表空间。原因是我们需要保证undo日志先应用，否则外部查询检索用户表的btree，试图通过回滚段指针查询undo page，可能对应的Undo还没构成。

### 日志文件管理

要实现上述架构，第一个要解决的问题是需要重新整理InnoDB的日志文件。 因为原生逻辑中，InnoDB采用循环写文件的方式，例如当我们设置innodb_log_files_in_group为4时，会创建4个ib logfile文件。当第四个文件写满时，会回到第一个文件循环写入。但是在物理复制架构下，我们需要保留老的日志文件，这些文件既可以防止例如网络出现问题，日志未曾及时传送到备库，也可以用于备份目的。

我们像binlog那样，当当前日志文件写满时，则切换到下一个日志文件，文件的序号总是向前递增的。然而这里需要解决的一个问题是：切换文件需要尽量减小对性能的影响，我们引入了独立的后台线程，并允许已被清理的日志文件重用。

和binlog类似，我们也需要清理已经没用的日志文件，既需要提供接口，由用户手动清理，也可以开启后台线程自动判断并进行清理，但两种方案都需要满足条件：

1. 不允许超过当前checkpoint所在的文件
2. 如果有正在连接的备库，则不允许清理尚未传送到备库的日志

文件架构如下图所示：
![14627791857156](http://img2.tbcdn.cn/L1/461/1/1c4c996ded90ec619402c9b1722787a524688254)

这里我们增加了一个新的文件ib_checkpoint，原因是原生逻辑中，checkpoint信息是存储在ib_logfile0中的，而在新的架构下，该文件可能被删除掉，我们需要单独对checkpoint信息进行存储，包含checkpoint no, checkpoint lsn, 以及该Lsn所在的日志文件号及文件内偏移量。

后台清理线程被称为log purge thread，当该线程被唤醒被执行清理操作时，将目标日志文件rename到以purged作为前缀，并放到一个回收池中，如果池子满了，则直接删除掉。

为了避免日志切换到新文件时造成的性能抖动，后台log file allocate线程总是预先将下一个文件准备好，也就是说，当前正在写第N个文件，后台线程会被唤醒，并创建好第N+1个文件。这样对前台线程的影响仅仅是关闭并打开新文件句柄。

log file allocate线程在准备下一个文件时，先尝试从回收池中获取文件，并进行必要的判断（确保下一个文件开始的LSN转换成block no后不和文件内的内容重叠），如果可以使用，则直接取出来并rename为下一个文件名。如果回收池无可用文件，则创建文件，并extend到指定的大小。通过这种方式，我们尽量保证了性能的平缓。

### 实例角色

和原生复制不同，对于备库，我们总是不允许做任何的数据变更，这种行为不受是否重启，是否崩溃而影响，只受failover影响。一台备库无论重启多少次总是为备库。

日志最初产生的服务器我们称为日志源实例。日志可能通过复杂的复制拓扑传递到多级级联实例上。但所有的这些备库都应具有相同的源实例信息。我们需要通过这个信息来判断一个dump请求是否是合法的，例如作为备库，所有dump的日志都应产自同一个日志源实例，除非在复制拓扑中发生了failover。

我们为实例定义了三种状态：master, slave,以及upgradable-slave;其中第三种是一种中间状态，只在failover时产生。

这些状态信息被持久化到本地文件innodb_repl.info文件中。同时也单独存储了日志源实例的server_uuid。

我们以下图为例：
![14627861736537](http://img1.tbcdn.cn/L1/461/1/e05ca3422ce4af4def37f3d2ecc9ff0c51323431)

server 1的uuid为1，和文件中记录的uuid相同，因此认为该实例为master;
server 2的uuid为2，和文件中记录的uuid不同，因为该实例为slave;
server 3的uuid为3，但文件中记录的值为0，表明最近刚发生过一次failover（server 1 和server 2发生过一次切换），但还没来得及获取到切换日志，因此该实例角色为upgradable-slave

innodb_repl.info文件维持了所有的复制和failover状态信息，很显然，如果我们想从已有的拓扑中restore出一个新的实例，对应的innodb_repl.info文件也要拷贝出来。

### 后台线程

有些后台线程可能对数据产生变更，因此在备库上我们需要禁止这些线程：

1. 不允许开启Purge线程
2. master线程不允许去做ibuf merge之类的工作，只负责定期做lazy checkpoint
3. dict_stats线程只负责更新表的内存统计信息，不可以触发统计信息的物理存储。

此外备库的page cleaner线程的刷脏算法也需要重新调整以尽量平缓，不要影响到日志apply。

## MySQL Server层数据复制

### 文件操作复制
为了实现Server-Engine的架构，MySQL在Server层另外冗余了一些元数据信息，以在存储引擎之上建立统一的标准。这些元数据文件包括FRM，PAR，DB.OPT，TRG，TRN以及代表数据库的目录。对这些文件和目录的操作都没有写到redo中。

为了能够实现文件层的操作，我们需要将文件变更操作写到日志中，主要扩展了三种新的日志类型：

`MLOG_METAFILE_CREATE: [FIL_NAME | CONTENT]
MLOG_METAFILE_RENAME: [ORIGINAL_NAME | TARGET_NAME]
MLOG_METAFILE_DELETE: [FIL_NAME]
`
这里包含了三种操作，文件的创建，重命名及删除。注意这里没有修改文件操作，原因是Server层总是通过创建新文件，删除旧文件的方式来进行元数据更新。

### DDL复制

当MySQL在执行DDL修改元数据时，是不允许访问表空间的，否则可能导致各种异常错误。MySQL使用排他的MDL锁来阻塞用户访问。我们需要在备库保持相同的行为。这就需要识别修改元数据的起点和结束点。我们引入两类日志来进行标识。

 Name
 Write On Master
 Apply On Slave

 MLOG_METACHANGE_BEGIN
 在获取MDL锁，修改元数据之前写入
 获取表上的显式排他MDL锁，同时失效该表的所有table cache对象

 MLOG_METACHANGE_END
 在释放MDL锁之前写入
 释放表上的MDL锁

举个简单的例子:

`执行： CREATE TABLE t1 (a INT PRIMARY KEY, b INT);
从Server层产生的日志包括：
* MLOG_METACHANGE_START
* MLOG_METAFILE_CREATE (test/t1.frm)
* MLOG_METACHANGE_END

执行： ALTER TABLE t1 ADD KEY (b);
从Server层产生的日志包括：
* Prepare Phase
 MLOG_METACHANGE_START
 MLOG_METAFILE_CREATE (test/#sql-3c36_1.frm)
 MLOG_METACHANGE_END
* In-place build…slow part of DDL
* Commit Phase
 MLOG_METACHANGE_START
 MLOG_METAFILE_RENAME(./test/#sql-3c36_1.frm to ./test/t1.frm)
 MLOG_METACHANGE_END

`

然而元数据修改开始点和结束点所代表的两个日志并不是原子的，这意味着主库上在修改元数据的过程中如果crash了，就会丢失后面的结束标记。备库可能一直持有这个表上的MDL锁无法释放。为了解决这个问题，我们在主库每次崩溃恢复后，都写一条特殊的日志，通知所有连接的备库释放其持有的所有MDL排他锁。

另外一个问题存在于备库，举个例子，执行MLOG_METACHANGE_START后，做一次checkpoint，在接受到MLOG_METACHANGE_END之前crash。当备库实例从崩溃中恢复时，需要能够继续保持MDL锁，避免用户访问。

为了能够恢复MDL，首先我们需要控制checkpoint的LSN，保证不超过所有未完成元数据变更的最老的开始点；其次，在重启时搜集未完成元数据变更的表名，并在崩溃恢复完成后依次把MDL 排他锁加上。

### Cache失效

在Server层还维护了一些Cache结构，然而数据的更新是体现在物理层的，备库在应用完redo后，需要感知到哪些Cache是需要进行更新的，目前来看主要有以下几种情况：

1. 权限操作，备库上需要进行ACL Reload，才能让新的权限生效；
2. 存储过程操作，例如增删存储过程，在备库需要递增一个版本号，以告诉用户线程重新载入cache；
3. 表级统计信息，主库上通过更新的行的数量来触发表统计信息更新；但在备库上，所有的变更都是基于块级别的，并不能感知到变化了多少行。因此每次主库更新统计信息时同时写一条日志到redo中，通知备库进行内存统计信息更新。

## 备库MVCC

### 视图控制
备库一致性读的最基本要求是用户线程不应该看到主库上尚未执行完成的事务所产生的变更。换句话说，当备库上开启一个read view时，在该时间点，如果有尚未提交的事务变更，这些变更应该是不可见的。

基于此，我们需要知道一个事务的开始点和结束点。我们增加了两种日志来进行标示：
**MLOG_TRX_START：** 在主库上为一个读写事务分配事务ID后，同时生成一条日志，日志中记录了该ID的值；由于是持有trx_sys->mutex锁生成的日志记录，因此保证写入redo的事务ID是有序的。
**MLOG_TRX_COMMIT：** 在事务提交阶段，标记undo状态为提交后，写入该类型日志，记录对应事务的事务ID

在备库上，我们通过这两类日志来重现事务场景，具体的我们采用一种延迟构建的方式：只有在完成apply一批日志后才对全局事务状态进行更新：

1. 在apply一批日志时，选择其中最大的MLOG_TRX_START+1来更新trx_sys->max_trx_id
2. 所有未提交的事务ID被加入到全局事务数组中。

如下图所示：
![14627994373389](http://img1.tbcdn.cn/L1/461/1/97d8f2c42781aaaffa34d4b884f09a486a68a320)

在初始状态下，最大未分配事务id（trx_sys->max_trx_id）为11，活跃事务ID数组为空；
在执行第一批日志期间，所有用户请求构建的视图都具有一样的结构。即low_limit_id = up_limit_id = 11，本地trx_ids为空；
在执行完第一批日志后，max_trx_id被被更新成12 + 1，未完成的事务ID 12加入到全局活跃事务ID数组中。
依次类推。该方案是复制效率和数据可见性的一个权衡。

注意如果主库崩溃，那么可能存在事务存在开始点，但丢失结束点的情况，因此主库在崩溃恢复后写入一条特殊的日志，以告诉所有的备库去通过遍历undo slot重新初始化全局事务状态。

### Purge控制
既然要维持MVCC特性，那么作为一致性读的重要组成部分的Undo log，就需要对其进行控制，那些仍然可能被读视图引用的Undo不应该被清理掉。这里我们提供了两种方式来供用户选择：

方案一：控制备库上的Purge
当主库每次Purge时，都将当前Purge的最老快照写入redo；备库在拿到这个快照后，会去判断其和当期实例上活跃的最老视图是否有可见性上的重叠，并等待直到这些视图关闭；我们也提供了一个超时选项，当等待时间过长时，就直接更新本地Purge视图，用户线程将获得一个错误码DB_MISSING_HISTORY

这种方案的缺点很明显：当备库读负载很重，或者存在大查询时，备库可能产生复制延迟。

方案二：控制主库上的Purge，备库定期向其连接的实例发送反馈，反馈的内容为当前可安全Purge的最小ID。如下图所示：
![14628005017014](http://img2.tbcdn.cn/L1/461/1/42b520501e911c90c731364ce16bbb193aafafd0)

这种方案的缺点是，牺牲了主库的Purge效率，在整个复制拓扑上，只要有长时间未关闭的视图，都有可能引起主库上的Undo膨胀。

### B-TREE结构变更复制

当发生B-TREE的结构变更时，例如Page合并或分裂，我们需要禁止用户线程对btree进行检索。

解决方案很简单：当主库上的mtr在commit时，如果是持有索引的排他锁，并且一个mtr中的变更超过一个page时，则将涉及的索引id写到日志中；备库在解析到该日志时，会产生一个同步点：完成已经解析的日志；获取索引X锁；完成日志组Apply；释放索引X锁。

## 复制Change Buffer

### 备库change buffer合并
Change buffer是InnoDB的一种特殊的缓存结构，其本质上是一棵存在于ibdata的btree。当修改用户表空间的二级索引页时，如果对应的page不在内存中，该操作将可能被记录到change buffer中，从而减少了二级索引的随机IO，并达到了合并更新的效果。

随后当对应的page被读入内存时，会进行一次merge操作；后台Master线程也会定期发起Merge。关于change buffer本文不做深入，感兴趣的可以阅读我之前的[这篇月报](http://mysql.taobao.org/monthly/2015/07/01/)

然而在备库，我们需要保证对数据不做任何的变更，只读操作不应该对物理数据产生任何的影响。为了实现这一点，我们采用了如下方式来解决这个问题：

1. 当将Page读入内存，如果发现其需要进行ibuf merge，则为其分配一个shadow page，将未修改的数据页保存到其中；
2. 将change buffer记录合并到数据页上，同时关闭该Mtr的redo log，这样修改后的Page就不会放到flush list上了；
3. change buffer bitmap页和change buffer btree上的页都不允许产生任何的修改；
4. 当数据页从buffer pool驱逐或者被log apply线程请求时，shadow page会被释放掉。

另外一个问题是，主备库的内存状态可能是不一样的，例如一个Page在主库上未读入内存，因此为其缓存到change buffer。但备库上这个page已经存在于buffer pool了。为了保证数据一致性，在备库上我们需要将新的change buffer记录合并到这个page上。

具体的，当在备库解析到新的change buffer entry时，如果对应的Page已经在内存中了，就对其打个标签。随后用户线程如果访问到这个page，就从shadow page中恢复出未修改的Page（如果有shadow page），再进行一次change buffer合并操作。

### 复制change buffer合并

由于一次change buffer merge涉及到ibuf bitmap page，二级索引页，change buffer btree三类，其存在严格的先后关系，而在备库上，我们是并行进行日志apply的。为了保证在合并的过程中，用户线程不能访问到正在被修改的数据页。我们增加了新的日志类型：

**MLOG_IBUF_MERGE_START ：** 在主库上进行ibuf merge之前写入；当备库解析到该日志时，apply所有已解析的日志，获取对应的block，并加上排他锁；如果有shadow page的话，则将未修改的数据恢复出来，并释放shadow page。

**MLOG_IBUF_MERGE_END：** 在主库上清除ibuf bitmap page上对应位后写入；备库解析到时apply所有已解析的日志并释放block锁。

很显然该方案构成了一个性能瓶颈点，可能会影响到复制性能。后续再研究下有没有更完美的解决方案。

## Failover

### Planned Failover

当执行计划中的切换时，我们需要执行严格的步骤，以确保在切换时所有的实例处于一致的状态。具体的分为4步：
Step1: 主库上执行降级操作，状态从MASTER修改成UPGRADABLE-SLAVE；这里会退出所有的读写事务，挂起或退出哪些可能修改数据的后台线程；同时一条MLOG_DEMOTE日志写入到redo文件中。

Step2: 所有连接的备库在读取到MLOG_DEMOTE日志后，将自己的状态修改为UPGRADALE-SLAVE；

Step3: 任意挑选一个复制拓扑中的实例，将其提升为主库，同时初始化各种内存状态值；并写入一条类型为MLOG_PROMOTE的日志；

Step4: 所有连接过来的备库在解析到MLOG_PROMOTE日志后，将自己的状态从UPGRADABLE-SLAVE修改成SLAVE

### Unplanned Failover

然而多数情况下，切换都是在意外情况下发生的，为了减少宕机时间，我们需要选择一个备库快速接管用户负载。这种场景下需要解决的问题是：老主库在恢复访问后，如何确保和新主库的状态一致。更具体的说，如果老主库上还有一部分日志还没传送到新主库，这部分的不一致数据该怎么恢复。

我们采用覆盖写的方法来解决这一问题：

1. 首先禁止老主库上所有的访问，包括查询；同时将老主库降级成备库；
2. 获取新主库切换时的LSN，然后在老主库上从这个LSN开始遍历redo日志，搜集所有影响到（space id, page no），如果发现有DDL操作，则认为恢复失败，需要从外部第三方工具进行比较同步，或者重做实例；
3. 从新主库上获取到这些page并在本地进行覆盖写操作；
4. 完成覆盖写后，将多出来的redo log从磁盘上truncate掉，同时更新checkpoint信息；
5. 恢复复制，并开启读请求。

## 测试及性能

我们测试了三个版本的性能：

* ALI_RDS_56_redo: 使用物理复制，并禁止binlog
* ALI_RDS_56: 目前RDS的MySQL版本
* MySQL5629: Upstream 5.6.29

测试环境

* Sysbench 0.5
* 50 tables, each with 200,000 records
* Buffer pool size: 16GB, 8 buffer pool instance, all data fit in memory
* innodb_thread_concurrency = 32
* Log file group is big enough, so no sharp checkpoint will happen Gtid disabled
* 2 threads per core; 6 cores per socket; 2 CPU sockets

Update_non_index (TPS)
![14628033739409](http://img3.tbcdn.cn/L1/461/1/62f16e30d8bf4f7127c0e917f467a11174aa09d1)

Update_non_index (RT)
![14628033889596](http://img2.tbcdn.cn/L1/461/1/8726dffa269451cc93f7581c7b3ca725bebc1069)

Update_non_index(TPS)
![14628034178982](http://img2.tbcdn.cn/L1/461/1/9591a71aa5ec98afcbefe3e9509142572b708e90)

Update_non_index(RT)
![14628034360808](http://img2.tbcdn.cn/L1/461/1/2c7ead4ac01d5ab41ec6e7bd49602a8ffd6f77ca)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)