# MySQL · 捉虫动态 · left-join多表导致crash

**Date:** 2016/05
**Source:** http://mysql.taobao.org/monthly/2016/05/10/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 05
 ](/monthly/2016/05)

 * 当期文章

 MySQL · 引擎特性 · 基于InnoDB的物理复制实现
* MySQL · 特性分析 · MySQL 5.7新特性系列一
* PostgreSQL · 特性分析 · 逻辑结构和权限体系
* MySQL · 特性分析 · innodb buffer pool相关特性
* PG&GP · 特性分析 · 外部数据导入接口实现分析
* SQLServer · 最佳实践 · 透明数据加密在SQLServer的应用
* MySQL · TokuDB · 日志子系统和崩溃恢复过程
* MongoDB · 特性分析 · Sharded cluster架构原理
* PostgreSQL · 特性分析 · 统计信息计算方法
* MySQL · 捉虫动态 · left-join多表导致crash

 ## MySQL · 捉虫动态 · left-join多表导致crash 
 Author: santo.lj 

 有一天小编胡乱写SQL, left join了30张表, 结果导致了Mysql server gone away…
我们来看看crash堆栈

`<signal handler called>
base_list_iterator::next
update_ref_and_keys
make_join_statistics
JOIN::optimize
mysql_execute_select
`
可以看出, 在产生执行计划过程中crash了。

## 追查
堆栈表明, `update_ref_and_keys`函数中`join_tab->join->join_list`为无效地址。 排查看到函数入口处这个变量还是ok的, 那么在gdb里watch一下。

`Hardware watchpoint 4: join_tab->join->join_list

Old value = (List<TABLE_LIST> *) 0x3431f60
New value = (List<TABLE_LIST> *) 0xc800000000000000
`
这么整齐的地址一看就有问题。函数栈:

`Key_field::Key_field
add_key_field
add_key_equal_fields
add_key_fields
update_ref_and_keys
`
而`add_key_fields`修改`join_tab->join->join_list`实际是不合理的, 因此这里说明一下路径上几个关键的函数。

## 原因分析
还要从子查询优化说起，当遇到semi-join子查询情况下, `JOIN::optimize()`会调用`JOIN::flatten_subqueries`改写SQL, 如下形式:

`SELECT ...
FROM ot1, ...
WHERE oe IN (SELECT ie FROM it1, ..., itN WHERE subq_where)
 AND outer_where
`

会被修改为:

`SELECT ...
FROM ot SEMI JOIN (it1, ... , itN),
WHERE outer_where AND subq_where AND oe=ie
`

函数`JOIN::flatten_subqueries`, 做了以下几件事:

* 创建semi join(it1, …, itN)的节点并添加到外层查询语句的FROM语法树下
* 将`subq_where AND oe=ie`加入到外层查询语句的WHERE树下
* 再移除原先的子查询语句

`JOIN::flatten_subqueries`中, 对于每一个子查询, 调用函数`JOIN::convert_subquery_to_semijoin`, 那么子查询上维护的query信息也要同步加到外部查询上。所以可见, 子查询中的信息, 会转交给外部查询。

之后, `JOIN::optimize()`调用`update_ref_and_keys`, 这个函数用来处理出最终查询要使用的索引。crash的问题也出现在这个函数中, 因此还要看`update_ref_and_keys`内部做了什么。

在函数`update_ref_and_keys`中, 一个重要的数组, key_fields, 用来存放所有可能用到的索引字段。先通过`key_fields=(Key_field*) thd->alloc(sz)`分配空间, 再调用`add_key_fields`递归遍历WHERE树, 遇到等值表达式, 会填充到`key_fields`数组中。而之前已经看到, add_key_field在写key_fields时却修改了`join_tab->join->join_list`。

`// add_key_fields中修改了join_tab->join->join_list的代码
new (*key_fields)
 Key_field(field, *value, and_level, exists_optimize, eq_func,
 null_rejecting, NULL, get_semi_join_select_list_index(field));
 (*key_fields)++;
`

可见在new的时候拿到了`join_tab->join->join_list`, 是(*key_fields++)的时候, 加过头了。从而可推断, key_fields没有分配到应该有的内存空间。那么出问题的就是sz用来分配空间的数字了。

`// sz的计算方法
sz= max(sizeof(Key_field), sizeof(SARGABLE_PARAM)) *
 (((select_lex->cond_count + 1) * 2 +
select_lex->between_count) * m + 1);
`

这里涉及到两个变量`select_lex->cond_count`和`select_lex->between_count`, 而cond_count就是number of conditions; 构造的语句中的等值表达式足有31条, 而这里在分配时是2, 活该内存越界。
而这个变量在子查询优化过程中, 子查询应该将其移交给外部查询语句。

## 修复
函数`JOIN::convert_subquery_to_semijoin`中, 改写完SQL后, 忘记把子查询的cond_count和between_cond信息更新到外部查询了, 这时只要手动添加即可。

[官方修复(5.6.25)参见](https://github.com/mysql/mysql-server/commit/71e74f2a0118f460abc4f7a3da215c61785d35f0)
[相关worklog参见](http://dev.mysql.com/worklog/task/?id=5275)

## 复现
可以通过以下方式复现

`create table t1 (
 `id` int(20),
 `col3` varchar(60) default null,
 primary key (id)
);

create view `v_test` as
select t1.col1 as col1,
 t2.col2 as col2,

 ...

 t30.col30 as col30
 from (((((((((((((((((((((((((((((t1
 left join t2 on (t1.id = t2.id))
 left join t3 on (t1.id = t3.id))

 ...

 left join t30 on (t1.id = t30.id));
`

然后执行

`create table tt (id int(20), b varchar(200));
select * from tt where b in (select col1_1 fromom v_test);
`
MySQL5.6在5.6.25之前的小版本都可以复现, 请尽情调戏 .^.

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)