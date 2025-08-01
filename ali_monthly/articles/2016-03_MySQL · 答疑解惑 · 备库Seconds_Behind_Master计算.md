# MySQL · 答疑解惑 · 备库Seconds_Behind_Master计算

**Date:** 2016/03
**Source:** http://mysql.taobao.org/monthly/2016/03/09/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 03
 ](/monthly/2016/03)

 * 当期文章

 MySQL · TokuDB · 事务子系统和 MVCC 实现
* MongoDB · 特性分析 · MMAPv1 存储引擎原理
* PgSQL · 源码分析 · 优化器逻辑推理
* SQLServer · BUG分析 · Agent 链接泄露分析
* Redis · 特性分析 · AOF Rewrite 分析
* MySQL · BUG分析 · Rename table 死锁分析
* MySQL · 物理备份 · Percona XtraBackup 备份原理
* GPDB · 特性分析· GreenPlum FTS 机制
* MySQL · 答疑解惑 · 备库Seconds_Behind_Master计算
* MySQL · 答疑解惑 · MySQL 锁问题最佳实践

 ## MySQL · 答疑解惑 · 备库Seconds_Behind_Master计算 
 Author: 济天 

 ## 背景

在mysql主备环境下，主备同步过程如下，主库更新产生binlog, 备库io线程拉取主库binlog生成relay log。备库sql线程执行relay log从而保持和主库同步。

理论上主库有更新时，备库都存在延迟，且延迟时间为备库执行时间+网络传输时间即t4-t2。

那么mysql是怎么来计算备库延迟的？

先来看show slave status中的一些信息，io线程拉取主库binlog的位置：

`Master_Log_File: mysql-bin.000001
Read_Master_Log_Pos: 107
`

sql线程执行relay log的位置：

` Relay_Log_File: slave-relay.000003
 Relay_Log_Pos: 253
`

sql线程执行的relay log相对于主库binlog的位置：

`Relay_Master_Log_File: mysql-bin.000001
Exec_Master_Log_Pos: 107
`

## 源码实现

Seconds_Behind_Master计算的源码实现如下：

`if ((mi->get_master_log_pos() == mi->rli->get_group_master_log_pos()) &&
 (!strcmp(mi->get_master_log_name(), mi->rli->get_group_master_log_name())))
{
 if (mi->slave_running == MYSQL_SLAVE_RUN_CONNECT)
 protocol->store(0LL);
 else
 protocol->store_null();
}
else
{
 long time_diff= ((long)(time(0) - mi->rli->last_master_timestamp)
 - mi->clock_diff_with_master);

 protocol->store((longlong)(mi->rli->last_master_timestamp ? max(0L, time_diff) : 0));
}
`

大致可以看出是通过时间和位点来计算的，下面详细分析下。

if里面条件表示如果io线程拉取主库binlog的位置和sql线程执行的relay log相对于主库binlog的位置相等，那么认为延迟为0。一般情况下，io线程比sql线程快。但如果网络状况特别差，导致sql线程需等待io线程的情况，那么这两个位点可能相等，会导致误认为延迟为0。

再看else里：

* `clock_diff_with_master`
 io线程启动时会向主库发送sql语句“SELECT UNIX_TIMESTAMP()”，获取主库当前时间，然而用备库当前时间减去此时间或者主备时间差值即为`clock_diff_with_master`。这里如果有用户中途修改了主库系统时间或修改了timestamp变量，那么计算出备库延迟时间就是不准确的。
* `last_master_timestamp`
表示主库执行binlog事件的时间。此时间在并行复制和非并行复制时的计算方法是不同的

非并行复制：
备库sql线程读取了relay log中的event，event未执行之前就会更新`last_master_timestamp`，这里时间的更新是以event为单位。

`rli->last_master_timestamp= ev->when.tv_sec + (time_t) ev->exec_time;
`

