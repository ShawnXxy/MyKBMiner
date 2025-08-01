# MySQL · 引擎特性 · Column Compression浅析

**Date:** 2016/11
**Source:** http://mysql.taobao.org/monthly/2016/11/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 11
 ](/monthly/2016/11)

 * 当期文章

 PgSQL · 特性分析 · 金融级同步多副本分级配置方法
* MySQL · myrocks · myrocks之事务处理
* MySQL · TokuDB · rbtree block allocator
* MySQL · 引擎特性 · Column Compression浅析
* MySQL · 引擎介绍 · Sphinx源码剖析（一）
* PgSQL · 特性分析 · PostgreSQL 9.6 如何把你的机器掏空
* PgSQL · 特性分析 · PostgreSQL 9.6 让多核并行起来
* MSSQL · 最佳实战 · 巧用COLUMNS_UPDATED获取数据变更
* PgSQL · GIS应用 · 物流, 动态路径规划
* PgSQL · 特性分析· JIT 在数据仓库中的应用价值

 ## MySQL · 引擎特性 · Column Compression浅析 
 Author: 印风 

 ## 前言

当用户的数据量比较大时，通常需要对数据进行压缩，以减少磁盘占用。InnoDB目前有两种方式来实现这一目的。

第一种是传统的数据压缩，通过指定row_format及key_block_size，能够将用户表压缩到指定的page size并进行存储，默认使用zlib。这种压缩方式使用比较简单，但也是诟病较多的， 代码陈旧，相关代码基本上几个大版本都没发生过变化，一些优化点还是从facebook移植过来的（集中在5.6版本中, 不过现在fb已经放弃优化InnoDB压缩了，转而聚集在自家压缩更好的myrock上）。InnoDB压缩表的性能瓶颈明显，尤其是在压缩page到指定size失败时触发索引分裂。

第二种是MySQL5.7引入的所谓transparent compression，通过文件系统punch hole和sparse file特性来实现的。具体的就是在将数据页进行压缩后，将留白的地方进行打洞，从而实现数据压缩的目的。这个实现的好处就是代码逻辑简单，整个feature的实现基本上没加多少代码，无需指定key_block_size（但依然需要根据文件系统block size对齐)，并且也能更方便的支持多种压缩算法。但缺点也明显，例如可能会产生大量的文件碎片，底层的文件管理可能更复杂；也无法降低buffer pool的占用(传统的压缩方式可以只在buffer pool保存压缩页)

另外还有一种方式是通过MySQL函数compress/decompress，由应用端来决定存入的数据是否压缩，并控制解压操作。但这种方式不够灵活，需要应用来修改代码。

在AliSQL中我们提供了一种新的列压缩方式，用户在建表时可以将列属性column_format指定为compressed，那么服务器就会在存入/取出这个列的数据时，自动对其进行压缩和解压动作。这个方案不仅降低了磁盘数据大小，而且也能最大程度的保证性能，例如在查询不涉及到压缩列时无需执行解压动作。该特性尤其适用于诸如blob或者text这样的大列。

Percona Server也基于该补丁进行了功能扩展和优化。社区用户现在可以同时从AliSQL及Percona Server中获得该特性。

本文主要简单介绍下AliSQL如何实现的该特性，以及Percona的实现方案。

## AliSQL实现

使用该特性非常简单，可以在建表时指定列属性，或者在ALTER TABLE来修改列属性。

`mysql> CREATE TABLE t1 (a INT PRIMARY KEY, b blob);
Query OK, 0 rows affected (0.00 sec)

mysql> ALTER TABLE t1 MODIFY COLUMN b BLOB COLUMN_FORMAT COMPRESSED;
Query OK, 0 rows affected (0.01 sec)
Records: 0 Duplicates: 0 Warnings: 0

mysql> SHOW CREATE TABLE t1\G
*************************** 1. row ***************************
Table: t1
Create Table: CREATE TABLE `t1` (
 `a` int(11) NOT NULL,
 `b` blob /*!50616 COLUMN_FORMAT COMPRESSED */,
 PRIMARY KEY (`a`)
 ) ENGINE=InnoDB DEFAULT CHARSET=utf8
1 row in set (0.00 sec)
`

目前仅支持对blob/text/varchar/varbinary这几种类型进行压缩，如果在其他类型列上定义compressed属性，会抛出一个warning，并忽略列属性:

