# MySQL · 源码分析 · MySQL Statement Digest

**Date:** 2020/11
**Source:** http://mysql.taobao.org/monthly/2020/11/01/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2020 / 11
 ](/monthly/2020/11)

 * 当期文章

 MySQL · 源码分析 · MySQL Statement Digest
* Database · 理论基础 · B-tree 物理结构的并发控制
* MySQL · 源码阅读 · 创建二级索引
* MySQL · 源码阅读 · Secondary Engine

 ## MySQL · 源码分析 · MySQL Statement Digest 
 Author: 乐峰 

 ## **什么是statement digest**

在MySQL中，performance_schema库中存储server执行过程中各种”event”相关的数据，通过这些数据，可以从多维度分析数据库的性能，比如SQL执行，文件I/O，锁等待等。

一些数据表会存储执行过的SQL语句和其digest，比如表events_statements_summary_by_digest中的第二个和第三个列，digest其实是一个MD5 hash值，接下来简单介绍下digest的生成过程。

`MySQL [test]> describe performance_schema.events_statements_summary_by_digest;
+-----------------------------+---------------------+------+-----+---------------------+-------+
| Field | Type | Null | Key | Default | Extra |
+-----------------------------+---------------------+------+-----+---------------------+-------+
| SCHEMA_NAME | varchar(64) | YES | | NULL | |
| DIGEST | varchar(32) | YES | | NULL | |
| DIGEST_TEXT | longtext | YES | | NULL | |
| COUNT_STAR | bigint(20) unsigned | NO | | NULL | |
| SUM_TIMER_WAIT | bigint(20) unsigned | NO | | NULL | |
| MIN_TIMER_WAIT | bigint(20) unsigned | NO | | NULL | |
| AVG_TIMER_WAIT | bigint(20) unsigned | NO | | NULL | |
| MAX_TIMER_WAIT | bigint(20) unsigned | NO | | NULL | |
| SUM_LOCK_TIME | bigint(20) unsigned | NO | | NULL | |
| SUM_ERRORS | bigint(20) unsigned | NO | | NULL | |
| SUM_WARNINGS | bigint(20) unsigned | NO | | NULL | |
| SUM_ROWS_AFFECTED | bigint(20) unsigned | NO | | NULL | |
| SUM_ROWS_SENT | bigint(20) unsigned | NO | | NULL | |
| SUM_ROWS_EXAMINED | bigint(20) unsigned | NO | | NULL | |
| SUM_CREATED_TMP_DISK_TABLES | bigint(20) unsigned | NO | | NULL | |
| SUM_CREATED_TMP_TABLES | bigint(20) unsigned | NO | | NULL | |
| SUM_SELECT_FULL_JOIN | bigint(20) unsigned | NO | | NULL | |
| SUM_SELECT_FULL_RANGE_JOIN | bigint(20) unsigned | NO | | NULL | |
| SUM_SELECT_RANGE | bigint(20) unsigned | NO | | NULL | |
| SUM_SELECT_RANGE_CHECK | bigint(20) unsigned | NO | | NULL | |
| SUM_SELECT_SCAN | bigint(20) unsigned | NO | | NULL | |
| SUM_SORT_MERGE_PASSES | bigint(20) unsigned | NO | | NULL | |
| SUM_SORT_RANGE | bigint(20) unsigned | NO | | NULL | |
| SUM_SORT_ROWS | bigint(20) unsigned | NO | | NULL | |
| SUM_SORT_SCAN | bigint(20) unsigned | NO | | NULL | |
| SUM_NO_INDEX_USED | bigint(20) unsigned | NO | | NULL | |
| SUM_NO_GOOD_INDEX_USED | bigint(20) unsigned | NO | | NULL | |
| FIRST_SEEN | timestamp | NO | | 0000-00-00 00:00:00 | |
| LAST_SEEN | timestamp | NO | | 0000-00-00 00:00:00 | |
+-----------------------------+---------------------+------+-----+---------------------+-------+
`

## **statement digest如何计算**

