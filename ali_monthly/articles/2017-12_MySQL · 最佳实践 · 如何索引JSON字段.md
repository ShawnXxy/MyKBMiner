# MySQL · 最佳实践 · 如何索引JSON字段

**Date:** 2017/12
**Source:** http://mysql.taobao.org/monthly/2017/12/09/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 12
 ](/monthly/2017/12)

 * 当期文章

 MySQL · 引擎特性 · InnoDB 事务系统
* MySQL · 引擎特性 · Innodb 锁子系统浅析
* MySQL · 特性分析 · LOGICAL_CLOCK 并行复制原理及实现分析
* PgSQL · 源码分析 · AutoVacuum机制之autovacuum launcher
* MSSQL · 最佳实践 · SQL Server备份策略
* MySQL · 最佳实践 · 一个“异常”的索引选择
* PgSQL · 内核开发 · 利用一致性快照迁移你的数据
* PgSQL · 应用案例 · 手机行业分析、决策系统设计-实时圈选、透视、估算
* MySQL · 最佳实践 · 如何索引JSON字段
* MySQL · myrocks · 相关tools介绍

 ## MySQL · 最佳实践 · 如何索引JSON字段 
 Author: 勋臣 

 ## 概述
MySQL从5.7.8起开始支持JSON字段，这极大的丰富了MySQL的数据类型。也方便了广大开发人员。但MySQL并没有提供对JSON对象中的字段进行索引的功能，至少没有直接对其字段进行索引的方法。本文将介绍利用MySQL 5.7中的虚拟字段的功能来对JSON对象中的字段进行索引。

### 示例数据

我们将基于下面的JSON对象进行演示

`{
 "id": 1, 
 "name": "Sally", 
 "games_played":{ 
 "Battlefield": {
 "weapon": "sniper rifle",
 "rank": "Sergeant V",
 "level": 20
 }, 
 "Crazy Tennis": {
 "won": 4,
 "lost": 1
 }, 
 "Puzzler": {
 "time": 7
 }
 }
 }
`

表的基本结构

`
CREATE TABLE `players` ( 
 `id` INT UNSIGNED NOT NULL,
 `player_and_games` JSON NOT NULL,
 PRIMARY KEY (`id`)
);

`

如果只是基于上面的表的结构我们是无法对JSON字段中的Key进行索引的。接下来我们演示如何借助虚拟字段对其进行索引

### 增加虚拟字段

虚拟列语法如下

`<type> [ GENERATED ALWAYS ] AS ( <expression> ) [ VIRTUAL|STORED ]
[ UNIQUE [KEY] ] [ [PRIMARY] KEY ] [ NOT NULL ] [ COMMENT <text> ]
`

在MySQL 5.7中，支持两种Generated Column，即Virtual Generated Column和Stored Generated Column，前者只将Generated Column保存在数据字典中（表的元数据），并不会将这一列数据持久化到磁盘上；后者会将Generated Column持久化到磁盘上，而不是每次读取的时候计算所得。很明显，后者存放了可以通过已有数据计算而得的数据，需要更多的磁盘空间，与Virtual Column相比并没有优势，因此，MySQL 5.7中，不指定Generated Column的类型，默认是Virtual Column。

如果需要Stored Generated Golumn的话，可能在Virtual Generated Column上建立索引更加合适，一般情况下，都使用Virtual Generated Column，这也是MySQL默认的方式

加完虚拟列的建表语句如下：

`CREATE TABLE `players` ( 
 `id` INT UNSIGNED NOT NULL,
 `player_and_games` JSON NOT NULL,
 `names_virtual` VARCHAR(20) GENERATED ALWAYS AS (`player_and_games` ->> '$.name') NOT NULL, 
 PRIMARY KEY (`id`)
);
`
Note: 利用操作符-» 来引用JSON字段中的KEY。在本例中字段names_virtual为虚拟字段，我把它定义成不可以为空。在实际的工作中，一定要集合具体的情况来定。因为JSON本身是一种弱结构的数据对象。也就是说的它的结构不是固定不变的。

我们插入数据

