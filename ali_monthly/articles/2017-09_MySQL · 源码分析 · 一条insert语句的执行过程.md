# MySQL · 源码分析 · 一条insert语句的执行过程

**Date:** 2017/09
**Source:** http://mysql.taobao.org/monthly/2017/09/10/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 09
 ](/monthly/2017/09)

 * 当期文章

 POLARDB · 新品介绍 · 深入了解阿里云新一代产品 POLARDB
* HybridDB · 最佳实践 · 阿里云数据库PetaData
* MySQL · 捉虫动态 · show binary logs 灵异事件
* MySQL · myrocks · myrocks之Bloom filter
* MySQL · 特性分析 · 浅谈 MySQL 5.7 XA 事务改进
* MySQL · 特性分析 · 利用gdb跟踪MDL加锁过程
* MySQL · 源码分析 · Innodb 引擎Redo日志存储格式简介
* MSSQL · 应用案例 · 日志表设计优化与实现
* PgSQL · 应用案例 · 海量用户实时定位和圈人-团圆社会公益系统
* MySQL · 源码分析 · 一条insert语句的执行过程

 ## MySQL · 源码分析 · 一条insert语句的执行过程 
 Author: xijia 

 本文只分析了insert语句执行的主路径，和路径上部分关键函数，很多细节没有深入，留给读者继续分析

create table t1(id int);

insert into t1 values(1)

略过建立连接，从 mysql_parse() 开始分析

`void mysql_parse(THD *thd, char *rawbuf, uint length,
 Parser_state *parser_state)
{
 /* ...... */
 
 /* 检查query_cache，如果结果存在于cache中，直接返回 */
 if (query_cache_send_result_to_client(thd, rawbuf, length) <= 0) 
 {
 LEX *lex= thd->lex;
 
 /* 解析语句 */
 bool err= parse_sql(thd, parser_state, NULL);
 
 /* 整理语句格式，记录 general log */
 /* ...... */
 /* 执行语句 */
 error= mysql_execute_command(thd);
 /* 提交或回滚没结束的事务（事务可能在mysql_execute_command中提交，用trx_end_by_hint标记事务是否已经提交） */
 if (!thd->trx_end_by_hint) 
 {
 if (!error && lex->ci_on_success)
 trans_commit(thd);
 
 if (error && lex->rb_on_fail)
 trans_rollback(thd);
 }
`

进入 mysql_execute_command()

` /* */
 /* ...... */
 
 case SQLCOM_INSERT:
 { 
 
 /* 检查权限 */
 if ((res= insert_precheck(thd, all_tables)))
 break;

 /* 执行insert */
 res= mysql_insert(thd, all_tables, lex->field_list, lex->many_values,
 lex->update_list, lex->value_list,
 lex->duplicates, lex->ignore);

 /* 提交或者回滚事务 */
 if (!res)
 {
 trans_commit_stmt(thd);
 trans_commit(thd);
 thd->trx_end_by_hint= TRUE;
 }
 else if (res)
 {
 trans_rollback_stmt(thd);
 trans_rollback(thd);
 thd->trx_end_by_hint= TRUE;
 }

`

进入 mysql_insert()

`bool mysql_insert(THD *thd,TABLE_LIST *table_list,
 List<Item> &fields, /* insert 的字段 */
 List<List_item> &values_list, /* insert 的值 */
 List<Item> &update_fields,
 List<Item> &update_values,
 enum_duplicates duplic,
 bool ignore)
{ 
 /*对每条记录调用 write_record */
 while ((values= its++))
 {
 if (lock_type == TL_WRITE_DELAYED)
 {
 LEX_STRING const st_query = { query, thd->query_length() };
 DEBUG_SYNC(thd, "before_write_delayed");
 /* insert delay */
 error= write_delayed(thd, table, st_query, log_on, &info);
 DEBUG_SYNC(thd, "after_write_delayed");
 query=0;
 }
 else 
 /* normal insert */
 error= write_record(thd, table, &info, &update);
 }
 
 /*
 这里还有
 thd->binlog_query()写binlog
 my_ok()返回ok报文，ok报文中包含影响行数
 */

`

进入 write_record

`/*
 COPY_INFO *info 用来处理唯一键冲突，记录影响行数
 COPY_INFO *update 处理 INSERT ON DUPLICATE KEY UPDATE 相关信息
*/
int write_record(THD *thd, TABLE *table, COPY_INFO *info, COPY_INFO *update)
{
 if (duplicate_handling == DUP_REPLACE || duplicate_handling == DUP_UPDATE)
 {
 /* 处理 INSERT ON DUPLICATE KEY UPDATE 等复杂情况 */
 }
 /* 调用存储引擎的接口 */
 else if ((error=table->file->ha_write_row(table->record[0])))
 {
 DEBUG_SYNC(thd, "write_row_noreplace");
 if (!ignore_errors ||
 table->file->is_fatal_error(error, HA_CHECK_DUP))
 goto err; 
 table->file->restore_auto_increment(prev_insert_id);
 goto ok_or_after_trg_err;
 }
}

`

