# MySQL · 源码分析 · CSV 引擎详解

**Date:** 2021/10
**Source:** http://mysql.taobao.org/monthly/2021/10/05/
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

 ## MySQL · 源码分析 · CSV 引擎详解 
 Author: 晓卓 

 ## CSV Engine
MySQL中有多种存储引擎，不同的存储引擎提供不同的存储机制、索引技巧、锁定水平等功能，使用不同的存储引擎，可以获得特定的功能。
​

这里介绍的CSV引擎主要特点是简单方便，可以直接将文本形式的数据存储mysql中的表。csv引擎不支持索引、事务、查询下推等，一般用于日志表的数据存储或者作为数据转换的中间表，可以将直接excel表或者csv文件导入mysql中，方便用户使用。

* csv文件的每条记录被分隔符分隔为字段（典型分隔符有逗号、分号或制表符；有时分隔符可以包括可选的空格）
* 每条记录都有同样的字段序列。
* mysql中支持的csv是逗号分隔，字符串型的field需要加双引号，field中的\r\n前需要加转义符，形式如下:

`1,"xuhaiyan","\\n"
2,"xuhaiyan","\\r"
3,"xuhaiyan","\\r"
`

## Relevant Code
MySQL CSV引擎的源码位于storage/csv下面。包括transparent_file 和 ha_tina 两个主要的部分。其中ha_tina继承文件操作的基类handler，与sever层进行交互。transparent_file 是作为csv文件写入和读取的file_buffer使用。

`class Transparent_file {
 File filedes;
 uchar *buff; /* in-memory window to the file or mmaped area */
 /* current window sizes */
 my_off_t lower_bound;
 my_off_t upper_bound;
 uint buff_size;

 public:
 Transparent_file();
 ~Transparent_file();

 void init_buff(File filedes_arg);
 uchar *ptr();
 my_off_t start();
 my_off_t end();
 char get_value(my_off_t offset);
 my_off_t read_next();
};
`

通过transparent_file类的定义我们可以看到该csv文件的file_buffer的一些基本描述信息。

* File类型的fileds是标识通过mysql_file_open()函数打开的文件。
— *buffer 是映射到文件的缓存，默认大小为4k。
* lower_bound和 upper_bound分别对应当前buff在文件中的位置的下界和上界。
* 类函数也比较简单，包括初始化、对buffer基本描述信息以及具体buffer中的内容的读取。

`class ha_tina : public handler {
 THR_LOCK_DATA lock; /* MySQL lock */
 TINA_SHARE *share; /* Shared lock info */
 my_off_t
 current_position; /* Current position in the file during a file scan */
 my_off_t next_position; /* Next position in the file scan */
 my_off_t local_saved_data_file_length; /* save position for reads */
 my_off_t temp_file_length;
 uchar byte_buffer[IO_SIZE];
 Transparent_file *file_buff;
 File data_file; /* File handler for readers */
 File update_temp_file;
 String buffer;
};
`

通过ha_tina的类定义，我们了解csv引擎所支持的文件操作特性。

* THR_LOCK_DATA lock是相关的锁，ha_tina引擎只支持表。
* TINA_SHARE *share 对于同一张csv表，同一时刻可有多个handler，他们之间的数据共享是通过共同维护一个TINA_SHARE的实例实现的。
* my_off_t current_position是当前数据读取的位置
* my_off_t next_position是下一个数据读取的位置

#### Table Scan
csv不支持索引，也没有行的概念。仅仅依靠识别下面三种行尾标记来判断行。只有读到该字符时才能感知到行的存在，因此无法任意读取某一行数据。仅支持全表扫描。

` /*
 '\r' -- Old Mac OS line ending
 '\n' -- Traditional Unix and Mac OS X line ending
 '\r''\n' -- DOS\Windows line ending
 */
 
 static my_off_t find_eoln_buff(Transparent_file *data_buff, my_off_t begin,
 my_off_t end, int *eoln_len) {
 *eoln_len = 0;

 for (my_off_t x = begin; x < end; x++) {
 /* Unix (includes Mac OS X) */
 if (data_buff->get_value(x) == '\n')
 *eoln_len = 1;
 else if (data_buff->get_value(x) == '\r') // Mac or Dos
 {
 /* old Mac line ending */
 if (x + 1 == end || (data_buff->get_value(x + 1) != '\n'))
 *eoln_len = 1;
 else // DOS style ending
 *eoln_len = 2;
 }

 if (*eoln_len) // end of line was found
 return x;
 }

 return 0;
}
`
全表扫描涉及的方法有rnd_init、rnd_next、rnd_end。