digest是基于一串字节文本做MD5计算出来的hash值，这个字节文本在parser解析SQL时根据识别出来的token和identifier构造。下面以MySQL 5.7的代码为例，简单介绍digest的生成过程。

当一个token被识别时，调用`store_token()`构造字节文本，

`File sql/sql_digest.cc

 71 /**
 72 Store a single token in token array.
 73 */
 74 inline void store_token(sql_digest_storage* digest_storage, uint token)
 75 {
 76 DBUG_ASSERT(digest_storage->m_byte_count <= digest_storage->m_token_array_length);
 77 
 78 if (digest_storage->m_byte_count + SIZE_OF_A_TOKEN <= digest_storage->m_token_array_length)
 79 {
 80 unsigned char* dest= & digest_storage->m_token_array[digest_storage->m_byte_count];
 81 dest[0]= token & 0xff;
 82 dest[1]= (token >> 8) & 0xff;
 83 digest_storage->m_byte_count+= SIZE_OF_A_TOKEN;
 84 }
 85 else
 86 {
 87 digest_storage->m_full= true;
 88 }
 89 }
 90 
`

当一个identifier被识别时，调用`store_token_identifier()`，传入token值，identifier name以及其长度，根据一定的规则构造字节文本，并append到之前构造的文本后面。

`File sql/sql_digest.cc

135 inline void store_token_identifier(sql_digest_storage* digest_storage,
136 uint token,
137 size_t id_length, const char *id_name)
138 {
139 DBUG_ASSERT(digest_storage->m_byte_count <= digest_storage->m_token_array_length);
140 
141 size_t bytes_needed= 2 * SIZE_OF_A_TOKEN + id_length;
142 if (digest_storage->m_byte_count + bytes_needed <= (unsigned int)digest_storage->m_token_array_length)
143 {
144 unsigned char* dest= & digest_storage->m_token_array[digest_storage->m_byte_count];
145 /* Write the token */
146 dest[0]= token & 0xff;
147 dest[1]= (token >> 8) & 0xff;
148 /* Write the string length */
149 dest[2]= id_length & 0xff;
150 dest[3]= (id_length >> 8) & 0xff;
151 /* Write the string data */
152 if (id_length > 0)
153 memcpy((char *)(dest + 4), id_name, id_length);
154 digest_storage->m_byte_count+= bytes_needed;
155 }
156 else
157 {
158 digest_storage->m_full= true;
159 }
160 }
`

可以看到，

* 前两个字节，函数中的dest[0] / dest[1] ，根据token值计算而来；
* 第三和第四个字节，dest[2] / dest[3]，根据id_length计算而来；
* 之后的地址存放id_name对应的文本。

`store_token()`和`store_token_identifier()`可以被调用多次，从而把不断识别出的token和identifier拼接成一个最终的字节文本，存放在digest_storage->m_token_array中。

相关的函数调用路径如下，