`mysql> CREATE TABLE t1 (a INT AUTO_INCREMENT PRIMARY KEY, b INT COLUMN_FORMAT COMPRESSED);
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> SHOW WARNINGS;
+---------+------+------------------------------------------------------------------------------------------+
| Level | Code | Message |
+---------+------+------------------------------------------------------------------------------------------+
| Warning | 3002 | Can not define column 'b' in compressed format, silently change column_format to default |
+---------+------+------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
`

也不支持在压缩列上创建二级索引，因为压缩后的数据可能已经不具备顺序性，在其上创建索引没有意义，一个错误码会被抛出:

`mysql> CREATE TABLE t1 (a INT AUTO_INCREMENT PRIMARY KEY, b BLOB COLUMN_FORMAT COMPRESSED, KEY (b(20)));
ERROR 3001 (HY000): Compressed BLOB/TEXT/VARCHAR/VARBINARY column 'b' used in key list is not allowed
`

由于大部分用户的引擎还是InnoDB，因此目前该特性仅支持InnoDB表（其实真实原因是笔者在写这个补丁时只对InnoDB比较了解…..），未来不排除这个特性实现到server层，这样就可以做到和引擎无关了。

代码的实现也比较简单，分为两部分

**压缩**

在InnoDB接受到行数据并进行任何处理之前，先将对应的列数据进行压缩.

入口函数：`row_compress_column`

压缩后的数据包含如下部分:

`1. 一个字节的header:
- COLUMN_COMPRESS_FLAG (1bit), 数据是否进行过压缩
- COLUMN_COMPRESS_DATA_LEN(2bits), 原始列的长度
- COLUMN_COMPRESS_ALG(3bits), 压缩算法，目前值为0，表示只支持zlib
- COLUMN_COMPRESS_WRAP(1bit), 标示zlib是否计算了adler32值
- 保留1个bit

2. 数据压缩前的长度，占用的字节数存储在COLUMN_COMPRESS_DATA_LEN

3. 压缩后的数据

`

如果发现压缩后的数据比原始数据还大，则放弃压缩，但会额外浪费1个字节来进行标识

我们提供了一些参数来对压缩进行控制，包括

1. innodb_rds_column_compression_level: zlib的压缩级别
2. innodb_rds_column_zip_mem_use_heap: 压缩过程中的内存分配/释放的回调函数，是使用InnoDB自带的还是系统自带的
3. innodb_rds_column_zip_threshold: 当数据长度超过这么大时，才去进行压缩; 这个参数需要根据数据特点来进行调整，否则如果对很小的字段进行压缩，没什么效果不说，反而还浪费cpu.
4. innodb_rds_column_zlib_strategy:使用的zlib压缩策略：
5. innodb_rds_column_zlib_wrap: 是否在压缩/解压时进行adler32校验

**解压**

在从InnoDB取到一条数据并返回到server层之前，对列进行解压

入口函数: `row_decompress_column`

解压也比较简单，首先根据Header中的信息判断是否进行了压缩；然后再读出原始数据的长度；找到压缩数据的起始位置并进行解压后，跟原始长度进行校验。

全局Status变量来监控压缩和解压的次数：

`mysql> show status like '%column%compress%';
+----------------------------+-------+
| Variable_name | Value |
+----------------------------+-------+
| Innodb_column_compressed | 0 |
| Innodb_column_decompressed | 0 |
+----------------------------+-------+
2 rows in set (0.00 sec)
`

