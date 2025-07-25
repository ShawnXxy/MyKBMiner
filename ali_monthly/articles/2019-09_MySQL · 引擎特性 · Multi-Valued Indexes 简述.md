# MySQL · 引擎特性 · Multi-Valued Indexes 简述

**Date:** 2019/09
**Source:** http://mysql.taobao.org/monthly/2019/09/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2019 / 09
 ](/monthly/2019/09)

 * 当期文章

 MySQL · 引擎特性 · 临时表改进
* MySQL · 引擎特性 · 初探 Clone Plugin
* MySQL · 引擎特性 · 网络模块优化
* MySQL · 引擎特性 · Multi-Valued Indexes 简述
* AliSQL · 引擎特性 · Statement Queue
* Database · 理论基础 · Palm Tree
* AliSQL · 引擎特性 · Returning
* PgSQL · 最佳实践 · 回归测试探寻
* MongoDB · 最佳实践 · 哈希分片为什么分布不均匀
* PgSQL · 应用案例 · PG有standby的情况下为什么停库可能变慢？

 ## MySQL · 引擎特性 · Multi-Valued Indexes 简述 
 Author: weixiang 

 本文主要简单介绍下8.0.17新引入的功能multi-valued index, 顾名思义，索引上对于同一个Primary key, 可以建立多个二级索引项，实际上已经对array类型的基础功能做了支持 （感觉官方未来一定会推出类似pg的array 列类型）， 并基于array来构建二级索引，这意味着该二级索引的记录数可以是多于聚集索引记录数的，因而该索引不可以用于通常意义的查询，只能通过特定的接口函数来使用，下面的例子里会说明。

本文不对代码做深入了解，仅仅记录下相关的入口函数，便于以后工作遇到时能快速查阅。在最后附上了对应worklog的连接，感兴趣的朋友可以直接阅读worklog去了解他是如何实现的。

## 范例
摘录自官方文档

`root@test 04:08:50>show create table customers\G
*************************** 1. row ***************************
Table: customers
Create Table: CREATE TABLE `customers` (
 `id` bigint(20) NOT NULL AUTO_INCREMENT,
 `modified` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
 `custinfo` json DEFAULT NULL,
 PRIMARY KEY (`id`),
 KEY `zips` ((cast(json_extract(`custinfo`,_latin1'$.zip') as unsigned array)))
 ) ENGINE=InnoDB AUTO_INCREMENT=6 DEFAULT CHARSET=latin1
1 row in set (0.00 sec)

 root@test 04:08:53>select * from customers;
 +----+---------------------+-------------------------------------------------------------------+
 | id | modified | custinfo |
 +----+---------------------+-------------------------------------------------------------------+
 | 1 | 2019-08-14 16:08:50 | {"user": "Jack", "user_id": 37, "zipcode": [94582, 94536]} |
 | 2 | 2019-08-14 16:08:50 | {"user": "Jill", "user_id": 22, "zipcode": [94568, 94507, 94582]} |
 | 3 | 2019-08-14 16:08:50 | {"user": "Bob", "user_id": 31, "zipcode": [94477, 94536]} |
 | 4 | 2019-08-14 16:08:50 | {"user": "Mary", "user_id": 72, "zipcode": [94536]} |
 | 5 | 2019-08-14 16:08:50 | {"user": "Ted", "user_id": 56, "zipcode": [94507, 94582]} |
 +----+---------------------+-------------------------------------------------------------------+
 5 rows in set (0.00 sec)
`
通过如下三个函数member of, json_contains, json_overlaps可以使用到该索引

