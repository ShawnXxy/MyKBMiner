# MySQL · 捉虫动态 · OPTIMIZE 不存在的表

**Date:** 2014/11
**Source:** http://mysql.taobao.org/monthly/2014/11/01/
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

 ## MySQL · 捉虫动态 · OPTIMIZE 不存在的表 
 Author: 

 **bug 描述**

这是一个和 GTID 相关的Bug，也就是说5.6才会有，并且出现这个 bug 需要满足条件：

做修改性质的表管理操作，如 OPTIMIZE/ANALYZE/REPAIR 可以，CHECK 就不可以
操作对应的表不存在
gtid_next 被设置为一个固定的值，并且 binlog 开启
在同时满足这3种条件下，会发现记录binlog时，对应的 Gtid_log_event 中的UUID会记为 00000000-0000-0000-0000-000000000000，并且这个对应的 gtid 不会记入 Executed_Gtid_Set。

**bug影响**

从 bug 描述可以看出，这个 bug 的表现特征就是 gtid_event 记错了，因此单实例的话基本不受影响的，因为主备复制时才会用到 gtid，所以主备场景会受到这个bug的影响。下面我们看下主备场景下这个bug是如何影响的：

M<->S : M 和 S 互为主备，都是5.6，以 gtid 协议进行复制，M是主库。

假设我们在主库上执行了 OPTIMIZE TABLE non_exist_table，这时候 gtid_next = ‘AUTOMATIC’，不是一个固定值，所以主库的 gtid 记录还是正常的，假设这时生成的 gtid_log_event 为 f3c1dd3e-395d-11e4-be45-4cb16c8f4abc:5，binlog 传到备库后，SQL 线程在 apply 的时候，会先将 f3c1dd3e-395d-11e4-be45-4cb16c8f4abc:5 设置为 gtid_next，然后同样做 OPTIMIZE TABLE non_exist_table，这个时候就触发了bug，备库的 gtid_log_event 记为00000000-0000-0000-0000-000000000000:5，并且不记入 Executed_Gtid_Set。主库继续接收用户的更新，同时会将备库的 binlog 拉过去应用，当做到 00000000-0000-0000-0000-000000000000:5 时，发现这个不在 Executed_Gtid_Set 中，就会执行，同样触发 bug， gtid_log_event 记为00000000-0000-0000-0000-000000000000:5，并且同样不记入 Executed_Gtid_Set。如此这样循环往复，会发现 OPTIMIZE TABLE non_exist_table 对应的binlog 在主备之前循环，充斥在 binlog 和 relay log 中。

**bug 分析**

之所以出现这个bug，是因为表管理操作的特殊性，OPTIMIZE/ANALYZE/REPAIR/CHECK TABLE 这些都统一调用 mysql_admin_table 函数进行管理操作，mysql_admin_table 执行失败的时候，执行线程并不报错，而是在 mysql_admin_table 函数结束前，清空线程中的error，将错误信息封装在结果集(result set)中发送给客户端，所以 OPTIMIZE/ANALYZE/REPAIR 虽然执行失败了，但仍然会记 binlog 。 按照这个逻辑来看，出错了仍然记binlog也是没问题，只要记对就行了，但是这里有一个问题，就是 mysql_admin_table 会调用 open_and_lock_tables，因为表不存在，所以 open_and_lock_tables 打开表的时候就出错，然后调用 trans_rollback_stmt ，之后会调到 gtid_rollback，最终调到 thd->variables.gtid_next.set_undefined()。

`void set_undefined()
{
if (type == GTID_GROUP)
type= UNDEFINED_GROUP;
}
`
可以看到，如果是 type == GTID_GROUP，就将 type 设置为 UNDEFINED_GROUP。那么什么情况下gtid_next 的 type 会是 GTID_GROUP，答案是为一个固定值的时候，即类似这种 f3c1dd3e-395d-11e4-be45-4cb16c8f4abc:5。

而在 Gtid_log_event::Gtid_log_event 有这段逻辑，

`if (spec.type == GTID_GROUP)
{
global_sid_lock->rdlock();
sid= global_sid_map->sidno_to_sid(spec.gtid.sidno);
global_sid_lock->unlock();
}
else
sid.clear();
`
我们会发现，这个时候sid会被清掉，clear 操作就是置全0，所以最终写入 binlog 的就是全0。

细心的同学会发现，当 gtid_next = automatic 的时候，也是会被 clear 的（automatic 对应的 group 是 AUTOMATIC_GROUP），其实如果 gtid_next = automatic 的话，只有在 binlog commit 的时候才调用 gtid_before_write_cache 生成 gtid，所以前面的 gtid_rollback 是不会影响 automatic 的。

关于不记 Executed_Gtid_Set 的问题，gtid_rollback 的时候，一方面通过 thd->variables.gtid_next.set_undefined() 把 gtid_next 的type设成UNDEFINED_GROUP，另一方面用 thd->clear_owned_gtids()，把 thd->owned_gtid 的 sidno 设为0，导致最终不会添加到 Executed_Gtid_Set 中。

**bug修复**

官方已经修复了这个bug，具体可以参见这2个 revno

revno: 5749
revno: 5751
主要是第一个，第二个是post-fix。修复方法是在 THD 中加一个标志 skip_gtid_rollback，在进入 mysql_admin_table 时先根据上下文设置thd->skip_gtid_rollback ，在退出mysql_admin_table 前重置标志，gtid_rollback 在执行clear前会判断下thd->skip_gtid_rollback。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)