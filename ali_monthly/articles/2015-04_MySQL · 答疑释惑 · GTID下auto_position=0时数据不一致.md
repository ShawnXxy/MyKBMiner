# MySQL · 答疑释惑 · GTID下auto_position=0时数据不一致

**Date:** 2015/04
**Source:** http://mysql.taobao.org/monthly/2015/04/10/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 04
 ](/monthly/2015/04)

 * 当期文章

 MySQL · 引擎特性 · InnoDB undo log 漫游
* TokuDB · 产品新闻 · RDS TokuDB小手册
* TokuDB · 特性分析 · 行锁(row-lock)与区间锁(range-lock)
* PgSQL · 社区动态 · 说一说PgSQL 9.4.1中的那些安全补丁
* MySQL · 捉虫动态 · 连接断开导致XA事务丢失
* MySQL · 捉虫动态 · GTID下slave_net_timeout值太小问题
* MySQL · 捉虫动态 · Relay log 中 GTID group 完整性检测
* MySQL · 答疑释惑 · UPDATE交换列单表和多表的区别
* MySQL · 捉虫动态 · 删被引用索引导致crash
* MySQL · 答疑释惑 · GTID下auto_position=0时数据不一致

 ## MySQL · 答疑释惑 · GTID下auto_position=0时数据不一致 
 Author: 沽月 

 ## 问题重现

搭建一主一备，主备配置分别如下 ，同时设置备库的auto_position=0

`$cat crash_recovery-slave.opt

gtid_mode=on 

enforce_gtid_consistency=on 

log_slave_updates=on

relay_log_purge=OFF

sync_relay_log_info=1000

sync_relay_log=1

sync_relay_log_info=100

$cat crash_recovery-master.opt

gtid_mode=on 

enforce_gtid_consistency=on 

log_slave_updates=on

`

用 sysbench 不断对主库进行压测，由于主库压力比较大，可以发现备库延迟不断增加，在有延迟的情况下，重启备库 OS 并启动备库 mysql server，关闭主库压力，待主备延迟为零的时候，做主备校验（这样的过程我们称之为一轮，在每一轮的结尾处做主备校验），这时可以发现会有一个表的 checksum 不一致，即产生了主备不一致的问题。

## 问题分析

1. 分别在主备库比较 `show global variables like '%gtid_executed%'` 可以发现主备的 gtid_executed 的值是相等的;
2. 将 checksum 不一致的表中的数据分别取出，然后vimdiff 一下，找到不一致表的具体数据的主建;
3. 在主库的binlog中找到对这条数据最近一次操作的gtid;
4. 解析备库的relaylog，并查找步骤 3) 中的gtid，可以发现，该gtid是一个relay log的结尾，且文件结尾处没有rotate log event.
5. 继续解析relaylog文件，可以发现在format event之后是一个table_map_event, update_rows, xid_log_event, gtid_log_event, 只是gtid_log_event 的 id 小于3)中出现问题的gtid;
6. 从 5）解析的relay log 来看，备库crash后，并不是接着crash之前的 binlog 来进行拉的，而是 crash 之前的一个位点，假设我们在crash之前拉取了 gtid 为30的binlog event，并sync 了relay log,此时，master_log_info记录的是之前的主库事务位点，假设为事务 10 的一个位点，那么当 OS 重启后，由于备库 auto_position=0, 会从master_log_info中的位点10来拉取binlog，从而形成了这样的binlog序列：

gtid_log_event(30), table_map_event(10), update_rows_log_event(10), xid_log_event(10), gtid_log_event(11)…..

这样的后果是将事务10的数据再次执行并误认为是事务30的数据，而直正拉取到事务30的binlog event时不执行，从而造成主备不一致的问题。

## 解决方案

1. 打开gtid时，必须指定auto_position= 1;
2. 备库在记录master_log_info时，以事务为单位记录位点信息，而不是以event为单位记录位点信息，这个需要在handle_slave_io中修改源码。

## 参数说明

sync_relay_log_info：Synchronously flush relay log info to disk after every #th transaction，每隔多少个事务 sync 一次 relay log 信息；

sync_master_info： Synchronously flush master info to disk after every #th event，每隔多少个log_event sync 一次 master log 信息；

sync_relay_log： Synchronously flush relay log to disk after every #th event，每隔多少个 log_event sync 一次 relay log 信息；

mysql 在读取binlog event时，会首先将位点信息写入操作系统的文件，但是没有 sync 操作，所以当OS crash时，会造成之前写但没有 sync 的位点信息丢失。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)