# MySQL · 引擎特性 · Cost Model,直方图及优化器开销优化

**Date:** 2018/10
**Source:** http://mysql.taobao.org/monthly/2018/10/02/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 10
 ](/monthly/2018/10)

 * 当期文章

 POLARDB · 最佳实践 · POLARDB不得不知道的秘密
* MySQL · 引擎特性 · Cost Model,直方图及优化器开销优化
* MSSQL · 最佳实践 · 使用混合密钥实现列加密
* MongoDB · 引擎特性 · 复制集原理
* Redis · lazyfree · 大key删除的福音
* Database · 理论基础 · 数据库事务隔离发展历史
* Database · 理论基础 · 关于一致性协议和分布式锁
* MySQL · RocksDB · Level Compact 分析
* MySQL · RocksDB · TransactionDB 介绍
* PgSQL · 应用案例 · 相似人群圈选，人群扩选，向量相似 使用实践

 ## MySQL · 引擎特性 · Cost Model,直方图及优化器开销优化 
 Author: zhaiweixiang 

 MySQL当前已经发布到MySQL8.0版本，在新的版本中，可以看到MySQL之前被人诟病的优化器部分做了很多的改动，由于笔者之前的工作环境是5.6，最近切换到最新的8.0版本，本文涵盖了一些本人感兴趣的和优化器相关的部分，主要包括MySQL5.7的cost model以及MySQL8.0的直方图功能。

本文基于当前最新的MySQL8.0.12版本，主要是讲下cost model 和 histogram的用法和相关代码

## Cost Model

### Configurable cost constants

为什么需要配置cost model常量 ? 我们知道MySQL已经发展了好几十年的历史，但是在优化器中依然使用了hardcode的权重值来衡量io, cpu等资源情况，而这些权重值实际上是基于多年前甚至十来年前的经验设定的。想想看，这么多年硬件的发展多么迅速。几十上百个核心的服务器不在少数甚至在某些大型公司大规模使用，ssd早就成为主流，NVME也在崛起。高速RDMA网络正在走入寻常百姓家。这一切甚至影响到数据库系统的实现和变革。显而易见，那些hardcode的权值已经过时了，我们需要提供给用户可定义的方式，甚至更进一步的，能够智能的根据硬件环境自动设定。

MySQL5.7引入两个新的系统表, 通过这两个系统表暴露给用户来进行更新，如下：

`root@(none) 04:05:24>select * from mysql.server_cost;
+------------------------------+------------+---------------------+---------+---------------+
| cost_name | cost_value | last_update | comment | default_value |
+------------------------------+------------+---------------------+---------+---------------+
| disk_temptable_create_cost | NULL | 2018-04-23 13:55:20 | NULL | 20 |
| disk_temptable_row_cost | NULL | 2018-04-23 13:55:20 | NULL | 0.5 |
| key_compare_cost | NULL | 2018-04-23 13:55:20 | NULL | 0.05 |
| memory_temptable_create_cost | NULL | 2018-04-23 13:55:20 | NULL | 1 |
| memory_temptable_row_cost | NULL | 2018-04-23 13:55:20 | NULL | 0.1 |
| row_evaluate_cost | NULL | 2018-04-23 13:55:20 | NULL | 0.1 |
 +------------------------------+------------+---------------------+---------+---------------+
6 rows in set (0.00 sec)

 其中default_value是generated column，其表达式已经固定死了默认值：

 `default_value` float GENERATED ALWAYS AS (
 (case `cost_name` 
 when _utf8mb3'disk_temptable_create_cost' then 20.0 
 when _utf8mb3'disk_temptable_row_cost' then 0.5 
 when _utf8mb3'key_compare_cost' then 0.05 
 when _utf8mb3'memory_temptable_create_cost' then 1.0 
 when _utf8mb3'memory_temptable_row_cost' then 0.1 
 when _utf8mb3'row_evaluate_cost' then 0.1 else NULL end)) VIRTUAL

 root@(none) 04:05:35>select * from mysql.engine_cost;
 +-------------+-------------+------------------------+------------+---------------------+---------+---------------+
 | engine_name | device_type | cost_name | cost_value | last_update | comment | default_value |
 +-------------+-------------+------------------------+------------+---------------------+---------+---------------+
 | default | 0 | io_block_read_cost | NULL | 2018-04-23 13:55:20 | NULL | 1 |
 | default | 0 | memory_block_read_cost | NULL | 2018-04-23 13:55:20 | NULL | 0.25 |
 +-------------+-------------+------------------------+------------+---------------------+---------+---------------+

`