* rnd_init()方法将buffer设置为文件开头。
* rnd_next()方法中核心方法为find_current_row，该方法会从缓冲区中读入一行中各个字段的值。研读find_current_row()的源码发现，其读取方式并不是流式读取的，在真正开始读取一行之前，需要调用find_eoln_buff()方法，从当前位置逐个扫描每个字节直到发现行尾部标记。再回到起始位置读取完整一行的数据并进行解析。但如果buffer_size小于甚至远远小于一行数据的大小，则会在扫描过程中进行多次额外的I/O操作，会影响性能。

`int ha_tina::find_current_row(uchar *buf) {
 
 // ...
 
 /*
 find end of row
 */
 if ((end_offset = find_eoln_buff(file_buff, current_position,
 local_saved_data_file_length, &eoln_len)) ==
 0)
 return HA_ERR_END_OF_FILE;

 /* We must read all columns in case a table is opened for update */
 read_all = !bitmap_is_clear_all(table->write_set);
 /* Avoid asserts in ::store() for columns that are not going to be updated */
 org_bitmap = dbug_tmp_use_all_columns(table, table->write_set);
 error = HA_ERR_CRASHED_ON_USAGE;

 memset(buf, 0, table->s->null_bytes);

 for (Field **field = table->field; *field; field++) {
 char curr_char;

 buffer.length(0);
 if (curr_offset >= end_offset) goto err;
 curr_char = file_buff->get_value(curr_offset);
 /*
 Parse the line obtained using the following algorithm

 BEGIN
 1) Store the EOL (end of line) for the current row
 2) Until all the fields in the current query have not been
 filled
 2.1) If the current character is a quote
 2.1.1) Until EOL has not been reached
 a) If end of current field is reached, move
 to next field and jump to step 2.3
 b) If current character is a \\ handle
 \\n, \\r, \\, \\"
 c) else append the current character into the buffer
 before checking that EOL has not been reached.
 2.2) If the current character does not begin with a quote
 2.2.1) Until EOL has not been reached
 a) If the end of field has been reached move to the
 next field and jump to step 2.3
 b) If current character begins with \\ handle
 \\n, \\r, \\, \\"
 c) else append the current character into the buffer
 before checking that EOL has not been reached.
 2.3) Store the current field value and jump to 2)
 TERMINATE
 */
 
 }
 next_position = end_offset + eoln_len;
 error = 0;

err:
 dbug_tmp_restore_column_map(table->write_set, org_bitmap);

 return error;
}
`
* rnd_end()在全表扫描结束后将是否知道行数的flag标记(records_is_known)为true。

#### Update & Delete
```
 struct tina_set {
 my_off_t begin;
 my_off_t end;
 };
 
 class ha_tina : public handler {
 /*
 The chain contains "holes" in the file, occurred because of
 deletes/updates. It is used in rnd_end() to get rid of them
 in the end of the query.
 */
 tina_set chain_buffer[DEFAULT_CHAIN_LENGTH];
 tina_set *chain;
 tina_set *chain_ptr;
 uchar chain_alloced;
 uint32 chain_size;
 uint local_data_file_version; /* Saved version of the data file used */
 bool records_is_known;
 MEM_ROOT blobroot;
 };

```

以上是跟数据更新相关的成员变量。

update、delete会改动数据文件，其中update操作会先将原记录delete，再插入新的数据。

update、delete操作在执行之前，需要执行rnd_next扫描表，找到所关联的row update、delete操作。

* chain_buffer中存储了当前所有被标记为delete的row。
* tina_set::begin指明该row在文件中的起点，tina_set::end为终点。
* chain指向本次迭代扫描时的chain链的起点，chain_ptr指向chain链的尾部。

每次执行update/delete，都会调用chain_append方法往chain链表尾部插入删除点。

