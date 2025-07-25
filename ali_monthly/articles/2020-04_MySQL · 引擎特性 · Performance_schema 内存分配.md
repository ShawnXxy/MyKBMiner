# MySQL · 引擎特性 · Performance_schema 内存分配

**Date:** 2020/04
**Source:** http://mysql.taobao.org/monthly/2020/04/05/
**Images:** 2 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2020 / 04
 ](/monthly/2020/04)

 * 当期文章

 PostgreSQL · 源码分析 · 回放分析（一）
* MySQL · 源码分析 · InnoDB读写锁实现分析
* MySQL · 最佳实践 · X-Engine并行扫描
* MySQL · 引擎特性 · 8.0 Window Functions 剖析
* MySQL · 引擎特性 · Performance_schema 内存分配
* MySQL · 引擎特性 · 手动分析InnoDB B+Tree结构
* Redis · 最佳实践 · 集群配置：Redis Cluster
* MongoDB · 引擎特性 · 大量集合启动加载优化原理
* MySQL · 引擎特性 · 8.0 Lock Manager

 ## MySQL · 引擎特性 · Performance_schema 内存分配 
 Author: 思勉 

 ## 概述
Performance Schema(pfs)是对MySQL的细力度的性能监控诊断工具，覆盖statement/io/memory/lock 等各个性能相关的模块。Pfs采集到的性能数据使用 performance_Schema 引擎存储，全部保存在内存。
本文关注 pfs 的内存管理。首先从代码中分析 pfs 内存管理机制，然后以一个监控项为例介绍 pfs 的流程，最后介绍下 pfs 内存相关的参数。本文代码基于 MySQL 8.0.18版本。

## Pfs内存管理
### 核心数据结构
**PFS_buffer_scalable_container**
PFS_buffer_scalable_container 用于内存管理(申请，扩容，释放)，内部结构如下图。 其中，global*container （以下称为 container ）为全局单例变量，下面是其示意图以及结构定义代码。Container 存储上分两层: page 和 record。 以 global_thread_container 为例，默认global_thread_container中包含 多个 PFS_thread_array(page), page 内部包含多个 PFS_thread(record)。 
![PFS_buffer_scalable_container](.img/cbfd9334e607_2020-04-26-simian-scal-container.png)
PFS_buffer_scalable_container 代码

`template <class T, int PFS_PAGE_SIZE, int PFS_PAGE_COUNT,
 class U = PFS_buffer_default_array<T>,
 class V = PFS_buffer_default_allocator<T>>
class PFS_buffer_scalable_container {
 typedef T value_type; // record 类型
 typedef U array_type; // page 类型
 typedef V allocator_type; // page 分配器，需实现 alloc_array/free_array
 value_type *allocate(pfs_dirty_state *dirty_state); // 分配记录
 void deallocate(value_type *pfs) { m_array.deallocate(pfs); } // 释放记录
 array_type m_array; // 内存起始位置
 size_t m_max; // PFS_PAGE_SIZE* PFS_PAGE_COUNT
 allocator_type *m_allocator; // 分配器
 }

class PFS_thread_allocator {
 public:
 int alloc_array(PFS_thread_array *array);
 void free_array(PFS_thread_array *array);
};
`
实例化后的 container 对象复制管理 pfs 各个模块的内存分配，其与系统表对应关系如下：

 global_account_container
 events_%_summary_by_account_by_event_name

 global_host_container
 events_%_summary_by_host_by_event_name

 global_thread_container
 events_%_summary_by_thread_by_event_name

 global_user_container
 events_%_summary_by_user_by_event_name

 global_mutex_container
 mutex_instances

 global_rwlock_container
 rwlock_instances

 global_cond_container
 cond_instances

 global_socket_container
 socket_instances

 global_mdl_container
 metadata_locks

