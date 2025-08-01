# MySQL · 源码分析 · BLOB字段UPDATE流程分析

**Date:** 2021/10
**Source:** http://mysql.taobao.org/monthly/2021/10/03/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2021 / 10
 ](/monthly/2021/10)

 * 当期文章

 MySQL · 引擎特性 · 庖丁解InnoDB之UNDO LOG
* 数据库系统 · 事物并发控制 · Two-phase Lock Protocol
* MySQL · 源码分析 · BLOB字段UPDATE流程分析
* MySQL · 源码分析· 跟着MySQL 8.0 学 C++：scope_guard
* MySQL · 源码分析 · CSV 引擎详解

 ## MySQL · 源码分析 · BLOB字段UPDATE流程分析 
 Author: zheyu 

 ## 准备

MySQL 8.0.25

## 相关背景

在游戏等业务场景中，常常会使用到如BLOB格式的可变长大字段，此类可变长大字段的处理与其余字段格式有所不同。
在处理如VARCHAR、VARBINARY、BLOB、TEXT等可变长度列时，若数据的长度过长，InnoDB不会直接将字段完整容纳在记录所在的B-Tree页上，而是会将过长的变长字段单独存放在溢出页（off-page）中，B-Tree页只会存储列值的部分前缀。

在分析BLOB字段的操作流程前，首先需要了解Innodb的行格式，具体介绍可以参见[MySQL行格式介绍](https://dev.mysql.com/doc/refman/8.0/en/innodb-row-format.html)，目前实际应用中的最常见的是compact（5.6的默认）和dynamic（5.7及以后的默认）这两种。

 行格式
 紧凑格式
 增强型变长列
 大索引前缀
 支持压缩
 支持的表空间

 REDUNDANT
 No
 No
 No
 No
 system, general, file-per-table

 COMPACT
 Yes
 No
 No
 No
 system, general, file-per-table

 DYNAMIC
 Yes
 Yes
 Yes
 No
 system, general, file-per-table

 COMPRESSED
 Yes
 Yes
 Yes
 Yes
 file-per-table, general

这里主要介绍一下Innodb采用不同行格式时对于过长字段的溢出处理（off-page列），当前版本Innodb判断是否将变长列存储在off-page上由页大小和记录长度决定，Innodb会把行中最长的列放到off-page直到数据页能存放下两行数据，可见`page_zip_rec_needs_ext()`函数。各个行格式的溢出列的转化逻辑略有不同，具体转化方法可参考 `dtuple_convert_big_rec()`函数：

* **Redundant** 行格式会把变长字段值的前768字节存在B-Tree的索引记录中，若字段超出长度，则其剩余数据被放在溢出页进行存储。对大于这个768字节的固定长度列，会被编码为可变长度列。对于采用溢出处理的off-page列，列数据末尾会存有指向剩余数据所在页地址的指针（占用20字节）；对于过长的溢出列（长度超过一个页），会以链表链接的形式存储在多个溢出页上。
* **Compact** 行格式在溢出页的处理上和redundant行格式基本一致，也是将超过768字节后的变长列放至溢出页上；但是此外，Compact行格式在行前有变长列表，其中对于off-page列的长度的存储记录为788字节 = 768字节 + 外部指针(reference)长度20字节。
* **Dynamic** 行格式和Compact行格式有类似的行存储格式（有变长列表等），但在其基础上增加了long variable-length columns和large index key prefixes特点，即会将大变长列完全存储在off-page上，聚集索引上只含有20字节的指针。
* **Compressed** 行格式在dynamic行格式基础上增加了压缩特性，对于off-page列的处理模式与Dynamic行格式基本一致。

## BLOB字段UPDATE流程
这里使用的是COMPACT行格式，RC隔离级别，BLOB字段的实际大小都约为10KB（此大小会导致溢出页的使用），表格格式如下：

`CREATE TABLE `blobtest1`(
 `uid` int(10) NOT NULL DEFAULT '0',
 `bin_data` mediumblob NOT NULL,
 `last_save_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
 PRIMARY KEY (`uid`) USING BTREE
) engine = InnoDB;
`

如下给出使用主键对BLOB字段进行UPDATE的代码执行流程：

`...
 ->>ha_innobase::read_range_first
 ...
 ->>row_search_mvcc
 ... (循环搜索至matching的记录)
 ->>sel_set_rec_lock (对record加X锁)
 ...
