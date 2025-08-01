# MySQL · 引擎特性 · InnoDB COUNT(*) 优化(?)

**Date:** 2016/06
**Source:** http://mysql.taobao.org/monthly/2016/06/10/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 06
 ](/monthly/2016/06)

 * 当期文章

 MySQL · 特性分析 · innodb 锁分裂继承与迁移
* MySQL · 特性分析 ·MySQL 5.7新特性系列二
* PgSQL · 实战经验 · 如何预测Freeze IO风暴
* GPDB · 特性分析· Filespace和Tablespace
* MariaDB · 新特性 · 窗口函数
* MySQL · TokuDB · checkpoint过程
* MySQL · 特性分析 · 内部临时表
* MySQL · 最佳实践 · 空间优化
* SQLServer · 最佳实践 · 数据库实现大容量插入的几种方式
* MySQL · 引擎特性 · InnoDB COUNT(*) 优化(?)

 ## MySQL · 引擎特性 · InnoDB COUNT(*) 优化(?) 
 Author: 印风 

 在5.7版本中，InnoDB实现了新的handler的records接口函数，当你需要表上的精确记录个数时，会直接调用该函数进行计算。

## 使用

实际上records接口函数是在优化阶段调用的，在满足一定条件时，直接去计算行级计数。其explain出来的结果相比老版本也有所不同，这里我们使用sysbench的sbtest表来进行测试，共200万行数据。

`mysql> show create table sbtest1\G
*************************** 1. row ***************************
 Table: sbtest1
Create Table: CREATE TABLE `sbtest1` (
 `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
 `k` int(10) unsigned NOT NULL DEFAULT '0',
 `c` char(120) NOT NULL DEFAULT '',
 `pad` char(60) NOT NULL DEFAULT '',
 PRIMARY KEY (`id`),
 KEY `k_1` (`k`)
) ENGINE=InnoDB AUTO_INCREMENT=2000001 DEFAULT CHARSET=utf8 MAX_ROWS=1000000
1 row in set (0.00 sec)

mysql> explain select count(*) from sbtest1\G
*************************** 1. row ***************************
 id: 1
 select_type: SIMPLE
 table: NULL
 partitions: NULL
 type: NULL
possible_keys: NULL
 key: NULL
 key_len: NULL
 ref: NULL
 rows: NULL
 filtered: NULL
 Extra: Select tables optimized away
1 row in set, 1 warning (0.00 sec)
`

注意这里Extra里为”Select tables optimized away”，表示在优化器阶段已经被优化掉了。如果给id列带上条件的话，则回退到之前的逻辑

`mysql> explain select count(*) from sbtest1 where id > 0\G
*************************** 1. row ***************************
 id: 1
 select_type: SIMPLE
 table: sbtest1
 partitions: NULL
 type: range
possible_keys: PRIMARY
 key: PRIMARY
 key_len: 4
 ref: NULL
 rows: 960984
 filtered: 100.00
 Extra: Using where; Using index
1 row in set, 1 warning (0.00 sec)
`

## 实现

在[WL#6742](http://dev.mysql.com/worklog/task/?id=6742)中，为InnoDB实现了handler的records函数接口

函数栈

`opt_sum_query
|--> get_exact_record_count
 |--> ha_records
 |--> ha_innobase::records
 |-->row_scan_index_for_mysql
`

* HA_HAS_RECORDS：引擎flag，表示是否可以把count(*)下推到引擎层
* 总是使用聚集索引来进行计算行数
* 只需要读取主键值，无需去读取外部存储列(row_prebuilt_t::read_just_key)，如果行记录较大的话，就可以节省客观的诸如内存拷贝之类的操作开销
* 计算过程可中断，每检索1000条记录，检查事务是否被中断
* 由于只有一次引擎层的调用，减少了Server层和InnoDB的交互，避免了无谓的内存操作或格式转换
* 对于分区表，在5.7版本已经下推到innodb层，因此分区表的计算方式(ha_innopart::records)是针对每个分区调用ha_innobase::records，再将结果累加起来

相关代码:
[commit1](https://github.com/mysql/mysql-server/commit/510dd48bf510dc0a3bda9e62cede698325d05fdd)
[commit2](https://github.com/mysql/mysql-server/commit/40ec5373c044547a66d5456b15d61553de8f3401)

## 缺点

由于总是强制使用聚集索引，缺点很明显：当二级索引的大小远小于聚集索引，且数据不在内存中时，使用二级索引显然要快些，因此文件IO更少。如下例：

默认情况下检索所有行(以下测试都是在清空buffer pool时进行的)：

`mysql> select count(*) from sbtest1;
+----------+
| count(*) |
+----------+
| 2000000 |
+----------+
1 row in set (3.92 sec)
`

即时强制指定索引也没用 :(

`mysql> select count(*) from sbtest1 force index(k_1);
+----------+
| count(*) |
+----------+
| 2000000 |
+----------+
1 row in set (3.86 sec)
`

但如果带上一个简单的条件，让select count(*)走索引k_1，耗费的时间立马下降了….

`mysql> select count(*) from sbtest1 where k > 0;
+----------+
| count(*) |
+----------+
| 2000000 |
+----------+
1 row in set (1.05 sec)
`

个人认为这算是一个性能退化，退一步讲，如果用户知道force index能够走一个更好的索引来计算行数，优化器应该做出选择，而不是总是无条件选择聚集索引，提了个[Bug到官方](http://bugs.mysql.com/bug.php?id=81854)

## 其他

从[WL#6742](http://dev.mysql.com/worklog/task/?id=6742)还提到了一个尚未公布的WL#6605，从其只言片语中可以推断官方有意向实现即时获得行数：

`The next worklog, WL#6605, is intended to return the COUNT(*) through this handler::records() interface almost immediately in all conditions just by keeping track if the base committed count along with transaction deltas
`

让我们继续对新版本保持期待吧 :)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)