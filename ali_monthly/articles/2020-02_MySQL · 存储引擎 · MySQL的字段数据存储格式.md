# MySQL · 存储引擎 · MySQL的字段数据存储格式

**Date:** 2020/02
**Source:** http://mysql.taobao.org/monthly/2020/02/05/
**Images:** 1 images downloaded

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

 ## MySQL · 存储引擎 · MySQL的字段数据存储格式 
 Author: Xin Jia 

 ## 概述

MySQL支持多种存储引擎，而InnoDB是MySQL事务型数据库的首选引擎，也是MySQL从5.6版本以来的默认存储引擎。

InnoDB的存储格式已经有太多介绍性的文章，讲述了Tablespaces, Segments, Exents, Pages, Records等概念。其中很少有人对行存Record的不同数据字段进行介绍。本文讨论分析一下常见的字段数据在MySQL和InnoDB种不同的存储格式，并给出方法，大家可以自行学习其他没有涉及的字段。这里讨论的MySQL和InnoDB都是MySQL 5.7或MySQL 8.0, 过早的版本不在本文讨论范围。

## 字段数据格式学习方法

因为MySQL可以对接不同独立的存储引擎，MySQL和其对应的存储引擎对数据的存储方式就可能不同。因此MySQL必然会在计算层和存储层有不同的存储格式，也会有对应的数据转化方法。

对于InnoDB而言，有两个方法对于数据格式的转化最为关键。

1. MySQL数据 -> InnoDB数据  (row0mysql.cc)
 `/** Stores a non-SQL-NULL field given in the MySQL format in the InnoDB format. */
row_mysql_store_col_in_innobase_format()
`
2. InnoDB数据 -> MySQL数据  (row0sel.cc)
 ```
/** Convert a field from Innobase format to MySQL format. */
row_sel_store_mysql_field

```

大家可以使用GDB设置断点在以上两个函数，就可以清楚的认识到不同字段数据的存储格式了。

## 常见字段数据格式

### Numeric Data Types
#### INTEGER, INT, SMALLINT, TINYINT, MEDIUMINT, BIGINT
**Table:  Required Storage and Range for Integer Types Supported by MySQL**

 Type
 Storage (Bytes)
 Minimum Value Signed
 Minimum Value Unsigned
 Maximum Value Signed
 Maximum Value Unsigned

 `TINYINT`
 1
 `-128`
 `0`
 `127`
 `255`

 `SMALLINT`
 2
 `-32768`
 `0`
 `32767`
 `65535`

 `MEDIUMINT`
 3
 `-8388608`
 `0`
 `8388607`
 `16777215`

 `INT`
 4
 `-2147483648`
 `0`
 `2147483647`
 `4294967295`

 `BIGINT`
 8
 `-2`
 `0`
 `2-1`
 `2-1`

MySQL使用little-endian格式存储integer数据，InnoDB使用big-endian格式，并且符号为是取反处理。InnoDB这样设计存储Integer的好处是在数据比较的时候，可以直接一个一个byte去比较 - memcmp。

举例：BIGINT value 1000
InnoDB format
1000 stored as bigint (8 bytes) in Hex as:
0x80 0x00 0x00 0x00 0x00 0x00 0x03 0xe8
-1000 stored as bigint (8 bytes) in Hex as:
0x7f 0xff 0xff 0xff 0xff 0xff 0xfc 0x18

MySQL format:
1000 stored in Hex as: 
0xe8    0x03    0x00    0x00    0x00    0x00    0x00    0x00

#### DECIMAL
这里MySQL和InnoDB存储格式一致，不需要做特别转换。Decimal需要声明precision和scale，例如decimal(30,15)。精度表示值存储的有效位数，小数位数表示小数点后可以存储的位数。

 举例：decimal(30,15) value
 1000.01 stored (14 bytes) in Hex as:
 0x80    0x00    0x00    0x00    0x00    0x03    0xe8    0x00
 0x98    0x96    0x80    0x00    0x00    0x00

#### FLOAT, DOUBLE
FLOAT/DOUBLE类型表示近似数字数据值。这里MySQL和InnoDB存储格式一致，不需要做特别转换。MySQL将四个字节用于单精度值，并将八个字节用于双精度值。单精度FLOAT列存储精度范围是从0到23, 双精度FLOAT列存储精度范围是从24到53。

举例：double 
1000.01 stored (8 bytes) in Hex as:
0xae    0x47    0xe1    0x7a    0x14    0x40    0x8f    0x40

### Date and Time Data Types
时间相关的存储格式：

 **Type**
 **Storage as of MySQL 5.6.4**

 YEAR
 1 byte, little endian

 DATE
 3 bytes, little endian

 TIME
 3 bytes + fractional-seconds storage, big endian

 TIMESTAMP
 4 bytes + fractional-seconds storage, big endian

 DATETIME
 5 bytes + fractional-seconds storage, big endian

* TIME encoding for non-fractional part:

` 1 bit sign (1= non-negative, 0= negative)
 1 bit unused (reserved for future extensions)
10 bits hour (0-838)
 6 bits minute (0-59) 
 6 bits second (0-59) 
---------------------
24 bits = 3 bytes
`

* DATETIME encoding for non-fractional part:
 ```
 1 bit sign (1= non-negative, 0= negative)
17 bits year*13+month (year 0-9999, month 0-12)
 5 bits day (0-31)
 5 bits hour (0-23)
 6 bits minute (0-59)
 6 bits second (0-59)
---------------------------
40 bits = 5 bytes

```

举例：
Datetime value: ‘1970-1-1 00:00:00’

* innodb data: 0x99    0x02    0xc2    0x00    0x00

Datetime value: ‘2019-12-19 03:14:07’

* innodb data: 0x99    0xa4    0xe6    0x33    0x87

## 查看落盘数据格式

还有其他常用字段大家可以通过前文的方法，自行学习下。在了解了不同字段的存储格式后，我们也可以从InnoDB落盘数据上得到验证。

`hexdump -C -v table.ibd > table.txt
`

找到对应table的数据文件，用hexdump把table数据以hex方式打印到一个文本文件内，然后就可以用编辑器打开浏览。这里可以结合上文中得到的不同字段hex的表示，在文本文件中搜寻。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)