...
 ->>ha_innobase::update_row
 ... (检查log空间、row update node)
 ->>row_upd_clust_step (启动mtr)
 ->>btr_pcur_restore_position (乐观获取leaf节点x-latch)
 ->>row_upd_clust_rec
 ->>btr_cur_optimistic_update (存在溢出列，乐观更新失败，返回DB_OVERFLOW)
 ->>btr_cur_prefetch_siblings
 ->>mtr->commit，mtr->start(释放获取的mtr资源，重启mtr)
 ->>btr_pcur_restore_position (进入悲观模式，重新定位cursor)
 ->>btr_cur_search_to_nth_level(获取index的sx-latch，从root搜索至目标叶子结点，最终会施加目标节点、父节点、左右节点的x-latch)
 ->>btr_cur_pessimistic_update 
 ->>btr_cur_optimistic_update (同上失败)
 ->>row_rec_to_index_entry (获得index entry，不包括溢出列)
 ->>row_upd_index_replace_new_col_vals_index_pos (更新index entry)
 ->>row_upd_index_replace_new_col_val_func (更新对应列，此时更新的大字段会被完全copy到index entry，老的溢出列会被标记)
 ->>dtuple_convert_big_rec (构建溢出列对象)
 ->>btr_cur_upd_lock_and_undo (进行undo记录)
 ->>mtr_x_lock_space(space, mtr) (在锁定LOB page前锁定file space，避免死锁)
 ->>fsp_reserve_free_extents (预留文件空间)
 ... (锁处理、删除原index记录)
 ->>btr_cur_insert_if_possible (乐观插入)
 |乐观插入失败|->>btr_cur_pessimistic_insert (悲观插入)
 ->>unmark_extern_fields (标记溢出列)
 ->>fil_space_release_free_extents (释放预留空间)
 ->>btr_store_big_rec_extern_fields (存储溢出列)
 ->>InsertContext::check_redolog (重定位cursor、重启mtr，可见index列和off-page列并不是一个mtr)
 |遍历所有溢出列|... (检查是否可以部分更新)
 ->>lob::insert (将溢出列实际插入tablespace)
 ->>first_page_t::alloc (分配首lob页)
 ->>first_page_t::write、设定相关信息
 |有后续lob页|->>data_page_t::alloc、data_page_t::write、设定相关信息
 ->> ... (更新upd_field_t中的溢出列ref)
 ->>mtr->commit (提交mtr，释放index、page锁等资源)
 ->>dtuple_big_rec_free (清理溢出页内存)
 ... (清理工作)
 ->>ha_innobase::read_range_next
...

`

### 源码分析
下面抽取出BLOB UPDATE过程中相关函数的核心部分并做注释，结合上述执行流程帮助理解：

```
static dberr_t row_upd_clust_rec()
{
 // 部分边界分支省略，仅抽象核心路径...

 // 乐观更新失败...

 mtr->start();

 // 重定位cursor，对index及所需leaf节点加锁
 ut_a(btr_pcur_restore_position(BTR_MODIFY_TREE, pcur, mtr));

 // 更新index列
 err = btr_cur_pessimistic_update(
 flags | BTR_NO_LOCKING_FLAG | BTR_KEEP_POS_FLAG, btr_cur, &offsets,
 offsets_heap, heap, &big_rec, node->update, node->cmpl_info, thr, trx_id,
 trx->undo_no, mtr);
 if (big_rec) {
 // 更新off-page列
 err = lob::btr_store_big_rec_extern_fields(
 trx, pcur, node->update, offsets, big_rec, mtr, lob::OPCODE_UPDATE);
 }

 mtr->commit();

// 清理工作...
 return (err);
}

