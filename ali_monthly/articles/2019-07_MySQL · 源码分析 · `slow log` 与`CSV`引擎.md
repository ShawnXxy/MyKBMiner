# MySQL · 源码分析 · `slow log` 与`CSV`引擎

**Date:** 2019/07
**Source:** http://mysql.taobao.org/monthly/2019/07/08/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2019 / 07
 ](/monthly/2019/07)

 * 当期文章

 MySQL · 最佳实践 · Statement Outline
* PgSQL · 新特性解读 · undo log 存储接口（上）
* MySQL · 引擎特性 · Buffer Pool 漫谈
* MongoDB · 引擎特性 · oplog 查询优化
* PgSQL · 最佳实践 · pg_cron 内核分析及用法简介
* MySQL · 引擎特性 · CTE(Common Table Expressions)
* Database · 理论基础 · Mass Tree
* MySQL · 源码分析 · `slow log` 与`CSV`引擎
* PgSQL · 应用案例 · 使用SQL查询数据库日志
* PgSQL · 应用案例 · PostgreSQL psql的元素周期表

 ## MySQL · 源码分析 · `slow log` 与`CSV`引擎 
 Author: 攒叶 

 ## Overview
`slow log`可帮助`DBA`定位可能存在问题的`SQ`语句，从而进行`SQL`语句层面的优化。`slow log`可以记录到文件或者`mysql.slow_log`表上，目前大部分情况下采用后者。`mysql.slow_log`采用`CSV`引擎进行存取。本文将结合两者阐述`mysql`记录`slow log`以及`CSV`引擎本身的相关实现细节。

## Recording slow log
1. 写入一条`slow log`的函数调用栈：
 `dispatch_command-->
log_slow_statement-->
log_slow_do-->
Query_logger::slow_log_write-->
Log_to_csv_event_handler::log_slow-->
handler::ha_write_row-->
ha_tina::write_row
`
2. 记录`low log`发生在`sql`语句执行完成后，而一条`sql`语句是否被记录取决于四方面：
 ```
 if (thd->enable_slow_log && opt_slow_log) {
 bool warn_no_index =
 ((thd->server_status &
 (SERVER_QUERY_NO_INDEX_USED | SERVER_QUERY_NO_GOOD_INDEX_USED)) &&
 opt_log_queries_not_using_indexes &&
 !(sql_command_flags[thd->lex->sql_command] & CF_STATUS_COMMAND));
 bool log_this_query =
 ((thd->server_status & SERVER_QUERY_WAS_SLOW) || warn_no_index) &&
 (thd->get_examined_row_count() >=
 thd->variables.min_examined_row_limit);
 bool suppress_logging = log_throttle_qni.log(thd, warn_no_index);

 if (!suppress_logging && log_this_query) DBUG_RETURN(true);
 }

```
 
 ```
 - 配置是否打开了`slow log`功能，由参数`opt_slow_log`决定
 - 查询语句执行的时间超过`long_query_time`、并且检索的行数超过`min_examined_row_limit`
 - 优化过程中发现无法使用索引、或者无高效索引的查询
 - 对于无法使用索引或无高效索引的查询下，通过限流器`Slow_log_throttle`，限制其（日志）产生的速度 ``` if (eligible && inc_log_count(*rate)) { /* Current query's logging should be suppressed. Add its execution time and lock time to totals for the current window. */ total_exec_time += (end_utime_of_query - thd->start_utime); total_lock_time += (thd->utime_after_lock - thd->start_utime); suppress_current = true; } ```

```
3. 然后通过`Query_logger`，该类封装了日志（目前只有`general log`、`slow log`）`handler`设置的接口（如`activate_log_handler`）、记录本次连接产生的慢查询日志次数等
4. `Query_logger`在`Log_event_handler`，`Log_to_file_event_handler`，`Log_to_csv_event_handler`中封装了日志handler的接口。`Log_event_handler`的派生类有`Log_to_file_event_handler`、`Log_to_csv_event_handler`，允许日志输出到文件或（和）表，可由`--log_ouput`指定
5. 由于`THD->TEX`中未有打开`slow log`的`table`，所以在`Log_to_csv_event_handler::log_slow`中构造`TABLE_LIST`指定打开`slow_log`表，并最终调用`ha_tina::write_row`

