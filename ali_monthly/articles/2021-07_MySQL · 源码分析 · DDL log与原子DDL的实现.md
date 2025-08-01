# MySQL · 源码分析 · DDL log与原子DDL的实现

**Date:** 2021/07
**Source:** http://mysql.taobao.org/monthly/2021/07/05/
**Images:** 3 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2021 / 07
 ](/monthly/2021/07)

 * 当期文章

 POLARDB · 引擎特性 · Logic Redo
* MySQL · 源码分析 · btr_cur_search_to_nth_level 函数分析
* PostgreSQL · 内核特性 · 死锁检测与解决
* MySQL · 源码分析 · 条件优化与执行分析
* MySQL · 源码分析 · DDL log与原子DDL的实现
* MySQL · 功能介绍 · GIS功能介绍
* MySQL · 源码分析 · 临时表与TempTable存储引擎Allocator

 ## MySQL · 源码分析 · DDL log与原子DDL的实现 
 Author: 白芦 

 ## 背景

原子性，通俗来说就是指一条命令要么全部执行，要么全部不执行。

当我们使用MySQL来存储应用数据的时候，MySQL同样也需要存储这些元数据。在MySQL8.0之前的版本中，这些元数据被存放在许多不同的文件中（.FRM，.PAR，.OPT，.TRN，.TRG文件等），这就导致了一系列弊端，包括数据可能不一致、API接口的复杂性等等，在之前的月报[[5]](http://mysql.taobao.org/monthly/2018/03/02/)中也有详细描述。元数据被放在许多不同的文件中，导致数据可能不一致的具体表现为：

1. Server层的metadata和Storage Engine层的metadata数据不一致；
2. InnoDB中的metadata和数据不一致；
3. Binlog和数据不一致。

 ![](.img/ccff6b816c39_Traditional-MySQL-Data-Dictionary.png) 
 MySQL8.0之前元数据被持久化存储的方案 

也是由于上述原因，MySQL一开始并没能实现DDL的原子性操作，举例来说，我们创建表时如果发生crash，建表不完整，可能会遗留下ibd文件或者.frm文件，这些文件不仅浪费了表空间，还有可能对后续的DDL操作造成影响。

为了实现AtomicDDL，MySQL 8.0进行了大刀阔斧的改革，目前，只有InnoDB存储引擎支持原子DDL。

MySQL8.0之后，分散的元数据被统一存放在Data Dictionary中，用户、Server层、引擎都可以通过DD的访问接口查询或者更新Metadata。与DD表有关的源码阅读可以参考之前的月报[[5]](http://mysql.taobao.org/monthly/2018/03/02/)。

 ![](.img/990e8eee6dc8_DD-in-InnoDB-tables-now.png) 
 MySQL8.0中的元数据存储方案 

此外，（以下全针对InnoDB存储引擎）还引入了一个特殊的数据结构DDL_log。InnoDB中通过DDL_log来保证DDL的原子性。在DDL执行期间跟踪文件和结构的创建，然后在提交/回滚时使用它来正确清理。

## DDL_log

为了实现原子DDL的提交和回滚，InnoDB存储引擎引入了一个表DDL_LOG，这是一个受保护的表，不允许外部用户查询和修改，包括对该表进行DDL以及DML。该表用来存储DDL执行期间InnoDB存储引擎需要对物理文件以及相关系统表操作的记录，对于添加到DDL_LOG的每一条记录，都会附加一个trx_id（事务id），因此在提交/回滚时，可以用事务标识这些条目，并采取适当的操作。在InnoDB提交/回滚和相应的操作之后，事务的所有记录将从DDL_LOG中删除。为了保证SERVER crash的时候DDL还能支持原子性，这个表必须尽快持久化，它需要进行同步刷新，不受`innodb_flush_log_at_trx_commit`的控制。

DDL Log Table的定义如下：

`CREATE TABLE mysql.innodb_ddl_log (
 id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY, // DDL log记录的唯一标志符
 thread_id BIGINT UNSIGNED NOT NULL, // 每个DDL日志记录分配一个thread_id，用于重放和删除属于特定DDL操作的DDL日志
 type INT UNSIGNED NOT NULL, // DDL操作的类型，包括FREE RENAME等
 space_id INT UNSIGNED, // 表空间的id
 page_no INT UNSIGNED, // 包含分配信息的页，比如索引树的root
 index_id BIGINT UNSIGNED, // 索引id
 table_id BIGINT UNSIGNED, // 表id
 old_file_path VARCHAR(512) COLLATE UTF8_BIN, // 旧的表空间文件路径，用于创建或删除表空间文件的DDL操作，也用于重命名表空间的DDL操作
 new_file_path VARCHAR(512) COLLATE UTF8_BIN, // 新的表空间文件路径，用于重命名表空间文件的DDL操作。
 KEY(thread_id)
);
`

DDL语句的执行分为以下几个阶段，有时候prepare和perform阶段可以在commit之前反复执行：

1. prepare：创建所需的对象并把DDL log写入 `mysql.innodb_ddl_log` ；
2. perform：执行DDL操作；
3. commit：更新数据字典并提交数据字典事务；
4. Post-DDL：重放或从`mysql.innodb_ddl_log` 中删除DDL log。

DDL操作类型如下：

`enum class Log_Type : uint32_t {
 /** Smallest log type */
 SMALLEST_LOG = 1,
 /** Drop an index tree */
 FREE_TREE_LOG = 1,
 /** Delete a file */
 DELETE_SPACE_LOG,
 /** Rename a file */
 RENAME_SPACE_LOG,
 /** Drop the entry in innodb_table_metadata */
 DROP_LOG,
 /** Rename table in dict cache. */
 RENAME_TABLE_LOG,
 /** Remove a table from dict cache */
 REMOVE_CACHE_LOG,
 /** Alter Encrypt a tablespace */
 ALTER_ENCRYPT_TABLESPACE_LOG,
 /** Biggest log type */
 BIGGEST_LOG = ALTER_ENCRYPT_TABLESPACE_LOG
};
`

1. FREE_TREE_LOG

 删除指定的索引。
2. DELETE_SPACE_LOG

 删除指定的idb表空间文件。
3. RENAME_SPACE_LOG

 删除指定的idb表空间文件。
4. DROP_LOG

 从 `mysql.innodb_dynamic_metadata` 表中删除指定表的信息。
5. RENAME_TABLE_LOG

 重命名dictionary cache中的表。
6. REMOVE_CACHE_LOG

 删除dictionary cache中指定的表。
7. ALTER_ENCRYPT_TABLESPACE_LOG

 用于记录对tablespace加密属性的修改。

DDL Log可以看作是Redo Log和Undo Log的一个合集。有些DDL把它用作Redo，有些DDL把它用做Undo，还有些DDL会把它同时当作Redo和Undo。有些DDL log是随着父事务一起提交的，有些则在Post-DDL阶段再执行，Post-DDL发生在父事提交或回滚之后，若事务回滚，根据DDL log做逆操作，若事务提交，在Post-DDL阶段做最后真正不可逆操作，在之后的小节会针对典型命令的操作过程进行分析。

## CREATE TABLE的执行

执行一条最简单的CREATE TABLE，来分析整个的代码执行逻辑。

`mysql> create table t1(a int);
`

创建表的执行过程如下：

1. 在SQL层，创建表对象（Table Object），然后对SE进行初始函数调用，以便SE能够初始化它对DDL的处理；
2. SE添加它此时拥有的SE私有数据，并将控制权返回给SQL层；
3. 然后将表存储在DD表中。对于支持原子DDL的存储引擎来说，此时还没有提交；
4. SQL层构建所有的内部结构，然后调用SE层的建表函数；
5. SE创建表空间/表/索引树，在DDL_LOG中记录上述物理文件和创建的索引，更新SE私有数据，并将控制权返回给SQL层。所有关于新表空间/索引的信息都通过DD对象传递给server层；
6. SQL层写入二进制日志，并根据执行状态提交或回滚事务；
7. SQL层在SE中调用一个post_ddl()的钩子函数对文件和树进行适当的清理，并删除事务的DDL_LOG中的条目。如果事务回滚，则post_ddl()会删除表空间和索引树。

详细的调用流程为：

`mysql_create_table
 --> mysql_create_table_no_lock
 --> create_table_impl
 --> rea_create_base_table
 --> dd::create_table //创建dd::Table
 | --> dd::create_dd_user_table / dd::create_dd_system_table 
 | // 根据create_info填充dd::Table
 --> dd::cache::Dictionary_client::store<dd::Table> 
 | // 判断dd:Table是否已经存入数据字典，如果没有才进入这个函数
 | --> dd::cache::Storage_adapter::store<dd::Table> // 将创建好的dd:Table存入dd表
 | --> dd::Weak_object_impl::store
 | --> dd::Table_impl::store_attributes // 更新mysql.tables
 | --> dd::Table_impl::store_children 
 | // 更新建表相关的数据字典表如indexes，foreign_keys，partitions
 --> ha_create_table // 实际创建表
 --> handler::ha_create
 | --> ha_innobase::create // 创建InnoDB表
 | --> innobase_basic_ddl::create_impl<dd::Table>
 | --> create_table_info_t::create_table
 | | --> create_table_info_t::create_table_def 
 | | | // 创建基于InnoDB数据库的表定义，检查表名是否合规
 | | | // 确定列数之后，在内存中创建了空的表
 | | | --> dict_mem_table_create 
 | | | // 在内存中创建表对象（空的，只申请了空间），设置了表的一些参数
 | | | // 然后对这张表进行了基础的填充，包括设定一些名称、为表加列
 | | | // 此时还没有对应的idb文件生成
 | | | --> row_create_table_for_mysql
 | | | --> dict_build_table_def // 在不更新系统表的情况下创建表定义definition
 | | | | --> dict_build_tablespace_for_table 
 | | | | // 创建表空间，由table->name确定ibd文件的路径
 | | | | // 之后写入ddl log文件，再创建ibd文件
 | | | | --> Log_DDL::write_delete_space_log 
 | | | | | // 调用Log_DDL::insert_delete_space_log写入ddl log
 | | | | --> fil_ibd_create // 这个函数执行完之后才真正创建了ibd文件
 | | | --> dict_table_add_system_columns // 给表加入系统列（system columns）
 | | | --> dict_table_add_to_cache // 将要创建的表加入dictionary cache
 | | | --> Log_DDL::write_remove_cache_log
 | | --> create_clustered_index_when_no_primary // 添加主键索引
 | | | --> dict_mem_index_create
 | | | // 在内存中申请了index的空间，设置了type、table_name等的参数
 | | | --> row_create_index_for_mysql
 | | | --> dict_build_index_def
 | | | // 创建index定义，不更新系统表，
 | | | // 更新了index_id，index->space和index->trx_id
 | | | --> dict_index_add_to_cache_w_vcol // 将index写入dictionary cache
 | | | --> dict_create_index_tree_in_mem
 | | | --> btr_create // 创建index树，返回root页
 | | | --> Log_DDL::write_free_tree_log
 | | | // 这里是先创建了索引然后再写入的ddl log，所以如果这时crash，
 | | | // （对其他的操作来说）索引还在，就没有办法找到索引对应的资源了，
 | | | // 但是因为这种情况很少见，所以可以接受。
 | | | // 不过对create table来说，如果file_per_table为true
 | | | // crash回滚的时候会删除整个表空间的。
 | | --> create_index // 有定义索引的话，会继续创建二级索引，本例没有就暂时不看了
 | --> create_table_info_t::create_table_update_global_dd<dd::Table> 
 | // 更新全局data dictionary，创建tablespace表
 | --> create_table_info_t::create_table_update_dict // 更新InnoDB数据库中的表
 | --> innobase_copy_frm_flags_from_create_info
 | // 有些flag位存在.frm文件里，拷贝他们过来
 | --> dict_stats_update // 更新一些表和索引的统计信息用于优化
 | --> innobase_parse_hint_from_comment
 | // 统计表和索引之间的联系，在dictionary中更新
 --> Dictionary_client::update // 更新持久化了的dictionary对象，但是共享缓存里的内容不变
write_bin_log // 写入binlog文件
Log_DDL::post_ddl // 对文件和树进行适当的清理，删除DDL_LOG中的记录。
 // 如果事务回滚，则post_ddl()物理删除表空间/ibd (file-per-table)并删除表的索引树。
`

## 典型命令的操作过程

MySQL提供了一个选项 `innodb_print_ddl_logs` ，通过设置该参数可以让MySQL将DDL logs写入stderr，从而可以从错误日志中看到一些典型命令的操作过程。

`log_error_verbosity` 是 `log_warnings` 的替代，当它等于3时表示各种信息都会写入错误日志，包括ERROR，WARNING和INFORMATION。

`mysql> SET GLOBAL innodb_print_ddl_logs = 1;
Query OK, 0 rows affected (0.00 sec)

mysql> SET GLOBAL log_error_verbosity = 3;
Query OK, 0 rows affected (0.00 sec)
`

### CREATE DATABASE

```
mysql> create database my_test;
Query OK, 1 row affected (0.00 sec)

```

创建数据库没有DDL log记录，所以如果创建数据库时中途失败，之后可能需要手动清除数据。

### CREATE TABLE

#### no index

`mysql> create table t1(a int, b int) partition by hash(a) partitions 2;
Query OK, 0 rows affected (0.59 sec)

[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=2, thread_id=12, space_id=2, old_file_path=./my_test/t1#P#p0.ibd]
[InnoDB] DDL log delete : 2
[InnoDB] DDL log insert : [DDL record: REMOVE CACHE, id=3, thread_id=12, table_id=1063, new_file_path=my_test/t1#P#p0]
[InnoDB] DDL log delete : 3
[InnoDB] DDL log insert : [DDL record: FREE, id=4, thread_id=12, space_id=2, index_id=149, page_no=4]
[InnoDB] DDL log delete : 4
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=5, thread_id=12, space_id=3, old_file_path=./my_test/t1#P#p1.ibd]
[InnoDB] DDL log delete : 5
[InnoDB] DDL log insert : [DDL record: REMOVE CACHE, id=6, thread_id=12, table_id=1064, new_file_path=my_test/t1#P#p1]
[InnoDB] DDL log delete : 6
[InnoDB] DDL log insert : [DDL record: FREE, id=7, thread_id=12, space_id=3, index_id=150, page_no=4]
[InnoDB] DDL log delete : 7
[InnoDB] DDL log post ddl : begin for thread id : 12
[InnoDB] DDL log post ddl : end for thread id : 12
`

所有插入的记录都是单独的事务，已经进行操作的反向操作。对于创建table space来说，它的反向操作就是DELETE_SPACE_LOG。

所得到的ddl log还含有 `DDL log delete` 操作，它其实也是记录，用来删除ddl log。如果最后DDL事务成功提交，delete操作最后就会起到作用，DDL log被清空，但如果DDL事务中途失败了，delete操作会回滚，insert的记录得到保留，这些ddl log会清理遗留的垃圾文件。

对建表逻辑来说，它包含三类：DELETE SPACE、REMOVE CACHE和FREE。因为建表时对其进行了分区，所以上述三条命令是呈分区倍数出现的。首先建立了第一个分区表，将其写入dictionary cache，再建立索引，然后再对后续的分区表进行同样的操作。ddl log记录的便是这些操作的逆向逻辑：删除数据文件，释放内存中的数据字典信息，删除索引btree。当事务最终提交，ddl log会将这些记录删除。在这里DDL log起到的就是Undo。

#### with index

`mysql> create table t2(a int, b int, key index_a(a)) partition by hash(a) partitions 2;
Query OK, 0 rows affected (0.60 sec)

[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=8, thread_id=12, space_id=4, old_file_path=./my_test/t2#P#p0.ibd]
[InnoDB] DDL log delete : 8
[InnoDB] DDL log insert : [DDL record: REMOVE CACHE, id=9, thread_id=12, table_id=1066, new_file_path=my_test/t2#P#p0]
[InnoDB] DDL log delete : 9
[InnoDB] DDL log insert : [DDL record: FREE, id=10, thread_id=12, space_id=4, index_id=151, page_no=4]
[InnoDB] DDL log delete : 10
[InnoDB] DDL log insert : [DDL record: FREE, id=11, thread_id=12, space_id=4, index_id=152, page_no=5]
[InnoDB] DDL log delete : 11
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=12, thread_id=12, space_id=5, old_file_path=./my_test/t2#P#p1.ibd]
[InnoDB] DDL log delete : 12
[InnoDB] DDL log insert : [DDL record: REMOVE CACHE, id=13, thread_id=12, table_id=1067, new_file_path=my_test/t2#P#p1]
[InnoDB] DDL log delete : 13
[InnoDB] DDL log insert : [DDL record: FREE, id=14, thread_id=12, space_id=5, index_id=153, page_no=4]
[InnoDB] DDL log delete : 14
[InnoDB] DDL log insert : [DDL record: FREE, id=15, thread_id=12, space_id=5, index_id=154, page_no=5]
[InnoDB] DDL log delete : 15
[InnoDB] DDL log post ddl : begin for thread id : 12
[InnoDB] DDL log post ddl : end for thread id : 12
`

相比于不含key的建表逻辑，可以看到这次的ddl log里多了两条FREE，应该就是对每一个分区建立索引的操作。

### ADD COLUMN

`mysql> alter table t1 add column c int;
Query OK, 0 rows affected (0.38 sec)
Records: 0 Duplicates: 0 Warnings: 0

[InnoDB] DDL log post ddl : begin for thread id : 12
[InnoDB] DDL log post ddl : end for thread id : 12
`

没有涉及对物理文件的改动，不需要ddl log来保证原子性，因此也没有ddl log记录。

### ADD KEY

`mysql> alter table t1 add key loc_a(a);
Query OK, 0 rows affected (0.12 sec)
Records: 0 Duplicates: 0 Warnings: 0

[InnoDB] DDL log insert : [DDL record: FREE, id=16, thread_id=12, space_id=2, index_id=155, page_no=5]
[InnoDB] DDL log delete : 16
[InnoDB] DDL log insert : [DDL record: FREE, id=17, thread_id=12, space_id=3, index_id=156, page_no=5]
[InnoDB] DDL log delete : 17
[InnoDB] DDL log post ddl : begin for thread id : 12
[InnoDB] DDL log post ddl : end for thread id : 12
`

创建索引采用inplace创建的方式，没有临时文件，但如果异常发生的话，依然需要在发生异常时清理临时索引。ADD KEY需要对每一个分区都建立新的索引，这里有两个分区，所以有两条FREE记录。

### DROP KEY

`mysql> alter table t2 add key index_b(b);
Query OK, 0 rows affected (4.50 sec)
Records: 0 Duplicates: 0 Warnings: 0
mysql> alter table t2 drop key index_b;
Query OK, 0 rows affected (0.16 sec)
Records: 0 Duplicates: 0 Warnings: 0

// 关注第二句
[InnoDB] DDL log insert : [DDL record: FREE, id=20, thread_id=12, space_id=4, index_id=157, page_no=6]
[InnoDB] DDL log insert : [DDL record: FREE, id=21, thread_id=12, space_id=5, index_id=158, page_no=6]
[InnoDB] DDL log post ddl : begin for thread id : 12
[InnoDB] DDL log replay : [DDL record: FREE, id=21, thread_id=12, space_id=5, index_id=158, page_no=6]
[InnoDB] DDL log replay : [DDL record: FREE, id=20, thread_id=12, space_id=4, index_id=157, page_no=6]
[InnoDB] DDL log post ddl : end for thread id : 12
`

DROP KEY的逻辑和前面几条命令的逻辑都不同，在执行阶段它只记录了ddl logs，记下需要删除的索引树，但并没有执行真正的删除，这也是因为如果删了之后发生crash，恢复起来会比较麻烦，它真正的删除操作是在post ddl阶段进行的。这里的DDL log就相当于Redo。

### DROP COLUMN

`mysql> alter table t1 drop column c;
Query OK, 0 rows affected (0.69 sec)
Records: 0 Duplicates: 0 Warnings: 0

[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=22, thread_id=12, space_id=6, old_file_path=./my_test/#sql-ib1063-4028979805.ibd]
[InnoDB] DDL log delete : 22
[InnoDB] DDL log insert : [DDL record: REMOVE CACHE, id=23, thread_id=12, table_id=1069, new_file_path=my_test/#sql-ib1063-4028979805]
[InnoDB] DDL log delete : 23
[InnoDB] DDL log insert : [DDL record: FREE, id=24, thread_id=12, space_id=6, index_id=159, page_no=4]
[InnoDB] DDL log delete : 24
[InnoDB] DDL log insert : [DDL record: FREE, id=25, thread_id=12, space_id=6, index_id=160, page_no=5]
[InnoDB] DDL log delete : 25
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=26, thread_id=12, space_id=7, old_file_path=./my_test/#sql-ib1064-4028979806.ibd]
[InnoDB] DDL log delete : 26
[InnoDB] DDL log insert : [DDL record: REMOVE CACHE, id=27, thread_id=12, table_id=1070, new_file_path=my_test/#sql-ib1064-4028979806]
[InnoDB] DDL log delete : 27
[InnoDB] DDL log insert : [DDL record: FREE, id=28, thread_id=12, space_id=7, index_id=161, page_no=4]
[InnoDB] DDL log delete : 28
[InnoDB] DDL log insert : [DDL record: FREE, id=29, thread_id=12, space_id=7, index_id=162, page_no=5]
[InnoDB] DDL log delete : 29
[InnoDB] DDL log insert : [DDL record: DROP, id=30, thread_id=12, table_id=1063]
[InnoDB] DDL log insert : [DDL record: DROP, id=31, thread_id=12, table_id=1064]
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=32, thread_id=12, space_id=2, old_file_path=./my_test/#sql-ib1069-4028979807.ibd, new_file_path=./my_test/t1#P#p0.ibd]
[InnoDB] DDL log delete : 32
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=33, thread_id=12, table_id=1063, old_file_path=my_test/#sql-ib1069-4028979807, new_file_path=my_test/t1#P#p0]
[InnoDB] DDL log delete : 33
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=34, thread_id=12, space_id=6, old_file_path=./my_test/t1#P#p0.ibd, new_file_path=./my_test/#sql-ib1063-4028979805.ibd]
[InnoDB] DDL log delete : 34
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=35, thread_id=12, table_id=1069, old_file_path=my_test/t1#P#p0, new_file_path=my_test/#sql-ib1063-4028979805]
[InnoDB] DDL log delete : 35
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=36, thread_id=12, space_id=3, old_file_path=./my_test/#sql-ib1070-4028979808.ibd, new_file_path=./my_test/t1#P#p1.ibd]
[InnoDB] DDL log delete : 36
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=37, thread_id=12, table_id=1064, old_file_path=my_test/#sql-ib1070-4028979808, new_file_path=my_test/t1#P#p1]
[InnoDB] DDL log delete : 37
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=38, thread_id=12, space_id=7, old_file_path=./my_test/t1#P#p1.ibd, new_file_path=./my_test/#sql-ib1064-4028979806.ibd]
[InnoDB] DDL log delete : 38
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=39, thread_id=12, table_id=1070, old_file_path=my_test/t1#P#p1, new_file_path=my_test/#sql-ib1064-4028979806]
[InnoDB] DDL log delete : 39
[InnoDB] DDL log insert : [DDL record: DROP, id=40, thread_id=12, table_id=1063]
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=41, thread_id=12, space_id=2, old_file_path=./my_test/#sql-ib1069-4028979807.ibd]
[InnoDB] DDL log insert : [DDL record: DROP, id=42, thread_id=12, table_id=1064]
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=43, thread_id=12, space_id=3, old_file_path=./my_test/#sql-ib1070-4028979808.ibd]
[InnoDB] DDL log post ddl : begin for thread id : 12
[InnoDB] DDL log replay : [DDL record: DELETE SPACE, id=43, thread_id=12, space_id=3, old_file_path=./my_test/#sql-ib1070-4028979808.ibd]
[InnoDB] DDL log replay : [DDL record: DROP, id=42, thread_id=12, table_id=1064]
[InnoDB] DDL log replay : [DDL record: DELETE SPACE, id=41, thread_id=12, space_id=2, old_file_path=./my_test/#sql-ib1069-4028979807.ibd]
[InnoDB] DDL log replay : [DDL record: DROP, id=40, thread_id=12, table_id=1063]
[InnoDB] DDL log replay : [DDL record: DROP, id=31, thread_id=12, table_id=1064]
[InnoDB] DDL log replay : [DDL record: DROP, id=30, thread_id=12, table_id=1063]
[InnoDB] DDL log post ddl : end for thread id : 12
`

alter table有很多种，这里是最复杂的重建表的逻辑。这种情况下DDL log既是redo，也是undo。

执行阶段首先是建立了两个分区表，一开始走了create table的逻辑，然后记录下要删除的原来的表（此时只是记录，留作post-ddl阶段再执行），之后是一系列重命名操作，把旧的表空间和旧的表重命名为新的，这里记录的也是实际执行过程的逆操作。之前的表空间和表名（以A代称）先被重命名成另外一个中间名（以C代称），然后把最初创建的新的表空间和表名（代称为B）重命名为正确的表名，也就是最开始的A名。而被代替的旧表和旧表空间C，先记录下来ddl log，等到post-ddl阶段再做删除。

### RENAME INDEX

`mysql> alter table t1 rename index loc_a to loc_aa;
Query OK, 0 rows affected (0.12 sec)
Records: 0 Duplicates: 0 Warnings: 0

[InnoDB] DDL log post ddl : begin for thread id : 12
[InnoDB] DDL log post ddl : end for thread id : 12
`

没有涉及对物理文件的改动，不需要ddl log来保证原子性，因此也没有ddl log记录。

### RENAME COLUMN

`mysql> alter table t1 add key loc_b(b);
Query OK, 0 rows affected (4.24 sec)
Records: 0 Duplicates: 0 Warnings: 0
mysql> alter table t1 rename column b to bb;
Query OK, 0 rows affected (0.12 sec)
Records: 0 Duplicates: 0 Warnings: 0

// 关注第二句
[InnoDB] DDL log post ddl : begin for thread id : 12
[InnoDB] DDL log post ddl : end for thread id : 12
`

没有涉及对物理文件的改动，不需要ddl log来保证原子性，因此也没有ddl log记录。

### RENAME TABLE

`mysql> rename table t1 to t11;
Query OK, 0 rows affected (0.11 sec)

[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=8, thread_id=12, space_id=2, old_file_path=./my_test/t11#P#p0.ibd, new_file_path=./my_test/t1#P#p0.ibd]
[InnoDB] DDL log delete : 8
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=9, thread_id=12, table_id=1063, old_file_path=my_test/t11#P#p0, new_file_path=my_test/t1#P#p0]
[InnoDB] DDL log delete : 9
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=10, thread_id=12, space_id=3, old_file_path=./my_test/t11#P#p1.ibd, new_file_path=./my_test/t1#P#p1.ibd]
[InnoDB] DDL log delete : 10
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=11, thread_id=12, table_id=1064, old_file_path=my_test/t11#P#p1, new_file_path=my_test/t1#P#p1]
[InnoDB] DDL log delete : 11
[InnoDB] DDL log post ddl : begin for thread id : 12
[InnoDB] DDL log post ddl : end for thread id : 12
`

对每个分区的表空间和表进行了rename操作。

### REBUILD

`mysql> alter table t1 engine=InnoDB;
Query OK, 0 rows affected (0.72 sec)
Records: 0 Duplicates: 0 Warnings: 0

[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=46, thread_id=12, space_id=8, old_file_path=./my_test/#sql-ib1069-4028979809.ibd]
[InnoDB] DDL log delete : 46
[InnoDB] DDL log insert : [DDL record: REMOVE CACHE, id=47, thread_id=12, table_id=1071, new_file_path=my_test/#sql-ib1069-4028979809]
[InnoDB] DDL log delete : 47
[InnoDB] DDL log insert : [DDL record: FREE, id=48, thread_id=12, space_id=8, index_id=165, page_no=4]
[InnoDB] DDL log delete : 48
[InnoDB] DDL log insert : [DDL record: FREE, id=49, thread_id=12, space_id=8, index_id=166, page_no=5]
[InnoDB] DDL log delete : 49
[InnoDB] DDL log insert : [DDL record: FREE, id=50, thread_id=12, space_id=8, index_id=167, page_no=6]
[InnoDB] DDL log delete : 50
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=51, thread_id=12, space_id=9, old_file_path=./my_test/#sql-ib1070-4028979810.ibd]
[InnoDB] DDL log delete : 51
[InnoDB] DDL log insert : [DDL record: REMOVE CACHE, id=52, thread_id=12, table_id=1072, new_file_path=my_test/#sql-ib1070-4028979810]
[InnoDB] DDL log delete : 52
[InnoDB] DDL log insert : [DDL record: FREE, id=53, thread_id=12, space_id=9, index_id=168, page_no=4]
[InnoDB] DDL log delete : 53
[InnoDB] DDL log insert : [DDL record: FREE, id=54, thread_id=12, space_id=9, index_id=169, page_no=5]
[InnoDB] DDL log delete : 54
[InnoDB] DDL log insert : [DDL record: FREE, id=55, thread_id=12, space_id=9, index_id=170, page_no=6]
[InnoDB] DDL log delete : 55
[InnoDB] DDL log insert : [DDL record: DROP, id=56, thread_id=12, table_id=1069]
[InnoDB] DDL log insert : [DDL record: DROP, id=57, thread_id=12, table_id=1070]
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=58, thread_id=12, space_id=6, old_file_path=./my_test/#sql-ib1071-4028979811.ibd, new_file_path=./my_test/t1#P#p0.ibd]
[InnoDB] DDL log delete : 58
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=59, thread_id=12, table_id=1069, old_file_path=my_test/#sql-ib1071-4028979811, new_file_path=my_test/t1#P#p0]
[InnoDB] DDL log delete : 59
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=60, thread_id=12, space_id=8, old_file_path=./my_test/t1#P#p0.ibd, new_file_path=./my_test/#sql-ib1069-4028979809.ibd]
[InnoDB] DDL log delete : 60
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=61, thread_id=12, table_id=1071, old_file_path=my_test/t1#P#p0, new_file_path=my_test/#sql-ib1069-4028979809]
[InnoDB] DDL log delete : 61
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=62, thread_id=12, space_id=7, old_file_path=./my_test/#sql-ib1072-4028979812.ibd, new_file_path=./my_test/t1#P#p1.ibd]
[InnoDB] DDL log delete : 62
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=63, thread_id=12, table_id=1070, old_file_path=my_test/#sql-ib1072-4028979812, new_file_path=my_test/t1#P#p1]
[InnoDB] DDL log delete : 63
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=64, thread_id=12, space_id=9, old_file_path=./my_test/t1#P#p1.ibd, new_file_path=./my_test/#sql-ib1070-4028979810.ibd]
[InnoDB] DDL log delete : 64
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=65, thread_id=12, table_id=1072, old_file_path=my_test/t1#P#p1, new_file_path=my_test/#sql-ib1070-4028979810]
[InnoDB] DDL log delete : 65
[InnoDB] DDL log insert : [DDL record: DROP, id=66, thread_id=12, table_id=1069]
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=67, thread_id=12, space_id=6, old_file_path=./my_test/#sql-ib1071-4028979811.ibd]
[InnoDB] DDL log insert : [DDL record: DROP, id=68, thread_id=12, table_id=1070]
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=69, thread_id=12, space_id=7, old_file_path=./my_test/#sql-ib1072-4028979812.ibd]
[InnoDB] DDL log post ddl : begin for thread id : 12
[InnoDB] DDL log replay : [DDL record: DELETE SPACE, id=69, thread_id=12, space_id=7, old_file_path=./my_test/#sql-ib1072-4028979812.ibd]
[InnoDB] DDL log replay : [DDL record: DROP, id=68, thread_id=12, table_id=1070]
[InnoDB] DDL log replay : [DDL record: DELETE SPACE, id=67, thread_id=12, space_id=6, old_file_path=./my_test/#sql-ib1071-4028979811.ibd]
[InnoDB] DDL log replay : [DDL record: DROP, id=66, thread_id=12, table_id=1069]
[InnoDB] DDL log replay : [DDL record: DROP, id=57, thread_id=12, table_id=1070]
[InnoDB] DDL log replay : [DDL record: DROP, id=56, thread_id=12, table_id=1069]
[InnoDB] DDL log post ddl : end for thread id : 12
`

rebuild的逻辑和alter table … add column一样，都是重建表，二者的ddl log也极其相似，在这里就不再重复rebuild的实现逻辑了。

### CHANGE COLUMN

`mysql> alter table t1 change column bb b char(10);
Query OK, 0 rows affected (5.28 sec)
Records: 0 Duplicates: 0 Warnings: 0

[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=70, thread_id=12, space_id=10, old_file_path=./my_test/#sql-e525_c#P#p0.ibd]
[InnoDB] DDL log delete : 70
[InnoDB] DDL log insert : [DDL record: REMOVE CACHE, id=71, thread_id=12, table_id=1073, new_file_path=my_test/#sql-e525_c#P#p0]
[InnoDB] DDL log delete : 71
[InnoDB] DDL log insert : [DDL record: FREE, id=72, thread_id=12, space_id=10, index_id=171, page_no=4]
[InnoDB] DDL log delete : 72
[InnoDB] DDL log insert : [DDL record: FREE, id=73, thread_id=12, space_id=10, index_id=172, page_no=5]
[InnoDB] DDL log delete : 73
[InnoDB] DDL log insert : [DDL record: FREE, id=74, thread_id=12, space_id=10, index_id=173, page_no=6]
[InnoDB] DDL log delete : 74
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=75, thread_id=12, space_id=11, old_file_path=./my_test/#sql-e525_c#P#p1.ibd]
[InnoDB] DDL log delete : 75
[InnoDB] DDL log insert : [DDL record: REMOVE CACHE, id=76, thread_id=12, table_id=1074, new_file_path=my_test/#sql-e525_c#P#p1]
[InnoDB] DDL log delete : 76
[InnoDB] DDL log insert : [DDL record: FREE, id=77, thread_id=12, space_id=11, index_id=174, page_no=4]
[InnoDB] DDL log delete : 77
[InnoDB] DDL log insert : [DDL record: FREE, id=78, thread_id=12, space_id=11, index_id=175, page_no=5]
[InnoDB] DDL log delete : 78
[InnoDB] DDL log insert : [DDL record: FREE, id=79, thread_id=12, space_id=11, index_id=176, page_no=6]
[InnoDB] DDL log delete : 79
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=80, thread_id=12, space_id=8, old_file_path=./my_test/#sql2-e525-c#P#p0.ibd, new_file_path=./my_test/t1#P#p0.ibd]
[InnoDB] DDL log delete : 80
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=81, thread_id=12, table_id=1071, old_file_path=my_test/#sql2-e525-c#P#p0, new_file_path=my_test/t1#P#p0]
[InnoDB] DDL log delete : 81
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=82, thread_id=12, space_id=9, old_file_path=./my_test/#sql2-e525-c#P#p1.ibd, new_file_path=./my_test/t1#P#p1.ibd]
[InnoDB] DDL log delete : 82
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=83, thread_id=12, table_id=1072, old_file_path=my_test/#sql2-e525-c#P#p1, new_file_path=my_test/t1#P#p1]
[InnoDB] DDL log delete : 83
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=84, thread_id=12, space_id=10, old_file_path=./my_test/t1#P#p0.ibd, new_file_path=./my_test/#sql-e525_c#P#p0.ibd]
[InnoDB] DDL log delete : 84
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=85, thread_id=12, table_id=1073, old_file_path=my_test/t1#P#p0, new_file_path=my_test/#sql-e525_c#P#p0]
[InnoDB] DDL log delete : 85
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=86, thread_id=12, space_id=11, old_file_path=./my_test/t1#P#p1.ibd, new_file_path=./my_test/#sql-e525_c#P#p1.ibd]
[InnoDB] DDL log delete : 86
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=87, thread_id=12, table_id=1074, old_file_path=my_test/t1#P#p1, new_file_path=my_test/#sql-e525_c#P#p1]
[InnoDB] DDL log delete : 87
[InnoDB] DDL log insert : [DDL record: DROP, id=88, thread_id=12, table_id=1071]
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=89, thread_id=12, space_id=8, old_file_path=./my_test/#sql2-e525-c#P#p0.ibd]
[InnoDB] DDL log insert : [DDL record: DROP, id=90, thread_id=12, table_id=1072]
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=91, thread_id=12, space_id=9, old_file_path=./my_test/#sql2-e525-c#P#p1.ibd]
[InnoDB] DDL log post ddl : begin for thread id : 12
[InnoDB] DDL log replay : [DDL record: DELETE SPACE, id=91, thread_id=12, space_id=9, old_file_path=./my_test/#sql2-e525-c#P#p1.ibd]
[InnoDB] DDL log replay : [DDL record: DROP, id=90, thread_id=12, table_id=1072]
[InnoDB] DDL log replay : [DDL record: DELETE SPACE, id=89, thread_id=12, space_id=8, old_file_path=./my_test/#sql2-e525-c#P#p0.ibd]
[InnoDB] DDL log replay : [DDL record: DROP, id=88, thread_id=12, table_id=1071]
[InnoDB] DDL log post ddl : end for thread id : 12
[InnoDB] DDL log post ddl : begin for thread id : 12
[InnoDB] DDL log post ddl : end for thread id : 12
`

和rebuild和alter table … add column也很相似，首先也是走了建表的逻辑，创建了name前缀为 `#sql-e525_c` 的表和表空间，之前的一通操作之后这里有三个key（包含默认的primary），所以有三个FREE的逻辑。之后就是重命名的逻辑，借助一个中间名 `#sql2-e525-c` （注意最后下划线不一样），把新创建的表和表空间和之前的进行交换，原来的表和表空间重命名为`#sql2-e525-c` 开头的文件，新生成的`#sql-e525_c` 开头的表和表空间重命名为正确的名字（ `t1#P#p0` 等），最后记录删除旧表和表空间的log，也就是现在开头为`#sql2-e525-c`的表空间和表，在post-ddl阶段执行。

### MODIFY COLUMN

`mysql> alter table t1 modify column b int;
Query OK, 0 rows affected (0.89 sec)
Records: 0 Duplicates: 0 Warnings: 0

[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=92, thread_id=12, space_id=12, old_file_path=./my_test/#sql-e525_c#P#p0.ibd]
[InnoDB] DDL log delete : 92
[InnoDB] DDL log insert : [DDL record: REMOVE CACHE, id=93, thread_id=12, table_id=1076, new_file_path=my_test/#sql-e525_c#P#p0]
[InnoDB] DDL log delete : 93
[InnoDB] DDL log insert : [DDL record: FREE, id=94, thread_id=12, space_id=12, index_id=177, page_no=4]
[InnoDB] DDL log delete : 94
[InnoDB] DDL log insert : [DDL record: FREE, id=95, thread_id=12, space_id=12, index_id=178, page_no=5]
[InnoDB] DDL log delete : 95
[InnoDB] DDL log insert : [DDL record: FREE, id=96, thread_id=12, space_id=12, index_id=179, page_no=6]
[InnoDB] DDL log delete : 96
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=97, thread_id=12, space_id=13, old_file_path=./my_test/#sql-e525_c#P#p1.ibd]
[InnoDB] DDL log delete : 97
[InnoDB] DDL log insert : [DDL record: REMOVE CACHE, id=98, thread_id=12, table_id=1077, new_file_path=my_test/#sql-e525_c#P#p1]
[InnoDB] DDL log delete : 98
[InnoDB] DDL log insert : [DDL record: FREE, id=99, thread_id=12, space_id=13, index_id=180, page_no=4]
[InnoDB] DDL log delete : 99
[InnoDB] DDL log insert : [DDL record: FREE, id=100, thread_id=12, space_id=13, index_id=181, page_no=5]
[InnoDB] DDL log delete : 100
[InnoDB] DDL log insert : [DDL record: FREE, id=101, thread_id=12, space_id=13, index_id=182, page_no=6]
[InnoDB] DDL log delete : 101
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=102, thread_id=12, space_id=10, old_file_path=./my_test/#sql2-e525-c#P#p0.ibd, new_file_path=./my_test/t1#P#p0.ibd]
[InnoDB] DDL log delete : 102
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=103, thread_id=12, table_id=1073, old_file_path=my_test/#sql2-e525-c#P#p0, new_file_path=my_test/t1#P#p0]
[InnoDB] DDL log delete : 103
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=104, thread_id=12, space_id=11, old_file_path=./my_test/#sql2-e525-c#P#p1.ibd, new_file_path=./my_test/t1#P#p1.ibd]
[InnoDB] DDL log delete : 104
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=105, thread_id=12, table_id=1074, old_file_path=my_test/#sql2-e525-c#P#p1, new_file_path=my_test/t1#P#p1]
[InnoDB] DDL log delete : 105
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=106, thread_id=12, space_id=12, old_file_path=./my_test/t1#P#p0.ibd, new_file_path=./my_test/#sql-e525_c#P#p0.ibd]
[InnoDB] DDL log delete : 106
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=107, thread_id=12, table_id=1076, old_file_path=my_test/t1#P#p0, new_file_path=my_test/#sql-e525_c#P#p0]
[InnoDB] DDL log delete : 107
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=108, thread_id=12, space_id=13, old_file_path=./my_test/t1#P#p1.ibd, new_file_path=./my_test/#sql-e525_c#P#p1.ibd]
[InnoDB] DDL log delete : 108
[InnoDB] DDL log insert : [DDL record: RENAME TABLE, id=109, thread_id=12, table_id=1077, old_file_path=my_test/t1#P#p1, new_file_path=my_test/#sql-e525_c#P#p1]
[InnoDB] DDL log delete : 109
[InnoDB] DDL log insert : [DDL record: DROP, id=110, thread_id=12, table_id=1073]
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=111, thread_id=12, space_id=10, old_file_path=./my_test/#sql2-e525-c#P#p0.ibd]
[InnoDB] DDL log insert : [DDL record: DROP, id=112, thread_id=12, table_id=1074]
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=113, thread_id=12, space_id=11, old_file_path=./my_test/#sql2-e525-c#P#p1.ibd]
[InnoDB] DDL log post ddl : begin for thread id : 12
[InnoDB] DDL log replay : [DDL record: DELETE SPACE, id=113, thread_id=12, space_id=11, old_file_path=./my_test/#sql2-e525-c#P#p1.ibd]
[InnoDB] DDL log replay : [DDL record: DROP, id=112, thread_id=12, table_id=1074]
[InnoDB] DDL log replay : [DDL record: DELETE SPACE, id=111, thread_id=12, space_id=10, old_file_path=./my_test/#sql2-e525-c#P#p0.ibd]
[InnoDB] DDL log replay : [DDL record: DROP, id=110, thread_id=12, table_id=1073]
[InnoDB] DDL log post ddl : end for thread id : 12
[InnoDB] DDL log post ddl : begin for thread id : 12
[InnoDB] DDL log post ddl : end for thread id : 12
`

和CHANGE COLUMN的逻辑一样，就不再多说了。

### TRUNCATE TABLE

`mysql> truncate table t2;
Query OK, 0 rows affected (0.65 sec)

[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=114, thread_id=12, space_id=4, old_file_path=./my_test/#sql-ib1066-4028979813.ibd, new_file_path=./my_test/t2#P#p0.ibd]
[InnoDB] DDL log delete : 114
[InnoDB] DDL log insert : [DDL record: DROP, id=115, thread_id=12, table_id=1066]
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=116, thread_id=12, space_id=4, old_file_path=./my_test/#sql-ib1066-4028979813.ibd]
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=117, thread_id=12, space_id=14, old_file_path=./my_test/t2#P#p0.ibd]
[InnoDB] DDL log delete : 117
[InnoDB] DDL log insert : [DDL record: REMOVE CACHE, id=118, thread_id=12, table_id=1079, new_file_path=my_test/t2#P#p0]
[InnoDB] DDL log delete : 118
[InnoDB] DDL log insert : [DDL record: FREE, id=119, thread_id=12, space_id=14, index_id=183, page_no=4]
[InnoDB] DDL log delete : 119
[InnoDB] DDL log insert : [DDL record: FREE, id=120, thread_id=12, space_id=14, index_id=184, page_no=5]
[InnoDB] DDL log delete : 120
[InnoDB] DDL log insert : [DDL record: RENAME SPACE, id=121, thread_id=12, space_id=5, old_file_path=./my_test/#sql-ib1067-4028979814.ibd, new_file_path=./my_test/t2#P#p1.ibd]
[InnoDB] DDL log delete : 121
[InnoDB] DDL log insert : [DDL record: DROP, id=122, thread_id=12, table_id=1067]
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=123, thread_id=12, space_id=5, old_file_path=./my_test/#sql-ib1067-4028979814.ibd]
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=124, thread_id=12, space_id=15, old_file_path=./my_test/t2#P#p1.ibd]
[InnoDB] DDL log delete : 124
[InnoDB] DDL log insert : [DDL record: REMOVE CACHE, id=125, thread_id=12, table_id=1080, new_file_path=my_test/t2#P#p1]
[InnoDB] DDL log delete : 125
[InnoDB] DDL log insert : [DDL record: FREE, id=126, thread_id=12, space_id=15, index_id=185, page_no=4]
[InnoDB] DDL log delete : 126
[InnoDB] DDL log insert : [DDL record: FREE, id=127, thread_id=12, space_id=15, index_id=186, page_no=5]
[InnoDB] DDL log delete : 127
[InnoDB] DDL log post ddl : begin for thread id : 12
[InnoDB] DDL log replay : [DDL record: DELETE SPACE, id=123, thread_id=12, space_id=5, old_file_path=./my_test/#sql-ib1067-4028979814.ibd]
[InnoDB] DDL log replay : [DDL record: DROP, id=122, thread_id=12, table_id=1067]
[InnoDB] DDL log replay : [DDL record: DELETE SPACE, id=116, thread_id=12, space_id=4, old_file_path=./my_test/#sql-ib1066-4028979813.ibd]
[InnoDB] DDL log replay : [DDL record: DROP, id=115, thread_id=12, table_id=1066]
[InnoDB] DDL log post ddl : end for thread id : 12
`

首先先把旧的表空间找个临时的名称先存起来，记录一下要删除旧的表和这个旧的表空间（先记录，post-ddl再删除），然后走了创建表的逻辑，也就是用空的表空间和表来代替原来的旧的表空间和表。post-ddl阶段删除原来的表空间和表。

### DROP TABLE

`mysql> drop table t2;
Query OK, 0 rows affected (0.09 sec)

[InnoDB] DDL log insert : [DDL record: DROP, id=128, thread_id=12, table_id=1079]
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=129, thread_id=12, space_id=14, old_file_path=./my_test/t2#P#p0.ibd]
[InnoDB] DDL log insert : [DDL record: DROP, id=130, thread_id=12, table_id=1080]
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=131, thread_id=12, space_id=15, old_file_path=./my_test/t2#P#p1.ibd]
[InnoDB] DDL log post ddl : begin for thread id : 12
[InnoDB] DDL log replay : [DDL record: DELETE SPACE, id=131, thread_id=12, space_id=15, old_file_path=./my_test/t2#P#p1.ibd]
[InnoDB] DDL log replay : [DDL record: DROP, id=130, thread_id=12, table_id=1080]
[InnoDB] DDL log replay : [DDL record: DELETE SPACE, id=129, thread_id=12, space_id=14, old_file_path=./my_test/t2#P#p0.ibd]
[InnoDB] DDL log replay : [DDL record: DROP, id=128, thread_id=12, table_id=1079]
[InnoDB] DDL log post ddl : end for thread id : 12
`

记录删除操作，留在post-ddl阶段再执行。

### DROP DATABASE

`mysql> drop database my_test;
Query OK, 1 row affected (0.14 sec)

[InnoDB] DDL log insert : [DDL record: DROP, id=154, thread_id=12, table_id=1081]
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=155, thread_id=12, space_id=16, old_file_path=./my_test/t1#P#p0.ibd]
[InnoDB] DDL log insert : [DDL record: DROP, id=156, thread_id=12, table_id=1082]
[InnoDB] DDL log insert : [DDL record: DELETE SPACE, id=157, thread_id=12, space_id=17, old_file_path=./my_test/t1#P#p1.ibd]
[InnoDB] DDL log post ddl : begin for thread id : 12
[InnoDB] DDL log replay : [DDL record: DELETE SPACE, id=157, thread_id=12, space_id=17, old_file_path=./my_test/t1#P#p1.ibd]
[InnoDB] DDL log replay : [DDL record: DROP, id=156, thread_id=12, table_id=1082]
[InnoDB] DDL log replay : [DDL record: DELETE SPACE, id=155, thread_id=12, space_id=16, old_file_path=./my_test/t1#P#p0.ibd]
[InnoDB] DDL log replay : [DDL record: DROP, id=154, thread_id=12, table_id=1081]
[InnoDB] DDL log post ddl : end for thread id : 12
`

只记录了删除表的操作，也就是只记录了drop table的逻辑，和create database的逻辑相似，涉及创建数据库和删除数据库的操作不受ddl log保护，不支持原子性。

## 参考文档

[1] [13.1.1 Atomic Data Definition Statement Support](https://dev.mysql.com/doc/refman/8.0/en/atomic-ddl.html)

[2] [Atomic DDL in MySQL 8.0](https://mysqlserverteam.com/atomic-ddl-in-mysql-8-0/)

[3] [MySQL 8.0: Data Dictionary Architecture and Design](http://mysqlserverteam.com/mysql-8-0-data-dictionary-architecture-and-design/)

[4] [MySQL · 源码分析 · 8.0 · DDL的那些事](http://mysql.taobao.org/monthly/2020/05/05/)

[5] [MySQL · 源码分析 · 原子DDL的实现过程](http://mysql.taobao.org/monthly/2018/03/02/)

[6] [MySQL · 源码分析 · 8.0 原子DDL的实现过程续](http://mysql.taobao.org/monthly/2018/07/02/)

[7] [深入解读MySQL8.0 新特性 ：Crash Safe DDL](https://developer.aliyun.com/article/692258)

[8] [Atomic DDL揭秘](https://mp.weixin.qq.com/s/yym9E9gkrxqflL5dOTU6BA)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)