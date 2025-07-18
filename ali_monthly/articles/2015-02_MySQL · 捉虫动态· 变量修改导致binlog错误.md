# MySQL · 捉虫动态· 变量修改导致binlog错误

**Date:** 2015/02
**Source:** http://mysql.taobao.org/monthly/2015/02/07/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 02
 ](/monthly/2015/02)

 * 当期文章

 MySQL · 性能优化· InnoDB buffer pool flush策略漫谈
* MySQL · 社区动态· 5.6.23 InnoDB相关Bugfix
* PgSQL · 特性分析· Replication Slot
* PgSQL · 特性分析· pg_prewarm
* MySQL · 答疑释惑· InnoDB丢失自增值
* MySQL · 答疑释惑· 5.5 和 5.6 时间类型兼容问题
* MySQL · 捉虫动态· 变量修改导致binlog错误
* MariaDB · 特性分析· 表/表空间加密
* MariaDB · 特性分析· Per-query variables
* TokuDB · 特性分析· 日志详解

 ## MySQL · 捉虫动态· 变量修改导致binlog错误 
 Author: 

 **背景**

MySQL 5.6.6 版本新加了这样一个参数——log_bin_use_v1_row_events，这个参数用来控制binlog中Rows_log_event的格式，如果这个值为1的话，就用v1版的Rows_log_event格式（即5.6.6之前的），默认是0，用新的v2版本的格式，更详细看[官方文档](http://dev.mysql.com/doc/refman/5.6/en/replication-options-binary-log.html#sysvar_log_bin_use_v1_row_events)。这个参数一般保持默认即可，但是当我们需要搭 5.6->5.5 这要的主备的时候，就需要把主库的这个值改为1，不然5.5的备库不能正确解析Rows_log_event。最近在使用这个参数的时候发现了一个bug，导致主库binlog写坏，备库复制中断，报错如下：

Last_SQL_Errno: 1594 Last_SQL_Error: Relay log read failure: Could not parse relay log event entry. The possible reasons are: the master's binary log is corrupted (you can check this by running 'mysqlbinlog' on the binary log), the slave's relay log is corrupted (you can check this by running 'mysqlbinlog' on the relay log), a network problem, or a bug in the master's or slave's MySQL code. If you want to check the master's binary log or slave's relay log, you will be able to know their names by issuing 'SHOW SLAVE STATUS' on this slave.

**bug 分析**

binlog event 结构

`event header
event body
-fixed part(postheader)
-variable part (payload)
`

如上所示，每种binlog event 都可以分为header 和 body 2部分，body 又可以分为 fixed part 和 variable part，其中event header的长度相同并且固定，5.0开始用的v4格式的binlog，其event header固定长度为19字节，包含多个字段，具体每个字段的含义可以看[这里](http://dev.mysql.com/doc/internals/en/event-structure.html)。 event body 中post header 长度也是固定的，所以叫 fixed part，但是不同类型event这一部分的长度不一样，最后的 variable part 就是event的主体了，这个就长度不一了。 log_bin_use_v1_row_events 这个值的不同，影响的部分就是 postheader 这里的长度，如果值为1的话，用v1格式，postheader 长度是8个字节，如果是0，用v2格式，其长度为10。每个Rows_log_event的event header的type字节会标明当前event是v1还是v2，试想一下，如果event header部分标明是v2，postheader却实际上只有8个字节，或者反过来，event header部分标明是v1，postheader有10个字节，备库拿到这样的binlog，去尝试解析的时候，就完全凌乱了。

为啥会出现这种一个event前后不一致的情况呢，代码编写不严谨！

 在写 Rows_log_event(Write/Update/Delete) 过程中，有2次用到 log_bin_use_v1_row_events 这个全局变量，一次是在构造函数处，一次是在写postheader时 Rows_log_event::write_data_header()，2次都是直接使用，如果正好在这2次中间，我们执行 set global log_bin_use_v1_row_events = 0
 1，改变原来的值，就会导致前后逻辑判断结果不一致。如果主库有频繁的更新操作，每次更新又比较大，只要修改这个值，就很容易触发这个bug。

另外官方还有点不严谨的是，[文档上](http://dev.mysql.com/doc/refman/5.6/en/replication-options-binary-log.html#sysvar_log_bin_use_v1_row_events)说这个值是 readonly的，实际代码是dynamic 的，如果是 readonly 的话，也就不会触发上面的bug了。

**bug修复**

修复很简单，把2次引用全局变量改成一次就好了，在Rows_log_event::write_data_header函数里直接使用已经保存的m_type，改法如下

`- if (likely(!log_bin_use_v1_row_events))
+
+
+ if (likely(!(m_type == WRITE_ROWS_EVENT_V1 ||
+ m_type == UPDATE_ROWS_EVENT_V1 ||
+ m_type == DELETE_ROWS_EVENT_V1 )))
`

这样改之后，就只会在构造函数中才用到全局变量。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)