## CSV Engine
`CSV`引擎可以将普通的`CSV`文件（逗号分隔的文件）作为`MySQL`的表处理。其主要代码在`storage/csv/`下。

### TINA_SHARE
同一张表的多个`handler`之间共享数据一般会采用`TABLE_SHARE`类，但`CSV`引擎采用了自定义的`TINA_SHARE`类进行数据共享。

`struct TINA_SHARE {
 char *table_name;
 char data_file_name[FN_REFLEN];
 uint table_name_length, use_count;

 bool is_log_table;
 my_off_t saved_data_file_length;
 mysql_mutex_t mutex;
 THR_LOCK lock;
 bool update_file_opened;
 bool tina_write_opened;
 File meta_file; /* Meta file we use */
 File tina_write_filedes; /* File handler for readers */
 bool crashed; /* Meta file is crashed */
 ha_rows rows_recorded; /* Number of rows in tables */
 uint data_file_version; /* Version of the data file used */
}
`
对于一张`CSV`表，同一时刻可有多个`handler`，但是只有一个`TINA_SHARE`实例。所有表的`TINA_SHARE`实例维护在`tina_open_tables`：

`static unique_ptr<collation_unordered_multimap<string, TINA_SHARE *>> tina_open_tables;
`
`TINA_SHARE`实例（以下采用`share`简称）采用引用计数`share->use_count`自动完成资源的回收。

