# MySQL · 引擎特性 · InnoDB隐式锁功能解析

**Date:** 2020/09
**Source:** http://mysql.taobao.org/monthly/2020/09/06/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2020 / 09
 ](/monthly/2020/09)

 * 当期文章

 MySQL · 性能优化 · PageCache优化管理
* MySQL · 分布式系统 · 一致性协议under the hood
* X-Engine · 性能优化 · Parallel WAL Recovery for X-Engine
* MySQL · 源码阅读 · InnoDB伙伴内存分配系统实现分析
* PgSQL · 新特性探索 · 浅谈postgresql分区表实现并发创建索引
* MySQL · 引擎特性 · InnoDB隐式锁功能解析
* MySQL · Optimizer · Optimizer Hints
* Database · 新特性 · 映射队列

 ## MySQL · 引擎特性 · InnoDB隐式锁功能解析 
 Author: 杭枫 

 ## 隐式锁概述
在数据库中，通常使用锁机制来协调多个线程并发访问某一资源。MySQL的锁类型分为表锁和行锁，表示对整张表加锁，主要用在DDL场景中，也可以由用户指定，主要由server层负责管理；而行锁指的是锁定某一行或几行，或者是行与行之间的间隙，行锁由存储引擎管理，例如最常使用的InnoDB。表锁占用系统资源小，实现简单，但锁定粒度大，发生锁冲突概率高，并发度比较低。行锁占用系统资源大，锁定粒度小，发生锁冲突概率低，并发度比较高。

InnoDB将锁分为锁类型和锁模式两类。锁类型包括表锁和行锁，而行锁还细分为记录锁、间隙锁、插入意向锁、Next-Key等更细的子类型。锁模式描述的是加什么锁，例如读锁和写锁, 在源码中的定义如下(基于MySQL 8.0)：

`/* Basic lock modes */
enum lock_mode {
 LOCK_IS = 0, /* intention shared */
 LOCK_IX, /* intention exclusive */
 LOCK_S, /* shared */
 LOCK_X, /* exclusive */
 LOCK_AUTO_INC, /* locks the auto-inc counter of a table in an exclusive mode*/
 ...
};
`
当事务需要加锁的时，如果这个锁不可能发生冲突，InnoDB会跳过加锁环节，这种机制称为隐式锁。隐式锁是InnoDB实现的一种延迟加锁机制，其特点是只有在可能发生冲突时才加锁，从而减少了锁的数量，提高了系统整体性能。另外，隐式锁是针对被修改的B+ Tree记录，因此都是记录类型的锁，不可能是间隙锁或Next-Key类型。

## Insert语句的加锁流程
隐式锁主要用在插入场景中。在Insert语句执行过程中，必须检查两种情况，一种是如果记录之间加有间隙锁，为了避免幻读，此时是不能插入记录的，另一中情况如果Insert的记录和已有记录存在唯一键冲突，此时也不能插入记录。除此之外，insert语句的锁都是隐式锁，但跟踪代码发现，insert时并没有调用lock_rec_add_to_queue函数进行加锁, 其实所谓隐式锁就是在Insert过程中不加锁。

只有在特殊情况下，才会将隐式锁转换为显示锁。这个转换动作并不是加隐式锁的线程自发去做的，而是其他存在行数据冲突的线程去做的。例如事务1插入记录且未提交，此时事务2尝试对该记录加锁，那么事务2必须先判断记录上保存的事务id是否活跃，如果活跃则帮助事务1建立一个锁对象，而事务2自身进入等待事务1的状态，可以参考如下例子：

