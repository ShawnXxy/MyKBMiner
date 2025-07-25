# MySQL ·  引擎特性 ·  INFORMATION_SCHEMA系统表的实现

**Date:** 2018/02
**Source:** http://mysql.taobao.org/monthly/2018/02/08/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 02
 ](/monthly/2018/02)

 * 当期文章

 MySQL · 源码分析 · 常用SQL语句的MDL加锁源码分析
* Influxdb · 源码分析 · Influxdb cluster实现探究
* MySQL · 源码分析 · 权限浅析
* PgSQL · 源码分析 · AutoVacuum机制之autovacuum worker
* MSSQL · 最佳实践 · 数据库恢复模式与备份的关系
* PgSQL · 最佳实践 · 利用异步 dblink 快速从 oss 装载数据
* MySQL · 源码分析 · 新连接的建立
* MySQL · 引擎特性 · INFORMATION_SCHEMA系统表的实现
* MySQL · 最佳实践 · 在线收缩UNDO Tablespace
* PgSQL · 应用案例 · 自定义并行聚合函数的原理与实践

 ## MySQL · 引擎特性 · INFORMATION_SCHEMA系统表的实现 
 Author: 元镇 

 ## 简介

在MySQL中， INFORMATION_SCHEMA 信息数据库提供了访问数据库元数据的方式，其中保存着关于MySQL服务器所维护的所有其他数据库的信息。如数据库名，数据库的表，列的数据类型与访问权限等。在INFORMATION_SCHEMA中，有数个只读表。它们实际上是视图，也不是基本表，因此，你将无法看到与之相关的任何文件。下面将介绍如何新加一个INFORMATION_SCHEMA系统表，以能够通过查询表的方式来查询我们希望得到的元数据信息。

## INFORMATION_SCHEMA表是作为MySQL的插件来实现的

INFORMATION_SCHEMA和我们经常讲到的引擎插件（Engine plugin）MYSQL_STORAGE_ENGINE_PLUGIN = 1类似，作为一个MySQL的插件来实现的。INFORMATION_SCHEMA的插件类型是MYSQL_INFORMATION_SCHEMA_PLUGIN = 4 。

`#define MYSQL_UDF_PLUGIN 0 /* User-defined function */
#define MYSQL_STORAGE_ENGINE_PLUGIN 1 /* Storage Engine */
#define MYSQL_FTPARSER_PLUGIN 2 /* Full-text parser plugin */
#define MYSQL_DAEMON_PLUGIN 3 /* The daemon/raw plugin type */
#define MYSQL_INFORMATION_SCHEMA_PLUGIN 4 /* The I_S plugin type */
#define MYSQL_AUDIT_PLUGIN 5 /* The Audit plugin type */
#define MYSQL_REPLICATION_PLUGIN 6 /* The replication plugin type */
#define MYSQL_AUTHENTICATION_PLUGIN 7 /* The authentication plugin type */
#define MYSQL_VALIDATE_PASSWORD_PLUGIN 8 /* validate password plugin type */
`
## INFORMATION_SCHEMA表插件接口

Mysql为要定义的INFORMATION_SCHEMA表提供了如下插件接口。主要通过st_mysql_plugin 结构实现的。

`struct st_mysql_plugin
{
 int type; /* the plugin type (a MYSQL_XXX_PLUGIN value) */
 void *info; /* pointer to type-specific plugin descriptor */
 const char *name; /* plugin name */
 const char *author; /* plugin author (for I_S.PLUGINS) */
 const char *descr; /* general descriptive text (for I_S.PLUGINS) */
 int license; /* the plugin license (PLUGIN_LICENSE_XXX) */
 int (*init)(MYSQL_PLUGIN); /* the function to invoke when plugin is loaded */
 int (*deinit)(MYSQL_PLUGIN);/* the function to invoke when plugin is unloaded */
 unsigned int version; /* plugin version (for I_S.PLUGINS) */
 struct st_mysql_show_var *status_vars;
 struct st_mysql_sys_var **system_vars;
 void * __reserved1; /* reserved for dependency checking */
 unsigned long flags; /* flags for plugin */
};
`

此接口要提供这个插件的类型。这里我们要添加一个INFORMATION_SCHEMA表，所以type就是MYSQL_INFORMATION_SCHEMA_PLUGIN； name就是你要定义的表名字, 就像INFORMATION_SCHEMA中的表INNODB_SYS_DATAFILES 一样。
这里比如INNODB_MY_TABLE； 另外比较重要的两个字段就是init和deinit，这两个字段分别是在这个插件在装载和卸载时调用的函数。如果没有特殊的需求deinit函数可以直接使用系统提供的i_s_common_deinit 通用函数。
在st_mysql_plugin结构中，init字段用来提供在这个插件装载时调用的函数。其实现主要是通过为ST_SCHEMA_TABLE结构提供INFORMATION_SCHEMA表结构定义和表填充数据函数。

`i_s_innodb_my_table_init(
 void* p) /*!< in/out: table schema object */
{ 
 ST_SCHEMA_TABLE* schema;

 schema = reinterpret_cast<ST_SCHEMA_TABLE*>(p);
 schema->fields_info = innodb_my_table_field;
 schema->fill_table = i_s_innodb_my_table_fill_table;
 
 DBUG_RETURN(0);
} 
`

