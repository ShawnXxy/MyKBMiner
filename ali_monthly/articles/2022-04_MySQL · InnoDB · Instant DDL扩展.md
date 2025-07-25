# MySQL · InnoDB · Instant DDL扩展

**Date:** 2022/04
**Source:** http://mysql.taobao.org/monthly/2022/04/05/
**Images:** 2 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2022 / 04
 ](/monthly/2022/04)

 * 当期文章

 MySQL · 源码阅读 · 数据库的扫描方法
* MariaDB · 功能特性 · 无DDL延迟的主备复制
* MySQL · 源码阅读 · mysqld_safe的代码考古
* MySQL · 源码阅读 · 非阻塞异步C API简介
* MySQL · InnoDB · Instant DDL扩展

 ## MySQL · InnoDB · Instant DDL扩展 
 Author: 无哈 

 ## 概述

DDL（Data Definition Language）定义了数据库内部对象（库、表、列等）上的操作语义。

在MySQL中，根据是否阻塞DML，DDL可分为Copy DDL和Online DDL。其中Copy DDL在执行过程全程持有表级MDL X锁，禁止了其他并发DML操作；而从5.6版本开始引入了Online DDL，即只在元数据操作阶段持有表级MDL X锁，而其他数据操作阶段降级为MDL S锁以支持并发DML操作。

