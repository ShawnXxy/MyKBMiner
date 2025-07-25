# MySQL · 内核分析 · InnoDB主键约束和唯一约束的实现分析

**Date:** 2021/04
**Source:** http://mysql.taobao.org/monthly/2021/04/05/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2021 / 04
 ](/monthly/2021/04)

 * 当期文章

 MySQL · 内核特性 · Automatic connection failover
* MongoDB · 内核特性 · 一致性模型设计与实现
* MySQL · 资源管理 · PFS内存管理分析
* MySQL · HTAP · 分析型执行引擎
* MySQL · 内核分析 · InnoDB主键约束和唯一约束的实现分析
* MySQL · 源码阅读 · Window function解析

 ## MySQL · 内核分析 · InnoDB主键约束和唯一约束的实现分析 
 Author: hangfeng 

 ## 概述
关系数据库通过约束机制来指定插入数据的规则，从而确保数据的完整性和一致性。MySQL的约束主要包括: 主键约束、唯一约束、非空约束以及外键约束。主键约束指定表中的一列或者几列的组合的值在表中不能出现空值和重复值，即唯一的标识一行记录。唯一约束指定表中某一列或多个列不能有相同的两行或者两行以上的数据存在，和主键约束不同，唯一约束允许为NULL，只是只能有一行。
InnoDB主要通过锁机制来实现主键约束和唯一约束，锁类型包括表锁和行锁，而行锁还细分为记录锁(RK)、间隙锁(GK)、插入意向锁(IK)、Next-Key(NK)等更细的子类型，锁兼容矩阵如下：

 行是已经存在的锁，列是要加的锁
 RK
 GK
 IK
 NK

 RK
 0
 1
 1
 0

 GK
 1
 1
 1
 1

 IK
 1
 0
 1
 0

 NK
 0
 1
 1
 0

## 主键约束
以插入为例，实现主键约束的主要过程如下：

` |-Sql_cmd_insert_values::execute_inner() // Insert one or more rows from a VALUES list into a table
 |-write_record
 |-handler::ha_write_row() // 调用存储引擎的接口
 |-ha_innobase::write_row()
 |-row_insert_for_mysql
 |-row_insert_for_mysql_using_ins_graph
 |-trx_start_if_not_started_xa
 |-trx_start_low // 激活事务，事务状态由 not_active 变为 active
 |-row_get_prebuilt_insert_row // Gets pointer to a prebuilt dtuple used in insertions
 |-row_mysql_convert_row_to_innobase // 记录格式从MySQL转换成InnoDB, 不同数据类型处理方式不同，比如整形server端是小端存储，innodb是大端存储
 |-row_ins_step
 |-trx_write_trx_id(node->trx_id_buf, trx->id)
 |-lock_table // 给表加IX锁
 |-row_ins // 插入记录
 |-while (node->index != NULL)
 |-row_ins_index_entry_step // 向索引中插入记录,把 innobase format field 的值赋给对应的index entry field
 |-row_ins_index_entry_set_vals // 根据该索引以及原记录，将组成索引的列的值组成一个记录
 |-dtuple_check_typed // 检查组成的记录的有效性
 |-row_ins_index_entry // 插入索引项
 |-row_ins_clust_index_entry // 插入聚集索引
 |-row_ins_clust_index_entry_low // 先尝试乐观插入，修改叶子节点 BTR_MODIFY_LEAF
 |-mtr_t::mtr_t()
 |-mtr_t::start()
 |-btr_pcur_t::open()
 |-btr_cur_search_to_nth_level // 将cursor移动到索引上待插入的位置
 |-buf_page_get_gen //取得本层页面，首次为根页面
 |-page_cur_search_with_match_bytes // 在本层页面进行游标定位
 |-row_ins_duplicate_error_in_clust // 判断插入项是否存在唯一键冲突
 |-row_ins_set_shared_rec_lock // 对cursor 对应的已有记录加S锁（可能会等待）保证记录上的操作，包括：Insert/Update/Delete 已经提交或者回滚
 |-lock_clust_rec_read_check_and_lock // 判断cursor对应的记录上是否存在隐式锁, 若存在，则将隐式锁转化为显示锁
 |-lock_rec_convert_impl_to_expl // 隐式锁转换
 |-lock_rec_lock //如果上面的隐式锁转化成功，此处加S锁将会等待，直到活跃事务释放锁。
 |-row_ins_dupl_err_with_rec // S锁加锁完成之后，可以再次做判断，最终决定是否存在唯一键冲突, 
 // 1. 判断insert记录与cursor对应的记录取值是否相同, 
 // 2. 二级唯一键值锁引，可以存在多个 NULL 值, 
 // 3. 最后判断记录的delete flag状态，判断记录是否被删除提交
 |-return !rec_get_deleted_flag();
 |-btr_cur_optimistic_insert // 乐观插入
 |-btr_cur_pessimistic_insert // 乐观插入失败则进行悲观插入
 |-mtr_t::commit() mtr_commit //Commit a mini-transaction.
 |-btr_pcur_t::close()
`

