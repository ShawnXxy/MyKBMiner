# MySQL · 源码分析 · btr_cur_search_to_nth_level 函数分析

**Date:** 2021/07
**Source:** http://mysql.taobao.org/monthly/2021/07/02/
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

 ## MySQL · 源码分析 · btr_cur_search_to_nth_level 函数分析 
 Author: 云乾 

 本文基于MySQL Community 8.0.25 Version

## 概述
btr_cur_search_to_nth_level 函数实现innodb中btree cursor的定位，从btree顶层的root到具体的位置，同时在该函数实现了btree的并发访问控制。

btr_cur_search_to_nth_level函数有1000多行的代码，阅读起来不太直观方便，本文对该函数的分析是基于mysql-8.0.25 源码，通过给关键函数模块框架加注释，同时删除一些非核心的代码来进行分析，目的是让阅读本文后，能对该函数有快速直观的了解，方便后续更深入的理解。

## 源码分析
```
void btr_cur_search_to_nth_level(
 dict_index_t *index, /*!< in: index */
 ulint level, /*!< in: the tree level of search */
 const dtuple_t *tuple, /*!< in: data tuple; NOTE: n_fields_cmp in
 tuple must be set so that it cannot get
 compared to the node ptr page number field! */
 page_cur_mode_t mode, /*!< in: PAGE_CUR_L, ...;
 Inserts should always be made using
 PAGE_CUR_LE to search the position! */
 ...)
{
 // 变量定义及一些基本的assertion
 page_t *page = nullptr; /* remove warning */
 buf_block_t *block;
 ...

 // 使用ahi查询，查询成功返回
 if (btr_search_guess_on_hash(index, info, tuple, mode, latch_mode, cursor,
 has_search_latch, mtr)) {
 /* Search using the hash index succeeded */
 return;
 }

 // 给index 加latch
 switch (latch_mode) {
 case BTR_MODIFY_TREE:
 if (...) {
 mtr_x_lock(dict_index_get_lock(index), mtr);
 } else {
 mtr_sx_lock(dict_index_get_lock(index), mtr);
 }
 upper_rw_latch = RW_X_LATCH;
 break;
 case BTR_CONT_MODIFY_TREE:
 case BTR_CONT_SEARCH_TREE:
 if (...) {
 upper_rw_latch = RW_X_LATCH;
 } else {
 upper_rw_latch = RW_NO_LATCH;
 }
 break;
 default:
 if (!srv_read_only_mode) {
 if (...) {
 mtr_s_lock(dict_index_get_lock(index), mtr);
 } else {
 /* BTR_MODIFY_EXTERNAL needs to be excluded */
 mtr_sx_lock(dict_index_get_lock(index), mtr);
 }
 upper_rw_latch = RW_S_LATCH;
 } else {
 upper_rw_latch = RW_NO_LATCH;
 }
 }

 // 初始化root page_id
 const space_id_t space = dict_index_get_space(index);
 const page_size_t page_size(dict_table_page_size(index->table));

 /* Start with the root page. */
 page_id_t page_id(space, dict_index_get_page(index)); // root page id

 // 非叶子节点调整search mode
 switch (mode) {
 case PAGE_CUR_GE:
 page_mode = PAGE_CUR_L;
 break;
 case PAGE_CUR_G:
 page_mode = PAGE_CUR_LE;
 break;
 default:
 page_mode = mode;
 break;
 }

// 循环、逐层的查找，直至达到传入的层数 level，一般是0（即叶子节点）
// 此处的分析忽略Spatial index的部分
// 从Buffer Pool或磁盘中得到索引页
search_loop:
 // 更新rw_latch for page latch mode
 if (height != 0) {
 if ((latch_mode != BTR_MODIFY_TREE || height == level) &&
 !retrying_for_search_prev) {
 rw_latch = RW_SX_LATCH;
 } else {
 rw_latch = upper_rw_latch;
 }
 }
 } else if (latch_mode <= BTR_MODIFY_LEAF) {
 rw_latch = latch_mode;

 // 如果判断应该操作change buffer，则更新fetch mode为从buffer pool中获取
 if (btr_op != BTR_NO_OP &&
 ibuf_should_try(index, btr_op != BTR_INSERT_OP)) {
 /* Try to buffer the operation if the leaf
 page is not in the buffer pool. */

 fetch = btr_op == BTR_DELETE_OP ? Page_fetch::IF_IN_POOL_OR_WATCH
 : Page_fetch::IF_IN_POOL;
 }
 }

// 获取page
retry_page_get:
 block = buf_page_get_gen(page_id, page_size, rw_latch, guess, fetch, file, line, mtr);

 // 当block为null时，尝试change buffer操作
 if (block == nullptr) {
 switch (btr_op) {
 case BTR_INSERT_OP:
 case BTR_INSERT_IGNORE_UNIQUE_OP:
 if (ibuf_insert(IBUF_OP_INSERT, tuple, index, page_id, page_size,
 cursor->thr)) {
 cursor->flag = BTR_CUR_INSERT_TO_IBUF;
 goto func_exit;
 }
 break;

 case BTR_DELMARK_OP:
 if (ibuf_insert(IBUF_OP_DELETE_MARK, tuple, index, page_id, page_size,
 cursor->thr)) {
 cursor->flag = BTR_CUR_DEL_MARK_IBUF;

 goto func_exit;
 }
 break;

 case BTR_DELETE_OP:
 if (!row_purge_poss_sec(cursor->purge_node, index, tuple)) {
 /* The record cannot be purged yet. */
 cursor->flag = BTR_CUR_DELETE_REF;
 } else if (ibuf_insert(IBUF_OP_DELETE, tuple, index, page_id, page_size,
 cursor->thr)) {
 /* The purge was buffered. */
 cursor->flag = BTR_CUR_DELETE_IBUF;
 } else {
 /* The purge could not be buffered. */
 buf_pool_watch_unset(page_id);
 break;
 }
 buf_pool_watch_unset(page_id);
 goto func_exit;

 default:
 ut_error;
 }
 // 操作change buffer没有成功，则更新fetch mode为从disk中获取该page，然后再次获取该page
 fetch = cursor->m_fetch_mode;
 goto retry_page_get;
 }

 // 如果search prev, 读取及latch 左节点
 if (retrying_for_search_prev && height != 0) {
 //获取左节点page_no
 left_page_no = btr_page_get_prev(buf_block_get_frame(block), mtr);

 if (left_page_no != FIL_NULL) {
 get_block = buf_page_get_gen(page_id_t(page_id.space(), left_page_no), page_size,
 rw_latch, nullptr, fetch, file, line, mtr);
 }

 // 重新获取该page，并给该page加latch，因为之前该page是no latch，所以需要重新获取并加latch
 block = buf_page_get_gen(page_id, page_size, rw_latch, nullptr, fetch, file, line, mtr);
 }

 page = buf_block_get_frame(block);

 // 当root page同时为Leaf page，latch不符合预期时，则重新循环获取page
 if (height == ULINT_UNDEFINED && page_is_leaf(page) &&
 rw_latch != RW_NO_LATCH && rw_latch != root_leaf_rw_latch) {
 goto search_loop;
 }

 // root page时，更新信息
 if (UNIV_UNLIKELY(height == ULINT_UNDEFINED)) {
 /* We are in the root node */

 height = btr_page_get_level(page, mtr);
 root_height = height;
 cursor->tree_height = root_height + 1;

 info->root_guess = block;
 }

 // 当达到leaf层时
 if (height == 0) {
 // 给叶子节点加latch
 if (rw_latch == RW_NO_LATCH) {
 latch_leaves = btr_cur_latch_leaves(block, page_id, page_size, latch_mode,
 cursor, mtr);
 }
 // 释放mtr中index latch和和上层blocks
 switch (latch_mode) {
 case BTR_MODIFY_TREE:
 case BTR_CONT_MODIFY_TREE:
 case BTR_CONT_SEARCH_TREE:
 break;
 default:
 // 释放index latch
 if (!s_latch_by_caller && !srv_read_only_mode && !modify_external) {
 mtr_release_s_latch_at_savepoint(mtr, savepoint,
 dict_index_get_lock(index));
 }
 // 释放各层blocks
 for (; n_releases < n_blocks; n_releases++) {
 if (n_releases == 0 && modify_external) {
 /* keep latch of root page */
 continue;
 }
 mtr_release_block_at_savepoint(mtr, tree_savepoints[n_releases],
 tree_blocks[n_releases]);
 }
 }

 page_mode = mode;
 }

 // index是spatial index，进行一些page search mode调整
 if (dict_index_is_spatial(index)) {
 if (page_mode == PAGE_CUR_RTREE_LOCATE && level == height) {
 if (level == 0) {
 page_mode = PAGE_CUR_LE;
 } else {
 page_mode = PAGE_CUR_RTREE_GET_FATHER;
 }
 }

 if (page_mode == PAGE_CUR_RTREE_INSERT) {
 page_mode = (level == height) ? PAGE_CUR_LE : PAGE_CUR_RTREE_INSERT;
 }
 }

 // 在page中找到rec并保存到page_cursor中
 page_cur_search_with_match_bytes(block, index, tuple, page_mode, &up_match,
 &up_bytes, &low_match, &low_bytes,
 page_cursor);

 /* If this is the desired level, leave the loop */

 // 没有达到目的level
 if (level != height) {
 // 往下层推进
 height--;

 /* If the rec is the first or last in the page for
 pessimistic delete intention, it might cause node_ptr insert
 for the upper level. We should change the intention and retry.
 */
 // 可能导致SMO，重置index root page为search page，重新开始循环
 if (latch_mode == BTR_MODIFY_TREE &&
 btr_cur_need_opposite_intention(page, lock_intention, node_ptr)) {
 page_id.reset(space, dict_index_get_page(index));

 goto search_loop;
 }

 // spatial index
 if (dict_index_is_spatial(index)) {
 ...
 }

 /* If the first or the last record of the page
 or the same key value to the first record or last record,
 the another page might be choosen when BTR_CONT_MODIFY_TREE.
 So, the parent page should not released to avoiding deadlock
 with blocking the another search with the same key value. */
 // 判断是否是第一个或者最后一个rec，或者和他们match，设置变量detected_same_key_root
 if (!detected_same_key_root && lock_intention == BTR_INTENTION_BOTH &&
 !dict_index_is_unique(index) && latch_mode == BTR_MODIFY_TREE &&
 (up_match >= rec_offs_n_fields(offsets) - 1 ||
 low_match >= rec_offs_n_fields(offsets) - 1)) {
 // 为第一个或者最后一个
 if (node_ptr == first_rec || page_rec_is_last(node_ptr, page)) {
 detected_same_key_root = true;
 } else {
 // 比较是否和第一个或者最后一个rec match
 detected_same_key_root = true;
 }
 }

 /* If the page might cause modify_tree,
 we should not release the parent page's lock. */
 if (!detected_same_key_root && latch_mode == BTR_MODIFY_TREE &&
 !btr_cur_will_modify_tree(index, page, lock_intention, node_ptr,
 node_ptr_max_size, page_size, mtr) &&
 !rtree_parent_modified) {
 /* we can release upper blocks */
 for (; n_releases < n_blocks; n_releases++) {
 if (n_releases == 0) {
 /* we should not release root page
 to pin to same block. */
 continue;
 }

 /* release unused blocks to unpin */
 mtr_release_block_at_savepoint(mtr, tree_savepoints[n_releases],
 tree_blocks[n_releases]);
 }
 }
 
 // 为BTR_MODIFY_TREE时给root page加 sx latch，给他其他page加x latch
 if (height == level && latch_mode == BTR_MODIFY_TREE) {
 ut_ad(upper_rw_latch == RW_X_LATCH);
 /* we should sx-latch root page, if released already.
 It contains seg_header. */
 if (n_releases > 0) {
 mtr_block_sx_latch_at_savepoint(mtr, tree_savepoints[0],
 tree_blocks[0]);
 }

 /* x-latch the branch blocks not released yet. */
 for (ulint i = n_releases; i <= n_blocks; i++) {
 mtr_block_x_latch_at_savepoint(mtr, tree_savepoints[i], tree_blocks[i]);
 }
 }

 /* We should consider prev_page of parent page, if the node_ptr
 is the leftmost of the page. because BTR_SEARCH_PREV and
 BTR_MODIFY_PREV latches prev_page of the leaf page. */
 if ((latch_mode == BTR_SEARCH_PREV || latch_mode == BTR_MODIFY_PREV) &&
 !retrying_for_search_prev) {
 if (btr_page_get_prev(page, mtr) != FIL_NULL &&
 page_rec_is_first(node_ptr, page)) {
 if (leftmost_from_level == 0) {
 leftmost_from_level = height + 1;
 }
 } else {
 leftmost_from_level = 0;
 }

 if (height == 0 && leftmost_from_level > 0) {
 /* should retry to get also prev_page
 from level==leftmost_from_level. */
 retrying_for_search_prev = true;

 page_id.reset(space, tree_blocks[idx]->page.id.page_no());

 for (ulint i = n_blocks - (leftmost_from_level - 1); i <= n_blocks;
 i++) {
 mtr_release_block_at_savepoint(mtr, tree_savepoints[i],
 tree_blocks[i]);
 }

 n_blocks -= (leftmost_from_level - 1);
 height = leftmost_from_level;

 /* replay up_match, low_match */
 for (ulint i = 0; i < n_blocks; i++) {
 page_cur_search_with_match(tree_blocks[i], index, tuple, page_mode,
 &up_match, &low_match, page_cursor,
 rtr_info);
 }

 goto search_loop;
 }
 }

 //重置page_id为下层子节点page，然后进入下层查找
 page_id.reset(space, btr_node_ptr_get_child_page_no(node_ptr, offsets));

 n_blocks++;

 if (UNIV_UNLIKELY(height == 0 && dict_index_is_ibuf(index))) {
 /* We're doing a search on an ibuf tree and we're one
 level above the leaf page. */

 ut_ad(level == 0);

 fetch = cursor->m_fetch_mode;
 rw_latch = RW_NO_LATCH;
 goto retry_page_get;
 }

 // 循环到下层
 goto search_loop;
 }

 // 找到对应rec位置，更新cursor的low_match up_match字段
 if (level != 0) {
 // assert page not corrupt
 btr_assert_not_corrupted(block, index);

 if (page_mode <= PAGE_CUR_LE) {
 cursor->low_match = low_match;
 cursor->up_match = up_match;
 }
 } else {
 cursor->low_match = low_match;
 cursor->low_bytes = low_bytes;
 cursor->up_match = up_match;
 cursor->up_bytes = up_bytes;

 // 根据查询的结果，更新该index的ahi信息
 if (btr_search_enabled && !index->disable_ahi) {
 btr_search_info_update(index, cursor);
 }
 }

// 函数退出
// 释放相关堆内存等
func_exit:
 if (UNIV_LIKELY_NULL(heap)) {
 mem_heap_free(heap);
 }

 if (retrying_for_search_prev) {
 ut_free(prev_tree_blocks);
 ut_free(prev_tree_savepoints);
 }
 
 ... 
}

```

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)