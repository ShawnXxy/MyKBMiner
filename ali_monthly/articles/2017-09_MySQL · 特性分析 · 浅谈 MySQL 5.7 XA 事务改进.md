# MySQL · 特性分析 · 浅谈 MySQL 5.7 XA 事务改进

**Date:** 2017/09
**Source:** http://mysql.taobao.org/monthly/2017/09/05/
**Images:** 2 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 09
 ](/monthly/2017/09)

 * 当期文章

 POLARDB · 新品介绍 · 深入了解阿里云新一代产品 POLARDB
* HybridDB · 最佳实践 · 阿里云数据库PetaData
* MySQL · 捉虫动态 · show binary logs 灵异事件
* MySQL · myrocks · myrocks之Bloom filter
* MySQL · 特性分析 · 浅谈 MySQL 5.7 XA 事务改进
* MySQL · 特性分析 · 利用gdb跟踪MDL加锁过程
* MySQL · 源码分析 · Innodb 引擎Redo日志存储格式简介
* MSSQL · 应用案例 · 日志表设计优化与实现
* PgSQL · 应用案例 · 海量用户实时定位和圈人-团圆社会公益系统
* MySQL · 源码分析 · 一条insert语句的执行过程

 ## MySQL · 特性分析 · 浅谈 MySQL 5.7 XA 事务改进 
 Author: 荣生 

 ## 关于MySQL XA 事务
MySQL XA 事务通常用于分布式事务处理当中。比如在分库分表的场景下，当遇到一个用户事务跨了多个分区，需要使用XA事务 来完成整个事务的正确的提交和回滚，即保证全局事务的一致性。

## XA 事务在分库分表场景的使用
下图是个典型的分库分表场景，前端是一个Proxy后面带若干个MySQL实例，每个实例是一个分区。

![XA-sharding.png](.img/29cb190f8de0_a09ea4a6e20928a84b5893a87e993be5.png)

假设一个表test定义如下，Proxy根据主键”a”算Hash决定一条记录应该分布在哪个节点上：

`create table test(a int primay key, b int) engine = innodb;
`

应用发到Proxy的一个事务如下：

`begin;
insert into test values (1, 1);
update test set b = 1 where a = 10;
commit;
`

Proxy收到这个事务需要将它转成XA事务发送到后端的数据库以保证这个事务能够安全的提交或回滚，一般的Proxy的处理步骤 如下：

1. Proxy先收到begin，它只需要设置一下自己的状态不需要向后端数据库发送
2. 当收到 insert 语句时Proxy会解析语句，根据“a”的值计算出该条记录应该位于哪个节点上，这里假设是“分库1”
3. Proxy就会向分库1上发送语句xa start ‘xid1’，开启一个XA事务，这里xid1是Proxy自动生成的一个全局事务ID；同时原来 的insert语句insert into values(1,1)也会一并发送到分库1上。
4. 这时Proxy遇到了update语句，Proxy会解析 where条件主键的值来决定该条语句会被发送到哪个节点上，这里假设是“分库2”
5. Proxy就会向分库2上发送语句xa start ‘xid1’，开启一个XA事务，这里xid1是Proxy之前已经生成的一个全局事务ID；同时原来 的update语句update test set b = 1 where a = 10也会一并发送到分库2上。
6. 最后当Proxy解析到commit语句时，就知道一个用户事务已经结束了，就开启提交流程
7. Proxy会向分库1和分库2发送 xa end ‘xid1’;xa prepare ‘xid1’语句，当收到执行都成功回复后，则继续进行到下一步，如果任何一个分 库返回失败，则向分库1和分库2 发送 xa rollback ‘xid1’，回滚整个事务
8. 当 xa prepare ‘xid1’都返回成功，那么 proxy会向分库1和分库2上发送 xa commit ‘xid1’，来最终提交事务。

这里有一个可能的优化，即在步骤4时如果Proxy计算出update语句发送的节点仍然是“分库1”时，在遇到commit时，由于只涉 及到一个分库，它可以直接向“分库1”发送 xa end ‘xid1’; xa commit ‘xid1’ one phase来直接提交该事务，避免走 prepare阶段来提高效率。

## XA对事务安全的影响分析

从以上分库分表场景下分布式事务的处理过程来看，整个分布式事务的安全性依赖是XA Prepare了的事务的可靠性，也就是在 数据库节点上 XA Prepare了的事务必须是持久化了的，这样当XA Commit发来时才可以提交。设想如下场景：

