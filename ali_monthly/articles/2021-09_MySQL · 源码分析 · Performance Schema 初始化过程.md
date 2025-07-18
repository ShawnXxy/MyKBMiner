# MySQL · 源码分析 · Performance Schema 初始化过程

**Date:** 2021/09
**Source:** http://mysql.taobao.org/monthly/2021/09/03/
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

 ## MySQL · 源码分析 · Performance Schema 初始化过程 
 Author: 张迎港 

 ​ Performance Schema可以用于监控MySQL server在运行过程中的资源消耗、资源等待等情况。在开启该功能后，MySQL会收集服务器的运行状态，并将信息记录在performance_schema库中对应的Table内。用户可通过查看对应的Table，了解数据库的运行状态。例如，借助performance_schema.metadata_locks表，可以获知当前MySQL中MDL锁的持有情况。借助performance_schema.events_stages_current表，我们可以获知当前DDL语句的执行进度。在MySQL中，开启/关闭Performance Schema均需要重启。该功能由如下参数控制：

 参数
 取值范围
 说明

 performance_schema
 [ON|OFF]
 控制是否启用Performance Schema功能

本文主要分析实例启动时，Performance Schema模块的初始化过程。以下分析均基于MySQL-8.0.13版本。

## 一、初始化过程分析

### 概述

MySQL实例的总入口函数为mysqld_main()。该函数中，与Performance Schema模块初始化相关的函数简化记录如下：

`int mysqld_main()
{
 ...
 1. pre_initialize_performance_schema();
 ...
 2. init_pfs_instrument_array（);
 ...
 3. ho_error = handle_early_options();
 ...
 4. initialize_performance_schema();
 ...
 5. init_server_psi_keys();
 6. my_thread_global_reinit();
 ...
 7. initialize_performance_schema_acl();
 ...
}
`

下面对各个函数进行详细分析说明。

### 1. pre_initialize_performance_schema()

`void pre_initialize_performance_schema() {
 pfs_initialized = false;
 // 初始化内存监控相关的instrument;
 init_all_builtin_memory_class();
 // 重置统计信息;
 PFS_table_stat::g_reset_template.reset();
 global_idle_stat.reset();
 global_table_io_stat.reset();
 global_table_lock_stat.reset();
 g_histogram_pico_timers.init();
 global_statements_histogram.reset();

 THR_PFS = nullptr;
 for (int i = 0; i < THR_PFS_NUM_KEYS; ++i) {
 THR_PFS_contexts[i] = nullptr;
 }
}
`

​ 其中，init_all_builtin_memory_class() 函数负责注册与pfs模块内存监控相关的instrument。该监控无法被用户修改。只要打开了PFS，就会默认开启对PFS模块内存消耗的监控。内存监控的结果可以通过表memory_summary_global_by_event_name进行查看。在实际应用中，由于pfs的表都是内存表，且PFS模块的的内存消耗具有如下特点：

* 启动时会预分配一定的空间；
* 运行时，若空间不足，则会申请更多空间
* 已申请的空间不会进行释放，但会进行复用。

​ 因此，如果负载过大，PFS模块可能会消耗大量内存。以MDL锁监控为例进行说明。如果MDL最大检测数目设置为-1，表示自适应调整：

`performance_schema_max_metadata_locks = -1;
`

​ 通过查阅源码可知，此时最大监控的数目为1024*1024，每个mdl锁监控消耗的内存大小为：

`sizeof(PFS_metadata_lock) = 512bytes
`

​ 若业务峰值mdl锁到达理论上限，则此时，仅mdl锁监控部分，消耗的内存为：

`1024*1024*sizeof(PFS_metadata_lock)=512Mb
`

​ 由于PFS模块申请的内存只会在重启后释放，因此这样的消耗对对低规格实例可能产生较大影响。实际引用中，用户可借助memory_summary_global_by_event_name表，及时了解具体的内存消耗，以便及时调整对应的参数。

### 2. init_pfs_instrument_array()

​ 本函数对全局数组Pfs_instr_config_array进行初始化。该数组用于记录用户在配置文件中关于表 setup_instrument 的配置。在配置文件中，用户可以使用如下参数对instrument进行配置：

`performance-schema-instrument='instrument_name = value'
`

​ 其中，用户可以通过performance_schema.setup_instruments表中的NAME列，查看支持的instrument。instrument_name支持正则匹配。例如，如下配置可以开启stage/innodb/下所有以alter开头的instrument。

`performance_schema_instrument='stage/innodb/alter%=YES'
`