ev->when.tv_sec表示事件的开始时间。exec_time指事件在主库的执行时间，只有`Query_log_event`和`Load_log_event`才会统计exec_time。
另外一种情况是sql线程在等待io线程获取binlog时，会将`last_master_timestamp`设为0，按上面的算法Seconds_Behind_Master为0，此时任务备库是没有延迟的。

并行复制：

 并行复制有一个分发队列gaq，sql线程将binlog事务读取到gaq，然后再分发给worker线程执行。并行复制时，binlog事件是并发穿插执行的，gaq中有一个checkpoint点称为lwm, lwm之前的binlog都已经执行，而lwm之后的binlog有些执行有些没有执行。
假设worker线程数为2，gap有1,2,3,4,5,6,7,8个事务。worker 1已执行的事务为1 4 6, woker 2执行的事务为2 3 ，那么lwm为4。

并行复制更新gap checkpiont时，会推进lwm点，同时更新`last_master_timestamp`为lwm所在事务结束的event的时间。因此，并行复制是在事务执行完成后才更新`last_master_timestamp`，更新是以事务为单位。同时更新gap checkpiont还受`slave_checkpoint_period`参数的影响。

这导致并行复制下和非并行复制统计延迟存在差距，差距可能为`slave_checkpoint_period` + 事务在备库执行的时间。这就是为什么在并行复制下有时候会有很小的延迟，而改为非并行复制时反而没有延迟的原因。

另外当sql线程等待io线程时且gaq队列为空时，会将`last_master_timestamp`设为0。同样此时认为没有延迟，计算得出`seconds_Behind_Master`为0。

## 位点信息维护

* io线程拉取binlog的位点

 `Master_Log_File 读取到主库ROTATE_EVENT时会更新(process_io_rotate)
Read_Master_Log_Pos:io线程每取到一个event都会从event中读取pos信息并更新
mi->set_master_log_pos(mi->get_master_log_pos() + inc_pos);
`
* sql线程执行relay log的位置

 `Relay_Log_File
 sql线程处理ROTATE_EVENT时更新(Rotate_log_event::do_update_pos)
Relay_Log_Pos:
 非并行复制时，每个语句执行完成更新(stmt_done)
并行复制时，事务完成时更新(Rotate_log_event::do_update_pos/ Xid_log_event::do_apply_event/stmt_done)
`
* sql线程执行的relay log相对于主库binlog的位置

 `Relay_Master_Log_File
 sql线程处理ROTATE_EVENT时更新(Rotate_log_event::do_update_pos)
Exec_Master_Log_Pos 和Relay_Log_Pos同时更新
 非并行复制时，每个语句执行完成更新(stmt_done)
 并行复制时，事务完成时更新(Rotate_log_event::do_update_pos/ Xid_log_event::do_apply_event/stmt_done)
`

谈到位点更新就有必要说到两个事件：HEARTBEAT_LOG_EVENT 和 ROTATE_EVENT。

* HEARTBEAT_LOG_EVENT
HEARTBEAT_LOG_EVENT我们的了解一般作用是，在主库没有更新的时候，每隔`master_heartbeat_period`时间都发送此事件保持主库与备库的连接。而HEARTBEAT_LOG_EVENT另一个作用是，在gtid模式下，主库有些gtid备库已经执行同时，这些事件虽然不需要再备库执行，但读取和应用binglog的位点还是要推进。因此，这里将这类event转化为HEARTBEAT_LOG_EVENT，由HEARTBEAT_LOG_EVENT帮助我们推进位点。
* ROTATE_EVENT

 主库binlog切换产生的ROTATE_EVENT，备库io线程收到时会也有切换relay log。此rotate也会记入relay log，sql线程执行ROTATE_EVENT只更新位点信息。备库io线程接受主库的HEARTBEAT_LOG_EVENT，一般不用户处理。前面提到，gtid模式下，当HEARTBEAT_LOG_EVENT的位点大于当前记录的位点时，会构建一个ROTATE_EVENT,从而让sql线程推进位点信息。

 `if (mi->is_auto_position() && mi->get_master_log_pos() < hb。log_pos
 && mi->get_master_log_name() != NULL)
{
 mi->set_master_log_pos(hb。log_pos);
 write_ignored_events_info_to_relay_log(mi->info_thd, mi); //构建ROTATE_EVENT
 ......
}
`

