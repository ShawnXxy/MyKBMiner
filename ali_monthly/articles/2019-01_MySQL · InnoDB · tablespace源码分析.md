# MySQL · InnoDB · tablespace源码分析

**Date:** 2019/01
**Source:** http://mysql.taobao.org/monthly/2019/01/08/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2019 / 01
 ](/monthly/2019/01)

 * 当期文章

 POLARDB · 理论基础 · 数据库故障恢复机制的前世今生
* POLARDB · 最佳实践 · POLARDB不得不知道的秘密(二)
* MongoDB · 原理介绍 · MongoDB从事务到复制
* PgSQL · 引擎特性 · PostgreSQL 并行查询概述
* MSSQL · 最佳实践 · 如何打码隐私数据列
* Redis · 引擎特性 · Lua脚本新姿势
* Mariadb · 源码分析 · proxy protocol
* MySQL · InnoDB · tablespace源码分析
* MySQL · 最佳实践 · MySQL中的IO共享操作
* PgSQL · 应用案例 · native partition 分区表性能优化

 ## MySQL · InnoDB · tablespace源码分析 
 Author: diaoliang 

 ## 简介

这里所有的代码都是基于MySQL 8.0.

首先来看tablespace的定义：

 A data file that can hold data for one or more InnoDB tables and associated indexes.

这里system tablespace是一个特殊的tablespace,他包含了很多数据文件(ibdata files).而且如果没有设置 file-per-table的话，所有的新创建的表的数据以及索引信息都会保存在它里面.

下面就是ibdata可能包含的内容

 A set of files with names such as ibdata1, ibdata2, and so on, that make up the InnoDB system tablespace. These files contain metadata about InnoDB tables, (the InnoDB data dictionary), and the storage areas for one or more undo logs, the change buffer, and the doublewrite buffer.

假设设置了file-per-table(默认打开),那么每一个表都会有自己的tablespace文件(.ibd).

 The .ibd file extension does not apply to the system tablespace, which consists of one or more ibdata files.

还有一种就是temporary tablespace，也就是临时表的tablespace.

 InnoDB uses two types of temporary tablespace. Session temporary tablespaces store user-created temporary tables and internal temporary tables created by the optimizer.

## 源码分析

通过上面我们可以简单的认为tablespace就是对于数据库中table的内部抽象.

既然我们已经知道tablespace是和table相关的，那么我们就按照这个逻辑来分析源码.

### shared tablespace
首先来看system tablespace的创建，在InnoDB中每一个tablespace都会有一个uint32类型的id,每一个唯一的id用来表示对应的tablespace,而system tablespace的space id就是0.

`static const space_id_t TRX_SYS_SPACE = 0;
`

而由于只有system tablespace和temporary tablespace是共享的，因此他们在InnoDB中有专门的数据结构来表示他们(Tablespace表示所有的shared tablespace的基类).

`class SysTablespace : public Tablespace {
 public:
 SysTablespace()
 : m_auto_extend_last_file(),
 m_last_file_size_max(),
 m_created_new_raw(),
 m_is_tablespace_full(false),
 m_sanity_checks_done(false) {
 /* No op */
 }
`

而对应的变量则是

`/** The control info of the system tablespace. */
SysTablespace srv_sys_space;

/** The control info of a temporary table shared tablespace. */
SysTablespace srv_tmp_space;

`

tablespace数据文件的创建则是在SysTablespace::open_or_create中，因此我们来看这个函数.这个函数主要功能是遍历当前system tablespace的所有文件(m_files),然后存在就打开，不存在就创建对应的文件.

`files_t::iterator begin = m_files.begin();
 files_t::iterator end = m_files.end();

 ut_ad(begin->order() == 0);

 for (files_t::iterator it = begin; it != end; ++it) {
 if (it->m_exists) {
 err = open_file(*it);

 } else {
 err = create_file(*it);
 }
`

当打开文件之后，将会把打开的文件进行缓存(InnoDB会将所有的tablespace缓存在fil_system中).

