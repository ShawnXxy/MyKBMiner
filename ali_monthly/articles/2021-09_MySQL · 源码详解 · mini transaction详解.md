# MySQL · 源码详解 · mini transaction详解

**Date:** 2021/09
**Source:** http://mysql.taobao.org/monthly/2021/09/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2021 / 09
 ](/monthly/2021/09)

 * 当期文章

 MySQL · 源码分析 · 事务锁调度分析
* PolarDB · 引擎特性 · DDL物理复制优化
* MySQL · 源码分析 · Performance Schema 初始化过程
* MySQL · 源码详解 · mini transaction详解

 ## MySQL · 源码详解 · mini transaction详解 
 Author: 沧何 

 ## mtr介绍
mini-transaction是mysql内部的对底层page的一个原子操作，保证并发事务操作下以及数据库异常时page中数据的一致性。
​

mini transaction 的信息保存在结构体 mtr_t 中，结构体成员描述详见[前文](http://mysql.taobao.org/monthly/2017/10/03/)，其中m_memo和m_log最为重要。本文的代码参照mysql 8.0.23。
​

m_memo管理mtr持有的锁信息。对于持有的page锁，还要保留page指针，这是为了在commit时，将修改的脏页加入flush list中。
​

m_log保存mtr修改操作对应的redo日志。在commit时，将redo日志一起拷贝到log_sys模块的公共日志buffer中。
​

而mtr的使用方式如下：

`mtr_t mtr;
mtr_start(&mtr);

// 1. 加锁

// 对待访问的index加锁
mtr_s_lock(rw_lock_t, mtr);
mtr_x_lock(rw_lock_t, mtr);

// 对待读写的page加锁
mtr_memo_push(mtr, buf_block_t, MTR_MEMO_PAGE_S_FIX);
mtr_memo_push(mtr, buf_block_t, MTR_MEMO_PAGE_X_FIX);

// 2. 访问或修改page
btr_cur_search_to_nth_level
btr_cur_optimistic_insert

// 3. 为修改操作生成redo
mlog_open
mlog_write_initial_log_record_fast
mlog_close

// 4. 持久化redo，解锁
mtr_commit(&mtr);
`
## mtr的成员
我们重点看下mtr_t中m_memo和m_log成员的实现。m_memo和m_log都是mtr_buf_t类型的对象，mtr_buf_t是由一个双向链表组成的动态buffer，每个元素是512B大小的buffer（512B刚好匹配一个log block大小）。随着mtr_buf_t存储的数据的增加，它会自动生成新的512B的buffer，并加入双向链表中。
​

m_memo使用动态buffer的方式是把锁类型、锁地址或page地址加入动态buffer。在mtr_s_lock或mtr_memo_push中会执行如下操作：

`mtr_memo_slot_t *slot;

// 先在动态buffer中申请能容纳锁类型+地址的空间，再对该空间进行初始化
slot = m_impl.m_memo.push<mtr_memo_slot_t *>(sizeof(*slot));

// 锁类型
slot->type = type;
// 锁地址或page地址
slot->object = object;
`
​

m_log使用动态buffer的方式是把日志类型、space id、page no、以及具体的操作信息加入动态buffer。mlog_open：预分配待写入的日志空间，若空间不够，则增加新的buffer到动态buffer中。
mlog_write_initial_log_record_fast：写入日志类型、space id、page no，且m_n_log_recs加1。

`// 将日志类型写入从动态buffer中申请的空间log_ptr
mach_write_to_1(log_ptr, type);
log_ptr++;

// 将space id和page no以压缩格式写入log_ptr
log_ptr += mach_write_compressed(log_ptr, space_id);
log_ptr += mach_write_compressed(log_ptr, page_no);

// m_n_log_recs加1，用于标识是single log record还是multiple log records
mtr->added_rec();
`
mlog_close：更新最终的日志大小m_size
​

## mtr的使用
在开启一个mini transaction时，会初始化mtr对象中的m_log和m_memo成员，设置m_state为active。

`mtr_t::start
 |
 |-> 初始化mtr.m_impl->m_log日志管理对象
 |
 |-> 初始化mtr.m_impl->m_memo锁管理对象
 | m_log和m_memo都是mtr_buf_t，以block_t节点m_node域构建的双向链表
 |
 |-> m_log_mode=MTR_LOG_ALL(记录所有的数据变更) & m_state=MTR_STATE_ACTIVE

`
​

提交一个mini transaction的过程比较复杂，大致流程是先将m_log中的日志写入公共log buffer，再将m_memo中的加锁并且发生修改的脏page加入flush list，最后释放m_memo中的所有锁。
​

公共log buffer是按照log block格式存储的（包含12B的header和4B的trailer，详见[前文](http://mysql.taobao.org/monthly/2017/09/07/)中的日志块结构），每个log block大小为512B，并且持久化时以512B进行对齐。每个log block中能存储日志内容的空间为512-12-4=496B。
​

公共log buffer有个原子变量log.sn，其统计的是公共buffer中曾经存储过的日志内容的大小。通过sn可以很容易计算出对应的lsn，其统计的是公共buffer中曾经存储过的以log block格式的日志量的大小。
lsn = (sn / 496 * 512 + sn % 512 + 12)

公共log buffer是个循环buffer，其中有三个重要的位点log.write_lsn，log.sn对应的lsn，log.buf_limit_sn对应的lsn。其中log.write_lsn表示已写入磁盘的日志位点（不要求flush），log.sn对应的lsn表示已占位待拷贝的日志位点，log.buf_limit_sn对应的lsn表示可以占位的最大日志位点。满足log.write_lsn <= log.sn对应的lsn <= log.buf_limit_sn对应的lsn。
​

将m_log中的日志写入公共log buffer：

* 根据日志数m_n_log_recs是否为1，来判断是single log还是multiple log。对于single log，在日志的开头的日志类型字段中增加MLOG_SINGLE_REC_FLAG。而对于multiple log，在日志结尾增加1B的MLOG_MULTI_REC_END。
* 在公共log buffer中使用原子变量log.sn进行日志占位。
* 在往已占位的日志空间中拷贝日志前，有以下两种情况需要等待：
 
 若当前的log.sn位点被SN_LOCKED锁定，则要等待log.sn_locked 超过占位前的log.sn。当公共log buffer需要在线变更大小的时候，会进行SN_LOCKED加锁。
* 若日志写入速度过快，来不及写磁盘，就会把log buffer占满，这时需要阻塞等待日志的写磁盘。

 将m_log动态buffer拷贝到公共log buffer，是按照512B大小的buffer粒度进行拷贝的：
 * 若日志长度超过log block剩余大小，则要做截断，并增加tail和新的header，以满足log block格式
* 若写到log buffer的结尾（默认大小为16M），要继续转向log buffer开头继续拷贝。由于log buffer大小是log block的倍数，所以这里不需要再次做截断。
* 每个buffer拷贝完成后触发一次log.recent_written的Link_buf更新（详见前文），log.recent_written记录完成拷贝的最大连续日志的lsn

 当m_log日志都写完，要检查已写入的日志是否横跨log block，若横跨了，则要在结尾的log block的header的LOG_BLOCK_FIRST_REC_GROUP字段中标识新mtr的位点end_lsn。

将m_memo中的加锁并且发生修改的脏page加入flush list：

* 遍历m_memo动态buffer中的每个buffer中的每个锁对象mtr_memo_slot_t
 
 若是page锁，且该page发生了修改，则将该page加入flush list

 触发一次log.recent_closed的Link_buf更新，log.recent_closed记录添加到flush list的最大连续日志的lsn

以下是详细流程图：

```
mtr_t::commit
 |
 |-> mtr_t::Command
 |
(m_n_log_recs>0 || m_modifications)
 |
 |-> (yes)
 | v
 | Command::execute 将mtr.m_impl->m_log写入公共log buffer，把脏页加入flush list
 | |
 | |-> prepare_write
 | | |
 | | |-> 若 mtr.m_impl->m_log_mode为 MTR_LOG_NO_REDO或MTR_LOG_NONE，则直接返回
 | | |
 | | |-> 若 mtr.m_impl->m_n_log_recs==1，则 m_log.front()->begin()|=MLOG_SINGLE_REC_FLAG，在日志头Type字段中标识，
 | | 否则 m_log->push(MLOG_MULTI_REC_END)，在日志结尾附加1B
 | |
 | |-> log_buffer_reserve 在公共log buffer中为日志预留空
 | | |
 | | |-> log_buffer_s_lock_enter_reserve
 | | | |
 | | | |-> 对 log.pfs_psi加 s-lock
 | | | |
 | | | |-> log.sn.fetch_add(mtr.m_impl->m_log.m_size) 在公共的log buffer中占位
 | | | |
 | | | |-> log_buffer_s_lock_wait 若log.sn被SN_LOCKED，则等待log.sn_locked 超过占位前的log.sn
 | | |
 | | |-> log_translate_sn_to_lsn 将日志内容的偏移量log.sn 转为log block格式的偏移量start_lsn，start_lsn可以唯一表示日志在log block和公共log buffer中的位置
 | | |
 | | |-> log_wait_for_space_after_reserving 若end_sn > log.buf_limit_sn，则等待
 | |
 | |-> mtr_write_log_t(mtr.m_impl->m_log.m_list) 将日志内容拷贝至预留的空间
 | | |
 | | (loop mtr_buf_t::block in m_list) 将mtr.m_impl->m_log中的日志按block粒度拷贝到公共log buffer
 | | |
 | | |-> log_buffer_write 以log block的 start_lsn%OS_FILE_LOG_BLOCK_SIZE位置的数据 拷贝到公共log buffer的 start_lsn%log.buf_size位置
 | | | |
 | | | |-> left = OS_FILE_LOG_BLOCK_SIZE - LOG_BLOCK_TRL_SIZE - offset 若日志长度超过log block剩余大小，则要做截断
 | | | |
 | | | |-> lsn_diff = left + LOG_BLOCK_TRL_SIZE + LOG_BLOCK_HDR_SIZE 若log block写满，要增加tail和新的header
 | | | |
 | | | |-> 若公共log buffer被写满，则下次从开头继续写。因为每个mtr的日志在解析时大小就不超过2M，肯定不会超过公共log buffer的大小16M
 | | | |
 | | | |-> log_block_set_first_rec_group 在新header中设置 LOG_BLOCK_FIRST_REC_GROUP为0
 | | |
 | | |-> log_buffer_set_first_record_group 若mtr日志都写完且 mtr开头和结尾不在同一个log block中，则在新header中设置 LOG_BLOCK_FIRST_REC_GROUP为 end_lsn
 | | |
 | | |-> log_buffer_write_completed 每个block拷贝完成后均触发一次Link_buf(并查集)的更新，log.recent_written记录完成拷贝的最大连续日志的lsn
 | | |
 | | |-> log.recent_written.add_link_advance_tail 在recent_written->m_links的slot中记录当前日志的end_lsn，m_tail表示已拷贝到log buffer连续日志的end_lsn
 | | | |
 | | | |-> 若m_tail为当前日志的start_lsn，则推进m_tail为当期日志的end_lsn
 | | | |
 | | | |-> 否则recent_written->m_links[start_lsn%capacity] = end_lsn，并推进m_tail
 | | | v
 | | | log.recent_written.advance_tail_until 等到log.recent_written.m_tail推进到最大lsn
 | | | |
 | | | |-> 若recent_written->m_links[m_tail%capacity] > m_tail，则使用cas更改recent_written->m_links[m_tail%capacity] = m_tail 来排他访问
 | | | |
 | | | |-> (loop next_position) 推进m_tail为此刻连续的最大lsn，即使没推进到当前日志，其它help线程会帮忙推进
 | | |
 | | |-> 若log.recent_written.m_tail > log.current_ready_waiting_lsn，则os_event_set(log.closer_event)
 | |
 | |-> add_dirty_blocks_to_flush_list(mtr.m_impl->m_memo) 将mtr锁管理中记录的脏页加入flush list
 | | |
 | | (reverse loop mtr_buf_t::block in m_memo)
 | | v
 | | (reverse loop mtr_memo_slot_t in block)
 | | |
 | | |-> add_to_flush 为了去掉flush_order_mutex，把mtr对应的脏页无序的添加到flush list，在做checkpoint时, 无法保证flush list 上面最头的page lsn是最小的
 | | v
 | | add_dirty_page_to_flush_list 把修改后的page加入flush list，当mtr_memo_slot_t.type为MTR_MEMO_PAGE_X_FIX或MTR_MEMO_PAGE_SX_FIX，
 | | | 或为MTR_MEMO_BUF_FIX，且mtr_memo_slot_t.object->made_dirty_with_no_latch
 | | v
 | | buf_flush_note_modification(mtr_memo_slot_t.object)
 | |
 | |-> log_buffer_close 将mtr锁管理中记录的脏页处理完后触发一次Link_buf更新，log.recent_closed记录添加到flush list的最大连续日志的lsn
 | | 以log.recent_closed.m_tail的lsn来做checkpoint肯定是安全的，
 | v
 | log_buffer_s_lock_exit_close
 | |
 | |-> 对 log.pfs_psi解锁 s-lock
 | |
 | |-> log.recent_closed.add_link_advance_tail
 |
 |-> Command::release_all
 | |
 | |-> Release_all(mtr.m_impl->m_memo) 释放mtr持有的锁
 |
 |-> Command::release_resources -> clean mtr.m_impl->m_log & m_memo

```

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)