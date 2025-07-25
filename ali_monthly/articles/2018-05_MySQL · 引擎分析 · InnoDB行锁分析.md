# MySQL · 引擎分析 · InnoDB行锁分析

**Date:** 2018/05
**Source:** http://mysql.taobao.org/monthly/2018/05/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 05
 ](/monthly/2018/05)

 * 当期文章

 MySQL · Community · Congratulations on MySQL 8.0 GA
* MySQL · 社区动态 · Online DDL 工具 gh-ost 支持阿里云 RDS
* MySQL · 特性分析 · MySQL 8.0 资源组 (Resource Groups)
* MySQL · 引擎分析 · InnoDB行锁分析
* PgSQL · 特性分析 · 神奇的pg_rewind
* MSSQL · 最佳实践 · 阿里云RDS SQL自动化迁移上云的一种解决方案
* MongoDB · 引擎特性 · journal 与 oplog，究竟谁先写入？
* MySQL · RocksDB · MANIFEST文件介绍
* MySQL · 源码分析 · change master to
* PgSQL · 应用案例 · 阿里云 RDS PostgreSQL 高并发特性 vs 社区版本

 ## MySQL · 引擎分析 · InnoDB行锁分析 
 Author: 勉仁 

 ## 前言
理解InnoDB行锁，分析一条SQL语句会加什么样的行锁，会锁住哪些数据范围对业务SQL设计和分析线上死锁问题都会有很大帮助。对于InnoDB的行锁，已经有多篇月报进行了介绍，这里笔者借鉴前面月报的内容，综合自己的理解，对源码的基础实现做一个介绍（会包含部分表锁介绍），然后结合具体SQL语句分析加锁类型和加锁范围。

## InnoDB锁类型的表示

