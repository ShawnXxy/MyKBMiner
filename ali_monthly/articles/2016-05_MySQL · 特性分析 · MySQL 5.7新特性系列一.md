# MySQL · 特性分析 · MySQL 5.7新特性系列一

**Date:** 2016/05
**Source:** http://mysql.taobao.org/monthly/2016/05/02/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 05
 ](/monthly/2016/05)

 * 当期文章

 MySQL · 引擎特性 · 基于InnoDB的物理复制实现
* MySQL · 特性分析 · MySQL 5.7新特性系列一
* PostgreSQL · 特性分析 · 逻辑结构和权限体系
* MySQL · 特性分析 · innodb buffer pool相关特性
* PG&GP · 特性分析 · 外部数据导入接口实现分析
* SQLServer · 最佳实践 · 透明数据加密在SQLServer的应用
* MySQL · TokuDB · 日志子系统和崩溃恢复过程
* MongoDB · 特性分析 · Sharded cluster架构原理
* PostgreSQL · 特性分析 · 统计信息计算方法
* MySQL · 捉虫动态 · left-join多表导致crash

 ## MySQL · 特性分析 · MySQL 5.7新特性系列一 
 Author: lengxiang 

 ## 1. 背景
MySQL 5.7在2015-10-21发布了GA版本，即5.7.9，目前小版本已经到了5.7.12。5.7新增了许多新的feature和优化，接下来一个系列，我们就一起来尝尝鲜。首先这次主要是预览feature的变化以及兼容性问题。后面的系列，会针对重要的feature展开来学习。

## 2 安全相关的特性

### 2.1 认证插件
mysql.user表中的plugin更改成not null，5.7开始不再支持mysql_old_password的认证插件，推荐全部使用mysql_native_password。从低版本升级到5.7的时候，需要处理两个兼容性问题。

**[兼容性]**
需要先迁移mysql_old_password的用户，然后进行user表结构的升级：

**1. 迁移mysql_old_password用户**
MySQL 5.7.2之前的版本，是根据password的hash value来判断使用的认证插件类型，5.7.2以后的版本，plugin字段为not null，就直接根据plugin来判断了。新的密码从password字段中，保存到新的字段authentication_string中，password字段废弃处理。

如果user是隐式的mysql_native_password。直接使用sql进行变更：

`UPDATE mysql.user SET plugin = 'mysql_native_password' WHERE plugin = '' AND (Password = '' OR LENGTH(Password) = 41);
FLUSH PRIVILEGES;
`
如果user是隐式的或者显示的mysql_old_password， 首先通过以下sql进行查询：

`SELECT User, Host, Password FROM mysql.user WHERE (plugin = '' AND LENGTH(Password) = 16) OR plugin = 'mysql_old_password';
`
如果存在记录，就表示还有使用mysql_old_password的user，使用以下sql进行用户的迁移：

`ALTER USER 'user1'@'localhost' IDENTIFIED WITH mysql_native_password BY 'DBA-chosen-password';
`

**2. user表结构升级**
通过mysql_upgrade直接进行升级，步骤如下[5.6->5.7]：

1. stop MySQL 5.6实例
2. 替换5.7的mysqld二进制版本
3. 使用5.7启动实例
4. run mysql_upgrade升级系统表
5. 重启MySQL 5.7实例

### 2.2 密码过期
用户可以通过 `ALTER USER 'jeffrey'@'localhost' PASSWORD EXPIRE;`这样的语句来使用户的密码过期。
并新增加 default_password_lifetime来表示用户密码自动过期时间，从5.7.10开始，其默认值从0变更到了360，也就是默认一年过期。
可以通过以下两种方法禁止过期：

`1. SET GLOBAL default_password_lifetime = 0;
2. ALTER USER 'jeffrey'@'localhost' PASSWORD EXPIRE NEVER;
`

**[兼容性]**
只需要通过mysql_upgrade升级mysql.user系统表就可以使用密码过期新功能。

### 2.3 账号锁定
用户可以通过以下语法进行账号锁定，阻止这个用户进行登录：

`ALTER USER 'jeffrey'@'localhost' ACCOUNT LOCK;
ALTER USER 'jeffrey'@'localhost' ACCOUNT UNLOCK;
`
**[兼容性]**
只需要通过mysql_upgrade升级mysql.user系统表就可以使用密码过期新功能。

### 2.4 SSL连接
如果mysqld编译使用的openssl，在启动的时候，默认创建SSL， RSA certificate 和 key 文件。
但不管是openssl还是yassl，如果没有设置ssl相关的参数，mysqld都会在data directory里查找ssl认证文件，来尽量打开ssl特性。

**[兼容性]**
不存在兼容性的问题

### 2.5 安装数据库
5.7开始建议用户使用 `mysqld --initialize`来初始化数据库，放弃之前的mysql_install_db的方式，新的方式只创建了一个root@localhost的用户，随机密码保存在~/.mysql_secret文件中，并且账号是expired，第一次使用必须reset password，并且不再创建test db。

**[兼容性]**
不存在兼容性的问题

## 3 sql mode变更
5.7 sql_mode的默认值变更为：

`mode_no_engine_substitution |
 mode_only_full_group_by |
 mode_strict_trans_tables |
 mode_no_zero_in_date |
 mode_no_zero_date |
 mode_error_for_division_by_zero |
 mode_no_auto_create_user
`

而在5.7之前，sql_mode的默认值都只有mode_no_engine_substitution。
所以在5.7默认的情况下，比如grant不存在的用户的时候，会报一下错误：