`/* Close the curent handles, add space and file info to the
 fil_system cache and the Data Dictionary, and re-open them
 in file_system cache so that they stay open until shutdown. */
 ulint node_counter = 0;
 for (files_t::iterator it = begin; it != end; ++it) {
 it->close();
 it->m_exists = true;
..............................
 page_no_t max_size =
 (++node_counter == m_files.size()
 ? (m_last_file_size_max == 0 ? PAGE_NO_MAX : m_last_file_size_max)
 : it->m_size);

 /* Add the datafile to the fil_system cache. */
 if (!fil_node_create(it->m_filepath, it->m_size, space,
 it->m_type != SRV_NOT_RAW, it->m_atomic_write,
 max_size)) {
 err = DB_ERROR;
 break;
 }
 }
`

fil_node_create就是将当前的文件加入到文件cache中，这是因为在InnoDB中所有的文件都会统一管理，包括redo/undo，所有的tablespace都会根据他们的spaceid进行缓存.

`char *fil_node_create(const char *name, page_no_t size, fil_space_t *space,
 bool is_raw, bool atomic_write, page_no_t max_pages) {
 auto shard = fil_system->shard_by_id(space->id);

 fil_node_t *file;

 file = shard->create_node(name, size, space, is_raw,
 IORequest::is_punch_hole_supported(), atomic_write,
 max_pages);

 return (file == nullptr ? nullptr : file->name);
}
`

此时的疑问就是m_files是何时被初始化的，也就是system tablespace文件名的初始化，这里system tablespace的文件名初始化是在数据库init的时候被初始化的，因此我们来看相关代码.

InnoDB中有一个变量叫做innobase_data_file_path，这个变量是一个字符串，这个字符串包含了所有的system tablespace需要创建 的文件以及一些属性，这个字符串默认值是在InnoDB引擎初始化的时候初始化的.

` /* Set default InnoDB temp data file size to 12 MB and let it be
 auto-extending. */
 if (!innobase_data_file_path) {
 innobase_data_file_path = (char *)"ibdata1:12M:autoextend";
 }
`

可以看到默认只创建ibdata1文件，并且大小为12m,自动扩展，而innobase_data_file_path的格式如下

 innodb_data_file_path=datafile_spec1[;datafile_spec2]..

file_name:file_size[:autoextend[:max:max_file_size]]

因此system tablespace创建文件也会根据这个配置来创建，也就是当InnoDB参数解析完毕之后进入system tablespace的文件创建.

` if (int error = innodb_init_params()) {
 DBUG_RETURN(error);
 }

 /* After this point, error handling has to use
 innodb_init_abort(). */

 if (!srv_sys_space.parse_params(innobase_data_file_path, true)) {
 ib::error(ER_IB_MSG_545)
 << "Unable to parse innodb_data_file_path=" << innobase_data_file_path;
 DBUG_RETURN(innodb_init_abort());
 }
`

parse_params这个函数将会解析innobase_data_file_path,然后根据不同的属性创建对应的文件，先来看参数解析.

`/*---------------------- PASS 1 ---------------------------*/
 /* First calculate the number of data files and check syntax. */
 while (*ptr != '\0') {
 filepath = ptr;

 ptr = parse_file_name(ptr);

 if (ptr == filepath) {
...........................
 return (false);
 }

 if (*ptr == '\0') {
.................................

 return (false);
 }

 ptr++;

 size = parse_units(ptr);

 if (size == 0) {
.........................
 return (false);
 }

 if (0 == strncmp(ptr, ":autoextend", (sizeof ":autoextend") - 1)) {
 ptr += (sizeof ":autoextend") - 1;

 if (0 == strncmp(ptr, ":max:", (sizeof ":max:") - 1)) {
 ptr += (sizeof ":max:") - 1;

 page_no_t max = parse_units(ptr);

 if (max < size) {
 goto invalid_size;
 }
 }

 if (*ptr == ';') {
..................................
 return (false);
 }
 }

 if (0 == strncmp(ptr, "new", (sizeof "new") - 1)) {
 ptr += (sizeof "new") - 1;
 }

 if (0 == strncmp(ptr, "raw", (sizeof "raw") - 1)) {
...............................
 return (false);
 }

 ptr += (sizeof "raw") - 1;
 }

 ++n_files;

 if (*ptr == ';') {
 ptr++;
 } else if (*ptr != '\0') {
.......................
 return (false);
 }
 }

 if (n_files == 0) {
......................
 return (false);
 }
`

