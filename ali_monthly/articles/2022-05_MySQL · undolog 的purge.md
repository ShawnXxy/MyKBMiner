# MySQL · undolog 的purge

**Date:** 2022/05
**Source:** http://mysql.taobao.org/monthly/2022/05/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2022 / 05
 ](/monthly/2022/05)

 * 当期文章

 MySQL · 引擎特性 · InnoDB Physiological logging 原理分析
* MySQL · 引擎特性 · InnoDB unique check 的问题
* InnoDB · 引擎特性 · LOB 物理结构
* MySQL · undolog 的purge

 ## MySQL · undolog 的purge 
 Author: 巴彦 

 本文基于MySQL Community 8.0.23 Version

通过之前 [InnoDB之UNDO LOG介绍](http://mysql.taobao.org/monthly/2021/12/02/)这篇文章的介绍，我们已经基本明白了undo log的整个生命周期是怎样的，但是其中对于具体undo log是如何purge的没有进行分析更深入的介绍。

具体的，只介绍到purge undo log record时主要分为两种情况：清理`TRX_UNDO_DEL_MARK_REC`记录或者清理`TRX_UNDO_UPD_EXIST_REC`记录；但是没有介绍如何对这两种记录进行清理。今天这篇文章我们就主要介绍一下innodb是如何对这两种记录进行清理的。

清理undo log record的入口函数是`row_purge_record()`，在这个函数中分别对 undo log record type为`TRX_UNDO_DEL_MARK_REC`和`TRX_UNDO_UPD_EXIST_REC` 完成清理，即可完成对旧索引记录的清理。

那么下面分别介绍一下这两种undo log record是如何进行清理的。

### TRX_UNDO_DEL_MARK_REC

`TRX_UNDO_DEL_MARK_REC`类型的undo log record的入口函数为`row_purge_del_mark()`将所有的聚集索引与二级索引记录都清理掉，其具体流程如下：

`static MY_ATTRIBUTE((warn_unused_result)) bool row_purge_del_mark(
 purge_node_t *node) /*!< in/out: row purge node */
{
 ...

 /** 依次遍历二级索引，挨个删除其记录 */
 while (node->index != NULL) {
 /* 跳过损坏的二级索引 */
 dict_table_skip_corrupt_index(node->index);

 row_purge_skip_uncommitted_virtual_index(node->index);

 if (!node->index) {
 break;
 }

 if (node->index->type != DICT_FTS) {
 /** 构造二级索引的entry */
 dtuple_t *entry = row_build_index_entry_low(node->row, NULL, node->index,
 heap, ROW_BUILD_FOR_PURGE);
 /** 从此二级索引中删除此entry */
 row_purge_remove_sec_if_poss(node, node->index, entry);
 mem_heap_empty(heap);
 }

 node->index = node->index->next();
 }

 ...

 /** 清理聚集索引 */
 return (row_purge_remove_clust_if_poss(node));
}
`

从上面的`row_purge_del_mark()`具体实现不难看出，主要清理逻辑都在`row_purge_remove_sec_if_poss()`与`row_purge_remove_clust_if_poss()`中，所以依次介绍这两个函数的实现

`row_purge_remove_sec_if_poss()`：

`void row_purge_remove_sec_if_poss(purge_node_t *node, /*!< in: row purge node */
 dict_index_t *index, /*!< in: index */
 const dtuple_t *entry) /*!< in: index entry */
{
 ...
 if (!entry) {
 /* 如果在这个索引创建之前就写入了这个undo log，那么就可能由于undo log中
 缺少某些 field 而导致获取不到 entry。 */
 return;
 }

 /** 从二级索引的 leaf page 上删除此 entry */
 if (row_purge_remove_sec_if_poss_leaf(node, index, entry)) {
 return;
 }
retry:
 /** 通过修改索引树来删除 entry */
 success = row_purge_remove_sec_if_poss_tree(node, index, entry);
 /* The delete operation may fail if we have little
 file space left: TODO: easiest to crash the database
 and restart with more file space */

 if (!success && n_tries < BTR_CUR_RETRY_DELETE_N_TIMES) {
 n_tries++;

 os_thread_sleep(BTR_CUR_RETRY_SLEEP_TIME);

 goto retry;
 }

 ut_a(success);
}

/**====================row_purge_remove_sec_if_poss_leaf========================*/
static MY_ATTRIBUTE((warn_unused_result)) bool row_purge_remove_sec_if_poss_leaf(
 purge_node_t *node, /*!< in: row purge node */
 dict_index_t *index, /*!< in: index */
 const dtuple_t *entry) /*!< in: index entry */
{
 ...

 /** 从此二级索引树上查找此 entry 的位置 */
 search_result = row_search_index_entry(index, entry, mode, &pcur, &mtr);

 ...

 switch (search_result) {
 case ROW_FOUND:
 /* 在尝试purge之前此entry满足下列条件其一就可删除：
 1. 从具体索引上回查主键，查不到主键。
 2. 没有更老的版本的数据*/
 if (row_purge_poss_sec(node, index, entry)) {
 btr_cur_t *btr_cur = btr_pcur_get_btr_cur(&pcur);

 ...

 goto func_exit_no_pcur;
 }

 ...

 /** 从此二级索引乐观删除此数据 */
 if (!btr_cur_optimistic_delete(btr_cur, 0, &mtr)) {
 /* The index entry could not be deleted. */
 success = false;
 }
 }
 /* fall through (the index entry is still needed,
 or the deletion succeeded) */
 case ROW_NOT_DELETED_REF:
 /* The index entry is still needed. */
 case ROW_BUFFERED:
 /* The deletion was buffered. */
 case ROW_NOT_FOUND:
 /* The index entry does not exist, nothing to do. */
 btr_pcur_close(&pcur);
 func_exit_no_pcur:
 mtr_commit(&mtr);
 return (success);
 }

 ut_error;
 return (false);
}

/**===========btr_cur_optimistic_delete=============*/
ibool btr_cur_optimistic_delete_func(
 btr_cur_t *cursor, /*!< in: cursor on leaf page, on the record to
 delete; cursor stays valid: if deletion
 succeeds, on function exit it points to the
 successor of the deleted record */
#ifdef UNIV_DEBUG
 ulint flags, /*!< in: BTR_CREATE_FLAG or 0 */
#endif /* UNIV_DEBUG */
 mtr_t *mtr) /*!< in: mtr; if this function returns
 TRUE on a leaf page of a secondary
 index, the mtr must be committed
 before latching any further pages */
{
 ...

 /* This is intended only for leaf page deletions */

 /** 获取 pcursor 所在的 block */
 block = btr_cur_get_block(cursor);

 ...

 /** 获取 record 及其 offset */
 rec = btr_cur_get_rec(cursor);
 offsets =
 rec_get_offsets(rec, cursor->index, offsets, ULINT_UNDEFINED, &heap);

 ...

 /** 如果为 true 说明 record 都在此page中，乐观直接删除即可；
 如果为false，则可能涉及到索引树的变化以及 blob 等数据，需要悲观删除。*/
 if (no_compress_needed) {
 ...

 /** 从 AHI hash 中删除此 record */
 btr_search_update_hash_on_delete(cursor);

 /** 从索引页上删除此 record */
 if (page_zip) {
 ...
 page_cur_delete_rec(btr_cur_get_page_cur(cursor), cursor->index, offsets,
 mtr);
 ...
 } else {
 ...

 page_cur_delete_rec(btr_cur_get_page_cur(cursor), cursor->index, offsets,
 mtr);

 ...
 }
 } else {
 /* 预取siblings block，为悲观删除做准备 */
 btr_cur_prefetch_siblings(block);
 }

 if (UNIV_LIKELY_NULL(heap)) {
 mem_heap_free(heap);
 }

 return (no_compress_needed);
}

/**===========row_purge_remove_sec_if_poss_tree=============*/
static MY_ATTRIBUTE((warn_unused_result)) ibool
 row_purge_remove_sec_if_poss_tree(
 purge_node_t *node, /*!< in: row purge node */
 dict_index_t *index, /*!< in: index */
 const dtuple_t *entry) /*!< in: index entry */
{
 ...

 /** 从此二级索引树上查找此 entry 的位置 */
 search_result = row_search_index_entry(
 index, entry, BTR_MODIFY_TREE | BTR_LATCH_FOR_DELETE, &pcur, &mtr);

 ...

 btr_cur = btr_pcur_get_btr_cur(&pcur);

 /* 在尝试purge之前此entry满足下列条件其一就可删除：
 1. 从具体索引上回查主键，查不到主键。
 2. 没有更老的版本的数据*/
 if (row_purge_poss_sec(node, index, entry)) {
 ...

 goto func_exit;
 }

 /** 悲观删除此 entry。 */
 btr_cur_pessimistic_delete(&err, FALSE, btr_cur, 0, false, 0, node->undo_no,
 node->rec_type, &mtr);
 ...
 }
 }

...

 return (success);
}

/**===========btr_cur_pessimistic_delete=============*/
ibool btr_cur_pessimistic_delete(
 dberr_t *err, /*!< out: DB_SUCCESS or DB_OUT_OF_FILE_SPACE;
 the latter may occur because we may have
 to update node pointers on upper levels,
 and in the case of variable length keys
 these may actually grow in size */
 ibool has_reserved_extents, /*!< in: TRUE if the
 caller has already reserved enough free
 extents so that he knows that the operation
 will succeed */
 btr_cur_t *cursor, /*!< in: cursor on the record to delete;
 if compression does not occur, the cursor
 stays valid: it points to successor of
 deleted record on function exit */
 ulint flags, /*!< in: BTR_CREATE_FLAG or 0 */
 bool rollback, /*!< in: performing rollback? */
 trx_id_t trx_id, /*!< in: the current transaction id. */
 undo_no_t undo_no,
 /*!< in: the undo number within the
 current trx, used for rollback to savepoint
 for an LOB. */
 ulint rec_type,
 /*!< in: undo record type. */
 mtr_t *mtr) /*!< in: mtr */
{
 ...

 /** 获取此 entry 相关信息 */
 block = btr_cur_get_block(cursor);
 page = buf_block_get_frame(block);
 index = btr_cur_get_index(cursor);

 ...
 
 /** 获取 将要删除的 record 及 offset 等信息 */
 rec = btr_cur_get_rec(cursor);
 ...
 offsets = rec_get_offsets(rec, index, NULL, ULINT_UNDEFINED, &heap);

 /** 释放 extern 数据，包括 blob 等。 */
 if (rec_offs_any_extern(offsets)) {
 lob::BtrContext btr_ctx(mtr, NULL, index, rec, offsets, block);

 btr_ctx.free_externally_stored_fields(trx_id, undo_no, rollback, rec_type);
 ...
 }

 /** 如果此 page 中只有这一条 record，并且此 page 不是 root page；
 那么需要将此 page 删除掉*/
 if (UNIV_UNLIKELY(page_get_n_recs(page) < 2) &&
 UNIV_UNLIKELY(dict_index_get_page(index) != block->page.id.page_no())) {
 /* If there is only one record, drop the whole page in
 btr_discard_page, if this is not the root page */

 btr_discard_page(cursor, mtr);

 ret = TRUE;

 goto return_after_reservations;
 }

 ...

 /** 获取此 page 在 B 树中的层高。 */
 level = btr_page_get_level(page, mtr);

 /** 如果层高大于0，并且此 rec 是 page 上最小的 rec
 需要处理下面三种情况：*/
 if (level > 0 &&
 UNIV_UNLIKELY(rec == page_rec_get_next(page_get_infimum_rec(page)))) {
 rec_t *next_rec = page_rec_get_next(rec);

 if (btr_page_get_prev(page, mtr) == FIL_NULL) {
 /** 此page为最左边的page，那么需要更新 B 树的最小rec */
 /* If we delete the leftmost node pointer on a
 non-leaf level, we must mark the new leftmost node
 pointer as the predefined minimum record */
 ...
 btr_set_min_rec_mark(next_rec, mtr);
 } else if (dict_index_is_spatial(index)) {
 /** 如果是R树，则需要更新parent page */
 /* For rtree, if delete the leftmost node pointer,
 we need to update parent page. */
 ...

 rtr_page_get_father_block(NULL, heap, index, block, mtr, NULL,
 &father_cursor);
 offsets = rec_get_offsets(btr_cur_get_rec(&father_cursor), index, NULL,
 ULINT_UNDEFINED, &heap);

 father_rec = btr_cur_get_rec(&father_cursor);
 rtr_read_mbr(rec_get_nth_field(father_rec, offsets, 0, &len),
 &father_mbr);

 upd_ret = rtr_update_mbr_field(&father_cursor, offsets, NULL, page,
 &father_mbr, next_rec, mtr);

 ...
 } else {
 /* Otherwise, if we delete the leftmost node pointer
 on a page, we have to change the parent node pointer
 so that it is equal to the new leftmost node pointer
 on the page */

 btr_node_ptr_delete(index, block, mtr);

 dtuple_t *node_ptr = dict_index_build_node_ptr(
 index, next_rec, block->page.id.page_no(), heap, level);

 btr_insert_on_non_leaf_level(flags, index, level + 1, node_ptr, mtr);

 ut_d(parent_latched = true);
 }
 }

 /** 从 AHI hash 中删除此 rec */
 btr_search_update_hash_on_delete(cursor);

 /** 从此 page 上删除此 rec*/
 page_cur_delete_rec(btr_cur_get_page_cur(cursor), index, offsets, mtr);

return_after_reservations:
 *err = DB_SUCCESS;

 ...

 DBUG_RETURN(ret);
}
`

综上，我们就知道从二级索引上清理需要进行那些操作了。

`row_purge_remove_clust_if_poss()`：

`static MY_ATTRIBUTE((warn_unused_result)) bool row_purge_remove_clust_if_poss(
 purge_node_t *node) /*!< in/out: row purge node */
{
 /** 尝试仅从叶子节点上删除此 rec */
 if (row_purge_remove_clust_if_poss_low(node, BTR_MODIFY_LEAF)) {
 return (true);
 }

 /** 从整个B树中删除此 rec */
 for (ulint n_tries = 0; n_tries < BTR_CUR_RETRY_DELETE_N_TIMES; n_tries++) {
 if (row_purge_remove_clust_if_poss_low(
 node, BTR_MODIFY_TREE | BTR_LATCH_FOR_DELETE)) {
 return (true);
 }

 os_thread_sleep(BTR_CUR_RETRY_SLEEP_TIME);
 }

 return (false);
}

/**===========row_purge_remove_clust_if_poss_low=============*/
static MY_ATTRIBUTE((warn_unused_result)) bool row_purge_remove_clust_if_poss_low(
 purge_node_t *node, /*!< in/out: row purge node */
 ulint mode) /*!< in: BTR_MODIFY_LEAF or BTR_MODIFY_TREE */
{
 ...

 /** 获取聚集索引 */
 index = node->table->first_index();

 ...

 /** 获取此 rec 在B树中的位置，如果获取不到说明此 rec 可能已被删除，直接退出 */
 if (!row_purge_reposition_pcur(mode, node, &mtr)) {
 /* The record was already removed. */
 goto func_exit;
 }

 /** 获取 rec 及 offsets 等信息 */
 rec = btr_pcur_get_rec(&node->pcur);

 offsets = rec_get_offsets(rec, index, offsets_, ULINT_UNDEFINED, &heap);

 /** 校验我们需要 purge 的 rec 中的 roll_ptr 与 B 树中 rec 的 roll_ptr 是否一致。
 如果不一致，说明此 record 又被更新了，故退出。*/
 if (node->roll_ptr != row_get_rec_roll_ptr(rec, index, offsets)) {
 /* Someone else has modified the record later: do not remove */
 goto func_exit;
 }

 ...

 /** 先尝试只从叶子节点上删除此 rec，不行则从整个B树上删除此 rec */
 if (mode == BTR_MODIFY_LEAF) {
 success =
 btr_cur_optimistic_delete(btr_pcur_get_btr_cur(&node->pcur), 0, &mtr);
 } else {
 dberr_t err;
 ut_ad(mode == (BTR_MODIFY_TREE | BTR_LATCH_FOR_DELETE));
 btr_cur_pessimistic_delete(&err, FALSE, btr_pcur_get_btr_cur(&node->pcur),
 0, false, node->trx_id, node->undo_no,
 node->rec_type, &mtr);

 ...
 }
 }

func_exit:
 ...

 return (success);
}
`

因为`btr_cur_optimistic_delete()`与`btr_cur_pessimistic_delete()`在之前介绍`row_purge_remove_sec_if_poss()`删除二级索引是就已经介绍，所以这里就不在赘述。

至此我们已经介绍完清理一条`TRX_UNDO_DEL_MARK_REC`undo log需要进行那些操作。

### TRX_UNDO_UPD_EXIST_REC

`TRX_UNDO_UPD_EXIST_REC`类型的undo log record的入口函数为`row_purge_upd_exist_or_extern()`将旧的二级索引记录清理掉，其具体流程如下：

`static void row_purge_upd_exist_or_extern_func(
#ifdef UNIV_DEBUG
 const que_thr_t *thr, /*!< in: query thread */
#endif /* UNIV_DEBUG */
 purge_node_t *node, /*!< in: row purge node */
 trx_undo_rec_t *undo_rec) /*!< in: record to purge */
{
 ...

 /** 如果 rec_type 为 TRX_UNDO_UPD_DEL_REC，那么说明此 rec 之前被删除，二级索引也被删除。
 如果cmpl_info 为 UPD_NODE_NO_ORD_CHANGE，那么说明二级索引没有被修改。*/
 if (node->rec_type == TRX_UNDO_UPD_DEL_REC ||
 (node->cmpl_info & UPD_NODE_NO_ORD_CHANGE)) {
 goto skip_secondaries;
 }

 ...

 /** 遍历改 rec 的所有二级索引 */
 while (node->index != NULL) {
 /** 跳过可能已经损坏的二级索引 */
 dict_table_skip_corrupt_index(node->index);

 row_purge_skip_uncommitted_virtual_index(node->index);

 if (!node->index) {
 break;
 }

 /** 构造出此 rec 更新之前的 rec */
 if (row_upd_changes_ord_field_binary(node->index, node->update, thr, NULL,
 NULL)) {
 /* Build the older version of the index entry */
 dtuple_t *entry = row_build_index_entry_low(node->row, NULL, node->index,
 heap, ROW_BUILD_FOR_PURGE);
 /** 将 old rec 从二级索引上删除 */
 row_purge_remove_sec_if_poss(node, node->index, entry);
 ...
 }

 node->index = node->index->next();
 }

 mem_heap_free(heap);

skip_secondaries:

 /* Free possible externally stored fields */
 for (ulint i = 0; i < upd_get_n_fields(node->update); i++) {
 const upd_field_t *ufield = upd_get_nth_field(node->update, i);

 if (dfield_is_ext(&ufield->new_val)) {
 buf_block_t *block;
 ...

 /* We use the fact that new_val points to
 undo_rec and get thus the offset of
 dfield data inside the undo record. Then we
 can calculate from node->roll_ptr the file
 address of the new_val data */

 internal_offset =
 ((const byte *)dfield_get_data(&ufield->new_val)) - undo_rec;

 ut_a(internal_offset < UNIV_PAGE_SIZE);

 trx_undo_decode_roll_ptr(node->roll_ptr, &is_insert, &rseg_id, &page_no,
 &offset);

 /* If table is temp then it can't have its undo log
 residing in rollback segment with REDO log enabled. */
 bool is_temp = node->table->is_temporary();

 undo_space_id = trx_rseg_id_to_space_id(rseg_id, is_temp);

 mtr_start(&mtr);

 /* We have to acquire an SX-latch to the clustered
 index tree (exclude other tree changes) */

 index = node->table->first_index();

 mtr_sx_lock(dict_index_get_lock(index), &mtr);

 /* NOTE: we must also acquire an X-latch to the
 root page of the tree. We will need it when we
 free pages from the tree. If the tree is of height 1,
 the tree X-latch does NOT protect the root page,
 because it is also a leaf page. Since we will have a
 latch on an undo log page, we would break the
 latching order if we would only later latch the
 root page of such a tree! */

 btr_root_get(index, &mtr);

 block = buf_page_get(page_id_t(undo_space_id, page_no), univ_page_size,
 RW_X_LATCH, &mtr);

 buf_block_dbg_add_level(block, SYNC_TRX_UNDO_PAGE);

 data_field = buf_block_get_frame(block) + offset + internal_offset;

 ut_a(dfield_get_len(&ufield->new_val) >= BTR_EXTERN_FIELD_REF_SIZE);

 byte *field_ref = data_field + dfield_get_len(&ufield->new_val) -
 BTR_EXTERN_FIELD_REF_SIZE;

 lob::BtrContext btr_ctx(&mtr, NULL, index, NULL, NULL, block);

 lob::DeleteContext ctx(btr_ctx, field_ref, 0, false);

 lob::ref_t lobref(field_ref);

 /** 将 blob 数据清理掉 */
 lob::purge(&ctx, index, node->modifier_trx_id,
 trx_undo_rec_get_undo_no(undo_rec), lobref, node->rec_type,
 ufield);

 mtr_commit(&mtr);
 }
 }
}
`

因为`row_purge_remove_sec_if_poss()`在之前介绍`row_purge_del_mark()`清理`TRX_UNDO_DEL_MARK_REC`类型的undo log时就已经介绍过，所以这里就不在赘述。

### 总结

至此，我们就已经把undo log是如何进行purge的已经全部介绍完；关于blob数据如何清理，后面有机会继续介绍。

### 参考内容

[MySQL 8.0.23’s source code](https://github.com/mysql/mysql-server/tree/mysql-8.0.23)

[MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/)

[InnoDB之UNDO LOG介绍](http://mysql.taobao.org/monthly/2021/12/02)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)