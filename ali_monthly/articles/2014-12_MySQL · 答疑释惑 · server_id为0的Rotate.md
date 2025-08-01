# MySQL · 答疑释惑 · server_id为0的Rotate

**Date:** 2014/12
**Source:** http://mysql.taobao.org/monthly/2014/12/06/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2014 / 12
 ](/monthly/2014/12)

 * 当期文章

 MySQL · 性能优化 · 5.7 Innodb事务系统
* MySQL · 踩过的坑 · 5.6 GTID 和存储引擎那会事
* MySQL · 性能优化 · thread pool 原理分析
* MySQL · 性能优化 · 并行复制外建约束问题
* MySQL · 答疑释惑 · binlog event有序性
* MySQL · 答疑释惑 · server_id为0的Rotate
* MySQL · 性能优化 · Bulk Load for CREATE INDEX
* MySQL · 捉虫动态·Opened tables block read only
* MySQL·　优化改进· GTID启动优化
* TokuDB · TokuDB · Binary Log Group Commit with TokuDB

 ## MySQL · 答疑释惑 · server_id为0的Rotate 
 Author: 

 **背景**

　　在MySQL的M-S结构里面，event是binlog日志的基本单位。每个event来源于主库，每个Event都包含了serverid，用于表示该event是哪个实例生成的。

　　在5.6里面，细心的同学会发现，备库的relaylog中出现了server_id为0的event，其类型为Rotate Event。

　　这里说说server_id=0的Rotate Event。

**心跳event**

　　MySQL Cluster中从NDB 6.3开始就出现的HEADBEAT event(hb event), 在社区版直到5.6.2才提供。

　　hb event的目的是为了保持M-S之间的心跳。用法上是slave在change master的时候可以指定MASTER_HEARTBEAT_PERIOD。当此值为0时，主库发送完所有事件后这个主备通道就一直idle直到发送新的event；当此值为非0的n时，主库通道在idle超过n秒之后，发一个hb event。

　　心跳event的另外一个作用是主库将当前的最新位点通知给备库。hb event中包含主库当前binlog最新位置的文件名和位点。备库收到hb event后判断主库位点是否大于本地保存的位点，若是，则在relay log中记录一个server_id为0的Rotate事件， 这意味着主库上新增了不需要发送给自己的event。

**出现条件**

　　在传统的主备环境中，正常情况下心跳事件是不会被触发写入到备库的relaylog的。这是因为所有的主库binlog中的事件都会发给备库，所以备库收到的hb event中的位点总是不大于备库已经接收到的binlog event最大值（注意到hb event只在通道idle时才发）。

　　但是在5.6启用了GTID以后，就出现了这样的case。最常见的是每个binlog文件开头用于表示之前所有binlog执行过的事件合集的Previous-GTIDs，这个事件需要记录在binlog中，但是不需要发给slave。这就会让备库在接收到hb之后记录一个server_id=0的Rotate event。

**主库relaylog**

　　与此相关的，一个可能出现的现象是双M单写场景下，备库没有更新，但是主库会一直写relay log。

　　步骤如下：

　　1、主备之间完成MM关系(GTID_MODE=on)

　　2、主库和备库各自stop slave

　　3、主库执行大量更新

　　4、主库start slave

　　5、备库start slave

　　在备库同步日志过程中生成了本地的binlog，这些binlog需要再发回给主库。5.6的一个机制是，如果发现通道对面的接收方的executed_set已经包含了这个事件，则不发送。

　　由于这些事件本身就是主库发送过来的，因此备库都不需要发回。但是备库必须通知主库本地的binlog的最新位点，因此构造了一个hb event。

　　主库收到hb event后记录在relaylog中，形式就是server_id=0的Rotate事件。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)