完整的补丁见[commit](https://github.com/alibaba/AliSQL/commit/f9753b591202241cbd9d1a02c2d95e8ce6fdd1a1)

## Percona实现

Percona Server的列压缩实现来自Pinterest的贡献，Pinterest以AliSQL的列压缩补丁作为基础，做了进一步的改进。他们写了一篇[博客](https://engineering.pinterest.com/blog/evolving-mysql-compression-part-1)描述了整个开发过程，感兴趣的可以点开看看。

为了实现更好的压缩比，Percona 实现了一个称为 “predefined dictionary”, 实际上这是引用了新版本的zlib的一个特性。在压缩初始化后(deflateInit2)，可以去设置一个预定义的数据词典.

参阅函数`row_compress_column`

`err = deflateInit2(&c_stream, srv_compressed_columns_zip_level,
 Z_DEFLATED, window_bits, MAX_MEM_LEVEL,
 srv_compressed_columns_zlib_strategy);
ut_a(err == Z_OK);

if (dict_data != 0 && dict_data_len != 0) {
 err = deflateSetDictionary(&c_stream, dict_data,
 dict_data_len);
 ut_a(err == Z_OK);
}
`

Percona利用这个特性，并增加了一系列的接口来管理预定义词典。每个压缩列都可以通过显式的命名指向一个词典。

**系统表**

增加了一个新的系统表SYS_ZIP_DICT, 用于存储词典数据, 定义如下：

`CREATE TABLE SYS_ZIP_DICT(
 ID INT UNSIGNED NOT NULL,
 NAME CHAR(64) NOT NULL,
 DATA BLOB NOT NULL
);

CREATE UNIQUE CLUSTERED INDEX SYS_ZIP_DICT_ID
ON SYS_ZIP_DICT (ID);
CREATE UNIQUE INDEX SYS_ZIP_DICT_NAME
ON SYS_ZIP_DICT (NAME);

你可以从information_schema.xtradb_zip_dict获得字段信息

`

系统表SYS_ZIP_DICT_COLS，用于存储哪些使用预定义压缩词典的列信息，定义如下:

`CREATE TABLE SYS_ZIP_DICT_COLS(
 TABLE_ID INT UNSIGNED NOT NULL,
 COLUMN_POS INT UNSIGNED NOT NULL,
 DICT_ID INT UNSIGNED NOT NULL
);

CREATE UNIQUE CLUSTERED INDEX SYS_ZIP_DICT_COLS_COMPOSITE ON SYS_ZIP_DICT_COLS (TABLE_ID, COLUMN_POS);

-- 建立在该表之上的视图：information_schema.xtradb_zip_dict_cols

`

**创建词典**

`语法： CREATE COMPRESSION_DICTIONARY <dict>(...)

例如： 

mysql> CREATE COMPRESSION_DICTIONARY dt1('abcd');
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT * FROM INFORMATION_SCHEMA.XTRADB_ZIP_DICT;
+----+------+----------+
| id | name | zip_dict |
+----+------+----------+
| 1 | dt1 | abcd |
+----+------+----------+
1 row in set (0.00 sec)

入口函数: innobase_create_zip_dict

`

**使用词典**

`mysql> CREATE TABLE t1 (a INT PRIMARY KEY, b BLOB COLUMN_FORMAT COMPRESSED WITH COMPRESSION_DICTIONARY dt1);
Query OK, 0 rows affected (0.01 sec)

mysql> SHOW CREATE TABLE t1\G
*************************** 1. row ***************************
Table: t1
Create Table: CREATE TABLE `t1` (
 `a` int(11) NOT NULL,
 `b` blob /*!50633 COLUMN_FORMAT COMPRESSED WITH COMPRESSION_DICTIONARY `dt1` */,
 PRIMARY KEY (`a`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
1 row in set (0.00 sec)

mysql> SELECT * FROM INFORMATION_SCHEMA.XTRADB_ZIP_DICT_COLS;
+----------+------------+---------+
| table_id | column_pos | dict_id |
+----------+------------+---------+
| 22 | 1 | 1 |
+----------+------------+---------+
1 row in set (0.00 sec)

可以看到该列使用的预定义词典序号为1，对应dt1

当表上有引用的词典时，在打开表时就要从系统表中去进行关联(ha_innobase::update_field_defs_with_zip_dict_info)

`

**删除词典**

`语法： DROP COMPRESSION_DICTIONARY <dict>

# 很显然，当有列引用到这个词典时，是不可以删除的

mysql> DROP COMPRESSION_DICTIONARY dt1;
ERROR 1894 (HY000): Compression dictionary 'dt1' is in use
mysql> ALTER TABLE t1 MODIFY COLUMN b BLOB COLUMN_FORMAT COMPRESSED;
Query OK, 0 rows affected (0.01 sec)
Records: 0 Duplicates: 0 Warnings: 0

mysql> DROP COMPRESSION_DICTIONARY dt1;
Query OK, 0 rows affected (0.01 sec)

mysql> SELECT * FROM INFORMATION_SCHEMA.XTRADB_ZIP_DICT_COLS;
Empty set (0.00 sec)

mysql> SELECT * FROM INFORMATION_SCHEMA.XTRADB_ZIP_DICT;
Empty set (0.00 sec

入口函数：innobase_drop_zip_dict

`

参考文档:
[Percona Column compression 文档](https://www.percona.com/doc/percona-server/5.6/flexibility/compressed_columns.html#compressed-columns)

[How to find a good/optimal dictionary for zlib](http://stackoverflow.com/questions/2011653/how-to-find-a-good-optimal-dictionary-for-zlib-setdictionary-when-processing-a)

[代码实现](https://github.com/percona/percona-server/commit/35d5d3faf00db7e32f48dcb39f776e43b83f1cb2)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)