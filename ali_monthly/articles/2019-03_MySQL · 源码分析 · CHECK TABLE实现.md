# MySQL · 源码分析 · CHECK TABLE实现

**Date:** 2019/03
**Source:** http://mysql.taobao.org/monthly/2019/03/05/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2019 / 03
 ](/monthly/2019/03)

 * 当期文章

 PgSQL · 特性分析 · 内存管理机制
* MongoDB · 同步工具 · MongoShake原理分析
* MySQL · InnoDB · Redo log
* MSSQL · 最佳实践 · Always Encrypted
* MySQL · 源码分析 · CHECK TABLE实现
* PgSQL · 原理介绍 · PostgreSQL中的空闲空间管理
* MySQL · 引擎特性 · 8.0 Descending Index
* 理论基础 · Raft phd 论文中的pipeline 优化
* MySQL · 引擎特性 · MySQL 状态信息Status实现
* PgSQL · 应用案例 · 使用PostgreSQL生成数独方法1

 ## MySQL · 源码分析 · CHECK TABLE实现 
 Author: zhiyi 

 ## 前言
MySQL利用`CHECK TABLE`检查一张表或数张表的正确性，也可以用于检查视图的正确性，例如视图定义中引用的表是否存在。CHECK TABLE同时支持InnoDB，MyISAM，ARCHIVE和CSV表。

`/* CHECK TABLE语法 */
CHECK TABLE tbl_name [, tbl_name] ... [option] ...
option: {FOR UPGRADE | QUICK | FAST | MEDIUM | EXTENDED | CHANGED}
/* CHECK TABLE同样支持分区表 */
ALTER TABLE ... CHECK PARTITION
`

#### 检测版本兼容性
`FOR UPGRADE`选项用于检测表与当前版本MySQL的兼容性。它用于检测在创建表之后，是否在数据类型或者索引上发生了一些不兼容的修改操作。如果检测到一些不兼容的操作，它会在表上执行完整的检测过程，这需要较长的检测时间。不兼容性可能在数据类型的存储格式发生变化或者它的排序顺序发生变化时发生，比如在MySQL 5.0.3和5.0.5两个版本间DECIMAL类型存储结构的变化，在MySQL 4.1和5.0两个版本间TEXT列索引顺序的变化。

#### 检测数据一致性
CHECK TABLE还提供了一些其它检查选项，这些选项信息被传递到存储引擎层，用于检测数据的一致性：

`/* 类型: 含义 */
QUICK: 不扫描记录去检查索引结构的正确性，适用于InnoDB/MyISAM
FAST: 只检查哪些没有被正常关闭的表，仅适用于MyISAM
CHANGED: 检查那些在没有被正常关闭或上一次检查后被修改的表，仅适用于MyISAM
MEDIUM: 扫描记录验证那些删除链接的正确性，同时验证checksum的正确性，仅适用于MyISAM
EXTENDED: 扫描所有的记录，确保整张表数据100%的正确性，需要较长的执行时间。仅适用于MyISAM
`

如果没有指定QUICK，MEDIUM或者EXTENED，在MyISAM中默认的检查类型是MEDIUM。这些检测选项也可以组合使用，例如CHECK TABLE test_table FAST QUICK，在表上执行一个快速的检查去检测它是否被正常关闭。但在InnoDB中，它只有QUICK和非QUICK两种类型。本文以InnoDB为代表，下面分析CHECK TABLE在InnoDB中的注意事项。

#### CHECK TABLE在InnoDB中的注意事项

如果CHECK TABLE遇到损坏的页面，MySQL实例将退出以防止错误的传播（Bug #10132）。如果数据损坏发生在二级索引中，但表数据依然是可读的，运行CHECK TABLE仍将导致MySQL实例停止。

如果CHECK TABLE在主键索引中遇到错误的DB_TRX_ID或DB_ROLL_PTR项，CHECK TABLE将导致InnoDB访问到一个错误的undo log日志记录，导致MVCC相关服务崩溃。

如果CHECK TABLE遇到innoDB表或索引中的错误（错误包括二级索引中不正确的条目数或者错误的链接），它将报告这个错误，并标记索引/表的状态，避免使用这个索引或者表。

CHECK TABLE检查索引页结构，然后检查每个条目，但它不检查指向主键记录的键指针或遵循BLOB指针的指针。

当一个InnoDB表存储在自己的.ibd文件中时，.ibd文件的前3页包含的是头部元数据，而不是表或索引数据。CHECK TABLE语句不检测这部分数据的不一致性。要验证innodb.ibd文件的全部内容，请使用innochecksum命令。