​ 在Pfs_instr_config_array数组中可以记录对同一个instrument的多个配置。生效规则如下：

​ 1、匹配最精确的配置项：

​ 如果通过不同的的正则表达式可以匹配到同一个instrument, 则匹配越精确的，优先级越高。例如，当配置文件中进行如下配置时，wait/lock/metadata/sql/mdl会被设置为打开状态。

`performance_schema_instrument='%=OFF'
performance_schema_instrument='wait/lock/metadata/sql/mdl = ON'
`

​ 2、相同匹配长度，越靠后优先级越高。

​ 当同一个instrument具有完全相反的配置，而且匹配长度一致时，越靠后者，优先级越高。例如，当配置文件中进行如下配置时，wait/lock/metadata/sql/mdl会被设置为打开状态。

`performance_schema_instrument='wait/lock/metadata/sql/mdl = OFF'
performance_schema_instrument='wait/lock/metadata/sql/mdl = ON'
`

### 3. handle_early_options()

​ 在MySQL进行初始化时，部分参数需要被用于初始化过程。本函数用于载入这一类参数。具体的，在sql/sys_vars.cc文件中，所有parse_flag= PARSE_EARLY 的变量，均会在此处被读取。这些参数可以被分为三类：

* Performance Schema 模块相关参数
* help 相关参数
* bootstrap 相关参数

对于Performance Schema模块的参数，可以进一步细分为如下三类：

* Performance Schema 控制开关

 ​ 如 performance_schema
* pfs内存分配&监控的相关参数

 ​ 控制监控项数目参数，如：performance_schema_max_stage_classes

 ​ 控制监控实体参数：如 performance_schema_max_thread_instances

 ​ 控制记录数目参数 如：performance_schema_events_stages_history_size
* 配置参数

 ​ 对consumers和instrument的配置等。如performance_schema_consumer_thread_instrumentation等

 上文提到的Pfs_instr_config_array数组，也是在此处被填充。

### 4. initialize_performance_schema()

​ 此处为PFS核心的初始化函数，在本函数中，完成了绝大多数的初始化工作。该函数简化后的伪代码如下：

