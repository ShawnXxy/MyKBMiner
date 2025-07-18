# MySQL · 捉虫动态 · GTID 和 binlog_checksum

**Date:** 2014/09
**Source:** http://mysql.taobao.org/monthly/2014/09/03/
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

 ## MySQL · 捉虫动态 · GTID 和 binlog_checksum 
 Author: 

 **现象描述**

　　在5.6主备环境下，主备都开启GTID-MODE，备库开启crc校验，主库不开。重启备库sql线程后，备库sql线程停止Last_Error显示：Relay log read failure: Could not parse relay log event entry. The possible reasons are: the master’s binary log is corrupted(you can check this by running ‘mysqlbinlog’ on the binary log), the slave’s relay log is corrupted (you can check this by running ‘mysqlbinlog’ on the relay log), a network problem, or a bug in the master’s or slave’s MySQL code. If you want to check the master’s binary log or slave’s relay log, you will be able to know their names by issuing ‘SHOW SLAVE STATUS’ on this slave.

　　从错误信息可以看出，可能是主库的binlog或备库的relaylog出错。

**关于GTIDs**

　　详见上一章节：MySQL·　限制改进·GTID和升级

**binlog头部信息**

　　FORMAT_DESCRIPTION：binlog格式信息，备库解析binlog的标准。

　　PREVIOUS_GTIDS_LOG：已产生的GTID集合，防止重复记录binlog。

　　ROTATE：备库binlog切换到主库binlog的转换标志。

**PREVIOUS_GTIDS_LOG**

　　开启GTID_MODE时，每个binlog文件的头部会有一个PREVIOUS_GTIDS_LOG，用于保存已产生的GTID。MySQL源码中的Gtid_set类用于实现这个功能，内部由链表实现，链表的每个节点保存了一个区间，用于指代一段连续的GNO。

**分析和修复**

　　在上述环境下，备库relay log的前几条应该是：
`
FORMAT_DESCRIPTION_EVENT (of slave)
PREVIOUS_GTIDS_LOG_EVENT (of slave)
ROTATE_EVENT (of master)
FORMAT_DESCRIPTION_EVENT (of master)
`　
 之前备库选取FORMAT 的策略是：先根据文件头备库的FORMAT_DESCRIPTION_EVENT确定FORMAT，然后继续向下读；

　　如果读到FORMAT_DESCRIPTION_EVENT，则更新FORMAT；如果读到ROTATE_EVENT，则继续向下读；

　　如果读到一条非FORMAT_DESCRIPTION_EVENT或ROTATE_EVENT的log，则停止更新FORMAT，选取当前FORMAT解析后面的log。

　　由备库前几条relay log可知，读到第二条PREVIOUS_GTIDS_LOG_EVENT时，已由备库的FORMAT_DESCRIPTION_EVENT确定FORMAT(binlog_checksum=on)，而略过主库的FORMAT_DESCRIPTION_EVENT。

　　到下面解析log时，会认为每条log尾部有crc校验信息。但校验信息实际是不存在的，所以会报crc校验的错误。

　　当读到PREVIOUS_GTIDS_LOG_EVENT时继续向下读，即可读到主库的FORMAT_DESCRIPTION_EVENT，解决这个bug。

**其他复现场景**

　　5.5/5.1会作为5.6的主库，此时备库开启GTID-MODE和crc校验。若中间出现主键冲突等错误，sql thread暂停后， 执行start slave，会报错 “Event crc check failed”。原因是5.5/5.1不支持crc校验，和5.6不开启crc校验相似。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)