`Breakpoint 1, store_token_identifier (digest_storage=0x7ff428002848, token=945, id_length=18, id_name=0x7ff428006118 "performance_schema")
 at /disk6/lefeng/porting/polardb571/sql/sql_digest.cc:139
139 DBUG_ASSERT(digest_storage->m_byte_count <= digest_storage->m_token_array_length);
(gdb) bt
#0 store_token_identifier (digest_storage=0x7ff428002848, token=945, id_length=18, id_name=0x7ff428006118 "performance_schema")
 at /disk6/lefeng/porting/polardb571/sql/sql_digest.cc:139
#1 0x000000000166049f in digest_add_token (state=0x7ff428002840, token=945, yylval=0x7ff4c0281a60) at /disk6/lefeng/porting/polardb571/sql/sql_digest.cc:590
#2 0x0000000001675311 in Lex_input_stream::add_digest_token (this=0x7ff4c0283568, token=488, yylval=0x7ff4c0281a60) at /disk6/lefeng/porting/polardb571/sql/sql_lex.cc:382
#3 0x00000000016777ab in MYSQLlex (yylval=0x7ff4c0281a60, yylloc=0x7ff4c0281a40, thd=0x7ff428000950) at /disk6/lefeng/porting/polardb571/sql/sql_lex.cc:1362
#4 0x00000000017fd83b in MYSQLparse (YYTHD=0x7ff428000950) at /disk6/lefeng/porting/polardb571/sql/sql_yacc.cc:20171
#5 0x00000000016b8801 in parse_sql (thd=0x7ff428000950, parser_state=0x7ff4c0283560, creation_ctx=0x0) at /disk6/lefeng/porting/polardb571/sql/sql_parse.cc:7578
#6 0x00000000016b5300 in mysql_parse (thd=0x7ff428000950, parser_state=0x7ff4c0283560) at /disk6/lefeng/porting/polardb571/sql/sql_parse.cc:5924
#7 0x00000000016a9e3e in dispatch_command (thd=0x7ff428000950, com_data=0x7ff4c0283dd0, command=COM_QUERY) at /disk6/lefeng/porting/polardb571/sql/sql_parse.cc:1550
#8 0x00000000016a8a5b in do_command (thd=0x7ff428000950) at /disk6/lefeng/porting/polardb571/sql/sql_parse.cc:1011
#9 0x00000000017eeb6e in handle_connection (arg=0x5cec640) at /disk6/lefeng/porting/polardb571/sql/conn_handler/connection_handler_per_thread.cc:303
#10 0x0000000001a9f465 in pfs_spawn_thread (arg=0x5d87a50) at /disk6/lefeng/porting/polardb571/storage/perfschema/pfs.cc:2188
#11 0x00007ff4c9795e25 in start_thread () from /lib64/libpthread.so.0
#12 0x00007ff4c865cbad in clone () from /lib64/libc.so.6
`

在SQL stament执行完后，调用函数`find_or_create_digest()`计算MD5 hash，

`File storage/perfschema/pfs_digest.cc

188 PFS_statement_stat*
189 find_or_create_digest(PFS_thread *thread,
190 const sql_digest_storage *digest_storage,
191 const char *schema_name,
192 uint schema_name_length)
193 {
 ...
 
202 LF_PINS *pins= get_digest_hash_pins(thread);
203 if (unlikely(pins == NULL))
204 return NULL;
205 
206 /*
207 Note: the LF_HASH key is a block of memory,
208 make sure to clean unused bytes,
209 so that memcmp() can compare keys.
210 */
211 PFS_digest_key hash_key;
212 memset(& hash_key, 0, sizeof(hash_key));
213 /* Compute MD5 Hash of the tokens received. */
214 compute_digest_md5(digest_storage, hash_key.m_md5);
215 memcpy((void*)& digest_storage->m_md5, &hash_key.m_md5, MD5_HASH_SIZE);
216 /* Add the current schema to the key */
217 hash_key.m_schema_name_length= schema_name_length;
218 if (schema_name_length > 0)
219 memcpy(hash_key.m_schema_name, schema_name, schema_name_length);
220 
221 ...
`

在storage/perfschema/pfs_digest.cc第214行，`find_or_create_digest()`会调用`compute_digest_md5()` ，`compute_digest_md5()`会从digest_storage->m_token_array读取构造好的字节文本，完成hash计算。

`File sql/sql_digest.cc

162 void compute_digest_md5(const sql_digest_storage *digest_storage, unsigned char *md5)
163 { 
164 compute_md5_hash((char *) md5,
165 (const char *) digest_storage->m_token_array,
166 digest_storage->m_byte_count);
167 } 
168 
`

## **statement digest计算示例**

接下来，我们以SQL语句 “TRUNCATE TABLE performance_schema.events_statements_summary_by_digest” 为例介绍 digest计算过程。

1. 首先识别出的token是859，对应的token定义如下，其token值用于填充字节文本的前2个字节，

`File sql/sql_yacc.h

#define TRUNCATE_SYM 859
`

函数调用栈如下，