### record count
MySQL中不同的存储引擎维护表的记录数的方法是不同的，如`InnoDB`中记录数只是个估计值，被用于优化器（[https://dev.mysql.com/doc/refman/8.0/en/innodb-restrictions.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-restrictions.html)）
`CSV`尝试维护准确的记录数，其方法：

* `handler::ha_statistics::stats::records`：该值会随着`write_row`、以及`delete_row`等而变化，但由于该值属于`handler`的成员变量，而一张表可有多个`handler`，并且每次打开`handler`该值都会被清`0`，所以它对于记录数的统计是不可靠的。
* `share->rows_recorded`：由于每张表只有一个`share`，所以大部分情况下该参数能够准确反映表的记录数。但`share->rows_recorded`只会在`share`的引用计数清`0`、以及全表扫描后调用`rnd_end()`方法时持久化到元数据文件。如果在使用过程中，`MySQL`发生`crash`，元数据文件的对应值也是不准确的。

`if (!--share->use_count) {
 // ... 
 /* Write the meta file. Mark it as crashed if needed. */
 (void)write_meta_file(share->meta_file, share->rows_recorded,
 share->crashed ? true : false);
 tina_open_tables->erase(share->table_name);
 // ... 
}
`
* `crash`重启后，可以通过`repair`方法修正`share->rows_recorded`：a）如果数据文件为空，则`share->rows_recorded`被重置为0；b）如果数据文件每一行都能正确读取，则`share->rows_recorded`被设置为数据文件的行数；c）如果发现错误行，则将数据文件截断到最近正确的行，丢弃掉错误记录后所有数据，并将`share->rows_recorded`设置为已读取的记录数。

```
int ha_tina::repair(THD *thd, HA_CHECK_OPT *) {
 if (!share->saved_data_file_length) // ...
 if (rc == HA_ERR_END_OF_FILE) // ...
 share->rows_recorded = rows_repaired; // ...
}

```

### Table Scan
`CSV`没有索引，仅有全表扫描，涉及的方法有`rnd_init`、`rnd_next`、`rnd_end`，`rnd_next`中核心方法为`find_current_row`，该方法会从缓冲区中读入一行中各个字段的值。`CSV`的数据文件（后缀为`.CSV`）也很简单，其典型例子为：

`"2019-07-31 07:11:30.173134","root[root] @ localhost []","838:59:59.000000","00:00:00.000332",4,4,"test",0,0,1,"select * from mysql.user",9
"2019-07-31 07:11:30.508952","root[root] @ localhost []","00:00:00.206624","00:00:00.002017",0,574,"mtr",0,0,1,"CALL mtr.check_warnings(@result)",10
"2019-07-31 07:11:30.644794","root[root] @ localhost []","00:00:00.006851","00:00:00.000716",1,655,"test",0,0,1,"SHOW VARIABLES LIKE 'debug'",11
`

### Update Row
`update`、`delete`会改动数据文件，其中`update`操作会先将原纪录`delete`，再插入新的数据。
`update`、`delete`操作在执行之前，需要执行`rnd_next`扫描表，找到所关联的`row`
`update`、`delete`操作依赖于:

`struct tina_set {
 my_off_t begin;
 my_off_t end;
}
class ha_tina : public handler {
 tina_set chain_buffer[DEFAULT_CHAIN_LENGTH];
 tina_set *chain;
 tina_set *chain_ptr;
}
`
* `chain_buffer`中存储了当前所有被标记为`delete`的`row`
* `tina_set::begin`指明该`row`在文件中的起点，`tina_set::end`为终点
* `chain`指向本次迭代扫描时的`chain`链的起点，`chain_ptr`指向`chain`链的尾部

每次执行`update/delete`，都会调用`chain_append`方法往`chain`链表尾部插入删除点。默认情况下，删除点`tina_set`会存放于预先分配的空间`chain_buffer`中。但当有大量删除点时，`chain_append`会调用`realloc/malloc`额外申请更大的空间

`int ha_tina::chain_append() {
 // 如果是连续的删除点，则合并
 if (chain_ptr != chain && (chain_ptr - 1)->end == current_position)
 
 // 如果空间不够，则申请内存，并将原数据拷贝到新空间(若采用malloc)
 if ((size_t)(chain_ptr - chain) == (chain_size - 1)) {
 chain_size += DEFAULT_CHAIN_LENGTH;
 chain = (tina_set *)my_realloc(...)
 // OR
 *ptr = (tina_set *)my_malloc(...)
 memcpy(ptr, chain, DEFAULT_CHAIN_LENGTH * sizeof(tina_set));
 }
 
 // 插入删除点
 chain_ptr->begin = current_position;
 chain_ptr->end = next_position;
 chain_ptr++;
}
`

对于`delete`操作，`chain_append`操作已经足够。对于`update`操作，则仍需要打开一个临时文件(后缀为`.CSN`)，将更新后的数据插入到临时文件中：

`int ha_tina::update_row(const uchar *, uchar *new_data) {
 if (open_update_temp_file_if_needed()) goto err;
 if (mysql_file_write(update_temp_file, (uchar *)buffer.ptr(), 
 size, MYF(MY_WME | MY_NABP)))
 got err;
}
`

当全表扫描结束后，则在`rnd_end`中将原数据文件未有被标记为`delete`的记录插入到临时文件中。最后，删除原文件，并将临时文件重命名为数据文件：

`int ha_tina::rnd_end() {
 while ((file_buffer_start != (my_off_t)-1))
 {
 mysql_file_write(update_temp_file, ...);
 if (in_hole) {
 // skip hole
 }
 }
 
 mysql_file_rename(...)
}
`
更新的记录会被放入数据文件头，原记录顺序会被打乱。

尽管`server`层会上表锁，但在`update`过程发生前，可能有其它`handler`已经打开。对于后者持有的数据文件描述符已经不再有效。`share->data_file_version`用于标识数据文件的版本，当`handler`发现`local_data_file_version`落后于`share->data_file_version`，则会重新打开数据文件

`int ha_tina::init_data_file() {
 if (local_data_file_version != share->data_file_version) {
 local_data_file_version = share->data_file_version;
 if (mysql_file_close(data_file, MYF(0)) ||
 (data_file = mysql_file_open(csv_key_file_data, share->data_file_name,
 O_RDONLY, MYF(MY_WME))) == -1)
 // ...
 }
}
`

## Summary
可以看到`CSV`是一款相当简单的引擎，没有索引，仅支持全表扫描，读写都简单地调用标准函数`read`、`write`，当主机`crash`掉后内核缓冲区里的数据会丢失，仅适用于如`slow log`这种对于可靠性、性能要求并不高的场景。

## Reference
* Slow Log Doc
* CSV Doc
* The Relevant Code

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)