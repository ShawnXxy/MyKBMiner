# MySQL · 最佳实践 · 一个TPC-C测试工具sqlbench使用

**Date:** 2018/07
**Source:** http://mysql.taobao.org/monthly/2018/07/09/
**Images:** 2 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 07
 ](/monthly/2018/07)

 * 当期文章

 MySQL · 引擎特性 · WAL那些事儿
* MySQL · 源码分析 · 8.0 原子DDL的实现过程续
* MongoDB · 引擎特性 · 事务实现解析
* MySQL · RocksDB · 写入逻辑的实现
* MySQL · 源码分析 · binlog crash recovery
* PgSQL · 新特征 · PG11并行Hash Join介绍
* MySQL · myrocks · clustered index特性
* MSSQL · 最佳实践 · 实例级别数据库上云RDS SQL Server
* MySQL · 最佳实践 · 一个TPC-C测试工具sqlbench使用
* PgSQL · 应用案例 · PostgreSQL flashback(闪回) 功能实现与介绍

 ## MySQL · 最佳实践 · 一个TPC-C测试工具sqlbench使用 
 Author: 荣生 

 TPC-C是数据库系统经常使用的一个性能测试标准，目前开源社区里有几个可以使用的TPC-C测试工具，如BenchmarkSQL、DBT2、 tpcc-mysql等。今天这里要介绍的是另一个TPC-C测试工具: [sqlbench](https://github.com/swida/sqlbench)。

## sqlbench 特性

sqlbench fork自DBT2，改写了整个架构，原来的DBT2把整个测试过程分成client和 driver 两个应用程序，每个terminal需要 2个线程。如果测试的warehouse较多需要占用机器大量资源。 而sqlbench对这方面做了优化，合并了这两个应用程序，同时优 化了线程的使用，使用1个线程处理多个terminal，大大减少了机器资源的使用。使一台机器可以运行更多的warehouse。另外 DBT2外部依赖较多，如对R环境的依赖，sqlbench去掉了不必要的外部依赖，目前sqlbench只依赖测试数据库的客户端库。下面 是sqlbench的部分特性：

* 支持多种数据库，目前支持PostgreSQL、MySQL、Kingbase、ODBC
* 支持多种协议和SQL执行方式，包括普通的SQL语句、PBE（Prepare、Bind、Execute）和存储过程1
* 支持每种数据库的数据快速加载接口，如：PostgreSQL的 Copy，MySQL的 Load Data Local InFile，同时实现并行加载使数 据加载更快

下面将主要以测试PostgreSQL为例介绍sqlbench的安装和使用，这里我们装载20个warehouse并运行测试120秒。

## sqlbench编译安装

sqlbench代码目前托管在github上，可以下载发布的版本，也可以直接使用仓库中的代码，这里以仓库中的代码为例。 下载代码解压后进入sqlbench目录先执行autreconf -if，然后执行configure:

`autoreconf -if
./configure --with-postgresql=yes --with-mysql=yes \
 --with-kingbase=<kingbase-installation-path> --with-odbc=yes
`

* 这里使用 –with-postgresql指定支持postgresql测试，如果–with-postgresql指定yes，configure会在当前PATH下找 pg_config，通过pg_config配置编译的选项
* –with-postgresql也可以接PostgreSQL的安装路径，它会根据指定的安装配置编译选项
* –with-mysql与–with-postgresql类似，只是当指定yes时使用的是mysql_config
* –with-kingbase只支持指定kingbase安装的路径
* –with-odbc打开ODBC支持只能指定yes

还可以通过给configure加上–prefix选项指定安装路径。configure执行完后执行make && make install安装

## 创建测试表

使用psql连接到数据库通过执行源代码目录下src/scripts/create-tables.sql文件来创建测试所需的表：

`psql tpcc -f src/scripts/create-tables.sql
# 对于MySQL使用：
mysql -D tpcc <src/scripts/create-tables.sql
`

我们先不创建索引，这样装载数据会快一些，当装载完数据再创建索引。

## 装载数据

src/core/datagen用来装载数据：

`src/data/datagen -t postgresql --dbname=tpcc --host=127.0.0.1 -w 20 -j5
# 对于MySQL类似
`

第一个选项使用-t来指定数据库类型（如果指定-t选项，-t必须是第一个选项，这和后面的sqlbench命令是一样的），后面的 长选项是该数据库的连接参数。对于PostgreSQL和MySQL可以指定数据库名、主机名、端口号等，具体可通过datagen –help查 看。这里如果没有指定-t选项，datagen会将数据写到文本文件，然后可通过数据库内置装载命令装载的数据库，默认文件会生 成在当前目录，可以-d改变生成文件的位置。上面还通过-w选项指定了要测试的warehouse数和-j指定了使用多少个并发装载数 据。

## 创建索引

数据装载完就可以创建索引了，类似于创建表，通过psql或mysql命令来执行src/scripts/create-indexes.sql脚本。创建完索 引后通过analyze命令对数据库进行analyze（MySQL对每个表运行analyze table），使用数据库在运行测试时可以生成正确的 查询计划。

## 运行测试

执行src/core/sqlbench命令运行测试：

`src/core/sqlbench -t postgresql --dbname=tpcc --host=127.0.0.1 -w 20 -l 120 -r 20 -c 5
`

这里:

* -t 指定PostgreSQL数据库 –dbname和–host为数据库的连接参数，其他没指定的使用默认参数
* -w 指定测试数据为20个warehouse
* -l 共运行测试时间为120秒
* -r 其中ramp up的时间为20秒
* -c 共使用5个数据库连接

sqlbench其他的常用参数包括：

* –no-thinktime 默认的TPC-C测试是有keying time和thinking time的，模拟真正的用户场景，可通过指定这个参数将相应 的时间设置成0，来只对CPU加压
* –sqlapi 选择SQL执行的方式，可选：
 
 simple 为普通SQL方式
* extended 使用prepare/bind/execute方式，该方式先生成查询计划缓存起来，后面直接执行，效率更高
* storeproc 使用存储过程，这种方式与比extended相比还节省了与数据库服务器通信的开销

 -s与-e指定开始和结束的warehouse数，在更多warehouse时，可以使用这2个选项分配warehouse，分成多个sqlbench压测同 一个数据库
 –altered默认情况sqlbench下根据TPC-C标准生成terminal数（每个terminal代表一个user，每个warehouse 10个terminal， 也可以使用–tpw改变），这个参数直接指定了terminal个数，被这些warehouse平均分。
 –sleep指定每创建一个线程后sleep的时间，默认为1s
 -o 用来指定output目录，用来存储错误日志及测试结果文件

sqlbench的其他参数是用来定制TPC-C标准的各个部分的，包括keying time、thinking time的时间，各个事务所占比率，各个 表的数据量等，默认值都是遵从TPC-C标准的。

## 查看测试结果

sqlbench测试过程中会将测试数据写到output目录下的mix.log中，而src/utils/post-process命令是用来处理这个文件查看测 试结果的：

`src/utils/post_process -l mix.log
`

```
 Response Time (s)
 Transaction % Average : 90th % Total Rollbacks %
------------ ----- ----------------- ------- ------------ -----
 Delivery 3.96 0.061 : 0.071 45 0 0.00
 New Order 46.21 0.038 : 0.052 525 5 0.95
Order Status 3.61 0.007 : 0.010 41 0 0.00
 Payment 43.13 0.013 : 0.017 490 0 0.00
 Stock Level 3.08 0.011 : 0.016 35 0 0.00
------------ ----- ----------------- ------- ------------ -----

262.50 new-order transactions per minute (NOTPM)
2.0 minute duration
0 total unknown errors
20 second(s) ramping up

```

以上结果第1列为事务类型，第2列为该事务类型所占的比例，第3列和第4列分别为平均响应时间和90百分位数响应时间（90th percentile）,TPC-C要求这2个响应时间必须小于5秒。第5列每种事务总数，第6列和第7列回滚事务数和回滚率，TPC-C要求New Order事务有一定的回滚率。最后的262.50为每分钟New Order事务的个数，这是TPC-C比较重要的一个性能指标。

post_process还可以通过-t选项输出每种事务随时间变化的每分钟执行事务数的统计数据，再配合脚本 src/utils/plot_transaction_rate可以得到对应每分钟事务数的变化曲线[2](#fn.2)：

`src/utils/post_process -l mix.log -t transactions-rate.log
src/utils/plot_transaction_rate transaction-rates.log \
 transactions-rate.png
`

下图是上面测试结果的每分钟事务数的变化曲线。 ![img](.img/a67122f8d89e_5db37e355c1a064ea1ffe86355374abd.png)

## 写在最后

目前sqlbench工具还不完善，欢迎试用这个工具，如果遇到什么问题可以在Github上提issue，如果感兴趣希望做改进也欢迎提 merge request。

### Footnotes

[1](#fnr.1) 目前MySQL、ODBC只支持普通的SQL方式

[2](#fnr.2) plot_transaction_rate需要系统安装gnuplot

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)