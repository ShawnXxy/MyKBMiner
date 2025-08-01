# MySQL · 最佳实践 · RDS 只读实例延迟分析

**Date:** 2016/04
**Source:** http://mysql.taobao.org/monthly/2016/04/08/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 04
 ](/monthly/2016/04)

 * 当期文章

 MySQL · 参数故事 · innodb_additional_mem_pool_size
* GPDB · 特性分析 · Segment事务一致性与异常处理
* GPDB · 特性分析 · Segment 修复指南
* MySQL · 捉虫动态 · 并行复制外键约束问题二
* PgSQL · 性能优化 · 如何潇洒的处理每天上百TB的数据增量
* Memcached · 最佳实践 · 热点 Key 问题解决方案
* MongoDB · 最佳实践 · 短连接Auth性能优化
* MySQL · 最佳实践 · RDS 只读实例延迟分析
* MySQL · TokuDB · TokuDB索引结构--Fractal Tree
* MySQL · TokuDB · Savepoint漫谈

 ## MySQL · 最佳实践 · RDS 只读实例延迟分析 
 Author: 玄惭 

 ## 前言
只读实例是目前 RDS 用户实现数据读写分离的一种常见架构，用户只需要将业务中的读请求分担到只读节点上，就可以缓解主库查询压力，同时也可以把一些 OLAP 的分析查询放到另外的只读节点上，减小复杂统计查询对主库的冲击，RDS只读节点架构图如下：