然后第二步就是存储对应的文件名以及属性到m_files中，这里有一个重要的数据结构就是Datafile,每一个数据文件在InnoDB中的抽象就是Datafile.

`while (*ptr != '\0') {
 filepath = ptr;

 ptr = parse_file_name(ptr);

 if (*ptr == ':') {
 /* Make filepath a null-terminated string */
 *ptr = '\0';
 ptr++;
 }

 size = parse_units(ptr);
 ut_ad(size > 0);

 if (0 == strncmp(ptr, ":autoextend", (sizeof ":autoextend") - 1)) {
 m_auto_extend_last_file = true;

 ptr += (sizeof ":autoextend") - 1;

 if (0 == strncmp(ptr, ":max:", (sizeof ":max:") - 1)) {
 ptr += (sizeof ":max:") - 1;

 m_last_file_size_max = parse_units(ptr);
 }
 }

 m_files.push_back(Datafile(filepath, flags(), size, order));
 Datafile *datafile = &m_files.back();
 datafile->make_filepath(path(), filepath, NO_EXT);

 if (0 == strncmp(ptr, "new", (sizeof "new") - 1)) {
 ptr += (sizeof "new") - 1;
 }

 if (0 == strncmp(ptr, "raw", (sizeof "raw") - 1)) {
 ut_a(supports_raw);

 ptr += (sizeof "raw") - 1;

 /* Initialize new raw device only during initialize */
 m_files.back().m_type =
#ifndef UNIV_HOTBACKUP
 opt_initialize ? SRV_NEW_RAW : SRV_OLD_RAW;
#else /* !UNIV_HOTBACKUP */
 SRV_OLD_RAW;
#endif /* !UNIV_HOTBACKUP */
 }

 if (*ptr == ';') {
 ++ptr;
 }
 order++;
 }
`

### 非共享的tablespace
然后我们来看非共享的tablespace的创建,一般来说每创建一个表都会创建一个新的ibd文件，也就是tablespace. 而这种tablespace以及Redolog/undolog都是属于fil_space_t这个结构体.通过上面的代码我们知道system tablespace最终也是创建一个fil_space_t然后再接入整个系统的tablespace的管理的.

这里先来看两个特殊的tablespace,也就是REDO log和UNDO log，由于这两个log也都是磁盘上的文件，因此在InnoDB中会讲这两个log文件作为一种特殊的tablespace，来看他们的初始化.

`static dberr_t create_log_files(char *logfilename, size_t dirnamelen, lsn_t lsn,
 char *&logfile0, lsn_t &checkpoint_lsn) {
........................
 /* Disable the doublewrite buffer for log files. */
 fil_space_t *log_space = fil_space_create(
 "innodb_redo_log", dict_sys_t::s_log_space_first_id,
 fsp_flags_set_page_size(0, univ_page_size), FIL_TYPE_LOG);
.......................
}

`

下面的undo log.

`static dberr_t srv_undo_tablespace_open(space_id_t space_id) {
....................................
 space = fil_space_create(undo_name, space_id, flags, FIL_TYPE_TABLESPACE);
..............
}
`

以后我们详细分析undolog/redolog的时候会再来分析这个地方

接下来我们就来看当一般表的创建的时候会发生什么(设置了file-per-table)，对于一般的表，创建tablespace是跟随着ibd文件一起创建的(上面的都是在初始化的时候创建)。

```
static bool dd_create_hardcoded(space_id_t space_id, const char *filename) {
 page_no_t pages = FIL_IBD_FILE_INITIAL_SIZE;

 dberr_t err = fil_ibd_create(space_id, dict_sys_t::s_dd_space_name, filename,
 predefined_flags, pages);

 if (err == DB_SUCCESS) {
 mtr_t mtr;
 mtr.start();

 bool ret = fsp_header_init(space_id, pages, &mtr, true);

 mtr.commit();

 if (ret) {
 btr_sdi_create_index(space_id, false);
 return (false);
 }
 }

 return (true);
}

```

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)