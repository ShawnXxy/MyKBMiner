# MySQL · BUG分析 · Rename table 死锁分析

**Date:** 2016/03
**Source:** http://mysql.taobao.org/monthly/2016/03/06/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 03
 ](/monthly/2016/03)

 * 当期文章

 MySQL · TokuDB · 事务子系统和 MVCC 实现
* MongoDB · 特性分析 · MMAPv1 存储引擎原理
* PgSQL · 源码分析 · 优化器逻辑推理
* SQLServer · BUG分析 · Agent 链接泄露分析
* Redis · 特性分析 · AOF Rewrite 分析
* MySQL · BUG分析 · Rename table 死锁分析
* MySQL · 物理备份 · Percona XtraBackup 备份原理
* GPDB · 特性分析· GreenPlum FTS 机制
* MySQL · 答疑解惑 · 备库Seconds_Behind_Master计算
* MySQL · 答疑解惑 · MySQL 锁问题最佳实践

 ## MySQL · BUG分析 · Rename table 死锁分析 
 Author: lengxiang 

 ## 背景
InnoDB buffer pool中的page管理牵涉到两个链表，一个是lru链表，一个是flush 脏块链表，由于数据库的特性：

1. 脏块的刷新，是异步操作；
2. page存在两个版本，一个是ibd文件的持久化版本，和buffer pool内存中的当前版本。

所以在对table对象进行ddl变更的时候，要维护两个版本之间的一致性，有一些操作需要同步进行page缓存的管理。例如以下三种ddl操作：

**1. flush table t for export**

这是MySQL 5.6提供的InnoDB transportable tablespace功能，用于在不同实例之间进行表传输。由于需要透明的在物理层面迁移ibd文件，所以需要保证buffer pool中的page和ibd文件中的page的一致性。其操作步骤如下：

1. 持有t表的MDL锁，保证在t表上没有活跃事务，即buffer pool中的脏page都是已提交事务；
2. 扫描buffer pool中的flush list，同步刷下脏块；
3. 记录数据字典信息到cfg文件，用于目标端的表结构匹配和验证，最后在目标端import的时候，变更page的space，max_lsn等。

**2. drop table t**
在对表进行删除的时候，需要清理掉buffer pool中的page，但如果表比较大，占用过多的buffer pool，清理的动作会影响到在线的业务，所以MySQL提供了lazy drop table的方式。

1. 同步方式: 扫描lru链表，如果page属于t表，就从lru链表，hash表， flush list中删除，回收block到free list中。
2. lazy方式: 扫描lru链表，如果page属于t表，就给page设置一个`space_was_being_deleted`属性，等lru置换或者checkpoint flush dirty block的时候进行清理。

**3. alter table t rename to t1**
rename table name操作，虽然是DDL，但rename操作只是变更了数据字典中的table name和文件系统的ibd文件名称，所以，在rename的过程中，不存在对buffer pool中属于t表的page的同步操作，但由于要变更表名，即需要同步对文件的IO操作。

而今天要讲的主题，就发生在这里，由于rename table进行IO操作同步的过程中，产生的死锁。

## 问题现象

在MySQL 5.5版本上，error日志大量报出以下的错误信息:

`InnoDB: fil_sys open file LRU len 0
InnoDB: Warning: too many (300) files stay open while the maximum
InnoDB: allowed value would be 300.
InnoDB: You may need to raise the value of innodb_open_files in my.cnf.
...
InnoDB: Warning: problems renaming 'db_1/#sql-xxx_xxx' to 'db_1/xxx', 1000 iterations
InnoDB: Warning: tablespace './db_1/#sql-xxx_xxx.ibd' has i/o ops stopped for a long time 1000
`

查看操作日志，是一个普通的rename语句操作，但持续很久，因为rename只是数据字典的变更，除了MDL锁阻塞以外
不应该持续这么长时间，pstack查看线程栈信息：