其中innodb_my_table_field就是此新加入INFORMATION_SCHEMA表结构定义；innodb_my_table_fill_table就是当我们查询这张表时，其中显示的数据就是通过这个函数提供的。

例如我们定义的新表插件接口如下：

`UNIV_INTERN struct st_mysql_plugin i_s_innodb_my_table =
{
 /* the plugin type (a MYSQL_XXX_PLUGIN value) */
 /* int */
 STRUCT_FLD(type, MYSQL_INFORMATION_SCHEMA_PLUGIN),

 /* pointer to type-specific plugin descriptor */
 /* void* */
 STRUCT_FLD(info, &i_s_info),

 /* plugin name */
 /* const char* */
 STRUCT_FLD(name, "INNODB_MY_TABLE”),

 /* plugin author (for SHOW PLUGINS) */
 /* const char* */
 STRUCT_FLD(author, "Alibaba"),
 
 /* general descriptive text (for SHOW PLUGINS) */
 /* const char* */
 STRUCT_FLD(descr, "InnoDB My table info.”),

 /* the plugin license (PLUGIN_LICENSE_XXX) */
 /* int */
 STRUCT_FLD(license, PLUGIN_LICENSE_GPL),

 /* the function to invoke when plugin is loaded */
 /* int (*)(void*); */
 STRUCT_FLD(init, i_s_innodb_my_table_init),

 /* the function to invoke when plugin is unloaded */
 /* int (*)(void*); */
 STRUCT_FLD(deinit, i_s_common_deinit),
 /* plugin version (for SHOW PLUGINS) */
 /* unsigned int */
 STRUCT_FLD(version, INNODB_VERSION_SHORT),

 /* struct st_mysql_show_var* */
 STRUCT_FLD(status_vars, NULL),

 /* struct st_mysql_sys_var** */
 STRUCT_FLD(system_vars, NULL),

 /* reserved for dependency checking */
 /* void* */
 STRUCT_FLD(__reserved1, NULL),

 /* Plugin flags */
 /* unsigned long */
 STRUCT_FLD(flags, 0UL),
};
`
## INFORMATION_SCHEMA表的结构定义
表结构定义在一个ST_FIELD_INFO结构中，这个结构定义了要添加到INFORMATION_SCHEMA表的各个字段，包括字段名、字段类型和字段长度等信息。这里就涉及到如何将这里定义的各个字段与在st_mysql_plugin 结构中定的表信息关联起来。这里就要用到在st_mysql_plugin 结构里定义的初始化函数有关了。实现init函数的参数，是一个void类型的输入参数，这个参数在information schema的插件中，就是一个ST_SCHEMA_TABLE结构指针，就在这个结构中要提供这个表的字段信息，以及查询表时调用填充数据的函数。

`static ST_FIELD_INFO innodb_my_table_field[] =
{ 
#define IDX_MY_TABLE_FIELD_0 0
 {STRUCT_FLD(field_name, “field0”),
 STRUCT_FLD(field_length, MY_INT64_NUM_DECIMAL_DIGITS),
 STRUCT_FLD(field_type, MYSQL_TYPE_LONGLONG),
 STRUCT_FLD(value, 0),
 STRUCT_FLD(field_flags, MY_I_S_UNSIGNED),
 STRUCT_FLD(old_name, ""),
 STRUCT_FLD(open_method, SKIP_OPEN_TABLE)},

#define IDX_MY_TABLE_FIELD_1 1 
 {STRUCT_FLD(field_name, “field1”),
 STRUCT_FLD(field_length, MY_INT64_NUM_DECIMAL_DIGITS),
 STRUCT_FLD(field_type, MYSQL_TYPE_LONGLONG),
 STRUCT_FLD(value, 0),
 STRUCT_FLD(field_flags, MY_I_S_UNSIGNED),
 STRUCT_FLD(old_name, ""),
 STRUCT_FLD(open_method, SKIP_OPEN_TABLE)},
….
}
`

填充函数就是用来实现往INFORMATION_SCHEMA里填充数据的函数，可以根据具体的需求和要填充的数据，根据实际情况读取引擎内部的状态信息，写入到这个表中。比如上面在初始化装载函数中赋值给field_table字段的函数i_s_innodb_my_table_fill_table（）。

## 加表到INFORMATION_SCHEMA元数据库
通过上述几步就把这个表定义出来了。 如何把这个表真正的加入到INFORMATION_SCHEMA里，客户端可以通过查询语句查询这张表呢？为了实现这个目标就要把这张表加入到ha_innodb.cc文件里，从mysql_declare_plugin(innobase) 到mysql_declare_plugin_end 之间的结构里。在这里我们可以看到已经定义了一系列的INFORMATION_SCHEMA表，包含常见到的i_s_innodb_trx、i_s_innodb_locks和i_s_innodb_sys_tables等表，只要把我们新实现的插件接口i_s_innodb_my_table加入到这个结构中，就成功把这张表加入了INFORMATION_SCHEMA元数据库中了。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)