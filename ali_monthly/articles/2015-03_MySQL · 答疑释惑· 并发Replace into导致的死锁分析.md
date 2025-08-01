# MySQL · 答疑释惑· 并发Replace into导致的死锁分析

**Date:** 2015/03
**Source:** http://mysql.taobao.org/monthly/2015/03/01/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 03
 ](/monthly/2015/03)

 * 当期文章

 MySQL · 答疑释惑· 并发Replace into导致的死锁分析
* MySQL · 性能优化· 5.7.6 InnoDB page flush 优化
* MySQL · 捉虫动态· pid file丢失问题分析
* MySQL · 答疑释惑· using filesort VS using temporary
* MySQL · 优化限制· MySQL index_condition_pushdown
* MySQL · 捉虫动态·DROP DATABASE外键约束的GTID BUG
* MySQL · 答疑释惑· lower_case_table_names 使用问题
* PgSQL · 特性分析· Logical Decoding探索
* PgSQL · 特性分析· jsonb类型解析
* TokuDB ·引擎机制· TokuDB线程池

 ## MySQL · 答疑释惑· 并发Replace into导致的死锁分析 
 Author: 

 **测试版本：**MySQL5.6.23

**测试表：**

`create table t1 (a int auto_increment primary key, b int, c int, unique key (b));并发执行SQL：
replace into t1(b,c) values (2,3) //使用脚本，超过3个会话
`

**背景**

Replace into操作可以算是比较常用的操作类型之一，当我们不确定即将插入的记录是否存在唯一性冲突时，可以通过Replace into的方式让MySQL自动处理：当存在冲突时，会把旧记录替换成新的记录。

我们先来理一下一条简单的replace into操作（如上例所示）的主要流程包括哪些。

**Step 1. 正常的插入逻辑**

首先插入聚集索引记录，在上例中a列为自增列，由于未显式指定自增值，每次Insert前都会生成一个不冲突的新值。

随后插入二级索引b，由于其是唯一索引，在检查duplicate key时，为其加上类型为LOCK_X的记录锁。

Tips：对于普通的INSERT操作，当需要检查duplicate key时，加LOCK_S锁，而对于Replace into 或者 INSERT..ON DUPLICATE操作，则加LOCK_X记录锁。

当UK记录已经存在时，返回错误DB_DUPLICATE_KEY。

**Step 2. 处理错误**

由于检测到duplicate key，因此第一步插入的聚集索引记录需要被回滚掉（row_undo_ins）。

**Step 3. 转换操作**

从InnoDB层失败返回到Server层后，收到duplicate key错误，首先检索唯一键冲突的索引，并对冲突的索引记录（及聚集索引记录）加锁。

随后确认转换模式以解决冲突：

* 如果发生uk冲突的索引是最后一个唯一索引、没有外键引用、且不存在delete trigger时，使用UPDATE ROW的方式来解决冲突；
* 否则，使用DELETE ROW + INSERT ROW的方式解决冲突。

**Step 4. 更新记录**

对于聚集索引，由于PK列发生变化，采用delete + insert 聚集索引记录的方式更新。

对于二级uk索引，同样采用标记删除 + 插入的方式。

我们知道，在尝试插入一条记录时，如果插入位置的下一条记录上存在记录锁，那么在插入时，当前session需要对其加插入意向锁，具体类型为LOCK_X | LOCK_GAP | LOCK_INSERT_INTENTION。这也是导致死锁的关键点之一。

**是否能保证自增列的有序性?**

默认情况下，参数innodb_autoinc_lock_mode的值为1，因此只在分配自增列时互斥（如果我们将其设为0的话，就会产生AUTO_INC类型的表级锁）。当分配完自增列值后，我们并不知道并发的replace into的顺序。

**死锁分析**

回到死锁线程分析，从死锁日志我们大致可以推断出如下序列（本例中死锁的heap no为5）：