在大型表上运行CHECK TABLE时，可能会在执行CHECK TABLE期间阻塞其他线程。为了避免超时，CHECK TABLE操作的信号量等待阈值（600秒）将延长2小时（7200秒）。如果InnoDB检测到信号量等待240秒或更长时间，它将开始向错误日志打印监控信息。如果锁请求超出信号量等待阈值，InnoDB将中止进程。

从MySQL 8.0.14开始，InnoDB支持并行访问主键索引，这有效提高了CHECK TABLE操作的性能。InnoDB在CHECK TABLE期间读取主键索引两次，第二次读取可以并行执行。要并行访问主键索引，必须将`innodb_parallel_read_threads`变量设置为大于1的值（默认值为4）。并行访问主键索引的线程数由innodb_parallel_read_threads设置或要扫描的索引子树数确定，以较小的值为准。

本文以MySQL 8.0.14代码为例，分析CHECK TABLE的实现。

#### CHECK TABLE的代码实现

`int ha_innobase::check(THD *thd, HA_CHECK_OPT *check_opt) 
{
 ...
 /* 如果表已经被标记为corrupted状态，就不需要再检查任何一个索引 */
 if (m_prebuilt->table->is_corrupted()) {
 if (thd_killed(m_user_thd)) {
 thd_set_kill_status(m_user_thd);
 }
 DBUG_RETURN(HA_ADMIN_CORRUPT);
 }
 /* 设置事务的隔离级别 */
 m_prebuilt->trx->isolation_level = TRX_ISO_REPEATABLE_READ;
 /* 遍历所有索引 */
 for (index = m_prebuilt->table->first_index(); index != NULL;
 index = index->next()) {
 /* 如果索引没有标记为corrupted并且check table的选项不是QUICK */
 if (!(check_opt->flags & T_QUICK) && !index->is_corrupted()) {
 /* 增大CHECK TABLE期间锁等待的时间 */
 os_atomic_increment_ulint(&srv_fatal_semaphore_wait_threshold,
 SRV_SEMAPHORE_WAIT_EXTENSION);
 /* 检查索引的一致性，这是非QUICK与QUICK的主要区别 */
 btr_validate_index(index, m_prebuilt->trx, false);
 /* 恢复锁等待的时间 */
 os_atomic_decrement_ulint(&srv_fatal_semaphore_wait_threshold,
 SRV_SEMAPHORE_WAIT_EXTENSION);
 }
 m_prebuilt->index/index_usable/sql_stat_start/template_type/n_template/.. = ..
 /* 设置并行线程数 */
 size_t n_threads = thd_parallel_read_threads(m_prebuilt->trx->mysql_thd);
 /* 并行扫描索引 */
 row_scan_index_for_mysql(m_prebuilt, index, n_threads, true, &n_rows);
 ... 
 }
 /* 恢复事务的隔离级别 */
 m_prebuilt->trx->isolation_level = old_isolation_level;
 ...
}
`

```
/* 检查索引结构的一致性 */
bool btr_validate_index(dict_index_t *index, const trx_t *trx, bool lockout)
{
 ...
 bool ok = true;
 mtr_t mtr;
 mtr_start(&mtr);
 /* 持有index的sx或者x锁 */
 if (lockout) mtr_x_lock(dict_index_get_lock(index), &mtr);
 else mtr_x_lock(dict_index_get_lock(index), &mtr) 
 /* 获取索引的根节点 */
 page_t *root = btr_root_get(index, &mtr);
 /* 获得树高 */
 ulint n = btr_page_get_level(root, &mtr);
 /* 验证每一层树结构的正确性 */
 for (ulint i = 0; i <= n; ++i) {
 if (!btr_validate_level(index, trx, n - i, lockout)) {
 ok = false;
 break;
 }
 }
 mtr_commit(&mtr);
 return ok;
}

```

```
/* 验证每一层树结构的正确性 */
static bool btr_validate_level(dict_index_t *index, const trx_t *trx, ulint level, bool lockout) 
{
 ...
 mtr_start(&mtr);
 mtr_sx_lock or mtr_x_lock(dict_index_get_lock(index), &mtr);
 /* 获得索引根节点的block/page/seg */
 block = btr_root_block_get(index, RW_SX_LATCH, &mtr);
 ...
 /* 遍历B-Tree，直到访问到指定的层次 */
 while (level != btr_page_get_level(page, &mtr))
 ...
 }
loop:
 ...
 /* 获取左右页号 */
 right_page_no = btr_page_get_next(page, &mtr);
 left_page_no = btr_page_get_prev(page, &mtr);
 /* 如果没有访问到这一层最后一个节点 */
 if (right_page_no != FIL_NULL) {
 /* 1. 根据right_page_no获取right_block / right_page, 检查链表指针的正确性 */
 /* 2. 检查page存储格式的正确性 */
 /* 3. 检查记录的有序性 */
 ...
 }
 /* 检查记录与父节点指针的正确性，并移动到下一个记录 */
 ...
node_ptr_fails:
 mtr_commit(&mtr);
 ...
 /* 如果没有到达这一层最后一个树节点，就goto loop */
 goto loop;
}

```

