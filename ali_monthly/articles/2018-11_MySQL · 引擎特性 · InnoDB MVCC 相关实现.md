# MySQL · 引擎特性 · InnoDB MVCC 相关实现

**Date:** 2018/11
**Source:** http://mysql.taobao.org/monthly/2018/11/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 11
 ](/monthly/2018/11)

 * 当期文章

 POLARDB · 理论基础 · 敢问路在何方 — 论B+树索引的演进方向（上）
* Database · 原理介绍 · Google Percolator 分布式事务实现原理解读
* Database · 原理介绍 · 关于Paxos 幽灵复现问题
* MySQL · 引擎特性 · InnoDB MVCC 相关实现
* MySQL · RocksDB · 数据的读取(一)
* PgSQL · 最佳实践 · EXPLAIN 使用浅析
* MSSQL · 最佳实践 · 列加密查询性能问题及解决方案
* MySQL · 最佳实践 · 性能问题多维度诊断
* MySQL · 最佳实践 · 8.0 CTE和窗口函数的用法
* PgSQL · 应用案例 · Heap Only Tuple (降低UPDATE引入的索引写IO放大)

 ## MySQL · 引擎特性 · InnoDB MVCC 相关实现 
 Author: mianren 

 InnoDB支持MVCC来提高系统读写并发性能。InnoDB MVCC的实现基于Undo log，通过回滚段来构建需要的版本记录。通过ReadView来判断哪些版本的数据可见。同时Purge线程是通过ReadView来清理旧版本数据。MVCC的相关知识在过去的月报中已有涉及，这里笔者从部分相关实现的角度做一个学习与分享。代码基于MySQL8.0。

之前月报涉及相关知识的有：