### Pfs内存管理模型
#### 1) 系统启动的时候预先分配内存，系统运行期间根据需要重新分配内存
Pfs 的内存分配发生在 page 分配(即alloc_array函数)，启动时初始化会分配部分page ，系统运行期间若 page 用满会分配新的 page。 在 page 内部分配 record 时，使用原子操作避免加锁。 下面是  `global_thread_container`  运行期间分配thread 的伪代码。

`PFS_thread *pfs = global_thread_container.allocate(&dirty_state)
{
 if (m_full) { m_lost++; return NULL; } // 如果container 满了直接返回
 while (monotonic < monotonic_max){
 array= m_pages[index]
 pfs = array->allocate(dirty_state); // 从现有 page 中分配
 pfs->m_page= reinterpret_cast<PFS_opaque_container_page *> (array);
 return pfs;
 }
 array = new array_type(); // 分配新 page
 int rc= m_allocator->alloc_array(array); // 内部调用PFS_MALLOC_ARRAY分配内存
}
`
#### 2) Record 采用定长方式存储，每次申请固定数量长度的内存，并初始化填0
真正的内存分配由m_allocator->alloc_array进行，我们以PFS_thread_allocator::alloc_array为例展开代码，PFS_thread中保存了线程粒度下的 statement/wait/error 等数据。 每个PFS_thread对象申请的内存为固定的，以statement为例，MySQL 支持的 statement 类型为220个，每个PFS_thread内会为220个类型提前分配位置并初始化为0，这也是 pfs 内存消耗的重要原因。

`int PFS_thread_allocator::alloc_array(PFS_thread_array *array) {
 size_t size = array->m_max; // 单个 page 内保存的记录(即 PFS_thread)数

 size_t index;
 size_t waits_sizing = size * wait_class_max; // wait_class_max 为等待事件的种类
 size_t statements_sizing = size * statement_class_max; // statement_class_max 语句类型个数
 size_t transactions_sizing = size * transaction_class_max; // 事务类型个数
 size_t errors_sizing = (max_server_errors != 0) ? size * error_class_max : 0; // error 类型个数
 ...
 array->m_ptr =
 PFS_MALLOC_ARRAY(&builtin_memory_thread, size, sizeof(PFS_thread),
 PFS_thread, MYF(MY_ZEROFILL));
 array->m_instr_class_waits_array = PFS_MALLOC_ARRAY(
 &builtin_memory_thread_waits, waits_sizing, sizeof(PFS_single_stat),
 PFS_single_stat, MYF(MY_ZEROFILL));
 array->m_instr_class_statements_array = PFS_MALLOC_ARRAY(
 &builtin_memory_thread_statements, statements_sizing,
 sizeof(PFS_statement_stat), PFS_statement_stat, MYF(MY_ZEROFILL));
 array->m_instr_class_errors_array = PFS_MALLOC_ARRAY(
 &builtin_memory_host_errors, errors_sizing, sizeof(PFS_error_stat),
 PFS_error_stat, MYF(MY_ZEROFILL)); 
 ... 
}
`
#### 3) 系统运行期间不释放内存，只在shutdown时 释放内存
下面是thread_container 释放thread 的代码逻辑

`global_thread_container.deallocate(pfs);
{ // 只是标记回收，并不会实际释放空间
 safe_pfs->m_lock.allocated_to_free(); 
 page->m_full = false;
 m_full = false;
}
`

#### 4) 数据在不同粒度的维度汇总
Pfs 数据库下可以看到对同一个监控指标有很多个不同的表，每个表代表一个统计的维度。 

`mysql> show tables like '%statement%summary%';
+----------------------------------------------------+
| Tables_in_performance_schema (%statement%summary%) |
+----------------------------------------------------+
| events_statements_summary_by_account_by_event_name |
| events_statements_summary_by_digest |
| events_statements_summary_by_digest_supplement |
| events_statements_summary_by_host_by_event_name |
| events_statements_summary_by_program |
| events_statements_summary_by_thread_by_event_name |
| events_statements_summary_by_user_by_event_name |
| events_statements_summary_global_by_event_name |
+----------------------------------------------------+
`
在内部，不同的统计维度被称为集合(aggregates)，对同一条数据在内部只会保存一份，运行期间会进行从细维度到高纬度的汇总。 pfs.cc代码注释中用这种图表的方式进行了说明，下面 以statement 为例介绍下汇总的过程，读者可以自己理解下。

