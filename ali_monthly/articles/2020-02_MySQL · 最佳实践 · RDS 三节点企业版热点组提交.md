# MySQL · 最佳实践 · RDS 三节点企业版热点组提交

**Date:** 2020/02
**Source:** http://mysql.taobao.org/monthly/2020/02/03/
**Images:** 2 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2020 / 02
 ](/monthly/2020/02)

 * 当期文章

 MySQL · 引擎特性 · 庖丁解InnoDB之REDO LOG
* MySQL · 引擎特性 · InnoDB Buffer Pool 浅析
* MySQL · 最佳实践 · RDS 三节点企业版热点组提交
* MySQL · 引擎特性 · 8.0 heap table 介绍
* MySQL · 存储引擎 · MySQL的字段数据存储格式
* MySQL · 引擎特性 · MYSQL Binlog Cache详解

 ## MySQL · 最佳实践 · RDS 三节点企业版热点组提交 
 Author: 甄平 

 RDS 5.7三节点企业版提供热点组提交功能，对“电商秒杀”等热点更新场景有大幅性能优化。

## 前提条件

当前仅RDS for MySQL 5.7三节点企业版1.5.0.4及以上版本支持该功能。

## 功能设计

“电商秒杀”等业务场景会对某一行数据短时间进行大量并发更新，这种热点操作对数据库的性能有很大的挑战。MySQL传统的更新模式“lock-update-unlock”，性能上基本无法满足实际的需求。业界针对这个问题有很多优化方案：基于缓存的方案不能保证数据一致性，有丢数据和超卖的风险；基于排队的方案仅仅缓解了大并发下的雪崩问题，依然受限于引擎层的lock/unlock性能损耗；预扣减、异步扣减等业务优化，增加了业务逻辑的复杂度，一定程度上也影响了客户体验。

热点组提交功能是RDS三节点企业版自研的特性。用户开启参数后，并为热点更新的SQL添加相关优化器hint，组提交模块会将同一数据行的热点请求自动合并成合适大小的Group，把多次逻辑更新映射成单次物理更新，最终下发到引擎层。该方法彻底的突破了InnoDB引擎层的性能上限，在单行更新的场景下测试，相较原生的MySQL，随着并发数上升，有十倍甚至上百倍的性能提升。

## 使用方式

热点功能涉及到三种新的优化器hint：

 commit_on_success
 更新成功自动提交
 必选

 rollback_on_fail
 更新失败自动回滚
 可选

 target_affect_row(1)
 显式指定该请求只会更新一行，若不符合，更新失败
 可选

同时需要打开hotspot相关参数：

`set global hotspot=ON;
set global hotspot_lock_type=ON;
`

需要注意的是，只有在打开参数配置的基础上，同时使用commit_on_success的hint，才能激活该功能。
样例SQL如下：

`mysql> create table test (id int primary key, data int);
Query OK, 0 rows affected (0.01 sec)

mysql> insert into test values (1, 1);
Query OK, 1 row affected (0.01 sec)

mysql> update /*+ commit_on_success */ test set data = data + 1 where id = 1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1 Changed: 1 Warnings: 0

mysql> select * from test;
+----+------+
| id | data |
+----+------+
| 1 | 2 |
+----+------+
1 row in set (0.00 sec)

mysql> update /*+ commit_on_success rollback_on_fail target_affect_row(1) */ test set data = data + 1 where id = 1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1 Changed: 1 Warnings: 0

mysql> select * from test;
+----+------+
| id | data |
+----+------+
| 1 | 3 |
+----+------+
1 row in set (0.00 sec)
`

此外也支持`select ... from update`的语法，可以直接返回更新后的数据。

`mysql> select * from test;
+----+------+
| id | data |
+----+------+
| 1 | 3 |
+----+------+
1 row in set (0.00 sec)

mysql> select id, data from update /*+ commit_on_success */ test set data = data + 1 where id = 1;
+----+------+
| id | data |
+----+------+
| 1 | 4 |
+----+------+
1 row in set (0.01 sec)
`

通过`show global status like "%Group_update%"`可以查询组提交状态。当`Group_update_leader_count`增加的时候，说明触发了热点组提交的优化逻辑。

`mysql> show global status like "%Group_update%";
+---------------------------------------+-------+
| Variable_name | Value |
+---------------------------------------+-------+
| Group_update_fail_count | 0 |
| Group_update_follower_count | 0 |
| Group_update_free_count | 1 |
| Group_update_gu_leak_count | 0 |
| Group_update_gu_lock_fail_count | 0 |
| Group_update_ignore_count | 0 |
| Group_update_insert_dup | 0 |
| Group_update_leader_count | 2 |
| Group_update_mgr_recycle_queue_length | 0 |
| Group_update_recycle_queue_length | 0 |
| Group_update_reuse_count | 1 |
| Group_update_total_count | 1 |
+---------------------------------------+-------+
12 rows in set (0.00 sec)
`

## 使用限制

* 只支持基于主键的单行更新。
* 无热点hint的SQL持有行锁的时间内，热点hint的SQL更新同一行会立刻冲突报错。因此不建议热点非热点混用。

## 相关参数

 参数
 说明

 hotspot
 ON,OFF 热点组提交功能开关。

 hotspot_lock_type
 ON,OFF 热点组提交锁优化开关。一般情况下，hotspot和hotspot_lock_type会同时开启。

 hotspot_update_max_wait_time
 热点组提交Group收集时间，一般保留默认参数即可。

 innodb_hotspot_kill_lock_holder
 ON,OFF 带有热点标记的SQL发现行锁被不带热点标记的事务持有后，主动kill持有锁的事务。

## 性能测试

测试不同并发数下，单行更新性能，统计tps和95%rt。

准备数据：

`root@test 03:34:13>create table t1(id int primary key auto_increment, data int);
Query OK, 0 rows affected (0.00 sec)

root@test 03:34:15>insert into t1(data) values (1);
Query OK, 1 row affected (0.00 sec)
`

压测SQL：

`UPDATE /*+ commit_on_success rollback_on_fail target_affect_row(1) */ t1 SET data=data+1 WHERE id=1;
`

64core 256G实例，测试结果：

 线程数
 hotspot=OFF
 hotspot=ON

 1
 6399.59 tps 0.17 ms
 3145.12 tps 0.33 ms

 4
 15473.29 tps 0.29 ms
 12009.01 tps 0.35 ms

 8
 14906.54 tps 0.58 ms
 22498.85 tps 0.38 ms

 16
 14930.81 tps 1.12 ms
 51153.38 tps 0.40 ms

 32
 14032.86 tps 2.38 ms
 77760.79 tps 0.46 ms

 64
 11334.73 tps 6.04 ms
 88099.79 tps 0.98 ms

 128
 5912.53 tps 22.15 ms
 90054.17 tps 1.75 ms

 256
 1869.35 tps 139.29 ms
 87724.28 tps 3.43 ms

 512
 379.01 tps 1495.24 ms
 89820.75 tps 6.57 ms

![perf](.img/e2ebef70a1e9_2020-02-05-zhenpin-perf.png)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)