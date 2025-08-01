# MySQL · 特性分析 · MySQL 8.0 资源组 (Resource Groups)

**Date:** 2018/05
**Source:** http://mysql.taobao.org/monthly/2018/05/03/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 05
 ](/monthly/2018/05)

 * 当期文章

 MySQL · Community · Congratulations on MySQL 8.0 GA
* MySQL · 社区动态 · Online DDL 工具 gh-ost 支持阿里云 RDS
* MySQL · 特性分析 · MySQL 8.0 资源组 (Resource Groups)
* MySQL · 引擎分析 · InnoDB行锁分析
* PgSQL · 特性分析 · 神奇的pg_rewind
* MSSQL · 最佳实践 · 阿里云RDS SQL自动化迁移上云的一种解决方案
* MongoDB · 引擎特性 · journal 与 oplog，究竟谁先写入？
* MySQL · RocksDB · MANIFEST文件介绍
* MySQL · 源码分析 · change master to
* PgSQL · 应用案例 · 阿里云 RDS PostgreSQL 高并发特性 vs 社区版本

 ## MySQL · 特性分析 · MySQL 8.0 资源组 (Resource Groups) 
 Author: 贤勇 

 MySQL 8.0已经正式发布。这个版本包含很多有意思的特性，例如，更快、性能更好的Schema和Information Schema、原子DDL、UNDO空间回收等，在很多的网站，博客等上面都有大量的推广介绍。本文将要介绍的一个很有用的特性，资源组，反而没有得到充分的宣传。如果没有特别的说明，本文的例子针对的是Linux系统。

## 什么是资源组

大家知道，MySQL实例里面包含系统的后台线程，例如Master Thread、IO Thread， Purge Thread等，也有处理用户请求的前台线程，在MySQL 8.0之前，所有的线程都是同等优先级的，官方的版本是不可以修改线程的优先级的，也没法将线程绑定到特定的cpu核上。在某些业务场景下，我们希望可以来做干涉，例如，白天业务高峰的时候，让用户线程的优先级更高，而到晚上做批量处理的时候，让负责批量处理的线程优先级更高，等等，另外，也希望可以给特定的线程来绑定cpu核，来保证服务质量等…

使用MySQL 8.0资源组这个特性就可以很方便的满足这些需求，要做的就是通过CREATE RESORUCE GROUP命令创建一个资源组，在这个DDL语句里面指定这个的类型（USER 或者 SYSTEM类型），优先级 （-20..0 for SYSTEM 类型线程, 0..19 for USER类型线程，数字越小优先级越高），以及可以使用的VCPU（逻辑CPU）的编号。之后，再使用SET RESOURCE GROUP语句将thread指派到某个组里面即可。

注： 系统可用的VCPU编号可以通过`cat cat /proc/cpuinfo`命令查看，processor字段就是对应的VCPU的编号；在没有开启超线程的情况下，逻辑CPU个数 = 物理CPU个数 * CPU内核数。

## 资源组使用方法与示例

我们来看几个例子，

* 创建一个叫sql_thread的资源组，将编号0和3的逻辑CPU指定到这个组，并将这个组的优先级设为最高 （THREAD_PRIORITY值越小，优先级越高，-20是最小的值）。

`CREATE RESOURCE GROUP sql_thread
 TYPE = USER
 VCPU = 1,3
 THREAD_PRIORITY = -20
`
* 将某个活动的MySQL Thread指派到sql_thread资源组中，Thread的id值需要查询Performance_Schema.threads的Thread_Id字段，目前Show Processlist命令的输出还不直接可以看到。
 ```
SET RESOURCE GROUP sql_thread FOR 10

```
* 修改资源组sql_thread绑定的逻辑CPU为编号5，6，并降低优先级到-5。

```
ALTER RESOURCE GROUP sql_thread VCPU = 5, 6 THREAD_PRIORITY = -5;

```

SYSTEM类型的Threads默认使用的是SYS_default资源组，USER类型的Threads默认使用的是USER_default资源组。看看这两种资源组的属性，没有绑定逻辑CPU，默认优先级都是0，见下面的输出 （例子环境下的逻辑CPU有24个，全部都可以使用）。