[MySQL · 引擎特性 · InnoDB undo log 漫游](http://mysql.taobao.org/monthly/2015/04/01/)

[MySQL · 引擎特性 · InnoDB 事务系统](http://mysql.taobao.org/monthly/2017/12/01/)

[MySQL · 引擎特性 · InnoDB 事务子系统介绍](http://mysql.taobao.org/monthly/2015/12/01/)

## Undo log

Undo log可以用来做事务的回滚操作，保证事务的原子性。同时可以用来构建数据修改之前的版本，支持多版本读。

InnoDB表数据组织方式是主键聚簇索引。二级索引通过索引键值加主键值组合来唯一确定一条记录。聚簇索引和二级索引都包含了DELETED BIT标记位来标识记录是否被删除，真正的删除在事务commit之后且没有读会引用该版本数据的时候。在聚簇索引上还有一些额外信息会存储，6字节的DB_TRX_ID字段，表示最近一次插入或者更新该记录的事务ID。7字节的DB_ROLL_PTR字段，指向该记录的rollback segment的undo log记录。6字节的DB_ROW_ID，当有新数据插入的时候会自动递增。当表上没有用户主键的时候，InnoDB会自动产生聚集索引，包含DB_ROW_ID字段。

对于聚簇索引，更新是在原记录位置更新，通过记录指向undo log的隐藏列来重构早期版本的数据。但对于二级索引，是没有聚簇索引上的这些隐藏列的，因此无法在原记录位置更新。当二级索引更新的时候，需要将原记录标记为删除，再插入新的数据记录。当快照读通过二级索引读取数据发现deleted标识或者更新的时候，如果二级索引页上无法判断可见性，InnoDB会查看聚簇索引上的记录行，通过行上的DB_TRX_ID判断可见性，找到正确的可见版本数据。

当用mvcc读取的时候（row_search_mvcc），对于聚簇索引，当拿到一条记录后，会先通过函数lock_clust_rec_cons_read_sees判断可见性，如果不可见会再构建老版本数据row_vers_build_for_consistent_read。

`dberr_t row_search_mvcc(byte *buf, page_cur_mode_t mode,
 row_prebuilt_t *prebuilt, ulint match_mode,
 ulint direction) {
 ...
 /* This is a non-locking consistent read: if necessary, fetch
 a previous version of the record */

 if (trx->isolation_level == TRX_ISO_READ_UNCOMMITTED) {
 /* Do nothing: we let a non-locking SELECT read the
 latest version of the record */

 } else if (index == clust_index) {
 /* Fetch a previous version of the row if the current
 one is not visible in the snapshot; if we have a very
 high force recovery level set, we try to avoid crashes
 by skipping this lookup */

 if (srv_force_recovery < 5 &&
 !lock_clust_rec_cons_read_sees(rec, index, offsets,
 trx_get_read_view(trx))) {
 rec_t *old_vers;
 /* The following call returns 'offsets' associated with 'old_vers' */
 err = row_sel_build_prev_vers_for_mysql(
 trx->read_view, clust_index, prebuilt, rec, &offsets, &heap,
 &old_vers, need_vrow ? &vrow : NULL, &mtr,
 prebuilt->get_lob_undo());
}
`

对于二级索引，拿到记录会先调用lock_sec_rec_cons_read_sees判断page上记录的最近一次修改trx id是否小于m_up_limit_id，如果小于即该page上数据可见，否则即调用row_search_idx_cond_check检查可见性，对于ICP，索引条件下推的，可以先判断索引条件是否满足条件，这样避免不满足条件的行回表；对于满足条件的行则回表查看可见性。

`dberr_t row_search_mvcc(byte *buf, page_cur_mode_t mode,
 row_prebuilt_t *prebuilt, ulint match_mode,
 ulint direction) {
 ...
 /* This is a non-locking consistent read: if necessary, fetch
 a previous version of the record */

 if (trx->isolation_level == TRX_ISO_READ_UNCOMMITTED) {
 /* Do nothing: we let a non-locking SELECT read the
 latest version of the record */

 } else if (index == clust_index) {
 ...
 } else {
 /* We are looking into a non-clustered index,
 and to get the right version of the record we
 have to look also into the clustered index: this
 is necessary, because we can only get the undo
 information via the clustered index record. */

 ut_ad(!index->is_clustered());

 if (!srv_read_only_mode &&
 !lock_sec_rec_cons_read_sees(rec, index, trx->read_view)) {
 /* We should look at the clustered index.
 However, as this is a non-locking read,
 we can skip the clustered index lookup if
 the condition does not match the secondary
 index entry. */
 switch (row_search_idx_cond_check(buf, prebuilt, rec, offsets)) {
 case ICP_NO_MATCH:
 goto next_rec;
 case ICP_OUT_OF_RANGE:
 err = DB_RECORD_NOT_FOUND;
 goto idx_cond_failed;
 case ICP_MATCH:
 goto requires_clust_rec;
 }
 ...
}

bool lock_sec_rec_cons_read_sees(
 const rec_t *rec, /*!< in: user record which
 should be read or passed over
 by a read cursor */
 const dict_index_t *index, /*!< in: index */
 const ReadView *view) /*!< in: consistent read view */
{
 ...
 trx_id_t max_trx_id = page_get_max_trx_id(page_align(rec));

 ut_ad(max_trx_id > 0);

 return (view->sees(max_trx_id));
}
`

在Undo log中会记录TRX_UNDO_TRX_ID事务ID和TRX_UNDO_TRX_NO事务Commit时的number值。其他的信息可以参考[MySQL · 引擎特性 · InnoDB undo log 漫游](http://mysql.taobao.org/monthly/2015/04/01/)。

当事务为读写事务的时候，事务会获取trx_id。

`/** Allocates a new transaction id.
 @return new, allocated trx id */
UNIV_INLINE
trx_id_t trx_sys_get_new_trx_id() {
 ut_ad(trx_sys_mutex_own());

 /* VERY important: after the database is started, max_trx_id value is
 divisible by TRX_SYS_TRX_ID_WRITE_MARGIN, and the following if
 will evaluate to TRUE when this function is first time called,
 and the value for trx id will be written to disk-based header!
 Thus trx id values will not overlap when the database is
 repeatedly started! */

 if (!(trx_sys->max_trx_id % TRX_SYS_TRX_ID_WRITE_MARGIN)) {
 trx_sys_flush_max_trx_id();
 }

 return (trx_sys->max_trx_id++);
}
`

当事务commit时会获取新的系统trx id作为trx_no。

`trx_commit_low->trx_write_serialisation_history->trx_serialisation_number_get

/** Set the transaction serialisation number.
 @return true if the transaction number was added to the serialisation_list. */
static bool trx_serialisation_number_get(
 trx_t *trx, /*!< in/out: transaction */
 trx_undo_ptr_t *redo_rseg_undo_ptr, /*!< in/out: Set trx
 serialisation number in
 referred undo rseg. */
 trx_undo_ptr_t *temp_rseg_undo_ptr) /*!< in/out: Set trx
 serialisation number in
 referred undo rseg. */
{
 ...
 trx->no = trx_sys_get_new_trx_id();
 ... 
}
`

由于Undo log会保留直到事务提交同时没有其他快照读引用后才会purge。所以需要尽量避免长语句或长事务的执行，避免因此导致的undo堆积或者undo链太长使读取变慢。

## Read View

ReadView主要结构

* m_low_limit_id。 事务ID大于等于该值的数据修改不可见
* m_up_limit_id. 事务ID小于该值的数据修改可见。
* m_creator_trx_id。创建该ReadView的事务，该事务ID的数据修改可见。
* m_ids。当快照创建时的活跃读写事务列表。
* m_low_limit_no。事务number，上一节介绍Undo log时候，事务提交时候获取同时写入Undo log中的值。事务number小于该值的对该ReadView不可见。利用该信息可以Purge不需要的Undo。
* m_closed。 标记该ReadView closed，用于优化减少trx_sys->mutex这把大锁的使用。

 可以看到在view_close的时候如果是在不持有trx_sys->mutex锁的情况下，会仅将ReadView标记为closed，并不会把ReadView从m_views的list中移除。

 `void MVCC::view_close(ReadView *&view, bool own_mutex) {
 uintptr_t p = reinterpret_cast<uintptr_t>(view);

 /* Note: The assumption here is that AC-NL-RO transactions will
 call this function with own_mutex == false. */
 if (!own_mutex) {
 /* Sanitise the pointer first. */
 ReadView *ptr = reinterpret_cast<ReadView *>(p & ~1);

 /* Note this can be called for a read view that
 was already closed. */
 ptr->m_closed = true;

 /* Set the view as closed. */
 view = reinterpret_cast<ReadView *>(p | 0x1);
 } else {
 view = reinterpret_cast<ReadView *>(p & ~1);

 view->close();

 UT_LIST_REMOVE(m_views, view);
 UT_LIST_ADD_LAST(m_free, view);

 ut_ad(validate());

 view = NULL;
 }
}
` 

 当再次调用view_open的时候，如果trx上的read view在产生之后没有新的读写事务发生就可以不用生成新的ReadView，避免加锁添加到m_views中的操作。

 `void MVCC::view_open(ReadView *&view, trx_t *trx) {
 ...
 if (view != NULL) {
 if (trx_is_autocommit_non_locking(trx) && view->empty()) {
 view->m_closed = false;

 if (view->m_low_limit_id == trx_sys_get_max_trx_id()) {
 return;
 } else {
 view->m_closed = true;
 }
 }
 }
 ...
}
` 
 
 m_view_list 用于MVCC链表中前后节点信息存储。

ReadView可见性判断：

* 如果记录trx_id小于m_up_limit_id或者等于m_creator_trx_id，表明ReadView创建的时候该事务已经提交，记录可见。
* 如果记录的trx_id大于等于m_low_limit_id，表明事务是在ReadView创建后开启的，其修改，插入的记录不可见。
* 当trx_id在m_up_limit_id和m_low_limit_id之间的时候，如果id在m_ids数组中，表明ReadView创建时候，事务处于活跃状态，因此记录不可见。

`bool changes_visible(trx_id_t id, const table_name_t &name) const
 MY_ATTRIBUTE((warn_unused_result)) {
 ut_ad(id > 0);

 if (id < m_up_limit_id || id == m_creator_trx_id) {
 return (true);
 }

 check_trx_id_sanity(id, name);

 if (id >= m_low_limit_id) {
 return (false);

 } else if (m_ids.empty()) {
 return (true);
 }

 const ids_t::value_type *p = m_ids.data();

 return (!std::binary_search(p, p + m_ids.size(), id));
}
`

Class MVCC封装了ReadView相关的访问。内部成员变量有 m_free存放释放的read view用来reuse避免重新构造。m_views存放active和closed状态的read view。该类提供的主要函数有

* clone_oldest_view(ReadView *view) 考虑最老的ReadView，用于purge线程清理deleted数据和不需要的旧版本数据。
 `trx_purge(
{
 trx_sys->mvcc->clone_oldest_view(&purge_sys->view);
}
`
* set_view_creator_trx_id(ReadView *view, trx_id_t id); 设置read view的creator trx id。
* size() 处于活跃状态的read view数目
* view_open(ReadView *&view, trx_t *trx); 创建read view。 view属于trx.
* view_close(ReadView *&view, bool own_mutex); close read view。当own_mutext为false的时候，设置view为closed不去从m_views中移除。
* view_release(ReadView *&view); release非活跃事务
* is_view_active(ReadView *view) read view是否活跃

## Semi consistent read

对于RC隔离级别或者设置innodb_locks_unsafe_for_binlog的情况下，当发生表扫描的UPDATE语句，如果数据行上有锁，UPDATE会先查看最近一次提交的数据是否满足条件，利用undo构建最近一次提交的数据。当满足条件再去读最新修改的行，这一次再等锁加锁，避免锁的等待。

`row_search_mvcc()
{
 case DB_LOCK_WAIT:
 /* Lock wait for R-tree should already
 be handled in sel_set_rtr_rec_lock() */
 ut_ad(!dict_index_is_spatial(index));
 /* Never unlock rows that were part of a conflict. */
 std::fill_n(prebuilt->new_rec_lock, row_prebuilt_t::LOCK_COUNT, false);

 if (UNIV_LIKELY(prebuilt->row_read_type !=
 ROW_READ_TRY_SEMI_CONSISTENT) ||
 unique_search || index != clust_index) {
 goto lock_wait_or_error;
 }

 /* The following call returns 'offsets' associated with 'old_vers' */
 row_sel_build_committed_vers_for_mysql(clust_index, prebuilt, rec,
 &offsets, &heap, &old_vers,
 need_vrow ? &vrow : NULL, &mtr);
}
`

这里当查询为unique_search并没有走semi consistent read，即对于’update t set … where pk = xx’的语句不会走semi consistent read。这里原因是bug[#52663](https://bugs.mysql.com/bug.php?id=52663)，在部分代码实现约束仅table scan的执行才可以。

同时Semi consistent read由于采用了非冲突串行化的处理方式，因此只能用在RC隔离级别或者设置innodb_locks_unsafe_for_binlog的情况下使用。

## 总结
InnoDB的多版本并不是直接存储多个版本的数据，而是所有更改操作利用行锁做并发控制，这样对某一行的更新操作是串行化的，然后用Undo log记录串行化的结果。当快照读的时候，利用Undo log重建需要读取版本的数据，从而实现读写并发。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)