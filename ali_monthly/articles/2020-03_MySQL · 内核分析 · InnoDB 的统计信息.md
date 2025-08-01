# MySQL · 内核分析 · InnoDB 的统计信息

**Date:** 2020/03
**Source:** http://mysql.taobao.org/monthly/2020/03/08/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2020 / 03
 ](/monthly/2020/03)

 * 当期文章

 MySQL · 引擎特性 · 8.0 Instant Add Column功能解析
* PgSQL · 引擎特性 · PostgreSQL 通信协议
* MySQL · 产品特性 · RDS三节点企业版的高可用体系
* AliSQL · 最佳实践 · Performance Agent
* MySQL · 内核分析 · InnoDB mutex 实现分析
* Database · 理论基础 · B link Tree
* MySQL · 引擎特性 · Latch 持有分析
* MySQL · 内核分析 · InnoDB 的统计信息
* MySQL · 引擎特性 · 排序实现
* PgSQL · 插件分析 · plProfiler

 ## MySQL · 内核分析 · InnoDB 的统计信息 
 Author: hw 

 ## 前言

ＭySQL 的InnoDB引擎会维护着用户表每个索引的统计信息， 来帮助查询优化器选择最优的执行计划，详细的来说， key的分布情况能决定多表join的顺序， 也能够决定查询使用哪一个索引。这些统计信息可以由专门的后台线程刷新，也可以由用户也可以显示的调用Analyze table的命令来刷新统计信息， 本文基于最新的MySQL 8.0来具体分析一下刷新统计信息的具体实现。

## 统计信息收集触发以及查看

MySQL有多种方法会触发统计信息的收集，显示的最典型就是Analyze Table 语法，并且由于在MySQL 8.0 中支持了直方图统计信息， 因此analyze table 还扩充了Histogram语法

`ANALYZE [NO_WRITE_TO_BINLOG | LOCAL]
 TABLE tbl_name [, tbl_name] ...

ANALYZE [NO_WRITE_TO_BINLOG | LOCAL]
 TABLE tbl_name
 UPDATE HISTOGRAM ON col_name [, col_name] ...
 [WITH N BUCKETS]

ANALYZE [NO_WRITE_TO_BINLOG | LOCAL]
 TABLE tbl_name
 DROP HISTOGRAM ON col_name [, col_name] ...
`

执行Analyze table 的用户需要拥有表的 SELECT 和 INSERT 权限，由于Analyze table会更新数据字典里的统计信息表（8.0）因此在innodb_read_only 开关被打开时有可能会导致执行失败。 在analyze table的过程中会持有InnoDB 表的 read only 锁， 因此会存在短暂的阻塞用户写入更新删除的操作。 除此之外analyze table 要把table 从 table definition cache 刷出来， 因此还会需要一个flush lock， 此时如果有长事务使用了这张表， 那么必须等待长事务结束。

其次还有自动触发的场景， InnoDB的表在做rebuild index， add column， truncate等涉及数据修改的DDL时会需要设置正确的统计信息。 除此之外在后台有专门的线程，叫做dict_stats_thread 来处理统计信息， InnoDB会长期追踪每一张表的行数， 判断条件是发现更新的记录超过表记录总数的1/10，那么就把这张表加入到后台的recalc pool 中， 而如果变更的行数超过 16+n_rows/16（6.25%），更新非持久化统计信息 。

具体的统计信息可以通过以下语句观察到

`SHOW [EXTENDED] {INDEX | INDEXES | KEYS}
 {FROM | IN} tbl_name
 [{FROM | IN} db_name]
 [WHERE expr]
`

如果开启了InnoDB统计信息持久化，也可以通过查询 innodb_table_stats 和 innodb_index_stats看到

 列名
 描述

 database_name
 库名

 table_name
 表名

 last_update
 最后更新这张表的时间

 n_rows
 表中的数据行数

 clustered_index_size
 聚集索引的页面数

 sum_of_other_index_sizes
 其他非主键索引的页面数

 列名
 描述

 database_name
 库名

 table_name
 表名

 index_name
 索引名

 last_update
 最后更新这张表的时间

 stat_name
 统计项名称

 stat_value
 统计项值

 sample_size
 采样的页面数

 stat_description
 统计的说明

其中stat_name 包括：

* n_diff_pfxNN （不同前缀列的cardinality）
* n_leaf_page (索引叶子节点数目)
* size (索引页面数目)

## 执行计划的相关变量

* innodb_stats_persistent变量控制统计信息是否持久化。统计信息在早期的MySQL中是不持久化， 在新版本的MySQL中持久化是默认的选项。当变量打开时，统计信息就会被持久化到物理表中，统计信息会更加的稳定和精确。否则表的统计信息会在诸如每次重启前周期性的计算。持久化的统计信息也可以手动修改， 修改完成后， 使用FLUSH TABLE 命令可以刷新统计信息（不推荐线上如此操作， 可能会引发一系列的SQL执行计划问题）
* innodb_stats_auto_recalc 变量控制表多少比例的行被修改后自动更新统计信息，默认是10%， 也可以在create 或者alter table 时通过STATS_AUTO_RECALC语法来指定比率。
* innodb_stats_include_delete_marked 变量控制是否在分析索引时包含打上删除标记的记录,在默认的情况下， InnoDB计算统计信息会读未提交的数据， 如果遇到有事务在删除表中的记录，会影响到统计信息的准确度
* innodb_stats_method 统计信息遇到NULL值如何处理， 可以认为相等，也可以认为不想等，或者忽略它们
* innodb_stats_on_metadata 在关闭持久化统计信息时，是否在show table status/查看information_schema的TABLES，STATISTICS表时更新统计信息
* innodb_stats_persistent_sample_pages 开启索引信息持久化后索引统计时采样的页面数， 默认20个页面
* innodb_stats_transient_sample_pages 关闭索引信息持久化后索引统计时采样的页面书， 默认8个页面