另外，在`replicate_same_server_id`为0时，备库接收到的binlog与主库severid相同时，备库会忽略此binlog，但位点仍然需要推进。为了效率，此binlog不需要记入relay log。而是替换为ROTATE_EVENT来推进位点。

## 延迟现象

初始主备是同步的，且没有任何更新。假设主备库执行某个DDL在都需要30s，执行某个大更新事务(例如insert..select * from )需要30s。

不考虑网络延迟。

* 非并行复制时

 执行DDL：t2时刻主库执行完，t2时刻备库执行show slave status，Seconds_Behind_Master值为0。同时t2至t3 Seconds_Behind_Master依次增大至30，然后跌0。

 执行大事务：t2时刻主库执行完，t2时刻备库执行show slave status，Seconds_Behind_Master值为30。同时t2至t3 Seconds_Behind_Master依次增大至60，然后跌0。

 以上区别的原因是exec_time只有`Query_log_event`和`Load_log_event`才会统计，普通更新没有统计导致。
* 并行复制时

 执行DDL：t2时刻主库执行完，t2至t3备库执行show slave status，Seconds_Behind_Master值一直为0

 执行大事务：t2时刻主库执行完，t2至t3备库执行show slave status，Seconds_Behind_Master值一直为0

 这是因为执行语句之前主备是完全同步的，gaq队列为空，会将`last_master_timestamp`设为0。而执行DDL过程中，gap checkpoint一直没有推进，`last_master_timestamp`一直未0，直到DDL或大事务完成。
所以t2至t3时刻Seconds_Behind_Master值一直为0。而t3时刻有一瞬间`last_master_timestamp`是会重置的，但又因`slave_checkpoint_period`会推进checkpoint,gaq队列变为空，会将`last_master_timestamp`重设为0。
因此t3时刻可能看到瞬间有延迟(对于DDL是延迟30s,对于大事务时延迟60s)。

 这似乎很不合理，gaq队列为空，会将`last_master_timestamp`设为0,这条规则实际可以去掉。

## 相关bug

[BUG#72376](http://bugs。mysql。com/bug。php?id=72376), PREVIOUS_GTIDS_LOG_EVENT 事件记录在每个binlog的开头，表示先前所有文件的gtid集合。relay-log本身event记录是主库的时间，但relay log开头的PREVIOUS_GTIDS_LOG_EVENT事件，是在slave端生成的，时间也是以slave为准的。因此不能用此时间计算`last_master_timestamp`。修复方法是在relay log写PREVIOUS_GTIDS_LOG_EVENT事件是标记是relay log产生的，在统计`last_master_timestamp`时，发现是relay产生的事件则忽略统计。

`if (is_relay_log)
 prev_gtids_ev。set_relay_log_event();
 ......
if (!(ev->is_artificial_event()||...))
 rli->last_master_timestamp= ev->when。tv_sec + (time_t) ev->exec_time;
`

## 总结

Seconds_Behind_Master的计算并不准确和可靠。并行复制下Seconds_Behind_Master值比非并行复制时偏大。因此当我们判断备库是否延迟时，根据Seconds_Behind_Master=0不一定可靠。但是，当我们进行主备切换时，在主库停写的情况下，我们可以根据位点来判断是否完全同步。

如果(Relay_Master_Log_File, Exec_Master_Log_Pos)和(Relay_Master_Log_File, Read_Master_Log_Pos)位置相等且Seconds_Behind_Master=0，那么我们可以认为主备是完成同步的，可以进行切换。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)