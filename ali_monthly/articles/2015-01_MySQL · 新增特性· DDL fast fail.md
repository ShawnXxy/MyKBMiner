# MySQL · 新增特性· DDL fast fail

**Date:** 2015/01
**Source:** http://mysql.taobao.org/monthly/2015/01/02/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 01
 ](/monthly/2015/01)

 * 当期文章

 MySQL · 性能优化· Group Commit优化
* MySQL · 新增特性· DDL fast fail
* MySQL · 性能优化· 启用GTID场景的性能问题及优化
* MySQL · 捉虫动态· InnoDB自增列重复值问题
* MySQL · 优化改进· 复制性能改进过程
* MySQL · 谈古论今· key分区算法演变分析
* MySQL · 捉虫动态· mysql client crash一例
* MySQL · 捉虫动态· 设置 gtid_purged 破坏AUTO_POSITION复制协议
* MySQL · 捉虫动态· replicate filter 和 GTID 一起使用的问题
* TokuDB·特性分析· Optimize Table

 ## MySQL · 新增特性· DDL fast fail 
 Author: 

 **背景**

项目的快速迭代开发和在线业务需要保持持续可用的要求，导致MySQL的ddl变成了DBA很头疼的事情，而且经常导致故障发生。本篇介绍RDS分支上做的一个功能改进，DDL fast fail。主要解决：DDL操作因为无法获取MDL排它锁，进入等待队列的时候，阻塞了应用所有的读写请求问题。

**MDL锁机制介绍**

首先介绍一下MDL(METADATA LOCK)锁机制，MySQL为了保证表结构的完整性和一致性，对表的所有访问都需要获得相应级别的MDL锁，比如以下场景：

`session 1: start transaction; select * from test.t1;
session 2: alter table test.t1 add extra int;
session 3: select * from test.t1;
`

session 1对t1表做查询，首先需要获取t1表的MDL_SHARED_READ级别MDL锁。锁一直持续到commit结束，然后释放。

session 2对t1表做DDL，需要获取t1表的MDL_EXCLUSIVE级别MDL锁，因为MDL_SHARED_READ与MDL_EXCLUSIVE不相容，所以session 2被session 1阻塞，然后进入等待队列。

session 3对t1表做查询，因为等待队列中有MDL_EXCLUSIVE级别MDL锁请求，所以session3也被阻塞，进入等待队列。

这种场景就是目前因为MDL锁导致的很经典的阻塞问题，如果session1长时间未提交，或者查询持续过长时间，那么后续对t1表的所有读写操作，都被阻塞。 对于在线的业务来说，很容易导致业务中断。

**aliyun RDS分支改进**

DDL fast fail并没有解决真正DDL过程中的阻塞问题，但避免了因为DDL操作没有获取锁，进而导致业务其他查询/更新语句阻塞的问题。

其实现方式如下:

alter table test.t1 no_wait/wait 1 add extra int;

在ddl语句中，增加了no_wait/wait 1语法支持。

其处理逻辑如下:

首先尝试获取t1表的MDL_EXCLUSIVE级别的MDL锁:

当语句指定的是no_wait，如果获取失败，客户端将得到报错信息：ERROR : Lock wait timeout exceeded; try restarting transaction。

当语句指定的是wait 1，如果获取失败，最多等待1s，然后得到报错信息：ERROR : Lock wait timeout exceeded; try restarting transaction。

另外，除了alter语句以外，还支持rename，truncate，drop，optimize，create index等ddl操作。

**与Oracle的比较**

在Oracle 10g的时候，DDL操作经常会遇到这样的错误信息：

ora-00054:resource busy and acquire with nowait specified

即DDL操作无法获取表上面的排它锁，而fast fail。

其实DDL获取排他锁的设计，需要考虑的就是两个问题：

1. 雪崩

如果你采用排队阻塞的机制，那么DDL如果长时间无法获取锁，就会导致应用的雪崩效应，对于高并发的业务，也是灾难。

2. 饿死

如果你采用强制式的机制，那么要防止DDL一直无法获取锁的情况，在业务高峰期，可能DDL永远无法成功。

在Oracle 11g的时候，引入了DDL_LOCK_TIMEOUT参数，如果你设置了这个参数，那么DDL操作将使用排队阻塞模式，可以在session和global级别设置， 给了用户更多选择。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)