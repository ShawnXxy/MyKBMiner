# MySQL · 源码分析 · 临时表与TempTable存储引擎Allocator

**Date:** 2021/07
**Source:** http://mysql.taobao.org/monthly/2021/07/07/
**Images:** 1 images downloaded

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

 ## MySQL · 源码分析 · 临时表与TempTable存储引擎Allocator 
 Author: 异陌 

 本文基于MySQL Community 8.0.25 Version

## 临时表创建
### 显式创建临时表
临时表可使用 `CREATE TEMPORARY TABLE` 语句创建，它仅在当前session中可以使用，session断开时临时表会被自动drop掉。因此开启不同的session时可以使用相同的临时表名。

`CREATE TEMPORARY TABLE temp_table (id int primary key auto_increment, payload int);
INSERT INTO temp_table (payload) VALUES (100), (200), (300);
`
8.0.13 版本后的MySQL的默认临时表存储引擎是 `TempTable`。
可以在创建临时表时指定存储引擎，如指定使用 `Memory`：

`CREATE TEMPORARY TABLE temp_table_memory (x int) ENGINE=MEMORY;
`

### 隐式创建临时表

一些情况下，server在执行语句时会创建内部临时表，用户无法对此进行直接控制。如：

* `UNION` 语句
* 派生表（即在查询的`FROM` 子句中生成的表）
* 通用表表达式（`WITH` 子句下的表达式）
* `DISTINCT` 组合 `ORDER BY`
* `INSERT...SELECT` 语句。MySQL会创建内部临时表保存`SELECT` 的结果，并将这些row `INSERT` 到目标表中
* 窗口函数
* `GROUP_CONCAT()`、`COUNT(DISTINCT)` 表达式

**如何判断SQL语句是否隐式使用了临时表**：使用`EXPLAIN`语句并检查`EXTRA` 列，若显示`Using temporary`，则说明使用了临时表。

## 临时表存储引擎
### Memory
在MySQL 8.0.13版本引入`TempTable` 存储引擎前，使用`Memory` 存储引擎来创建内存中的临时表。

但是它有不足之处如下：

* 不支持含有BLOB或TEXT类型的表，这种情况在8.0.13版本前只能将临时表建在disk上
* 对于VARCHAR类型的字段，如VARCHAR(200)，映射到内存里处理的字段变为CHAR(200)，容易造成空间浪费
 ### InnoDB / MyISAM
 MySQL 8.0.16及以后，server使用`InnoDB` 存储引擎在disk上创建临时表。`internal_tmp_disk_storage_engine` 变量已被删除，用户无法再自定义选择`MyISAM` 存储引擎。

 ### TempTable
 MySQL 8.0.13版本引入 `TempTable` 存储引擎，此后`TempTable` 成为在内存中创建临时表的默认存储引擎。使用`internal_tmp_mem_storage_engine`可以指定内存中创建临时表的存储引擎（另一个是`MEMORY`）。
相较于`MEMORY` 存储引擎，`TempTable` 存储引擎可以支持变长数据类型的存储。

## TempTable 内存分配策略及源码分析
MySQL 8.0.23版本之后引入`temptable_max_mmap` 变量，因此本次分析针对8.0.23版本后的策略及其源码。

### 内存分配策略
* 若临时表分配空间未超过`temptable_max_ram`值，则使用`TempTable`存储引擎在RAM中为临时表分配空间
* 若临时表大小超过了`temptable_max_ram`值
 
 若`temptable_use_mmap=on`且`temptable_max_mmap > 0`，则从memory-maped file中为临时表分配空间
 
 在此分配过程中，若临时表大小小于了`temptable_max_ram`值，则可以继续从RAM中分配空间
* 若临时表大小超过了`temptable_max_mmap`值，则使用`InnoDB` 临时表从disk上分配空间，并将内存中的临时表迁移到disk上

 若`temptable_use_mmap=off`或`temptable_max_mmap=0`，则使用`InnoDB` 存储引擎从disk上分配空间，并将内存中的临时表迁移到disk上

### 源码分析

#### Allocator类及RAM、MMAP空间分配

TempTable 存储引擎分配空间由`Allocator`类完成，位于storage/temptable/include/temptable/allocator.h。

`template <class T,
 class AllocationScheme = Exponential_growth_preferring_RAM_over_MMAP>
class Allocator {
 ...
`