* Session 1 执行到Step4, 准备更新二级Uk索引，因此持有uk上heap no 为5的X 行锁和PK上的X行锁；
* Session 2 检查到uk冲突，需要加X行锁；
* Session 1 在标记删除记录后，尝试插入新的uk记录，发现预插入点的下一条记录(heap no =5) 上有锁请求，因此尝试加插入意向X锁，产生锁升级， 死锁路径：Session1 => Session 2 => Session1。

到这里其实问题已经很明显了，我们考虑如下场景：假设当前表内数据为：

` root@sb1 08:57:41&gt;select * from t1;
 +---------+------+------+
 | a | b | c |
 +---------+------+------+
 | 2100612 | 2 | 3 |
 +---------+------+------+
 1 row in set (0.00 sec)
`

由于不能保证自增列被更新的有序性，我们假定有三个并发的会话，并假定表上只有一条记录。

session 1获得自增列值为2100619， session 2 获得的自增列值为2100614， session 3获得的自增列值为2100616。

Session 1: replace into t1 values (2100619, 2, 3); // uk索引上记录(2, 2100612)被标记删除，同时插入新记录(2, 2100619)

* Purge线程启动，(2, 2100612)被物理删除，Page上只剩下唯一的物理记录(2, 2100619)。

Session 2: replace into t1 values (2100614, 2, 3);

这里我们使用gdb的non-stop模式，使其断在row_update_for_mysql函数(insert尝试失败后，会转换成update)，此时session2持有(2, 2100619) 的X锁。

` Tips：我们可以通过如下命令使用gdb的non-stop模式：
 1\. 以gdb启动mysqld
 2\. 设置： 
 set target-async 1 
 set pagination off 
 set non-stop on
 3\. 设置函数断点，然后run
`

Session 3: replace into t1 values (2100616, 2, 3); // 检测到uk有冲突键，需要获取记录(2, 2100619) 的X锁，等待session 2。

Session 2:

* a)标记删除记录(2, 2100619)，同时插入新记录(2, 2100614)；
* b) (2, 2100614) 比(2, 2100619) 要小，因此定位到该记录之前，也就是系统记录infimum；
* c)infimum记录的下一条记录(2, 2100619)上有锁等待，需要升级成插入意向X锁，导致死锁发生。

**如果Purge线程一直停止，会发生什么呢 ？**

我们随便建一个表，然后执行FLUSH TABLE tbname FOR EXPORT来让purge线程停止。

假设当前表上数据为：

` root@sb1 10:26:05&gt;select * from t1;
 +---------+------+------+
 | a | b | c |
 +---------+------+------+
 | 2100710 | 2 | 3 |
 +---------+------+------+
 1 row in set (0.00 sec)
`

Session 1：replace into t1 values (2100720, 2, 3);

此时Page上存在记录(infimum), (2, 2100710), (2, 2100720), (supremum)。

Session 2：replace into t1 values (2100715, 2, 3);

同上例，使用gdb断到函数row_update_for_mysql。由于没有启动purge线程，因此老的被标记删除的记录还存在于page内，在扫描二级索引重复键时，也会依次给这些老记录加锁，因此session 2会持有 (2, 2100710)和 (2, 2100720)的X锁。

Session 3：replace into t1 values (2100718, 2, 3); // 被session2阻塞，等待(2,2100710)的X锁

Session 2：在标记删除二级索引记录，并进行插入时，选择的插入位置为 (2, 2100710), (2,2100720)之间，插入点的下一条记录(2,2100720)上没有其他线程锁等待，当前session锁升级成功；

完成插入后，page上的记录分布为(infimum), (2, 2100710), (2, 2100715), (2, 2100720), (supremum)。

Session 3：完成插入，最终page内的记录为(infimum), (2, 2100710), (2, 2100715), (2, 2100718), (2, 2100720), (supremum)。其中只有用户记录(2, 2100718)未被标记删除。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)