# MySQL · 专家投稿 · MySQL5.7 的 JSON 实现

**Date:** 2016/01
**Source:** http://mysql.taobao.org/monthly/2016/01/03/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 01
 ](/monthly/2016/01)

 * 当期文章

 MySQL · 引擎特性 · InnoDB 事务锁系统简介
* GPDB   · 特性分析· GreenPlum Primary/Mirror 同步机制
* MySQL · 专家投稿 · MySQL5.7 的 JSON 实现
* MySQL · 特性分析 · 优化器 MRR & BKA
* MySQL · 答疑解惑 · 物理备份死锁分析
* MySQL · TokuDB · Cachetable 的工作线程和线程池
* MySQL · 特性分析 · drop table的优化
* MySQL · 答疑解惑 · GTID不一致分析
* PgSQL · 特性分析 · Plan Hint
* MariaDB · 社区动态 · MariaDB on Power8 (下)

 ## MySQL · 专家投稿 · MySQL5.7 的 JSON 实现 
 Author: laimingxing 

 ## 介绍

本文将介绍 MySQL 5.7 中如何实现非结构化（JSON）数据的存储，在介绍 MySQL 5.7 的非结构化数据存储之前，首先介绍在之前的 MySQL 的版本中，用户如何通过 BLOB 实现 JSON 对象的存储，以及这样处理的缺点是什么，这些缺点也就是 MySQL 5.7 支持 JSON 的理由；然后我们介绍了 MySQL 5.7 如何支持 JSON 格式，本文将重点关注MySQL 5.7 JSON 的存储格式。

## 5.7 之前 BLOB 方式实现 JSON 对象的存储

MySQL 是一个关系型数据库，在 MySQL 5.7 之前，没有提供对非结构化数据的支持，但是如果用户有这样的需求，也可以通过 MySQL 的 BLOB 来存储非结构化的数据。如下所示：

`mysql> create table t(json_data blob);
Query OK, 0 rows affected (0.13 sec)

mysql> insert into t values('{"key1":"data1", "key2":2, "key3":{"sub_key1":"sub_val1"}}');
Query OK, 1 row affected (0.01 sec)

mysql> select * from t;
+------------------------------------------------------------+
| json_data |
+------------------------------------------------------------+
| {"key1":"data1", "key2":2, "key3":{"sub_key1":"sub_val1"}} |
+------------------------------------------------------------+
1 row in set (0.00 sec)
`

在本例中，我们使用 BLOB 来存储 JSON 数据，使用这种方法，需要用户保证插入的数据是一个能够转换成 JSON 格式的字符串，MySQL 并不保证任何正确性。在MySQL看来，这就是一个普通的字符串，并不会进行任何有效性检查，此外提取 JSON 中的字段，也需要用户的代码中完成，如下所示：

`#!/usr/bin/python

import pymysql
import json

try:
 conn = pymysql.connect(host="127.0.0.1", db="test", user="root", passwd="root", port=7799)
 sql = "select * from t"
 cur = conn.cursor()
 cur.execute(sql)
 rows = cur.fetchall()
 print json.dumps(json.loads(rows[0][0]), indent=4)
except:
 conn.close()
`

执行python脚本的结果如下所示：

`root@dev1:~# python test.py
{
 "key3": {
 "sub_key1": "sub_val1"
 },
 "key2": 2,
 "key1": "data1"
}
`

这种方式虽然也能够实现 JSON 的存储，但是有诸多缺点，最为显著的缺点有：

1. 需要用户保证 JSON 的正确性，如果用户插入的数据并不是一个有效的 JSON 字符串，MySQL 并不会报错；
2. 所有对 JSON 的操作，都需要在用户的代码里进行处理，不够友好；
3. 即使只是提取 JSON 中某一个字段，也需要读出整个 BLOB，效率不高；
4. 无法在 JSON 字段上建索引。

## 5.7中的JSON实现