`int initialize_performance_schema() {
 ...
 pfs_enabled = param->m_enabled;//pfs全局状态变量
 // 1. 配置默认参数
 pfs_automated_sizing(param);
 init_timers();// 配置定时器
 init_event_name_sizing(param);
 // 2. 注册全局instrument.
 register_global_classes();
 // 3. pfs初始化内存分配
 if (init_sync_class(param->m_mutex_class_sizing, param->m_rwlock_class_sizing,
 param->m_cond_class_sizing) ||
 init_thread_class(param->m_thread_class_sizing) ||
 init_table_share(param->m_table_share_sizing) ||
 init_table_share_lock_stat(param->m_table_lock_stat_sizing) ||
 init_table_share_index_stat(param->m_index_stat_sizing) ||
 init_file_class(param->m_file_class_sizing) ||
 init_stage_class(param->m_stage_class_sizing) ||
 init_statement_class(param->m_statement_class_sizing) ||
 init_socket_class(param->m_socket_class_sizing) ||
 init_memory_class(param->m_memory_class_sizing) ||
 init_instruments(param) ||
 init_events_waits_history_long(
 param->m_events_waits_history_long_sizing) ||
 init_events_stages_history_long(
 param->m_events_stages_history_long_sizing) ||
 init_events_statements_history_long(
 param->m_events_statements_history_long_sizing) ||
 init_events_transactions_history_long(
 param->m_events_transactions_history_long_sizing) ||
 init_file_hash(param) || init_table_share_hash(param) ||
 init_setup_actor(param) || init_setup_actor_hash(param) ||
 init_setup_object(param) || init_setup_object_hash(param) ||
 init_host(param) || init_host_hash(param) || init_user(param) ||
 init_user_hash(param) || init_account(param) ||
 init_account_hash(param) || init_digest(param) ||
 init_digest_hash(param) || init_program(param) ||
 init_program_hash(param) || init_prepared_stmt(param) ||
 init_error(param)) {
 /*
 The performance schema initialization failed.
 Free the memory used, and disable the instrumentation.
 */
 cleanup_performance_schema();
 return 2;
 }
 // 3. setup_consumer表配置
 if (param->m_enabled) {
 /** Default values for SETUP_CONSUMERS */
 flag_events_stages_current =
 param->m_consumer_events_stages_current_enabled;
 flag_events_stages_history =
 param->m_consumer_events_stages_history_enabled;
 flag_events_stages_history_long =
 param->m_consumer_events_stages_history_long_enabled;
 flag_events_statements_current =
 param->m_consumer_events_statements_current_enabled;
 flag_events_statements_history =
 param->m_consumer_events_statements_history_enabled;
 flag_events_statements_history_long =
 param->m_consumer_events_statements_history_long_enabled;
 flag_events_transactions_current =
 param->m_consumer_events_transactions_current_enabled;
 flag_events_transactions_history =
 param->m_consumer_events_transactions_history_enabled;
 flag_events_transactions_history_long =
 param->m_consumer_events_transactions_history_long_enabled;
 flag_events_waits_current = param->m_consumer_events_waits_current_enabled;
 flag_events_waits_history = param->m_consumer_events_waits_history_enabled;
 flag_events_waits_history_long =
 param->m_consumer_events_waits_history_long_enabled;
 flag_global_instrumentation =
 param->m_consumer_global_instrumentation_enabled;
 flag_thread_instrumentation =
 param->m_consumer_thread_instrumentation_enabled;
 flag_statements_digest = param->m_consumer_statement_digest_enabled;
 } else {
 flag_events_stages_current = false;
 flag_events_stages_history = false;
 flag_events_stages_history_long = false;
 flag_events_statements_current = false;
 flag_events_statements_history = false;
 flag_events_statements_history_long = false;
 flag_events_transactions_current = false;
 flag_events_transactions_history = false;
 flag_events_transactions_history_long = false;
 flag_events_waits_current = false;
 flag_events_waits_history = false;
 flag_events_waits_history_long = false;
 flag_global_instrumentation = false;
 flag_thread_instrumentation = false;
 flag_statements_digest = false;
 }

 pfs_initialized = true;
 // 4. 获取相关接口的实现函数
 if (param->m_enabled) {
 install_default_setup(&pfs_thread_bootstrap);
 *thread_bootstrap = &pfs_thread_bootstrap;
 *mutex_bootstrap = &pfs_mutex_bootstrap;
 *rwlock_bootstrap = &pfs_rwlock_bootstrap;
 *cond_bootstrap = &pfs_cond_bootstrap;
 *file_bootstrap = &pfs_file_bootstrap;
 *socket_bootstrap = &pfs_socket_bootstrap;
 *table_bootstrap = &pfs_table_bootstrap;
 *mdl_bootstrap = &pfs_mdl_bootstrap;
 *idle_bootstrap = &pfs_idle_bootstrap;
 *stage_bootstrap = &pfs_stage_bootstrap;
 *statement_bootstrap = &pfs_statement_bootstrap;
 *transaction_bootstrap = &pfs_transaction_bootstrap;
 *memory_bootstrap = &pfs_memory_bootstrap;
 *error_bootstrap = &pfs_error_bootstrap;
 *data_lock_bootstrap = &pfs_data_lock_bootstrap;
 *system_bootstrap = &pfs_system_bootstrap;
 }

 /* Initialize plugin table services */
 init_pfs_plugin_table();

 return 0;
}
`

部分重点函数详细介绍如下

#### 4.1 pfs_automated_sizing(param)

​ 在大多数情况下，用户只会对某些参数下进行设置。对于其他参数不作调整。因此，PFS会根据当前服务器的table_def_size、table_cache_size、max_connections三个参数，来自动选择合适的配置。判断条件如下：

`/**
 MAX_CONNECTIONS_DEFAULT = 151
 TABLE_DEF_CACHE_DEFAULT = 400
 TABLE_DEF_CACHE_DEFAULT = 4000 
*/
static PFS_sizing_data *estimate_hints(PFS_global_param *param) {
 if (max_connections <= MAX_CONNECTIONS_DEFAULT) &&
 (table_def_size <= TABLE_DEF_CACHE_DEFAULT) &&
 (table_cache_size <= TABLE_OPEN_CACHE_DEFAULT)) {
 return &small_data;
 }
 if ((max_connections <= MAX_CONNECTIONS_DEFAULT * 2) &&
 (table_def_size <= TABLE_DEF_CACHE_DEFAULT * 2) &&
 (table_cache_size <= TABLE_OPEN_CACHE_DEFAULT * 2)) {
 return &medium_data;
 }
 return &large_data;
}
`

​ small、medium和large对应的参数如下，若用户未指定的某参数的具体值，则根据上述的判断条件，此处会将其设置为如下默认值。