## 不带直方图的analyze

Analyze table 是可以探测key的分布情况，并且将其记录到系统表，在每次analyze的时候也会检测数据表是否发生过变化

统计信息会获取非常多的信息， 包括索引的修改时间、大小，等等在诸多的统计信息中其中Cardinality是一个很特殊的维度， 对于Cardinality的评估是通过采样评估的方式对表的每一个索引进行统计， 所以得到的是一个估算值而不是精确值。很多的查询选择到了错误的执行计划也是如此原因。

具体Analyze的代码路径为:

`sql_admin.cc:ql_cmd_analyze_table::execute
 sql_admin.cc:mysql_admin_table
 handler.cc:ha_analyze
 ha_innodb.cc:ha_innodbase::analyze
 ha_innnodb.cc:ha_innodbase::info_low
 dict0stats.cc:dict_stats_update
 dict0stats.cc:dict_stats_update_persistent
 dict0stats.cc:dict_stats_analyze_index
 dict0stats.cc:dict_stats_analyze_index_level
 dict0stats.cc:dict_stats_update_transient
`

在这条路径中我们发现了一个非常有意思的BUG，涉及到最新的5.6/5.7/8.0，在InnoDB的rebuild table 类型的Online DDL 过程中，如果恰好此时有用户做了analyze table 或者InnoDB的后台刷新统计信息的线程刷新到这张表的主键，此时会导致 dict0stats.cc:dict_stats_analyze_index 在调用 btr_get_size 时返回一个空的统计值，这样的后果是让查询优化器会选择全表扫描， 从而导致大量的慢SQL， 直到做完online DDL再此刷新统计信息以后才能恢复正常， 具体的BUG描述可见 [https://bugs.mysql.com/bug.php?id=98132](https://bugs.mysql.com/bug.php?id=98132)

整个统计信息刷新的过程， 如果是主动发起的Analyze table， 会加上server层的MDL_SHARED_READ 锁并且将表从Table Definition Cache中淘汰出去。 以下几类情况比较特殊

* innodb_force_recovery 大于等于4
* innodb_read_only_mode 那么，统计信息不会持久化， 而是走内存
* rtree索引是不采集统计信息的

线程首先获取树的高度， 然后自顶向下， 逐层分析， 如果是复合索引，那么通过逐渐增加前缀依次计算cardinality， 每一层最多扫描N_SAMPLE_PAGES(index）个页面， 如果diff值超过 N_DIFF_REQUIRED(index) = (N_SAMPLE_PAGES(index) * 10)， 那么认为是found_level, 停止扫描， 然后从该层开始（很可能是非叶节点层）， 根据扫描过的记录对数据进行分组，分成若干个Segment， 随机选择每个segment中的一条记录向下探测， 然后计算叶节点的diff值以及external pages。

8.0 中InnoDB的统计做了进一步的细化， 会统计索引页面在缓存Buffer中的比率， Buffer中一个根据Index ID作为Key的哈希结构存储着页面数目， 缓存中的数据和外存中的冷数据不同， 访问的代价差别也是巨大的， 因此这个数据有助于进一步细化

## 直方图的最新变化
直方图是MySQL 8.0 中新增的统计信息方式， Analyze table 加上直方图语句就可以操作直方图的信息, 直方图并不是存储引擎层实现的， 而是在Server层利用InnoDB存储引擎实现的系统表mysql.column_stats，MySQL利用JSON类型的字段来保存直方图的信息，其实现的核心代码在sql/histogram 目录下

具体的操作包括：更新直方图以及drop 直方图， 其中更新直方图还可以重新指定bucket的数目， 需要注意的是直方图不支持加密表， 不支持GIS列以及JSON列，以及不支持单列唯一索引的列。

通过 histogram_generation_max_mem_size参数可以调整用于生成直方图的采样记录内存大小，通过查看information_schema的column_statistic表可以查看 sampling-rate

具体的MySQL 8.0的直方图分析的文章可参考往期的月报文章

[http://mysql.taobao.org/monthly/2016/10/09/](http://mysql.taobao.org/monthly/2016/10/09/)

最新的MySQL-8.0.19 中， InnoDB实现了自己的采样算法，来避免全表扫描。在MySQL计算直方图填充时会调用Handler层的ha_sample_init, ha_sample_next 以及 ha_sample_end 接口。 在8.0.19前 InnoDB并没有实现 sample的接口， 而是用的Handler层的默认实现rnd_next，也就是全表扫描， 直到独到采样比率的数据为止。这里有一个问题，如果采样率设置为10%， 那采样只是读前10%的记录。 更科学的做法是在整棵索引树上均匀的采样。 在新版本中终于有了InnoDB引擎层的sample实现。 目前的代码只支持单线程的采样， 但是从代码架构看已经实现了parallel_reader的接口，不久后一定会实现多线程并行的采样。InnoDB的采样是交给了单独的worker线程来实现的，一般是对主键进行。整体思路就是根据采样比率相对平均的选择叶子节点页面，假设采样率是10%， 那么会选择一个叶子页面后跳过9个叶子页面， 被选中的页面中会对所有的记录进行采样。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)