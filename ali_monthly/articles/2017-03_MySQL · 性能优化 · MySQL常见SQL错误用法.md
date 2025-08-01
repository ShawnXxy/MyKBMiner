# MySQL · 性能优化 · MySQL常见SQL错误用法

**Date:** 2017/03
**Source:** http://mysql.taobao.org/monthly/2017/03/03/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 03
 ](/monthly/2017/03)

 * 当期文章

 MySQL · 引擎特性 · InnoDB IO子系统
* PgSQL · 特性分析 · Write-Ahead Logging机制浅析
* MySQL · 性能优化 · MySQL常见SQL错误用法
* MSSQL · 特性分析 · 列存储技术做实时分析
* MySQL · 新特性分析 · 5.7中Derived table变形记
* MySQL · 实现分析 · 对字符集和字符序支持的实现
* MySQL · 源码分析 · MySQL BINLOG半同步复制数据安全性分析
* HybridDB · 性能优化 · Count Distinct的几种实现方式
* PgSQL · 应用案例 · PostgreSQL OLAP加速技术之向量计算
* MySQL · myrocks · myrocks监控信息

 ## MySQL · 性能优化 · MySQL常见SQL错误用法 
 Author: 西扬 

 ## 前言

MySQL在2016年仍然保持强劲的数据库流行度增长趋势。越来越多的客户将自己的应用建立在MySQL数据库之上，甚至是从Oracle迁移到MySQL上来。但也存在部分客户在使用MySQL数据库的过程中遇到一些比如响应时间慢，CPU打满等情况。阿里云RDS专家服务团队帮助云上客户解决过很多紧急问题。现将《ApsaraDB专家诊断报告》中出现的部分常见SQL问题总结如下，供大家参考。

## 常见SQL错误用法

### 1. LIMIT 语句

分页查询是最常用的场景之一，但也通常也是最容易出问题的地方。比如对于下面简单的语句，一般DBA想到的办法是在type, name, create_time字段上加组合索引。这样条件排序都能有效的利用到索引，性能迅速提升。

`SELECT * 
FROM operation 
WHERE type = 'SQLStats' 
 AND name = 'SlowLog' 
ORDER BY create_time 
LIMIT 1000, 10; 
`

好吧，可能90%以上的DBA解决该问题就到此为止。但当 LIMIT 子句变成 “LIMIT 1000000,10” 时，程序员仍然会抱怨：我只取10条记录为什么还是慢？

要知道数据库也并不知道第1000000条记录从什么地方开始，即使有索引也需要从头计算一次。出现这种性能问题，多数情形下是程序员偷懒了。在前端数据浏览翻页，或者大数据分批导出等场景下，是可以将上一页的最大值当成参数作为查询条件的。SQL重新设计如下：

`SELECT * 
FROM operation 
WHERE type = 'SQLStats' 
AND name = 'SlowLog' 
AND create_time > '2017-03-16 14:00:00' 
ORDER BY create_time limit 10;
`

在新设计下查询时间基本固定，不会随着数据量的增长而发生变化。

### 2. 隐式转换
SQL语句中查询变量和字段定义类型不匹配是另一个常见的错误。比如下面的语句：

`mysql> explain extended SELECT * 
 > FROM my_balance b 
 > WHERE b.bpn = 14000000123 
 > AND b.isverified IS NULL ;
mysql> show warnings;
| Warning | 1739 | Cannot use ref access on index 'bpn' due to type or collation conversion on field 'bpn'
`

其中字段bpn的定义为varchar(20)，MySQL的策略是将字符串转换为数字之后再比较。函数作用于表字段，索引失效。

上述情况可能是应用程序框架自动填入的参数，而不是程序员的原意。现在应用框架很多很繁杂，使用方便的同时也小心它可能给自己挖坑。

### 3. 关联更新、删除
虽然MySQL5.6引入了物化特性，但需要特别注意它目前仅仅针对查询语句的优化。对于更新或删除需要手工重写成JOIN。

比如下面UPDATE语句，MySQL实际执行的是循环/嵌套子查询（DEPENDENT SUBQUERY)，其执行时间可想而知。

`UPDATE operation o 
SET status = 'applying' 
WHERE o.id IN (SELECT id 
 FROM (SELECT o.id, 
 o.status 
 FROM operation o 
 WHERE o.group = 123 
 AND o.status NOT IN ( 'done' ) 
 ORDER BY o.parent, 
 o.id 
 LIMIT 1) t); 
`

执行计划：

