# MySQL · 引擎特性 · MySQL 状态信息Status实现

**Date:** 2019/03
**Source:** http://mysql.taobao.org/monthly/2019/03/09/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2019 / 03
 ](/monthly/2019/03)

 * 当期文章

 PgSQL · 特性分析 · 内存管理机制
* MongoDB · 同步工具 · MongoShake原理分析
* MySQL · InnoDB · Redo log
* MSSQL · 最佳实践 · Always Encrypted
* MySQL · 源码分析 · CHECK TABLE实现
* PgSQL · 原理介绍 · PostgreSQL中的空闲空间管理
* MySQL · 引擎特性 · 8.0 Descending Index
* 理论基础 · Raft phd 论文中的pipeline 优化
* MySQL · 引擎特性 · MySQL 状态信息Status实现
* PgSQL · 应用案例 · 使用PostgreSQL生成数独方法1

 ## MySQL · 引擎特性 · MySQL 状态信息Status实现 
 Author: 贤勇 

 ## 什么是MySQL状态信息

通过Show Status命令查看MySQL server的状态信息是MySQL日常运维中常见的诊断手段。这个命令可以返回在实例运行期间，或者是当前会话范围内的指定的状态信息。具体语法见[https://dev.mysql.com/doc/refman/8.0/en/show-status.html](https://dev.mysql.com/doc/refman/8.0/en/show-status.html)，
MySQL8.0可以支持的Status列表见 [https://dev.mysql.com/doc/refman/8.0/en/server-status-variables.html](https://dev.mysql.com/doc/refman/8.0/en/server-status-variables.html)。

## MySQL Status代码实现
本文将和大家一起看看Status的内部实现机制，相关的代码是基于MySQL 8.0。

MySQL使用了两张`Performance_Schema`数据库的表，`session_status`和`global_status`分别对应Session级别以及Global级别的状态信息访问。
两张表的定义如下，包含的字段是一样的。

`CREATE TABLE `session_status` (
 `VARIABLE_NAME` varchar(64) NOT NULL DEFAULT '',
 `VARIABLE_VALUE` varchar(1024) DEFAULT NULL
) ENGINE=PERFORMANCE_SCHEMA DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci

CREATE TABLE `global_status` (
 `VARIABLE_NAME` varchar(64) NOT NULL DEFAULT '',
 `VARIABLE_VALUE` varchar(1024) DEFAULT NULL
) ENGINE=PERFORMANCE_SCHEMA DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
`

`Show session|global Status`命令会在MySQL server层解析成对`performance_schema.session_status`或者`performance_schema.global_status`的Select语句。具体参见
`build_show_session_status`或`build_show_global_status`方法，这两个方法都会调用`build_query`方法来真正生成对应的SELECT_LEX，之后执行。

### 主要的数据结构

#### Status变量定义见`SHOW_VAR`结构体
`struct SHOW_VAR {
 const char *name; //Status名字
 char *value; //Status的值
 enum enum_mysql_show_type type; //Status类型，比如SHOW_ARRAY, SHOW_LONGLONG，不同的类型Status处理的方法不同
 enum enum_mysql_show_scope scope; //Status作用范围，包括 SHOW_SCOPE_UNDEF，SHOW_SCOPE_GLOBAL， SHOW_SCOPE_SESSION和SHOW_SCOPE_ALL
};
`
SHOW_SCOPE_UNDEF

* 未定义作用域,。当成SHOW_SCOPE_ALL对待

SHOW_SCOPE_GLOBAL

* 全局作用域。这种Status在整个MySQL实例生命周期里面只有一个，且全局的值。只可以通过SHOW GLOBAL STATUS，或者Select
Performance_Schema.global_status表查看。
例子: Aborted_connects, Binlog_cache_use

SHOW_SCOPE_SESSION

* Session级别作用域。这种Status只作用于各自的连接，不会在全局范围做聚集。只可以通过SHOW SESSION STATUS，或者Select Performance_Schema.session_status表查看。
例子: Compression, Last_query_cost

SHOW_SCOPE_ALL

* 这种Status变量可以作用于各自的连接，也会在全局范围做聚集。既可以通过SHOW SESSION STATUS，或者Select Performance_Schema.session_status表查看，也可以通过SHOW GLOBAL STATUS 或者 Select Performance_Schema.global_status查看。
例子: Bytes_sent, Open_tables, Com variables

#### all_status_vars

在mysqld.cc文件中定义的Status变量的全局动态SHOW_VAR数组，所有的server以及plugin的Status变量都会在里面定义。在MySQL实例初始化阶段，Status变量都会被组装成这个all_status_vars的全局动态数组，plugin在load的时候，带入的新的status变量会被加到这个数组里面。all_status_vars里面包含的Status的定义都是有序的存放的，以方便Show Status命令或者是对相关系统表session_status或者是global_status的访问。

#### System_status_var
thread级别的status变量，变量类型必须是long/ulong。

#### Global locks

LOCK_status 是一个全局mutex，它被用来在初始化阶段以及SHOW STATUS执行阶段保护all_status_vars。

#### Thread locks

在做多个thread Status聚合的时候后，global thread manager会持有两把锁来防止thread被移除。`Global_THD_manager::LOCK_thd_remove`和`Global_THD_manager::LOCK_thd_count`。 但，两种锁的持有时间不同，LOCK_thd_remove会在整个聚合执行期间一直持有， LOCK_thd_count 锁
会在当前thread list的快照做了copy之后就会释放。

### Show Status实现
对Performance_schema.session_status或者是Performance_schema.global_status的访问，会需要调用join_materialize_scan方法，并进一步调用ha_perfschema::rnd_init/rnd_next, PFS_variable_cache::materialize_all等。

取session域的Status的调用栈如下

`join_materialize_derived-> TABLE_LIST::materialize_derived->…
 ->ha_perfschema::rnd_init->table_session_status::rnd_init
 -> PFS_variable_cache<Status_variable>::materialize_all //Materialize output status
 -> PFS_status_variable_cache::do_materialize_all->PFS_status_variable_cache::manifest

join_materialize_derived-> TABLE_LIST::materialize_derived…->ha_perfschema::rnd_next->table_session_status::rnd_next //访问所有的status，做过滤
`
取global域的Status的调用栈如下，和session域的调用栈很类似

`join_materialize_derived-> TABLE_LIST::materialize_derived->…
 ->ha_perfschema::rnd_init->table_session_status::rnd_init
 -> PFS_variable_cache<Status_variable>::materialize_all //Materialize output status
 ->PFS_status_variable_cache::do_materialize_all->PFS_status_variable_cache::manifest

join_materialize_derived-> TABLE_LIST::materialize_derived->…
 ->ha_perfschema::rnd_next->table_session_status::rnd_next
`
type为SHOW_FUNC的Status，例如Aborted_connects，访问方法比较特别，会调用SHOW_VAR里面定义的函数来处理，

`PFS_status_variable_cache::manifest
…
 /* 
 If the value is a function reference, then execute the function and
 reevaluate the new SHOW_TYPE and value. Handle nested case where
 SHOW_FUNC resolves to another SHOW_FUNC.
 */
 if (show_var_ptr->type == SHOW_FUNC) {
 show_var_tmp = *show_var_ptr;
 /* 
 Execute the function reference in show_var_tmp->value, which returns
 show_var_tmp with a new type and new value.
 */
 for (const SHOW_VAR *var = show_var_ptr; var->type == SHOW_FUNC;
 var = &show_var_tmp) {
 ((mysql_show_var_func)(var->value))(thd, &show_var_tmp, value_buf.data); //调用指定的函数，函数名在status_vars数组里定义
 } 
 show_var_ptr = &show_var_tmp;
 } 

`
例如取Aborted_connects，就会调用show_aborted_connects这个函数来获取真正的值。

`{"Aborted_connects", (char *)&show_aborted_connects, SHOW_FUNC, SHOW_SCOPE_GLOBAL},
`
### 如何添加一个新的Status

MySQL当前的架构已经对Status扩展做了很好的支持。我们如果想添加一个新的Status，一般的步骤是在System_status_var里面添加一个相应的变量，并在status_vars数组里面
也对应添加一个对应的SHOW_VAR条目。需要处理的是对Status 变量做累计，这个就要根据具体的逻辑了。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)