你可以通过update语句来进行更新, 例如：

` root@(none) 04:05:52>update mysql.server_cost set cost_value = 40 where cost_name = 'disk_temptable_create_cost';
Query OK, 1 row affected (0.05 sec)
 Rows matched: 1 Changed: 1 Warnings: 0

 root@(none) 04:07:13>select * from mysql.server_cost where cost_name = 'disk_temptable_create_cost';
 +----------------------------+------------+---------------------+---------+---------------+
 | cost_name | cost_value | last_update | comment | default_value |
 +----------------------------+------------+---------------------+---------+---------------+
 | disk_temptable_create_cost | 40 | 2018-06-23 16:07:05 | NULL | 20 |
 +----------------------------+------------+---------------------+---------+---------------+
1 row in set (0.00 sec)

 //更新后执行一次flush optimizer_costs操作来更新内存
 //但老的session还是会用老的cost数据
 root@(none) 10:10:12>flush optimizer_costs;
Query OK, 0 rows affected (0.00 sec)

`

可以看到用法也非常简单，上面包含了两张表：server_cost及engine_cost，分别对server层和引擎层进行配置

### 相关代码:

#### 全局cache Cost_constant_cache

全局cache维护了一个当前的cost model信息, 用户线程在lex_start时会去判断其有没有初始化本地指针，如果没有的话就去该cache中将指针拷贝到本地

初始化全局cache:

` Cost_constant_cache::init
 :

 创建Cost_model_constants， 其中包含了两类信息: server层cost model和引擎层cost model, 类结构如下：

 Cost_constant_cache ----> Cost_model_constants
 ---> Server_cost_constants
 //server_cost
 ---> Cost_model_se_info
 --->SE_cost_constants
 //engine_cost 如果存储引擎提供了接口函数get_cost_constants的话,则从存储引擎那取

`

从系统表读取配置，适用于初始化和flush optimizer_costs并更新cache：

`read_cost_constants()
 |--> read_server_cost_constants
 |--> read_engine_cost_constants

`

由于用户可以动态的更新系统表，执行完flush optimizer_costs后，有可能老的版本还在被某些session使用，因此需要引用计数，老的版本ref counter被降为0后才能被释放

#### 线程cost model初始化

* Cost_model_server

在每个线程的thd上，挂了一个Cost_model_server的对象THD::m_cost_model, 在lex_start()时，如果发现线程的m_cost_model没有初始化，就会去获取全局的指针，存储到本地：

` Cost_model_server::init

 const Cost_model_constants *m_cost_constants = cost_constant_cache->get_cost_constants();
 // 会增加一个引用计数，以确保不会在引用时被删除

 const Server_cost_constants *m_server_cost_constants = m_cost_constants->get_server_cost_constants();
 // 同样获取的是全局指针

`

可见thd不创建自己的cost model, 只引用cache中的指针

#### Table Cost Model

struct TABLE::m_cost_model, 类型：Cost_model_table

其值取自上述thd中存储的cost model对象

#### Cost_estimate

统一的对象类型cost_estimate来存储计算的cost结果，包含四个维度：

` double io_cost; ///< cost of I/O operations
 double cpu_cost; ///< cost of CPU operations
 double import_cost; ///< cost of remote operations
 double mem_cost; ///< memory used (bytes)

`

### 未来

目前来看，除非根据工作负载，经过充分的测试才能得出合理的配置值，但如何配置，什么是合理的值，个人认为应该是可以自动调整配置的。关键是找出配置和硬件条件的对应关系。 这也是我们未来可以努力的一个方向。

### reference:

[1. Cost Model官方文档](https://dev.mysql.com/doc/refman/5.7/en/cost-model.html)
 [2. 官方博客1:The MySQL Optimizer Cost Model Project](https://mysqlserverteam.com/the-mysql-optimizer-cost-model-project/)
 [3. 官方博客2: A new dimension to MySQL query optimizations ](http://mysqlserverteam.com/a-new-dimension-to-mysql-query-optimizations-part-2/)
 [4. Optimizer Cost Model Improvements in MySQL 5.7.5 DMR](https://mysqlserverteam.com/optimizer-cost-model-improvements-in-mysql-5-7-5-dmr/)
 [5.Slide: MySQL Cost Model ](https://www.slideshare.net/olavsa/mysql-optimizer-cost-model)

Related Worklog:
 [WL#7182: Optimizer Cost Model API](https://dev.mysql.com/worklog/task/?id=7182) 

 [WL#7209: Handler interface changes for new cost model](https://dev.mysql.com/worklog/task/?id=7209)
 [WL#7276: Configuration data base for Optimizer Cost Model](https://dev.mysql.com/worklog/task/?id=7276)
 [WL#7315 Optimizer cost model: main memory management of cost constants](https://dev.mysql.com/worklog/task/?id=7315)
 [WL#7316 Optimizer cost model: Command for online updating of cost model constants](https://dev.mysql.com/worklog/task/?id=7316)

## Histogram

直方图也是MySQL一个万众期待的功能了，这个功能实际上在其他数据库产品中是很常见的，可以很好的指导优化器选择执行路径。利用直方图存储了指定列的数据分布。MariaDB从很早的10.0.2版本支持这个[功能](https://mariadb.com/kb/en/library/histogram-based-statistics/)， 而MySQL在最新的8.0版本中也开始支持

### 使用

MySQL里使用直方图是通过[ANALYZE TABLE](https://dev.mysql.com/doc/refman/8.0/en/analyze-table.html)语法来执行：

` ANALYZE [NO_WRITE_TO_BINLOG | LOCAL]
 TABLE tbl_name
 UPDATE HISTOGRAM ON col_name [, col_name] ...
 [WITH N BUCKETS]

 ANALYZE [NO_WRITE_TO_BINLOG | LOCAL]
 TABLE tbl_name
 DROP HISTOGRAM ON col_name [, col_name] ...
`

举个简单的例子：

` 我们以普通的sysbench表为例：

 root@sb1 05:16:33>show create table sbtest1\G
 *************************** 1. row ***************************
 Table: sbtest1
 Create Table: CREATE TABLE `sbtest1` (
 `id` int(11) NOT NULL AUTO_INCREMENT,
 `k` int(11) NOT NULL DEFAULT '0',
 `c` char(120) NOT NULL DEFAULT '',
 `pad` char(60) NOT NULL DEFAULT '',
 PRIMARY KEY (`id`),
 KEY `k_1` (`k`)
 ) ENGINE=InnoDB AUTO_INCREMENT=200001 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.01 sec)

# 创建直方图并存储到数据词典中

 root@sb1 05:16:38>ANALYZE TABLE sbtest1 UPDATE HISTOGRAM ON k with 10 BUCKETS;
 +-------------+-----------+----------+----------------------------------------------+
 | Table | Op | Msg_type | Msg_text |
 +-------------+-----------+----------+----------------------------------------------+
 | sb1.sbtest1 | histogram | status | Histogram statistics created for column 'k'. |
 +-------------+-----------+----------+----------------------------------------------+
1 row in set (0.55 sec)

 root@sb1 05:17:03>ANALYZE TABLE sbtest1 UPDATE HISTOGRAM ON k,pad with 10 BUCKETS;
 +-------------+-----------+----------+------------------------------------------------+
 | Table | Op | Msg_type | Msg_text |
 +-------------+-----------+----------+------------------------------------------------+
 | sb1.sbtest1 | histogram | status | Histogram statistics created for column 'k'. |
 | sb1.sbtest1 | histogram | status | Histogram statistics created for column 'pad'. |
 +-------------+-----------+----------+------------------------------------------------+
2 rows in set (7.98 sec)

 删除pad列上的histogram:
 root@sb1 05:17:51>ANALYZE TABLE sbtest1 DROP HISTOGRAM ON pad;
 +-------------+-----------+----------+------------------------------------------------+
 | Table | Op | Msg_type | Msg_text |
 +-------------+-----------+----------+------------------------------------------------+
 | sb1.sbtest1 | histogram | status | Histogram statistics removed for column 'pad'. |
 +-------------+-----------+----------+------------------------------------------------+
1 row in set (0.06 sec)

 root@sb1 05:58:12>ANALYZE TABLE sbtest1 DROP HISTOGRAM ON k;
 +-------------+-----------+----------+----------------------------------------------+
 | Table | Op | Msg_type | Msg_text |
 +-------------+-----------+----------+----------------------------------------------+
 | sb1.sbtest1 | histogram | status | Histogram statistics removed for column 'k'. |
 +-------------+-----------+----------+----------------------------------------------+
1 row in set (0.08 sec)

# 如果不指定bucket的话，默认Bucket的数量是100

 root@sb1 05:58:27>ANALYZE TABLE sbtest1 UPDATE HISTOGRAM ON k;
 +-------------+-----------+----------+----------------------------------------------+
 | Table | Op | Msg_type | Msg_text |
 +-------------+-----------+----------+----------------------------------------------+
 | sb1.sbtest1 | histogram | status | Histogram statistics created for column 'k'. |
 +-------------+-----------+----------+----------------------------------------------+
1 row in set (0.56 sec)

`

直方图统计信息存储于InnoDB数据词典中，可以通过information_schema表来获取

` root@information_schema 05:34:49>SHOW CREATE TABLE INFORMATION_SCHEMA.COLUMN_STATISTICS\G
 *************************** 1. row ***************************
 View: COLUMN_STATISTICS
Create View: CREATE ALGORITHM=UNDEFINED DEFINER=`mysql.infoschema`@`localhost` SQL SECURITY DEFINER VIEW `COLUMN_STATISTICS` AS select `mysql`.`column_statistics`.`schema_name` AS `SCHEMA_NAME`,`mysql`.`column_statistics`.`table_name` AS `TABLE_NAME`,`mysql`.`column_statistics`.`column_name` AS `COLUMN_NAME`,`mysql`.`column_statistics`.`histogram` AS `HISTOGRAM` from `mysql`.`column_statistics` where can_access_table(`mysql`.`column_statistics`.`schema_name`,`mysql`.`column_statistics`.`table_name`)
 character_set_client: utf8
 collation_connection: utf8_general_ci
1 row in set (0.00 sec)
`

从column_statistics表的定义可以看到，有一个名为mysql.column_statistics系统表，但被隐藏了，没有对外暴露

以下举个简单的例子：

` root@sb1 05:58:55>ANALYZE TABLE sbtest1 UPDATE HISTOGRAM ON k WITH 4 BUCKETS;
 +-------------+-----------+----------+----------------------------------------------+
 | Table | Op | Msg_type | Msg_text |
 +-------------+-----------+----------+----------------------------------------------+
 | sb1.sbtest1 | histogram | status | Histogram statistics created for column 'k'. |
 +-------------+-----------+----------+----------------------------------------------+
1 row in set (0.63 sec)

# 查询表上的直方图信息

 root@sb1 06:00:43>SELECT JSON_PRETTY(HISTOGRAM) FROM INFORMATION_SCHEMA.COLUMN_STATISTICS WHERE SCHEMA_NAME='sb1' AND TABLE_NAME = 'sbtest1'\G
 *************************** 1. row ***************************
 JSON_PRETTY(HISTOGRAM): {
 "buckets": [
 [
 38671,
 99756,
 0.249795,
 17002
 ],
 [
 99757,
 100248,
 0.500035,
 492
 ],
 [
 100249,
 100743,
 0.749945,
 495
 ],
 [
 100744,
 172775,
 1.0,
 16630
 ]
 ],
 "data-type": "int",
 "null-values": 0.0,
 "collation-id": 8,
 "last-updated": "2018-09-22 09:59:30.857797",
 "sampling-rate": 1.0,
 "histogram-type": "equi-height",
 "number-of-buckets-specified": 4
 }
1 row in set (0.00 sec)

`

从输出的json可以看到，在执行了上述语句后产生的直方图，有4个bucket，数据类型为Int, 类型为equi-height，即等高直方图(另外一种是等宽直方图，即SINGLETON)。每个Bucket中，描述的信息包括：数值的上界和下界, 频率以及不同值的个数。通过这些信息可以获得比较精确的数据分布情况，从而优化器来根据这些统计信息决定更优的执行计划。

如果列上存在大量的重复值，那么MySQL也可能选择等宽直方图，例如上例，我们将列k上的值更新为一半10一半为20， 那么出来的直方图数据如下：

` root@sb1 10:41:17>SELECT JSON_PRETTY(HISTOGRAM) FROM INFORMATION_SCHEMA.COLUMN_STATISTICS WHERE SCHEMA_NAME='sb1' AND TABLE_NAME = 'sbtest1'\G
 *************************** 1. row ***************************
 JSON_PRETTY(HISTOGRAM): {
 "buckets": [
 [
 10,
 0.499995
 ],
 [
 20,
 1.0
 ]
 ],
 "data-type": "int",
 "null-values": 0.0,
 "collation-id": 8,
 "last-updated": "2018-09-22 14:41:17.312601",
 "sampling-rate": 1.0,
 "histogram-type": "singleton",
 "number-of-buckets-specified": 100
 }
1 row in set (0.00 sec)

`

如上，对于SINGLETON类型，每个bucket只包含两个值：列值，及对应的累计频率（即百分之多少的数据比当前Bucket里的值要小或相等）

注意这里的sampling-rate, 这里的值为1，表示读取了表上所有的数据来进行统计，但通常对于大表而言，我们可能不希望读太多的数据，因为可能产生过度的内存消耗，因此MySQL还提供了一个参数[histogram_generation_max_mem_size](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_histogram_generation_max_mem_size)来限制内存的使用上限。

如果表上的DML不多，那直方图基本是稳定的，但频繁写入的话，那我们就可能需要去定期更新直方图，MySQL本身不会去主动更新。

优化器通过histogram来计算列的过滤性，大多数的谓词都可以使用到。具体参阅[官方文档](https://dev.mysql.com/doc/refman/8.0/en/optimizer-statistics.html)

关于直方图影响查询计划，这篇[博客](https://lefred.be/content/mysql-8-0-histograms/) 及 [这篇博客](https://mysqlserverteam.com/histogram-statistics-in-mysql/)

### 相关代码

**代码结构：**
 以MySQL8.0.12为例，主要代码在sql/histogram目录下:

` ls sql/histograms/
 equi_height_bucket.cc 
 equi_height_bucket.h 
 equi_height.cc 
 equi_height.h histogram.cc 
 histogram.h singleton.cc 
 singleton.h 
 value_map.cc 
 value_map.h 
 value_map_type.h

 类结构：

 namespace histograms
 |---> Histogram //基类
 |--> Equi_height //等高直方图，模板类，实例化参数为数据类型，需要针对类型显示定义
 // 见文件 "equi_height.cc"
 |--> Singleton
 //等宽直方图，只有值和其出现的频度被存储下来

`

**创建及存储histogram:**

处理histogram的相关函数和堆栈如下:

` Sql_cmd_analyze_table::handle_histogram_command
 |--> update_histogram //更新histogram
 |-->histograms::update_histogram //调用namespace内的接口函数
 a. 判断各个列：
 //histograms::field_type_to_value_map_type: 检查列类型是否支持
 //covered_by_single_part_index: 如果列是Pk或者uk，不会为其创建histogram
 //如果是generated column, 则找到其依赖的列加入到set中
 b. 判断取样的半分比，这主要受参数histogram_generation_max_mem_size限制，如果设的足够大，则会去读取全表数据进行分析
 |-> fill_value_maps //开始从表上读取需要分析的列数据
 |->ha_sample_init
 |->ha_sample_next
 |--> handler::sample_next //读取下一条记录，通过随机数的方式来进行取样
 Value_map<T>::add_values // 将读到的数据加入到map中
 |->...
 |->ha_sample_end

 |-> build_histogram //创建histogram对象
 a. 确定histogram类型：如果值的个数小于桶的个数，则使用Singleton，否则使用Equi_height类型
 |->Singleton<T>::build_histogram
 |->Equi_height<T>::build_histogram

 |-> Histogram::store_histogram //将histogram信息存储到column_statistic表中
 |-> dd::cache::Dictionary_client::update<dd::Column_statistics>

 |--> drop_histogram //删除直方图

`

**使用histogram**

使用的方式就比较简单了:

首先在表对象TABLE_SHARE中，增加成员m_histograms，其结构为一个unordered map，key值为field index, value为相应的histogram对象

获取列值过滤性的相关堆栈如下：

` get_histogram_selectivity
 |-->Histogram::get_selectivity
 |->get_equal_to_selectivity_dispatcher
 |->get_greater_than_selectivity_dispatcher
 |->get_less_than_selectivity_dispatcher
 |-->write_histogram_to_trace // 写到optimizer_trace中
`

MySQL支持多种操作类型对直方图的使用，包括：

` col_name = constant
 col_name <> constant
 col_name != constant
 col_name > constant
 col_name < constant
 col_name >= constant
 col_name <= constant
 col_name IS NULL
 col_name IS NOT NULL
 col_name BETWEEN constant AND constant
 col_name NOT BETWEEN constant AND constant
 col_name IN (constant[, constant] ...)
col_name NOT IN (constant[, constant] ...)
`

通过直方图，我们可以根据列上的条件判断出列值的过滤性，来辅助选择更优的执行计划。在没有直方图之前我们需要通过在列上建立索引来获得相对精确的列值分布。但我们知道索引是有很大的维护开销的，而直方图则可以灵活的按需创建。

### reference

[WL#5384 PERFORMANCE_SCHEMA, HISTOGRAMS](https://dev.mysql.com/worklog/task/?id=5384)
 [WL#8706 Persistent storage of Histogram data](https://dev.mysql.com/worklog/task/?id=8706)
 [WL#8707 Classes/structures for Histograms](https://dev.mysql.com/worklog/task/?id=8707)
 [WL#8943 Extend ANALYZE TABLE with histogram support](https://dev.mysql.com/worklog/task/?id=8943)
 [WL#9223 Using histogram statistics in the optimizer](https://dev.mysql.com/worklog/task/?id=9223)

## 其他

### 优化rec_per_key

相关worklog:
 [WL#7338: Interface for improved records per key estimates](https://dev.mysql.com/worklog/task/?id=7338)
 [WL#7339 Use improved records per key estimate interface in optimizer](https://dev.mysql.com/worklog/task/?id=7339)

MySQL通过rec_per_key 接口来估算记录的个数（暗示每个索引Key对应的记录个数），但在早前版本中这个数字是整数，对于小数会取整，不能表示准确的rec_per_key，从而影响到索引的选择，因此在5.7版本中，将其记录的值改成了float类型

### 引入数据cache状态计算开销

相关worklog:

[WL#7168 API for estimates for how much of table and index data that is in memory buffer](https://dev.mysql.com/worklog/task/?id=7168)
 [WL#7170: InnoDB buffer estimates for tables and indexes](https://dev.mysql.com/worklog/task/?id=7170)
 [WL#7340 IO aware cost estimate function for data access](https://dev.mysql.com/worklog/task/?id=7340)

在之前的版本中，优化器是无法知道数据的状态，是否是cache在内存中，还是需要从磁盘读出来的，缺乏这部分信息，导致优化器统一认为数据属于磁盘的来计算开销。这可能导致低效的执行计划。

相关代码：

server层新增api，用于获取表或索引上有百分之多少的数据是存储在cache中的

` handler::table_in_memory_estimate
 handler::index_in_memory_estimate
`

而在innodb层，增加了一个全局变量`buf_stat_per_index` (对应类型为`buf_stat_per_index_t`) 来维护每个索引在内存中的leaf page个数, 其内部实现了一个lock-free的hash结构，Key值为`(m_space_id) << 32 | m_index_id)`, 在读入page时或者内存中创建新page时， 如果对应的page是leaf page，就递增计数；当从page hash中移除时，则递减计数。

为了减少性能的影响，计数器是通过lock-free hash的结构存储的，对应的结构为`ut_lock_free_hash_t`。
 基本的实现思路是：hash是一个定长的数组，数组元素为(key, val), 根据Key计算一个hash值再模上array size, 找到对应的槽位, 如果槽位被占用了，则向右查找一个空闲的slot。
 当数组满了的时候，会创建一个新的更大的数组，在数据还没Move到这个新hash之前，所有的search都需要查询两个数组。当所有的记录到迁移到新数组，并且没有线程访问老的数组时，就可以把老的hash删除掉了。

在hash中存储的counter本身，也考虑到多核和numa架构，避免同时更新引起的cpu cache失效。在大量core的场景下这个问题可能很明显。Innodb封装计数操作到类`ut_lock_free_cnt_t`中，使用数组维护counter, 按照cpu no作为index更新，需要获取counter值时则累加数组中的值。

这个Lock free hash并不是个通用场景的hash结构：例如处理冲突的时候，可能占用其他key的槽位，hash不够用时，需要迁移到新的array中。实际上mysql本身实现了一个lf_hash，在扩展Hash时无需迁移数据，有空单独开篇博客讲一下。

你可以从`information_schema.innodb_cached_indexes`表中读取到每个索引cache的page个数。

当定义好接口，并且Innodb提供相应的统计数据后，优化器就可以利用这些信息来计算开销：

* Cost_model_table::page_read_cost
* Cost_model_table::page_read_cost_index

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)