```

```
dberr_t btr_cur_pessimistic_update()
{
 // 部分边界分支省略，仅抽象核心路径...

 // 构建更新后的dtuple_t
 dtuple_t *new_entry = row_rec_to_index_entry(rec, index, *offsets, entry_heap);

 row_upd_index_replace_new_col_vals_index_pos(new_entry, index, update, FALSE, entry_heap);

 // 选定溢出列，会copy老的溢出页指针
 if (page_zip_rec_needs_ext(/*...*/)) {
 big_rec_vec = dtuple_convert_big_rec(index, update, new_entry);
 }

 err = btr_cur_upd_lock_and_undo(flags, cursor, *offsets, update, cmpl_info,
 thr, mtr, &roll_ptr);
 if (err != DB_SUCCESS) {
 goto err_exit;
 }

 if (optim_err == DB_OVERFLOW) {
 // 提前锁定file space，防止死锁
 fil_space_t *space = fil_space_get(index->space);
 mtr_x_lock_space(space, mtr);
 }

 // 标记原溢出页首页为不可部分更新
 lob::mark_not_partially_updatable(trx, index, update, mtr);

 if (optim_err == DB_OVERFLOW) {
 //为索引树的文件段预留足够的空间
 ulint n_extents = cursor->tree_height / 16 + 3;

 if (!fsp_reserve_free_extents(
 &n_reserved, index->space, n_extents,
 flags & BTR_NO_UNDO_LOG_FLAG ? FSP_CLEANING : FSP_NORMAL, mtr)) {
 err = DB_OUT_OF_FILE_SPACE;
 goto err_exit;
 }
 }

 // 更新系统列...

 // 将record锁移至下边界以更新record
 if (!dict_table_is_locking_disabled(index->table)) {
 lock_rec_store_on_page_infimum(block, rec);
 }

 // 删除index上的原纪录
 btr_search_update_hash_on_delete(cursor);
 page_cursor = btr_cur_get_page_cur(cursor);
 page_cur_delete_rec(page_cursor, index, *offsets, mtr);
 page_cur_move_to_prev(page_cursor);

 // 乐观插入
 rec = btr_cur_insert_if_possible(cursor, new_entry, offsets, offsets_heap, mtr);

 if (rec) {
 // 乐观插入成功
 page_cursor->rec = rec;
 // 将锁从下边界移回更新后的record上
 if (!dict_table_is_locking_disabled(index->table)) {
 lock_rec_restore_from_page_infimum(btr_cur_get_block(cursor), rec, block);
 }

 if (!rec_get_deleted_flag(rec, rec_offs_comp(*offsets))) {
 // 更新新index列的溢出列标记
 lob::BtrContext btr_ctx(mtr, pcur, index, rec, *offsets, block);
 btr_ctx.unmark_extern_fields();
 }

 bool adjust = big_rec_vec && (flags & BTR_KEEP_POS_FLAG);

 // 尝试压缩...

 err = DB_SUCCESS;
 goto return_after_reservations;
 } else {
 // 空间不足，插入失败...
 }

 if (big_rec_vec != nullptr && !index->table->is_intrinsic()) {
 // btr_cur_pessimistic_insert()会释放index的sx锁，在此先再次加锁保证持有index锁，以在同一mtr构建和存储big_rec
 ut_ad(mtr_memo_contains_flagged(mtr, dict_index_get_lock(index),
 MTR_MEMO_X_LOCK | MTR_MEMO_SX_LOCK));
 mtr_sx_lock(dict_index_get_lock(index), mtr);
 }

 was_first = page_cur_is_before_first(page_cursor);

 err = btr_cur_pessimistic_insert(
 BTR_NO_UNDO_LOG_FLAG | BTR_NO_LOCKING_FLAG | BTR_KEEP_SYS_FLAG, cursor,
 offsets, offsets_heap, new_entry, &rec, &dummy_big_rec, nullptr, mtr);

 // 更新系统列...

 if (!rec_get_deleted_flag(rec, rec_offs_comp(*offsets))) {
 // 更新新index列的溢出列标记
 buf_block_t *rec_block = btr_cur_get_block(cursor);
 page_zip = buf_block_get_page_zip(rec_block);

 lob::BtrContext btr_ctx(mtr, nullptr, index, rec, *offsets, rec_block);
 btr_ctx.unmark_extern_fields();
 }

 if (!dict_table_is_locking_disabled(index->table)) {
 lock_rec_restore_from_page_infimum(btr_cur_get_block(cursor), rec, block);
 }
 if (!was_first && !dict_table_is_locking_disabled(index->table)) {
 btr_cur_pess_upd_restore_supremum(btr_cur_get_block(cursor), rec, mtr);
 }

return_after_reservations:
 // 清理环境、传递big_rec_vec溢出列对象...
 return err;
}