#### 主键判重主函数
row_ins_duplicate_error_in_clust是主键判重的主函数，关键代码如下：

`static MY_ATTRIBUTE((warn_unused_result)) dberr_t
 row_ins_duplicate_error_in_clust(
 ulint flags, /*!< in: undo logging and locking flags */
 btr_cur_t *cursor, /*!< in: B-tree cursor */
 const dtuple_t *entry, /*!< in: entry to insert */
 que_thr_t *thr, /*!< in: query thread */
 mtr_t *mtr) /*!< in: mtr */
{
 dberr_t err;
 rec_t *rec;
 ulint n_unique;
 trx_t *trx = thr_get_trx(thr);
 mem_heap_t *heap = NULL;
 ulint offsets_[REC_OFFS_NORMAL_SIZE];
 ulint *offsets = offsets_;
 rec_offs_init(offsets_);
 
 n_unique = dict_index_get_n_unique(cursor->index);

 if (cursor->low_match >= n_unique) {
 rec = btr_cur_get_rec(cursor);

 if (!page_rec_is_infimum(rec)) {
 offsets =
 rec_get_offsets(rec, cursor->index, offsets, ULINT_UNDEFINED, &heap);

 ulint lock_type;

 lock_type = ((trx->isolation_level <= TRX_ISO_READ_COMMITTED) ||
 (cursor->index->table->skip_gap_locks()))
 ? LOCK_REC_NOT_GAP
 : LOCK_ORDINARY;
 
 if (flags & BTR_NO_LOCKING_FLAG) {
 /* Do nothing if no-locking is set */
 err = DB_SUCCESS;
 } else if (trx->duplicates) {
 err =
 row_ins_set_exclusive_rec_lock(lock_type, btr_cur_get_block(cursor),
 rec, cursor->index, offsets, thr);
 } else {
 // 对cursor对应的已有记录加S锁（可能会等待）保证记录上的操作，
 // 包括：Insert/Update/Delete 已经提交或者回滚
 err = row_ins_set_shared_rec_lock(lock_type, btr_cur_get_block(cursor),
 rec, cursor->index, offsets, thr);
 }

 switch (err) {
 case DB_SUCCESS_LOCKED_REC:
 case DB_SUCCESS:
 break;
 default:
 goto func_exit;
 }
 
 // S锁加锁完成之后，可以再次做判断，最终决定是否存在唯一键冲突
 if (row_ins_dupl_error_with_rec(rec, entry, cursor->index, offsets)) {
 duplicate:
 trx->error_info = cursor->index;
 err = DB_DUPLICATE_KEY;
 goto func_exit;
 }
 }
 }

 if (cursor->up_match >= n_unique) {
 rec = page_rec_get_next(btr_cur_get_rec(cursor));

 if (!page_rec_is_supremum(rec)) {
 offsets =
 rec_get_offsets(rec, cursor->index, offsets, ULINT_UNDEFINED, &heap);

 if (trx->duplicates) {
 err = row_ins_set_exclusive_rec_lock(LOCK_REC_NOT_GAP,
 btr_cur_get_block(cursor), rec,
 cursor->index, offsets, thr);
 } else {
 err = row_ins_set_shared_rec_lock(LOCK_REC_NOT_GAP,
 btr_cur_get_block(cursor), rec,
 cursor->index, offsets, thr);
 }

 switch (err) {
 case DB_SUCCESS_LOCKED_REC:
 case DB_SUCCESS:
 break;
 default:
 goto func_exit;
 }

 if (row_ins_dupl_error_with_rec(rec, entry, cursor->index, offsets)) {
 goto duplicate;
 }
 }

 /* This should never happen */
 ut_error;
 }

 err = DB_SUCCESS;
func_exit:
 if (UNIV_LIKELY_NULL(heap)) {
 mem_heap_free(heap);
 }
 return (err);
}
`