` statement_locker(T, S)
 |
 | [1]
 |
1a |-> pfs_thread(T).event_name(S) =====>> [A], [B], [C], [D], [E]
 | |
 | | [2]
 | |
 | 2a |-> pfs_account(U, H).event_name(S) =====>> [B], [C], [D], [E]
 | . |
 | . | [3-RESET]
 | . |
 | 2b .....+-> pfs_user(U).event_name(S) =====>> [C]
 | . |
 | 2c .....+-> pfs_host(H).event_name(S) =====>> [D], [E]
 | . . |
 | . . | [4-RESET]
 | 2d . . |
1b |----+----+----+-> pfs_statement_class(S) =====>> [E]
 |
1c |-> pfs_thread(T).statement_current(S) =====>> [F]
 |
1d |-> pfs_thread(T).statement_history(S) =====>> [G]
 |
1e |-> statement_history_long(S) =====>> [H]
 |
1f |-> statement_digest(S) =====>> [I]

@endverbatim

 Implemented as:
 - [1] #pfs_start_statement_v2(), #pfs_end_statement_v2()
 (1a, 1b) is an aggregation by EVENT_NAME,
 (1c, 1d, 1e) is an aggregation by TIME,
 (1f) is an aggregation by DIGEST
 all of these are orthogonal,
 and implemented in #pfs_end_statement_v2().
 - [2] #pfs_delete_thread_v1(), #aggregate_thread_statements()
 - [3] @c PFS_account::aggregate_statements()
 - [4] @c PFS_host::aggregate_statements()
 - [A] EVENTS_STATEMENTS_SUMMARY_BY_THREAD_BY_EVENT_NAME,
 @c table_esms_by_thread_by_event_name::make_row()
 - [B] EVENTS_STATEMENTS_SUMMARY_BY_ACCOUNT_BY_EVENT_NAME,
 @c table_esms_by_account_by_event_name::make_row()
 - [C] EVENTS_STATEMENTS_SUMMARY_BY_USER_BY_EVENT_NAME,
 @c table_esms_by_user_by_event_name::make_row()
 - [D] EVENTS_STATEMENTS_SUMMARY_BY_HOST_BY_EVENT_NAME,
 @c table_esms_by_host_by_event_name::make_row()
 - [E] EVENTS_STATEMENTS_SUMMARY_GLOBAL_BY_EVENT_NAME,
 @c table_esms_global_by_event_name::make_row()
 - [F] EVENTS_STATEMENTS_CURRENT,
 @c table_events_statements_current::make_row()
 - [G] EVENTS_STATEMENTS_HISTORY,
 @c table_events_statements_history::make_row()
 - [H] EVENTS_STATEMENTS_HISTORY_LONG,
 @c table_events_statements_history_long::make_row()
 - [I] EVENTS_STATEMENTS_SUMMARY_BY_DIGEST
 @c table_esms_by_digest::make_row()
`
## Pfs性能监控过程
这里以statement 的一个监控项为例来介绍 pfs 性能数据采集的整个过程。 监控数据最终记录在 `events_statements_summary_by_thread_by_event_name` 表中，需提前打开 `setup_consumers.thread_instrumentation` 开关。

### 线程创建
调用入口:  `PSI_THREAD_CALL(new_thread)` 
线程启动时进行在全局container( `global_thread_container` )中申请内存空间，并进行一系列的监控数据初始化。 首先尝试在现有的 page 中申请空闲的record， 找不到的话申请新的page。

### 语句开始前
调用入口:  `MYSQL_START_STATEMENT` 
在语句开始的位置调用进行，比如 在`dispatch_command` 函数中，进行statement 统计的初始化，记录 sql 启动时间。