首先可以看到`Allocator`类有一个模版参数类`AllocationScheme`

`template <typename Block_size_policy, typename Block_source_policy>
struct Allocation_scheme {
 static Source block_source(size_t block_size) {
 return Block_source_policy::block_source(block_size);
 }
 static size_t block_size(size_t number_of_blocks, size_t n_bytes_requested) {
 return Block_size_policy::block_size(number_of_blocks, n_bytes_requested);
 }
};
`
它用来控制Allocator的分配机制。其中`block_size()`方法会指定每次分配的Block大小，`block_source()`方法会指定存储介质使用策略。

`Allocator`的默认模版参数为`Exponential_growth_preferring_RAM_over_MMAP`，其具体为：

`using Exponential_growth_preferring_RAM_over_MMAP =
 Allocation_scheme<Exponential_policy, Prefer_RAM_over_MMAP_policy>;
`
可以看到，默认的block_size的分配策略为`Exponential_policy`、block_source策略为`Prefer_RAM_over_MMAP_policy`。

`struct Exponential_policy {
 static size_t block_size(size_t number_of_blocks, size_t n_bytes_requested) {
 size_t block_size_hint;
 if (number_of_blocks < ALLOCATOR_MAX_BLOCK_MB_EXP) {
 block_size_hint = (1ULL << number_of_blocks) * 1_MiB;
 } else {
 block_size_hint = ALLOCATOR_MAX_BLOCK_BYTES;
 }
 return std::max(block_size_hint, Block::size_hint(n_bytes_requested));
 }
};
`
由上述代码可见，`Exponential_policy`的`block_size()`方法每次分配的block大小`block_size_hint`是指数级增加的，直到到了最大值`ALLOCATOR_MAX_BLOCK_BYTES`，此后每次分配的block的大小都是`ALLOCATOR_MAX_BLOCK_BYTES`。若函数调用者请求的block大小大于`block_size_hint`，则返回该请求大小，即使它可能大于`ALLOCATOR_MAX_BLOCK_BYTES`。

`struct Prefer_RAM_over_MMAP_policy {
 static Source block_source(uint32_t block_size) {
 if (MemoryMonitor::RAM::consumption() < MemoryMonitor::RAM::threshold()) {
 if (MemoryMonitor::RAM::increase(block_size) <=
 MemoryMonitor::RAM::threshold()) {
 return Source::RAM;
 } else {
 MemoryMonitor::RAM::decrease(block_size);
 }
 }
 if (MemoryMonitor::MMAP::consumption() < MemoryMonitor::MMAP::threshold()) {
 if (MemoryMonitor::MMAP::increase(block_size) <=
 MemoryMonitor::MMAP::threshold()) {
 return Source::MMAP_FILE;
 } else {
 MemoryMonitor::MMAP::decrease(block_size);
 }
 }
 throw Result::RECORD_FILE_FULL;
 }
};
`

由上述代码可见，Allocator会首先在RAM分配空间，若RAM消耗超过了`RAM::threshold()`，即`temptable_max_ram`，则会开始尝试在mmap files上分配空间。
`MMAP::threshold()`代码为：

`static size_t threshold() {
 if (temptable_use_mmap) {
 return temptable_max_mmap;
 } else {
 return 0;
 }
}
`
若`temptable_use_mmap=false`，`threashold()`函数返回0；当`temptable_max_mmap=0`时，`threshold()` 函数实际返回的也是0。这两种情况下不会在mmap file上分配空间，而是直接抛出`Result::RECORD_FILE_FULL`异常。所以可以看到`temptable_max_mmap=0`实际上是等价于`temptable_use_mmap=false`的。

当`temptable_use_map=true`、`temptable_max_mmap>0`且mmap file分配空间小于`temptable_max_mmap`时，Allocator会在mmap file上为临时表分配空间。

如果在此过程中，RAM分配使用空间小于了`temptable_max_ram`，则还会优先从RAM分配空间。

`Allocator`类的`allocate(size_t)`方法会调用以上方法为临时表进行空间的分配：

`else if (m_state->current_block.is_empty() ||
 !m_state->current_block.can_accommodate(n_bytes_requested)) {
 const size_t block_size = AllocationScheme::block_size(
 m_state->number_of_blocks, n_bytes_requested);
 m_state->current_block =
 Block(block_size, AllocationScheme::block_source(block_size));
 block = &m_state->current_block;
 ++m_state->number_of_blocks;
} 
`