![screenshot](http://img4.tbcdn.cn/L1/461/1/a518d69f97eba439223723b6d52a6190024490e5)

由于RDS只读节点采用原生的MySQL Binlog复制技术，那么延迟必然会成为其成立之初就会存在的问题。延迟会导致只读节点与主库的数据出现不一致，进而可能造成业务上逻辑的混乱或者数据不正确。

最近也收到了很多用户关于只读实例延迟的问题反馈，下面将会分析RDS只读实例出现延迟的几种常见场景，希望能够帮助用户理解和处理只读节点的延迟，更好地使用只读节点：

1. 只读节点规格过小(10%)
2. 主库的TPS过高(20%)
3. 主库的DDL(alter、drop、repair)(40%)
4. 主库大事务(insert..select)(20%)
5. 其他（无主键）(10%)

![screenshot](http://img3.tbcdn.cn/L1/461/1/5970648db169633ea35733b8eed06084e4cb0a3e)

## 场景一：只读实例规格配置过小导致延迟

这类延迟场景的出现往往是主节点购买的一个较大规格的配置，而只读节点却购买了一个最小规格的配置（例如240M内存/150 IOPS）。

分析：只读节点的数据为了和主节点保持同步，采用了MySQL原生的binlog复制技术，由一个IO线程和一个SQL线程来完成，IO线程负责将主库的binlog拉取到只读节点，SQL线程负责消费这些binlog日志，这两个线程会消耗掉只读节点的IO资源，所以当只读节点IOPS配置不够的时候，则会导致只读节点的数据出现延迟：

![screenshot](http://img4.tbcdn.cn/L1/461/1/3a299caa1994347806c9d6e300206dee1c6c3c03)

可以通过只读节点性能监控来判断是否已经达到只读实例的资源配额:

![screenshot](http://img2.tbcdn.cn/L1/461/1/8d27619f9192489c7e692694b9c874a8fdad4593)

![screenshot](http://img3.tbcdn.cn/L1/461/1/7a585ec3a31541f6c99efafffc2fcf3d53804879)

所以当这样的延迟情况的发生的时候，需要用户升级只读实例的规格（可以参考主库此时的IOPS的消耗情况），防止由于只读实例的规格较小导致了数据延迟。

最佳实践：只读实例节点的配置大于或者等于主节点的配置；

## 场景二：主库的TPS过高导致只读节点延迟

这一类的延迟也是非常常见的延迟，由于只读节点与主库的同步采用的是单线程同步，而主库的压力是并发多线程写入，这样势必会导致只读节点的数据延迟，可以通过观察只读节点的TPS与主节点的TPS性能数据来完成判断：

主库的TPS性能数据：

![screenshot](http://img2.tbcdn.cn/L1/461/1/37f6fd7fe01a0737dc451bf42713b613ff259e13)

只读节点的TPS性能数据：

![screenshot](http://img1.tbcdn.cn/L1/461/1/003d57df35c5ce4775be360751856838cbb93fe2)

针对这样场景的延迟，开启只读节点的并行复制是解决这一问题的根本方法，目前RDS生产环境默认开启了并行复制。但是并行复制也不能够彻底解决单表更新的问题，所以用户需要排查业务写入压力是否正常，适当对业务进行优化或者拆分，保证主库的TPS不会导致slave出现延迟。

## 场景三：主库的DDL(alter、drop、repair、create)导致只读节点延迟

这种延迟是非常常见的延迟， 可以分为两类：

第一类：只读节点与主库的DDL同步是串行进行的，如果DDL操作在主库执行时间很长，那么同样在备库也会消耗同样的时间，比如在主库对一张500W的表添加一个字段耗费了10分钟，那么在只读节点上也同样会耗费10分钟，所以只读节点会延迟600S，其他常见操作比如：

`create index，repair table，
alter table add column；
`

范例：只读节点出现延迟

![screenshot](http://img4.tbcdn.cn/L1/461/1/93fd3efa8a8b2d2a4c18316720958237ca7c17ea)

主实例备库同样出现延迟：

![screenshot](http://img4.tbcdn.cn/L1/461/1/bbae1e0aa5a806aac1ecc2ed075d4df702df4f2e)

查看主库这这一段时间是否存在DDL，发现主库在添加索引：

![screenshot](http://img4.tbcdn.cn/L1/461/1/f988134d07f8608402ae680c29f2ce8b777575c0)

第二类：由于只读节点上会有用户的查询在上面运行，所以如果只读节点上有一个执行时间非常长的的查询正在执行，那么这个查询会堵塞来自主库的DDL，直到查询结束为止，进而导致了只读节点的数据延迟。在只读节点上可以通过执行show processlist命令查看连接的状态处于: `Waiting for table metadata lock`

![screenshot](http://img2.tbcdn.cn/L1/461/1/003943f5fa8f8d86e12ed7ef918d61b128c75fa4)

这个时候只需要kill掉只读节点上的大查询就可以恢复只读节点与主节点的数据同步。

## 场景四：主库执行大事务导致延迟

这一种延迟场景也是比较常见的，比如在主库执行一个大的update、delete、insert … select的事务操作，产生大量的binlog传送到只读节点，只读节点需要花费与主库相同的时间来完成该事务操作，进而导致了只读节点的延迟。只读实例发生延迟，在只读节点执行show slave status\G命令，可以通过两个关键的位点参数来判断只读实例上是否在执行大事务：Seconds_Behind_Master不断增加，但是Exec_Master_Log_Pos 却没有发生变化，这样则可以判断只读节点的SQL线程在执行一个大的事务或者DDL操作。

例如下面的例子，用户在主库执行了一条insert … select非常大的插入操作，该操作产生了近几十G的binlog文件传输到只读节点，进而导致了只读节点出现应用binlog延迟：

![screenshot](http://img1.tbcdn.cn/L1/461/1/1cbee1fc9a79dfc581803ae159e01825fd35ba83)

![screenshot](http://img2.tbcdn.cn/L1/461/1/7f0dd91056e48cd4513fc1cab07bbdd5ca1422f8)

针对此类大事务延迟的场景，需要将大事务拆分成为小事务进行排量提交，这样只读节点就可以迅速的完成事务的执行，不会造成数据的延迟。

## 场景五：其他只读实例出现延迟的情况

如对无主键的表进行删除（可以参考[MySQL主键的缺少导致备库hang](https://yq.aliyun.com/articles/9066)），RDS目前已经支持对表添加隐式主键，但是对于以前历史创建的表需要进行重建才能支持隐式主键。

## 总结
综上所述，当只读实例出现延迟后的排查思路：

1. 看只读节点IOPS定位是否存在资源瓶颈；
2. 看只读节点的binlog增长量定位是否存在大事务；
3. 看只读节点的comdml性能指标，对比主节点的commdml定位是否是主库写入压力过高导致；
4. 看只读节点show full processlist，判断是否有Waiting for table metadata lock和alter，repair，create等ddl操作。

**最佳实践**

1. 使用innodb存储引擎；
2. 只读实例的规格不低于主实例；
3. 大事务拆分为小事务；
4. 购买多个只读节点冗余；
5. DDL变更期间观察是否有大查询。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)