MySQL 8.0版本对DDL的进行了重大调整，除了引入原子DDL功能外，在8.0.12版本还为Online DDL的ALGORITHM参数引入了新的选项INSTANT，即对于只涉及到数据字典中的元数据的DDL操作可以在持有X锁的很短时间内“立即”完成元数据更新并返回。彼时支持在表的最后新增数据列、新增或删除虚拟列、修改列默认值、修改ENUM/SET的定义、修改索引类型、表重命名。前序数据库内核月报文章[MySQL特性-Instant Add Column](http://mysql.taobao.org/monthly/2020/03/01/)介绍了MySQL自8.0.12版本引入的Instant DDL Add Column的功能。本文介绍MySQL后续版本对Instant DDL的增强。

## Instant Rename Column

MySQL 8.0.28版本增强Instant DDL对列重命名的支持。

在Instant Add Column基础上实现列重命名较为简单直接。

* prepare_inplace_altr_table阶段加强对列重命名的检查，新增禁止外键引用列上执行instant rename column
* 新增Alter Flag INNOBASE_INSTANT_ALLOWED用于简化判断
* 新增INSTANT_OPERATION枚举类型来定义Instant DDL操作类型。

`const Alter_inplace_info::HA_ALTER_FLAGS INNOBASE_INSTANT_ALLOWED =
 Alter_inplace_info::ALTER_COLUMN_NAME |
 Alter_inplace_info::ADD_VIRTUAL_COLUMN |
 Alter_inplace_info::DROP_VIRTUAL_COLUMN |
 Alter_inplace_info::ALTER_VIRTUAL_COLUMN_ORDER |
 Alter_inplace_info::ADD_STORED_BASE_COLUMN;

enum class INSTANT_OPERATION {
 COLUMN_RENAME_ONLY, // 仅列重命名
 VIRTUAL_ADD_DROP_ONLY, // 仅添加或删除虚拟列
 VIRTUAL_ADD_DROP_WITH_RENAME, // 添加或删除虚拟列 + 列重命名
 INSTANT_ADD, // 添加列、添加或删除虚拟列 + 列重命名
 NONE
 };
`

## Instant Drop Column

### 元数据修改

* 系统表中引入TOTAL_ROW_VERSIONS，用于追踪当前表执行INSTANT DDL的次数，初始值为0，每次执行INSTANT Add/Drop column都会递增该值。每次表数据被重建时都会将TOTAL_ROW_VERSIONS重置为0。TOTAL_ROW_VERSIONS最大值为64，即最多支持连续64次INSTALT Add/Drop column操作。
* dd::Column的se_private_data中新增version_added、version_dropped、physical_pos，分别表示当且列添加、删除时的row_version和记录行中的物理位置。

### check_if_supported_inplace_alter

该函数用于在Server层通过handler接口检查存储引擎对当前DDL操作关于inplace逻辑的支持情况，新版本新增以下条件检查

* 检查dict_table_t上当前行版本号，如果行版本已经达到最大支持的版本（即64），如果DDL语句指定使用INSTANT算法则报错返回，否则回退为INPLACE DDL并重建表数据。
* 调用Instant_ddl_impl::is_instant_add_possible函数检查新增列（如果存在）是否会导致行记录大小超过物理页允许的最大大小。

### Instant_ddl_impl类

在新版本中，所有Instant DDL逻辑操作的实现都封装在Instant_ddl_impl类中。处理前述is_instant_add_possible函数外，主要是封装了commit_inplace_alter_table函数需要调用commit_instant_ddl函数

` // 对于其他只有非列操作、列重命名以及虚拟列相关的Instant操作保持原来版本
 case Instant_Type::INSTANT_ADD_DROP_COLUMN:
 dd_copy_private(*m_new_dd_tab, *m_old_dd_tab); // 从旧表中复制se_private_data到新的dd::Table中
 populate_to_be_instant_columns(); // 收集Instant DDL期间发生重命名、Drop和Add列的信息
 if (!m_cols_to_drop.empty()) { // 存在Instant drop的列
 commit_instant_drop_col();
 |--> commit_instant_drop_col_low
 |----> dd_copy_table_columns // 从旧的dd::Table复制列的元信息，跳过Instant Drop列，如果是第一次执行Instant Add/Drop还需要设置列Physical Position信息
 |----> dd_drop_instant_columns
 |--> copy_dropped_columns // 如果旧的dd::Table上已经存在Instant DDL删除过的列，需要将其从旧的dd::Table复制到新的dd::Table上，设置HIDDEN属性，并复制元信息
 // 遍历当前Instant DDL要删除的列，在dd::Table中添加新的dd::Column对象，设置HIDDEN属性，列名以_dropped_<row_version>为后缀
 |--> instant_update_table_cols_count // 更新dict_table_t上的列计数
 }
 if (!m_cols_to_add.empty()) { // 存在Instant add的列
 commit_instant_add_col();
 |--> commit_instant_add_col_low
 |----> dd_copy_table_columns
 |----> dd_add_instant_columns
 |--> copy_dropped_columns // 如果旧表上已经存在Instant DDL删除过的列，需要将其从旧表复制到新表上，设置HIDDEN属性，并复制元信息
 // 遍历当前Instant DDL要添加的列，在dd::Column的se_private_data中设置version_added、physical_pos、default/default_null属性的值
 |--> instant_update_table_cols_count // 更新dict_table_t上的列计数
 |----> dd_update_v_cols // 如果存在新增的虚拟列，设置相关属性信息
 }

 m_dict_table->current_row_version++; // 递增table上的row version
 innobase_discard_table(m_thd, m_dict_table); // 清楚统计信息，设置discard_after_ddl为true
`

### 实验

创建以下2张表，并插入100,000行数据

`CREATE TABLE sale(
 cn INT NOT null,
 vn INT NOT null,
 pn INT NOT null,
 dt DATE NOT null,
 qty INT NOT null,
 prc FLOAT NOT null
) ENGINE InnoDB;
CREATE TABLE sale_2 LIKE sale;
`

在sale上执行INSTANT DDL，sale_2上执行INPLACE DDL，执行效果如下图所示。
![pic](.img/e1b78016e2e0_instant_drop_column.jpg)
其中sale表上的INSTANT DDL执行耗时0.01s，更新TOTAL_ROW_VERSIONS为1；而sale_2表上的INPLACE DDL执行耗时0.22s，更新了TABLE_ID但未更新TOTAL_ROW_VERSIONS。

## Reference

* MySQL 8.0.28 Release Note
* MySQL 8.0.29 Release Note
* MySQL 8.0 源代码

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)