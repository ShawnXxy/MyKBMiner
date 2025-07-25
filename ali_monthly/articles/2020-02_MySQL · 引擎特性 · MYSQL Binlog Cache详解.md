# MySQL · 引擎特性 · MYSQL Binlog Cache详解

**Date:** 2020/02
**Source:** http://mysql.taobao.org/monthly/2020/02/06/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2020 / 02
 ](/monthly/2020/02)

 * 当期文章

 MySQL · 引擎特性 · 庖丁解InnoDB之REDO LOG
* MySQL · 引擎特性 · InnoDB Buffer Pool 浅析
* MySQL · 最佳实践 · RDS 三节点企业版热点组提交
* MySQL · 引擎特性 · 8.0 heap table 介绍
* MySQL · 存储引擎 · MySQL的字段数据存储格式
* MySQL · 引擎特性 · MYSQL Binlog Cache详解

 ## MySQL · 引擎特性 · MYSQL Binlog Cache详解 
 Author: jiangyi 

 ### MYSQL Binlog Cache详解

最近在线上遇到一个突发情况：某客户出现了超大事务，该事务运行时占据的磁盘空间超过800GB，但du -sh时未发现任何线索。于是刨根溯源，找到了最终的原因并紧急处理了该问题。本文便是对该问题涉及的binlog cache知识进行整理，希望也能造福更多的朋友。本文会涉及到如下几个概念：

* binlog cache：它是用于缓存binlog event的内存，大小由binlog_cache_size控制
* binlog cache 临时文件：是一个临时磁盘文件，存储由于binlog cache不足溢出的binlog event，该文件名字由”ML”打头，由参数*max_binlog_cache_size*控制该文件大小
* binlog file：代表binglog 文件，由*max_binlog_size*指定大小
* binlog event：代表binlog中的记录，如MAP_EVENT/QUERY EVENT/XID EVENT/WRITE EVENT等

#### 事务binlog event写入流程

binlog cache和binlog临时文件都是在事务运行过程中写入，一旦事务提交，binlog cache和binlog临时文件都会释放掉。而且如果事务中包含多个DML语句，他们共享binlog cache和binlog 临时文件。整个binlog写入流程：

1. 事务开启
2. 执行dml语句，在dml语句第一次执行的时候会分配内存空间binlog cache
3. 执行dml语句期间生成的event不断写入到binlog cache
4. 如果binlog cache的空间已经满了，则将binlog cache的数据写入到binlog临时文件，同时清空binlog cache。如果binlog临时文件的大小大于了max_binlog_cache_size的设置则抛错ERROR 1197
5. 事务提交，整个binlog cache和binlog临时文件数据全部写入到binlog file中，同时释放binlog cache和binlog临时文件。但是注意此时binlog cache的内存空间会被保留以供THD上的下一个事务使用，但是binlog临时文件被截断为0，保留文件描述符。其实也就是IO_CACHE(参考后文)保留，并且保留IO_CACHE中的分配的内存空间，和物理文件描述符
6. 客户端断开连接，这个过程会释放IO_CACHE同时释放其持有的binlog cache内存空间以及持有的binlog 临时文件。
本文主要关注步骤3和4过程中对binlog cache以及binlog 临时文件的写入细节。

#### 数据结构

**binlog_cache_mngr**

这个类中包含了两个cache：binlog cache和binlog stmt cache。同时包含了将binlog event flush到binlog file的方法。

**binlog_trx_cache_data**

暂时不表

**Binlog_cache_storage**

暂时不表

**IO_CACHE_binlog_cache_storage**

暂时不表

**IO_CACHE**

将binlog event写入到binlog cache 或者 binlog临时文件都是由 IO_CACHE子系统实现的。IO_CACHE子系统实现了写缓存以及在缓存不足时写入物理文件的功能。它包含读缓存，写缓存以及访问物理文件等信息。其维护的核心成员有：

* 读缓存 uchar *buffer;
* 写缓存 uchar *write_buffer;
* 物理文件 File file;

同时IO_CACHE也支持多种访问模式如READ_CACHE/WRITE_CACHE/SEQ_READ_APPEND，这里就暂时不表。

#### binlog_cache_size & max_binlog_cache_size
如果开启binlog，那么binlog_cache_size用来在事务运行期间在内存中缓存binlog event。如果经常使用大事务应该加大这个缓存，避免过多的磁盘使用影响性能。

当binlog_cache_size不足以容纳所有的binlog event时，便转而使用临时文件来缓存binlog event。从Binlog_cache_use和Binlog_cache_disk_use可以看出是否使用了binlog cache或binlog 临时文件用于保存binlog event。

#### binlog cache创建
事务开启时，如果开启binlog功能，便会创建binlog cache。