`1. 创建测试表
root@localhost : (none) 14:24:01> Ceate table t(a int not null, b blob, primary key(a));
Query OK, 1 row affected (0.01 sec)
 
2. 事务1插入数据
root@localhost : mytest 14:24:16> begin;
Query OK, 0 rows affected (0.00 sec)

// 创建隐式锁，不需要创建锁结构，也不需要添加到lock hash table中
root@localhost : mytest 14:24:21> insert into t values (2, repeat('b',7000)); 
Query OK, 1 row affected (0.02 sec)

root@localhost : mytest 14:35:20> select * from performance_schema.data_locks; // 此时只有表锁，没有行锁
+--------+------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+-----------+-------------+-----------+
| ENGINE | ENGINE_LOCK_ID | ENGINE_TRANSACTION_ID | THREAD_ID | EVENT_ID | OBJECT_SCHEMA | OBJECT_NAME | PARTITION_NAME | SUBPARTITION_NAME | INDEX_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE | LOCK_MODE | LOCK_STATUS | LOCK_DATA |
+--------+------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+-----------+-------------+-----------+
| INNODB | 47865673030896:1063:47865663453848 | 1811 | 75 | 1 | mytest | t | NULL | NULL | NULL | 47865663453848 | TABLE | IX | GRANTED | NULL |
+--------+------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+-----------+-------------+-----------+
1 row in set (0.01 sec

3. 事务2插入相同的数据
root@localhost : mytest 14:29:45> begin; 
Query OK, 0 rows affected (0.01 sec)

root@localhost : mytest 14:29:48> insert into t values (2, repeat('b',7000)); // 主键冲突，将事务1的隐式锁转换为显示锁，事务2则创建S锁并等待

root@localhost : mytest 14:36:04> select * from performance_schema.data_locks;
+--------+-------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+---------------+-------------+-----------+
| ENGINE | ENGINE_LOCK_ID | ENGINE_TRANSACTION_ID | THREAD_ID | EVENT_ID | OBJECT_SCHEMA | OBJECT_NAME | PARTITION_NAME | SUBPARTITION_NAME | INDEX_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE | LOCK_MODE | LOCK_STATUS | LOCK_DATA |
+--------+-------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+---------------+-------------+-----------+
| INNODB | 47865673032032:1063:47865663454744 | 1816 | 77 | 1 | mytest | t | NULL | NULL | NULL | 47865663454744 | TABLE | IX | GRANTED | NULL |
| INNODB | 47865673032032:2:4:2:47865661626392 | 1816 | 77 | 1 | mytest | t | NULL | NULL | PRIMARY | 47865661626392 | RECORD | S,REC_NOT_GAP | WAITING | 2 |
| INNODB | 47865673030896:1063:47865663453848 | 1811 | 75 | 1 | mytest | t | NULL | NULL | NULL | 47865663453848 | TABLE | IX | GRANTED | NULL |
| INNODB | 47865673030896:2:4:2:47865661623320 | 1811 | 77 | 1 | mytest | t | NULL | NULL | PRIMARY | 47865661623320 | RECORD | X,REC_NOT_GAP | GRANTED | 2 |
+--------+-------------------------------------+-----------------------+-----------+----------+---------------+-------------+----------------+-------------------+------------+-----------------------+-----------+---------------+-------------+-----------+
4 rows in set (0.01 sec)
 
root@localhost : mytest 14:36:54> select * from performance_schema.data_lock_waits;
+--------+-------------------------------------+----------------------------------+----------------------+---------------------+----------------------------------+-------------------------------------+--------------------------------+--------------------+-------------------+--------------------------------+
| ENGINE | REQUESTING_ENGINE_LOCK_ID | REQUESTING_ENGINE_TRANSACTION_ID | REQUESTING_THREAD_ID | REQUESTING_EVENT_ID | REQUESTING_OBJECT_INSTANCE_BEGIN | BLOCKING_ENGINE_LOCK_ID | BLOCKING_ENGINE_TRANSACTION_ID | BLOCKING_THREAD_ID | BLOCKING_EVENT_ID | BLOCKING_OBJECT_INSTANCE_BEGIN |
+--------+-------------------------------------+----------------------------------+----------------------+---------------------+----------------------------------+-------------------------------------+--------------------------------+--------------------+-------------------+--------------------------------+
| INNODB | 47865673032032:2:4:2:47865661626392 | 1816 | 77 | 1 | 47865661626392 | 47865673030896:2:4:2:47865661623320 | 1811 | 77 | 1 | 47865661623320 |
+--------+-------------------------------------+----------------------------------+----------------------+---------------------+----------------------------------+-------------------------------------+--------------------------------+--------------------+-------------------+--------------------------------+
1 row in set (0.00 sec)
`