`Breakpoint 2, store_token (digest_storage=0x7ff428002848, token=859) at /disk6/lefeng/porting/polardb571/sql/sql_digest.cc:76
76 DBUG_ASSERT(digest_storage->m_byte_count <= digest_storage->m_token_array_length);
(gdb) n
78 if (digest_storage->m_byte_count + SIZE_OF_A_TOKEN <= digest_storage->m_token_array_length)
(gdb) 
80 unsigned char* dest= & digest_storage->m_token_array[digest_storage->m_byte_count];
(gdb) 
81 dest[0]= token & 0xff;
(gdb) 
82 dest[1]= (token >> 8) & 0xff;
(gdb) 
83 digest_storage->m_byte_count+= SIZE_OF_A_TOKEN;
(gdb) 
89 }
(gdb) p digest_storage->m_byte_count
(gdb) 2

(gdb) bt
#0 store_token (digest_storage=0x7ff428002848, token=859) at /disk6/lefeng/porting/polardb571/sql/sql_digest.cc:76
#1 0x00000000016604c2 in digest_add_token (state=0x7ff428002840, token=859, yylval=0x7ff4c0281a60) at /disk6/lefeng/porting/polardb571/sql/sql_digest.cc:599
#2 0x0000000001675311 in Lex_input_stream::add_digest_token (this=0x7ff4c0283568, token=859, yylval=0x7ff4c0281a60) at /disk6/lefeng/porting/polardb571/sql/sql_lex.cc:382
#3 0x00000000016777ab in MYSQLlex (yylval=0x7ff4c0281a60, yylloc=0x7ff4c0281a40, thd=0x7ff428000950) at /disk6/lefeng/porting/polardb571/sql/sql_lex.cc:1362
#4 0x00000000017fd83b in MYSQLparse (YYTHD=0x7ff428000950) at /disk6/lefeng/porting/polardb571/sql/sql_yacc.cc:20171
#5 0x00000000016b8801 in parse_sql (thd=0x7ff428000950, parser_state=0x7ff4c0283560, creation_ctx=0x0) at /disk6/lefeng/porting/polardb571/sql/sql_parse.cc:7578
#6 0x00000000016b5300 in mysql_parse (thd=0x7ff428000950, parser_state=0x7ff4c0283560) at /disk6/lefeng/porting/polardb571/sql/sql_parse.cc:5924
#7 0x00000000016a9e3e in dispatch_command (thd=0x7ff428000950, com_data=0x7ff4c0283dd0, command=COM_QUERY) at /disk6/lefeng/porting/polardb571/sql/sql_parse.cc:1550
#8 0x00000000016a8a5b in do_command (thd=0x7ff428000950) at /disk6/lefeng/porting/polardb571/sql/sql_parse.cc:1011
#9 0x00000000017eeb6e in handle_connection (arg=0x5cec640) at /disk6/lefeng/porting/polardb571/sql/conn_handler/connection_handler_per_thread.cc:303
#10 0x0000000001a9f465 in pfs_spawn_thread (arg=0x5d87a50) at /disk6/lefeng/porting/polardb571/storage/perfschema/pfs.cc:2188
#11 0x00007ff4c9795e25 in start_thread () from /lib64/libpthread.so.0
#12 0x00007ff4c865cbad in clone () from /lib64/libc.so.6
`

1. 其次识别出的token是835，其对应的token定义如下，其token值用于填充接下来的2字节，

```
File sql/sql_yacc.h

#define TABLE_SYM 835

```

1. 然后识别出来并用于构造字节文本的token是488，对应的token定义是，

```
#define IDENT_QUOTED 488

```

该token在`digest_add_token()`中被转换为token 945 （参考588行）

`File sql/sql_digest.cc

379 sql_digest_state* digest_add_token(sql_digest_state *state,
380 uint token,
381 LEX_YYSTYPE yylval)
...
401 switch (token)
402 {
403 case NUM:
404 case LONG_NUM:
405 case ULONGLONG_NUM:
406 case DECIMAL_NUM:
407 case FLOAT_NUM:
408 case BIN_NUM:
409 case HEX_NUM:
...
571 case IDENT:
572 case IDENT_QUOTED:
573 case TOK_IDENT_AT:
574 {
575 YYSTYPE *lex_token= yylval;
576 char *yytext= lex_token->lex_str.str;
577 size_t yylen= lex_token->lex_str.length;
578 
579 /*
580 REDUCE:
581 TOK_IDENT := IDENT | IDENT_QUOTED
582 The parser gives IDENT or IDENT_TOKEN for the same text,
583 depending on the character set used.
584 We unify both to always print the same digest text,
585 and always have the same digest hash.
586 */
587 if (token != TOK_IDENT_AT)
588 token= TOK_IDENT;
589 /* Add this token and identifier string to digest storage. */
590 store_token_identifier(digest_storage, token, yylen, yytext);
591 
592 /* Update the index of last identifier found. */
593 state->m_last_id_index= digest_storage->m_byte_count;
594 break;
595 }
`