`Thread 5 (Thread 0x50ad7940 (LWP 25047)):
#0 0x000000364aacced2 in select () from /lib64/libc.so.6
#1 0x00002aaab2e595fb in os_thread_sleep ()
#2 0x00002aaab2e1a3e2 in fil_rename_tablespace ()
#3 0x00002aaab2e0672b in dict_table_rename_in_cache ()
#4 0x00002aaab2e86af5 in row_rename_table_for_mysql ()
#5 0x00002aaab2e316db in ha_innodb::rename_table ()
#6 0x00000000006bea6c in mysql_rename_table ()
#7 0x00000000006c77ff in mysql_alter_table ()
#8 0x00000000005c6a8e in mysql_execute_command ()
#9 0x00000000005cd371 in mysql_parse ()
#10 0x00000000005cd773 in dispatch_command ()
#11 0x00000000005cea04 in do_command ()
#12 0x00000000005bf0d7 in handle_one_connection ()
#13 0x000000364b6064a7 in start_thread () from /lib64/libpthread.so.0
#14 0x000000364aad3c2d in clone () from /lib64/libc.so.6

Thread 100 (Thread 0x42945940 (LWP 3870)):
#0 0x000000364b60ab99 in pthread_cond_wait@@GLIBC_2.3.2 ()
#1 0x00002aaab2e589a5 in os_event_wait_low ()
#2 0x00002aaab2e57dd4 in os_aio_simulated_handle ()
#3 0x00002aaab2e14ccc in fil_aio_wait ()
#4 0x00002aaab2ea2418 in io_handler_thread ()
#5 0x000000364b6064a7 in start_thread () from /lib64/libpthread.so.0
#6 0x000000364aad3c2d in clone () from /lib64/libc.so.6

Thread 120 (Thread 0x40da6940 (LWP 3882)):
#0 0x000000364aacced2 in select () from /lib64/libc.so.6
#1 0x00002aaab2e595fb in os_thread_sleep ()
#2 0x00002aaab2e18838 in fil_mutex_enter_and_prepare_for_io ()
#3 0x00002aaab2e18aa5 in fil_io ()
#4 0x00002aaab2df5b63 in buf_flush_buffered_writes ()
#5 0x00002aaab2df6048 in buf_flush_batch ()
#6 0x00002aaab2ea13d8 in srv_master_thread ()
#7 0x000000364b6064a7 in start_thread () from /lib64/libpthread.so.0
#8 0x000000364aad3c2d in clone () from /lib64/libc.so.6
`

这里我只列了有意义的三个线程：

1. 用户线程Thread 5
用户线程确实在进行rename操作，但阻塞在`fil_rename_tablespace`函数中。
2. master线程Thread 120
InnoDB的master线程阻塞在`fil_mutex_enter_and_prepare_for_io`函数中。
3. IO线程Thread 100
InnoDB的IO线程一共有8个，4个读，4个写线程，发现都在`os_event_wait_low`中，也就是都空闲着等待condition中。

从上面的调用栈来看，线程之间长时间维持在这种状态下，明显发生了死锁，在我们解这个死锁之前，我们先来回顾一点背景知识，然后再说明死锁的真正原因。

## InnoDB背景

### checkpoint

由于对数据库的数据操作也遵循read-update-write的方式，所以数据的更新，会把buffer pool中的page变成脏块，由于write-ahead logs机制保证事务的完整性，脏块的write可以变成异步的，但又由于buffer pool的大小终究有限，而且对于recovery的时间的要求，又要求脏块的flush又要持续保证。

MySQL 5.5的版本由master thread来承担dirty flush的角色， dirty flush的过程就称为making checkpoint，lsn的推进保证了recovery的时间不被持续的变长。刷新的策略，受到当前IO pending的情况，double write-buffer是否打开，buffer pool中dirty page所占的比例，以及`innodb_max_dirty_pages_pct`参数的设置，进行灵活刷新，具体的代码细节，这里就不展开了。

### 异步IO

由于dirty flush是异步的，所以，master thread只负责提交IO请求，真正的IO操作是由IO helper thread来完成的。InnoDB使用的simulate AIO和native AIO会有一些差别，我们这里以simulate AIO为例进行说明。假设double write-buffer是打开的：

1. 首先master thread搜集dirty pages，同步写入double write-buffer；
2. 由于double write-buffer的方式是buffered write，所以等double write-buffer写满了之后；
3. 同步把double write-buffer的page顺序写入到ibdata系统表空间中，如果完成之后系统crash，可以使用持久化的double write-buffer进行page恢复；
4. 开始把 double write-buffer中的page，写入真正的ibd文件中。依次提交异步IO操作，提交IO操作的步骤分为：
 * 持有fil_system mutex，判断当前tablespace是否可用，