`ERROR 1133 (42000): Can't find any matching row in the user table
`
必须先使用create user，然后再使用grant user。

**[兼容性]**
默认sql mode发生变更会导致sql的行为不一致。

## 4. online alter table
支持online rename index操作， in_place并且不需要table copy。
**[兼容性]**
不存在兼容性的问题

## 5. InnoDB增强

### 5.1 varchar长度变更支持inplace
变更varchar 类型字段的长度支持inplace方法，但有一个限制，即用于表示varchar字段长度的字节数不能发生变化，也就是支持比如varchar的长度在255以下变更或者255以上的范围进行变更，因为从小于255变更到大于255，其size的字节需要从1个增加到2个。

注意：减少varchar的长度，仍然需要table copy。

### 5.2 优化InnoDB临时表
因为InnoDB临时表的数据不再不受redo保护，而redo只保护临时表的元数据，所以大幅提升了临时表的性能。
并且InnoDB临时表的元数据保存在一个新的系统表中即innodb_temp_table_info，
临时表将建立一个统一的表空间，我们称之为临时表空间，其目录地址可以通过参数innodb_temp_data_file_path来设置。系统在启动的时候，都会新建这个表空间，重启会删除重建。

例如：

`mysql> show global variables like '%temp_data_file_path%';
+----------------------------+-----------------------+
| Variable_name | Value |
+----------------------------+-----------------------+
| innodb_temp_data_file_path | ibtmp1:12M:autoextend |
+----------------------------+-----------------------+
`
并且5.7存储引擎默认都变更成InnoDB了：

`mysql> show global variables like '%storage_engine%';
+----------------------------------+--------+
| Variable_name | Value |
+----------------------------------+--------+
| default_storage_engine | InnoDB |
| default_tmp_storage_engine | InnoDB |
| disabled_storage_engines | |
| internal_tmp_disk_storage_engine | InnoDB |
+----------------------------------+--------+
`
**注意：** 在开启gtid的情况下，非auto commit或者显示begin的context下，create 或者drop 临时表，仍然和5.6一样：

`ERROR 1787 (HY000): Statement violates GTID consistency: CREATE TEMPORARY TABLE and DROP TEMPORARY TABLE can only be executed outside transactional context.
`
另外， insert into t select * from t也会遇到错误，不能在一个sql语句中reference两次临时表。

**备注：** 因为InnoDB临时表进行了比较大的变动，我们会专门进行一次详细的介绍。

### 5.3 InnoDB原生支持DATA_GEOMETRY类型
并且支持在spatial data types上建立index，加速查询。

### 5.4 buffer pool dump
buffer pool dump和load支持一个新的参数innodb_buffer_pool_dump_pct，即dump的比例，并且使用innodb_io_capacity 来控制load过程中的IO吞吐量。

### 5.5 多线程flush dirty
从5.7.4开始，innodb_page_cleaners参数可以设置，支持多线程flush dirty page，加快脏块的刷新。

### 5.6 NVM file system
MySQL 一直使用double write buffer来解决一个page写入的partial write问题，但在linux系统上的Fusion-io Non-Volatile Memory (NVM) file system支持原子的写入。
这样就可以省略掉double write buffer的使用， 5.7.4以后，如果Fusion-io devices支持atomic write，那么MySQL自动把dirty block直接写入到数据文件了。这样减少了一次内存copy和IO操作。

### 5.7 InnoDB分区表
MySQL 5.7之前的版本，InnoDB并不支持分区表，分区表的支持是在ha_partition引擎上支持的，从5.7开始，InnoDB支持原生的分区表，并且可以使用传输表空间。

**[兼容性]**
mysql_upgrade会扫描ha_partition引擎支持的InnoDB表，并升级成InnoDB分区表，5.7.9之后，可以通过命令ALTER TABLE … UPGRADE PARTITIONING.进行升级。如果之前的版本大量使用了分区表，要注意使用mysql_upgrade会消耗非常长的时间来升级分区表。

### 5.8 动态调整buffer pool size
MySQL 5.7.5之后，可以online动态调整buffer pool size，通过设置动态的参数innodb_buffer_pool_size来调整，并且根据Innodb_buffer_pool_resize_status状态来查看resize的进度，因为resize的过程是以chunk为大小，把pages从一个内存区域copy到另一片内存的。

### 5.9 加快recovery
MySQL 5.7.5之前，在recovery的过程中，需要扫描所有的ibd文件，获取元信息， 5.7.5之后，新加了一种redo log类型，即MLOG_FILE_NAME， 记录从上一次checkpoint以来，发生过变更的文件，这样在recovery的过程中，只需要打开这些文件就可以了。
**[兼容性]**
因为增加了新的log record type，需要安全的关闭5.7之前的实例，清理掉redo。

### 5.10 表空间管理
支持创建表空间，例如

`CREATE TABLESPACE `tablespace_name`
ADD DATAFILE 'file_name.ibd'
[FILE_BLOCK_SIZE = n]
`
并可以在创建表的时候，指定属于哪个表空间，

**[兼容性]**
因为可以任意指定空间目录，要注意升级过程中，不要漏掉目录。

### 5.11 InnoDB Tablespace Encryption
支持InnoDB数据文件加密，其依赖keyring plugin来进行秘钥的管理，后面我们单独来介绍InnoDB加密的方法，并且RDS也实现了一种InnoDB数据文件透明加密方法，并通过KMS系统来管理秘钥。例如：

`create table t(id int) encryption='y';
`

未完待续

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)