根据token值和id_name的长度构造4字节文本数据，之后把”performance_schema”追加到其后。

```
Breakpoint 1, store_token_identifier (digest_storage=0x7ff428002848, token=945, id_length=18, id_name=0x7ff428006118 "performance_schema")
 at /disk6/lefeng/porting/polardb571/sql/sql_digest.cc:139
139 DBUG_ASSERT(digest_storage->m_byte_count <= digest_storage->m_token_array_length);
(gdb) p digest_storage->m_byte_count
$3 = 4
(gdb) n
141 size_t bytes_needed= 2 * SIZE_OF_A_TOKEN + id_length;
(gdb) 
142 if (digest_storage->m_byte_count + bytes_needed <= (unsigned int)digest_storage->m_token_array_length)
(gdb) 
144 unsigned char* dest= & digest_storage->m_token_array[digest_storage->m_byte_count];
(gdb) 
146 dest[0]= token & 0xff;
(gdb) 
147 dest[1]= (token >> 8) & 0xff;
(gdb) 
149 dest[2]= id_length & 0xff;
(gdb) 
150 dest[3]= (id_length >> 8) & 0xff;
(gdb) 
152 if (id_length > 0)
(gdb) 
153 memcpy((char *)(dest + 4), id_name, id_length);
(gdb) 
154 digest_storage->m_byte_count+= bytes_needed;
(gdb) 
160 }
(gdb) p digest_storage->m_byte_count
$4 = 26

```

1. 接下来识别出的token是46

```
Breakpoint 2, store_token (digest_storage=0x7ff428002848, token=46) at /disk6/lefeng/porting/polardb571/sql/sql_digest.cc:76
76 DBUG_ASSERT(digest_storage->m_byte_count <= digest_storage->m_token_array_length);
78 if (digest_storage->m_byte_count + SIZE_OF_A_TOKEN <= digest_storage->m_token_array_length)
(gdb) 
80 unsigned char* dest= & digest_storage->m_token_array[digest_storage->m_byte_count];
(gdb) 
81 dest[0]= token & 0xff;
(gdb) 
82 dest[1]= (token >> 8) & 0xff;
(gdb) 
83 digest_storage->m_byte_count+= SIZE_OF_A_TOKEN;
(gdb) 
89 }
(gdb) p digest_storage->m_byte_count
$11 = 28

```

1. 最后识别出来且用于构造字节文本的token是945 （同上，488转换而来），”events_statements_summary_by_digest”会被追加到文本末尾。

```
Breakpoint 1, store_token_identifier (digest_storage=0x7ff428002848, token=945, id_length=35, id_name=0x7ff428006130 "events_statements_summary_by_digest")
 at /disk6/lefeng/porting/polardb571/sql/sql_digest.cc:139
139 DBUG_ASSERT(digest_storage->m_byte_count <= digest_storage->m_token_array_length);
(gdb) n
141 size_t bytes_needed= 2 * SIZE_OF_A_TOKEN + id_length;
(gdb) 
142 if (digest_storage->m_byte_count + bytes_needed <= (unsigned int)digest_storage->m_token_array_length)
(gdb) 
144 unsigned char* dest= & digest_storage->m_token_array[digest_storage->m_byte_count];
(gdb) 
146 dest[0]= token & 0xff;
(gdb) 
147 dest[1]= (token >> 8) & 0xff;
(gdb) 
149 dest[2]= id_length & 0xff;
(gdb) 
150 dest[3]= (id_length >> 8) & 0xff;
(gdb) 
152 if (id_length > 0)
(gdb) 
153 memcpy((char *)(dest + 4), id_name, id_length);
(gdb) 
154 digest_storage->m_byte_count+= bytes_needed;
(gdb) 
160 }
(gdb) p digest_storage->m_byte_count
$12 = 67

```