## 如何判断隐式锁是否存在
InnoDB的每条记录中都一个隐含的trx_id字段，这个字段存在于聚集索引的B+Tree中。假设只有主键索引，则在进行插入时，行数据的trx_id被设置为当前事务id；假设存在二级索引，则在对二级索引进行插入时，需要更新所在page的max_trx_id。

因此对于主键，只需要通过查看记录隐藏列trx_id是否是活跃事务就可以判断隐式锁是否存在。
对于对于二级索引会相对比较麻烦，先通过二级索引页上的max_trx_id进行过滤，如果无法判断是否活跃则需要通过应用undo日志回溯老版本数据，才能进行准确的判断。

## 隐式锁转换
将记录上的隐式锁转换为显示锁是由函数lock_rec_convert_impl_to_expl完成的，代码如下：

`static void lock_rec_convert_impl_to_expl(const buf_block_t *block,
 const rec_t *rec, dict_index_t *index,
 const ulint *offsets) {
 trx_t *trx;

 ut_ad(!LockMutexOwner::own(LOCK_REC_SHARD, block->page.id));
 ut_ad(page_rec_is_user_rec(rec));
 ut_ad(rec_offs_validate(rec, index, offsets));
 ut_ad(!page_rec_is_comp(rec) == !rec_offs_comp(offsets));

 if (index->is_clustered()) {
 trx_id_t trx_id;
 // 对于主键，获取记录上的DB_TRX_ID系统隐藏列，获取事务ID
 trx_id = lock_clust_rec_some_has_impl(rec, index, offsets);
 // 根据事务 ID，判断当前事务是否为活跃事务，若为活跃事务，则返回此活跃事务对象
 trx = trx_rw_is_active(trx_id, NULL, true);
 } else {
 ut_ad(!dict_index_is_online_ddl(index));
 // 对于二级索引，通过Page的MAX_TRX_ID判断事务是否活跃
 trx = lock_sec_rec_some_has_impl(rec, index, offsets);

 if (trx && !can_trx_be_ignored(trx)) {
 ut_ad(!lock_rec_other_trx_holds_expl(LOCK_S | LOCK_REC_NOT_GAP, trx, rec,
 block));
 }
 }

 if (trx != 0) {
 ulint heap_no = page_rec_get_heap_no(rec);

 ut_ad(trx_is_referenced(trx));

 /* If the transaction is still active and has no
 explicit x-lock set on the record, set one for it.
 trx cannot be committed until the ref count is zero. */
 
 // 如果是活跃事务，则将隐式锁转换为显示锁
 lock_rec_convert_impl_to_expl_for_trx(block, rec, index, offsets, trx,
 heap_no);
 }
}
`

## 主键的隐式锁转换
对于主键，通过lock_clust_rec_some_has_impl函数读取记录上的事务ID，然后再判断该事务是否活跃，判断事务是否提交由函数trx_rw_is_active完成，代码如下：