```

```

dberr_t btr_store_big_rec_extern_fields()
{ /* 将big_rec_vec中的溢出字段存储到表空间并将指针指向对应的record。
 存储这些字段的页面会从索引的叶节点文件段上分配。*/
 
 // 建立context
 BtrContext btr_ctx(btr_mtr, pcur, index, rec, offsets, rec_block, op);
 InsertContext ctx(btr_ctx, big_rec_vec);
 // 设定溢出列的"being modified"位
 Being_modified bm(btr_ctx, big_rec_vec, pcur, offsets, op, btr_mtr);

 // 流程为：store position -> commit mtr -> check log free -> start mtr -> restore position
 // mtr的提交，重启后在btr_pcur_restore_position里走悲观加锁，重新获取index及相应page锁
 ctx.check_redolog(); 
 
 // 显示Uncompressed LOB的路径
 // 
 for (uint i = 0; i < big_rec_vec->n_fields; i++) {
 // 参数传递，记录定位...

 // 根据partially updatable标志及页类型检测是否可以部分更新
 bool can_do_partial_update = false;
 if (op == lob::OPCODE_UPDATE && upd != nullptr &&
 big_rec_vec->fields[i].ext_in_old) {
 can_do_partial_update = blobref.is_lob_partially_updatable(index);
 }

 if (page_zip != nullptr) {
 // 省略压缩格式...
 } else {
 // 非压缩格式溢出列
 bool do_insert = true;

 // 尝试以update方式更新溢出列
 if (op == lob::OPCODE_UPDATE && upd != nullptr &&
 blobref.is_big(rec_block->page.size) && can_do_partial_update) {
 // 尝试部分更新，成功返回，失败则标记以通知purge thread可以清除老溢出列...
 }

 // 使用insert方式更新溢出列
 if (do_insert) {
 // 实际将LOB的溢出列部分插入tablespace，首先会
 error = lob::insert(&ctx, trx, blobref, &big_rec_vec->fields[i], i);

 if (op == lob::OPCODE_UPDATE && upd != nullptr) {
 upd_field_t *uf = upd->get_field_by_field_no(field_no, index);
 if (uf != nullptr) {
 // 更新upd_field_t中溢出列的reference
 dfield_t *new_val = &uf->new_val;
 if (dfield_is_ext(new_val)) {
 byte *field_ref = new_val->blobref();
 blobref.copy(field_ref);
 ref_t::set_being_modified(field_ref, false, nullptr);
 }
 }
 }
 }
 }
 }
 return (error);
// ...
// 其他特殊格式溢出页处理...
}

```

### 注意事项
1. 当更新目标存在off-page列时，Innodb会默认走悲观更新逻辑，会持有index的sx-latch，目标节点、相邻节点的x-latch，大表并发情况下可能产生瓶颈。
2. 由于LOB字段本身较大，redo的产生量较大；过程中check并记录redo，redo刷写性能不高的情况下持续写入大字段可导致redo buffer打满而卡住写入。
3. 考虑性能等问题，记录的index页和LOB页的写入、过大LOB字端本身的写入（每写64KLOB数据mtr提交一次）并不是在同一个mtr commit流程中，将原子性拆分。
4. 目前的机制确保超过约8K的记录才会移至溢出页，由于每次分配LOB页空间会分配一个空白页，因此LOB字段可以存在空间浪费（写放大）的情况。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)