`root@test 04:09:00>SELECT * FROM customers WHERE 94507 MEMBER OF(custinfo->'$.zipcode');
+----+---------------------+-------------------------------------------------------------------+
| id | modified | custinfo |
+----+---------------------+-------------------------------------------------------------------+
| 2 | 2019-08-14 16:08:50 | {"user": "Jill", "user_id": 22, "zipcode": [94568, 94507, 94582]} |
| 5 | 2019-08-14 16:08:50 | {"user": "Ted", "user_id": 56, "zipcode": [94507, 94582]} |
+----+---------------------+-------------------------------------------------------------------+
2 rows in set (0.00 sec)

 root@test 04:09:41>SELECT * FROM customers WHERE JSON_CONTAINS(custinfo->'$.zipcode', CAST('[94507,94582]' AS JSON));
 +----+---------------------+-------------------------------------------------------------------+
 | id | modified | custinfo |
 +----+---------------------+-------------------------------------------------------------------+
 | 2 | 2019-08-14 16:08:50 | {"user": "Jill", "user_id": 22, "zipcode": [94568, 94507, 94582]} |
 | 5 | 2019-08-14 16:08:50 | {"user": "Ted", "user_id": 56, "zipcode": [94507, 94582]} |
 +----+---------------------+-------------------------------------------------------------------+
 2 rows in set (0.00 sec)

 root@test 04:09:54>SELECT * FROM customers WHERE JSON_OVERLAPS(custinfo->'$.zipcode', CAST('[94507,94582]' AS JSON));
 +----+---------------------+-------------------------------------------------------------------+
 | id | modified | custinfo |
 +----+---------------------+-------------------------------------------------------------------+
 | 1 | 2019-08-14 16:08:50 | {"user": "Jack", "user_id": 37, "zipcode": [94582, 94536]} |
 | 2 | 2019-08-14 16:08:50 | {"user": "Jill", "user_id": 22, "zipcode": [94568, 94507, 94582]} |
 | 5 | 2019-08-14 16:08:50 | {"user": "Ted", "user_id": 56, "zipcode": [94507, 94582]} |
 +----+---------------------+-------------------------------------------------------------------+
 3 rows in set (0.00 sec)
`

## 接口函数
multi-value index是functional index的一种实现，列的定义是一个虚拟列，值是从json column上取出来的数组
数组上存在相同值的话，会只存储一个到索引上。支持的类型：DECIMAL, INTEGER, DATETIME,VARCHAR/CHAR。另外index上只能有一个multi-value column。

下面简单介绍下相关的接口函数

数组最大容量：
入口函数： ha_innobase::mv_key_capacity

插入记录:
入口函数 row_ins_sec_index_multi_value_entry
通过类Multi_value_entry_builder_insert来构建tuple, 然后调用正常的接口函数row_ins_sec_index_entry插入到二级索引中.
已经解析好，排序并去重的数据存储在结构struct multi_value_data , 指针在dfield_t::data中. multi_value_data结构也是multi-value具体值的内存表现

删除记录：
入口函数: row_upd_del_multi_sec_index_entry
基于类Multi_value_entry_builder_normal构建tuple, 并依次从索引中删除

更新记录
入口函数：row_upd_multi_sec_index_entry
由于可能不是所有的二级索引记录都需要更新，需要计算出diff，找出要更新的记录calc_row_difference –> innobase_get_multi_value_and_diff, 设置一个需要更新的bitmap

事务回滚
相关函数：

`row_undo_ins_remove_multi_sec
row_undo_mod_upd_del_multi_sec
row_undo_mod_del_mark_multi_sec
`

回滚的时候通过trx_undo_rec_get_multi_value从undo log中获取multi-value column的值，通过接口Multi_value_logger::read来构建并存储到field data中

记录undo log
函数: trx_undo_store_multi_value

通过Multi_value_logger::log将multi-value的信息存储到Undo log中. ‘Multi_value_logger’是一个辅助类，用于记录multi-value column的值以及如何读出来

purge 二级索引记录
入口函数:

`row_purge_del_mark
row_purge_upd_exist_or_extern_func
 |--> row_purge_remove_multi_sec_if_poss
`

## 参考文档
[WL#10604: Create multi-value index](https://yq.aliyun.com/go/articleRenderRedirect?spm=a2c4e.11153940.0.0.51ea6296s7kpxq&url=https%3A%2F%2Fdev.mysql.com%2Fworklog%2Ftask%2F%3Fid%3D10604)

[WL#8763: support multi-value functional index for InnoDB](https://yq.aliyun.com/go/articleRenderRedirect?spm=a2c4e.11153940.0.0.51ea6296s7kpxq&url=https%3A%2F%2Fdev.mysql.com%2Fworklog%2Ftask%2F%3Fid%3D8763)

[WL#8955: Add support for multi-valued indexes](https://yq.aliyun.com/go/articleRenderRedirect?spm=a2c4e.11153940.0.0.51ea6296s7kpxq&url=https%3A%2F%2Fdev.mysql.com%2Fworklog%2Ftask%2F%3Fid%3D8955)

[官方文档](https://yq.aliyun.com/go/articleRenderRedirect?spm=a2c4e.11153940.0.0.51ea6296s7kpxq&url=https%3A%2F%2Fdev.mysql.com%2Fdoc%2Frefman%2F8.0%2Fen%2Fcreate-index.html%23create-index-multi-valued)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)