MySQL本身已经是一个比较完备的数据库系统，对于底层存储并不适合有太大的改动，那么 MySQL 是如何支持 JSON 格式的呢？说来也巧，和我们前面的做法几乎一样——通过 BLOB 来存储。也就是说，MySQL 5.7支持 JSON 的做法是，在server层提供了一堆便于操作 JSON 的函数，至于存储，就是简单地将 JSON 编码成 BLOB，然后交由存储引擎层进行处理，也就是说，MySQL 5.7的JSON 支持与存储引擎没有关系，MyISAM 存储引擎也支持 JSON 格式，如下所示：

`mysql> create table t_innodb(data json)engine=innodb;
Query OK, 0 rows affected (0.18 sec)

mysql> insert into t_innodb values('{"key":"val"}');
Query OK, 1 row affected (0.03 sec)

mysql> create table t_myisam(data json)engine=myisam;
Query OK, 0 rows affected (0.02 sec)

mysql> insert into t_myisam values('{"key":"val"}');
Query OK, 1 row affected (0.00 sec)
`

MySQL 5.7 提供了很多操作 JSON 的函数，都是为了提高易用性，可以参考[官方文档](https://dev.mysql.com/doc/refman/5.7/en/json-function-reference.html)。本文将主要关注实现。

关于MySQL 5.7的JSON存储，MySQL的源码里写得比较清楚，在sql/json_binary.h中有下面这段注释：

`If the value is a JSON object, its binary representation will have a
header that contains:

- the member count
- the size of the binary value in bytes
- a list of pointers to each key
- a list of pointers to each value

The actual keys and values will come after the header, in the same
order as in the header.

Similarly, if the value is a JSON array, the binary representation
will have a header with

- the element count
- the size of the binary value in bytes
- a list of pointers to each value
`

从注释里面我们可以知道，对于JSON数组和JSON对象，MySQL如何编码成BLOB对象，数组比较简单，下面给出JSON对象的示意图（见json_binary.cc中的`serialize_json_object`函数），如下所示：

说明如下：首先存放的是 JSON 的元素个数，然后存放的是转换成 BLOB 以后的字节数，接下来存放的是key pointers和value pointers。为了加快查找速度，MySQL 内部会对key进行排序，以便对key进行二分查找，以提高处理速度。

此外，对于key pointers，有如下注释：

`/*
 The size of key entries for objects when using the small storage
 format or the large storage format. In the small format it is 4
 bytes (2 bytes for key length and 2 bytes for key offset). In the
 large format it is 6 (2 bytes for length, 4 bytes for offset).
*/
#define KEY_ENTRY_SIZE_SMALL (2 + SMALL_OFFSET_SIZE)
#define KEY_ENTRY_SIZE_LARGE (2 + LARGE_OFFSET_SIZE)
`

也就是说，在MySQL 5.7中，key的长度只用2个字节保存（65535），如果超过这个长度，MySQL将报错，如下所示：

`mysql> insert into t1 values(JSON_OBJECT(repeat('a', 65535), 'val'));
Query OK, 1 row affected (0.37 sec)

mysql> insert into t1 values(JSON_OBJECT(repeat('a', 65536), 'val'));
ERROR 3151 (22032): The JSON object contains a key name that is too long.
`

如果查看MySQL的源码，可以看到，与JSON相关的文件有：

`json_binary.cc
json_binary.h
json_dom.cc
json_dom.h
json_path.cc
json_path.h
`

其中，json_binary 处理JSON 的编码、解码，json_dom 是 JSON 的内存表示，json_path 用以将字符串解析成 JSON，具体说明见[WL#7909](http://dev.mysql.com/worklog/task/?id=7909)。

对于 JSON 的编码，入口是json_binary.cc 文件中的`serialize`函数，对于 JSON 的解码，即将 BLOB 解析成 JSON 对象，入口是json_binary.cc文件中的`parse_binary`函数，只要搞清楚了 JSON 的存储格式，这两个函数是很好理解的。

 作者介绍
赖明星 厦门大学硕士毕业，网易杭研服务器端开发工程师，MySQL 爱好者，网名“不知一不知二”。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)