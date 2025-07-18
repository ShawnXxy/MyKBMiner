# MySQL · 专家投稿 · InnoDB物理行中null值的存储的推断与验证

**Date:** 2016/08
**Source:** http://mysql.taobao.org/monthly/2016/08/07/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 08
 ](/monthly/2016/08)

 * 当期文章

 MySQL · 特性分析 ·MySQL 5.7新特性系列四
* PgSQL · PostgreSQL 逻辑流复制技术的秘密
* MySQL · 特性分析 · MyRocks简介
* GPDB · 特性分析· Greenplum 备份架构
* SQLServer · 最佳实践 · RDS for SQLServer 2012权限限制提升与改善
* TokuDB · 引擎特性 · REPLACE 语句优化
* MySQL · 专家投稿 · InnoDB物理行中null值的存储的推断与验证
* PgSQL · 实战经验 · 旋转门压缩算法在PostgreSQL中的实现
* MySQL · 源码分析 · Query Cache并发处理
* PgSQL · 源码分析· pg_dump分析

 ## MySQL · 专家投稿 · InnoDB物理行中null值的存储的推断与验证 
 Author: fiona514 

 ## 前言
想写这边文章，是因为之前想写一个解析innodb ibd文件的工具，在写这个工具的过程中，发现逻辑记录转物理记录的转换中，最难的有两部分，一是每行每字段null值占用的字节和存储，二是变长字段占用的字节和存储的格式。本文中重点针对第一种情况。
之前看有关介绍compact行记录格式:

 变长字段之后的第二个部分是NULL标志位，该位指示了该行数据中是否有NULL值，有则用1表示。该部分所占字节为1字节
 —–《InnoDB存储引擎》

之后便思考是否不管有多少个列都是NULL，该部分都只占1个字节呢？
便有了如下测试

## 本文约定
逻辑记录:record (元组)
物理记录:row(行)
只讨论compact行格式

## 所用工具
自己python写的工具innodb_extract

## 测试数据

### 表结构

`localhost.test>desc null_test;
+------------------+--------------+------+-----+---------+----------------+
| Field | Type | Null | Key | Default | Extra |
+------------------+--------------+------+-----+---------+----------------+
| id | bigint(20) | NO | PRI | NULL | auto_increment | 
| name | varchar(20) | YES | | NULL | | 
| legalname | varchar(25) | YES | | NULL | | 
| industry | varchar(10) | YES | | NULL | | 
| province | varchar(10) | YES | | NULL | | 
| city | varchar(15) | YES | | NULL | | 
| size | varchar(15) | YES | | NULL | | 
| admin_department | varchar(128) | YES | | NULL | | 
+------------------+--------------+------+-----+---------+----------------+
8 rows in set (0.00 sec)

`

### 表内数据

```
+----+------+-----------+----------+----------+------+------+------------------+
| id | name | legalname | industry | province | city | size | admin_department |
+----+------+-----------+----------+----------+------+------+------------------+
| 1 | NULL | NULL | NULL | NULL | NULL | NULL | NULL | 
| 2 | TOM | NULL | NULL | NULL | NULL | NULL | NULL | 
| 3 | ALEX | NULL | NULL | NULL | NULL | NULL | HR | 
+----+------+-----------+----------+----------+------+------+------------------+
3 rows in set (0.00 sec)

```

## 分析数据

通过工具看三行数据

`# python innodb_extract.py null_test.ibd
infimum
7f 000010001c 8000000000000001 0000f1e27b17 b5000001680084
1 
7e 0000180020 8000000000000002 0000f1e27b17 b5000001680094 544f4d

2 TOM 
3e 000020ffb6 8000000000000003 0000f1e27b17 b50000016800a4 414c4558 4852

3 ALEX HR 
`

**第一行:**
null标志位:0x7f (01111111)
说明:从右向左方向写，一共7个null值
record header:000010001c
Transaction Id:0000f1e27b17
Roll Pointer:b5000001680084
数据:

**第二行:**
null标志位:0x7e (01111110)
说明：除第二列，其余均是null值
record header:0000180020
Transaction Id:0000f1e27b17
Roll Pointer:b5000001680084
数据:
第二列:544f4d => TOM

**第三行:**
null标志位:0x3e (00111110)
说明:除了第2列和第8列，其余均是null值
record header:000020ffb6
Transaction Id:0000f1e27b17
Roll Pointer:b5000001680084
数据:
第二列:414c4558 => ALEX
第八列:4852 => HR

## 假设
继续上面，如果包含Null值的字段是8个，或者9个会是怎样？

