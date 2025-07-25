# MySQL · 捉虫动态 · show binary logs 灵异事件

**Date:** 2017/09
**Source:** http://mysql.taobao.org/monthly/2017/09/03/
**Images:** 1 images downloaded

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

 ## MySQL · 捉虫动态 · show binary logs 灵异事件 
 Author: fungo 

 ## 问题背景

最近在运维 MySQL 中遇到一个神奇的问题，分享给大家。现象是这样的，`show binary logs` 没有返回结果，`flush binary logs` 后也不行，
但是 binlog 是正常工作的，`show master staus` 是有输出的。

`mysql> show binary logs;
Empty set (0.00 sec)

mysql> show master status\G
*************************** 1. row ***************************
 File: master-bin.000004
 Position: 120
 Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)

mysql> show binary logs;
Empty set (0.00 sec)

mysql> show master status\G
*************************** 1. row ***************************
 File: master-bin.000004
 Position: 120
 Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)

mysql> flush binary logs;
Query OK, 0 rows affected (0.01 sec)

mysql> show binary logs;
Empty set (0.00 sec)

mysql> show master status\G
*************************** 1. row ***************************
 File: master-bin.000005
 Position: 120
 Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)

mysql>
`

## 问题排查

这个问题是笔者第一次遇到，从问题的现象看，binlog 是没有问题的，可以正常写入和切换，只是 show 命令看不到 binlog 文件列表，我们知道 MySQL 是用一个 index 元文件来维护当前使用的 binlog 的，而 `show binary logs` 也是读这个文件来展示的，因此问题应该出在 index 文件上。

我们首先检查 index 文件的权限，发现也是没问题的，mysqld 进程用户是有读写权限的，然后我们用 `tail -f` 命令监控 index 文件，另一个窗口连接mysql，执行 `flush binary logs`，发现新产生的 binlog 文件也是会追加到 index 里。越排查越觉得诡异，并且没有排查下去的思路了，难道是 `show binary logs` 逻辑有问题，翻开代码确认了下，主体逻辑非常简单，就是从 index 文件头开始遍历，一行对应一个 binlog 文件，每一个 binlog 文件都获取下文件size，然后把结果发给客户端，详见 `rpl_master.cc:show_binlogs()`:

` reinit_io_cache(index_file, READ_CACHE, (my_off_t) 0, 0, 0);

 /* The file ends with EOF or empty line */
 while ((length=my_b_gets(index_file, fname, sizeof(fname))) > 1)
 {
 int dir_len;
 ulonglong file_length= 0; // Length if open fails
 fname[--length] = '\0'; // remove the newline

 protocol->prepare_for_resend();
 dir_len= dirname_length(fname);
 length-= dir_len;
 protocol->store(fname + dir_len, length, &my_charset_bin);

 if (!(strncmp(fname+dir_len, cur.log_file_name+cur_dir_len, length)))
 file_length= cur.pos; /* The active log, use the active position */
 else
 {
 /* this is an old log, open it and find the size */
 if ((file= mysql_file_open(key_file_binlog,
 fname, O_RDONLY | O_SHARE | O_BINARY,
 MYF(0))) >= 0)
 {
 file_length= (ulonglong) mysql_file_seek(file, 0L, MY_SEEK_END, MYF(0));
 mysql_file_close(file, MYF(0));
 }
 }
 protocol->store(file_length);
 if (protocol->write())
 {
 DBUG_PRINT("info", ("stopping dump thread because protocol->write failed at line %d", __LINE__));
 goto err;
 }
 }

`

代码逻辑看起来没毛病，心想这问题真是神奇了。。。笔者都准备掏出 gdb 一步一步跟代码看了，在此之前抱着试试看的心态，用 vim 打开了 index 文件，准备人肉看一遍，一打开就发现了可疑的地方，文件内容如下：

`
./master-bin.000001
./master-bin.000002
./master-bin.000003
./master-bin.000004
./master-bin.000005
`

有没有看出什么？

细心的读者可能已经发现，第一行是空行，再看下刚的代码，有这么一个判断逻辑:

` /* The file ends with EOF or empty line */
 while ((length=my_b_gets(index_file, fname, sizeof(fname))) > 1)
`

空行被认为是文件结束，WTF！心中万头羊驼奔腾。

解法很简单，删了第一行的空行，然后 `flush binary logs` 生成新的 index 文件把 cache 失效掉，就可以了。

`mysql> flush binary logs;
Query OK, 0 rows affected (0.00 sec)

mysql> show binary logs;
+-------------------+-----------+
| Log_name | File_size |
+-------------------+-----------+
| master-bin.000001 | 467 |
| master-bin.000002 | 168 |
| master-bin.000003 | 168 |
| master-bin.000004 | 168 |
| master-bin.000005 | 168 |
| master-bin.000006 | 120 |
+-------------------+-----------+
6 rows in set (0.00 sec)
`

那么下一个问题来了，为什么第一行会是个空行呢，因为之前主机磁盘被堆满，为了快速清出空间，运维同学把一些老的 binlog 清理掉了，同时 “贴心的” 的把 index 文件也同步手动编辑了，但是因为手残留下了一个空行。。。

## 问题总结

这个问题是比较简单的，遇到过一次的话，后面就不会被坑了。。。

从这个问题我们也可以看出，MySQL 在有些时候的逻辑处理非常粗糙简单，对于文件格式没有适当地检测机制，像这种诡异问题就被隐藏吞没掉。如果翻看 commit 历史的话，可以看到“空行就认为是文件结束”的逻辑，在2002年之后就一直是这样的了:-(

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)