默认情况下，删除点tina_set会存放于预先分配的空间chain_buffer中。但当有大量删除点时，chain_append会调用realloc/malloc额外申请更大的空间。

对于delete操作，chain_append操作已经足够。对于update操作，则仍需要打开一个临时文件(后缀为.CSN)，将更新后的数据插入到临时文件中。

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
当全表扫描结束后，则在rnd_end中将原数据文件未有被标记为delete的记录插入到临时文件中。最后，删除原文件，并将临时文件重命名为数据文件。

#### Repair and Check

CSV存储引擎支持CHECK TABLE和REPAIR TABLE语句来验证损坏的CSV表，并尽可能修复CSV表。

当运行CHECK TABLE语句时，将通过查找正确的字段分隔符、转义字段(匹配或缺少引号)、与表定义比较的正确字段数量以及是否存在相应的CSV元文件来检查CSV文件的有效性。

`int ha_tina::check(THD *thd, HA_CHECK_OPT *) {
 // ...
 /* Read the file row-by-row. If everything is ok, repair is not needed. */
 while (!(rc = find_current_row(buf))) {
 thd_inc_row_count(thd);
 count--;
 current_position = next_position;
 }
 // ...
 if ((rc != HA_ERR_END_OF_FILE) || count) {
 share->crashed = true;
 return HA_ADMIN_CORRUPT;
 }
 
 return HA_ADMIN_OK;
 }
`
使用REPAIR TABLE修复表，它从现有CSV数据复制尽可能多的有效行，然后用恢复的行替换现有CSV文件。损坏数据以外的任何行都将丢失。

如果文件为空，更改文件中的行号并完成恢复。否则，扫描表寻找坏行。

如果没有找到，则将该文件标记为良好文件并返回。如果遇到坏行，则截断数据文件直到最后一个好的行。代码流程如下：

`int ha_tina::repair(THD *thd, HA_CHECK_OPT *) {
 |-// ...
 |
 | /* empty file */
 |-if (!share->saved_data_file_length) {
 | share->rows_recorded = 0;
 | goto end;
 | }
 |
 |-// ...
 |
 | /* Read the file row-by-row. If everything is ok, repair is not needed. */
 |-while (!(rc = find_current_row(buf))) {
 | // ...
 | }
 | current_position = next_position;
 | }
 |
 | /* all rows good，the file does not need repair */
 |-if (rc == HA_ERR_END_OF_FILE) {
 | // ...
 | }
 |
 | /* encountered a bad row => repair is needed =>create a temporary file */
 |-if(repair_file = mysql_file_create())
 | // ...
 | }
 | /* we just truncated the file up to the first bad row. update rows count. */
 | /* write repaired file */
 |-while (1) { 
 | |-mysql_file_write();
 | |
 | |-file_buff->read_next();
 | }
 | /* Close the files and rename repaired file to the datafile. */
 |-if (share->tina_write_opened) {
 | /* Data file might be opened twice, close both instances */
 | |-if (mysql_file_close(share->tina_write_filedes, MYF(0)))
 | |-return my_errno() ? my_errno() : -1;
 | |-share->tina_write_opened = false;
 | }
 |-if (mysql_file_close(data_file, MYF(0)) ||
 | mysql_file_close(repair_file, MYF(0)) ||
 | mysql_file_rename(csv_key_file_data, repaired_fname,
 | share->data_file_name, MYF(0)))
 | return -1;
 | /* Open the file again, it should now be repaired */
 |-if ((data_file = mysql_file_open(csv_key_file_data, share->data_file_name,
 | O_RDWR | O_APPEND, MYF(MY_WME))) == -1)
 | return my_errno() ? my_errno() : -1;
 | /* Set new file size. */
 |-local_saved_data_file_length = (size_t)current_position;
 |
end:
 |-share->crashed = false;
 |-return HA_ADMIN_OK;
}
`

在修复期间，只有从CSV文件到第一个损坏行的行被复制到新表中。从第一个损坏行到表尾的所有其他行都将被删除，即使是有效的行。

## Reference
[CSV Doc](https://dev.mysql.com/doc/refman/8.0/en/csv-storage-engine.html) 

 [The Relevant Code](https://github.com/mysql/mysql-server/tree/8.0/storage/csv)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)