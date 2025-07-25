# MySQL ·  引擎特性 ·  MySQL内核对读写分离的支持

**Date:** 2018/01
**Source:** http://mysql.taobao.org/monthly/2018/01/02/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 01
 ](/monthly/2018/01)

 * 当期文章

 MySQL · 引擎特性 · Group Replication内核解析之二
* MySQL · 引擎特性 · MySQL内核对读写分离的支持
* PgSQL · 内核解析 · 同步流复制实现分析
* MySQL · 捉虫动态 · UK 包含 NULL 值备库延迟分析
* MySQL · 捉虫动态 · Error in munmap() "Cannot allocate memory"
* MSSQL · 最佳实践 · 数据库备份链
* MySQL · 捉虫动态 · 字符集相关变量介绍及binlog中字符集相关缺陷分析
* PgSQL · 应用案例 · 传统分库分表(sharding)的缺陷与破解之法
* MySQL · MyRocks · MyRocks参数介绍
* PgSQL · 应用案例 · 惊天性能！单RDS PostgreSQL实例支撑 2000亿

 ## MySQL · 引擎特性 · MySQL内核对读写分离的支持 
 Author: 贤勇 

 ## 读写分离的场景应用

随着业务增长，数据越来越大，用户对数据的读取需求也随之越来越多，比如各种AP操作，都需要把数据从数据库中读取出来，用户可以通过开通多个只读实例，将读请求业务直接连接到只读实例上。使用RDS云数据库的读写分离功能，用户只需要一个请求地址，业务不需要做任何修改，由RDS自带的读写分离中间件服务来完成读写请求的路由及根据不同的只读实例规格进行不同的负载均衡，同时当只读实例出现故障时能够主动摘除，减少对用户的影响。对用户达到一键开通，一个地址，快速使用。
MySQL内核为读写分离的实现提供了支持，包括通过系统variable设置目标节点，session或者是事务的只读属性，等待/检查指定的事务是否已经apply到只读节点上，以及事务状态的实时动态跟踪等的能力。本文会带领大家一起来看看这些特征。说明一下，本文的内容基于RDS MySQL 5.6与RDS MySQL 5.7。

## 只读属性设定
如下的system variables可以将目标节点，session或者是事务设置为只读

### read_only

 支持的MySQL版本
 5.6/5.7

 作用范围
 global

 是否支持动态修改
 是

 默认值
 off

如需设置节点为只读状态，将该read_only参数设置为1或TRUE，但设置 read_only=1 状态有几个需要注意的地方：

1. read_only=1只读模式，不会影响slave同步复制的功能，所以在MySQL slave库中设定了read_only=1后，通过 `show slave status\G` 命令查看salve状态，可以看到salve仍然会读取master上的日志，并且在slave库中应用日志，保证主从数据库同步一致
2. read_only=1只读模式，可以限定普通用户进行数据修改的操作，但不会限定具有super权限的用户的数据修改操作；在MySQL中设置read_only=1后，普通的应用用户进行`insert、update、delete`等会产生数据变化的DML操作时，都会报出数据库处于只读模式不能发生数据变化的错误，但具有super权限的用户，例如在本地或远程通过root用户登录到数据库，还是可以进行数据变化的DML操作；
3. 临时表的操作不受限制
4. log表（mysql.general_log和mysql.slow_log）的插入不受影响
5. Performance Schema表的update，例如update和truncate操作
6. ANALYZE TABLE或者OPTIMIZE TABLE语句
为了让所有的用户都不能进行读写操作，MySQL 5.6就需要执行给所有的表加读锁的命令 `flush tables with read lock\G`，这样使用具有super权限的用户登录数据库，想要发生数据变化的操作时，也会提示表被锁定不能修改的报错，同时，slave的同步复制也会受到影响。

### super_read_only

 支持的MySQL版本
 5.7

 作用范围
 global

 是否支持动态修改
 是

 默认值
 off

在5.7之后，我们可以通过设置这个variable, 使得具有super权限的用户也不能对数据做修改操作，而不必通过`flush tables with read locK\G`的方式了。把super_read_only设置成on， read_only会隐式的被设置成on；反过来，把read_only设置成off，super_read_only就会隐式的被设置成off。

### tx_read_only

 支持的MySQL版本
 5.6/5.7

 作用范围
 global

 是否支持动态修改
 是

 默认值
 off

如果这个variable设置为ON，事务的访问模式就变成了只读，不能对表做更新，但对临时表的更新操作仍然是允许的。设置只读事务在引擎层可以走优化过的逻辑，相比读写事务的开销更小，例如不用分配事务id，不用分配回滚段，不用维护到全局事务链表中。

## 读一致性保证

读写节点之间的数据通常是有gap的，如果有办法知道在主节点上的执行的事务已经被复制到了只读节点，对这（些）事务敏感的读操作就可以被路由到只读节点上，这就是“读一致性”。MySQL 5.6 引入了GTID （Global transaction Identifier)，提升了MySQL节点复制的功能。(关于GTID的详细信息，请参看[云栖文章](https://yq.aliyun.com/articles/68441))

MySQL 5.6提供了`WAIT_UNTIL_SQL_THREAD_AFTER_GTIDS(GTID_SET[,TIMEOUT])`函数来等待从节点把GTID_SET指定事务都执行完毕，除非timeout（以秒为单位）的时间已经耗费而超时。
这个方法存在一些缺点，例如：

1. 该功能依赖于slave来运行，如果复制线程没有启动或者出错了，就会返回错误。在某些情况下我们需要一直等待；
2. 返回的是执行的事件的个数，这通常是没有意义的，返回成功或者失败即可。

MySQL 5.7为解决上面的几个问题，又添加了新的函数 `WAIT_FOR_EXECUTED_GTID_SET\(GTID_SET[,TIMEOUT])`。当`GTID_SUBSET(GTID_SET, @@global.gtid_executed)`成立时，即指定的GTID是gtid_executed的子集时，返回0表示成功，否则返回1，表示失败，如果超时，也会失败。

## 事务精细拆分路由

在MySQL 5.7中，我们可以通过设置`session_track_transaction_info`变量来跟踪事务的状态。在一个负载均衡系统中，你需要知道哪些statement开启或处于一个事务中，哪些statement允许连接分配器调度到另外一个connection。`session_track_transaction_info`是多个字符组成的字符串，各个位置的字符代表特定的状态。比如place 3的字符如果是’R’就代表事务里有一个或者多个的事务性table被读，’_’代表没有这样的table被事务读；place 5的字符如果是’W’就代表事务里有一个或多个的事务性table被写，‘_’就代表没有。前面我们提到的通过system variables设置只读属性的操作，也可能会改变`session_track_transaction_info`的值。
关于MySQL 5.7跟踪事务状态功能的详情请参考 MySQL的[WL文档](https://dev.mysql.com/worklog/task/?spm=5176.100239.blogcont221.10.57969ea05YrWEE&id=6631)。

## 总结

读写分离是MySQL实现负载均衡，保证高可用和高扩展性的重要手段，MySQL内核提供了对读写分离的多种手段的支持，从通过设置系统variable在事务，session，以及节点级别设置只读属性，到通过使用GTID和`WAIT_FOR_EXECUTED_GTID_SET`函数，可以保证只读节点与主几点的读一致性，再到MySQL 5.7事务状态字的方式精细记录，给事务的精细拆分路由提供了更多的支持， RDS的读写分离中间件与MySQL内核有深度的整合，来改善用户体验，提高系统吞吐。

## 参考资料

1. https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.htm1l
2. https://yq.aliyun.com/articles/41155

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)