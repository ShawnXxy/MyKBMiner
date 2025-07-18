# MySQL · 捉虫动态 · SIGHUP 导致 binlog 写错

**Date:** 2014/11
**Source:** http://mysql.taobao.org/monthly/2014/11/02/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2014 / 11
 ](/monthly/2014/11)

 * 当期文章

 MySQL · 捉虫动态 · OPTIMIZE 不存在的表
* MySQL · 捉虫动态 · SIGHUP 导致 binlog 写错
* MySQL · 5.7改进 · Recovery改进
* MySQL · 5.7特性 · 高可用支持
* MySQL · 5.7优化 · Metadata Lock子系统的优化
* MySQL · 5.7特性 · 在线Truncate undo log 表空间
* MySQL · 性能优化 · hash_scan 算法的实现解析
* TokuDB · 版本优化 · 7.5.0
* TokuDB · 引擎特性 · FAST UPDATES
* MariaDB · 性能优化 · filesort with small LIMIT optimization

 ## MySQL · 捉虫动态 · SIGHUP 导致 binlog 写错 
 Author: 

 **bug描述**

这是5.6中和gtid相关的一个bug，当 mysqld 收到 sighup 信号 （比如 kill -1） 的时候，会 flush binlog，但是新生成binlog开头没写 Previous_gtids_log_event，这会导致下面 2 个问题：

1. 这个时候 mysqld 重启的话，会发现再也起不来了，error log 里有这样的错
The binary log file ‘mysql/mysql-bin.000020’ is logically corrupted: The first global transaction identifier was read, but no other information regarding identifiers existing on the previous log files was found.
2. 这个时候主库继续更新，然后从库来拉取 binlog 的时候，io 线程会停下来
Last_IO_Error: Got fatal error 1236 from master when reading data from binary log: ‘Error reading header of binary log while looking for the oldest binary log that contains any GTID that is not in the given gtid set’

**bug 分析**

mysqld 在收到 sighup 信号后，signal_hand 线程会调用 reload_acl_and_cache 函数 (sql_reload.cc)，最终会调用 MYSQL_BIN_LOG::open_binlog，open_binlog 有这段逻辑:

`if (current_thd && gtid_mode > 0)
{
if (need_sid_lock)
global_sid_lock->wrlock();
else
global_sid_lock->assert_some_wrlock();
Previous_gtids_log_event prev_gtids_ev(previous_gtid_set);
if (need_sid_lock)
global_sid_lock->unlock();
prev_gtids_ev.checksum_alg= s.checksum_alg;
if (prev_gtids_ev.write(&log_file))
goto err;
bytes_written+= prev_gtids_ev.data_written;
}
`
signal_hand 没有调用 store_globals 设置 THR_THD 这个key，所以这个时候 current_thd 得到的值是空的，因此prev_gtids_event 也就不会写进新binlog中的。

2个问题的分析

mysqld 重启不起来的原因：
mysqld 在启动的时候会通过 mysql_bin_log.init_gtid_sets 来初始化 gtid_executed 和 gtid_purged 2个set，初使化 gtid_executed 时，会读最新的binlog，将文件开头 Previous_gtids_log_event 的 gtid set 和文件里所有的 gtid_event 加起来，放进 gtid_executed，在读文件过程中，如果发现没有 Previous_gtids_log_event ，就报错，程序退出。

备库的错误信息解释：
在gtid协议下，主库向备库发 binlog 是用 com_binlog_dump_gtid 函数，这个函数会调到 MYSQL_BIN_LOG::find_first_log_not_in_gtid_set()，这个函数的作用是找到备库需要的第一个 binlog 文件，逻辑是这样的，从编号最大的binlog 往前找，对每个binlog，读取 Previous_gtids_log_event，如果发现这个集合是备库的发来的 gtid_set 的子集，就停止，当前这个binlog文件就是备库需要的第一个binlog文件。找的过程中，如果发现没有 Previous_gtids_log_event，就把错误信息 ER_MASTER_FATAL_ERROR_READING_BINLOG 发给备库。

**问题的解决方法**

对server 起不来的，只能手动删所有 binlog 文件了，同时还要清空 binlog.index 文件，有备库的话要重搭备库。
对于主备场景下，备库停掉的，purge 主库的binlog，如果主备不致的话，比如主库sighup后又有新的更新，这时候需要重做备库，因为binlog已经没了，只能拿主库的数据来重新做一个。

**bug 修复**

这个bug官方已经修复，具体可以参考 revno: 5908。

修复方法类似reload_acl_and_cache 中 REFRESH_GRANT 的逻辑，生成一个临时的 THD 作为 current_thd，在flush logs 完后释放掉。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)