`+----+--------------------+-------+-------+---------------+---------+---------+-------+------+-----------------------------------------------------+
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
+----+--------------------+-------+-------+---------------+---------+---------+-------+------+-----------------------------------------------------+
| 1 | PRIMARY | o | index | | PRIMARY | 8 | | 24 | Using where; Using temporary |
| 2 | DEPENDENT SUBQUERY | | | | | | | | Impossible WHERE noticed after reading const tables |
| 3 | DERIVED | o | ref | idx_2,idx_5 | idx_5 | 8 | const | 1 | Using where; Using filesort |
+----+--------------------+-------+-------+---------------+---------+---------+-------+------+-----------------------------------------------------+
`

重写为JOIN之后，子查询的选择模式从DEPENDENT SUBQUERY变成DERIVED,执行速度大大加快，从7秒降低到2毫秒。

`UPDATE operation o 
 JOIN (SELECT o.id, 
 o.status 
 FROM operation o 
 WHERE o.group = 123 
 AND o.status NOT IN ( 'done' ) 
 ORDER BY o.parent, 
 o.id 
 LIMIT 1) t
 ON o.id = t.id 
SET status = 'applying' 
`

执行计划简化为：

`+----+-------------+-------+------+---------------+-------+---------+-------+------+-----------------------------------------------------+
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
+----+-------------+-------+------+---------------+-------+---------+-------+------+-----------------------------------------------------+
| 1 | PRIMARY | | | | | | | | Impossible WHERE noticed after reading const tables |
| 2 | DERIVED | o | ref | idx_2,idx_5 | idx_5 | 8 | const | 1 | Using where; Using filesort |
+----+-------------+-------+------+---------------+-------+---------+-------+------+-----------------------------------------------------+
`

### 4. 混合排序
MySQL不能利用索引进行混合排序。但在某些场景，还是有机会使用特殊方法提升性能的。

`SELECT * 
FROM my_order o 
 INNER JOIN my_appraise a ON a.orderid = o.id 
ORDER BY a.is_reply ASC, 
 a.appraise_time DESC 
LIMIT 0, 20 
`

执行计划显示为全表扫描：

`+----+-------------+-------+--------+-------------+---------+---------+---------------+---------+-+
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra 
+----+-------------+-------+--------+-------------+---------+---------+---------------+---------+-+
| 1 | SIMPLE | a | ALL | idx_orderid | NULL | NULL | NULL | 1967647 | Using filesort |
| 1 | SIMPLE | o | eq_ref | PRIMARY | PRIMARY | 122 | a.orderid | 1 | NULL |
+----+-------------+-------+--------+---------+---------+---------+-----------------+---------+-+
`

由于is_reply只有0和1两种状态，我们按照下面的方法重写后，执行时间从1.58秒降低到2毫秒。

`SELECT * 
FROM ((SELECT *
 FROM my_order o 
 INNER JOIN my_appraise a 
 ON a.orderid = o.id 
 AND is_reply = 0 
 ORDER BY appraise_time DESC 
 LIMIT 0, 20) 
 UNION ALL 
 (SELECT *
 FROM my_order o 
 INNER JOIN my_appraise a 
 ON a.orderid = o.id 
 AND is_reply = 1 
 ORDER BY appraise_time DESC 
 LIMIT 0, 20)) t 
ORDER BY is_reply ASC, 
 appraisetime DESC 
LIMIT 20; 
`

### 5. EXISTS语句
MySQL对待EXISTS子句时，仍然采用嵌套子查询的执行方式。如下面的SQL语句：

`SELECT *
FROM my_neighbor n 
 LEFT JOIN my_neighbor_apply sra 
 ON n.id = sra.neighbor_id 
 AND sra.user_id = 'xxx' 
WHERE n.topic_status < 4 
 AND EXISTS(SELECT 1 
 FROM message_info m 
 WHERE n.id = m.neighbor_id 
 AND m.inuser = 'xxx') 
 AND n.topic_type <> 5 
`

执行计划为：

`+----+--------------------+-------+------+-----+------------------------------------------+---------+-------+---------+ -----+
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
+----+--------------------+-------+------+ -----+------------------------------------------+---------+-------+---------+ -----+
| 1 | PRIMARY | n | ALL | | NULL | NULL | NULL | 1086041 | Using where |
| 1 | PRIMARY | sra | ref | | idx_user_id | 123 | const | 1 | Using where |
| 2 | DEPENDENT SUBQUERY | m | ref | | idx_message_info | 122 | const | 1 | Using index condition; Using where |
+----+--------------------+-------+------+ -----+------------------------------------------+---------+-------+---------+ -----+
`

