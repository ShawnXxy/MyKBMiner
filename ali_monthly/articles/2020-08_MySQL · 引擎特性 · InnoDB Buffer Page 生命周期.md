# MySQL · 引擎特性 · InnoDB Buffer Page 生命周期

**Date:** 2020/08
**Source:** http://mysql.taobao.org/monthly/2020/08/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2020 / 08
 ](/monthly/2020/08)

 * 当期文章

 MySQL · 引擎特性 · truncate table在大buffer pool下的优化
* MySQL · 引擎特性 · INNODB UNDO LOG分配
* MySQL · 内核特性 · Redo Logging动态开关
* MySQL · 引擎特性 · InnoDB Buffer Page 生命周期
* MySQL · 引擎特性 · InnoDB UNDO LOG写入
* MySQL · 引擎特性 · InnoDB 数据文件简述
* Database · 案例分析 · UTF8与GBK数据库字符集

 ## MySQL · 引擎特性 · InnoDB Buffer Page 生命周期 
 Author: 翊云 

 ## 前言

InnoDB 没有使用操作系统自己的 Page Cache 机制，而是自己设计了一套 Buffer Pool 来进行 Page 的管理，关于 InnoDB Buffer Pool 的介绍，可以参考[这篇文章](http://mysql.taobao.org/monthly/2017/05/01/)，里面对 InnoDB Buffer Pool 作了比较深入的介绍。本文尝试从另外一个角度介绍一下一个 Buffer Page 的生命周期。本文给出的所有示例代码均基于 MySQL 8.0.18 版本。

## 申请

### Page 读取

Page 的读取有一个统一的入口函数 `buffer_page_get_gen` ，该方法的主要入参为 `page_id` ，即获取指定的页，MySQL 8.0 中的主要流程如下：

`/* 以 Buf_fetch_normal 为例 */
|--> fetch.single_page
| |--> get(block) // loop
| | |--> lookup
| | | |--> buf_page_hash_get_low // 检查 page_hash 中是否存在
| | |--> buf_block_fix // buf_fix_count 计数 +1
| | |
| | |--> read_page
| | | |--> buf_read_page // 从文件中读取 page
| | | | |--> buf_read_page_low
| | | | | |--> buf_page_init_for_read
| | | | | | |--> buf_LRU_get_free_block // 申请 1 个 block
| | | | | | |--> buf_page_hash_get_low // 再次检查 page_hash 中是否存在
| | | | | | |--> buf_page_init
| | | | | | | |--> buf_block_init_low
| | | | | | | |--> buf_page_init_low
| | | | | | | |--> HASH_INSERT // 插入 page_hash
| | | | | | |--> buf_page_set_io_fix // io_fix 设置为 BUF_IO_READ
| | | | | | |--> buf_LRU_add_block // 添加到 LRU
| | | | | |
| | | | | |--> _fil_io // 读取文件
| | | | | |--> buf_page_io_complete // 同步模式 IO 完成
| | | | | | |--> buf_page_set_io_fix // io_fix 设置为 BUF_IO_NONE
| |
| |--> buf_page_make_young_if_needed
| |
| |--> buf_read_ahead_linear
`

读取 1 个 Page 时，首先会检查 `page_hash` ，如果 `page_hash` 中存在，则直接读取并设置 `buf_fix_count` 后返回；否则需要从文件中读取 Page，从文件中读取 Page 时首先需要申请 1 个 Block（具体申请过程在后面介绍），然后添加到 `page_hash` 和 `LRU` 列表中，最后进行数据的读取。对于 1 个新的 Page 的创建过程，入口函数为 `buf_page_create` ，基本流程如下：

`/* buf_page_create */
|--> buf_page_create
| |--> buf_LRU_get_free_block // 申请 1 个 block
| |--> buf_page_hash_get_low // 检查 page_hash 中是否存在
| |
| |--> buf_page_init
| | |--> buf_block_init_low
| | |--> buf_page_init_low
| | |--> HASH_INSERT // 插入 page_hash
| |--> buf_block_buf_fix_inc // buf_fix_count 计数 +1
| |--> buf_LRU_add_block // 添加到 LRU
`

### Block 申请

Block 申请的入口函数为 `buf_LRU_get_free_block` ，该方法会从 Buffer Pool 中申请 1 个 Block 供后续的 Page 读取使用。Block 申请的主要流程如下：

`|--> buf_LRU_get_free_block // loop
| |--> buf_LRU_get_free_only // 从 free_list 分配
| |
| |--> buf_LRU_scan_and_free_block // 从 LRU 中回收
| | |--> buf_LRU_free_from_unzip_LRU_list
| | | |--> buf_LRU_free_page
| | |--> buf_LRU_free_from_common_LRU_list
| | | |--> buf_flush_ready_for_replace
| | | |--> buf_LRU_free_page
| |
| |--> os_event_set(buf_flush_event) // 唤醒刷脏线程
| |
| |--> buf_flush_single_page_from_LRU // 从 LRU 中刷脏
| | |--> buf_LRU_free_page
| | |--> buf_flush_page
`

Buffer Pool 中维护了三个列表：`free_list` 、`LRU` 、`flush_list` 。其中 `free_list` 列表是当前可供使用的 Block，`LRU` 列表中保存了当前所有已经使用的 Block，`flush_list` 列表中保存了所有脏页 Block。申请 1 个 Block 时：

1. 首先判断当前 `free_list` 列表是否为空，若 `free_list` 列表非空，则直接从 `free_list` 列表中进行分配。若无法直接从 `free_list` 列表分配，则会尝试从 `LRU` 列表中进行回收。
2. `LRU` 是一个非严格的最近使用列表，从 `LRU` 列表回收时会从列表尾部往前遍历（加入 `LRU` 列表时从头部加入），如果找到可回收的 Page（遇到脏页会跳过），则会释放 Page 并将对应的 Block 重新放入 `free_list` 列表中。`LRU` 列表的遍历过程并不是无限的，例如：在第一次遍历时，当检查的 Page 数目达到 `BUF_LRU_SEARCH_SCAN_THRESHOLD` 时会退出遍历过程。
3. 如果无法从 `LRU` 列表中回收 Block，则会唤醒刷脏线程，刷脏线程的处理流程在下面会做介绍。
4. 同时还会从 `LRU` 列表中进行刷脏，该过程是同步的，依然是遍历 `LRU` 列表，但此时不会跳过脏页，遇到脏页直接进行刷脏。

## 管理

### 加入 flush_list

前面提到过 `flush_list` 列表中保存的是所有脏页 Block，脏页在 mtr 提交时会加入 `flush_list` 中，基本过程如下：

`|--> mtr_t::Command::execute()
| |--> add_dirty_page_to_flush_list
| | |--> buf_flush_note_modification
| | | |--> buf_flush_insert_into_flush_list
| | | | |--> UT_LIST_ADD_FIRST // 插入 flush_list 头部
`

注意：`flush_list` 是一个非严格有序的列表（可以看做按照 `oldest_modification` 有序），脏页插入列表后位置不再修改，再次修改时仅修改 `newest_modification` 。

### 加入 LRU

`LRU` 列表中保存了当前所有已经使用的 Block，申请完 1 个 Block 并完成初始化后会加到 `LRU` 列表中（默认会加入到 old 区域头部），加入 `LRU` 列表的基本过程如下：

`|--> buf_LRU_add_block
| |--> buf_LRU_add_block_low
| | |--> UT_LIST_ADD_FIRST // 插入 young 区域头部
| | |--> UT_LIST_INSERT_AFTER // 插入 old 区域头部
| | |
| | |--> buf_LRU_old_adjust_len // 调整 LRU
`

### 管理 LRU

前面提到 `LRU` 是一个非严格的最近使用列表，InnoDB 将 `LRU` 列表划分为两个区域：young 区域和 old 区域。`LRU` 列表的示意图如下：

`/** LRU 列表示意图
 LRU_old
 |
**********************young************************|********old*********
|==================================================|===================|

几个主要的常量：
BUF_LRU_OLD_TOLERANCE 20
BUF_LRU_NON_OLD_MIN_LEN 5
BUF_LRU_OLD_MIN_LEN 512
BUF_LRU_OLD_RATIO_DIV 1024

参数控制：
innodb_old_block_pct old 区域占比

*/
`

`buf_LRU_old_adjust_len` 方法会根据 `innodb_old_block_pct` 参数，维护 young 区域和 old 区域的长度，主要逻辑如下：

1. 当 `LRU` 长度小于 ` BUF_LRU_OLD_MIN_LEN` 时，不划分区域。
2. 不是每次操作 `LRU` 列表后都需要立即调整，`BUF_LRU_OLD_TOLERANCE` 可以看成是容忍范围。
3. 当 old 区域变大时，LRU_old 指针向前移动；反之向后移动。

当 Block 被再次访问时，会触发 `buf_page_make_young_if_needed` 函数进行 Block 位置的调整，基本过程如下：

`|--> buf_page_make_young_if_needed
| |--> buf_page_peek_if_too_old // 判断访问间隔
| | |--> buf_page_peek_if_young // 判断 young 区域位置
| |
| |--> buf_page_make_young
| | |--> buf_LRU_make_block_young
| | | |--> buf_LRU_remove_block
| | | |--> buf_LRU_add_block_low
`

`buf_page_make_young_if_needed` 移动 Block 时需要考虑：

1. 访问间隔需要大于 `buf_LRU_old_threshold_ms` 。
2. 当 Block 在 young 区域前 1/4 时，不需要移动。

InnoDB 中 `LRU` 列表的设计虽然简单，但是也有许多优化在里面，感兴趣的同学可以仔细研究，本文仅是一个简单的介绍。

## 回收

### 释放 Page

前面提到，当 `free_list` 列表为空时，会首先尝试从 `LRU` 列表中进行回收，Page 的释放入口函数为 `buf_LRU_free_page` ，该方法的主要处理流程如下：

`|--> buf_LRU_free_page
| |--> buf_page_can_relocate // 检查 buf_fix_count 计数和 io_fix 状态
| |
| |--> buf_LRU_block_remove_hashed // 从 LRU 和 page_hash 中删除
| | |--> buf_LRU_remove_block
| | | |--> buf_LRU_old_adjust_len
| | |--> HASH_DELETE
| |
| |--> btr_search_drop_page_hash_index // 从 AHI 中删除
| |
| |--> buf_LRU_block_free_hashed_page // 放回 free_list
| | |--> buf_LRU_block_free_non_file_page
`

释放 1 个 Page 时，首先需要检查 `io_fix` 状态和 `buf_fix_count` 计数，确保当前 Page 没有被使用，然后将 Block 从依次从 `LRU` 列表、`page_hash` 、`AHI` 中删除，最后将 Block 重新放入到 `free_list` 列表中。

### 同步刷脏

同步刷脏的入口函数为 `buf_flush_page` ，同步刷脏过程仅会刷 1 个 Page，保证能够获取到 1 个可用的 Block，主要处理流程如下：

`|--> buf_flush_page // 刷单个 page
| |--> buf_page_set_io_fix // io_fix 设置为 BUF_IO_WRITE
| |--> buf_flush_write_block_low
| | |--> log_write_up_to // 写 redo
| | |
| | |--> fil_io
| | |--> buf_dblwr_write_single_page // 写数据页
| | |
| | |--> fil_flush
| | |--> buf_page_io_complete
| | | |--> buf_flush_write_complete
| | | | |--> buf_flush_remove
| | | | |--> buf_page_set_io_fix // io_fix 设置为 BUF_IO_NONE
| | | |
| | | |--> buf_LRU_free_page
`

InnoDB 通过严格的 WAL 机制保证数据的一致性，刷脏过程同样如此。首先需要保证对应的日志文件落盘，然后再写入数据页。最后将 Block 从 `flush_list` 列表中移除，此时 Page 变成可回收状态，再次调用 `buf_LRU_free_page` 进行回收。

同步刷脏的过程不仅在获取 Block 时会被调用，在表删除的时候同样会被调用，表删除时会根据 `space_id` 进行批量的刷脏，入口函数为 `buf_LRU_flush_or_remove_pages` ，处理流程如下：

`|--> buf_LRU_flush_or_remove_pages // 根据 space_id 刷脏
| |--> buf_LRU_drop_page_hash_for_tablespace // 遍历 LRU
| | |--> buf_LRU_drop_page_hash_batch
| | | |--> btr_search_drop_page_hash_when_freed
| | | | |--> buf_page_get_gen
| | | | |--> btr_search_drop_page_hash_index // 从 AHI 中删除
| |
| |--> buf_LRU_remove_pages
| | |--> buf_LRU_remove_all_pages // 遍历 LRU
| | | |--> buf_LRU_block_remove_hashed // 从 LRU 和 page_hash 中删除
| | | |--> buf_LRU_block_free_hashed_page // 放回 free_list
| | |
| | |--> buf_flush_dirty_pages
| | | |--> buf_flush_or_remove_pages // 遍历 flush_list
| | | | |--> buf_flush_or_remove_page
| | | | | |--> buf_flush_remove
| | | | | |
| | | | | |--> buf_flush_page
`

具体的过程在此不再赘述，大家可以自己去阅读相应的代码。需要注意的是：如果单个 session 中使用了临时表，那么在 session 退出的时候，也会进入到上述的刷脏流程，当 `LRU` 列表很大时，session 退出的性能将会受到很大的影响。AliSQL 对此进行了优化，欢迎试用。

### 异步刷脏

除了同步刷脏之外，MySQL 中还引入单独的刷脏线程进行异步刷脏。刷脏线程按照功能划分包括两种：coordinator 线程和 cleaner 线程。coordinator 线程会计算最大的刷脏量，然后分配刷脏任务给 cleaner 线程，cleaner 线程进行实际的刷脏工作（coordinator 线程本身也会参与刷脏）。异步刷脏的入口函数为 `buf_flush_page_cleaner_init` ，基本流程如下：

`|--> buf_flush_page_coordinator_thread
| |--> os_event_wait(buf_flush_event)
| 
| /* loop */
| |--> page_cleaner_flush_pages_recommendation // 计算最大刷脏量
| |--> pc_request // 任务分发，slot 数目等于 bp_instance 数目
| | |--> os_event_set(page_cleaner->is_requested)
| |--> pc_flush_slot // 参与刷脏
| |--> pc_wait_finished

|--> buf_flush_page_cleaner_thread
| |--> os_event_wait(page_cleaner->is_requested)
| |--> pc_flush_slot // 1 个线程处理 1 个 bp_instance
| | |--> buf_flush_LRU_list // 从 LRU 中刷脏
| | | |--> buf_flush_do_batch(BUF_FLUSH_LRU)
| | |
| | |--> buf_flush_do_batch(BUF_FLUSH_LIST) // 从 flush_list 刷脏
| | | |--> buf_flush_batch
| | | | |--> buf_do_LRU_batch
| | | | | |--> buf_free_from_unzip_LRU_list_batch
| | | | | |--> buf_flush_LRU_list_batch
| | | | | | |--> buf_LRU_free_page
| | | | | | |--> buf_flush_page_and_try_neighbors
| | | | | | | |--> buf_flush_try_neighbors
| | | | | | | | |--> buf_flush_page
| | | | |
| | | | |--> buf_do_flush_list_batch
| | | | | |--> buf_flush_page_and_try_neighbors
`

异步刷脏的具体过程可以参考[这篇文章](http://mysql.taobao.org/monthly/2018/09/02/)，异步刷脏过程中有一个非常重要的点就是 `page_cleaner_flush_pages_recommendation` 计算最大刷脏量，相关的细节在此不再展开，后面有机会再单独整理一篇各种后台线程的更新逻辑。

## 总结

本文从申请、管理、回收三部分对 InnoDB Buffer Page 的生命周期管理进行了介绍，文中的内容只是一个基本概要，更多的细节还需要读者在阅读代码的过程中慢慢发掘。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)