`SELECT * FROM INFORMATION_SCHEMA.RESOURCE_GROUPS\G
*************************** 1. row ***************************
 RESOURCE_GROUP_NAME: USR_default
 RESOURCE_GROUP_TYPE: USER
RESOURCE_GROUP_ENABLED: 1
 VCPU_IDS: 0-23
 THREAD_PRIORITY: 0
*************************** 2. row ***************************
 RESOURCE_GROUP_NAME: SYS_default
 RESOURCE_GROUP_TYPE: SYSTEM
RESOURCE_GROUP_ENABLED: 1
 VCPU_IDS: 0-23
 THREAD_PRIORITY: 0
`

注： 目前，只支持创建USER或者SYSTEM类型的资源组，如果通过`SET RESOURCE GROUP FOR thread_id`来将某个特定的threa_id指定到某个资源组，需要保证这个线程的TYPE和资源组一致。查询线程类型的方法也是要查询PFS threads表，看TYPE字段。如果类型不匹配，MySQL会报错3661。

另外两种使用指定资源组的方法，

* 通过SET RESOURCE GROUP 命令将当前session的thread指定到某资源组

` SET RESOURCE GROUP sql_thread;
 CREATE TABLE tb1 (col1 INT):
 INSERT INTO tbl1 VALUES(1); <=== CREATE 和 INSERT语句的执行都会使用sql_thread资源组指定的逻辑CPU和执行优先级
`
* 通过RESOURCE_GROUP optimizer hint `INSERT /*+ RESOURCE_GROUP(sql_thread) */ INTO tbl1 VALUES(1);` <== 当前语句会使用sql_thread资源组

## 资源组特性引入的系统表更改，权限要求

* PFS的threads表增加了RESOURCE_GROUP字段
* 新添了数据字典表 mysql.resource_groups，通过视图 INFORMATION_SCHEMA.RESOURCE_GROUPS 可以访问这个表
* 执行CREATE, ALTER, DROP, SET资源组命令，需要有RESOURCE_GROUP_ADMIN权限
* 在Linux系统上, mysqld可执行命令需要有CAP_SYS_NICE能力。sudo权限用户可以通过setcap来授权，例如，

```
shell> sudo setcap cap_sys_nice+ep ./bin/mysqld //授权 
shell> getcap ./bin/mysqld 
 ./bin/mysqld = cap_sys_nice+ep 

```

## 资源组代码分析

