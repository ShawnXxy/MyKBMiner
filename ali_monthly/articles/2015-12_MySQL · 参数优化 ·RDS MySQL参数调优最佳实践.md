# MySQL · 参数优化 ·RDS MySQL参数调优最佳实践

**Date:** 2015/12
**Source:** http://mysql.taobao.org/monthly/2015/12/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 12
 ](/monthly/2015/12)

 * 当期文章

 MySQL · 引擎特性 · InnoDB 事务子系统介绍
* PgSQL · 特性介绍 · 全文搜索介绍
* MongoDB · 捉虫动态 · Kill Hang问题排查记录
* MySQL · 参数优化 ·RDS MySQL参数调优最佳实践
* PgSQL · 特性分析 · 备库激活过程分析
* MySQL · TokuDB · 让Hot Backup更完美
* PgSQL · 答疑解惑 · 表膨胀
* MySQL · 特性分析 · Index Condition Pushdown (ICP)
* MariaDB · 社区动态 · MariaDB on Power8
* MySQL · 特性分析 · 企业版特性一览

 ## MySQL · 参数优化 ·RDS MySQL参数调优最佳实践 
 Author: 玄惭 

 ## 前言
很多时候，RDS用户经常会问如何调优RDS MySQL的参数，为了回答这个问题，写一篇blog来进行解释：

1. 哪一些参数不能修改，那一些参数可以修改；
2. 这些提供修改的参数是不是已经是最佳设置，如何才能利用好这些参数；

## 哪些参数可以改

细心的用户在购买RDS的时候都会看到，不同规格能够提供的最大连接数以及内存是不同的，所以这一些产品规格的限制参数：连接数、内存用户是不能够修改的，如果内存或者连接数出现了瓶颈：

1. 内存瓶颈：实例会出现OOM，然后导致主备发生切换
2. 连接数瓶颈：应用不能新建立连接到数据库

则需要进行应用优化、慢SQL优化或者进行弹性升级实例规格来解决。

还有一些涉及主备数据安全的参数比如`innodb_flush_log_at_trx_commit`、`sync_binlog`、`gtid_mode`、`semi_sync`、`binlog_format`等为了保证主备的数据安全，目前还暂不提供给用户进行修改。

除上述的这些参数外，绝大部分的参数都已经由DBA团队和源码团队优化过，用户不需要过多调整线上的参数就可以把数据库比较好的运行起来。但这些参数只是适合大多数的应用场景，个别特殊的场景还是需要个别对待，比如使用了tokudb引擎，这个时候就需要调整tokudb引擎能使用的内存比例(`tokudb_buffer_pool_ratio`)；又比如我的应用特点本身需要很大的一个锁超时时间，那么则需要调整`innodb_lock_wait_timeout`参数的大小以适应应用等等。

## 如何调参数

下面我将把控制台中能够修改的一些比较重要的参数给大家介绍一下，这些参数如果设置不当，则可能会出现性能问题或应用报错。

### open_files_limit

作用：该参数用于控制MySQL实例能够同时打开使用的文件句柄数目。
原因：当数据库中的表（MyISAM 引擎表在被访问的时候需要消耗文件描述符，InnoDB引擎会自己管理已经打开的表—`table_open_cache`）打开越来越多后，会消耗分配给每个实例的文件句柄数目，RDS在起初初始化实例的时候设置的`open_files_limit`为8192，当打开的表数目超过该参数则会导致所有的数据库请求报错误。
现象：如果参数设置过小可导致应用报错

`[ERROR] /mysqld: Can't open file: './mysql/user.frm' (errno: 24 -Too many open files);
`
建议：提高`open_files_limit`的值，RDS目前可以支撑最大为65535，，同时建议替换MyISAM存储引擎为InnoDB引擎。

### back_log

作用：MySQL每处理一个连接请求的时候都会对应的创建一个新线程与之对应，那么在主线程创建新线程期间，如果前端应用有大量的短连接请求到达数据库，MySQL 会限制此刻新的连接进入请求队列，由参数`back_log`控制，如果等待的连接数量超过`back_log`，则将不会接受新的连接请求，所以如果需要MySQL能够处理大量的短连接，需要提高此参数的大小。
现象：如果参数过小可能会导致应用报错

`SQLSTATE[HY000] [2002] Connection timed out;
`
建议：提高此参数值的大小，注意需要重启实例，RDS在起初初始化的值的默认值是50，现在初始化值已经调大了3000。

### innodb_autoinc_lock_mode

作用：在MySQL5.1.22后，InnoDB为了解决自增主键锁表的问题，引入了参数`innodb_autoinc_lock_mode`，用于控制自增主键的锁机制，该参数可以设置的值为0/1/2，RDS 默认的参数值为1，表示InnoDB使用轻量级别的mutex锁来获取自增锁，替代最原始的表级锁，但是在load data（包括：INSERT … SELECT, REPLACE … SELECT）场景下会使用自增表锁，这样会则可能导致应用在并发导入数据出现死锁。
现象：如果应用并发使用load data(包括：INSERT … SELECT, REPLACE … SELECT)导入数据的时候出现死锁：