## 深度剖析
代码片段，该函数将物理记录转化为逻辑记录，版本5.5.31,源文件rem0rec.c，

`rec_convert_dtuple_to_rec_comp(
/*===========================*/
 rec_t* rec, /*!< in: origin of record */
 const dict_index_t* index, /*!< in: record descriptor */
 const dfield_t* fields, /*!< in: array of data fields */
 ulint n_fields,/*!< in: number of data fields */
 ulint status, /*!< in: status bits of the record */
 ibool temp) /*!< in: whether to use the
 format for temporary files in
 index creation */
{
 const dfield_t* field;
 const dtype_t* type;
 byte* end;
 byte* nulls;
 byte* lens;
 ulint len;
 ulint i;
 ulint n_node_ptr_field;
 ulint fixed_len;
 ulint null_mask = 1;
 ut_ad(temp || dict_table_is_comp(index->table));
 ut_ad(n_fields > 0);

 if (temp) {
 ut_ad(status == REC_STATUS_ORDINARY);
 ut_ad(n_fields <= dict_index_get_n_fields(index));
 n_node_ptr_field = ULINT_UNDEFINED;
 nulls = rec - 1;
 if (dict_table_is_comp(index->table)) {
 /* No need to do adjust fixed_len=0. We only
 need to adjust it for ROW_FORMAT=REDUNDANT. */
 temp = FALSE;
 }
 } else {
 nulls = rec - (REC_N_NEW_EXTRA_BYTES + 1);

 switch (UNIV_EXPECT(status, REC_STATUS_ORDINARY)) {
 case REC_STATUS_ORDINARY:
 ut_ad(n_fields <= dict_index_get_n_fields(index));
 n_node_ptr_field = ULINT_UNDEFINED;
 break;
 case REC_STATUS_NODE_PTR:
 ut_ad(n_fields
 == dict_index_get_n_unique_in_tree(index) + 1);
 n_node_ptr_field = n_fields - 1;
 break;
 case REC_STATUS_INFIMUM:
 case REC_STATUS_SUPREMUM:
 ut_ad(n_fields == 1);
 n_node_ptr_field = ULINT_UNDEFINED;
 break;
 default:
 ut_error;
 return;
 }
 }

 end = rec;
 lens = nulls - UT_BITS_IN_BYTES(index->n_nullable);
 /* clear the SQL-null flags */
 memset(lens + 1, 0, nulls - lens);

`
结合COMPACT row格式来看:

`row记录格式如下:

|---------------------extra_size-----------------------------------------|---------fields_data------------|
|--columns_lens---|---null lens----|------fixed_extrasize(5)-------------|--col1---|---col2---|---col2----|
|end<--------begin|end<-------beign|-------------------------------------|orgin---------------------------|

`

* 先看nulls = rec - (REC_N_NEW_EXTRA_BYTES + 1)
rec为记录开始的offset，也就是,extrasize也就是固定长度的record header的长度。注意null标志位和变长字段长度列表是从右->左的方向写的(原因可参见下部分代码)。所以nulls指向的是`null lens`后一字节开始的位置。
* 再看lens = nulls - UT_BITS_IN_BYTES(index->n_nullable)
index->n_nullable指的是表结构中定义can be null的字段的个数，一个字段用一个bit来标记，UT_BITS_IN_BYTES将占用bit数转为占用的字节数。所以lens指向的是column_lens后面一个字节的位置，即跳过了Null标志的占用的空间，同样在写入值的时候也是从后面向前面写。
* memset(lens + 1, 0, nulls - lens) 将nulls空间清零。

之后就是遍历每一个字段，先对定义了can be null字段进行处理

`/* Store the data and the offsets */

 for (i = 0, field = fields; i < n_fields; i++, field++) {
 const dict_field_t* ifield;

 type = dfield_get_type(field);
 len = dfield_get_len(field);

 if (UNIV_UNLIKELY(i == n_node_ptr_field)) {
 ut_ad(dtype_get_prtype(type) & DATA_NOT_NULL);
 ut_ad(len == REC_NODE_PTR_SIZE);
 memcpy(end, dfield_get_data(field), len);
 end += REC_NODE_PTR_SIZE;
 break;
 }

 if (!(dtype_get_prtype(type) & DATA_NOT_NULL)) {
 /* nullable field */
 ut_ad(index->n_nullable > 0);

 if (UNIV_UNLIKELY(!(byte) null_mask)) {
 nulls--;
 null_mask = 1;
 }

`