资源组特性主要对应于MySQL [WL#9467](https://dev.mysql.com/worklog/task/?id=9467), 代码修改的主体见commit log c47051b4be2110ed6225860448fe8657cf500a4a，涉及到数据字典的修改，新SQL语法支持等等多个方面。resource_group_sql_cmd.h包含了代表RESOURCE GROUP各种命令的类，比如，Sql_cmd_create_resource_group这个类代表CREATE RESOURCE GROUP命令，Sql_cmd_drop_resource_group代表DROP RESOURCE GROUP命令， Sql_cmd_set_resource_group类代表SET RESOURCE GROUP 命令，它们都是Sql_cmd的子类，execute()方法的实现都在Resource_group_mgr的方法里Resource_group_mgr这个类提供了资源组操作的各种接口，例如

`bool add_resource_group(std::unique_ptr<Resource_group> resource_group_ptr); //增添一个资源组

void remove_resource_group(const std::string &name);//按名字删除一个资源组

void set_res_grp_in_pfs(const char *name, int length, ulonglong thread_id); //为对应id的Thread的指定资源组

bool acquire_shared_mdl_for_resource_group(THD *thd, const char *res_grp_name, //给指定的资源组加共享MDL锁
 enum_mdl_duration lock_duration,
 MDL_ticket **ticket,
 bool acquire_lock);

static bool acquire_exclusive_mdl_for_resource_group(THD *thd, //给指定资源组加排他DML锁
 const char *res_grp_name);

`
Thread_resource_control这个类提供了对线程实施控制的接口，包括优先级和逻辑CPU绑定， 例如

`void set_priority(int priority) //设置优先级

set_vcpu_vector(const std::vector<Range> &vcpu_vector)//设置绑定逻辑CPU列表

bool apply_control(my_thread_os_id_t thread_os_id); //对指定的线程实施设置的控制
`
sql/resourcegroups/platform/thread_attrs_api.h包含了具体实施线程控制到具体操作系统的接口，包括

`bool set_thread_priority(int priority); //设置优先级

uint32_t num_vcpus(); //获取逻辑CPU个数

bool bind_to_cpus(const std::vector<cpu_id_t> &cpu_ids); //绑定线程到指定的逻辑CPU

`
MySQL 8.0提供对Apple，FreeBSD， Linux和一般平台的实现，分别对应 thread_attrs_api_apple.cc， thread_attrs_freebsd.cc， thread_attrs_api_linux.cc和 thread_attrs_api_common.cc，

基于在Linux上gdb调试，我们来看一下CREATE RESOURCE GROUP以及SET RESOURCE GROUP的代码流程。

* CREATE RESOURCE GROUP命令

resourcegroups::Sql_cmd_create_resource_group::execute()方法是主体入口

` if (!sctx->has_global_grant(STRING_WITH_LEN("RESOURCE_GROUP_ADMIN")).first) //必须具有 RESOURCE_GROUP_ADMIN权限
 if (validate_vcpu_range_vector(vcpu_range_vector.get(), m_cpu_list, num_vcpus)//验证逻辑cpu list的有效性
 if (acquire_shared_backup_lock(thd, thd->variables.lock_wait_timeout)) //获取共享backup锁
 if (acquire_exclusive_mdl_for_resource_group(thd, m_name.str))//获取资源组MDL排它锁
 if (res_grp_mgr->get_resource_group(m_name.str) != nullptr) // 检查内存中是否已经存在同名的Resource Group
 if (dd::resource_group_exists(thd->dd_client(), dd::String_type(m_name.str) //disk上的数据字典表也要查 
 auto resource_group_ptr = res_grp_mgr->create_and_add_in_resource_group_hash // 如果不存在就创建并加入资源组hash表里面
if (dd::create_resource_group(thd, *resource_group_ptr)) //在字典表登记这个资源组
`

我们再看看create_and_add_in_resource_group_hash()这个方法

` auto thr_res_ctrl = resource_group_ptr->controller();
thr_res_ctrl->set_priority(priority); //设置资源组优先级
thr_res_ctrl->set_vcpu_vector(*vcpu_range_vector); //设置资源组逻辑CPU
// add to in-memory hash
Resource_group_mgr::instance()->add_resource_group //将新建的资源组加入内存Hash表
std::unique_ptr<Resource_group>(resource_group_ptr));
`

* SET RESOURCE GROUP命令Thread_resource_control::apply_control()方法是主体入口

```
ret = resourcegroups::platform::bind_to_cpus(cpu_ids) || //将当前thread绑定到资源组对应的逻辑CPU上，并应用指定的优先级
resourcegroups::platform::set_thread_priority(m_priority);

```

Linux系统上bind_to_cpus()方法调用栈

`
 bind_to_cpu(cpu_id_t cpu_id) -> bind_to_cpu(cpu_id_t cpu_id, my_thread_os_id_t thread_id)
 ->::sched_setaffinity(thread_id, sizeof(cpu_set), &cpu_set)

`

Linux系统上set_thread_priority()方法调用栈

```
 set_thread_priority(int priority, my_thread_os_id_t thread_id)-> setpriority(PRIO_PROCESS, thread_id, priority)

```

## 使用资源组的限制
* 目前仅支持对CPU的设定，不包含IO，内存等
* 资源组类型只支持USER和SYSTEM两种类型，而常见的线程类型是FOREGROUN和BACKGROUND，直接通过SET RESOURCE GROUP来指定线程的资源组往往会报3661错
* 对操作系统平台有强依赖

## 总结

* 资源组是MySQL 8.0一个方便DBA调控线程优先级和绑定CPU核的特性。
* 这个特性依赖于底层操作系统的支持，在Linux环境下，mysqld可执行命令需要有CAP_SYS_NICE能力，可以选择的逻辑CPU的列表也是依赖于具体的Linux环境。
* 具备RESOURCE_GROUP_ADMIN权限的用户才可以创建，修改和删除资源组。
* 支持的用户组类型目前只有USER和SYSTEM两种，在通过SET RESOURCE GROUP来修改线程的资源组的时候，线程的类型和资源池类型必须匹配。
* 支持session级别和语句级别指定资源组。
* 对资源组的操作会涉及到对backup锁，资源组DML锁的操作。

## 参考资料
1. https://dev.mysql.com/doc/refman/8.0/en/resource-groups.html
2. https://linux.die.net/man/1/taskset

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)