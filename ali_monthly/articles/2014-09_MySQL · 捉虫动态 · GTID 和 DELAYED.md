# MySQL · 捉虫动态 · GTID 和 DELAYED

**Date:** 2014/09
**Source:** http://mysql.taobao.org/monthly/2014/09/01/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2014 / 09
 ](/monthly/2014/09)

 * 当期文章

 MySQL · 捉虫动态 · GTID 和 DELAYED
* MySQL · 限制改进 · GTID和升级
* MySQL · 捉虫动态 · GTID 和 binlog_checksum
* MySQL · 引擎差异·create_time in status
* MySQL · 参数故事 · thread_concurrency
* MySQL · 捉虫动态 · auto_increment
* MariaDB · 性能优化 · Extended Keys
* MariaDB · 主备复制 · CREATE OR REPLACE
* TokuDB · 参数故事 · 数据安全和性能
* TokuDB · HA方案 · TokuDB热备

 ## MySQL · 捉虫动态 · GTID 和 DELAYED 
 Author: 

 **描述**

　　这是一个MySQL 5.6才有的bug，影响包含最新版本。涉及到的概念有GTID、DELAYED。

**现象**

　　在5.6主备都开启GTID-MODE的时候，备库同步线程停止，且Last_SQL_Error显示“When @@SESSION.GTID_NEXT is set to a GTID, you must explicitly set it to a different value after a COMMIT or ROLLBACK. Please check GTID_NEXT variable manual page for detailed explanation. Current @@SESSION.GTID_NEXT is … ” 　　

　　查到这个位置正在执行的日志是一个INSERT语句，并且主库上使用的语法是 INSERT DELAYED INTO。 　　

**GTID限制**

　　众所周知，在打开gtid-mode的时候，MySQL不允许执行create table xx as select … 这个语句。其原因是每个GTID编号(gno)需要唯一对应一个事务，而在ROW格式binlog模式下，上述语句会被写成一个create语句和一个insert事务。这样违背唯一对应约束。

**关于DELAYED**

　　往数据库里插入数据的标准命令是INSERT，而DELAYED的意思，则是异步插入。也就是说，MySQL接受这个命令后，保存命令就直接返回给客户端，因此用户会发现在某些场景下INSERT DELAYED性能优于”INSERT，实际上只是更快的返回，而非更快的完成。

　　既然执行线程已经返回给用户，那么这个INSERT任务就是由一个后台线程执行的。这里有一个优化：执行线程每次循环获取现有的任务列表，多个一起执行。

　　这样就可能连续执行N个INSER操作，生成多个INSERT事件。而在生成GTID时，就只对应一个gno。

　　这就违反了上一节提到的GTID限制。

　　这个binlog传到备库后，备库在执行完这个gno对应的第一个事件后，操作表是一个MyISAM表（DELAYED语法只MyISAM引擎支持），自动提交事务，在执行下一个事务时，发现“少了”新的gno，因此报错。

**分析修复**

　　上述bug的根本原因是DELAYED语法生成了违反GTID限制的binlog。实际上这个语法应该也设定为：在GTID模式下禁止。

　　若从减少应用的报错考虑，另一种修复策略是在GTID模式下，自动将INSERT DELAYED转为INSERT。

**DELAYED相关**

　　a) InnoDB不支持DELAYED语法，因为这破坏了事务的原子性和可见性。

　　b) 即使对于MyISAM，官方已经将DELAYED语法在5.6列为deprecated, 在5.7取消。

　　c) 目前能够使用DELAYED的语法有 INSERT DELAYED 和 REPLACE DELAYED。

　　d) DELAYED 命令统一使用ROW格式binlog。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)