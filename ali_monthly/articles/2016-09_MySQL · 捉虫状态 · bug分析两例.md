# MySQL · 捉虫状态 · bug分析两例

**Date:** 2016/09
**Source:** http://mysql.taobao.org/monthly/2016/09/06/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 09
 ](/monthly/2016/09)

 * 当期文章

 MySQL · 社区贡献 · AliSQL那些事儿
* PetaData · 架构体系 · PetaData第二代低成本存储体系
* MySQL · 社区动态 · MariaDB 10.2 前瞻
* MySQL · 特性分析 · 执行计划缓存设计与实现
* PgSQL · 最佳实践 · pg_rman源码浅析与使用
* MySQL · 捉虫状态 · bug分析两例
* PgSQL · 源码分析 · PG优化器浅析
* MongoDB · 特性分析· Sharding原理与应用
* PgSQL · 源码分析 · PG中的无锁算法和原子操作应用一则
* SQLServer · 最佳实践 · TEMPDB的设计

 ## MySQL · 捉虫状态 · bug分析两例 
 Author: 济天 

 ## BUG 1 IN查询结果不对

### 背景

在mysql5.6.16版本下，构建如下测试用例

`CREATE TABLE `a` (
`c1` varchar(512) NOT NULL DEFAULT ''
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
INSERT INTO `a` VALUES ('i-28s18atup'),('i-2850jdoa2'),('i-2872jv9o8'),('i-289z59set'),('i-2812c0mz1'),(''),(''),('i-28xut6ybi'),('i-28w4b8qmq'),('i-289x1nfxb'),('i-28ae3hs3l'),('');

CREATE TABLE `b` (
`c1` varchar(512) NOT NULL DEFAULT ''
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO `b` VALUES ('i-2872jv9o8'),('i-289z59set'),('i-2812c0mz1'),('i-2816iktqq'),('i-2817bpwuf'),('i-281hwn59l'),('i-281uacgga'),('i-2857ezdnp'),('i-288mhwbtf'),('i-28gpk1d8n'),('i-28imr0oqh'),('i-28k90goqz'),('i-28ljdyl40'),('i-28n95axfc'),('i-28oud8zm4'),('i-28poeqcfg'),('i-28s4qqv37'),('i-28sqirs1x'),('i-28teqx5wl'),('i-28tscpaga'),('i-28udhuliu'),('i-28ukb9exr'),('i-28weof7iq'),('i-28x2ks3t0'),('i-28x4dwhx2'),('i-28xkvidhs'),('i-28yxqwfn3'),('i-280g6jlvv'),('i-281uto1kh'),('i-287sdi05n'),('i-289mdvf5g'),('i-289u2lqsc'),('i-28aay99cm'),('i-28egwihhf'),('i-28f6rjma6'),('i-28gkqmzyr'),('i-28i0fxf1s'),('i-28luhd7ps'),('i-28lv319r0'),('i-28n3qoup1'),('i-28nwfea7f'),('i-28ot1q2xi'),('i-28pyi35g8'),('i-28qu9xsi3'),('i-28rea4oni'),('i-28u67zgrf'),('i-28v3b30me'),('i-28vmsbvfo'),('i-28wjg9vk6'),('i-28y57ow32'),('i-28zvbc1ez'),('i-2808tzqlt'),('i-280lh3rs8'),('i-280qx4wi4'),('i-2822dphhh'),('i-282bi6laz'),('i-286vzupf5'),('i-289j094i1'),('i-28alrnwls'),('i-28cv4vbno'),('i-28f9b4h8i'),('i-28gzw4zy7'),('i-28k22cqsp'),('i-28kks1i6x'),('i-28kzegmw4'),('i-28l7zc3fk'),('i-28mixznev'),('i-28ozch1ez'),('i-28pb5f875'),('i-28ph2u6t6'),('i-28pqnqp7j'),('i-28qcpitrp'),('i-28qqnhqfh'),('i-28u22af72'),('i-28uj4p5tv'),('i-281ivhmz5'),('i-2822cx06o'),('i-282mbjqm0'),('i-283d8bvyb'),('i-283quwmu3'),('i-286g6g1ro'),('i-288j0bnt7'),('i-28by5aph5'),('i-28eieau49'),('i-28f62bsn1'),('i-28hf1eagg'),('i-28i65v11j'),('i-28jxqptk6'),('i-28larsas5'),('i-28opsiq8x'),('i-28phn0vxr'),('i-28qvnojgc'),('i-28suce82g'),('i-28ugoo095'),('i-28vezpnzi'),('i-2807j8ugs'),('i-280by00qq'),('i-280il7op3'),('i-280jkmjaz'),('i-280ntguko'),('i-280oek885'),('i-281bqee7p'),('i-281on7t0b'),('i-281qw9cxp'),('i-281yhxvsv'),('i-281z84d1e'),('i-2822x8yps'),('i-282dkngx2'),('i-282r9e38f'),('i-282rnc33f'),('i-282v6g5h4'),('i-282xiua07'),('i-283182ago'),('i-2832s21gb'),('i-2834tiucp'),('i-283cn5cxq'),('i-283wi9g1h'),('i-283z5b9c0'),('i-2841ipeb4'),('i-28437k5oc'),('i-2846zaxqr'),('i-284984ozz'),('i-284a1ssib'),('i-2855d5g88'),('i-285j5p5bk'),('i-285mc530r'),('i-285x7qkd3'),('i-286ge4k0u'),('i-286jo1dlm'),('i-286l67owf'),('i-286l9nw58'),('i-286sfdy4l'),('i-286wqq5ru'),('i-28782gcz0'),('i-287emfglt'),('i-287gte4x2'),('i-287iojtlb'),('i-287iuvbak'),('i-287krjdsj'),('i-287xw6apz'),('i-288qvr8nc'),('i-288r3rnp5'),('i-288vyt7rn'),('i-28945gqjc'),('i-289o6exr0'),('i-289plb8da'),('i-289zzqybq'),('i-28a7s5emw'),('i-28a80onkx'),('i-28ac0jtrv'),('i-28aljhpg7'),('i-28anbj66d'),('i-28aph3ftl'),('i-28avc59am'),('i-28axepg7l'),('i-28basvzz1'),('i-28bc4rrlf'),('i-28bexmi8s'),('i-28bibye65'),('i-28bt7b3e0'),('i-28c7xkjcx'),('i-28cavwsws'),('i-28d1k1f9a'),('i-28d6vdy8y'),('i-28dbchfjy'),('i-28dnywe05'),('i-28dp7a49v'),('i-28e020whv'),('i-28eeda8sl'),('i-28eei9ril'),('i-28efkucv4'),('i-28eqlvjh7'),('i-28eqoq6jz'),('i-28eu8b3cn'),('i-28ey2azsi'),('i-28ezvmkfw'),('i-28flibcp6'),('i-28fx0xpcy'),('i-28fxgaiky'),('i-28g3txm3b'),('i-28g3v1iry'),('i-28g9b6020'),('i-28gb1btyc'),('i-28gb3ycy0'),('i-28gedglzn'),('i-28gltwwep'),('i-28gq7in88'),('i-28gyuxcbh'),('i-28h3iifgj'),('i-28h62fexn'),('i-28h6aoaad'),('i-28h6t1m4y'),('i-28hmc6k68'),('i-28hwfarg5'),('i-28ia4sfjq'),('i-28iahbwm8'),('i-28ixabmik'),('i-28j9zlf7i'),('i-28jszuxbi'),('i-28jve5u7y'),('i-28jxfhc0t'),('i-28k065l2d'),('i-28k8izbox'),('i-28kkduhlu'),('i-28lds3u8j'),('i-28ll2kzzv'),('i-28lmdk35t'),('i-28lnwhrp9'),('i-28lyjao0s'),('i-28m89ppr9'),('i-28mgx6z78'),('i-28mmrax6a'),('i-28mn693s6'),('i-28mr3daqe'),('i-28msqlshi'),('i-28ne2cj29'),('i-28njhhqlx'),('i-28nrmz2xi'),('i-28obphyxu'),('i-28oqcp376'),('i-28p8gdsa1'),('i-28pkmbhz0'),('i-28pm3ae7s'),('i-28pq0fyad'),('i-28pt7gxr9'),('i-28q3c1uwv'),('i-28qbdjifr'),('i-28qbupqww'),('i-28qeldn3h'),('i-28qh9hm6x'),('i-28qhuqgrb'),('i-28qq7xquk'),('i-28qsp23rw'),('i-28rbr6n7j'),('i-28rd5sit0'),('i-28re1wghi'),('i-28rg46u2o'),('i-28ro15iho'),('i-28rr8t7e9'),('i-28rtwybki'),('i-28s1a15nf'),('i-28s4lcrrk'),('i-28s6gmfdr'),('i-28s75rgku'),('i-28spofoco'),('i-28spzksyq'),('i-28sre8jn1'),('i-28t0bhx0s'),('i-28t5jr47m'),('i-28tf05t7d'),('i-28tqmqj5f'),('i-28tsv0bbf'),('i-28u4wrz31'),('i-28ufrk8ah'),('i-28uoipuyh'),('i-28ur0wkpc'),('i-28uribj7j'),('i-28ut2dbya'),('i-28uv1pvfn'),('i-28v16yjyb'),('i-28v7dt011'),('i-28vj5l7r9'),('i-28vpwpxm8'),('i-28wga3np5'),('i-28wjoby7z'),('i-28wnnsjjt'),('i-28x0ikpj1'),('i-28x70dsak'),('i-28xbvelc6'),('i-28xivp7gj'),('i-28y3qaea3'),('i-28yh1ownr'),('i-28ysliksd'),('i-28zyq34ab'),('i-28xut6ybi'),('i-28w4b8qmq');

select c1 from a where c1='i-28w4b8qmq';
+-------------+
| c1 |
+-------------+
| i-28w4b8qmq |
+-------------+
1 row in set (0.01 sec)

select c1 from b where c1='i-28w4b8qmq';
+-------------+
| c1 |
+-------------+
| i-28w4b8qmq |
+-------------+

set tmp_table_size=16*1024*1024;
explain select c1 from a where c1='i-28w4b8qmq' and c1 in (select c1 from b);
+----+--------------+-------------+--------+---------------+------------+---------+---------------+------+-------------+
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
+----+--------------+-------------+--------+---------------+------------+---------+---------------+------+-------------+
| 1 | SIMPLE | a | ALL | NULL | NULL | NULL | NULL | 12 | Using where |
| 1 | SIMPLE | <subquery2> | eq_ref | <auto_key> | <auto_key> | 1538 | cleaneye.a.c1 | 1 | NULL |
| 2 | MATERIALIZED | b | ALL | NULL | NULL | NULL | NULL | 276 | NULL |
+----+--------------+-------------+--------+---------------+------------+---------+---------------+------+-------------+
select c1 from a where c1='i-28w4b8qmq' and c1 in (select c1 from b);
c1
+-------------+
| c1 |
+-------------+
| i-28w4b8qmq |
+-------------+

set tmp_table_size=262144;
explain select c1 from a where c1='i-28w4b8qmq' and c1 in (select c1 from b);
+----+--------------+-------------+--------+---------------+------------+---------+---------------+------+-------------+
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | Extra |
+----+--------------+-------------+--------+---------------+------------+---------+---------------+------+-------------+
| 1 | SIMPLE | a | ALL | NULL | NULL | NULL | NULL | 12 | Using where |
| 1 | SIMPLE | <subquery2> | eq_ref | <auto_key> | <auto_key> | 1538 | cleaneye.a.c1 | 1 | NULL |
| 2 | MATERIALIZED | b | ALL | NULL | NULL | NULL | NULL | 276 | NULL |
+----+--------------+-------------+--------+---------------+------------+---------+---------------+------+-------------+
select c1 from a where c1='i-28w4b8qmq' and c1 in (select c1 from b); // 查询结果为空，预期结果应返回一行
Empty set (0.01 sec)

`