进入ha_write_row、write_row

`/* handler 是各个存储引擎的基类，这里我们使用InnoDB引擎*/
int handler::ha_write_row(uchar *buf)
{
 /* 指定log_event类型*/
 Log_func *log_func= Write_rows_log_event::binlog_row_logging_function;
 error= write_row(buf);
}

`

进入引擎层，这里是innodb引擎，handler对应ha_innobase
插入的表信息保存在handler中

`int
ha_innobase::write_row(
/*===================*/
 uchar* record) /*!< in: a row in MySQL format */
{
 error = row_insert_for_mysql((byte*) record, prebuilt);
}
`

```
UNIV_INTERN
dberr_t
row_insert_for_mysql( 
/*=================*/
 byte* mysql_rec, /*!< in: row in the MySQL format */
 row_prebuilt_t* prebuilt) /*!< in: prebuilt struct in MySQL
 handle */
{
 /*记录格式从MySQL转换成InnoDB*/
 row_mysql_convert_row_to_innobase(node->row, prebuilt, mysql_rec);
 
 thr->run_node = node;
 thr->prev_node = node;
 
 /*插入记录*/
 row_ins_step(thr);
}

```

```
UNIV_INTERN
que_thr_t*
row_ins_step(
/*=========*/
 que_thr_t* thr) /*!< in: query thread */
{
 /*给表加IX锁*/
 err = lock_table(0, node->table, LOCK_IX, thr);
 
 /*插入记录*/
 err = row_ins(node, thr);
}

```

InnoDB表是基于B+树的索引组织表

如果InnoDB表没有主键和唯一键，需要分配隐含的row_id组织聚集索引

row_id分配逻辑在row_ins中，这里不详细展开

`static __attribute__((nonnull, warn_unused_result))
dberr_t
row_ins(
/*====*/
 ins_node_t* node, /*!< in: row insert node */
 que_thr_t* thr) /*!< in: query thread */
{
 if (node->state == INS_NODE_ALLOC_ROW_ID) {
 /*若innodb表没有主键和唯一键，用row_id组织索引*/
 row_ins_alloc_row_id_step(node);
 
 /*获取row_id的索引*/
 node->index = dict_table_get_first_index(node->table);
 node->entry = UT_LIST_GET_FIRST(node->entry_list);
 }
 
 /*遍历所有索引，向每个索引中插入记录*/
 while (node->index != NULL) {
 if (node->index->type != DICT_FTS) {
 /* 向索引中插入记录 */
 err = row_ins_index_entry_step(node, thr);

 if (err != DB_SUCCESS) {

 return(err);
 }
 } 
 
 /*获取下一个索引*/
 node->index = dict_table_get_next_index(node->index);
 node->entry = UT_LIST_GET_NEXT(tuple_list, node->entry);

 }
 }
}
`
插入单个索引项

`static __attribute__((nonnull, warn_unused_result))
dberr_t
row_ins_index_entry_step( 
/*=====================*/
 ins_node_t* node, /*!< in: row insert node */
 que_thr_t* thr) /*!< in: query thread */
{
 dberr_t err;

 /*给索引项赋值*/
 row_ins_index_entry_set_vals(node->index, node->entry, node->row);

 /*插入索引项*/
 err = row_ins_index_entry(node->index, node->entry, thr);

 return(err);
}
`

```
static
dberr_t
row_ins_index_entry( 
/*================*/
 dict_index_t* index, /*!< in: index */
 dtuple_t* entry, /*!< in/out: index entry to insert */
 que_thr_t* thr) /*!< in: query thread */
{

 if (dict_index_is_clust(index)) {
 /* 插入聚集索引 */
 return(row_ins_clust_index_entry(index, entry, thr, 0));
 } else {
 /* 插入二级索引 */
 return(row_ins_sec_index_entry(index, entry, thr));
 }
}

```

row_ins_clust_index_entry 和 row_ins_sec_index_entry 函数结构类似，只分析插入聚集索引

`UNIV_INTERN
dberr_t
row_ins_clust_index_entry(
/*======================*/
 dict_index_t* index, /*!< in: clustered index */
 dtuple_t* entry, /*!< in/out: index entry to insert */
 que_thr_t* thr, /*!< in: query thread */
 ulint n_ext) /*!< in: number of externally stored columns */
{
 if (UT_LIST_GET_FIRST(index->table->foreign_list)) {
 err = row_ins_check_foreign_constraints(
 index->table, index, entry, thr);
 if (err != DB_SUCCESS) {
 return(err);
 }
 }
 
 /* flush log，make checkpoint（如果需要） */
 log_free_check();

 /* 先尝试乐观插入，修改叶子节点 BTR_MODIFY_LEAF */
 err = row_ins_clust_index_entry_low(
 0, BTR_MODIFY_LEAF, index, n_uniq, entry, n_ext, thr, 
 &page_no, &modify_clock);
 
 if (err != DB_FAIL) {
 DEBUG_SYNC_C("row_ins_clust_index_entry_leaf_after");
 return(err);
 } 
 
 /* flush log，make checkpoint（如果需要） */
 log_free_check();

 /* 乐观插入失败，尝试悲观插入 BTR_MODIFY_TREE */
 return(row_ins_clust_index_entry_low(
 0, BTR_MODIFY_TREE, index, n_uniq, entry, n_ext, thr,
 &page_no, &modify_clock));
`