1. 至此，字节文本构造完毕，接下来计算MD5 hash,

```
Breakpoint 3, find_or_create_digest (thread=0x7ff4c7cc2c00, digest_storage=0x7ff428002848, schema_name=0x7ff428002930 "", schema_name_length=0)
 at /disk6/lefeng/porting/polardb571/storage/perfschema/pfs_digest.cc:194
194 DBUG_ASSERT(digest_storage != NULL);
(gdb) n
196 if (statements_digest_stat_array == NULL)
(gdb) 
199 if (digest_storage->m_byte_count <= 0)
(gdb) 
202 LF_PINS *pins= get_digest_hash_pins(thread);
(gdb) 
203 if (unlikely(pins == NULL))
(gdb) 
212 memset(& hash_key, 0, sizeof(hash_key));
(gdb) 
214 compute_digest_md5(digest_storage, hash_key.m_md5);
(gdb) 
215 memcpy((void*)& digest_storage->m_md5, &hash_key.m_md5, MD5_HASH_SIZE);
(gdb) 
217 hash_key.m_schema_name_length= schema_name_length;
(gdb) p /x hash_key.m_md5
$13 = {0xf8, 0x37, 0x3f, 0x7b, 0xed, 0x47, 0x77, 0x3d, 0x4c, 0xd1, 0xd5, 0xc0, 0xab, 0xb7, 0x88, 0xc9}
(gdb) bt
#0 find_or_create_digest (thread=0x7ff4c7cc2c00, digest_storage=0x7ff428002848, schema_name=0x7ff428002930 "", schema_name_length=0)
 at /disk6/lefeng/porting/polardb571/storage/perfschema/pfs_digest.cc:217
#1 0x0000000001aa648e in pfs_end_statement_v1 (locker=0x7ff428002888, stmt_da=0x7ff428003890) at /disk6/lefeng/porting/polardb571/storage/perfschema/pfs.cc:5405
#2 0x00000000016a5c46 in inline_mysql_end_statement (locker=0x7ff428002888, stmt_da=0x7ff428003890) at /disk6/lefeng/porting/polardb571/include/mysql/psi/mysql_statement.h:228
#3 0x00000000016ab574 in dispatch_command (thd=0x7ff428000950, com_data=0x7ff4c0283dd0, command=COM_QUERY) at /disk6/lefeng/porting/polardb571/sql/sql_parse.cc:2023
#4 0x00000000016a8a5b in do_command (thd=0x7ff428000950) at /disk6/lefeng/porting/polardb571/sql/sql_parse.cc:1011
#5 0x00000000017eeb6e in handle_connection (arg=0x5cec640) at /disk6/lefeng/porting/polardb571/sql/conn_handler/connection_handler_per_thread.cc:303
#6 0x0000000001a9f465 in pfs_spawn_thread (arg=0x5d87a50) at /disk6/lefeng/porting/polardb571/storage/perfschema/pfs.cc:2188
#7 0x00007ff4c9795e25 in start_thread () from /lib64/libpthread.so.0
#8 0x00007ff4c865cbad in clone () from /lib64/libc.so.6

```

1. 最后，查询performance_schema.events_statements_summary_by_digest，显示计算出的MD5 hash值。

```
MySQL [(none)]> SELECT SCHEMA_NAME, DIGEST, DIGEST_TEXT FROM performance_schema.events_statements_summary_by_digest;
+-------------+----------------------------------+------------------------------------------------------------------------------+
| SCHEMA_NAME | DIGEST | DIGEST_TEXT |
+-------------+----------------------------------+------------------------------------------------------------------------------+
| NULL | f8373f7bed47773d4cd1d5c0abb788c9 | TRUNCATE TABLE `performance_schema` . `events_statements_summary_by_digest` |
+-------------+----------------------------------+------------------------------------------------------------------------------+
1 row in set (11.13 sec)

```

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)