去掉exists更改为join，能够避免嵌套子查询，将执行时间从1.93秒降低为1毫秒。

`SELECT *
FROM my_neighbor n 
 INNER JOIN message_info m 
 ON n.id = m.neighbor_id 
 AND m.inuser = 'xxx' 
 LEFT JOIN my_neighbor_apply sra 
 ON n.id = sra.neighbor_id 
 AND sra.user_id = 'xxx' 
WHERE n.topic_status < 4 
 AND n.topic_type <> 5 
`

新的执行计划：

`+----+-------------+-------+--------+ -----+------------------------------------------+---------+ -----+------+ -----+
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
+----+-------------+-------+--------+ -----+------------------------------------------+---------+ -----+------+ -----+
| 1 | SIMPLE | m | ref | | idx_message_info | 122 | const | 1 | Using index condition |
| 1 | SIMPLE | n | eq_ref | | PRIMARY | 122 | ighbor_id | 1 | Using where |
| 1 | SIMPLE | sra | ref | | idx_user_id | 123 | const | 1 | Using where |
+----+-------------+-------+--------+ -----+------------------------------------------+---------+ -----+------+ -----+
`

### 6. 条件下推

外部查询条件不能够下推到复杂的视图或子查询的情况有：

1. 聚合子查询；
2. 含有LIMIT的子查询；
3. UNION 或UNION ALL子查询；
4. 输出字段中的子查询；

如下面的语句，从执行计划可以看出其条件作用于聚合子查询之后：

`SELECT * 
FROM (SELECT target, 
 Count(*) 
 FROM operation 
 GROUP BY target) t 
WHERE target = 'rm-xxxx' 
`

```
+----+-------------+------------+-------+---------------+-------------+---------+-------+------+-------------+
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
+----+-------------+------------+-------+---------------+-------------+---------+-------+------+-------------+
| 1 | PRIMARY | <derived2> | ref | <auto_key0> | <auto_key0> | 514 | const | 2 | Using where |
| 2 | DERIVED | operation | index | idx_4 | idx_4 | 519 | NULL | 20 | Using index |
+----+-------------+------------+-------+---------------+-------------+---------+-------+------+-------------+

```

确定从语义上查询条件可以直接下推后，重写如下：

`SELECT target, 
 Count(*) 
FROM operation 
WHERE target = 'rm-xxxx' 
GROUP BY target
`

执行计划变为：

`+----+-------------+-----------+------+---------------+-------+---------+-------+------+--------------------+
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
+----+-------------+-----------+------+---------------+-------+---------+-------+------+--------------------+
| 1 | SIMPLE | operation | ref | idx_4 | idx_4 | 514 | const | 1 | Using where; Using index |
+----+-------------+-----------+------+---------------+-------+---------+-------+------+--------------------+
`

关于MySQL外部条件不能下推的详细解释说明请参考以前文章：[MySQL · 性能优化 · 条件下推到物化表](http://mysql.taobao.org/monthly/2016/07/08/)

### 7. 提前缩小范围

先上初始SQL语句：

`SELECT * 
FROM my_order o 
 LEFT JOIN my_userinfo u 
 ON o.uid = u.uid
 LEFT JOIN my_productinfo p 
 ON o.pid = p.pid 
WHERE ( o.display = 0 ) 
 AND ( o.ostaus = 1 ) 
ORDER BY o.selltime DESC 
LIMIT 0, 15 
`

该SQL语句原意是：先做一系列的左连接，然后排序取前15条记录。从执行计划也可以看出，最后一步估算排序记录数为90万，时间消耗为12秒。

`+----+-------------+-------+--------+---------------+---------+---------+-----------------+--------+----------------------------------------------------+
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
+----+-------------+-------+--------+---------------+---------+---------+-----------------+--------+----------------------------------------------------+
| 1 | SIMPLE | o | ALL | NULL | NULL | NULL | NULL | 909119 | Using where; Using temporary; Using filesort |
| 1 | SIMPLE | u | eq_ref | PRIMARY | PRIMARY | 4 | o.uid | 1 | NULL |
| 1 | SIMPLE | p | ALL | PRIMARY | NULL | NULL | NULL | 6 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+--------+---------------+---------+---------+-----------------+--------+----------------------------------------------------+
`

由于最后WHERE条件以及排序均针对最左主表，因此可以先对my_order排序提前缩小数据量再做左连接。SQL重写后如下，执行时间缩小为1毫秒左右。

`SELECT * 
FROM (
SELECT * 
FROM my_order o 
WHERE ( o.display = 0 ) 
 AND ( o.ostaus = 1 ) 
ORDER BY o.selltime DESC 
LIMIT 0, 15
) o 
 LEFT JOIN my_userinfo u 
 ON o.uid = u.uid 
 LEFT JOIN my_productinfo p 
 ON o.pid = p.pid 
ORDER BY o.selltime DESC
limit 0, 15
`

再检查执行计划：子查询物化后（select_type=DERIVED)参与JOIN。虽然估算行扫描仍然为90万，但是利用了索引以及LIMIT 子句后，实际执行时间变得很小。