1. Proxy已经向分库1和分库2上发送完了 xa prepare ‘xid1’语句，并得到了成功的回复
2. Proxy向分库1上发送了 ‘xa commit ‘xid1’语句，并已经成功返回
3. 当 Proxy向分库2上发送 ‘xa commit ‘xid1’时，网络断开了，或者分库2的数据库实例被kill了
4. 当网络恢复（这时相关的Session已经退出了）或数据库实例再启动后（或切换到备库），XA prepare了的事务已经回滚了， 当Proxy XA commit ‘xid1’发过来后数据库实例根本找不到xid1这个xa事务

上面的过程就导致了分布式事务的不一致：分库1提交了事务，分库2回滚了事务，整个事务提交了一半，回滚了一半。

在MySQL 5.6中以上过程是可能发生的，因为xa prepare并没有严格的持久化，当Session断开，数据库崩溃等情况下这些事务 会被回滚掉，而且的当一个主库配置了SemiSync的备库时xa prepare了的事务也不会被发送的备库，如果主库切换到备库这些 事务也会丢失。

## MySQL 5.7 XA可靠性改进

MySQL 5.7解决了 xa prepare了的事务的严格持久化问题，也就是在session断开和实例崩溃重启情况下这些事务不丢，同时在 xa prepare ‘xid1’返回之前XA事务也会同步到备库。下面将通过在5.6和5.7上分别执行xa prepare并对binlog event进行分析 来演示这个改进。

### 断开连接对xa prepare的事务影响

在5.6和5.7上分别执行如下sql然后断开连接，再重新连接使用的xa recover验证 XA 事务是否回滚了。

`xa start 'xid1';
insert into test values(1, 1);
xa end 'xid1';
xa prepare 'xid1';
-- 这里断开再连上新连接执行 xa recover
`

在 5.6 的版本上将返回空的结果，在 5.7 的版本上返回：

`mysql> xa recover;
+----------+--------------+--------------+------+
| formatID | gtrid_length | bqual_length | data |
+----------+--------------+--------------+------+
| 1 | 4 | 0 | xid1 |
+----------+--------------+--------------+------+
1 row in set (0.00 sec)
`

说明断开连接后 5.7的prepare了的xa事务没有丢失。

### XA 事务的 Binlog events 异同

在5.6和5.7上分别执行如下事务，然后用 show binlog events 查看两者binlog的不同：

`xa start 'xid1';
insert into test values(1, 1);
xa end 'xid1';
xa prepare 'xid1';
xa commit 'xid1';
`

5.6的结果：

`mysql-bin.000001 | 304 | Gtid | 3706 | 352 | SET @@SESSION.GTID_NEXT= 'uuid:2'
mysql-bin.000001 | 352 | Query | 3706 | 424 | BEGIN
mysql-bin.000001 | 424 | Table_map | 3706 | 472 | table_id: 71 (test.test)
mysql-bin.000001 | 472 | Write_rows | 3706 | 516 | table_id: 71 flags: STMT_END_F
mysql-bin.000001 | 516 | Query | 3706 | 589 | COMMIT
`

5.7的结果：

`mysql-bin.000001 | 544 | Gtid | 3707 | 592 | SET @@SESSION.GTID_NEXT= 'uuid:3'
mysql-bin.000001 | 592 | Query | 3707 | 685 | XA START X'78696431',X'',1
mysql-bin.000001 | 685 | Table_map | 3707 | 730 | table_id: 74 (test.t) 
mysql-bin.000001 | 730 | Write_rows | 3707 | 774 | table_id: 74 flags: STMT_END_F
mysql-bin.000001 | 774 | Query | 3707 | 865 | XA END X'78696431',X'',1 
mysql-bin.000001 | 865 | XA_prepare | 3707 | 905 | XA PREPARE X'78696431',X'',1
mysql-bin.000001 | 905 | Gtid | 3707 | 953 | SET @@SESSION.GTID_NEXT= 'uuid:4' |
mysql-bin.000001 | 953 | Query | 3707 | 1047 | XA COMMIT X'78696431',X'',1
`

可以看到 MySQL 5.6 XA 事务和普通事务的binlog是一样的，并没有体现 xa prepare。而到了 MySQL 5.7 XA 事务的binlog和 普通的事务是完全不同的，XA Prepare有单独的Log event类型，有自己的Gtid，当开启semi-sync的情况下，MySQL 5.7 执行 XA prepare 时会等备库回复后才返回结果给客户端，这样XA prepare执行完就是安全的。

通过以上分析可以看出 MySQL 5.7在XA事务安全性方面做了很大的改进，后续月报文章将会对它的实现做分析。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)