### 语句结束后
调用入口: `MYSQL_END_STATEMENT` 

`pfs_end_statement_v2(PSI_statement_locker *locker, void *stmt_da)
{
 PSI_statement_locker_state *state =
 reinterpret_cast<PSI_statement_locker_state *>(locker);
 // 填充 pfs
 PFS_events_statements *pfs =
 reinterpret_cast<PFS_events_statements *>(state->m_statement);
 insert_events_statements_history(thread, pfs); // 写入到 EVENTS_STATEMENTS_HISTORY
 insert_events_statements_history_long(pfs); // 写入到 EVENTS_STATEMENTS_HISTORY_LONG
 // 获取写入的位置
 event_name_array = thread->write_instr_class_statements_stats(); // PFS_statement_stat*
 stat = &event_name_array[index];
 // 开始填充 stat，写入汇总表
 stat->m_lock_time += state->m_lock_time; 
}
`
### 线程结束
调用入口: `PSI_THREAD_CALL(delete_current_thread)`

`void pfs_delete_current_thread_vc(void) {
 // 将线程的数据汇总到 account 或者 host 统计中
 aggregate_thread(thread, thread->m_account, thread->m_user, thread->m_host);
 ...
 // 销毁 pfs thread, global_thread_container 收回空间
 global_thread_container.deallocate(pfs);

}
`

## Pfs内存参数设置
主要看下影响pfs内存使用的相关参数

### performance_schema%max%instance
控制监控实体的个数，内部即限制对应 container 的容量。

`+------------------------------------------------------+-------+
| Variable_name | Value |
+------------------------------------------------------+-------+
| performance_schema_max_cond_instances | -1 |
| performance_schema_max_file_instances | -1 |
| performance_schema_max_mutex_instances | -1 |
| performance_schema_max_prepared_statements_instances | -1 |
| performance_schema_max_program_instances | -1 |
| performance_schema_max_rwlock_instances | -1 |
| performance_schema_max_socket_instances | -1 |
| performance_schema_max_table_instances | -1 |
| performance_schema_max_thread_instances | -1 |
+------------------------------------------------------+-------+
performance_schema_max_cond_instances global_cond_container
performance_schema_max_file_instances global_file_container
performance_schema_max_mutex_instances global_mutex_container
performance_schema_max_prepared_statements_instances global_prepared_stmt_container
performance_schema_max_program_instances global_program_container
performance_schema_max_rwlock_instances global_rwlock_container
performance_schema_max_socket_instances global_socket_container
performance_schema_max_table_instances global_table_share_container
performance_schema_max_thread_instances global_thread_container 
`
### performance_schema_%_size
影响对应表的记录上限

`ysql> show global variables like 'performance_schema_%_size';
+----------------------------------------------------------+-------+
| Variable_name | Value |
+----------------------------------------------------------+-------+
| performance_schema_accounts_size | -1 |
| performance_schema_digests_size | 100 |
| performance_schema_error_size | 20 |
| performance_schema_events_stages_history_long_size | 10000 |
| performance_schema_events_stages_history_size | 10 |
| performance_schema_events_statements_history_long_size | 10000 |
| performance_schema_events_statements_history_size | 10 |
| performance_schema_events_transactions_history_long_size | 10000 |
| performance_schema_events_transactions_history_size | 10 |
| performance_schema_events_waits_history_long_size | 10000 |
| performance_schema_events_waits_history_size | 10 |
| performance_schema_hosts_size | -1 |
| performance_schema_session_connect_attrs_size | 512 |
| performance_schema_setup_actors_size | -1 |
| performance_schema_setup_objects_size | -1 |
| performance_schema_users_size | -1 |
+----------------------------------------------------------+-------+
`

### 其他参数:
performance_schema_error_size:
监控的系统错误码个数，如果对错误码没有监控需求，建议调低
performance_schema_digests_size:
events_statements_summary_by_digest 表的最大容量

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)