`void *handler_create_thd(
 bool enable_binlog) /*!< in: whether to enable binlog */
{
 ...
 if (enable_binlog) {
 thd->binlog_setup_trx_data();
 }
 return (thd);
}

int THD::binlog_setup_trx_data() {
 binlog_cache_mngr *cache_mngr = thd_get_cache_mngr(this);
 cache_mngr = (binlog_cache_mngr *)my_malloc(key_memory_binlog_cache_mngr,
 sizeof(binlog_cache_mngr),
 MYF(MY_ZEROFILL));
 cache_mngr = new (cache_mngr)
 binlog_cache_mngr(&binlog_stmt_cache_use, &binlog_stmt_cache_disk_use,
 &binlog_cache_use, &binlog_cache_disk_use);
 if (cache_mngr->init()) {
 ...
 }
}

class binlog_cache_mngr {
 public:
 bool init() {
 return stmt_cache.open(binlog_stmt_cache_size,
 max_binlog_stmt_cache_size) ||
 trx_cache.open(binlog_cache_size, max_binlog_cache_size);
 }
}

class binlog_cache_data {
 public:
 bool open(my_off_t cache_size, my_off_t max_cache_size) {
 return m_cache.open(cache_size, max_cache_size);
 }
}

// 分配binlog cache内存缓存空间以及创建临时文件
// 最终是进入了函数init_io_cache_ext处理，暂且不表
bool Binlog_cache_storage::open(my_off_t cache_size, my_off_t max_cache_size) {
 const char *LOG_PREFIX = "ML";
 if (m_file.open(mysql_tmpdir, LOG_PREFIX, cache_size, max_cache_size))
 return true;
 m_pipeline_head = &m_file;
 return false;
}
`
binlog临时文件会被存放到tmpdir的目录下，并以”ML”作为文件名开头。但该文件无法用ls命令看到，因为使用了LINUX创建临时API（mkstemp），以避免其他进程破坏文件内容。也就是说，这个文件是mysqld进程内部专用的，我们在后面会给出访问该文件的方法。

#### binlog写入cache和临时文件
binlog event写入binlog cache和临时文件是通过函数_my_b_write进行的：

`bool IO_CACHE_binlog_cache_storage::write(const unsigned char *buffer,
 my_off_t length) {
 return my_b_safe_write(&m_io_cache, buffer, length);
}

int my_b_safe_write(IO_CACHE *info, const uchar *Buffer, size_t Count) {
 if (info->type == SEQ_READ_APPEND) return my_b_append(info, Buffer, Count);
 return my_b_write(info, Buffer, Count);
}

// 如果binlog cache缓存当前写入的位置加上本次写入的总量大于了binlog cache的内存地址的边界
// 则我们需要进行通过*(info)->write_function将binlog cache的内容写到磁盘了
// 这样才能腾出空间给新的binlog event存放。这个回调函数就是_my_b_write。
#define my_b_write(info, Buffer, Count) \
 ((info)->write_pos + (Count) <= (info)->write_end \
 ? (memcpy((info)->write_pos, (Buffer), (size_t)(Count)), \
 ((info)->write_pos += (Count)), 0) \
 : (*(info)->write_function)((info), (uchar *)(Buffer), (Count)))

int _my_b_write(IO_CACHE *info, const uchar *Buffer, size_t Count) {
 size_t rest_length, length;
 my_off_t pos_in_file = info->pos_in_file;
 // 如果超过临时文件大小设置,则报错
 if (pos_in_file + info->buffer_length > info->end_of_file) {
 errno = EFBIG;
 set_my_errno(EFBIG);
 return info->error = -1;
 }

 // 首先将binlog内容拷贝至内存cache,将cache填满
 rest_length = (size_t)(info->write_end - info->write_pos);
 memcpy(info->write_pos, Buffer, (size_t)rest_length);
 Buffer += rest_length;
 Count -= rest_length;
 info->write_pos += rest_length;

 if (my_b_flush_io_cache(info, 1)) return 1;
 if (Count >= IO_SIZE) { /* Fill first intern buffer */
 length = Count & (size_t) ~(IO_SIZE - 1);
 ...
 if (mysql_file_write(info->file, Buffer, length, info->myflags | MY_NABP))
 return info->error = -1;
 ...
 Count -= length;
 Buffer += length;
 info->pos_in_file += length;
 }
 memcpy(info->write_pos, Buffer, (size_t)Count);
 info->write_pos += Count;
 return 0;
}
`

#### 运维技巧：查看binlog 临时文件

因为没法直接通过ls来查看binlog临时缓存文件，但可以使用lsof|grep delete来观察到这种文件

```
[root@test ~]# lsof|grep delete|grep ML
mysqld 21414 root 77u REG 252,3 65536 1856092 /var/tmp/mysqld.1/MLUFzokf

```

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)