* 判断当前fil_space的stop_io标示，如果设置就循环等待
* 如果stop_io没有标示，就打开fil_space对应的ibd文件句柄，然后递增 fil_space->n_pending
* 提交IO请求
5. 等double write-buffer中的pages提交完所有的IO请求，使用`os_aio_simulated_wake_handler_threads`来唤醒IO helper thread来完成IO操作。

### Rename 操作

接下来我们来看下rename操作的步骤：

1. 首先在server层hold MDL锁；
2. 进入InnoDB层，首先使用自治事务变更数据字典，包括SYS_TABLES，SYS_FOREIGN；
3. 变更数据字典的内存对象，包括table, index, foreign list等；
4. 变更fil_space对象以及对应的ibd数据文件名称，其中变更文件系统名称的时候：
 * 设置当前的fil_space的stop_io，阻止再进行IO操作
* 判断当前是否有IO pending，如果有，就等IO pending结束
* 如果没有IO pending，就关闭opened的句柄，并rename文件名称
* 恢复stop_io标示
5. 提交自治事务。

有了这些操作的具体步骤，我们就可以清晰的分析出死锁的原因。

## 死锁原因

两个线程，一个是master thread，需要提交flush dirty block的异步IO请求；一个是user thread，需要进行rename操作。

Rename操作，只变更数据字典和ibd文件名，并不需要同步buffer pool中的page，唯一需要同步的就是IO操作，通俗一点说，也就是在user thread进行rename table需要变更ibd文件名的时候，其它线程暂时不要对这个文件进行IO操作，等rename完成后，可以重新打开这个ibd文件，接着进行IO操作。

InnoDB使用两个标识来进行IO同步操作，即stop_io，n_pending。
**stop_io**：user thread要进行rename操作，提前设置这个标识，表示IO操作可以先hold暂停。
**n_pending**：master thread要进行flush操作，我已经提交了IO请求，user thread要进行rename可以先hold，等IO完成。

假设下面的时序：

1. master thread提交了1个IO请求，设置了n_pending；
2. rename操作设置stop_io，判断n_pending>0 就等待；
3. master thread需要提交剩下的几个IO，发现stop_io已设置，就等待；
4. 由于master thread没有提交完这批IO，没有唤醒IO helper thread，导致第1个IO请求无法完成，n_pending一直等于1；
5. rename操作因为n_pending一直等于1，陷入了死等；
6. master thread发现stop_io等于true，陷入了死等。

具体的代码可以参考：

**1. master thread**
fil0fil.cc: `fil_mutex_enter_and_prepare_for_io`

`space = fil_space_get_by_id(space_id);
if (space != NULL && space->stop_ios) {
 /* We are going to do a rename file and want to stop new i/o's for a while */
 if (count2 > 20000) {
 fputs("InnoDB: Warning: tablespace ", stderr);
 ut_print_filename(stderr, space->name);
 fprintf(stderr,
 " has i/o ops stopped for a long time %lu\n",
 (ulong) count2);
 }
 mutex_exit(&fil_system->mutex);
 os_thread_sleep(20000);
 count2++;
 goto retry;
}
`

**2. user thread**
fil0fil.cc: `fil_rename_tablespace`

`/* We temporarily close the .ibd file because we do not trust that
operating systems can rename an open file. For the closing we have to
wait until there are no pending i/o's or flushes on the file. */

space->stop_ios = TRUE;
ut_a(UT_LIST_GET_LEN(space->chain) == 1);
node = UT_LIST_GET_FIRST(space->chain);
if (node->n_pending > 0 || node->n_pending_flushes > 0) {
 /* There are pending i/o's or flushes, sleep for a while and retry */
 mutex_exit(&fil_system->mutex);
 os_thread_sleep(20000);
 goto retry;
`

## 修复方法

修复的方法也比较简单，在`fil_rename_tablespace`的时候，如果发现node->n_pending > 0的时候，在sleep之前，发起一次唤醒动作，即`os_aio_simulated_wake_handler_threads`，IO helper thread去完成master thread已经提交的IO请求，这样n_pending就会降到0，死锁就解开了。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)