如在月报[MySQL · 引擎特性 · Innodb 锁子系统浅析](http://mysql.taobao.org/monthly/2017/12/02/)所述，在InnoDB内部用uint32类型数据表示锁的类型, 最低的 4 个 bit 表示 lock_mode, 5-8 bit 表示 lock_type(目前只用了 5 和 6 位，大小为 16 和 32 ，表示 LOCK_TABLE 和 LOCK_REC), 剩下的高位 bit 表示行锁的类型record_lock_type。下面会结合源码介绍lock_mode和record_lock_type。

## lock mode

`/* Basic lock modes */
enum lock_mode {
 LOCK_IS = 0, /* intention shared */
 LOCK_IX, /* intention exclusive */
 LOCK_S, /* shared */
 LOCK_X, /* exclusive */
 LOCK_AUTO_INC, /* locks the auto-inc counter of a table
 in an exclusive mode */
 LOCK_NONE, /* this is used elsewhere to note consistent read */
 LOCK_NUM = LOCK_NONE, /* number of lock modes */
 LOCK_NONE_UNSET = 255
};
`

### LOCK_IS/LOCK_IX

LOCK_IS: 表级锁，意向共享锁。表示将要在表上加共享锁。

LOCK_IX：表级锁，意向排他锁。表示是将要在表上加排他锁。

当对记录加LOCK_S或LOCK_X锁的时候，要确保在表上加了LOCK_IS或LOCK_IX锁。

`lock_rec_lock(
{
 ...
 ut_ad((LOCK_MODE_MASK & mode) != LOCK_S
 || lock_table_has(thr_get_trx(thr), index->table, LOCK_IS));
 ut_ad((LOCK_MODE_MASK & mode) != LOCK_X
 || lock_table_has(thr_get_trx(thr), index->table, LOCK_IX));
 ...
}
`

### LOCK_S

表共享锁、行共享锁

表共享锁用在:

* ALTER语句第一阶段，当ALTER语句不能ONLINE执行的时间添加

` storage/innobase/handler/handler0alter.cc
 prepare_inplace_alter_table_dict(
 {
 if (ctx->online) {
 error = DB_SUCCESS;
 } else {
 error = row_merge_lock_table(
 ctx->prebuilt->trx, ctx->new_table, LOCK_S);

 if (error != DB_SUCCESS) {

 goto error_handling;
 }
 } 
 }
`

* autocommit=0, innodb_table_locks=1时, lock table t read添加

```
 storage/innobase/handler/ha_innodb.cc

 ha_innobase::external_lock(
 {
 /* Starting from 4.1.9, no InnoDB table lock is taken in LOCK
 TABLES if AUTOCOMMIT=1. It does not make much sense to acquire
 an InnoDB table lock if it is released immediately at the end
 of LOCK TABLES, and InnoDB's table locks in that case cause
 VERY easily deadlocks.

 We do not set InnoDB table locks if user has not explicitly
 requested a table lock. Note that thd_in_lock_tables(thd)
 can hold in some cases, e.g., at the start of a stored
 procedure call (SQLCOM_CALL). */
 if (m_prebuilt->select_lock_type != LOCK_NONE) {

 if (thd_sql_command(thd) == SQLCOM_LOCK_TABLES
 && THDVAR(thd, table_locks)
 && thd_test_options(thd, OPTION_NOT_AUTOCOMMIT)
 && thd_in_lock_tables(thd)) {

 dberr_t error = row_lock_table_for_mysql(
 m_prebuilt, NULL, 0);
 }

```

行共享锁应用场景比较复杂，这里结合源码做一个介绍，后面会在各种场景分析会也会涉及。

* 事务读在隔离级别为SERIALIZABLE时会给记录加 LOCK_S 锁。

` mysql_lock_tables->lock_external->handler::ha_external_lock->ha_innobase::external_lock

 storage/innobase/handler/ha_innodb.cc

 ha_innobase::external_lock
 {
 if (trx->isolation_level == TRX_ISO_SERIALIZABLE
 && m_prebuilt->select_lock_type == LOCK_NONE
 && thd_test_options(
 thd, OPTION_NOT_AUTOCOMMIT | OPTION_BEGIN)) {

 /* To get serializable execution, we let InnoDB
 conceptually add 'LOCK IN SHARE MODE' to all SELECTs
 which otherwise would have been consistent reads. An
 exception is consistent reads in the AUTOCOMMIT=1 mode:
 we know that they are read-only transactions, and they
 can be serialized also if performed as consistent
 reads. */

 m_prebuilt->select_lock_type = LOCK_S;
 m_prebuilt->stored_select_lock_type = LOCK_S;
 }
 }

 ha_innobase::general_fetch->row_search_mvcc
 storage/innobase/row/row0sel.cc

 row_search_mvcc
 {
 err = sel_set_rec_lock(pcur,
 rec, index, offsets,
 prebuilt->select_lock_type,
 lock_type, thr, &mtr);
 }
`

* SELECT … IN SHARE MODE

```
 mysql_lock_tables->get_lock_data->ha_innobase::store_lock
 file:storage/innobase/handler/ha_innodb.cc
 function:store_lock
 else if ((lock_type == TL_READ && in_lock_tables)
 || (lock_type == TL_READ_HIGH_PRIORITY && in_lock_tables)
 || lock_type == TL_READ_WITH_SHARED_LOCKS
 || lock_type == TL_READ_NO_INSERT
 || (lock_type != TL_IGNORE
 && sql_command != SQLCOM_SELECT)) {

 /* The OR cases above are in this order:
 1) MySQL is doing LOCK TABLES ... READ LOCAL, or we
 are processing a stored procedure or function, or
 2) (we do not know when TL_READ_HIGH_PRIORITY is used), or
 3) this is a SELECT ... IN SHARE MODE, or
 4) we are doing a complex SQL statement like
 INSERT INTO ... SELECT ... and the logical logging (MySQL
 binlog) requires the use of a locking read, or
 MySQL is doing LOCK TABLES ... READ.
 5) we let InnoDB do locking reads for all SQL statements that
 are not simple SELECTs; note that select_lock_type in this
 case may get strengthened in ::external_lock() to LOCK_X.
 Note that we MUST use a locking read in all data modifying
 SQL statements, because otherwise the execution would not be
 serializable, and also the results from the update could be
 unexpected if an obsolete consistent read view would be
 used. */

 /* Use consistent read for checksum table */

 if (sql_command == SQLCOM_CHECKSUM
 || ((srv_locks_unsafe_for_binlog
 || trx->isolation_level <= TRX_ISO_READ_COMMITTED)
 && trx->isolation_level != TRX_ISO_SERIALIZABLE
 && (lock_type == TL_READ
 || lock_type == TL_READ_NO_INSERT)
 && (sql_command == SQLCOM_INSERT_SELECT
 || sql_command == SQLCOM_REPLACE_SELECT
 || sql_command == SQLCOM_UPDATE
 || sql_command == SQLCOM_CREATE_TABLE))) {

 //这里对于INSERT ... SELECT等语句如果没有UPDATE/IN SHARE MODE不需要当前读,
 //所以不需要LOCK_S锁
 /* If we either have innobase_locks_unsafe_for_binlog
 option set or this session is using READ COMMITTED
 isolation level and isolation level of the transaction
 is not set to serializable and MySQL is doing
 INSERT INTO...SELECT or REPLACE INTO...SELECT
 or UPDATE ... = (SELECT ...) or CREATE ...
 SELECT... without FOR UPDATE or IN SHARE
 MODE in select, then we use consistent read 
 for select. */

 m_prebuilt->select_lock_type = LOCK_NONE;
 m_prebuilt->stored_select_lock_type = LOCK_NONE;
 } else {
 m_prebuilt->select_lock_type = LOCK_S;
 m_prebuilt->stored_select_lock_type = LOCK_S;
 }
 }

```

* 普通insert语句遇到duplicate key。

普通INSERT语句如果没有duplicate key是不用加行锁的，当遇到duplicate key就需要加LOCK_S锁。

` row_ins_duplicate_error_in_clust
 {

 if (flags & BTR_NO_LOCKING_FLAG) {
 /* Do nothing if no-locking is set */
 err = DB_SUCCESS;
 } else if (trx->duplicates) {

 /* If the SQL-query will update or replace
 duplicate key we will take X-lock for
 duplicates ( REPLACE, LOAD DATAFILE REPLACE,
 INSERT ON DUPLICATE KEY UPDATE). */

 err = row_ins_set_exclusive_rec_lock(
 lock_type,
 btr_cur_get_block(cursor),
 rec, cursor->index, offsets, thr); //对于REPLACE、INSERT ON DUPLICATE KEY要更新的语句，加排他锁。
 } else {

 err = row_ins_set_shared_rec_lock(
 lock_type,
 btr_cur_get_block(cursor), rec,
 cursor->index, offsets, thr);
 }
 }
`

* 外键检查到引用行时对引用行添加

```
 row_ins_check_foreign_constraint
 {
 if (rec_get_deleted_flag(rec,
 rec_offs_comp(offsets))) {
 err = row_ins_set_shared_rec_lock(
 lock_type, block,
 rec, check_index, offsets, thr);
 switch (err) {
 case DB_SUCCESS_LOCKED_REC:
 case DB_SUCCESS:
 break;
 default:
 goto end_scan;
 }
 } else {
 /* Found a matching record. Lock only
 a record because we can allow inserts
 into gaps */

 err = row_ins_set_shared_rec_lock(
 LOCK_REC_NOT_GAP, block,
 rec, check_index, offsets, thr);
 }
 }

```

### LOCK_X锁

表排他锁，行排他锁

表排他锁

* ALTER语句最后一个阶段。

 在源码注释中也解释了原因，用来确保没有别的事务在修改表定义的时候持有表锁。因为外键检查和crash recovery过程是仅InnoDB持有锁，所以这里无法仅靠Server层的Meta-Data Lock。

` storage/innobase/handler/handler0alter.cc
 commit_inplace_alter_table()
 {
 for (inplace_alter_handler_ctx** pctx = ctx_array; *pctx; pctx++) {
 ha_innobase_inplace_ctx* ctx
 = static_cast<ha_innobase_inplace_ctx*>(*pctx);
 DBUG_ASSERT(ctx->prebuilt->trx == m_prebuilt->trx);

 /* Exclusively lock the table, to ensure that no other
 transaction is holding locks on the table while we
 change the table definition. The MySQL meta-data lock
 should normally guarantee that no conflicting locks
 exist. However, FOREIGN KEY constraints checks and any
 transactions collected during crash recovery could be
 holding InnoDB locks only, not MySQL locks. */

 error = row_merge_lock_table(
 m_prebuilt->trx, ctx->old_table, LOCK_X);

 if (error != DB_SUCCESS) {
 my_error_innodb(
 error, table_share->table_name.str, 0);
 DBUG_RETURN(true);
 }
 }
 }
`

* autocommit=0, innodb_table_locks=1时, lock table t write语句添加

 这里源码的位置和表级锁LOCK_S添加一致。
* IMPORT/DISCARD TABLESPACE 语句的执行

` storage/innobase/handler/ha_innodb.cc
 discard_or_import_tablespace()
 {
 /* Obtain an exclusive lock on the table. */
 dberr_t err = row_mysql_lock_table(
 m_prebuilt->trx, dict_table, LOCK_X,
 discard ? "setting table lock for DISCARD TABLESPACE"
 : "setting table lock for IMPORT TABLESPACE");
 }
`

行排他锁

* UPDATE/DELETE需要阻止并发对同一行数据进行修改语句的执行

` storage/innobase/handler/ha_innodb.cc
 ha_innobase::external_lock(
 if (lock_type == F_WRLCK) {

 /* If this is a SELECT, then it is in UPDATE TABLE ...
 or SELECT ... FOR UPDATE */
 m_prebuilt->select_lock_type = LOCK_X;
 m_prebuilt->stored_select_lock_type = LOCK_X;
 }
`

### LOCK_AUTO_INC

表级锁，用来用来保护自增列的值，这里不再展开叙述，可以参考之前月报[MySQL · 引擎特性 · InnoDB 事务锁系统简介](http://mysql.taobao.org/monthly/2016/01/01/)。

### lock_mode兼容性

` static const byte lock_compatibility_matrix[5][5] = {
 /** IS IX S X AI */
 /* IS */ { TRUE, TRUE, TRUE, FALSE, TRUE},
 /* IX */ { TRUE, TRUE, FALSE, FALSE, TRUE},
 /* S */ { TRUE, FALSE, TRUE, FALSE, FALSE},
 /* X */ { FALSE, FALSE, FALSE, FALSE, FALSE},
 /* AI */ { TRUE, TRUE, FALSE, FALSE, FALSE}
 };
`

## record_lock_type

* LOCK_WAIT 256 表示正在等待锁
* LOCK_ORDINARY 0 表示next-key lock ，锁住记录本身和记录之前的 gap，区别LOCK_GAP和LOCK_REC_NOT_GAP。
当用RR隔离级别的时候，为了防止当前读语句的幻读使用。

 `storage/innobase/row/row0sel.cc
row_search_mvcc
if (set_also_gap_locks
 && !(srv_locks_unsafe_for_binlog
 || trx->isolation_level <= TRX_ISO_READ_COMMITTED)
 && prebuilt->select_lock_type != LOCK_NONE
 && !dict_index_is_spatial(index)) {

 /* Try to place a lock on the index record */

 /* If innodb_locks_unsafe_for_binlog option is used
 or this session is using a READ COMMITTED isolation
 level we do not lock gaps. Supremum record is really
 a gap and therefore we do not set locks there. */

 offsets = rec_get_offsets(rec, index, offsets,
 ULINT_UNDEFINED, &heap);
 err = sel_set_rec_lock(pcur,
 rec, index, offsets,
 prebuilt->select_lock_type,
 LOCK_ORDINARY, thr, &mtr);
...
/* We are ready to look at a possible new index entry in the result
set: the cursor is now placed on a user record */

if (prebuilt->select_lock_type != LOCK_NONE) {
 /* Try to place a lock on the index record; note that delete
 marked records are a special case in a unique search. If there
 is a non-delete marked record, then it is enough to lock its
 existence with LOCK_REC_NOT_GAP. */

 /* If innodb_locks_unsafe_for_binlog option is used
 or this session is using a READ COMMITED isolation
 level we lock only the record, i.e., next-key locking is
 not used. */

 ulint lock_type;

 if (!set_also_gap_locks
 || srv_locks_unsafe_for_binlog
 || trx->isolation_level <= TRX_ISO_READ_COMMITTED
 || (unique_search && !rec_get_deleted_flag(rec, comp))
 || dict_index_is_spatial(index)) {

 goto no_gap_lock;
 } else {
 lock_type = LOCK_ORDINARY;
 }
 ...
 no_gap_lock:
 lock_type = LOCK_REC_NOT_GAP;
 }
`

从这里源码可以看到当参数innodb_locks_unsafe_for_binlog为ON时，只会对行加锁，不会锁范围。这个时候实际RR隔离级别对于当前读已经退化为RC隔离级别。

* LOCK_GAP 512

只锁住索引记录之间或者第一条索引记录前或者最后一条索引记录之后的范围，并不锁住记录本身。

例如在RR隔离级别下，非唯一索引条件上的等值当前读，会在等值记录上加NEXT-KEY LOCK同时锁住行和前面范围的记录，同时会在后面一个值上加LOCK_GAP锁住下一个值前面的范围。下面的例子就会在索引i_c2上给c2 = 5上NEXT-KEY LOCK(LOCK_ORDINARY|LOCK_X)，同时给c2 = 10加上LOCK_GAP|LOCK_X锁。这里是因为非唯一索引，对同一个值可以多次插入，为确保当前读的可重复读，需要锁住前后的范围，确保不会有相同等值插入。

` create table t1(c1 int primary key, c2 int, c3 int, index i_c2(c2));

 insert into t1 values(1, 2, 3), (2, 5, 7), (3, 10, 9);

 set tx_isolation='repeatable-read';

 select * from t1 where c2 = 5 for update;

`

```
 源码:
 ha_innobase::index_next_same(读下一行)-> ha_innobase::general_fetch->row_search_mvcc

 storage/innobase/row/row0sel.cc
 row_search_mvcc
 /* fputs("Comparing rec and search tuple\n", stderr); */

 if (0 != cmp_dtuple_rec(search_tuple, rec, offsets)) {

 if (set_also_gap_locks
 && !(srv_locks_unsafe_for_binlog
 || trx->isolation_level
 <= TRX_ISO_READ_COMMITTED)
 && prebuilt->select_lock_type != LOCK_NONE
 && !dict_index_is_spatial(index)) {

 /* Try to place a gap lock on the index
 record only if innodb_locks_unsafe_for_binlog
 option is not set or this session is not
 using a READ COMMITTED isolation level. */

 err = sel_set_rec_lock(
 pcur,
 rec, index, offsets,
 prebuilt->select_lock_type, LOCK_GAP,
 thr, &mtr);

```

* LOCK_REC_NOT_GAP 1024

仅锁住记录行，不锁范围。

RC隔离级别下的当前读大多是该方式。相关源码可以见LOCK_ORDINARY源码分析中no_gap_lock跳转。同时在上述例子中，RR隔离级别下，非唯一索引上的等值当前读，也会给主键上对应行加LOCK_X|LOCK_REC_NOT_GAP锁。

* LOCK_INSERT_INTENTION 2048

插入意向锁，当插入索引记录的时候用来判断是否有其他事务的范围锁冲突，如果有就需要等待。

例如上面LOCK_GAP中的例子，如果此时另一个session执行insert into t1 values(11, 9, 0);就会调用lock_rec_insert_check_and_lock函数，用插入意向锁来检查是否需要等待。同时插入意向锁之间并不冲突，在一个GAP锁上可以有多个意向锁等待。

` file:lock0lock.cc
 function:lock_rec_insert_check_and_lock
 /* If another transaction has an explicit lock request which locks
 the gap, waiting or granted, on the successor, the insert has to wait.

 An exception is the case where the lock by the another transaction
 is a gap type lock which it placed to wait for its turn to insert. We
 do not consider that kind of a lock conflicting with our insert. This
 eliminates an unnecessary deadlock which resulted when 2 transactions
 had to wait for their insert. Both had waiting gap type lock requests
 on the successor, which produced an unnecessary deadlock. */

 const ulint type_mode = LOCK_X | LOCK_GAP | LOCK_INSERT_INTENTION;

 const lock_t* wait_for = lock_rec_other_has_conflicting(
 type_mode, block, heap_no, trx);
`

* LOCK_PREDICATE 8192

谓词锁 用于支持GIS索引

* LOCK_PRDT_PAGE 16384

用在page上的谓词锁 用于支持GIS索引

LOCK_PREDICATE LOCK_PRDT_PAGE 用于支持GIS的锁，这里不做更多介绍，感兴趣的可以查看[WL #6968](https://dev.mysql.com/worklog/task/?id=6968)， [#WL 6609](https://dev.mysql.com/worklog/task/?id=6609)， [#WL 6745](https://dev.mysql.com/worklog/task/?id=6745)。

## 事务隔离级别与行锁

快照读和当前读。

快照读使用MVCC读取数据记录某一个版本数据，不需要加锁。当前读读取最新数据，需要对记录或者某一个查询范围加锁。

InnoDB支持的隔离级别有：

* Read Uncommited

 可以读未提交记录
* Read Committed (RC)

 读取已提交数据。会存在幻读。
* Repeatable Read (RR)

 可重复读。当前读的时候，部分语句会加范围锁，保证当前读的可重复。
* Serializable

 可串行化。不存在快照读，所有读操作都会加锁。

对于普通的插入INSERT语句，在没有冲突key情况下，InnoDB并不会对记录上锁，所以这里不再分析简单插入的情况，只分析当前读需要加锁的语句。

分析使用的表Schema和数据如下：

`create table t(c1 int primary key, c2 int, c3 int, c4 int, unique index i_c2(c2), index i_c3(c3));

insert into t values (10, 11, 12, 13), (20, 21, 22, 23), (30, 31, 32, 33), (40, 41, 42, 43);
`

## Read-Uncommitted/RC级别加锁分析

### 查询条件为主键等值

* SELECT … WHERE PK = XX FOR UPDATE;

select * from t where c1 = 20 for update;

只需要在c1 = 20的主键记录上加X锁即可，加锁为LOCK_X|LOCK_REC_NOT_GAP。

select * from t where c1 = 15 for update;

没有满足记录的行，不加锁。在Read-Uncommitted/RC级别下，对于主键等值查询没有符合条件的查询并不加锁，后面其他语句情况一样，后面不再分析没有满足记录行情况。

* SELECT … WHERE PK = XX LOCK IN SHARE MODE;

select * from t where c1 = 20 lock in share mode;

只需要在c1 = 20的主键记录上加S锁即可，加锁为LOCK_S|LOCK_REC_NOT_GAP。

* UPDATE … WHERE PK = XX;

 未更新索引列。

 update t set c4 = 12 where c1 = 20;

 只需要在c1 = 20的主键记录上加X锁即可，加锁为LOCK_X|LOCK_REC_NOT_GAP。

 更新包括索引列。

 update t set c2 = 12 where c1 = 20;

 除了主键记录加X锁，还需要在c2的索引上加LOCK_X|LOCK_REC_NOT_GAP，同理c3索引列。
* DELETE … WHERE PK = XX;

 delete from t where c1 = 20;

 对主键、各个索引对应的记录都要加X锁，LOCK_X|LOCK_REC_NOT_GAP。

### 查询条件为主键范围

对满足条件的行依次加上述等值分析中需要的锁。

例如：

`select * from t where c1 >= 20 for update;

会分别对c1 in (20, 30, 40)加X锁（LOCK\_X\|LOCK\_REC\_NOT\_GAP）。

select * from t where c1 <=20 for update;

会分别对c1 in (10,20,30)加X锁，然后server层判断c1=30不满足条件随机释放锁。

update t set c2 = c2 + 1 where c1 >= 20;

会分别对c1 in (20, 30, 40)依次对主键行加X锁，对应的索引行做加X锁操作。

select * from t where c1 <= 20 for update;

会对c1 in (10,20,30)主键行依次加X锁，但c1=30不满足条件，即再释放c1=30上的锁。

update t set c2 = c2 + 1 where c1 <= 20;

会对c1 in (10,20）主键行和对应索引行依次加X锁，对c1=30加X锁判断不符合条件，随即释放c1=30上的锁。
`

Server层对于非条件下推(Index Condition Pushdown-ICP)的场景扫描到不符合条件的行后即释放锁，InnoDB在RC隔离级别下会真正的释放锁，但在RR级别下为了防止其当前读不可重复读的情况，扫描路径上的加锁并不会释放。ICP的相关知识可以查看月报[MySQL · 特性分析 · Index Condition Pushdown (ICP)](http://mysql.taobao.org/monthly/2015/12/08/)

`file:sql/sql_executor.cc
function: int handler::read_range_next()
result= ha_index_next(table->record[0]);
if (result)
 DBUG_RETURN(result);

if (compare_key(end_range) <= 0)
{
 DBUG_RETURN(0);
}
else
{
 /*
 The last read row does not fall in the range. So request
 storage engine to release row lock if possible.
 */
 unlock_row();
 DBUG_RETURN(HA_ERR_END_OF_FILE);
}

function: int handler::compare_key(key_range *range)
{
 int cmp;
 if (!range || in_range_check_pushed_down)
 return 0; // No max range
 cmp= key_cmp(range_key_part, range->key, range->length);
 if (!cmp)
 cmp= key_compare_result_on_equal;
 return cmp;
}
`

### 查询条件为唯一索引等值

* SELECT … WHERE UK = XX FOR UPDATE;

 select * from t where c2 = 21 for update;

 需要在c2 = 21的索引记录上加X锁:LOCK_X|LOCK_REC_NOT_GAP，同时还要在对应主键行上加X锁。

 select * from t where c2 = 16 for update;

 这里没有满足条件的行，不加锁。其他唯一索引等值，没有满足条件行情况下也是没有加锁，后面不再叙述。
* SELECT … WHERE UK = XX LOCK IN SHARE MODE;

 select * from t where c2 = 21 lock in share mode;

 需要在c2 = 21的索引记录上加S锁:LOCK_S|LOCK_REC_NOT_GAP，同时还要在对应主键行上加S锁。
* UPDATE … WHERE UK = XX;

 未更新其他索引列

 update t set c4 = 12 where c2 = 21;

 对唯一索引上数据加X锁（LOCK_X|LOCK_REC_NOT_GAP），然后对应的主键行也需要加X锁。

 更新其他索引列

 update t set c3 = 12 where c2 = 21;

 依次对唯一索引数据、主键行、索引数据加X锁。
* DELETE … WHERE UK = XX;

 delete from t where c2 = 21;

 会对唯一索引数据加X锁，根据唯一索引找到主键行后，会再依次对主键行、唯一索引、索引数据加X锁。

### 查询条件为唯一索引范围

* SELECT … WHERE UK >= XX FOR UPDATE;

 select * from t where c2 >= 21 for update;

 这条语句执行的时候会对主键行c1 in (10, 20, 30, 40)依次加X锁， 同时在对c1=10加锁后，分析发现不满足条件会立即释放该行上的X锁。

 **Note:这里为什么没有对唯一索引加锁？**上面的语句优化器判断走主键更优，就走了主键，只对主键加对应X锁。后面还会对选择不同路径的加锁区别做叙述。对于当前读的不同条件的查询，本质上我们都是在分析不同查询路径时加锁的不同。同时对于Read Uncommited和Read Committed隔离级别，会对不满足条件的行立即释放锁。

 如果我们改为select * from t force index(i_c2) where c2 >= 21 for update;强制走唯一索引就会发现，引擎依次对满足条件的唯一索引、主键记录加X锁。
* SELECT … WHERE UK <= XX FOR UPDATE;

 select * from t force index(i_c2) where c2 <= 21 for update;

 这里会对唯一索引i_c2上c2 in (11, 21)和对应主键行依次加X锁，对于i_c2上c2=31加X锁，且并不会释放。这时候如果另一个session走索引i_c2对c2=31加锁，会发现需要等锁。这里由于索引条件（Index Condition Pushdown-ICP)下推到引擎层，引擎层即判断唯一索引上c2=31不满足条件，即在ha_index_next中返回HA_ERR_END_OF_FILE, c2=31上的X锁，server层并不会去释放。对于该场景，in_range_check_pushed_down为true，在Server层comare_key时并不会去比较range。
* UPDATE … WHERE UK <= XX FOR UPDATE;

 update t force index(i_c2) set c4 = 1 where c2 <= 21;

 这里会对唯一索引i_c2 c2 in (11, 21, 31)和对应主键行依次加X锁，然后判断c2=31并不满足range条件，随机释放c2=31唯一索引和对应主键行上的X锁。

 ICP只用在SELECT语句执行中，所以这里是server层判断是否满足range条件，然后去释放最后不满足range条件的行。

 update t force index(i_c2) set c3 = 1 where c2 <= 21;

 会对满足range条件的唯一索引i_c2 c2 in (11, 21)和对应主键行、对应非唯一索引i_c3加X锁；对c2=31对应唯一索引和主键行加X锁，判断不符合条件后释放锁。

其他语句形式分析同上，这里不再赘述。

### 查询条件为非唯一索引

实际这里通过上面的分析，你也一定已经知道在<= RC隔离级别下非唯一索引的加锁情况。

* SELECT … WHERE IDX = XX FOR UPDATE;

 select * from t where c3 = 22 for update;

 对c3 = 22的索引行加X锁，对主键行加X锁。

 上面分析过不同路径对加锁的影响，如果这里执行select * from t force index (primary) where c3 = 22 for update;会是什么样的加锁、释放锁顺序呢？

实际非唯一索引情况与前面唯一索引情况加锁情况一致，这里不再展开叙述。

### 查询条件上无索引

先分析如下例子：

* SELECT … WHERE COL = XX FOR UPDATE;

 select * from t where c4 = 23 for update;

 会依次对c1 in (10, 20, 30, 40)依次加X锁，分析是否满足条件，不满足即释放。为c1 = 10行加锁，不满足条件释放锁；c1=20加锁，满足条件，保留锁；c1=30加锁，不满足条件，释放；c1=40行加锁，不满足条件，释放。

其他语句情况类似，由于路径选择主键，会依次对主键行加锁，分析条件，不满足条件释放锁，满足条件持有锁，不再赘述。

### 多条件查询

当存在多个条件的时候，除了主键行上的锁，其他的加锁情况取决于选择的路径。如下例子：

* select * from t where c2 = 21 and c3 = 22 for update;

 这里选择了走唯一索引，就会对满足条件的唯一索引行加X锁，然后对主键行加X锁。
* select * from t where c2 = 21 or c3 = 22 for update;

 选择主键路径，就会对所有行一次加X锁，分析条件，最终持有主键上c1 = 20的X锁。

## Repeatable-Read隔离级别加锁分析

### 查询条件为主键等值

* SELECT … WHERE PK = XX FOR UPDATE;

 select * from t where c1 = 20 for update;

 由于主键具有唯一性，等值查询这里加锁与RC级别一致，对c1=20加X锁(LOCK_X|LOCK_REC_NOT_GAP)。

 select * from t where c1 = 15 for update;

 对于没有满足条件的行情况，会对后面的c1=20加GAP锁，（LOCK_X|LOCK_GAP），防止有其他语句插入c1=15的行。

 update t set c4 = 12 where c1 = 15;

 没有满足条件的行不加锁。

其他情况也与RC一致。

### 查询条件为主键范围

* SELECT … WHERE PK >= XX FOR UPDATE;

 select * from t where c1 >= 20 for update;

 这里会对c1=20加X锁(LOCK_X|LOCK_REC_NOT_GAP)，对c1=30, c1=40对应的行加exclusive next-key lock(LOCK_X|LOCK_ORDINARY)，同时会对表示记录上界的’supremum’加exclusive next-key lock。这样做到阻塞其他事务对c1>=20的加锁操作。
* SELECT … WHERE PK >= XX LOCK IN SHARE MODE;

 select * from t where c1 >= 20 LOCK IN SHARE MODE;

 这里会对c1=20加S锁(LOCK_S|LOCK_REC_NOT_GAP)，对c1=30, c1=40对应的行加share next-key lock(LOCK_S|LOCK_ORDINARY)，同时会对表示记录上界的’supremum’加share next-key lock。
* SELECT … WHERE PK <= XX FOR UPDATE;

 select * from t where c1 <= 20 for update;

 这里会对c1 in(10,20,30)依次加exclusive next-key lock(LOCK_X|LOCK_ORDINARY)。且在判断c1=30不符合查询条件后，虽然server层调用unlock_row，但对于RC隔离级别以上且没有设置innodb_locks_unsafe_for_binlog那么并不会释放锁。

 `file: ha_innodb.cc
function: ha_innobase::unlock_row(void)
switch (m_prebuilt->row_read_type) {
 case ROW_READ_WITH_LOCKS:
 if (!srv_locks_unsafe_for_binlog
 && m_prebuilt->trx->isolation_level
 > TRX_ISO_READ_COMMITTED) {
 break;
 }
`
* UPDATE … WHERE PK >= XX;

 未更新其他索引列。

 update t set c4 = 1 where c1 >= 20;

 加锁与上面SELECT… WHERE PK >= XX FOR UPDATE;一致。

 更新包含索引列。

 update t set c2 = c2 + 1 where c1 >= 20;

 对主键c1=20加X锁，i_c2索引行加X锁，然后对c1=30,c1=40的主键行加exclusive next-key lock(LOCK_X|LOCK_ORDINARY)，同时对应的i_c2索引行加X锁，最后对表示记录上界的’supremum’加exclusive next-key lock。
* UPDATE … WHERE PK <= XX;

 update t set c4 = 1 where c1 <= 20;

 加锁与SELECT… WHERE PK <= XX FOR UPDATE;一致

 包含索引列。

 update t set c2 = c2 + 1 where c1 <= 20;

 对主键c1 in(10,20)加exclusive next-key lock(LOCK_X|LOCK_ORDINARY)，同时对应的i_c2索引行加X锁。然后对c1=30加加exclusive next-key lock，因不满足条件，因此server层查询停止。同样并不会释放c1=30上的锁。
* DELETE … WHERE PK >= XX;

 会对c1=20加X锁，对c1=20对应的i_c2索引，i_c3索引加X锁，然后依次对c1=30, c1=40加exclusive next-key lock(LOCK_X|LOCK_ORDINARY)，同时i_c2和i_c3对应的索引行加X锁，最后对’supremum’加LOCK_X|LOCK_ORDINARY。

### 查询条件为唯一索引等值

由于唯一索引中非NULL值具有唯一性，所以这里的加锁和RC会一致。但由于唯一索引可以有多个null值，对于col is null的条件加锁是不一样的。

* SELECT … WHERE UK = XX FOR UPDATE;

 select * from t where c2 = 21 for update;

 这里与RR下主键等值加锁一致，对c2=21的值加X锁，对应主键行加X锁。
* SELECT … WHERE UK IS NULL FOR UPDATE;

 select * from t where c2 is null for update;

 这里由于c2上没有为null值的record，所以这里对c2=11的record上加GAP LOCK(LOCK_X|LOCK_GAP)。

其他等值语句的执行与唯一索引等值在RC下一致。

如果再在table t中插入(50, null, 52, 53);为NULL的值，那么update t set c4 = 1 where c2 is null会对c2为NULL的行加NEXT-KEY LOCK(LOCK_X|LOCK_ORDINARY)，对应主键加X锁，并在c2=11上加GAP LOCK(LOCK_X|LOCK_GAP)。实际上唯一索引is null的加锁和非唯一索引等值加锁类似，后面会对非唯一索引情况做进一步描述。

### 查询条件为唯一索引范围

* SELECT … WHERE UK >= XX FOR UPDATE;

 select * from t where c2 >= 21 for update;

 对于该语句执行，默认会选择主键路径，对c1 in (10, 20, 30, 40)分别加exclusive next-key lock(LOCK_X|LOCK_ORDINARY)，同时对上界’supremum’加exclusive next-key lock，锁住全部数据范围。

 select * from t force index(i_c2) where c2 >= 21 for update;

 如果指定走i_c2索引，那么会对c2 in (21, 31, 41)分别加exclusive next-key lock，对应主键行加X锁，同时对i_c2上’supremum’ record加exclusive next-key lock。
* SELECT … WHERE UN <= XX FOR UPDATE;

 select * from t force index(i_c2) where c2 <= 21 for update;

 这里会对i_c2索引上c2 in (11,21)加exclusive next-key lock，对对应的主键行加X锁，然后对c2=31加exclusive next-key lock，且并不会去释放。
* UPDATE … WHERE UK >= XX;

 未包含索引列。

 update t force index (i_c2) set c4 = 1 where c2 >= 21;

 等同上面指定走唯一索引的SELECT…FOR UPDATE语句加锁。

 包含索引列。

 update t force index (i_c2) set c3 = 1 where c2 >= 21;

 除了上述语句的加锁外，还会对c1 in (20, 30, 40)对应索引i_c3上的行加X锁。
* UPDATE … WHERE UN <= XX;

 未包含索引列。

 update t force index(i_c2) set c4 = 1 where c2 <= 21;

 这里会对i_c2索引上c2 in (11,21,31)加exclusive next-key lock，对对应的主键行加X锁。因为没有ICP，这里c2=31对应的索引行和主键行也会加X锁，同时不会释放。

 包含索引列

 这里会对i_c2索引上c2 in (11,21)加exclusive next-key lock，对对应的主键行和索引i_c3加X锁。对c2=31加exclusive next-key lock，对应主键行加X锁，因不符合range条件，对i_c3不做操作不会加锁。
* DELETE … WHERE UK >= XX;

 delete from t where c2 >= 41;

 上述语句选择了i_c2索引，会对c2 = 41加exclusive next-key lock，对应主键行加X锁，i_c2，i_c3上数据行进行加X锁操作，对i_c2上’supremum’ record加exclusive next-key lock。

### 查询条件为非唯一索引等值

* SELECT … WHERE INDEX = XX FOR UPDATE;

 select * from t where c3 = 22 for update;

 会对c3 =22在i_c3索引上加exclusive next-key lock(LOCK_X|LOCK_ORDINARY)，对应主键加X锁(LOCK_X|LOCK_REC_NOT_GAP)，然后在下一条记录上加exclusive gap lock(LOCK_X|LOCK_GAP)。即该语句会锁定范围(11, 31)。
* SELECT … WHERE INDEX = XX LOCK IN SHARE MODE;

 加锁为：将上述FOR UPDATE语句的exclusive(LOCK_X)改为share(LOCK_S)。
* UPDATE … WHERE INDEX = XX;

 未包含索引列。

 update t set c4 = 2 where c3 = 22;

 加锁与上述FOR UPDATE一致。

 包含索引列。

 update t set c2 = 2 where c3 = 22;

 除了上述锁，对c1 = 20对应的唯一索引(i_c2)行加X锁。
* DELETE … WHERE INDEX = XX;

 除了SELECT … WHERE INDEX = XX FOR UPDATE的锁，添加对唯一索引、索引做加X锁操作。

### 查询条件为非唯一索引范围

这里加锁与唯一索引的当前读范围查询一致，不在赘述。

## Serializable 级别加锁分析

Serializable的加锁与RR隔离级别大多情形下一致，不同点是：

Serializable下普通SELECT语句查询也是当前读。例如下面语句：

select * from t where c1 = 20就会对c1=20的主键行加S锁(LOCK_S|LOCK_REC_NOT_GAP)。

对于UPDATE等做更新修改的语句，没有满足条件的行，也会对后面的行加GAP锁。例如下面语句：

update t set c4 = 12 where c1 = 15;也会对c1=20主键行加LOCK_X|LOCK_GAP锁。

## 总结

本文学习了InnoDB行锁相关源码，并对不同事务隔离级别下加锁进行了分析，对应知识点可以用于帮助分析SQL语句加锁情况。上面分析过程也可以发现，在RR隔离级别和Serializable隔离级别下，不同的路径选择不仅影响本语句执行效率，还会影响锁定的数据范围，严重影响并发。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)