`PFS_sizing_data small_data = {
 /* History sizes */
 5, 100, 5, 100, 5, 100, 5, 100,
 /* Digests */
 1000,
 /* Session connect attrs. */
 512};

PFS_sizing_data medium_data = {
 /* History sizes */
 10, 1000, 10, 1000, 10, 1000, 10, 1000,
 /* Digests */
 5000,
 /* Session connect attrs. */
 512};

PFS_sizing_data large_data = {
 /* History sizes */
 10, 10000, 10, 10000, 10, 10000, 10, 10000,
 /* Digests */
 10000,
 /* Session connect attrs. */
 512};
`

​ 映射关系为：

`struct PFS_sizing_data {
 /** Default value for @c PFS_param.m_events_waits_history_sizing. */
 ulong m_events_waits_history_sizing;
 /** Default value for @c PFS_param.m_events_waits_history_long_sizing. */
 ulong m_events_waits_history_long_sizing;
 /** Default value for @c PFS_param.m_events_stages_history_sizing. */
 ulong m_events_stages_history_sizing;
 /** Default value for @c PFS_param.m_events_stages_history_long_sizing. */
 ulong m_events_stages_history_long_sizing;
 /** Default value for @c PFS_param.m_events_statements_history_sizing. */
 ulong m_events_statements_history_sizing;
 /** Default value for @c PFS_param.m_events_statements_history_long_sizing. */
 ulong m_events_statements_history_long_sizing;
 /** Default value for @c PFS_param.m_events_transactions_history_sizing. */
 ulong m_events_transactions_history_sizing;
 /** Default value for @c PFS_param.m_events_transactions_history_long_sizing.
 */
 ulong m_events_transactions_history_long_sizing;
 /** Default value for @c PFS_param.m_digest_sizing. */
 ulong m_digest_sizing;
 /** Default value for @c PFS_param.m_session_connect_attrs_sizing. */
 ulong m_session_connect_attrs_sizing;
};
`

#### 4.2 register_global_classes();

​ 注册全局的instrument监控项。在MySQL中，全局控制的instrument项有如下六种，分别解释如下：

 NAME
 解释

 global_table_io_class
 TABLE IO相关的instrument

 global_table_lock_class
 与TABLE 锁相关的instrument

 global_idle_class
 空闲事件相关的instrument

 global_metadata_class
 与metadata_lock相关的instrument

 global_error_class
 错误信息相关的instrument

 global_transaction_class
 事务相关的instrument

​ 在对上述六类全局instrument进行注册后，会根据Pfs_instr_config_array数组对相应的instrument进行配置。

#### 4.3 内存初始化

​ 此处完成对PFS所需要的内存进行初始化。内存分配可以分为三大类：

​ 1、init_xxx_class()

​ 该内存用于存储xxx类的instrument的配置项。这一类内存不会随着负载的增加而增大。内存大小与xxx类的 instruments的最大数目有关。此类内存在memory_summary_global_by_event_name表中记录为xxx_class类型。

​ 2、init_events_xxx_history_long()

​ 该内存用于记录全局的xxx类的历史信息。该信息记录在表EVENTS_xxx_HISTORY_LONG中。该类型的内存在表memory_summary_global_by_event_name中记录为：events_xxx_history_long。此类型的内存在pfs启动时分配，不会随着负载增加而增大。

​ 3、instrument相关的内存

​ 用于记录instrument所收集的信息。此类型内存多使用container内存模型，具体可以参见扩展阅读文章。此类型的内存大小会随着负载增加而增加。这也是PFS模块内存消耗的主要来源。

#### 4.4 SETUP_CONSUMERS表配置

​ 对表SETUP_CONSUMERS中的配置项进行配置。PFS采用生产-消费模型。当关闭对应的CONSUMERS时，对应的instrument也不会再收集相关的信息。当用户未进行人为配置时，默认的配置为：

`mysql> select * from performance_schema.setup_consumers;
+----------------------------------+---------+
| NAME | ENABLED |
+----------------------------------+---------+
| events_stages_current | NO |
| events_stages_history | NO |
| events_stages_history_long | NO |
| events_statements_current | YES |
| events_statements_history | YES |
| events_statements_history_long | NO |
| events_transactions_current | YES |
| events_transactions_history | YES |
| events_transactions_history_long | NO |
| events_waits_current | NO |
| events_waits_history | NO |
| events_waits_history_long | NO |
| events_parallel_query_current | YES |
| events_parallel_query_history | YES |
| global_instrumentation | YES |
| thread_instrumentation | YES |
| statements_digest | YES |
+----------------------------------+---------+
17 rows in set (0.04 sec)
`

