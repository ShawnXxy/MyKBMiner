# MySQL ·  引擎特性 ·  DROP TABLE之binlog解析

**Date:** 2017/11
**Source:** http://mysql.taobao.org/monthly/2017/11/02/
**Images:** 2 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 11
 ](/monthly/2017/11)

 * 当期文章

 MySQL · 数据恢复 · undrop-for-innodb
* MySQL · 引擎特性 · DROP TABLE之binlog解析
* MSSQL · 最佳实践 · SQL Server三种常见备份
* MySQL · 最佳实践 · 什么时候该升级内存规格
* MySQL · 源码分析 · InnoDB LRU List刷脏改进之路
* MySQL · 特性分析 · MySQL 5.7 外部XA Replication实现及缺陷分析
* PgSQL · 最佳实践 · 双十一数据运营平台订单Feed数据洪流实时分析方案
* MySQL · 引擎特性 · TokuDB hot-index机制
* MySQL · 最佳实践 · 分区表基本类型
* PgSQL · 应用案例 · 流式计算与异步消息在阿里实时订单监测中的应用

 ## MySQL · 引擎特性 · DROP TABLE之binlog解析 
 Author: 智邻 

 ## Drop Table的特殊之处

Drop Table乍一看，与其它DDL 也没什么区别，但当你深入去研究它的时候，发现还是有很多不同。最明显的地方就是DropTable后面可以紧跟多个表，并且可以是不同类型的表，这些表还不需要显式指明其类型，比如是普通表还是临时表，是支持事务的存储引擎的表还是不支持事务的存储引擎的表等。这些特殊之处对于代码实现有什么影响呢？对于普通表，无论是创建还是删除，数据库都会产生相应的binlog日志，而对于临时表来说，记录binlog日志就不是必须的。对于采用不同存储引擎的表来说，更是如此，有些存储引擎不支持事务如MyISAM，而有些存储引擎支持事务如Innodb，对于支持事务和不支持事务的存储引擎，处理方式也有些许差异。而Drop Table可以跟多种不同类型的表，就必须对这些情况分类处理。因此有必要对MySQL的DROP TABLE实现进行更深入的研究，以了解个中不同之处，防止被误解误用。

## MySQL中Drop Table不支持事务

MySQL中对于DDL本身的实现与其它数据库也存在一些不同，比如无论存储引擎是什么，支持事务Innodb或是不支持事务MyISAM，MySQL的DDL都不支持事务，也不能被包含在一个长事务中，即使用begin/end或start transaction/commit包含多条语句的事务。如果在长事务中出现DDL，则在执行DDL之前，数据库会自动将DDL之前的事务提交。Drop Table可以同时删除多个表，这些表可能存在，也可能不存在。如果删除列表中的某个表不存在，数据库仍会继续删除其它存在的表，但最终会输出一条表不存在的错误消息。如要删除t1,t2,t3,t4,t5，则t1,t2,t5表存在，t3,t4表不存在，则语句Drop Table t1,t2,t3,t4,t5;会删除t1,t2,t5,然后返回错误：ERROR 1051 (42S02): Unknown table ‘test.t3,test.t4’而在其它数据库中，比如PostgreSQL，就会将事务回滚，不会删除任何一张表。

## Drop Table如何记录binlog？

在MySQL中，通过binlog进行主备之间的复制，保证主备节点间的数据一致，对于Drop table又有什么不同吗？仔细研究一下，还真的有很大的不同。MySQL支持两种binlog格式，statement和row，实践中还有一种是两者混合格式mixed。不同的binlog格式对SQL语句的binlog产生也会有不同的影响，尤其对Drop table来说，因为Drop table有很多之前提到的特殊之处，如可能同时删除多个不同类型的表，甚至删除不存在的表，因此在产生binlog时必须对这些不同类型的表或者不存在的表进行特殊的处理。

### 不存在表的处理
对于不存在表，实际上也没有表的定义, MySQL将其统一认作普通表，并按普通表来记录binlog。如Drop table if exists t1, t2,t3; 其中t1,t3存在，t2不存在；则会产生binlog如下所示：DROP TABLE IF EXISTS `t1`,`t2`,`t3`;

### 临时表的处理

此外影响最大的就是对临时表的处理，在statement格式下，所有对临时表的操作都要记录binlog，包括创建、删除及DML语句；但在row格式下，只有Drop table才会记录binlog，而对临时表的创建及DML语句是不记录binlog的。为什么会这样？通常情况下，主机的临时表在备机上是没有用的，临时表只在当前session中有效，即使将临时表同步到备机，当用户从主机切换到备机时，原来session已经中断，与session关联的临时表也会被清除，用户会重建session到新的主机。但在一些特殊情况下，还是需要将主机的临时表同步到备机的，比如主机上执行insert into t1 select * from temp1，其中t1是普通表，而temp1是临时表。当binlog格式为statement时，这条语句会被记录到binlog，然后同步到备机，在备机上replay，若备机之前没有将主机上的临时表同步过来，那这条语句的replay就会出现问题。因此在statement格式下，对临时表的操作如创建、删除及其它DML语句都必须记录binlog，然后同步到备机执行replay。但在row格式下，因为binlog中已经记录了实际的row，那么对临时表的创建、DML语句是不是记录binlog就不是那么重要了。这里有一点比较特殊，对临时表的删除还是要记录binlog。因为用户可以随时修改binlog的格式，若之前创建临时表时是statement格式，而创建成功后，又修改为row格式，若row格式下删除表不记录binlog，那么在备机上就会产生问题，创建了临时表，但却没有删除它。因此对drop table语句，无论binlog格式采用statement或是row格式，都会记录binlog。而对于创建临时表语句，只有statement格式会记录binlog，而在row格式下，不记录binlog。为防止row格式下在备机上replay时drop不存在的临时表，会将drop临时表的binlog中添加IF EXISTS，防止删除不存在的表replay失败。

### 不同类型表的处理

另外，drop table在产生binlog还有一个诡异的地方，通常一条SQL语句只会产生一个binlog event，占用一个gitd_executed，但drop table有可能会产生多个binlog event，并占用多个gtid_executed。如下示例：DROP TABLE t1, tmp1, i1, no1;其中t1为普通表，tmp1为innodb引擎的临时表，i1为MyISAM引擎的临时表，no1为不存在的表。则会产生3条binlog events，并且每个binlog events都有自己的gtid_executed。如下所示：
![binlog.png](.img/c427ce35ba43_2c28aa7c193a94643a27ea63f85826fd.png)

## 总结

由于历史原因，MySQL支持多种存储引擎，也支持多种复制模式，binlog的格式也从statement一种发展到现在的statement、row和mixed三种，为了兼容不同的存储引擎和不同的复制模式，在代码实现上做了很多折衷，这也要求我们要了解历史、了解未来，只有这样才能更好的使用、改进MySQL，为用户提供更好的云服务体验。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)