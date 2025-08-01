# MySQL · 源码分析 ·  Row log分析

**Date:** 2022/03
**Source:** http://mysql.taobao.org/monthly/2022/03/02/
**Images:** 4 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2022 / 03
 ](/monthly/2022/03)

 * 当期文章

 MySQL · 源码阅读 · Purge sys介绍
* MySQL · 源码分析 · Row log分析

 ## MySQL · 源码分析 · Row log分析 
 Author: 张迎港 

 ## 引言

早期的MySQL仅支持copy模式的DDL。在MySQL 5.5中，引入了inplace算法，可以将部分DDL操作交给引擎层进行处理，但是在进行DDL期间，依旧会阻塞DML操作。在5.6中，部分inplace DDL操作可以采用online算法。该算法允许用户在进行DDL操作过程中，并行的执行写入操作。有关上述三种DDL的差异及主要特点，可以参照文末的扩展阅读部分。

以add column操作为例，下图给出了online算法进行DDL的主要流程。

![img](.img/2255c70bf661_online_add_column.png)

可以看到，在online DDL过程中，使用row log 记录并行的写入请求，等待DDL操作完成后，再将该部分写入请求应用在最新的表中。通过这种方式，能够减少DDL操作对业务影响。本文主要聚焦于row log的格式、生成和应用的相关函数解析。以下分析均基于MySQL-8.0.18版本。

## 一. row log 格式

根据不同的DML操作，row log分为三种类型，分别是ROW_T_INSERT、ROW_T_UPDATE和ROW_T_DELETE，分别对应insert，update和delete操作。一条row log记录通常由如下四部分组成。

`Part 1: opt type
 | 1 byte: 记录row log的操作类型。也即ROW_T_INSERT、ROW_T_UPDATE、ROW_T_DELETE
Part 2: old pk (optional)
 | 1 byte: extra size，也即old pk对应的rec的extra部分长度。
 | extra size byte: 存储old pk对应rec的extra数据
 | 不定长: 存储old pk对应rec的 field 部分数据。
Part 3: record data (optional)
 | 1-2 bytes：extra size 记录当前rec对应的extra数据长度
 | extra size: 待写入rec的extra部分数据
 | 不定长：待写入rec的field部分数据。
part 4: virtual column data (optional)
 | 2 bytes: 虚拟列部分总的数据长度
 | 不定长：虚拟列对应的field数据。
`

其中，不同类型的DML操作，对应所记录的row log内容也有一定差别。具体表现为：

* insert 类型

insert 类型的row log日志，在插入新表时，借助于col map即可映射到新表，无需记录旧表的主键信息。因此该类型row log仅包含标识符、record data和虚拟列信息（如有）；

* update类型

与insert操作相比，update类型的row log日志中，可能会包含old pk信息。这是因为如果新旧表之间主键信息发生了变化，则需要借助旧表的old pk信息在新表上进行定位。其余格式与insert 保持一致。

* delete 类型

delete 类型的row log在应用时，仅需在新表中将对应rec进行打标。因此，该类型仅需要记录old pk数据即可。不存在rec data和virtual field部分。

## 二. row log生成

在MySQL内部，与row log生成相关的函数可以分为主键索引和二级索引两大类。

### 1. 表相关的DML操作。

对表数据的操作可以分为INSERT、UPDATE和DELETE三类，分别由 row_log_table_insert() row_log_table_update(),row_log_table_delete()记录。其中由于insert 与update对应的row log格式相似，因此在内部统一调用函数row_log_table_low()进行记录。

### 2. 二级索引相关DML操作。

在innodb引擎中，对二级索引的update操作是通过delete+insert 方式进行的。因此对于二级索引，row log只有insert和update两种类型。统一由函数row_log_online_op()进行记录。

## 三. row log 应用

row log的应用发生在online DDL的commit阶段。在该阶段，会对所有在DDL执行阶段记录的row log进行回放。回放函数为row_log_table_apply_op()。该函数根据row log的操作类型，对row log进行解析处理。row_log_table_apply_op()函数功能可以抽象概括如下：

`const mrec_t *row_log_table_apply_op(
{
 ...
 /* Load row log type. */
 switch (row_log_type) {
 default:
 ut_ad(0);
 *error = DB_CORRUPTION;
 return (NULL);
 case ROW_T_INSERT:
 row_log_table_apply_insert();
 break;
 case ROW_T_DELETE:
 *error = row_log_table_apply_delete();
 break;
 case ROW_T_UPDATE:
 *error = row_log_table_apply_update();
 break;
 }
 ....
 return ;
}
`

## 四. row log中的record data

### 1. temp格式

读者在阅读row log代码时，会注意到，在这一过程中会出现temp格式的record。这是官方为了节约row log大小而引入的格式。以compact格式为例，其格式如下所示：

![img](.img/6e63349ba5de_compact_record.png)

其中，上文提到的extra部分指代的是变长字段长度列表和NULL字段标志位，并不包含记录头信息。也即与完整的compact、redunment格式相比，temp格式胜过省略记录头信息，降低了row log的大小。

针对此类格式的record，官方也提供了相关的接口函数。其中rec_get_converted_size_temp()可以用于计算temp格式rec的大小。rec_convert_dtuple_to_temp可以将tuple转化为temp格式的rec。对应的，rec_init_offsets_temp()和rec_init_null_and_len_temp()则用于对temp格式的rec进行解析。

事实上，temp格式的record不仅用于row log中，还被用于sort buffer中, 减少sort buffer的大小。

### 2. instant add column对row log格式的修改

MySQL在8.0版本引入了instant add column功能，该功能可以通过仅修改元数据的方式，实现加列操作。该功能是通过对列格式进行修改而实现的。以compact格式为例，修改后的列格式为：

![img](.img/9edfda5e17c6_compact_instant_record.png)

可以看出，其通过在record的记录头中，添加了一个INSTANT_FLAG标志位，来表示该记录中是否存在字段数量字段。借助该信息，来正确的解析instant add column前后的record数据。

同样的，对于执行过instant add column操作的表，在row log中记录record数据时，也同样需要记录字段数量信息。为了解决这一问题，MySQL对row log进行了扩展。在compact格式的reord，在生成row log时，如果当前表进行过instant add column操作，则会额外记录record的info bits信息。对应的，在调用函数rec_init_null_and_len_temp()用于解析时，也使用index中的instant标志位，辅助判断当前的temp格式的rec中是否存在info bits 信息。若存在该信息，则使用该信息辅助用于record解析。

## 结语

本文对row log格式以及相关的生成和应用函数做了简单介绍。希望能够帮助读者更好的了解row log的作用原理。受限于时间和笔者能力水平，文章内容可能存在错误，请大家批评指正。

## 扩展阅读

1. 《MySQL · 源码阅读 · 白话Online DDL》
2. 《MySQL · 引擎特性 · 8.0 Instant Add Column功能解析》
3. 《MySQL · 源码阅读 · 创建二级索引》

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)