当当前block为空时，需要进行空间分配，会调用Block构造函数为`current_block`分配空间，Block构造函数中会掉用上述的`Exponential_policy::block_size()`方法进行空间分配大小计算以及调用`Prefer_RAM_over_MMAP_policy::block_source()`进行存储介质的选择以及空间分配。

#### disk空间分配

由上述代码可知，RAM空间不够且mmap file不允许使用或mmap file空间不够时，`TempTable` 存储引擎会抛出`Result::RECORD_FILE_FULL`异常，即`HA_ERR_RECORD_FILE_FULL`异常。

此时MySQL会将临时表迁移到disk上，由于使用的是`InnoDB` 存储引擎，所以在disk上建临时表的代码自然不会在`storage/temptable`路径下，而是在更上层的server层。

此功能的代码逻辑为：当server层调用`ha_write_row`向临时表中写入row时，同时会调用`create_ondisk_from_heap()`函数。

如sql/sql_union.cc的 `Query_result_union::send_data()`函数中：

`const int error = table->file->ha_write_row(table->record[0]);
if (!error) {
 m_rows_in_table++;
 return false;
}
// create_ondisk_from_heap will generate error if needed
if (!table->file->is_ignorable_error(error)) {
 bool is_duplicate;
 if (create_ondisk_from_heap(thd, table, error, true, &is_duplicate))
 return true; /* purecov: inspected */
 // Table's engine changed, index is not initialized anymore
 if (table->hash_field) table->file->ha_index_init(0, false);
 if (!is_duplicate) m_rows_in_table++;
}
`

`create_ondisk_from_heap()`函数的作用是当接受到`HA_ERR_RECORD_FILE_FULL`异常时，即内存中的表已满时，会将该表迁移到disk上。

`create_ondisk_from_heap()`函数中，当接收到的error不是`HA_ERR_RECORD_FILE_FULL`时，会直接返回：

`if (error != HA_ERR_RECORD_FILE_FULL) {
 /*
 We don't want this error to be converted to a warning, e.g. in case of
 INSERT IGNORE ... SELECT.
 */
 wtable->file->print_error(error, MYF(ME_FATALERROR));
 return true;
}
`

它会使用`InnoDB` 存储引擎创建新的表：

`share.db_plugin = ha_lock_engine(thd, innodb_hton);
// ... 
new_table.s = &share; // New table points to new share

new_table.file =
 get_new_handler(&share, false, old_share->alloc_for_tmp_file_handler,
 new_table.s->db_type());
`
并将临时表迁移到该位于disk的表上：

`/*
copy all old rows from heap table to on-disk table
This is the only code that uses record[1] to read/write but this
is safe as this is a temporary on-disk table without timestamp/
autoincrement or partitioning.
*/
while (!table->file->ha_rnd_next(new_table.record[1])) {
 write_err = new_table.file->ha_write_row(new_table.record[1]);
 DBUG_EXECUTE_IF("raise_error", write_err = HA_ERR_FOUND_DUPP_KEY;);
 if (write_err) goto err_after_open;
}
/* copy row that filled HEAP table */
if ((write_err = new_table.file->ha_write_row(table->record[0]))) {
 if (!new_table.file->is_ignorable_error(write_err) ||
 !ignore_last_dup)
 goto err_after_open;
 if (is_duplicate) *is_duplicate = true;
} else {
 if (is_duplicate) *is_duplicate = false;
}
`

## 参考资料
[Internal Temporary Table Use in MySQL](https://dev.mysql.com/doc/refman/8.0/en/internal-temporary-tables.html)
[MySQL · 引擎特性 · 临时表改进](http://mysql.taobao.org/monthly/2019/09/01/)
[MySQL 8.0: Support for BLOBs in TempTable engine](https://mysqlserverteam.com/mysql-8-0-support-for-blobs-in-temptable-engine/?spm=a2c4e.10696291.0.0.496519a4wfiSPz)
[Temporary Tables in MySQL](https://blog.toadworld.com/2017/09/27/temporary-tables-in-mysql)
[mysql-server 8.0.25 Source Code](https://github.com/mysql/mysql-server/tree/mysql-cluster-8.0.25)/

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)