​ 用户可通过在配置文件或启动参数中，设置对应的配置项。

`performance_schema_consumer_xxx = ON/OFF
`

​ 启动后，用户也可通过update操作对表performance_schema.setup_consumers进行更新，来修改对应配置。

`UPDATE performance_schema.setup_consumers SET ENABLED = 'YES' WHERE NAME IN ('global_instrumentation');
`

#### 4.5 注册PFS相关的接口到PSI中。

​ 在MySQL中定义了诸多PFS相关的接口。在peferformance模块对这些接口进行了实现。此处将pfs中对应接口的实现函数进行返回。

### 5. init_server_psi_keys()

​ 此函数注册所有除4.2中全局instrument以外的所有监控项。在具体实现上，通过调用mysql_xxx_register()函数实现对xxx类instrement的注册。以thread类为例，其注册instrument的函数调用链路为：

`init_server_psi_keys()
 -- mysql_thread_register()
 -- inline_mysql_thread_register()
 -- PSI_THREAD_CALL(register_thread)
 -- pfs_register_thread_v1()
 -- register_thread_class()
`

### 6. my_thread_global_reinit()

​ 至此已经完成了对PFS模块的初始化过程。由于当前线程中也存在一些需要检测的线程相关的锁对象，这里使用重新创建的方式，来监测这些锁对象。由于目前为止都是单个线程在运行，没有新的线程被创建，所以这里是线程安全的。

### 7. initialize_performance_schema_acl

​ 初始化performance_schema库的权限。由于无论是否启用Performance Schema功能，该库下面的表始终是可见的。因此需要进行权限控制。

## 二、功能拓展

​ 在了解Performance Schema的初始化过程后，便可以通过修改源码的形式，实现一些有趣的功能。

### 1、修改pfs默认配置

​ 当我们在默认状态下开启Performance Schema功能时，pfs会运行在一套默认参数下。此处可以通过修改源码的形式，配置多套默认参数。以方便不同规格实例进行切换。具体方法如下：

​ 在文件sql/sys_vars.cc中，新增变量performance_schema_params_control；

`static Sys_var_long Sys_pfs_params_control(
 "performance_schema_params_control,
 "control which params grop will be used."
 " Use 0，1，2 to choose different param_group.",
 READ_ONLY GLOBAL_VAR(pfs_param.m_params_control),
 CMD_LINE(REQUIRED_ARG), VALID_RANGE(0, 2),
 DEFAULT(0), BLOCK_SIZE(1), PFS_TRAILING_PROPERTIES);
`

​ 在文件storage/perfschema/pfs_server.h的PFS_global_param结构体中，新增变量如下变量:

`struct PFS_global_param {
 ...
 /* control which param_grop will be used.*/
 long m_params_control;
 ...
}
`

​ 结合上述分析，在PFS初始化过程中，参数配置主要是由pfs_automated_sizing()函数完成的。因此，我们仅需要在该函数之后，对默认参数进行覆盖即可。伪代码如下：

`int initialize_performance_schema() {
 ...
 pfs_automated_sizing(param);
 // 覆盖默认参数
 switch (param->m_params_control)
 {
 case 0:
 param_group_1;
 break;
 case 1:
 param_group_2;
 break;
 case 2:
 param_group_3;
 break;
 default:
 default_param_group;
 break; 
 }
 ...
 return 0;
}
`

​ 通过这种方式，可以通过该参数，控制pfs启动时选择不同组的配置参数，方便灵活配置。

### 2、修改默认监控项

​ 在PFS启动时，若用户未进行指定，会默认开启部分监控。在了解了PFS初始化的过程后，就可以通过修改源码的形式，修改setup_instrument表中的默认状态。

​ 根据上文介绍，instrument的配置项是记录在Pfs_instr_config_array数组中的。而该数组是由函数handle_early_options()进行填充的。因此，可以通过在handle_early_options()之后，修改Pfs_instr_config_array的值，便可以对instrument的状态进行配置。例如，如果希望关闭所有监控项，仅开启mdl锁的监控，可以进行如下修改：

```
int mysqld_main()
{
 ...
 ho_error = handle_early_options();
 add_pfs_instr_to_array("%", "OFF");
 add_pfs_instr_to_array("wait/lock/metadata/sql/mdl", "ON");
 ...
}

```

## 扩展阅读
1. MySQL · 资源管理 · PFS内存管理分析
2. MySQL · 引擎特性 · Performance_schema 内存分配
3. MySQL 8.0 Reference Manual · Performance Schema

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)