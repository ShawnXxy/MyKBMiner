# MySQL · 源码分析 · InnoDB 异步IO工作流程

**Date:** 2017/07
**Source:** http://mysql.taobao.org/monthly/2017/07/10/
**Images:** 2 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 07
 ](/monthly/2017/07)

 * 当期文章

 MySQL · 引擎特性 · InnoDB崩溃恢复
* PgSQL · 应用案例 · 阿里云RDS金融数据库(三节点版) - 背景篇
* AliSQL · 特性介绍 · 支持 Invisible Indexes
* TokuDB · 引擎特性 · HybridDB for MySQL高压缩引擎TokuDB 揭秘
* MySQL · myrocks · myrocks写入分析
* MSSQL · 实现分析 · Extend Event实现审计日志对SQL Server性能影响
* HybridDB · 源码分析 · MemoryContext 内存管理和内存异常分析
* MySQL · 实现分析 · HybridDB for MySQL 数据压缩
* PgSQL · 最佳实践 · CPU满问题处理
* MySQL · 源码分析 · InnoDB 异步IO工作流程

 ## MySQL · 源码分析 · InnoDB 异步IO工作流程 
 Author: 令猴 

 之前的一篇内核月报[InnoDB IO子系统](http://mysql.taobao.org/monthly/2017/03/01/) 中介绍了InnoDB IO子系统中包含的同步IO以及异步IO。本篇文章将从源码层面剖析一下InnoDB IO子系统中，数据页的同步IO以及异步IO请求的具体实现过程。

在MySQL5.6中，InnoDB的异步IO主要是用来处理预读以及对数据文件的写请求的。而对于正常的页面数据读取则是通过同步IO进行的。到底二者在代码层面上的实现过程有什么样的区别？ 接下来我们将以Linux native io的执行过程为主线，对IO请求的执行过程进行梳理。

## 重点数据结构

* os_aio_array_t

`/** 用来记录某一类（ibuf,log,read,write）异步IO（aio）请求的数组类型。每一个异步IO请求都会在类型对应的数组中注册一个innodb
 aio对象。*/

os_aio_array_t {
 
 os_ib_mutex_t mutex; // 主要用来控制异步read/write线程的并发操作。对于ibuf,log类型，由于只有一个线程，所以不存在并发操作问题
 os_event_t not_full; // 一个条件变量event，用来通知等待获取slot的线程是否os_aio_array_t数组有空闲的slot供aio请求

 os_event_t is_empty; // 条件变量event，用来通知IO线程os_aio_array_t数组是否有pening的IO请求。
 
 ulint n_slots; // 数组容纳的IO请求数。= 线程数 * 每个segment允许pending的请求数（256）
 
 ulint n_segments; // 允许独立wait的segment数，即某种类型的IO的允许最大线程数

 ulint cur_seg; /* IO请求会按照round robin的方式分配到不同的segment中，该变量指示下一个IO请求可以分配的segment */
 ulint n_reserved; // 已经Pending的IO请求数

 os_aio_slot_t* slots; // 用来记录具体的每个IO请求对象的数组，也即n_segments 个线程共用n_slots个槽位来存放pending io请求 

 \#ifdef __WIN__

 HANDLE* handles;
 /*!< Pointer to an array of OS native
 event handles where we copied the
 handles from slots, in the same
 order. This can be used in
 WaitForMultipleObjects; used only in
 Windows */

 \#endif __WIN__

 \#if defined(LINUX_NATIVE_AIO)

 io_context_t* aio_ctx; // aio上下文的数组，每个segment拥有独立的一个aio上下文数组，用来记录以及完成的IO请求上下文

 struct io_event* aio_events; // 该数组用来记录已经完成的IO请求事件。异步IO通过设置事件通知IO线程处理完成的IO请求

 struct iocb** pending; // 用来记录pending的aio请求

 ulint* count; // 该数组记录了每个segment对应的pending aio请求数量

 \#endif /* LINUX_NATIV_AIO */

 }
`

* os_aio_slot_t

```
// os_aio_array_t数组中用来记录一个异步IO(aio)请求的对象
 os_aio_slot_t {

 ibool is_read; /*!< TRUE if a read operation */

 ulint pos; // os_aio_array_t数组中所在的位置 

 ibool reserved; // TRUE表示该Slot已经被别的IO请求占用了

 time_t reservation_time; // 占用的时间

 ulint len; // io请求的长度

 byte* buf; // 数据读取或者需要写入的buffer，通常指向buffer pool的一个页面，压缩页面有特殊处理

 ulint type; /* 请求类型，即读还是写IO请求 */ 

 os_offset_t offset; /*!< file offset in bytes */

 os_file_t file; /*!< file where to read or write */

 const char* name; /*!< 需要读取的文件及路径信息 */

 ibool io_already_done; /* TRUE表示IO已经完成了

 fil_node_t* message1; /* 该aio操作的innodb文件描述符（f_node_t）*/

 void* message2; /* 用来记录完成IO请求所对应的具体buffer pool bpage页 */

 \#ifdef WIN_ASYNC_IO

 HANDLE handle; /*!< handle object we need in the
 OVERLAPPED struct */

 OVERLAPPED control; /*!< Windows control block for the
 aio request */

 \#elif defined(LINUX_NATIVE_AIO)

 struct iocb control; /* 该slot使用的aio请求控制块iocb */

 int n_bytes; /* 读写bytes */

 int ret; /* AIO return code */

 \#endif /* WIN_ASYNC_IO */

}

```

## 流程图

![flow-aio.png](.img/fdbdc2e396d7_7d6c30f8fddb081537559589b4bd2508.png)

## 源码分析

* 物理数据页操作入口函数os_aio_func

`ibool
os_aio_func(
/*========*/
 ulint type, /* IO类型，READ还是WRITE IO */
 ulint mode, /* 这里表示是否使用SIMULATED aio执行异步IO请求 */
 const char* name, /* IO需要打开的tablespace路径+名称 */
 os_file_t file, /* IO操作的文件 */
 void* buf, // 数据读取或者需要写入的buffer，通常指向buffer pool的一个页面，压缩页面有特殊处理
 os_offset_t offset, /*!< in: file offset where to read or write */
 ulint n, /* 读取或写入字节数 */
 fil_node_t* message1, /* 该aio操作的innodb文件描述符（f_node_t），只对异步IO起作用 */ 
 void* message2, /* 用来记录完成IO请求所对应的具体buffer pool bpage页，只对异步IO起作用 */
 ibool should_buffer, // 是否需要缓存aio请求，该变量主要对预读起作用
 ibool page_encrypt,
 /*!< in: Whether to encrypt */
 ulint page_size)
 /*!< in: Page size */
{
...

 wake_later = mode & OS_AIO_SIMULATED_WAKE_LATER;
 mode = mode & (~OS_AIO_SIMULATED_WAKE_LATER);

 if (mode == OS_AIO_SYNC
#ifdef WIN_ASYNC_IO
 && !srv_use_native_aio
#endif /* WIN_ASYNC_IO */
 ) {
 /* This is actually an ordinary synchronous read or write:
 no need to use an i/o-handler thread. NOTE that if we use
 Windows async i/o, Windows does not allow us to use
 ordinary synchronous os_file_read etc. on the same file,
 therefore we have built a special mechanism for synchronous
 wait in the Windows case.
 Also note that the Performance Schema instrumentation has
 been performed by current os_aio_func()'s wrapper function
 pfs_os_aio_func(). So we would no longer need to call
 Performance Schema instrumented os_file_read() and
 os_file_write(). Instead, we should use os_file_read_func()
 and os_file_write_func() */

 /* 这里如果是同步IO，并且native io没有开启的情况下，直接使用os_file_read/write函数进行读取，
 不需要经过IO线程进行处理 */

 if (type == OS_FILE_READ) {
 if (page_encrypt) {
 return(os_file_read_decrypt_page(file, buf, offset, n, page_size));
 } else {
 return(os_file_read_func(file, buf, offset, n));
 }
 }
 ut_ad(!srv_read_only_mode);
 ut_a(type == OS_FILE_WRITE);
 if (page_encrypt) {
 return(os_file_write_encrypt_page(name, file, buf, offset, n, page_size));
 } else {
 return(os_file_write_func(name, file, buf, offset, n));
 }
 }
try_again:
 switch (mode) {
 // 根据访问类型，定位IO请求数组
 case OS_AIO_NORMAL:
 if (type == OS_FILE_READ) {
 array = os_aio_read_array;
 } else {
 ut_ad(!srv_read_only_mode);
 array = os_aio_write_array;
 }
 break;
 case OS_AIO_IBUF:
 ut_ad(type == OS_FILE_READ);
 /* Reduce probability of deadlock bugs in connection with ibuf:
 do not let the ibuf i/o handler sleep */

 wake_later = FALSE;

 if (srv_read_only_mode) {
 array = os_aio_read_array;
 }
 break;
 case OS_AIO_LOG:
 if (srv_read_only_mode) {
 array = os_aio_read_array;
 } else {
 array = os_aio_log_array;
 }
 break;
 case OS_AIO_SYNC:
 array = os_aio_sync_array;
#if defined(LINUX_NATIVE_AIO)
 /* In Linux native AIO we don't use sync IO array. */
 ut_a(!srv_use_native_aio);
#endif /* LINUX_NATIVE_AIO */
 break;
 default:
 ut_error;
 array = NULL; /* Eliminate compiler warning */
 }
 // 阻塞为当前IO请求申请一个用来执行异步IO的slot
 slot = os_aio_array_reserve_slot(type, array, message1, message2, file,
 name, buf, offset, n, page_encrypt, page_size);

 DBUG_EXECUTE_IF("simulate_slow_aio",
 {
 os_thread_sleep(1000000);
 }
 );
 if (type == OS_FILE_READ) {
 if (srv_use_native_aio) {
 os_n_file_reads++;
 os_bytes_read_since_printout += n;
#ifdef WIN_ASYNC_IO
 // 这里是Windows用来处理异步IO读请求
 ret = ReadFile(file, buf, (DWORD) n, &len,
 &(slot->control));

#elif defined(LINUX_NATIVE_AIO)
 // 这里是Linux来处理native io
 if (!os_aio_linux_dispatch(array, slot, should_buffer)) {
 goto err_exit;
#endif /* WIN_ASYNC_IO */
 } else {
 if (!wake_later) {
 // 唤醒simulated aio处理线程
 os_aio_simulated_wake_handler_thread(
 os_aio_get_segment_no_from_slot(
 array, slot));
 }
 }
 } else if (type == OS_FILE_WRITE) {
 ut_ad(!srv_read_only_mode);
 if (srv_use_native_aio) {
 os_n_file_writes++;
#ifdef WIN_ASYNC_IO
 // 这里是Windows用来处理异步IO写请求
 ret = WriteFile(file, buf, (DWORD) n, &len,
 &(slot->control));

#elif defined(LINUX_NATIVE_AIO)
 // 这里是Linux来处理native io
 if (!os_aio_linux_dispatch(array, slot, false)) {
 goto err_exit;
 }
#endif /* WIN_ASYNC_IO */
 } else {
 if (!wake_later) {
 // 唤醒simulated aio处理线程
 os_aio_simulated_wake_handler_thread(
 os_aio_get_segment_no_from_slot(
 array, slot));
 }
 }
 } else {
 ut_error;
 }

...
}

`

* 负责通知Linux内核执行native IO请求的函数os_aio_linux_dispatch

```
static
ibool
os_aio_linux_dispatch(
/*==================*/
 os_aio_array_t* array, /* IO请求函数 */
 os_aio_slot_t* slot, /* 申请好的slot */
 ibool should_buffer) // 是否需要缓存aio 请求，该变量主要对预读起作用
{
 ...

 /* Find out what we are going to work with.
 The iocb struct is directly in the slot.
 The io_context is one per segment. */

 // 每个segment包含的slot个数，Linux下每个segment包含256个slot
 slots_per_segment = array->n_slots / array->n_segments;
 iocb = &slot->control;
 io_ctx_index = slot->pos / slots_per_segment;
 if (should_buffer) {
 /* 这里也可以看到aio请求缓存只对读请求起作用 */
 ut_ad(array == os_aio_read_array);
 
 ulint n;
 ulint count;
 os_mutex_enter(array->mutex);
 /* There are array->n_slots elements in array->pending, which is divided into
 * array->n_segments area of equal size. The iocb of each segment are 
 * buffered in its corresponding area in the pending array consecutively as
 * they come. array->count[i] records the number of buffered aio requests in
 * the ith segment.*/
 n = io_ctx_index * slots_per_segment
 + array->count[io_ctx_index];
 array->pending[n] = iocb;
 array->count[io_ctx_index] ++; 
 count = array->count[io_ctx_index];
 os_mutex_exit(array->mutex);
 // 如果当前segment的slot都已经被占用了，就需要提交一次异步aio请求
 if (count == slots_per_segment) {
 os_aio_linux_dispatch_read_array_submit(); //no cover line
 } 
 // 否则就直接返回
 return (TRUE); 
 } 
 // 直接提交IO请求到内核
 ret = io_submit(array->aio_ctx[io_ctx_index], 1, &iocb);
 ...
}

```

* IO线程负责监控aio请求的主函数fil_aio_wait

```
void
fil_aio_wait(
/*=========*/
 ulint segment) /*!< in: the number of the segment in the aio
 array to wait for */
{
 ibool ret; 
 fil_node_t* fil_node;
 void* message;
 ulint type;

 ut_ad(fil_validate_skip());

 if (srv_use_native_aio) { // 使用native io
 srv_set_io_thread_op_info(segment, "native aio handle");
#ifdef WIN_ASYNC_IO
 ret = os_aio_windows_handle( // Window监控入口
 segment, 0, &fil_node, &message, &type);
#elif defined(LINUX_NATIVE_AIO)
 ret = os_aio_linux_handle( // Linux native io监控入口
 segment, &fil_node, &message, &type);
#else
 ut_error;
 ret = 0; /* Eliminate compiler warning */
#endif /* WIN_ASYNC_IO */
 } else {
 srv_set_io_thread_op_info(segment, "simulated aio handle");

 ret = os_aio_simulated_handle( // Simulated aio监控入口
 segment, &fil_node, &message, &type);
 }

 ut_a(ret);
 if (fil_node == NULL) {
 ut_ad(srv_shutdown_state == SRV_SHUTDOWN_EXIT_THREADS);
 return;
 }
 srv_set_io_thread_op_info(segment, "complete io for fil node");
 mutex_enter(&fil_system->mutex);

 // 到这里表示至少有一个IO请求已经完成，该函数设置状态信息
 fil_node_complete_io(fil_node, fil_system, type);

 mutex_exit(&fil_system->mutex);

 ut_ad(fil_validate_skip());

 /* Do the i/o handling */
 /* IMPORTANT: since i/o handling for reads will read also the insert
 buffer in tablespace 0, you have to be very careful not to introduce
 deadlocks in the i/o system. We keep tablespace 0 data files always
 open, and use a special i/o thread to serve insert buffer requests. */

 if (fil_node->space->purpose == FIL_TABLESPACE) { // 数据文件读写IO
 srv_set_io_thread_op_info(segment, "complete io for buf page");
 // IO请求完成后，这里处理buffer pool对应的bpage相关的一些状态信息并根据checksum验证数据的正确性
 buf_page_io_complete(static_cast<buf_page_t*>(message));
 } else { // 日志文件的读写IO
 srv_set_io_thread_op_info(segment, "complete io for log");
 log_io_complete(static_cast<log_group_t*>(message));
 }
}
#endif /* UNIV_HOTBACKUP */

```

* IO线程负责处理native IO请求的函数os_aio_linux_handle

```
ibool
os_aio_linux_handle(ulint global_seg, // 属于哪个segment
 fil_node_t**message1, /* 该aio操作的innodb文件描述符（f_node_t）*/
 void** message2, /* 用来记录完成IO请求所对应的具体buffer pool bpage页 */
 ulint* type){ // 读or写IO
 // 根据global_seg获得该aio 的os_aio_array_t数组，并返回对应的segment
 segment = os_aio_get_array_and_local_segment(&array, global_seg); 
 n = array->n_slots / array->n_segments; //获得一个线程可监控的io event数
 /* Loop until we have found a completed request. */
 for (;;) {
 ibool any_reserved = FALSE;
 os_mutex_enter(array->mutex);
 for (i = 0; i < n; ++i) { // 遍历该线程所发起的所有aio请求
 slot = os_aio_array_get_nth_slot(
 array, i + segment * n); 
 if (!slot->reserved) { // 该slot是否被占用
 continue;
 } else if (slot->io_already_done) { // IO请求已经完成，可以通知主线程返回数据了
 /* Something for us to work on. */
 goto found;
 } else {
 any_reserved = TRUE;
 }
 }
 os_mutex_exit(array->mutex);
 // 到这里说明没有找到一个完成的io，则再去collect
 os_aio_linux_collect(array, segment, n); 
found: // 找到一个完成的io，将内容返回
 *message1 = slot->message1; 
 *message2 = slot->message2; // 返回完成IO所对应的bpage页
 *type = slot->type;
 if (slot->ret == 0 && slot->n_bytes == (long) slot->len) {
 if (slot->page_encrypt
 && slot->type == OS_FILE_READ) {
 os_decrypt_page(slot->buf, slot->len, slot->page_size, FALSE);
 } 

 ret = TRUE;
 } else {
 errno = -slot->ret;
 /* os_file_handle_error does tell us if we should retry
 this IO. As it stands now, we don't do this retry when
 reaping requests from a different context than
 the dispatcher. This non-retry logic is the same for
 windows and linux native AIO.
 We should probably look into this to transparently
 re-submit the IO. */
 os_file_handle_error(slot->name, "Linux aio");

 ret = FALSE;
 }

 os_mutex_exit(array->mutex);

 os_aio_array_free_slot(array, slot);
 return(ret);
}

```

* 等待native IO请求完成os_aio_linux_collect

```
os_aio_linux_collect(os_aio_array_t* array,
 ulint segment, 
 ulint seg_size){
 events = &array->aio_events[segment * seg_size]; // 定位segment所对应的io event的数组位置
 /* 获得该线程的aio上下文数组 */
 io_ctx = array->aio_ctx[segment];
 /* Starting point of the segment we will be working on. */
 start_pos = segment * seg_size;
 /* End point. */
 end_pos = start_pos + seg_size;

retry: 
 /* Initialize the events. The timeout value is arbitrary.
 We probably need to experiment with it a little. */
 memset(events, 0, sizeof(*events) * seg_size);
 timeout.tv_sec = 0;
 timeout.tv_nsec = OS_AIO_REAP_TIMEOUT;

 ret = io_getevents(io_ctx, 1, seg_size, events, &timeout); // 阻塞等待该IO线程所监控的任一IO请求完成

 if (ret > 0) { // 有IO请求完成
 for (i = 0; i < ret; i++) {
 // 记录完成IO的请求信息到对应的os_aio_slot_t 对象
 os_aio_slot_t* slot;
 struct iocb* control;
 control = (struct iocb*) events[i].obj; // 获得完成的aio的iocb，即提交这个aio请求的iocb
 ut_a(control != NULL);
 slot = (os_aio_slot_t*) control->data; // 通过data获得这个aio iocb所对应的os_aio_slot_t
 /* Some sanity checks. */
 ut_a(slot != NULL);
 ut_a(slot->reserved);
 os_mutex_enter(array->mutex);
 slot->n_bytes = events[i].res; // 将该io执行的结果保存到slot里
 slot->ret = events[i].res2;
 slot->io_already_done = TRUE; // 标志该io已经完成了，这个标志也是外层判断的条件
 os_mutex_exit(array->mutex);
 }
 return;
 }
…
}

```

综上重点对InnoDB navtive IO读写数据文件从源码角度进行了分析，有兴趣的读者也可以继续了解InnoDB自带的simulated IO的实现过程，原理雷同native IO，只是在实现方式上自己进行了处理。本篇文章对InnoDB IO请求的执行流程进行了梳理，对重点数据结构以及函数进行了分析，希望对读者日后进行源码阅读及修改有所帮助。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)