`
+----+-------------+------------+--------+---------------+---------+---------+-------+--------+----------------------------------------------------+
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
+----+-------------+------------+--------+---------------+---------+---------+-------+--------+----------------------------------------------------+
| 1 | PRIMARY | <derived2> | ALL | NULL | NULL | NULL | NULL | 15 | Using temporary; Using filesort |
| 1 | PRIMARY | u | eq_ref | PRIMARY | PRIMARY | 4 | o.uid | 1 | NULL |
| 1 | PRIMARY | p | ALL | PRIMARY | NULL | NULL | NULL | 6 | Using where; Using join buffer (Block Nested Loop) |
| 2 | DERIVED | o | index | NULL | idx_1 | 5 | NULL | 909112 | Using where |
+----+-------------+------------+--------+---------------+---------+---------+-------+--------+----------------------------------------------------+
`

### 8. 中间结果集下推

再来看下面这个已经初步优化过的例子(左连接中的主表优先作用查询条件)：

`SELECT a.*, 
 c.allocated 
FROM ( 
 SELECT resourceid 
 FROM my_distribute d 
 WHERE isdelete = 0 
 AND cusmanagercode = '1234567' 
 ORDER BY salecode limit 20) a 
LEFT JOIN 
 ( 
 SELECT resourcesid， sum(ifnull(allocation, 0) * 12345) allocated 
 FROM my_resources 
 GROUP BY resourcesid) c 
ON a.resourceid = c.resourcesid
`

那么该语句还存在其它问题吗？不难看出子查询 c 是全表聚合查询，在表数量特别大的情况下会导致整个语句的性能下降。

其实对于子查询 c，左连接最后结果集只关心能和主表resourceid能匹配的数据。因此我们可以重写语句如下，执行时间从原来的2秒下降到2毫秒。

`SELECT a.*, 
 c.allocated 
FROM ( 
 SELECT resourceid 
 FROM my_distribute d 
 WHERE isdelete = 0 
 AND cusmanagercode = '1234567' 
 ORDER BY salecode limit 20) a 
LEFT JOIN 
 ( 
 SELECT resourcesid， sum(ifnull(allocation, 0) * 12345) allocated 
 FROM my_resources r, 
 ( 
 SELECT resourceid 
 FROM my_distribute d 
 WHERE isdelete = 0 
 AND cusmanagercode = '1234567' 
 ORDER BY salecode limit 20) a 
 WHERE r.resourcesid = a.resourcesid 
 GROUP BY resourcesid) c 
ON a.resourceid = c.resourcesid
`

但是子查询 a 在我们的SQL语句中出现了多次。这种写法不仅存在额外的开销，还使得整个语句显的繁杂。使用WITH语句再次重写：

`WITH a AS 
( 
 SELECT resourceid 
 FROM my_distribute d 
 WHERE isdelete = 0 
 AND cusmanagercode = '1234567' 
 ORDER BY salecode limit 20)
SELECT a.*, 
 c.allocated 
FROM a 
LEFT JOIN 
 ( 
 SELECT resourcesid， sum(ifnull(allocation, 0) * 12345) allocated 
 FROM my_resources r, 
 a 
 WHERE r.resourcesid = a.resourcesid 
 GROUP BY resourcesid) c 
ON a.resourceid = c.resourcesid
`

AliSQL即将推出WITH语法，敬请期待。

## 总结

1. 数据库编译器产生执行计划，决定着SQL的实际执行方式。但是编译器只是尽力服务，所有数据库的编译器都不是尽善尽美的。上述提到的多数场景，在其它数据库中也存在性能问题。了解数据库编译器的特性，才能避规其短处，写出高性能的SQL语句。
2. 程序员在设计数据模型以及编写SQL语句时，要把算法的思想或意识带进来。
3. 编写复杂SQL语句要养成使用WITH语句的习惯。简洁且思路清晰的SQL语句也能减小数据库的负担 ^^。
4. 使用云上数据库遇到难点（不局限于SQL问题），随时寻求阿里云原厂专家服务的帮助。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)