`INSERT INTO `players` (`id`, `player_and_games`) VALUES (1, '{ 
 "id": 1, 
 "name": "Sally",
 "games_played":{ 
 "Battlefield": {
 "weapon": "sniper rifle",
 "rank": "Sergeant V",
 "level": 20
 }, 
 "Crazy Tennis": {
 "won": 4,
 "lost": 1
 }, 
 "Puzzler": {
 "time": 7
 }
 }
 }'
);
...

`

查看表里的数据

`SELECT * FROM `players`;

+----+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+
| id | player_and_games | names_virtual |
+----+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+
| 1 | {"id": 1, "name": "Sally", "games_played": {"Puzzler": {"time": 7}, "Battlefield": {"rank": "Sergeant V", "level": 20, "weapon": "sniper rifle"}, "Crazy Tennis": {"won": 4, "lost": 1}}} | Sally |
| 2 | {"id": 2, "name": "Thom", "games_played": {"Puzzler": {"time": 25}, "Battlefield": {"rank": "Major General VIII", "level": 127, "weapon": "carbine"}, "Crazy Tennis": {"won": 10, "lost": 30}}} | Thom |
| 3 | {"id": 3, "name": "Ali", "games_played": {"Puzzler": {"time": 12}, "Battlefield": {"rank": "First Sergeant II", "level": 37, "weapon": "machine gun"}, "Crazy Tennis": {"won": 30, "lost": 21}}} | Ali |
| 4 | {"id": 4, "name": "Alfred", "games_played": {"Puzzler": {"time": 10}, "Battlefield": {"rank": "Chief Warrant Officer Five III", "level": 73, "weapon": "pistol"}, "Crazy Tennis": {"won": 47, "lost": 2}}} | Alfred |
| 5 | {"id": 5, "name": "Phil", "games_played": {"Puzzler": {"time": 7}, "Battlefield": {"rank": "Lt. Colonel III", "level": 98, "weapon": "assault rifle"}, "Crazy Tennis": {"won": 130, "lost": 75}}} | Phil |
| 6 | {"id": 6, "name": "Henry", "games_played": {"Puzzler": {"time": 17}, "Battlefield": {"rank": "Captain II", "level": 87, "weapon": "assault rifle"}, "Crazy Tennis": {"won": 68, "lost": 149}}} | Henry |
+----+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------+
`

查看表Players的字段

`SHOW COLUMNS FROM `players`;

+------------------+------------------+------+-----+---------+-------------------+
| Field | Type | Null | Key | Default | Extra |
+------------------+------------------+------+-----+---------+-------------------+
| id | int(10) unsigned | NO | PRI | NULL | |
| player_and_games | json | NO | | NULL | |
| names_virtual | varchar(20) | NO | | NULL | VIRTUAL GENERATED |
+------------------+------------------+------+-----+---------+-------------------+
`
我们看到虚拟字段names_virtual的类型是VIRTUAL GENERATED。MySQL只是在数据字典里保存该字段元数据，并没有真正的存储该字段的值。这样表的大小并没有增加。我们可以利用索引把这个字段上的值进行物理存储。

### 在虚拟字段上加索引

再添加索引之前，让我们先看下面查询的执行计划

`EXPLAIN SELECT * FROM `players` WHERE `names_virtual` = "Sally"\G 
*************************** 1. row ***************************
 id: 1
 select_type: SIMPLE
 table: players
 partitions: NULL
 type: ALL
possible_keys: NULL 
 key: NULL
 key_len: NULL
 ref: NULL
 rows: 6
 filtered: 16.67
 Extra: Using where
`

添加索引

`CREATE INDEX `names_idx` ON `players`(`names_virtual`); 

`
再执行上面的查询语句，我们将得到不一样的执行计划

`EXPLAIN SELECT * FROM `players` WHERE `names_virtual` = "Sally"\G 
*************************** 1. row ***************************
 id: 1
 select_type: SIMPLE
 table: players
 partitions: NULL
 type: ref
possible_keys: names_idx 
 key: names_idx
 key_len: 22
 ref: const
 rows: 1
 filtered: 100.00
 Extra: NULL
`
如我们所见，最新的执行计划走了新建的索引。

## 小结
本文介绍了如何在MySQL 5.7中保存JSON文档。为了高效的检索JSON中内容，我们可以利用5.7的虚拟字段来对JSON的不同的KEY来建索引。极大的提高检索的速度。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)