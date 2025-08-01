# MySQL · 答疑释惑 · UPDATE交换列单表和多表的区别

**Date:** 2015/04
**Source:** http://mysql.taobao.org/monthly/2015/04/08/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 04
 ](/monthly/2015/04)

 * 当期文章

 MySQL · 引擎特性 · InnoDB undo log 漫游
* TokuDB · 产品新闻 · RDS TokuDB小手册
* TokuDB · 特性分析 · 行锁(row-lock)与区间锁(range-lock)
* PgSQL · 社区动态 · 说一说PgSQL 9.4.1中的那些安全补丁
* MySQL · 捉虫动态 · 连接断开导致XA事务丢失
* MySQL · 捉虫动态 · GTID下slave_net_timeout值太小问题
* MySQL · 捉虫动态 · Relay log 中 GTID group 完整性检测
* MySQL · 答疑释惑 · UPDATE交换列单表和多表的区别
* MySQL · 捉虫动态 · 删被引用索引导致crash
* MySQL · 答疑释惑 · GTID下auto_position=0时数据不一致

 ## MySQL · 答疑释惑 · UPDATE交换列单表和多表的区别 
 Author: 彭立勋 

 ## 背景描述

之前我们遇到一个咨询，客户说：

1. 同一个表，col1=a，col2=b，做 update，set col1=col2，col2=col1，这时候两个都是b
2. 不同表，A表 col1=a，B表 col2=b，做 update，就能进行交换
为什么不同表就能交换呢？

## 问题实验

### 一张表的测试

`root@localhost : test 12:36:09> select * from upt;
+------+------+ 
| c1 | c2 | 
+------+------+ 
| a | b | 
+------+------+ 
1 row in set (0.03 sec) 
root@localhost : test 12:36:20> update upt set c1=c2,c2=c1;
Query OK, 1 row affected (2 hours 47 min 59.80 sec)
Rows matched: 1 Changed: 1 Warnings: 0
root@localhost : test 03:24:32> select * from upt;
+------+------+ 
| c1 | c2 | 
+------+------+ 
| b | b | 
+------+------+ 
1 row in set (0.00 sec) 
`

### 两张表的测试

```
root@localhost : test 02:45:13> select * from upt1;
+------+------+------+ 
| c1 | c2 | id | 
+------+------+------+ 
| a | b | 1 | 
| c | d | 2 | 
+------+------+------+ 
2 rows in set (0.00 sec) 
root@localhost : test 02:45:18> select * from upt2;
+------+------+------+ 
| c1 | c2 | id | 
+------+------+------+ 
| e | f | 1 | 
| g | h | 2 | 
+------+------+------+ 
2 rows in set (0.00 sec) 
root@localhost : test 02:47:50> update upt1, upt2 set upt1.c1=upt2.c1, upt2.c1=upt1.c1 where upt1.id=upt2.id;
Query OK, 4 rows affected (0.04 sec) 
Rows matched: 4 Changed: 4 Warnings: 0 
root@localhost : test 02:48:25> select * from upt1;
+------+------+------+ 
| c1 | c2 | id | 
+------+------+------+ 
| e | b | 1 | 
| g | d | 2 | 
+------+------+------+ 
2 rows in set (0.00 sec) 
root@localhost : test 02:48:35> select * from upt2;
+------+------+------+ 
| c1 | c2 | id | 
+------+------+------+ 
| a | f | 1 | 
| c | h | 2 | 
+------+------+------+ 
2 rows in set (0.01 sec) 

```

## 问题分析

### 一张表的情况

UPDATE并没有把c1和c2列的值做交换，而是用c2列的值覆盖了c1列的值。而如果c1和c2来自不同的表，则会交换值，原因何在呢？

单张表的UPDATE函数入口为 `mysql_uptate()`，函数有两个参数 `List<Item> &fields，List<Item> &values` 分别表示要修改的列，和它们的目标值。

在上面例子中SET子句等号的左边，依次出现的是c1和c2，所以在fields数组中，顺序是field(c1)->field(c2)，在SET子句等号的右边，依次出现的是c2和c1，所以在values数组中，顺序是value(c2)->value(c1)。

对于单表UPDATE，MySQL调用了read_record()来读取values，所以会得到 value(c2).str_value=’b’->value(c1).str_value=’a’。然后在fill_record()中，根据fields的顺序依次调用value->save_in_field()来把values填入fields。

因此value(c2)会被首先赋值给field(c1)，因此field(c1).str_value=’b’，然后value(c1).str_value此时已经成为了’b’，因此value(c1)复制给filed(c2)依然还是’b’。

我们用三个列来验证我们的分析

`root@localhost : test 03:54:55> select * from upt;
+------+------+------+ 
| c1 | c2 | c3 | 
+------+------+------+ 
| a | b | c | 
+------+------+------+ 
1 row in set (0.01 sec) 
root@localhost : test 03:55:05> update upt set c1=c2, c2=c3, c3=c1;
Query OK, 1 row affected (0.00 sec) 
Rows matched: 1 Changed: 1 Warnings: 0 
root@localhost : test 03:55:45> select * from upt;
+------+------+------+ 
| c1 | c2 | c3 | 
+------+------+------+ 
| b | c | b | 
+------+------+------+ 
1 row in set (0.00 sec) 
`

可见，c1被赋值为c2的时候，c2还是’b’，c2被赋值为c3的时候，c3还是’c’。但是当c3被赋值为c1的时候，c1之前已经被赋值为’b’，所以c3也就成了’b’。

## 两张表的分析

对于不同表的UPDATE，MySQL调用的是mysql_multi_update()，定义一个multi_update类来处理，最终在 `multi_update::do_updates()` 中进行修改。

这里有什么不同的呢？

通过调研 `multi_update::do_updates()` 函数发现，multi_update类中的copy_field数组暂存了要更新的列值

`for ( ; *field ; field++) 
{ 
Item_field *item= (Item_field* ) field_it++; 
(copy_field_ptr++)->set(item->field, *field, 0); 
} 
`

然后从原表中读取一行记录，并存到table->record[1]，

`tbl->file->ha_rnd_pos(tbl->record[0], (uchar *) tmp_table->field[field_num]->ptr))) 
... 
store_record(table,record[1]); 
`

接着再把暂存的列值拷贝回table->record[0]，

`for (copy_field_ptr=copy_field; 
 copy_field_ptr != copy_field_end; 
 copy_field_ptr++) 
 (*copy_field_ptr->do_copy)(copy_field_ptr); 
`

最后调用ha_update_row这个API更新这行数据,

`local_error= table->file->ha_update_row(table->record[1], table->record[0]);
`

这样就不会因为列值被修改，而导致后续利用列值更新其他列的时候值变化了，这就是UPDATE多表和单表逻辑中区别的关键。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)