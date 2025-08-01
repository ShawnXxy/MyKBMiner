# MySQL · 捉虫动态· replicate filter 和 GTID 一起使用的问题

**Date:** 2015/01
**Source:** http://mysql.taobao.org/monthly/2015/01/09/
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

 ## MySQL · 捉虫动态· replicate filter 和 GTID 一起使用的问题 
 Author: 

 **问题描述**

当单个 MySQL 实例的数据增长到很多的时候，就会考虑通过库或者表级别的拆分，把当前实例的数据分散到多个实例上去，假设原实例为A，想把其中的5个库（db1/db2/db3/db4/db5）拆分到5个实例（B1/B2/B3/B4/B5）上去。

拆分过程一般会这样做，先把A的相应库的数据导出，然后导入到对应的B实例上，但是在这个导出导入过程中，A库的数据还是在持续更新的，所以还需在导入完后，在所有的B实例和A实例间建立复制关系，拉取缺失的数据，在业务不繁忙的时候将业务切换到各个B实例。

在复制搭建时，每个B实例只需要复制A实例上的一个库，所以只需要重放对应库的binlog即可，这个通过 replicate-do-db 来设置过滤条件。如果我们用备库上执行 show slave status\G 会看到Executed_Gtid_Set是断断续续的，间断非常多，导致这一列很长很长，看到的直接效果就是被刷屏了。

为啥会这样呢，因为设了replicate-do-db，就只会执行对应db对应的event，其它db的都不执行。主库的执行是不分db的，对各个db的操作互相间隔，记录在binlog中，所以备库做了过滤后，就出现这种断断的现象。

除了这个看着不舒服外，还会导致其它问题么？

假设我们拿B1实例的备份做了一个新实例，然后接到A上，如果主库A又定期purge了老的binlog，那么新实例的IO线程就会出错，因为需要的binlog在主库上找不到了；即使主库没有purge 老的binlog，新实例还要把主库的binlog都从头重新拉过来，然后执行的时候又都过滤掉，不如不拉取。

有没有好的办法解决这个问题呢？SQL线程在执行的时候，发现是该被过滤掉的event，在不执行的同时，记一个空事务就好了，把原事务对应的GTID位置占住，记入binlog，这样备库的Executed_Gtid_Set就是连续的了。

**bug 修复**

对这个问题，官方有一个相应的bugfix，参见 revno: [5860](http://bazaar.launchpad.net/~mysql/mysql-server/5.6/revision/5860) ，有了这个patch后，备库B1的 SQL 线程在遇到和 db2-db5 相关的SQL语句时，在binlog中把对应的GTID记下，同时对应记一个空事务。

这个 patch 只是针对Query_log_event，即 statement 格式的 binlog event，那么row格式的呢？ row格式原来就已经是这种行为，通过check_table_map 函数来过滤库或者表，然后生成一个空事务。

另外这个patch还专门处理了下 CREATE/DROP TEMPORARY TABLE 这2种语句，我们知道row格式下，对临时表的操作是不会记入binlog的。如果主库的binlog格式是 statement，备库用的是 row，CREATE/DROP TEMPORARY TABLE 对应的事务传到备库后，就会消失掉，Executed_Gtid_Set集合看起来是不连续的，但是主库的binlog记的gtid是连续的，这个 patch 让这种情况下的CREATE/DROP TEMPORARY TABLE在备库同样记为一个空事务。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)