从例子可以看到当tmp_table_size=262144时，查询结果不对，而tmp_table_size=16*1024*1024时查询结果是正确的。

### 分析
查询结果跟tmp_table_size有关，而tmp_table_size是控制查询时生成的临时表是MEMORY还是MyISAM类型的。临时表超过tmp_table_size会生成MyISAM类型的。
另外，查询计划中可以看到MATERIALIZED，MATERIALIZED优化是将子查询的结果存储在临时表中，避免多次执行子查询。
而5.6存在一个问题是，MATERIALIZED对MyISAM临时表中存在unique constraint的情况支持不友好。此问题直到5.7才解决，参见[worklog#6711](https://dev.mysql.com/worklog/task/?id=6711)

 MyISAM表中unique constraint，是指当MyISAM中存在blob,或key长度超长时， 索引会自动转为unique constraint
unique constraint会自动增加一列存记录的hash值, 以下是代码片段

`create_myisam_tmp_table:

if (keyinfo->key_length >= table->file->max_key_length() ||
 keyinfo->user_defined_key_parts > table->file->max_key_parts() ||
 share->uniques)
{
 /* Can't create a key; Make a unique constraint instead of a key */
 share->keys= 0;
 share->uniques= 1;
 using_unique_constraint=1;
 memset(&uniquedef, 0, sizeof(uniquedef));
 uniquedef.keysegs=keyinfo->user_defined_key_parts;
 uniquedef.seg=seg;
 uniquedef.null_are_equal=1;

 /* Create extra column for hash value */
 memset(*recinfo, 0, sizeof(**recinfo));
 (*recinfo)->type= FIELD_CHECK;
 (*recinfo)->length=MI_UNIQUE_HASH_LENGTH;
 (*recinfo)++;
 share->reclength+=MI_UNIQUE_HASH_LENGTH;
`

而本例中，c1字段varchar(512)是utf8字符集 512*3 > 1000 超过最大key大小，正好进入上述代码逻辑。

子查询先物化到此临时表，然后后续查询从临时表读数据，然而这里对unique constraint的读取操作(ha_myisam::index_read_map)还不支持

`ha_myisam::index_read_map
handler::ha_index_read_map 
join_read_key 
sub_select
evaluate_join_record 
sub_select
do_select
JOIN::exec
mysql_execute_select
mysql_select 
handle_select
`
读取出错后得到HA_ERR_KEY_NOT_FOUND，此错误并没有退出。从而导致外界认为查询结果为空集，所以我们测例中的查询也错误的返回了空集。

`error= table->file->ha_index_read_map(table->record[0],
 tab->ref.key_buff,
 make_prev_keypart_map(tab->ref.key_parts), HA_READ_KEY_EXACT);
if (error &&
 error != HA_ERR_KEY_NOT_FOUND && error != HA_ERR_END_OF_FILE)
 error= report_handler_error(table, error);
else{
`

### 修复
虽然5.7修复了此问题，但改动较大，同时还存在部分bug.
5.6采取了折衷的修复方法，当MyISAM临时表存在unique contraint时，则不采用MATERIALIZED优化，从而避免了产生MyISAM临时表。
修复方法参见[bugfix](https://github.com/mysql/mysql-server/commit/a24950aca8530fe04782e24de1d40a91e1ec023f)

## BUG 2 表不存在

### 现象
从备份集中恢复的实例中查询发现表不存在，但实际frm,idb文件都存在

`select * from t2;
Table 'test.t2' doesn't exist
`
从备份的源实例中查看，貌似一切正常

`show create table t2;
CREATE TABLE `t2` (
 `c1` int(11) NOT NULL,
 `c2` int(11) DEFAULT NULL,
 PRIMARY KEY (`c1`),
 CONSTRAINT `t2_ibfk_1` FOREIGN KEY (`c2`) REFERENCES `t1` (`c1`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1

show create table t1;
Create Table
t1 CREATE TABLE `t1` (
 `c1` int(11) NOT NULL,
 `c2` int(11) DEFAULT NULL,
 PRIMARY KEY (`c1`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
`

### 分析
首先，怀疑是备份集的问题。因此，重新备份后再测试，依然是表不存在。
 排除备份集的问题后，通过调试源码的方法来找原因

`dict_foreign_qualify_index
dict_foreign_find_index
dict_foreign_add_to_cache
dict_load_foreign 
dict_load_foreigns
dict_load_table
dict_table_open_on_name
ha_innobase::open
handler::ha_open
open_table_from_share
open_table
open_and_process_table
open_tables
open_normal_and_derived_tables
execute_sqlcom_select
mysql_execute_command
`
上述堆栈是表在load过程中构建foreign信息。查找t2表列c2的foreign key时失败了。

查看INNODB_SYS_INDEXES表也可以发现，t2上只有一个primary索引，外键约束虽在，但外键在t2上的索引并不存在。

`select * from information_schema.INNODB_SYS_TABLES t,information_schema.INNODB_SYS_indexes i,information_schema.INNODB_SYS_fields f where t.name='test/t2' and t.table_id=i.table_id and i.index_id=f.index_id;
+----------+---------+------+--------+-------+-------------+------------+---------------+----------+---------+----------+------+----------+---------+-------+----------+------+-----+
| TABLE_ID | NAME | FLAG | N_COLS | SPACE | FILE_FORMAT | ROW_FORMAT | ZIP_PAGE_SIZE | INDEX_ID | NAME | TABLE_ID | TYPE | N_FIELDS | PAGE_NO | SPACE | INDEX_ID | NAME | POS |
+----------+---------+------+--------+-------+-------------+------------+---------------+----------+---------+----------+------+----------+---------+-------+----------+------+-----+
| 21 | test/t2 | 1 | 5 | 7 | Antelope | Compact | 0 | 23 | PRIMARY | 21 | 3 | 1 | 3 | 7 | 23 | c1 | 0 |
+----------+---------+------+--------+-------+-------------+------------+---------------+----------+---------+----------+------+----------+---------+-------+----------+------+-----+

select * from information_schema.INNODB_SYS_FOREIGN f,information_schema.INNODB_SYS_FOREIGN_COLS fc where f.for_name='test/t2' and f.id=fc.id;
+----------------+----------+----------+--------+------+----------------+--------------+--------------+-----+
| ID | FOR_NAME | REF_NAME | N_COLS | TYPE | ID | FOR_COL_NAME | REF_COL_NAME | POS |
+----------------+----------+----------+--------+------+----------------+--------------+--------------+-----+
| test/t2_ibfk_1 | test/t2 | test/t1 | 1 | 0 | test/t2_ibfk_1 | c2 | c1 | 0 |
+----------------+----------+----------+--------+------+----------------+--------------+--------------+-----+

`

### 场景恢复
为什么外键在t2上的索引会不存在呢，是参数FOREIGN_KEY_CHECKS捣的鬼，我们看看如下例子

`create table t1(c1 int primary key, c2 int) engine=innodb;
create table t2(c1 int primary key, c2 int , key idx1(c2), foreign key (c2) references t1(c1)) engine=innodb;
set FOREIGN_KEY_CHECKS=1;
alter table t2 drop key idx1;
ERROR 1553 (HY000): Cannot drop index 'c2': needed in a foreign key constraint

set FOREIGN_KEY_CHECKS=0;
alter table t2 drop key idx1;//可以删除成功

`
删除后表还是可以正常访问的。但一旦表定义踢出缓存或数据库重启，重新加载数据字典信息时，就会出现前面堆栈中的找不到外键索引的问题，从而导致表不存在的错误。

这个错误是非常严重的，会导致用户无法访问数据。

一旦出现此种情况，需要修改代码绕过加载外键数据字典信息的错误，才能恢复出数据，比较麻烦。

而对于我们源实例的场景算比较幸运，t2的表定义还在内存中，这时只需要把idx1重新建回去即可。再重新备份就可以生成有效的备份集了。

### 修复
此bug出现在版本5.5,5.6中，5.7已修复参考[bug#76940](http://bugs.mysql.com/bug.php?id=76940)

修复方法是FOREIGN_KEY_CHECKS=0时不允许删除外键所在索引。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)