`RECORD LOCKS space id xx page no xx n bits xx index PRIMARY of table xx.xx trx id xxx lock_mode X insert intention waiting. TABLE LOCK table xxx.xxx trx id xxxx lock mode AUTO-INC waiting；
`
建议：建议将参数设置改为2，则表示所有情况插入都使用轻量级别的mutex锁(只针对row模式)，这样就可以避免auto_inc的死锁，同时在INSERT … SELECT 的场景下会提升很大的性能（注意该参数设置为2，binlog的格式需要设置为row）。

### query_cache_size

作用：该参数用于控制MySQL query cache的内存大小；如果MySQL开启query cache，再执行每一个query的时候会先锁住query cache，然后判断是否存在query cache中，如果存在直接返回结果，如果不存在，则再进行引擎查询等操作；同时insert、update和delete这样的操作都会将query cahce失效掉，这种失效还包括结构或者索引的任何变化，cache失效的维护代价较高，会给MySQL带来较大的压力，所以当我们的数据库不是那么频繁的更新的时候，query cache是个好东西，但是如果反过来，写入非常频繁，并集中在某几张表上的时候，那么query cache lock的锁机制会造成很频繁的锁冲突，对于这一张表的写和读会互相等待query cache lock解锁，导致select的查询效率下降。
现象：数据库中有大量的连接状态为checking query cache for query、Waiting for query cache lock、storing result in query cache；
建议：RDS默认是关闭query cache功能的，如果您的实例打开了query cache，当出现上述情况后可以关闭query cache；当然有些情况也可以打开query cache，比如：巧用query cache解决数据库性能问题。

### net_write_timeout

作用：等待将一个block发送给客户端的超时时间。
现象：参数设置过小可能导致客户端报错the last packet successfully received from the server was milliseconds ago，the last packet sent successfully to the server was milliseconds ago。
建议：该参数在RDS中默认设置为60S，一般在网络条件比较差的时，或者客户端处理每个block耗时比较长时，由于`net_write_timeout`设置过小导致的连接中断很容易发生，建议增加该参数的大小；

### tmp_table_size

作用：该参数用于决定内部内存临时表的最大值，每个线程都要分配（实际起限制作用的是`tmp_table_size`和`max_heap_table_size`的最小值），如果内存临时表超出了限制，MySQL就会自动地把它转化为基于磁盘的MyISAM表，优化查询语句的时候，要避免使用临时表，如果实在避免不了的话，要保证这些临时表是存在内存中的。
现象：如果复杂的SQL语句中包含了group by/distinct等不能通过索引进行优化而使用了临时表，则会导致SQL执行时间加长。
建议：如果应用中有很多group by/distinct等语句，同时数据库有足够的内存，可以增大`tmp_table_size`(`max_heap_table_size`)的值，以此来提升查询性能。

## RDS MySQL 新增参数

下面介绍几个比较有用的 RDS MySQL 新增参数。

### rds_max_tmp_disk_space

作用：用于控制MySQL能够使用的临时文件的大小，RDS初始默认值是10G，如果临时文件超出此大小，则会导致应用报错。
现象：The table ‘/home/mysql/dataxxx/tmp/#sql_2db3_1’ is full。
建议：需要先分析一下导致临时文件增加的SQL语句是否能够通过索引或者其他方式进行优化，其次如果确定实例的空间足够，则可以提升此参数的值，以保证SQL能够正常执行。注意此参数需要重启实例；

### tokudb_buffer_pool_ratio

作用：用于控制TokuDB引擎能够使用的buffer内存大小，比如`innodb_buffer_pool_size`设置为1000M，`tokudb_buffer_pool_ratio`设置为50（代表50%），那么tokudb引擎的表能够使用的buffer 内存大小则为500M；
建议：该参数在RDS中默认设置为0，如果RDS中使用tokudb引擎，则建议调大该参数，以此来提升TokuDB引擎表的访问性能。该参数调整需要重启数据库实例。

### max_statement_time

作用：用于控制查询在MySQL的最长执行时间，如果超过该参数设置时间，查询将会自动失败，默认是不限制。
建议：如果用户希望控制数据库中SQL的执行时间，则可以开启该参数，单位是毫秒。
现象：ERROR 3006 (HY000): Query execution was interrupted, max_statement_time exceeded

### rds_threads_running_high_watermark

作用：用于控制MySQL并发的查询数目，比如将`rds_threads_running_high_watermark`该值设置为100，则允许MySQL同时进行的并发查询为100个，超过水位的查询将会被拒绝掉，该参数与`rds_threads_running_ctl_mode`配合使用（默认值为select）。
建议：该参数常常在秒杀或者大并发的场景下使用，对数据库具有较好的保护作用。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)