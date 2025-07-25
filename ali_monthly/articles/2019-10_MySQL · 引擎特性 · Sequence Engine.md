# MySQL · 引擎特性 · Sequence Engine

**Date:** 2019/10
**Source:** http://mysql.taobao.org/monthly/2019/10/05/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2019 / 10
 ](/monthly/2019/10)

 * 当期文章

 MySQL · 引擎特性 · Innodb 表空间
* MySQL · 引擎特性 · POLARDB 并行查询加速全程详解
* MySQL · Optimizer · Parallel Index Scans, One is Better Than Two
* MySQL · 最佳实践 · X-Engine MySQL RDS 用户的新选择
* MySQL · 引擎特性 · Sequence Engine
* PgSQL · 应用案例 · 分组提交的使用与注意
* MongoDB · 最佳实践 · Spark Connector 实战指南
* PgSQL · 应用案例 · PG 12 tpcc - use sysbench-tpcc by Percona-Lab
* PgSQL · 应用案例 · 阿里云RDS PG 11开放dblink, postgres_fdw权限
* PgSQL · 应用案例 · Oracle 20c 新特性 - 翻出了PG十年前的特性

 ## MySQL · 引擎特性 · Sequence Engine 
 Author: lengxiang 

 ## Introduction
单调递增的唯一值，是在持久化数据库系统中常见的需求，无论是单节点中的业务主键，还是分布式系统中的全局唯一值，亦或是多系统中的幂等控制。不同的数据库系统有不同的实现方法，比如MySQL提供的AUTO_INCREMENT，Oracle，SQL Server 提供 SEQUENCE 等。

在 MySQL 数据库中，如果业务系统希望封装唯一值，比如增加日期，用户等信息，AUTO_INCREMENT 的方法会带来很大的不便，在实际的系统设计的时候, 也存在不同的折中方法，比如：

* 序列值由 Application 或者 Proxy 来生成，不过弊端很明显，状态带到应用端，增加了扩容和缩容的复杂度。
* 序列值由数据库通过模拟的表来生成，但需要中间件来封装和简化获取唯一值的逻辑。

AliSQL 自主实现了 SEQUENCE ENGINE，通过引擎的设计方法，尽可能的兼容其他数据库的使用方法，简化获取序列值复杂度。

## Description
AliSQL 支持的 SEQUENCE，实现了MySQL存储引擎的设计接口，但底层的数据仍然使用现有的存储引擎，比如 InnoDB 或者 MyISAM 来保存持久化数据，以便尽可能的保证现有的外围工具比如XtraBackup等工具的兼容，所以 SEQUENCE ENGINE 仅仅是一个逻辑引擎。

对 sequence 对象的访问通过 SEQUENCE handler 接口，这一层逻辑引擎主要实现 NEXTVAL 的滚动，CACHE 的管理等，最后透传给底层的基表数据引擎，实现最终的数据访问。

下面我们透过语法来看下 AliSQL SEQUENCE 的使用。

### Syntax
**1. CREATE SEQUENCE Syntax:** 

`CREATE SEQUENCE [IF NOT EXISTS] schema.sequence_name
[START WITH <constant>]
[MINVALUE <constant>]
[MAXVALUE <constant>]
[INCREMENT BY <constant>]
[CACHE <constant> | NOCACHE]
[CYCLE | NOCYCLE]
;
`
SEQUENCE OPTIONS:

* START

Sequence的起始值
* MINVALUE

Sequence的最小值，如果这一轮结束并且是cycle的，那么下一轮将从MINVALUE开始
* MAXVALUE

Sequence的最大值，如果到最大值并且是nocycle的，那么将会得到以下报错：

`ERROR HY000: Sequence 'db.seq' has been run out.`
* INCREMENT BY

Sequence的步长
* CACHE/NOCACHE

Cache的大小，为了性能考虑，可以设置cache的size比较大，但如果遇到实例重启，cache内的值会丢失
* CYCLE/NOCYCLE

表示sequence如果用完了后，是否允许从MINVALUE重新开始

例如：

`create sequence s
start with 1
minvalue 1
maxvalue 9999999
increment by 1
cache 20
cycle;
`

**2. SHOW SEQUENCE Syntax** 

`SHOW CREATE [TABLE|SEQUENCE] schema.sequence_name;

CREATE SEQUENCE schema.sequence_name (
 `currval` bigint(21) NOT NULL COMMENT 'current value',
 `nextval` bigint(21) NOT NULL COMMENT 'next value',
 `minvalue` bigint(21) NOT NULL COMMENT 'min value',
 `maxvalue` bigint(21) NOT NULL COMMENT 'max value',
 `start` bigint(21) NOT NULL COMMENT 'start value',
 `increment` bigint(21) NOT NULL COMMENT 'increment value',
 `cache` bigint(21) NOT NULL COMMENT 'cache size',
 `cycle` bigint(21) NOT NULL COMMENT 'cycle state',
 `round` bigint(21) NOT NULL COMMENT 'already how many round'
 ) ENGINE=InnoDB DEFAULT CHARSET=latin1
`

由于SEQUENCE是通过真正的引擎表来保存的，所以SHOW COMMAND看到仍然是engine table。

**3. QUERY STATEMENT Syntax** 

`SELECT [nextval | currval | *] FROM seq;
SELECT nextval(seq),currval(seq);
SELECT seq.currval, seq.nextval from dual;
`

**4. 兼容性** 

因为要兼容MYSQLDUMP的备份方式，所以支持另外一种CREATE SEQUENCE方法，即：通过创建SEQUENCE表和INSERT一行初始记录的方式, 比如：

`CREATE SEQUENCE schema.sequence_name (
 `currval` bigint(21) NOT NULL COMMENT 'current value',
 `nextval` bigint(21) NOT NULL COMMENT 'next value',
 `minvalue` bigint(21) NOT NULL COMMENT 'min value',
 `maxvalue` bigint(21) NOT NULL COMMENT 'max value',
 `start` bigint(21) NOT NULL COMMENT 'start value',
 `increment` bigint(21) NOT NULL COMMENT 'increment value',
 `cache` bigint(21) NOT NULL COMMENT 'cache size',
 `cycle` bigint(21) NOT NULL COMMENT 'cycle state',
 `round` bigint(21) NOT NULL COMMENT 'already how many round'
 ) ENGINE=InnoDB DEFAULT CHARSET=latin1

INSERT INTO schema.sequence_name VALUES(0,0,1,9223372036854775807,1,1,10000,1,0);
COMMIT;
`

但强烈建议使用native的CREATE SEQUENCE方法。

**5. 语法限制** 

* Sequence不支持 subquery 和 join
* 可以使用SHOW CREATE TABLE或者SHOW CREATE SEQUENCE来访问SEQUENCE结构，但不能使用SHOW CREATE SEQUENCE 访问普通表
* 不支持 CREATE TABLE 的时候指定 SEQUENCE 引擎，sequence 表只能通过 CREATE SEQUENCE 的语法来创建

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)