#### 对cursor对应的已有记录加锁
通常MySQL的插入操作是不加锁的，但如果在插入或更新记录时，检查到存在重复记录（有可能被标记为删除），对于普通的INSERT/UPDATE，会加S锁，而对于类似REPLACE INTO或者INSERT … ON DUPLICATE这种SQL加的则是X锁。如果是RC隔离级别，加的是LOCK_REC_NOT_GAP类型的锁，如果是RR隔离级别，加的则是next-key类型的锁。以S锁为例，调用的函数是lock_clust_rec_read_check_and_lock，其关键代码如下:

`dberr_t lock_clust_rec_read_check_and_lock(
 ulint flags, const buf_block_t *block, const rec_t *rec,
 dict_index_t *index, const ulint *offsets, select_mode sel_mode,
 lock_mode mode, ulint gap_mode, que_thr_t *thr) {
 dberr_t err;
 ulint heap_no;

 if ((flags & BTR_NO_LOCKING_FLAG) || srv_read_only_mode ||
 index->table->is_temporary()) {
 return (DB_SUCCESS);
 }

 heap_no = page_rec_get_heap_no(rec);
 
 // 判断记录上是否存在隐式锁，如果存在则将其转换为显示锁
 if (heap_no != PAGE_HEAP_NO_SUPREMUM) {
 lock_rec_convert_impl_to_expl(block, rec, index, offsets);
 }

 LockSysGuard lock_guard(LOCK_REC_SHARD, block);
 
 // 加S锁, 如果上面的隐式锁转化成功, 此处加锁将会等待, 直到活跃事务释放锁
 err = lock_rec_lock(false, sel_mode, mode | gap_mode, block, heap_no, index,
 thr, &lock_guard);

 lock_guard.release();
 DEBUG_SYNC_C("after_lock_clust_rec_read_check_and_lock");
 ut_ad(err == DB_SUCCESS || err == DB_SUCCESS_LOCKED_REC ||
 err == DB_LOCK_WAIT || err == DB_DEADLOCK || err == DB_SKIP_LOCKED ||
 err == DB_LOCK_NOWAIT);
 return (err);
}
`
隐式锁转换的相关资料可以参考文章《InnoDB隐式锁功能解析》[http://mysql.taobao.org/monthly/2020/09/06/](http://mysql.taobao.org/monthly/2020/09/06/)

## 二级索引唯一约束
继续以插入为例，二级索引唯一约束的主要实现过程如下：

` |-Sql_cmd_insert_values::execute_inner
 |-write_record
 |-handler::ha_write_row
 |-ha_innobase::write_row
 |-row_insert_for_mysql
 |-row_insert_for_mysql_using_ins_graph
 |-trx_start_if_not_started_xa
 |-trx_start_low // 激活事务，事务状态由 not_active 变为 active
 |-row_get_prebuilt_insert_row // Gets pointer to a prebuilt dtuple used in insertions
 |-row_mysql_convert_row_to_innobase // 记录格式从MySQL转换成InnoDB
 |-row_ins_step
 |-trx_write_trx_id(node->trx_id_buf, trx->id)
 |-lock_table // 给表加IX锁
 |-row_ins // 插入记录
 |-while (node->index != NULL)
 |-row_ins_index_entry_step // 向索引中插入记录,把 innobase format field 的值赋给对应的index entry field
 |-row_ins_index_entry_set_vals // 根据该索引以及原记录，将组成索引的列的值组成一个记录
 |-dtuple_check_typed // 检查组成的记录的有效性
 |-row_ins_index_entry // 插入索引项
 |-row_ins_sec_index_entry // 插入二级索引
 |-row_ins_sec_index_entry_low // 先尝试乐观插入，修改叶子节点 BTR_MODIFY_LEAF
 |-mtr_t::mtr_t()
 |-mtr_t::start()
 |-btr_cur_search_to_nth_level // 将cursor移动到索引上待插入的位置, PAGE_CUR_LE(<=), BTR_MODIFY_LEAF, dtuple带主键
 |-buf_page_get_gen //取得本层页面，首次为根页面
 |-page_cur_search_with_match_bytes // 在本层页面进行游标定位
 |-if (dict_index_is_unique(index) && // 先做粗略判断，跳过不需要判重的情况
 (cursor.low_match >= n_unique || cursor.up_match >= n_unique))
 |-mtr_commit(&mtr); // 释放latch
 |-row_ins_scan_sec_index_for_duplicate
 |-n_fields_cmp = dtuple_get_n_fields_cmp(entry);
 |-dtuple_set_n_fields_cmp(entry, n_unique); // 只包含二级索引唯一键
 |-btr_pcur_open_low // 只包含二级索引唯一键, 返回时持有第一个满足条件记录所在的page latch
 |-btr_cur_search_to_nth_level // PAGE_CUR_GE(>=), BTR_SEARCH_LEAF, dtuple只包含二级索引唯一键
 |-page_cur_search_with_match
 |-do
 |-const rec_t* rec = btr_pcur_get_rec(&pcur);
 |-row_ins_set_shared_rec_lock // 加行锁，包括deleted record
 |-row_ins_dupl_error_with_rec // 判断是否重复
 |-while (btr_pcur_move_to_next(&pcur, mtr)) // 扫描下一个记录
 |-mtr_commit(&mtr); // 释放page latch
 |-btr_cur_optimistic_insert // 乐观插入
 |-btr_cur_pessimistic_insert // 乐观插入失败则进行悲观插入
 |-mtr_t::commit() mtr_commit // Commit a mini-transaction.
 |-btr_pcur_t::close()
`

#### 唯一二级索引判重主函数
row_ins_scan_sec_index_for_duplicate是唯一二级索引判重的主函数，关键代码如下：

`static MY_ATTRIBUTE((warn_unused_result)) dberr_t
 row_ins_scan_sec_index_for_duplicate(
 ulint flags, /*!< in: undo logging and locking flags */
 dict_index_t *index, /*!< in: non-clustered unique index */
 dtuple_t *entry, /*!< in: index entry */
 que_thr_t *thr, /*!< in: query thread */
 bool s_latch, /*!< in: whether index->lock is being held */
 mtr_t *mtr, /*!< in/out: mini-transaction */
 mem_heap_t *offsets_heap)
{
 n_unique = dict_index_get_n_unique(index);

 n_fields_cmp = dtuple_get_n_fields_cmp(entry);
 
 // 只包含二级索引唯一键，不包括主键字段
 dtuple_set_n_fields_cmp(entry, n_unique);
 
 // 只包含二级索引唯一键, 返回时持有第一个满足条件记录所在的page latch
 btr_pcur_open(
 index, entry, PAGE_CUR_GE,
 s_latch ? BTR_SEARCH_LEAF | BTR_ALREADY_S_LATCHED : BTR_SEARCH_LEAF,
 &pcur, mtr);

 allow_duplicates = thr_get_trx(thr)->duplicates;

 do {
 const rec_t *rec = btr_pcur_get_rec(&pcur);
 const buf_block_t *block = btr_pcur_get_block(&pcur);
 
 // 无论是RC还是RR隔离级别，均将加锁类型设置为LOCK_ORDINARY
 const ulint lock_type =
 index->table->skip_gap_locks() ? LOCK_REC_NOT_GAP : LOCK_ORDINARY;

 if (page_rec_is_infimum(rec)) {
 continue;
 }

 offsets =
 rec_get_offsets(rec, index, offsets, ULINT_UNDEFINED, &offsets_heap);

 if (flags & BTR_NO_LOCKING_FLAG) {
 } else if (allow_duplicates) {
 // 加X锁
 err = row_ins_set_exclusive_rec_lock(lock_type, block, rec, index,
 offsets, thr);
 } else {
 if (index->table->skip_gap_locks()) {
 if (page_rec_is_supremum(rec)) {
 continue;
 }

 if (cmp_dtuple_rec(entry, rec, index, offsets) < 0) {
 goto end_scan;
 }
 }
 // 加S锁
 err = row_ins_set_shared_rec_lock(lock_type, block, rec, index, offsets,
 thr);
 }

 switch (err) {
 case DB_SUCCESS_LOCKED_REC:
 err = DB_SUCCESS;
 case DB_SUCCESS:
 break;
 default:
 goto end_scan;
 }

 if (page_rec_is_supremum(rec)) {
 continue;
 }
 
 cmp = cmp_dtuple_rec(entry, rec, index, offsets);

 if (cmp == 0 && !index->allow_duplicates) {
 // 加锁成功后判断是否重复
 if (row_ins_dupl_error_with_rec(rec, entry, index, offsets)) {
 err = DB_DUPLICATE_KEY;
 thr_get_trx(thr)->error_info = index;
 goto end_scan;
 }
 } else {
 ut_a(cmp < 0 || index->allow_duplicates);
 goto end_scan;
 }
 // 扫描下一个记录，直到遇到第一个不同的记录
 } while (btr_pcur_move_to_next(&pcur, mtr));

end_scan:
 dtuple_set_n_fields_cmp(entry, n_fields_cmp);

 DBUG_RETURN(err);
}
`

#### 唯一二级索引判重时的加锁逻辑
在函数row_ins_scan_sec_index_for_duplicate中，无论是RR还是RC隔离级别，当插入一条带唯一约束的记录时，如果表上已经存在了这条记录，或者有一条标记删除的相同键值记录时，就需要对这条记录加S LOCK_ORDINARY锁。在MySQL 5.6.12版本之后，有人认为RC隔离级别下无需加LOCK_ORDINARY锁, 只需要加LOCK_REC_NOT_GAP类型的S锁，这个改动导致了非常严重的二级索引唯一约束失效问题([bug#68021](http://bugs.mysql.com/bug.php?id=68021))。

`二级索引唯一约束失效重现步骤：

修改row0ins.cc 函数row_ins_scan_sec_index_for_duplicate:
 const ulint lock_type =
 index->table->skip_gap_locks() ? LOCK_REC_NOT_GAP : LOCK_ORDINARY;
 const ulint lock_type = LOCK_REC_NOT_GAP;

CREATE TABLE t1 (
 c1 int(11) NOT NULL AUTO_INCREMENT,
 c2 int(11) DEFAULT NULL,
 PRIMARY KEY (c1),
 UNIQUE KEY k_c2 (c2)
 ) ENGINE=InnoDB AUTO_INCREMENT=5;

insert into t1 values (3, 5), (9, 10);

Transaction 1:
begin;
delete from t1 where c2 = 5;

Transaction 2:
insert into t1 select 1,5; // wait

Transaction 3:
insert into t1 select 2,5; // wait

Transaction 1:
commit;

root@localhost : test 10:31:03> select * from t1; // 二级索引唯一约束失效, trx2和trx3均插入成功
+----+------+
| c1 | c2 |
+----+------+
| 1 | 5 |
| 2 | 5 |
| 9 | 10 |
+----+------+
3 rows in set (0.00 sec)
`

二级索引唯一约束失效的原因

`Transaction 1:
begin;
delete from t1 where c2 = 5; // 加X NOT_GAP lock，成功

Transaction 2:
insert into t1 select 1,5; // 加S NOT_GAP lock，等待

Transaction 3:
insert into t1 select 2,5; // 加S NOT_GAP lock，等待

Transaction 1:
commit;
T1 释放X NOT_GAP lock
T2 加S NOT_GAP lock成功
T3 加S NOT_GAP lock成功（S锁互相兼容）
T2 加X insert intention lock成功（X IK和S RK兼容）
T3 加X insert intention lock成功（X IK和S RK,X IK均兼容）
T2 插入记录成功
T3 插入记录成功
`
官方发现这个问题后又把RK改成了NK，如果T2,T3加的是LOCK_ORDINARY lock，由于LOCK_ORDINARY lock与insert intention lock不兼容，在插入阶段T2和T3会发生死锁，其中一个事务会回滚，因此不会出现二级索引唯一约束失效的问题。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)