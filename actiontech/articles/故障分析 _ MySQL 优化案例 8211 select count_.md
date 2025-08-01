# 故障分析 | MySQL 优化案例 &#8211; select count(*)

**原文链接**: https://opensource.actionsky.com/20200707-mysql/
**分类**: MySQL 新特性
**发布时间**: 2020-07-07T00:43:57-08:00

---

作者：xuty
本文来源：原创投稿
*爱可生开源社区出品，原创内容未经授权不得随意使用，转载请联系小编并注明来源。
本文关键字：count、SQL、二级索引
相关文章推荐：
[故障分析 | MySQL 优化案例 &#8211; 字符集转换](https://opensource.actionsky.com/20200630-mysql/)
[技术分享 | MySQL 监控利器之 Pt-Stalk](https://opensource.actionsky.com/20200522-mysql/)
**一、故事背景**
项目组联系我说是有一张 500w 左右的表做 **select count(*)** 速度特别慢。
**二、原 SQL 分析**
**Server version: 5.7.24-log MySQL Community Server (GPL)**
SQL 如下，仅仅就是统计 **api_runtime_log** 这张表的行数，一条简单的不能再简单的 SQL：
- 
`select count(*) from api_runtime_log;`
我们先去运行一下这条 SQL，可以看到确实运行很慢，要 40 多秒左右，确实很不正常~- 
- 
- 
- 
- 
- 
- 
```
mysql> select count(*) from api_runtime_log;`+----------+``| count(*) |``+----------+``|  5718952 |``+----------+``1 row in set (42.95 sec)
```
我们再去看下表结构，看上去貌似也挺正常的~存在主键，表引擎也是 InnoDB，字符集也没问题。- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
```
CREATE TABLE `api_runtime_log_copy` (``  `BelongXiaQuCode` varchar(50) DEFAULT NULL,``  `OperateUserName` varchar(50) DEFAULT NULL,``  `OperateDate` datetime DEFAULT NULL,``  `Row_ID` int(11) DEFAULT NULL,``  `YearFlag` varchar(4) DEFAULT NULL,``  `RowGuid` varchar(50) NOT NULL,``   ......``  `apiid` varchar(50) DEFAULT NULL,``  `apiname` varchar(50) DEFAULT NULL,``  `apiguid` varchar(50) DEFAULT NULL,``  PRIMARY KEY (`RowGuid`)``) ENGINE=InnoDB DEFAULT CHARSET=utf8
```
**三、执行计划**
通过执行计划，我们看下是否可以找到什么问题点。- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
mysql> explain select count(*) from api_runtime_log \G;``*************************** 1. row ***************************``           id: 1``  select_type: SIMPLE``        table: api_runtime_log``   partitions: NULL``         type: index``possible_keys: NULL``          key: PRIMARY``      key_len: 152``          ref: NULL``         rows: 5718952``     filtered: 100.00``      Extra: Using index`可以看到，查询走的是 PRIMARY，也就是主键索引。貌似也没有什么问题，走索引了呀！那么是不是真的就没问题呢？
**四、原理**
为了找到答案，通过 Google 查找 MySQL 下 **select count(*)** 的原理，找到了答案。这边省略过程，直接上结果。
简单介绍下原理：
- 聚簇索引：每一个 InnoDB 存储引擎下的表都有一个特殊的索引用来保存每一行的数据，称为聚簇索引（通常都为**主键**），聚簇索引实际保存了 B-Tree 索引和行数据，所以大小实际上约等于为表数据量
- 二级索引：除了聚集索引，表上其他的索引都是二级索引，索引中仅仅存储了对应索引列及主键列
在 **InnoDB 存储引擎**中，**count(*)** 函数是先从内存中读取数据到**内存缓冲区**，然后进行扫描获得行记录数。这里 InnoDB 会**优先走二级索引**；如果同时存在多个二级索引，会选择**key_len 最小**的二级索引；如果不存在二级索引，那么会走**主键索引**；如果连主键都不存在，那么就走**全表扫描**！
这里我们由于走的是**主键索引**，所以 MySQL 需要先把整个**主键索引**读取到内存缓冲区，这是个从磁盘读写到内存的过程，而且主键索引基本等于整个表数据量（**10GB+**），所以非常耗时！
那么如何解决呢？
答案就是：建二级索引。
因为二级索引只包含对应的索引列及主键列，所以体积非常小。在 **select  count(*)** 的查询过程中，只需要将二级索引读取到内存缓冲区，只有**几十 MB** 的数据量，所以速度会非常快。
举个形象的比喻，我们想知道一本书的页数：- 走聚集索引：从第一页翻到最后一页，知道总页数；
- 走二级索引：通过目录直接知道总页数。
**五、验证**
创建二级索引后，再次执行 SQL 及查看执行计划。- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
`mysql> create index idx_rowguid on api_runtime_log(rowguid);``Query OK, 0 rows affected (0.01 sec)``Records: 0  Duplicates: 0  Warnings: 0``
``mysql> select count(*) from api_runtime_log;``+----------+``| count(*) |``+----------+``|  5718952 |``+----------+``1 row in set (0.89 sec)``
``mysql> explain select count(*) from api_runtime_log \G;``*************************** 1. row ***************************``           id: 1``  select_type: SIMPLE``        table: api_runtime_log``   partitions: NULL``         type: index``possible_keys: NULL``          key: idx_rowguid``      key_len: 152``          ref: NULL``         rows: 5718952``     filtered: 100.00``        Extra: Using index``1 row in set, 1 warning (0.00 sec)`可以看到添加二级索引后，确实速度明显变快，而且执行计划也变成了走二级索引。至此这个问题其实已经解决了，就是**由于表上缺少二级索引导致**。
**六、深入测试**
为了进一步验证上述的推论，所以就做了如下的测试。
测试过程如下：
1. 通过 sysbench 创建了一张 500W 的测试表 **sbtest1**，表上仅仅包含一个主键索引，表大小为 1125MB；2. 调整部分 MySQL 参数，重启 MySQL，保证目前 **innodb buffer pool **(内存缓冲区) 中为空，不缓存任何数据；3. 执行 **select count(*)**，理论上走**主键索引**，查看当前内存缓冲区中缓存的数据量（理论上会缓存整个聚簇索引）；4. 在测试表 **sbtest1** 上**添加二级索引**，索引大小为 55MB；5. 再次重启 MySQL，保证内存缓冲区为空；6. 再次执行 **select count(*)**，理论上走**二级索引**；7. 再次查看内存缓冲区中缓存的数据量（理论上只会缓存二级索引）。
测试结果如下：
**1. 聚簇索引**查询当前内存缓冲区状态，结果为空证明不缓存测试表数据。- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
`mysql> select * from sys.innodb_buffer_stats_by_table where object_schema = 'test';``Empty set (1.92 sec)``
``mysql>  select count(*) from test.sbtest1;``+----------+``| count(*) |``+----------+``|  5188434 |``+----------+``1 row in set (5.52 sec)`再次查看内存缓冲区，发现缓存了 **sbtest1** 表上 1G 多的数据，基本等于整个表数据量。- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
```
mysql> select * from sys.innodb_buffer_stats_by_table where object_schema = 'test' \G;`*************************** 1. row ***************************``object_schema: test``  object_name: sbtest1``    allocated: 1.08 GiB``         data: 1.01 GiB``        pages: 71081`` pages_hashed: 0``    pages_old: 28119``  rows_cached: 5189798
```
最后我们再来看下执行计划，确实走的是主键索引，放在最后执行是为了避免影响缓冲区。- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
```
mysql> explain  select count(*) from test.sbtest1 \G;                                          ``*************************** 1. row ***************************``           id: 1``  select_type: SIMPLE``        table: sbtest1``   partitions: NULL``         type: index``possible_keys: NULL``          key: PRIMARY``      key_len: 4``          ref: NULL``         rows: 5117616``     filtered: 100.00``        Extra: Using index
```
**2. 二级索引**创建二级索引 idx_id，查看 sbtest1 表上主键索引与二级索引的数据量。- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
```
mysql> create index idx_id on sbtest1(id);``Query OK, 0 rows affected (12.97 sec)``Records: 0  Duplicates: 0  Warnings: 0``
``mysql> SELECT sum(stat_value) pages ,index_name ,``(round((sum(stat_value) * @@innodb_page_size)/1024/1024)) as MB ``  FROM mysql.innodb_index_stats ``  WHERE table_name = 'sbtest1' ``  AND database_name = 'test' ``  AND stat_description = 'Number of pages in the index' ``  GROUP BY index_name;``+-------+------------+------+``| pages | index_name | MB   |``+-------+------------+------+``| 72000 | PRIMARY    | 1125 |``|  3492 | idx_id     |   55 |``+-------+------------+------+
```
重启 MySQL，再次查看缓冲区同样为空，证明没有缓存测试表上的数据。- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
```
mysql> select * from sys.innodb_buffer_stats_by_table where object_schema = 'test';``Empty set (1.49 sec)``
``mysql> select count(*) from test.sbtest1;``+----------+``| count(*) |``+----------+``|  5188434 |``+----------+``1 row in set (2.92 sec)
```
再次查看内存缓冲区，发现仅仅缓存了 sbtest1 表上的 50M 数据，约等于二级索引的数据量。- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
```
mysql> select * from sys.innodb_buffer_stats_by_table where object_schema = 'test' \G;``*************************** 1. row ***************************``object_schema: test``  object_name: sbtest1``    allocated: 49.48 MiB``         data: 46.41 MiB``        pages: 3167`` pages_hashed: 0``    pages_old: 1575``rows_cached: 2599872
```
最后确认下执行计划，确实走的是二级索引。- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
- 
```
mysql> explain select count(*) from test.sbtest1 \G;``*************************** 1. row ***************************``           id: 1``  select_type: SIMPLE``        table: sbtest1``   partitions: NULL``         type: index``possible_keys: NULL``          key: idx_id``      key_len: 4``          ref: NULL``         rows: 5117616``     filtered: 100.00`        Extra: Using index
```
**七、案例总结**
从上述这个测试结果可以看出，和之前的推论基本吻合。
如果 **select count(*)** 走的是主键索引，那么会缓存整个表数据，大量查询时间会花费在读取表数据到缓冲区。
如果存在二级索引，那么只需要读取索引页到缓冲区即可，速度自然快。> 另：项目上由于磁盘性能层次不齐，所以当遇上这种情况时，性能较差的磁盘更会放大这个问题；一张超级大表，统计行数时如果走了主键索引，后果可想而知~
**八、优化建议**
此次测试过程中我们仅仅模拟是**百万数据量**，此时我们通过二级索引统计表行数，只需要读取几十 M 的数据量，就可以得到结果。
那么当我们的表数据量是**上千万**，甚至**上亿**时呢。此时即便是最小的二级索引也是 **几百 M、过 G** 的数据量，如果继续通过二级索引来统计行数，那么速度就不会如此迅速了。
这个时候可以通过避免直接 **select count(*) from table** 来解决，方法较多，例如：1. 使用 MySQL 触发器 + 统计表实时计算表数据量；2. 使用 MyISAM 替换 InnoDB，因为 MyISAM 自带计数器，坏处就不多说了；3. 通过 ETL 导入表数据到其他更高效的异构环境中进行计算；4. 升级到 MySQL 8 中，使用并行查询，加快检索速度。
当然，什么时候 InnoDB 存储引擎可以直接实现计数器的功能就好了！