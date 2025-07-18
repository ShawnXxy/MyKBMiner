# MySQL · 功能介绍 · binlog拉取速度的控制

**Date:** 2015/07
**Source:** http://mysql.taobao.org/monthly/2015/07/09/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 07
 ](/monthly/2015/07)

 * 当期文章

 MySQL · 引擎特性 · Innodb change buffer介绍
* MySQL · TokuDB · TokuDB Checkpoint机制
* PgSQL · 特性分析 · 时间线解析
* PgSQL · 功能分析 · PostGIS 在 O2O应用中的优势
* MySQL · 引擎特性 · InnoDB index lock前世今生
* MySQL · 社区动态 · MySQL内存分配支持NUMA
* MySQL · 答疑解惑 · 外键删除bug分析
* MySQL · 引擎特性 · MySQL logical read-ahead
* MySQL · 功能介绍 · binlog拉取速度的控制
* MySQL · 答疑解惑 · 浮点型的显示问题

 ## MySQL · 功能介绍 · binlog拉取速度的控制 
 Author: 沽月 

 ## binlog拉取存在的问题

MySQL 主备之间数据同步是通过binlog进行的，当主库更新产生binlog时，备库需要同步主库的数据，通过binlog协议从主库拉取binlog进行数据同步，以达到主备数据一致性的目的。但当主库tps较高时会产生大量的binlog，以致备库拉取主库产生的binlog时占用较多的网络带宽，引起以下问题：

1. 在MySQL中，写入与读取binlog使用的是同一把锁(Lock_log)，频繁的读取binlog，会加剧Lock_log冲突，影响主库执行，进而造成TPS降低或抖动；
2. 当备库数量较多时，备库拉取binlog会占用过多的带宽，影响应用的响应时间。

为了解决上面提到的问题，需要对binlog的拉取速度进行限制。

## 问题存在的原因

备库或应用通过binlog协议向主库发送消息，告诉主库要拉取binlog，主库经过权限认证后，以binlog_event为单位读取在本地的binlog，然后将这些binlog_event发送给应用，其过程简单描述如下：

1. 从mysql-bin.index中找到用户消息中的指定文件，如果没有指定要拉取的binlog文件名称，则用第一个；
2. 上Lock_log锁，从1)或4) 中的binlog file中读取一个binlog_event，释放Lock_log锁，判断binlog_event的类型；
3. 如果是普通binlog_event，则将binlog_event发送到net 缓冲区；
4. 如果是Rotate_log_event，则取出要Rotate到的文件，执行2)；
5. 如果当前读的文件是最后一个文件且已经读到了文件的结尾，则会释放Lock_log锁，并等待新的Log_event信号。

从以上过程可以看出，binlog的发送速度和IO、网络有很大的关系，只要这三者不受限制，程序会就尽力发送binlog而没有限制。

## 解决问题的方法

由3、4可以看出，程序在读取和发送之间是没有其它工作的，如果IO很强，读取的速度很快，那么binlog的发送速度就会很快且不受限制，进而造成本文开始所描述的问题；针对binlog发送速度的问题，rds_mysql 通过设置binlog发送线程的发送频率、休眠时间来调整binlog的发送速度，因此 rds_mysql 引入了两个新的参数：

1. binlog_send_idle_period binlog发送线程的每次休眠时间，单位微秒，默认值100；
2. binlog_send_limit_users binlog发送线程的速度配置，默认值”“。

举例如下：
set global binlog_send_limit_users=”rep1:3,rep2:10” 的作用是设置rep1拉取binlog的上限速度是3M/s, rep2拉取binlog的上限速度是10M/s，其中rep2、rep2指的是应用连接的用户名，对于binlog的拉取速度控制主要分为两个方面：

## binlog 发送速度监控线程

速度监控线程随着mysqld的启动而启动，用于定时扫描限速列表，计算列表中的每一个binlog dump线程的binlog发送速度，并根据计算的速度调整binlog的发送频率，其工作过程描述如下：

1. 速度监控线程随着mysqld的启动而启动，并初始化限速列表；
2. 对限速列表进行依次扫描，如果取到的线程不为空，转2);
3. 计算当前线程的发送速度，与用户设定的速度进行比较，大于设定的发送速度，转3)，如果小于用户设定的发送速度，则转4)
4. 通过调整当前线程的net_thread_frequency 成员，降低发送频率；
5. 通过调整当前线程的net_thread_frequency 成员，增加发送频率；
6. 遍历完限速列表后让出CPU 1毫秒，转1)

由以上描述可以看出，监控线程每毫秒执行一次，根据发送的字节数来计算binlog发送线程的发送速度是否超过设定的速度，并通过调整发送频率来调整binlog的发送速度，监控线程的限速列表是这样构造的：

1. binlog dump 线程在拉取binlog前会先根据连接的用户名判断是否应该对该用户限速，如果需要限速，则需要将当前dump线程加入限速列表；
2. 当binlog dump结束或断开连接时，从限速列表移除；
3. 当设置参数binlog_send_limit_users时，会对当前所有线程进行遍历，将被限制的用户加入限速列表，对不受限制的用户移出限制列表，所有受影响的线程不需要重新连接，可以实时生效。

## binlog dump 线程

dump 线程用于发送binlog，在发送过程中会根据监控线程设置的发送频率来调整binlog发送的速度，可以分为以下几步：

1. binlog dump 线程在拉取binlog前会先根据连接的用户名判断是否将本用户的线程加入限速列表；
2. 读取binlog，并查看是否需要休眠，需要休眠转3)，否则转4)；
3. 休眠binlog_send_idle_period；
4. 发送读取到的binlog event，转2。

因此可以通过设置binlog的发送频率及休眠时间精确调整binlog的发送速度

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)