row_ins_clust_index_entry_low 和 row_ins_sec_index_entry_low 函数结构类似，只分析插入聚集索引

`UNIV_INTERN
dberr_t
row_ins_clust_index_entry_low(
/*==========================*/
 ulint flags, /*!< in: undo logging and locking flags */
 ulint mode, /*!< in: BTR_MODIFY_LEAF or BTR_MODIFY_TREE,
 depending on whether we wish optimistic or
 pessimistic descent down the index tree */
 dict_index_t* index, /*!< in: clustered index */
 ulint n_uniq, /*!< in: 0 or index->n_uniq */
 dtuple_t* entry, /*!< in/out: index entry to insert */
 ulint n_ext, /*!< in: number of externally stored columns */
 que_thr_t* thr, /*!< in: query thread */
 ulint* page_no,/*!< *page_no and *modify_clock are used to decide
 whether to call btr_cur_optimistic_insert() during
 pessimistic descent down the index tree.
 in: If this is optimistic descent, then *page_no
 must be ULINT_UNDEFINED. If it is pessimistic
 descent, *page_no must be the page_no to which an
 optimistic insert was attempted last time
 row_ins_index_entry_low() was called.
 out: If this is the optimistic descent, *page_no is set
 to the page_no to which an optimistic insert was
 attempted. If it is pessimistic descent, this value is
 not changed. */
 ullint* modify_clock) /*!< in/out: *modify_clock == ULLINT_UNDEFINED
 during optimistic descent, and the modify_clock
 value for the page that was used for optimistic
 insert during pessimistic descent */
{
 /* 将cursor移动到索引上待插入的位置 */
 btr_cur_search_to_nth_level(index, 0, entry, PAGE_CUR_LE, mode, 
 &cursor, 0, __FILE__, __LINE__, &mtr);
 
 /*根据不同的flag检查主键冲突*/
 err = row_ins_duplicate_error_in_clust_online(
 n_uniq, entry, &cursor,
 &offsets, &offsets_heap);
 
 err = row_ins_duplicate_error_in_clust(
 flags, &cursor, entry, thr, &mtr);

 /*
 如果要插入的索引项已存在，则把insert操作改为update操作
 索引项已存在，且没有主键冲突，是因为之前的索引项对应的数据被标记为已删除
 本次插入的数据和上次删除的一样，而索引项并未删除，所以变为update操作 
 */
 if (row_ins_must_modify_rec(&cursor)) {
 /* There is already an index entry with a long enough common
 prefix, we must convert the insert into a modify of an
 existing record */
 mem_heap_t* entry_heap = mem_heap_create(1024);
 
 /* 更新数据到存在的索引项 */
 err = row_ins_clust_index_entry_by_modify(
 flags, mode, &cursor, &offsets, &offsets_heap,
 entry_heap, &big_rec, entry, thr, &mtr);
 
 /*如果索引正在online_ddl，先记录insert*/
 if (err == DB_SUCCESS && dict_index_is_online_ddl(index)) {
 row_log_table_insert(rec, index, offsets);
 }

 /*提交mini transaction*/
 mtr_commit(&mtr);
 mem_heap_free(entry_heap);
 } else {
 rec_t* insert_rec;

 if (mode != BTR_MODIFY_TREE) {
 /*进行一次乐观插入*/
 err = btr_cur_optimistic_insert(
 flags, &cursor, &offsets, &offsets_heap,
 entry, &insert_rec, &big_rec,
 n_ext, thr, &mtr);
 } else {
 /*
 如果buffer pool余量不足25%，插入失败，返回DB_LOCK_TABLE_FULL
 处理DB_LOCK_TABLE_FULL错误时，会回滚事务
 防止大事务的锁占满buffer pool(注释里写的)
 */
 if (buf_LRU_buf_pool_running_out()) {

 err = DB_LOCK_TABLE_FULL;
 goto err_exit;
 }

 if (/*太长了，略*/) {
 /*进行一次乐观插入*/
 err = btr_cur_optimistic_insert(
 flags, &cursor,
 &offsets, &offsets_heap,
 entry, &insert_rec, &big_rec,
 n_ext, thr, &mtr);
 } else {
 err = DB_FAIL;
 }

 if (err == DB_FAIL) {
 /*乐观插入失败，进行悲观插入*/
 err = btr_cur_pessimistic_insert(
 flags, &cursor,
 &offsets, &offsets_heap,
 entry, &insert_rec, &big_rec,
 n_ext, thr, &mtr);
 }
 }

}

`
btr_cur_optimistic_insert 和 btr_cur_pessimistic_insert 涉及B+树的操作，内部细节很多，以后再做分析

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)