`UNIV_INLINE
trx_t *trx_rw_is_active(trx_id_t trx_id, /*!< in: trx id of the transaction */
 ibool *corrupt, /*!< in: NULL or pointer to a flag
 that will be set if corrupt */
 bool do_ref_count) /*!< in: if true then increment the
 trx_t::n_ref_count */
{
 trx_t *trx;

 /* Fast checking. If it's smaller than minimal active trx id, just
 return NULL. */
 if (trx_sys->min_active_id.load() > trx_id) {
 return (NULL);
 }

 trx_sys_mutex_enter();

 trx = trx_rw_is_active_low(trx_id, corrupt);

 if (trx != 0) {
 trx = trx_reference(trx, do_ref_count);
 }

 trx_sys_mutex_exit();

 return (trx);
}
`
MySQL早期版本在判断事务活跃并且转换隐式锁的全过程都要持有lock_sys mutex全局锁，目的是防止在此期间事务提交或回滚，但在读写事务并发很高的情况下，这种开销是非常大的。MySQL在5.7版本引入了隐式锁转换的优化：[http://dev.mysql.com/worklog/task/?id=6899](http://dev.mysql.com/worklog/task/?id=6899)，通过在事务对象上增加引用计数，可以在不全程持有lock_sys mutex全局锁的情况下，保证进行隐式锁转换的事务不会提交或回滚。lock_rec_convert_impl_to_expl_for_trx负责将隐式锁转化为显示锁，创建显示锁结构并且加入到lock hash table中。锁模式为LOCK_REC | LOCK_X | LOCK_REC_NOT_GAP，由于隐式锁针对的是被修改的B+树记录，因此不是Gap或Next-Key类型，都是Record类型的锁。

## 二级索引的隐式锁转换
由于二级索引的记录不包含事务ID，如何判断二级索引记录上是否有隐式锁呢？前面提到二级索引页的PAGE_MAX_TRX_ID字段保存了一个最大事务ID，当二级索引页中的任何记录更新后，都会更新PAGE_MAX_TRX_ID的值。因此，我们先可以通过PAGE_MAX_TRX_ID进行判断，如果当前PAGE_MAX_TRX_ID的值小于当前活跃事务的最新ID，说明修改这条记录的事务已经提交，则不存在隐式锁，反之则可能存在隐式锁，需要通过聚集索引进行判断，其判断过程由函数row_vers_impl_x_locked_low完成，关键代码如下：

`trx_t *row_vers_impl_x_locked_low(
 const rec_t *clust_rec, /*!< in: clustered index record */
 dict_index_t *clust_index, /*!< in: the clustered index */
 const rec_t *rec, /*!< in: secondary index record */
 dict_index_t *index, /*!< in: the secondary index */
 const ulint *offsets, /*!< in: rec_get_offsets(rec, index) */
 mtr_t *mtr) /*!< in/out: mini-transaction */
{
 trx_id_t trx_id;
 ibool corrupt;
 ulint comp;
 ulint rec_del;
 const rec_t *version;
 rec_t *prev_version = NULL;
 ulint *clust_offsets;
 mem_heap_t *heap;
 dtuple_t *ientry = NULL;
 mem_heap_t *v_heap = NULL;
 const dtuple_t *cur_vrow = NULL;

 DBUG_ENTER("row_vers_impl_x_locked_low");

 ut_ad(rec_offs_validate(rec, index, offsets));

 heap = mem_heap_create(1024);

 clust_offsets =
 rec_get_offsets(clust_rec, clust_index, NULL, ULINT_UNDEFINED, &heap);
 
 // 获取保存在聚集索引记录上的事务ID
 trx_id = row_get_rec_trx_id(clust_rec, clust_index, clust_offsets);
 corrupt = FALSE;
 
 // 判断事务是否活跃
 trx_t *trx = trx_rw_is_active(trx_id, &corrupt, true);
 
 // 事务已提交，返回0
 if (trx == 0) {
 DBUG_RETURN(0);
 }

 comp = page_rec_is_comp(rec);
 
 // 获取deleted_flag
 rec_del = rec_get_deleted_flag(rec, comp);

 for (version = clust_rec;; version = prev_version) {
 // 通过undo日志获取老版本记录
 trx_undo_prev_version_build(clust_rec, mtr, version, clust_index,
 clust_offsets, heap, &prev_version, NULL,
 dict_index_has_virtual(index) ? &vrow : NULL, 0,
 nullptr);
 
 // 没有之前老版本的记录，即是当前事务插入的记录，则二级索引记录rec含有implicit lock
 if (prev_version == NULL) {
 if (rec_del) {
 trx_release_reference(trx);
 trx = 0;
 }

 break;
 }
 
 // 获取获取lao'ban'b的各个字段的偏移量
 clust_offsets = rec_get_offsets(prev_version, clust_index, NULL,
 ULINT_UNDEFINED, &heap);
 
 // 获取老版本记录的deleted_flag
 vers_del = rec_get_deleted_flag(prev_version, comp);
 
 // 获取老版本记录的事务ID
 prev_trx_id = row_get_rec_trx_id(prev_version, clust_index, clust_offsets);
 
 // 构造老版本tuple
 row = row_build(ROW_COPY_POINTERS, clust_index, prev_version, clust_offsets,
 NULL, NULL, NULL, &ext, heap);
 
 // 构造老版本二级索引tuple
 entry = row_build_index_entry(row, ext, index, heap);

 // 两个版本的二级索引记录相等
 if (0 == cmp_dtuple_rec(entry, rec, index, offsets)) {
 // 两个记录的deleted_flag位不同，则表示某活跃事务删除了记录，因此二级索引记录含有隐式锁
 if (rec_del != vers_del) {
 break;
 }
 
 dtuple_set_types_binary(entry, dtuple_get_n_fields(entry));

 if (0 != cmp_dtuple_rec(entry, rec, index, offsets)) {
 break;
 }

 } else if (!rec_del) {
 // 两个版本的二级索引不相同，且记录rec的deleted_flag为0, 表示某活跃事务
 // 更新了二级索引记录，因此二级索引记录含有隐式锁
 break;
 }

 result_check:
 // 如果两个版本的二级索引记录相等，并且两个记录的deleted_flag位是相同的, 
 // 或者两个版本的二级索引不相同，且记录rec的deleted_flag为1，此时判断trx->id
 // 和prev_trx_id，如果不相等则表示之前的事务已经修改了记录，因此记录上不含有隐式锁。
 // 否则，需要通过再之前的记录版本进行判断。
 if (trx->id != prev_trx_id) {
 /* prev_version was the first version modified by
 the trx_id transaction: no implicit x-lock */

 trx_release_reference(trx);
 trx = 0;
 break;
 }
 }

 DBUG_PRINT("info", ("Implicit lock is held by trx:" TRX_ID_FMT, trx_id));

 if (v_heap != NULL) {
 mem_heap_free(v_heap);
 }

 mem_heap_free(heap);
 DBUG_RETURN(trx);
}
`
二级索引在判断出隐式锁存在后，也是调用lock_rec_convert_impl_to_expl_for_trx函数将隐式锁转化为显示锁，并将其加入到lock hash table中。

## 判重过程
基于隐式锁，如何保证插入数据时主键或唯一二级索引的unique特性呢 ? 对于主键，插入时判重主要调用流程如下：

`|-row_ins_step 插入记录
 |-memset(node->trx_id_buf, 0, DATA_TRX_ID_LEN);
 |-trx_write_trx_id(node->trx_id_buf, trx->id)
 |-lock_table 给表加IX锁
 |-row_ins 插入记录
 |-if (node->state == INS_NODE_ALLOC_ROW_ID)
 |-row_ins_alloc_row_id_step
 |-if (dict_index_is_unique())
 |-return
 |-dict_sys_get_new_row_id 分配一个rowid
 |-mutex_enter(&dict_sys-|-mutex);
 |-if (0 == (id % DICT_HDR_ROW_ID_WRITE_MARGIN))
 |-dict_hdr_flush_row_id()
 |-dict_sys-|-row_id++
 |-PolicyMutex::exit()
 |-dict_sys_write_row_id
 |-node->state = INS_NODE_INSERT_ENTRIES;
 |-while (node->index != NULL)
 |-row_ins_index_entry_step 向索引中插入记录,把 innobase format field 的值赋给对应的index entry field
 |-n_fields = dtuple_get_n_fields(entry); // 包含系统列
 |-dtuple_check_typed 检查要插入的行的每个列的类型有效性
 |-row_ins_index_entry_set_vals 根据该索引以及原记录，将组成索引的列的值组成一个记录
 |-for (i = 0; i < n_fields + num_v; i++)
 |-field = dtuple_get_nth_field(entry, i);
 |-row_field = dtuple_get_nth_field(row, ind_field->col->ind);
 |-dfield_set_data(field, dfield_get_data(row_field), len);
 |-field->data = (void *)data;
 |-dtuple_check_typed 检查组成的记录的有效性
 |-row_ins_index_entry 插入索引项
 |-dict_index_t::is_clustered()
 |-row_ins_clust_index_entry 插入聚集索引
 |-dict_index_is_unique
 |-log_free_check
 |-row_ins_clust_index_entry_low 先尝试乐观插入，修改叶子节点 BTR_MODIFY_LEAF
 |-mtr_t::mtr_t()
 |-mtr_t::start()
 |-初始化mtr的各个状态变量
 |-默认模式为MTR_LOG_ALL，表示记录所有的数据变更
 |-mtr状态设置为ACTIVE状态（MTR_STATE_ACTIVE）
 |-为锁管理对象和日志管理对象初始化内存（mtr_buf_t）,初始化对象链表
 |-btr_pcur_t::open() btr_pcur_open_low
 |-btr_cur_search_to_nth_level 将cursor移动到索引上待插入的位置
 |-取得根页页号
 |-page_cursor = btr_cur_get_page_cur(cursor);
 space = dict_index_get_space(index);
 page_no = dict_index_get_page(index);
 |-buf_page_get_gen 取得本层页面，首次为根页面
 |-mtr_memo_push
 |-page_cur_search_with_match_bytes 在本层页面进行游标定位
 |-btr_cur_get_page 取得本层页面，首次为根页面
 |-page_get_infimum_offset
 |-page_rec_get_next
 |-page_rec_is_supremum
 |-row_ins_must_modify_rec
 |-row_ins_duplicate_error_in_clust // Checks if a unique key violation error would occur at an index entry insert
 |-row_ins_set_shared_rec_lock 对cursor 对应的已有记录加 S 锁（可能会等待）保证记录上的操作，包括：Insert/Update/Delete
 |-lock_clust_rec_read_check_and_lock 判断 cursor 对应的记录上是否存在隐式锁（有活跃事务）, 若存在，则将隐式锁转化为显示锁
 |-lock_rec_convert_impl_to_expl 如果是活跃事务，则将隐式锁转换为显示锁
 |-lock_rec_lock 如果上面的隐式锁转化成功，此处加S锁将会等待，直到活跃事务释放锁。
 |-row_ins_dupl_err_with_rec // S锁加锁完成之后，再次判断最终决定是否存在unique冲突, 1. 判断insert 记录与 cursor 对应的记录取值是否相同, 
 2.二级唯一键值锁引，可以存在多个NULL值, 3.最后判断记录的delete_bit状态，判断记录是否被删除提交
 |-cmp_dtuple_rec_with_match
 |-return !rec_get_deleted_flag();
 |-btr_cur_optimistic_insert // 插入记录
 |-mtr_t::commit() // 提交mtr
`

插入主键时如果出现了重复的行，持有重复行数据的事务并没有提交或者回滚，需要等其事务完成提交或者回滚，如果存在重复行则报错，否则继续插入。在判重过程中，对游标对应的已有记录加S锁，保证记录上的操作(包括Insert/Update/Delete) 已经提交或者回滚, 在真正进行insert操作进行时，会尝试对下一个record加X锁。

当更新修改聚簇索引记录时，将对受影响的二级索引记录加隐式锁，在插入新的二级索引记录之前执行duplicate check, 如果修改二级索引的记录是活跃的，则先将隐式锁转换成显示锁，然后对二级索引记录尝试加S锁，加锁成功后再进行duplicate check。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)