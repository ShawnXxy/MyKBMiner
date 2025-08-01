# MySQL · 谈古论今· key分区算法演变分析

**Date:** 2015/01
**Source:** http://mysql.taobao.org/monthly/2015/01/06/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 01
 ](/monthly/2015/01)

 * 当期文章

 MySQL · 性能优化· Group Commit优化
* MySQL · 新增特性· DDL fast fail
* MySQL · 性能优化· 启用GTID场景的性能问题及优化
* MySQL · 捉虫动态· InnoDB自增列重复值问题
* MySQL · 优化改进· 复制性能改进过程
* MySQL · 谈古论今· key分区算法演变分析
* MySQL · 捉虫动态· mysql client crash一例
* MySQL · 捉虫动态· 设置 gtid_purged 破坏AUTO_POSITION复制协议
* MySQL · 捉虫动态· replicate filter 和 GTID 一起使用的问题
* TokuDB·特性分析· Optimize Table

 ## MySQL · 谈古论今· key分区算法演变分析 
 Author: 

 本文说明一个物理升级导致的 "数据丢失"。

**现象**

在mysql 5.1下新建key分表，可以正确查询数据。

`drop table t1;

create table t1 (c1 int , c2 int) 
PARTITION BY KEY (c2) partitions 5; 
insert into t1 values(1,1785089517),(2,null); 
mysql&gt; select * from t1 where c2=1785089517;
+------+------------+
| c1 | c2 |
+------+------------+
| 1 | 1785089517 |
+------+------------+
1 row in set (0.00 sec)
mysql&gt; select * from t1 where c2 is null;
+------+------+
| c1 | c2 |
+------+------+
| 2 | NULL |
+------+------+
1 row in set (0.00 sec)
`

而直接用mysql5.5或mysql5.6启动上面的5.1实例，发现(1,1785089517)这行数据不能正确查询出来。

`alter table t1 PARTITION BY KEY ALGORITHM = 1 (c2) partitions 5;
mysql&gt; select * from t1 where c2 is null;
+------+------+
| c1 | c2 |
+------+------+
| 2 | NULL |
+------+------+
1 row in set (0.00 sec)
mysql&gt; select * from t1 where c2=1785089517;
Empty set (0.00 sec)
`

**原因分析**

跟踪代码发现，5.1 与5.5,5.6 key hash算法是有区别的。

5.1 对于非空值的处理算法如下

`void my_hash_sort_bin(const CHARSET_INFO *cs __attribute__((unused)),
const uchar *key, size_t len,ulong *nr1, ulong *nr2)
{
const uchar *pos = key; 

key+= len;

for (; pos &lt; (uchar*) key ; pos++)
{
nr1[0]^=(ulong) ((((uint) nr1[0] &amp; 63)+nr2[0]) * 
((uint)*pos)) + (nr1[0] &lt;&lt; 8);
nr2[0]+=3;
}
}
`

通过此算法算出数据(1,1785089517)在第3个分区

5.5和5.6非空值的处理算法如下

`void my_hash_sort_simple(const CHARSET_INFO *cs,
const uchar *key, size_t len,
ulong *nr1, ulong *nr2)
{
register uchar *sort_order=cs-&gt;sort_order;
const uchar *end;

/* 
Remove end space. We have to do this to be able to compare
&#039;A &#039; and &#039;A&#039; as identical
*/ 
end= skip_trailing_space(key, len);

for (; key &lt; (uchar*) end ; key++)
{
nr1[0]^=(ulong) ((((uint) nr1[0] &amp; 63)+nr2[0]) * 
((uint) sort_order[(uint) *key])) + (nr1[0] &lt;&lt; 8);
nr2[0]+=3;
}
}
`

通过此算法算出数据(1,1785089517)在第5个分区，因此，5.5,5.6查询不能查询出此行数据。

5.1,5.5,5.6对于空值的算法还是一致的,如下

`if (field-&gt;is_null())
{
nr1^= (nr1 &lt;&lt; 1) | 1;
continue;
}
`

都能正确算出数据(2, null)在第3个分区。因此，空值可以正确查询出来。

那么是什么导致非空值的hash算法走了不同路径呢？在5.1下，计算字段key hash固定字符集就是my_charset_bin，对应的hash 函数就是前面的my_hash_sort_simple。而在5.5，5.6下，计算字段key hash的字符集是随字段变化的，字段c2类型为int对应my_charset_numeric，与之对应的hash函数为my_hash_sort_simple。具体可以参考函数Field::hash

那么问题又来了，5.5后为什么算法会变化呢？原因在于官方关于字符集策略的调整，详见[WL#2649](http://dev.mysql.com/worklog/) 。

**兼容处理**

前面讲到，由于hash 算法变化，用5.5，5.6启动5.1的实例，导致不能正确查询数据。那么5.1升级5.5,5.6就必须兼容这个问题.mysql 5.5.31以后，提供了专门的语法 ALTER TABLE … PARTITION BY ALGORITHM=1 [LINEAR] KEY … 用于兼容此问题。对于上面的例子，用5.5或5.6启动5.1的实例后执行

`mysql&gt; alter table t1 PARTITION BY KEY ALGORITHM = 1 (c2) partitions 5;
Query OK, 2 rows affected (0.02 sec)
Records: 2 Duplicates: 0 Warnings: 0
`

```
mysql&gt; select * from t1 where c2=1785089517;
+------+------------+
| c1 | c2 |
+------+------------+
| 1 | 1785089517 |
+------+------------+
1 row in set (0.00 sec)

```

数据可以正确查询出来了。

而实际上5.5,5.6的mysql_upgrade升级程序已经提供了兼容方法。mysql_upgrade 执行check table xxx for upgrade 会检查key分区表是否用了老的算法。如果使用了老的算法，会返回

`mysql&gt; CHECK TABLE t1 FOR UPGRADE\G
*************************** 1\. row ***************************
Table: test.t1
Op: check
Msg_type: error
Msg_text: KEY () partitioning changed, please run:
ALTER TABLE `test`.`t1` PARTITION BY KEY /*!50611 ALGORITHM = 1 */ (c2)
PARTITIONS 5
*************************** 2\. row ***************************
Table: test.t1
Op: check
Msg_type: status
Msg_text: Operation failed
2 rows in set (0.00 sec)
`

检查到错误信息后会自动执行以下语句进行兼容。

```
ALTER TABLE `test`.`t1` PARTITION BY KEY /*!50611 ALGORITHM = 1 */ (c2) PARTITIONS 5。

```

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)