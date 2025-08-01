# MySQL · 捉虫动态 · ORDER/GROUP BY 导致 mysqld crash

**Date:** 2015/11
**Source:** http://mysql.taobao.org/monthly/2015/11/08/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 11
 ](/monthly/2015/11)

 * 当期文章

 MySQL · 社区见闻 · OOW 2015 总结 MySQL 篇
* MySQL · 特性分析 · Statement Digest
* PgSQL · 答疑解惑 · PostgreSQL 用户组权限管理
* MySQL · 特性分析 · MDL 实现分析
* PgSQL · 特性分析 · full page write 机制
* MySQL · 捉虫动态 · MySQL 外键异常分析
* MySQL · 答疑解惑 · MySQL 优化器 range 的代价计算
* MySQL · 捉虫动态 · ORDER/GROUP BY 导致 mysqld crash
* MySQL · TokuDB · TokuDB 中的行锁
* MySQL · 捉虫动态 · order by limit 造成优化器选择索引错误

 ## MySQL · 捉虫动态 · ORDER/GROUP BY 导致 mysqld crash 
 Author: santo 

 ## 问题描述

表结构如下所示:

`show create table test\G
 Table: test
Create Table: CREATE TABLE `test` (
 `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
 `id2` varchar(50) DEFAULT NULL
 `id3` varchar(100) DEFAULT NULL
 `some_text` varchar(200) DEFAULT NULL
 `name` varchar(20) DEFAULT NULL
 `another_text` varchar(500) DEFAULT NULL
 `ctime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
 PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=1024 DEFAULT CHARSET=utf8
`

对 mysql 执行如下语句:

`select count(distinct(id2))
from santo_test
where id3 = 'hahaha'
group by substr(ctime, 0, 10)
`
会导致mysql crash(signal 11)。

崩溃堆栈如下:

`pthread_kill ()
handle_segfault (sig=11)
 <signal handler called>
ptr_compare ()
queue_insert ()
merge_buffers()
merge_many_buff()
filesort()
create_sort_index()
JOIN::exec()
mysql_select()
handle_select()
execute_sqlcom_select()
mysql_execute_command()
mysql_parse()
...
`

[官方bug传送](https://bugs.mysql.com/bug.php?id=19660891)。

**Bug复现小贴士**
一条select语句搞挂MySQL Server? 当然还是需要苛刻条件的：

* 需要保证 sort by/group by 的列本身是 CHAR(0) NOT NULL, 值也要多样化, 不然会直接在优化器被优化掉;
* 接着该列不能有索引, 确保逻辑走到filesort(在对索引列做GROUP BY/ORDER BY时直接走索引)；
* 之后要配备足够小的`sort_buffer_size`, 和足够量大的数据撑满 sort_buffer，如@@sort_buffer_size = 32768时，40行数据就可以触发；
* 然后默默的给 substr 函数投喂错误的参数。

**BOOM!**

搞完破坏, 我们来看问题怎么解。

## 成因解析

在看到触发 crash 语句的时候，一定有读者发现哪里不对了。这里使用的 substr(some_string, 0, some_length) 这样的写法，而官方文档中 substr 函数的 @param2 实际上是从1开始计算，当起始位置置为0的时候，这条语句返回值其实是空的。当然，最终导致压坏 mysql server 的一根稻草，正是这个长度为0的字符串。

现在我们沿着执行路线来探索 mysql 是如何一步步挂掉的，在 select 语句中使用 order by/group by 语句时，server 通常调用排序，主要通过索引或者 filesort 来实现排序，在 group by/order by 的列上不存在索引时，server 会选择使用 filesort，其主要逻辑见 filesort.cc:filesort()。这里还会涉及到一个变量，`sort_buffer_size`，当需要排序的数据量超过 `sort_buffer_size` 大小时，server 会将数据划分为 trunks，这时调用 `merge_many_buffers()`。随后一路调用到 mysys/ptr_cmp.c 文件中的比较函数，这里的比较函数是按字节进行的，每四个字节为一个比较单位，当传入的参数长度小于4时，会调用 `ptr_compare()`，而在上节的调用栈可以看到，最后 crash 就是在这个函数里。函数槽点如下:

`static int ptr_compare(size_t *compare_length, uchar **a, uchar **b)
{
 reg3 int length= *compare_length;
 reg1 uchar *first,*last;

 first= *a; last= *b;
 while ( --length)
 {
 if (*first++ != *last++)
 return (int) first[-1] - (int) last[-1];
 }
 return (int) first[0] - (int) last[0];
}
`

在 lengh == 0 时，while 里就会根本停不下来，直到被比较的两位指针不停自加到一个不能访问的内存区域，逼迫系统用 signal 11 杀死 mysql server。

## 解决方案

比较长度为0的字符串本身是个意外, 所以解决方案就是添加一个辅助函数 `ptr_compare_length_zero`，在 length 为0时直接返回0，在做排序函数分派时，将长度为0的比较指派到`ptr_compare_length_zero`。
因此，想搞挂MySQL Server，这条路已经被堵上了，还是多修bug少搞破坏比较好 :-)

1. 官方fix160c6920509516a1e05b855799479a59c27803191
2. 官方fix2 b62c5daa646434290c9b2d1c9b162487cb8edf04
3. MySQL · 社区动态 · MySQL5.6.26 ReleaseNote解读

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)