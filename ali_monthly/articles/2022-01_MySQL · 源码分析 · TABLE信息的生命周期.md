# MySQL · 源码分析 · TABLE信息的生命周期

**Date:** 2022/01
**Source:** http://mysql.taobao.org/monthly/2022/01/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2022 / 01
 ](/monthly/2022/01)

 * 当期文章

 DataBase · 理论基础 · B+树数据库加锁历史
* MySQL · 引擎特性 · Redo Log record编码格式
* SQL Server · 引擎特性 · 从SQL Server看列式存储
* MySQL · 源码分析 · TABLE信息的生命周期

 ## MySQL · 源码分析 · TABLE信息的生命周期 
 Author: 桐铭 

 MySQL通过TABLE对象进行表的读写等操作，对于构建TABLE对象所需的表定义相关信息，MySQL会通过`Dictionary_client`与DD模块进行交互。DD模块通过多级缓存的结构提供了高效而安全的DD信息访问方式，具体介绍见[http://mysql.taobao.org/monthly/2021/08/02/](http://mysql.taobao.org/monthly/2021/08/02/)。本文旨在讨论TABLE对象及其用到的DD模块中TableImpl信息的生命周期，包括他们的构建、缓存、清理和移除。
​

##相关数据结构：
TABLE结构描述表的相关信息，其中表定义相关的一些DD信息，如包含的字段等，记录在`TABLE_SHARE`中，`handler`是该表所使用的存储引擎接口，两个`TABLE`指针串起了正在操作这个`TABLE`对象的THD所控制的所有`TABLE`对象，即`THD::opened_tables`。另外，`TABLE`结构中还有字段信息、保存操作过程中读写的数据的内存区域等，与本文主题无关。

`struct TABLE {
 TABLE_SHARE *s{nullptr};
 handler *file{nullptr};
 TABLE *next{nullptr}, *prev{nullptr};

 private:
 TABLE *cache_next{nullptr}, **cache_prev{nullptr};

 /*
 Give table_cache_element access to the above two members to allow
 using them for linking TABLE objects in a list.
 */
 friend class table_cache_element;

 public:
 
 THD *in_use{nullptr}; /* Which thread uses this */
 Field **field{nullptr}; /* Pointer to fields */

 bool m_needs_reopen{false};

 }
`

`table_cache_element`是table cache中的一个元素，它表示了一张表在线程对应的table cache中构建的所有`TABLE`实例，其中有一些位于`thd->opened_tables`中正在使用（`used_tables`），也有一些已经被释放，缓存供后续使用（`free_tables`）。这些表共用一个TABLE_SHARE对象维护dd信息等。

`/**
 Element that represents the table in the specific table cache.
 Plays for table cache instance role similar to role of TABLE_SHARE
 for table definition cache.

 It is an implementation detail of table_cache and is present
 in the header file only to allow inlining of some methods.
*/

class table_cache_element {
 TABLE_list used_tables;
 TABLE_list free_tables;
 TABLE_SHARE *share;
};
`

`table_cache`维护了相关的几个THD所使用的所有`table_cache_element`，即所有正在使用或曾经打开过的`TABLE`对象。`m_unused_tables`是所有`table_cache_element`中free_tables的并集。

`class table_cache {
 /**
 The hash of table_cache_element objects, each table/table share that
 has any TABLE object in the table_cache has a table_cache_element from
 which the list of free TABLE objects in this table cache AND the list
 of used TABLE objects in this table cache is stored.
 We use table_cache_element::share::table_cache_key as key for this hash.
 */
 std::unordered_map<std::string, std::unique_ptr<table_cache_element>> m_cache;

 /**
 List that contains all TABLE instances for tables in this particular
 table cache that are in not use by any thread. Recently used TABLE
 instances are appended to the end of the list. Thus the beginning of
 the list contains which have been least recently used.
 */
 TABLE *m_unused_tables;

 /**
 Total number of TABLE instances for tables in this particular table
 cache (both in use by threads and not in use).
 This value summed over all table caches is accessible to users as
 Open_tables status variable.
 */
 uint m_table_count;
};
`

`table_cache_manager`是所有table cache的集合，该对象有一个全局单例。需要操作table cache保存或者借用空闲`TABLE`对象时，根据THD ID访问对应的table cache并进行相应操作。

`/**
 Container class for all table cache instances in the system.
*/

class table_cache_manager {
 /**
 An array of table_cache instances.
 Only the first table_cache_instances elements in it are used.
 */
 table_cache m_table_cache[MAX_table_cacheS];
};

extern table_cache_manager table_cache_manager;
`

##DD信息创建：create table
create table时会通过`open_table`调用`check_if_table_exists`判断表是否存在，以便在表存在时报错。这里会产生一次空读，这次空读会完整读穿DD模块直至底层存储引擎，不过由于此时相关DD信息尚未构建，暂时略去。之后在`rea_create_base_table`时才会构建DD信息并保存到存储引擎InnoDB。
​

`static bool rea_create_base_table(...) {

// 构造dd::Table对象，并用create_info中的信息填充
 std::unique_ptr<dd::Table> table_def_res =
 dd::create_table(thd, sch_obj, table_name, create_info, create_fields,
 key_info, keys, keys_onoff, fk_key_info, fk_keys, file);

 if (do_not_store_in_dd) {
 ......
 } else {
 
 // Storage_adapter::store(m_thd, object) 保存在Storage_adapter（innoDB）中。然后调用register_uncommitted_object将table_def_res的clone()保存在Dictionary_client的m_registry_uncommitted中。
 bool result = thd->dd_client()->store(table_def_res.get());

 ......

 // 获取Dictionary_client的m_registry_uncommitted中dd::Table的clone() 【为啥不直接用table_def_res，反正存进去的也是clone】
 if (thd->dd_client()->acquire_for_modification(db, table_name, &table_def))
 DBUG_RETURN(true);
 }

...... (根据前面的table_def信息构造TABLE_SHARE并让存储引擎完成实际的table create，与本文主题无关)

}
`

前面的`rea_create_base_table`中，表定义信息被保存到了`Dictionary_client`的`m_registry_uncommitted`中，即保存在多级缓存中的局部缓存中，并保存在了存储引擎，但是并不会放置到共享缓存`Shared_dictionary_cache`层中。接下来，事务提交过程会在将dd表信息落盘的同时清理之前放入缓存的表信息：

`bool trans_commit_implicit(THD *thd, bool ignore_global_read_lock) {

 ......
 
 // 调用各个dd数据类型的remove_uncommitted_objects清理dd cache中的Abstract_table、Schema、Tablespace等各种缓存
 thd->dd_client()->commit_modified_objects();
}

void Dictionary_client::remove_uncommitted_objects(
 bool commit_to_shared_cache) {

 if (commit_to_shared_cache) {
 typename Multi_map_base<typename T::Cache_partition>::Const_iterator it;
 
 // 防止Shared_dictionary_cache中残留有新建的表的信息，为m_registry_uncommitted中的每一个表信息调用invalidate，以清理Shared_dictionary_cache中相应的表信息（如果有）
 for (it = m_registry_uncommitted.begin<typename T::Cache_partition>();
 it != m_registry_uncommitted.end<typename T::Cache_partition>();
 it++) {
 typename T::Cache_partition *uncommitted_object =
 const_cast<typename T::Cache_partition *>(it->second->object());
 DBUG_ASSERT(uncommitted_object != nullptr);

 // 防止Shared_dictionary_cache中残留有新建的表的信息
 invalidate(uncommitted_object);
 }

 // 仅对于bootstrap过程中initialize的dd表（mysql.tables等），将uncommitted cache中的表信息转移到Shared_dictionary_cache和m_registry_committed中
 if (m_thd->is_dd_system_thread() &&
 bootstrap::DD_bootstrap_ctx::instance().get_stage() <
 bootstrap::Stage::FINISHED) {
 // We must do this in two iterations to handle situations where two
 // uncommitted objects swap names.
 for (it = m_registry_uncommitted.begin<typename T::Cache_partition>();
 it != m_registry_uncommitted.end<typename T::Cache_partition>();
 it++) {
 typename T::Cache_partition *uncommitted_object =
 const_cast<typename T::Cache_partition *>(it->second->object());
 DBUG_ASSERT(uncommitted_object != nullptr);

 Cache_element<typename T::Cache_partition> *element = NULL;

 // In put, the reference counter is stepped up, so this is safe.
 Shared_dictionary_cache::instance()->put(
 static_cast<const typename T::Cache_partition *>(
 uncommitted_object->clone()),
 &element);

 m_registry_committed.put(element);
 // Sign up for auto release.
 m_current_releaser->auto_release(element);
 }
 }
 } // commit_to_shared_cache

 // 清理需要清理的m_registry_uncommitted和m_registry_dropped中的信息
 m_registry_uncommitted.erase<typename T::Cache_partition>();
 m_registry_dropped.erase<typename T::Cache_partition>();
}
`

以上，create user table的过程中不会遗留任何类型的DD cache信息，因为对所有类型的cache均调用了`remove_uncommitted_objects`函数，保证临时产生的`dd::Table`对象不会残留在缓存中，而仅仅是进行了落盘。但这并不代表建表过程完全不会污染内存缓存信息，比如，建表过程中创建tablespace所调用的`fil_space_create`会将表空间信息保存在全局变量`fil_system`对应的`Fil_shard`中。
​

##DD信息读取和TABLE对象构建：open_table

所有的DML、DQL在访问表数据之前都要先访问表的`Table_Impl`等DD信息，以便读取、解析或写入相关的数据。这一访问过程在开表过程中构造`TABLE_SHARE`时实现。

`TABLE_SHARE *get_TABLE_SHARE(THD *thd, const char *db, const char *table_name,
 const char *key, size_t key_length, bool open_view,
 bool open_secondary) {
 ......
 /*
 Read table definition from the cache. If the share is being opened,
 wait for the appropriate condition. The share may be destroyed if
 open fails, so after cond_wait, we must repeat searching the
 hash table.
 */
 for (;;) {
 auto it = table_def_cache->find(string(key, key_length));
 // table_def_cache中找不到，确认加了schema的MDL锁之后建立新的TABLE_SHARE
 if (it == table_def_cache->end()) {
 if (thd->mdl_context.owns_equal_or_stronger_lock(
 MDL_key::SCHEMA, db, "", MDL_INTENTION_EXCLUSIVE)) {
 break;
 }
 mysql_mutex_unlock(&LOCK_open);

 if (dd::mdl_lock_schema(thd, db, MDL_TRANSACTION)) {
 // Lock LOCK_open again to preserve function contract
 mysql_mutex_lock(&LOCK_open);
 DBUG_RETURN(nullptr);
 }

 mysql_mutex_lock(&LOCK_open);
 // Need to re-try the find after getting the mutex again
 continue;
 }
 share = it->second.get();
 // m_open_in_progress说明有其他线程创建了TABLE_SHARE并放进table_def_cache，但此时尚未完成信息填充，于是等待创建线程填充信息完成后通过COND_open通知，然后再次进入循环查询。
 if (!share->m_open_in_progress)
 DBUG_RETURN(process_found_TABLE_SHARE(thd, share, open_view));

 DEBUG_SYNC(thd, "get_share_before_COND_open_wait");
 mysql_cond_wait(&COND_open, &LOCK_open);
 }

 /*
 申请新的TABLE_SHARE
 */
 if (!(share = alloc_TABLE_SHARE(db, table_name, key, key_length,
 open_secondary))) {
 DBUG_RETURN(NULL);
 }

 /*
 We assign a new table id under the protection of LOCK_open.
 We do this instead of creating a new mutex
 and using it for the sole purpose of serializing accesses to a
 static variable, we assign the table id here. We assign it to the
 share before inserting it into the table_def_cache to be really
 sure that it cannot be read from the cache without having a table
 id assigned.

 CAVEAT. This means that the table cannot be used for
 binlogging/replication purposes, unless get_TABLE_SHARE() has been
 called directly or indirectly.
 */
 assign_new_table_id(share);

 // 将TABLE_SHARE放入table_def_cache，此时另一个线程如果进入前面的循环，他就已经能读到了，但由于信息还未完成填充，所以还不能用。
 table_def_cache->emplace(to_string(share->table_cache_key),
 unique_ptr<TABLE_SHARE, TABLE_SHARE_deleter>(share));

 /*
 We must increase ref_count prior to releasing LOCK_open
 to keep the share from being deleted in tdc_remove_table()
 and TABLE_SHARE::wait_for_old_version. We must also set
 m_open_in_progress to indicate allocated but incomplete share.
 */
 share->increment_ref_count(); // Mark in use
 share->m_open_in_progress = true; // Mark being opened

 /*
 Temporarily release LOCK_open before opening the table definition,
 which can be done without mutex protection.
 */
 mysql_mutex_unlock(&LOCK_open);

#if defined(ENABLED_DEBUG_SYNC)
 if (!thd->is_attachable_ro_transaction_active())
 DEBUG_SYNC(thd, "get_share_before_open");
#endif

 {
 // We must make sure the schema is released and unlocked in the right order.
 dd::cache::Dictionary_client::Auto_releaser releaser(thd->dd_client());
 const dd::Schema *sch = nullptr;
 const dd::Abstract_table *abstract_table = nullptr;
 
 if (thd->dd_client()->acquire(share->db.str, &sch) ||
 thd->dd_client()->acquire(share->db.str, share->table_name.str,
 &abstract_table)) {
 }
 ......
 // 通过刚刚读到的abstract_table 填充TABLE_SHARE中的一些细节信息
 }

 /*
 Get back LOCK_open before continuing. Notify all waiters that the
 opening is finished, even if there was a failure while opening.
 */
 
 // 设置m_open_in_progress表示信息已完成填充，并通过COND_open通知可能在等待的其他线程。
 mysql_mutex_lock(&LOCK_open);
 share->m_open_in_progress = false;
 mysql_cond_broadcast(&COND_open);

......
 DBUG_RETURN(share);
}
`

以上，`get_TABLE_SHARE`过程中会通过`thd->dd_client()->acquire()`获取`Table_Impl`信息并保存到`abstract_table`中。同时TABLE_SHARE会被缓存在`table_def_cache`中，以避免重复构建TABLE_SHARE的开销。TABLE_SHARE为所有TABLE对象所共有，直到TABLE被删除或因开表过多需要关掉时才会释放。

`bool Dictionary_client::acquire(const K &key, const T **object,
 bool *local_committed,
 bool *local_uncommitted) {
 ......
 
 // 首先尝试是否有线程本地的uncommited cache，一般存在于create过程中
 acquire_uncommitted(key, &uncommitted_object, &dropped);
 if (uncommitted_object || dropped) {
 ......
 return false;
 }
 
 // 本地的uncommited cache中没有时，访问本地commited cache，一般存在于当前THD正在处理表操作时
 m_registry_committed.get(key, &element);
 if (element) {
 ......
 return false;
 }

 // The element is not present locally.
 *local_committed = false;

 // 查找共享缓存或存储系统中是否有表信息
 if (Shared_dictionary_cache::instance()->get(m_thd, key, &element)) {
 DBUG_ASSERT(m_thd->is_system_thread() || m_thd->killed ||
 m_thd->is_error());
 return true;
 }

 // Add the element to the local registry and assign the output object.
 if (element) {
 ......
 }
 return false;
}

// 查找共享缓存对应dd信息类型的字典，找不到时会查找存储引擎，并保存在相应字典中。
bool Shared_dictionary_cache::get(THD *thd, const K &key,
 Cache_element<T> **element) {
 bool error = false;
 DBUG_ASSERT(element);
 if (m_map<T>()->get(key, element)) {
 // Handle cache miss.
 const T *new_object = NULL;
 error = get_uncached(thd, key, ISO_READ_COMMITTED, &new_object);

 // Add the new object, and assign the output element, even in the case of
 // a miss error (needed to remove the missed key).
 m_map<T>()->put(&key, new_object, element);
 }
 return error;
}

// 访问存储引擎查找dd信息
bool Shared_dictionary_cache::get_uncached(THD *thd, const K &key,
 enum_tx_isolation isolation,
 const T **object) const {
 DBUG_ASSERT(object);
 bool error = Storage_adapter::get(thd, key, isolation, false, object);
 DBUG_ASSERT(!error || thd->is_system_thread() || thd->killed ||
 thd->is_error());

 return error;
}
`
​

一般而言，在第一次访问表的`Table_Impl`信息时（包括create table后、宕机重启后等情况下），`Table_Impl`信息不在dd cache中，于是acquire函数会通过访问`Storage_adapter`获取并保存到`Shared_multi_map`的 `committed registry` 中。这里还会有`set_missed`机制防止多线程同时访问存储引擎 前面提到的月报文章中已有介绍。之后自己或其他工作线程再需要访问dd信息时（如新的SQL需要开表或show create时），就可以从共享缓存`Shared_multi_map`中得到TableImpl信息。
​

获取`TableImpl`并构建和缓存TABLE_SHARE后，mysql需要获取`TABLE`对象以完成开表操作。这一获取操作有两种方式：
1、该TABLE对象已经在`thd->open_tables`中,根据匹配的TABLE对象持有的锁等级选择`best_table`

`for (table = thd->open_tables; table; table = table->next) {
 // find best_table
}

table_list->table = best_table
`

2、`thd->open_tables`中不存在时，相应TABLE对象已经在thd对应的table_cache中**一定不存在**于`used_tables`中，但当`free_tables`中存在时：从table_cache中读取，并加入`el->used_tables`。

`table = tc->get_table(thd, key, key_length, &share);

TABLE *tableTABLE\_SHAREcache::get_table(THD *thd, const char *key, size_t key_length,
 TABLE_SHARE **share) {
 ......
 const auto el_it = m_cache.find(key_str);
 if (el_it == m_cache.end()) return NULL;
 tableTABLE\_SHAREcache_element *el = el_it->second.get();

 if ((table = el->free_tables.front())) {
 DBUG_ASSERT(!table->in_use);

 el->free_tables.remove(table);

 el->used_tables.push_front(table);

 table->in_use = thd;
 }

 return table;
}
`

3、get_table发现该TABLE对象不在table_cache中，通过`open_table_from_share`构建对象，然后`tc->add_used_table`放入used table cache
​

获取到TABLE对象后，该对象也已经缓存在table_cache的`used_tables`中，接下来通过将`thd->set_open_tables(table)`对象记录在`thd->open_tables`中，保证前面提到的，`thd->open_tables`中不存在时，相应TABLE对象已经在thd对应的table_cache中一定不存在于`used_tables`中 这一约束成立。
​

##清理：close_table
关表操作一般发生于完成一些操作后需要释放资源时，`mysql_execute_command`结束后`close_thread_tables(thd);`关闭所有`thd->open_tables`，然后清理相应的table cache：一般而言都是把table cache中in use的TABLE对象置为free，当缓存对象数过多时，也会进行清理（`free_unused_tables_if_necessary`）。
​

##删除：drop table
drop table时会清空前面提到的一切数据，包括清空表相关的table cache、释放TABLE_SHARE并从`table_def_cache`中移除、删除DD信息并清理dd信息的多级缓存等。
​

首先是清空dd cache。相关函数为`table_cache_manager::free_table`。该函数会遍历所有table_cache并删除相匹配的item，同时降低TABLE_SHARE的引用计数。其调用栈如下：

`#0 tableTABLE\_SHAREcache_manager::free_table
#1 0x0000000003338b39 in remove_table
#2 0x0000000003338dc0 in tdc_remove_table
#3 0x00000000034b3d32 in drop_base_table
#4 0x00000000034b523d in mysql_rm_table_no_locks
#5 0x00000000034b105f in mysql_rm_table
`

TABLE_SHARE的引用计数降为0（即所有TABLE对象释放完毕）后就可以释放了。
​
`tdc_remove_table`函数清理完table_cache和table_def_cache后，`dd::drop_table`函数负责调用`dd::cache::Dictionary_client::drop<dd::Table>`清理dd cache和删除DD信息，主要分为两步操作：
1、`Storage_adapter::drop`从存储引擎删除持久化的DD信息
2、`invalidate(object);`将DD信息标记为已删除，这样本线程一旦再需要访问或验证成功删除时，就不会产生读穿透至存储引擎的情况。

以上，给予MySQL 8.0.13我们分析的TABLE信息和相关的DD信息的正常流程下的生命周期，至于其他流程，包括超过容量之后的淘汰、TABLE的reopen机制等，我们以后再行研究。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)