因为方向是从右向左写，也就是从后往前写，如果该字段为null，则将null标志位设为1并向前移1位，如果满了8个，也就是有8个字段都为null则offset向左移1位，并将null_mask置为1

从这段代码看出之前的猜想，也就是并不是Null标志位只固定占用1个字节==，而是以8为单位，满8个null字段就多1个字节，不满8个也占用1个字节，高位用0补齐

` ut_ad(*nulls < null_mask);

 /* set the null flag if necessary */
 if (dfield_is_null(field)) {
 *nulls |= null_mask;
 null_mask <<= 1;
 continue;
 }

 null_mask <<= 1;
 }
 
`

这段代码是就是设置null字段与null标志位的映射关系，如果字段为null，则设置标志位为1。

## 栗子验证

翻过来再看之前的例子，我们逐步的添加字段并设置default null看下null标志位的变化

* step 1，添加两个并设置default null

`localhost.test>alter table null_test add column `kind` varchar(15) DEFAULT NULL after `size`;
Query OK, 3 rows affected (0.09 sec)
Records: 3 Duplicates: 0 Warnings: 0

localhost.test>alter table null_test add column licenseno varchar(15) DEFAULT NULL after `kind`;
Query OK, 3 rows affected (0.11 sec)
Records: 3 Duplicates: 0 Warnings: 0.11

`

那么理论来讲，第一行数据有9个null列了。满8个null列之后，继续向左写移，写1个bit之后开始占据两个字节。我们通过工具解析之后看下

`# python innodb_extract.py null_test.ibd
01ff 000010001d 8000000000000001 0000f1e27c81 980000028c0084
1 
01fe 0000180021 8000000000000002 0000f1e27c81 980000028c0094 544f4d
2 TOM 
00fe 000020ffb3 8000000000000003 0000f1e27c81 980000028c00a4 414c455848
3 ALEX HR 

`

第一行null标志位变为0x01ff,即`00000001 11111111`一共有9个null字段，满了8位之后，继续向前占1个字节从右往左继续写
同理，第二行0x01fe,即`00000001 11111110`
第三行0x00fe,`00000000 11111110`

再继续添加8个字段并设置default null

`localhost.test>desc null_test;
+------------------+--------------+------+-----+---------+----------------+
| Field | Type | Null | Key | Default | Extra |
+------------------+--------------+------+-----+---------+----------------+
| id | bigint(20) | NO | PRI | NULL | auto_increment | 
| name | varchar(20) | YES | | NULL | | 
| legalname | varchar(25) | YES | | NULL | | 
| industry | varchar(10) | YES | | NULL | | 
| province | varchar(10) | YES | | NULL | | 
| city | varchar(15) | YES | | NULL | | 
| size | varchar(15) | YES | | NULL | | 
| kind | varchar(15) | YES | | NULL | | 
| licenseno | varchar(15) | YES | | NULL | | 
| admin_department | varchar(128) | YES | | NULL | | 
| null_col1 | varchar(15) | YES | | NULL | | 
| null_col2 | varchar(15) | YES | | NULL | | 
| null_col3 | varchar(15) | YES | | NULL | | 
| null_col4 | varchar(15) | YES | | NULL | | 
| null_col5 | varchar(15) | YES | | NULL | | 
| null_col6 | varchar(15) | YES | | NULL | | 
| null_col7 | varchar(15) | YES | | NULL | | 
| null_col8 | varchar(15) | YES | | NULL | | 
+------------------+--------------+------+-----+---------+----------------+
18 rows in set (0.00 sec)

`
最多Null字段的第一行目前有个17个null字段，对应17个Null bit

`root@hebe211 ibd]# python innodb_extract.py null_test.ibd

01ffff 000010001e 8000000000000001 0000f1e27cce c60000017600840301fffe0000
1 
01fffe 0000180022 8000000000000002 0000f1e27cce c6000001760094 544f4d
2 TOM 
01fefe 000020ffb0 8000000000000003 0000f1e27cce c60000017600a4 414c45 5848
3 ALEX HR 

`

第一行null标志位变为0x01ff,即`00000001 11111111 11111111` 一共有17个null字段，满了两个8位之后，继续向前占1个字节从右往左继续写
同理，第二行0x01fe,即`00000001 11111111 11111110`
第三行0x00fe,`00000001 11111110 11111110`

## 结论

允许null的字段需要额外的空间来保存字段Null到null标志位映射的对应关系，所以保存这个映射关系的null标志位长度并不是固定的。也就是null字段越多并不是越省空间。实际生产环境中应尽量减少can be null的字段。

 作者介绍
赵晨@微博研发中心, 微博：@fiona514

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)