# MySQL · 捉虫动态· mysql client crash一例

**Date:** 2015/01
**Source:** http://mysql.taobao.org/monthly/2015/01/07/
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

 ## MySQL · 捉虫动态· mysql client crash一例 
 Author: 

 **背景**

客户使用mysqldump导出一张表，然后使用mysql -e 'source test.dmp'的过程中client进程crash，爆出内存的segment fault错误，导致无法导入数据。

**问题定位**

test.dmp文件大概50G左右，查看了一下文件的前几行内容，发现：

`A partial dump from a server that has GTIDs will by default include the GTIDs of all transactions, even those that changed suppressed parts of the database If you don&#039;t want to restore GTIDs pass set-gtid-purged=OFF. To make a complete dump, pass...
-- MySQL dump 10.13 Distrib 5.6.16, for Linux (x86_64)
--
-- Host: 127.0.0.1 Database: carpath
-- ------------------------------------------------------
-- Server version 5.6.16-log
/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
`

问题定位到第一行出现了不正常warning的信息，是由于客户使用mysqldump命令的时候，重定向了stderr。即:

mysqldump …>/test.dmp 2>&1

导致error或者warning信息都重定向到了test.dmp, 最终导致失败。

**问题引申**

问题虽然定位到了，但却有几个问题没有弄清楚：

问题1. 不正常的sql，执行失败，报错出来就可以了，为什么会导致crash？

mysql.cc::add_line函数中，在读第一行的时候，读取到了don't,发现有一个单引号，所以程序死命的去找匹配的另外一个单引号，导致不断的读取文件，分配内存，直到crash。

假设没有这个单引号，mysql读到第六行，发现;号，就会执行sql，并正常的报错退出。

问题2. 那代码中对于大小的边界到底是多少？比如insert语句支持batch insert时，语句的长度多少，又比如遇到clob字段呢？

1. 首先clob字段的长度限制

clob家族类型的column长度受限于max_allowed_packet的大小，MySQL 5.5中，对于max_allowd_packet的大小限制在(1024, 1024*1024*1024)之间。

2. mysqldump导出insert语句的时候，如何分割insert语句？

mysqldump时候支持insert t1 value(),(),();这样的batch insert语句。

mysqldump其实是根据opt_net_buffer_length来进行分割，当一个insert语句超过这个大小，就强制分割到下一个insert语句中，这样更多的是在做网络层的优化。

又如果遇到大的clob字段怎么办？ 如果一行就超过了opt_net_buffer_length，那就强制每一行都分割。

3. mysql client端读取dump文件的时候, 到底能分配多大的内存？

mysql.cc中定义了:

`#define MAX_BATCH_BUFFER_SIZE (1024L * 1024L * 1024L)`

也就是mysql在执行语句的时候，最多只能分配1G大小的缓存。

所以，正常情况下，max_allowed_packet现在的最大字段长度和MAX_BATCH_BUFFER_SIZE限制的最大insert语句，是匹配的。

**RDS问题修复原则**

从问题的定位上来看，这一例crash属于客户错误使用mysqldump导致的问题，Aliyun RDS分支对内存导致的crash问题，都会定位并反馈给用户。 但此例不做修复，而是引导用户正确的使用mysqldump工具。
</div>

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)