除了上述非QUICK选项需要执行的索引结构检查，CHECK TABLE还需要执行row_scan_index_for_mysql函数，确保所有记录的顺序是正确的。这部分代码在MySQL 8.0中有较大的变化，当扫描操作是非堵塞的并且–innodb-parallel-read-threads大于1时，它将索引划分成多个子树，支持多线程扫描。

`/* 针对COUNT(*)或者CHECK TABLE扫描索引。如果是CHECK TABLE，检查所有记录的顺序 */
dberr_t row_scan_index_for_mysql(row_prebuilt_t *prebuilt, const dict_index_t *index, size_t n_threads, bool check_keys, ulint *n_rows)
{
 ...
 /* 进行一系列检查，满足条件后执行多线程CHECK TABLE */
 if (prebuilt->select_lock_type == LOCK_NONE && index->is_clustered() &&
 (check_keys || prebuilt->trx->mysql_n_tables_locked == 0) &&
 !prebuilt->ins_sel_stmt && n_threads > 1) {
 /* 开启事务，设置视图 */
 trx_start_if_not_started_xa(prebuilt->trx, false);
 trx_assign_read_view(prebuilt->trx);
 /* 注册按照key值分区的reader对象 */
 Key_reader reader(prebuilt->table, trx, index, prebuilt, n_threads);
 /* 进入多线程检查函数 */
 if (!check_keys) {
 return (parallel_select_count_star(reader, n_rows));
 }
 return (parallel_check_table(reader, n_rows));
 }
 /* 以下单线程处理部分和5.6源码类似 */
 ...
 /* 定位到index的起始cursor */
 row_search_for_mysql(buf, PAGE_CUR_G, prebuilt, 0, 0);
loop: 
 /* 比较rec的大小，确保有序的状态是正确的 */
 ...
next_rec:
 /* 获取下一个rec */
 ret = row_search_for_mysql(buf, PAGE_CUR_G, prebuilt, 0, ROW_SEL_NEXT);
 /* 循环执行 */
 goto loop;
}
`

Key_reader在持有index的SX锁情况下，针对所有子树创建cursor，然后释放index的SX锁。子树的扫描过程为：

1. 从根结点开始读取每一层最左边的树节点；
2. 如果这一层能划分的子树数量少于指定线程数，就继续往下搜索。划分的方法包括按照page或者key值划分，分别在Phy_reader/Key_reader中实现；
3. 否则，使用该层，根据该层的最左记录向下查找直到叶子节点，然后开始扫描叶节点。

我们以`parallel_check_table`函数为例，分析多线程代码实现，具体原理读者可以查询 [WL#11720: InnoDB: Parallel read of index](https://dev.mysql.com/worklog/task/?id=11720)。

`static dberr_t parallel_check_table(Key_reader &reader, ulint *n_rows) 
{
 /* 初始化一系列的容器，例如Counter::Shards n_recs/n_dups/n_corrupt, std::vector类型的Tuples/Heaps/Blocks等等 */
 ... 
 /* 注册reader对象的回调函数，从而线程知道如何处理得到的每一行 */
 err = reader.read([&](size_t id, const buf_block_t *block, const rec_t *rec,
 dict_index_t *index, row_prebuilt_t *prebuilt) {
 ...
 auto heap = heaps[id];
 auto prev_tuple = prev_tuples[id];
 auto offsets = rec_get_offsets(rec, index, nullptr, ULINT_UNDEFINED, &heap);
 /* 比较rec和prev_tuple */
 auto cmp = cmp_dtuple_rec_with_match(prev_tuple, rec, index, offsets, &matched_fields);
 /* 根据cmp结果，判断是否出现顺序出错或者重复key的问题 */
 ... 
 /* 将这个rec和block记录到prev_blocks/prev_tuples中后返回 */
 ...
 return (DB_SUCCESS);
 }
 /* 收尾的一些工作 */
 ...
}
`

本文初步分析了CHECK TABLE的功能与实现，后续笔者会详细分析并行查询的代码实现与优化空间。欢迎大家持续关注内核月报。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)