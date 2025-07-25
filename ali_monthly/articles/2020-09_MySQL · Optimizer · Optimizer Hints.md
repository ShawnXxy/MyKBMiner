# MySQL · Optimizer · Optimizer Hints

**Date:** 2020/09
**Source:** http://mysql.taobao.org/monthly/2020/09/07/
**Images:** 2 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2020 / 09
 ](/monthly/2020/09)

 * 当期文章

 MySQL · 性能优化 · PageCache优化管理
* MySQL · 分布式系统 · 一致性协议under the hood
* X-Engine · 性能优化 · Parallel WAL Recovery for X-Engine
* MySQL · 源码阅读 · InnoDB伙伴内存分配系统实现分析
* PgSQL · 新特性探索 · 浅谈postgresql分区表实现并发创建索引
* MySQL · 引擎特性 · InnoDB隐式锁功能解析
* MySQL · Optimizer · Optimizer Hints
* Database · 新特性 · 映射队列

 ## MySQL · Optimizer · Optimizer Hints 
 Author: 开旺 

 ## 背景

优化器是关系数据库的重要模块 [1] [2]，它决定 SQL 执行计划的好坏。但是，优化器的影响因素很多，由于数据变化和估计准确性等因素，它不能总是产出最优的执行计划 [3] 。选择了不同的执行计划，执行效果差异可能非常大，甚至达到数量级差异，可能对生产系统产生严重影响。虽然学术和业界长期致力于优化器的改进，但对于业务系统而言，在优化器犯错的时候，需要有一些直接有效的干预办法。

Optimizer Hints (下文简称 Hints ) 是一套干预优化器的实用机制，不同数据库厂商都有各自的实现方式。Oracle 可能是将 [Hints 机制](https://docs.oracle.com/en/database/oracle/oracle-database/18/tgsql/influencing-the-optimizer.html#GUID-8758EF88-1CC6-41BD-8581-246702414D1D)发挥到极致的数据库大厂。而即使像 PostgreSQL 这样拒绝 hints 的学院派数据库，“民间”也自发搞了个 [pg_hint_plan 插件](http://mysql.taobao.org/monthly/2016/01/09/) ，让大家能够尽快地解决执行计划走错的问题。

Hints 的干预方式是向优化器提供现成的优化决策，从而缩小执行计划的选择范围。通常在人为干预优化器时，只需要在关键决策点提供具体决策，就可以规避错误的执行计划；当然也可以提供所有决策，这样可以产生确定的执行计划。

MySQL [新一代 Hints](https://dev.mysql.com/worklog/task/?id=3996) ，是在 [5.7.7](https://dev.mysql.com/doc/relnotes/mysql/5.7/en/news-5-7-7.html#mysqld-5-7-7-optimizer) (2015-04-08 RC) 作为比 [optimizer_switch](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_optimizer_switch) 更精细的优化器干预机制而引入的，直到 [8.0.20](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-20.html#mysqld-8-0-20-optimizer) (2020-04-27 GA) 引入 [Index-Level Optimizer Hints](https://dev.mysql.com/doc/refman/8.0/en/optimizer-hints.html#optimizer-hints-index-level) 取代古董级的 [Index Hints](https://dev.mysql.com/doc/refman/5.6/en/index-hints.html) ，终于完成了“统一大业”，成为 MySQL 社区唯一推荐的优化器干预机制。

## 使用

在使用 hints 的时候，有一个非常重要的概念，就是标定被干预对象，也就是说优化决策是如何匹配的。然后才是施以具体动作，影响优化器的行为。从这个视角来看， hints 也是一套支持“匹配-动作”的规则系统。

被干预对象分为四个层次：语句、查询块、表和索引（如下图灰色节点）。一条简单的语句可能只有一个查询块，而 UNION 和子查询都会引入新的查询块。虽然语义上查询块是可以套嵌的，但由于 MySQL 里使用统一编号，所以，在 Hint 视角其实是一视同仁的。

像下面这个语句就包含了两个查询块，即 select#1 和 select#2 ，分别对应两个 UNION 分支：

`/* select#1 */ SELECT c1 FROM t1 WHERE c2 = 1 UNION ALL /* select#2 */ SELECT c1 FROM t1 WHERE c3 >= 1;
`

如果把第一个 UNION 分支单独拿出来，加一个索引选择的 hint （注：大写表达概念，小写表示使用），那就是这样：

`SELECT /*+ INDEX(t1 idx_1) */ c1 FROM t1 WHERE c2 = 1;
`

如果要全表扫描，那就是这样

`SELECT /*+ NO_INDEX(t1) */ c1 FROM t1 WHERE c2 = 1;
`

从这两个例子出发，可以简单归纳一下 Hint 的表达方式：

1. `/*+ */` 是新一代 Hints 的专用注释格式
2. `INDEX` 是决策动作，即干预索引选择，而 `NO_INDEX` 表示反向决策，即禁止选择指定索引
3. t1 表示被干预对象是当前查询块的 t1 表（注：这是简写，完整写法是 t1@qb，其中 qb 是 QB_NAME 起的别名)
4. idx_1 是动作参数，即要选 idx_1 索引

## Hint 的种类

MySQL Hints 目前已经支持干预的优化决策有：变形策略、表连接顺序、表连接算法、表访问路径和一些特殊决策。除此之外， 它还可以用于其他场景，例如设置系统变量等。下表是 8.0.22 支持的 Hint 列表（详见[官方文档](https://dev.mysql.com/doc/refman/8.0/en/optimizer-hints.html)）。

 干预的类型
 Hints

 变形策略
 SEMIJOIN, SUBQUERY, MERGE, ICP, DERIVED_CONDITION_PUSHDOWN

 表连接顺序
 JOIN_ORDER, JOIN_PREFIX, JOIN_SUFFIX, JOIN_FIXED_ORDER

 表连接算法
 BNL, HASH_JOIN

 表访问路径
 BKA, MRR, INDEX, INDEX_MERGE, SKIP_SCAN, JOIN_INDEX, GROUP_INDEX, ORDER_INDEX

 特殊控制
 NO_RANGE_OPTIMIZATION

 其他
 QB_NAME, SET_VAR, RESOURCE_GROUP, MAX_EXEC_TIME

因为实际应用场景中，绝大部分执行计划错误，都是表序和索引选择导致，所以，最常用的是 JOIN_ORDER 和 INDEX ，它们分别指定连接顺序和候选索引。此外， 8.0 增加了视图合并的功能（默认开启），有时候需要用 NO_MERGE 来关掉该特性，发挥物化表的一些优势，这主要发生 5.7 迁移场景中。

当然，MySQL 优化器的决策点不仅限于这个列表，而且社区也在不断加强优化器。可以预知的是，随着业务场景的强烈诉求和优化器特性的不断丰富， hint 种类会越来越多，这样才能精细地干预优化器。比如说，DERIVED_CONDITION_PUSHDOWN 就是 8.0.22 新增的。由此我们也可以看到，MySQL 优化器的发展策略基本上还是实用主义至上，侧重于增强变形能力而不是变形决策，并没有在优化器框架上进行较大的改进。不过，可能是高级特性开发受制于现有框架，最近社区也开始了新优化框架的尝试，让我们拭目以待吧。

## 内核实现

前面讲到，Hints 其实是一套干预机制，它匹配被干预对象，施以动作来影响优化器行为。

内核实现分为三个部分：统一的 Hint 语法支持、Hint 的内部组织形式和对优化器的影响方式。

语法支持和组织形式，也称为新一代 Hint 基础架构，新开发的 Hint 只需要按照约定在其中增加声明和校验机制。但影响方式则是因 Hint 相关的优化器行为而异的，简单的只需要查一下 hint 参数来决定是否启用一段代码分支（例如 MERGE ），而复杂的就需要修改优化器数据结构，像 JOIN_ORDER 就要根据参数来建立表依赖关系，并修改相应的运行期数据结构内容。

### 统一的 Hint 语法支持

语法支持分为两部分，即在客户端的专用注释类型和在服务端语法解析，都在 WL#8016 设计范围里。

客户端其实没有做语法解析，只是在 client/mysql.cc 的 add_line() 函数里，将新一代 hints 的注释转发到服务端。顺便说一句，虽然在 8.0.20 里已经支持了以系统化命名机制来引用查询块，但在客户端代码里却未做相应的处理，所以，这个还是未公开行为。

在服务端解析代码设计上，为了尽量避免修改 main parser (sql_yacc.yy) ， WL#8016 选择了共用 token 空间，但独立的 Hint parser 和 lexer 的方式。只在遇到特定的 token 才切换到 Hint parser 消费掉所有新一代 hints 注释 (consume_optimizer_hints) ，产生的 hints 列表 (PT_hint_list) 则返回给 main parser 。只有 5 种子句支持 hint ，即 SELECT INSERT DELETE UPDATE REPLACE 。

相关源代码文件：

`sql/lex.h // symbol
sql/gen_lex_token.{h,cc} // token
sql/sql_lex_hints.{h,cc} // hint lexer
sql/sql_hints.yy // hint parser
sql/parse_tree_hints.{h,cc} // PT_hint_list, PT_hint, PT_{qb,table,key}_level_hint, PT_hint_sys_var, ...
sql/sql_lex.cc // consume_optimizer_hints()
`

### Hint 的内部组织形式

Hints 会在 parse 后的 contextualization 阶段注册到一个称为 hints tree 的四层树状结构中。每个 PT_hint 子类都需要提供相应的 contextualize() 实现，它主要作用是检查 hint 的合法性和相互是否有冲突，然后转成 hints tree 表达形式。这些在 WL#8017 的设计范围里。

Hints tree 的节点类型是 Opt_hints ，四个层次分别是语句、查询块、表和索引，相应的子类是 Opt_hints_global, Opt_hints_qb, Opt_hints_table, Opt_hints_key 。也就是说，每个被干预对象，都是 hints tree 的一个节点。然后在优化过程中的每个决策点，优化器都会到这个 hints tree (lex->opt_hints_global) 查找匹配的 hint ，并采取相应的动作，而查找结果还会缓存在被干预对象中，例如 SELECT_LEX::opt_hints_qb 和 TABLE_LIST::opt_hints_table 。如下图所示：

![](.img/9a2548e90784_hints-tree.png)

下面是 Opt_hints 结构。每个节点都有一个 hints_map ，用于表示每个类型的 hint 是否指定以及开关状态。可以看到，MySQL Hints 目前最多支持 64 种。

`class Opt_hints_map {
 Bitmap<64> hints;
 Bitmap<64> hints_specified;
};
class Opt_hints {
 const LEX_CSTRING *name; // 用于匹配的名字
 Opt_hints *parent;
 Mem_root_array<Opt_hints *> child_array;
 Opt_hints_map hints_map; // 每个 Hint 是否指定，及其开关状态
 // ...
};
`

而每个层级都可以有相应的额外信息，例如，语句级 hint 记录全局设定，查询块级 hint 会有相应的变形和表序决策，表级 hint 则有索引选择的决策。索引级 hint 没有额外信息，因为索引上的 hint 都是开关类型的。

`class Opt_hints_global : public Opt_hints {
 PT_hint_max_execution_time *max_exec_time;
 Sys_var_hint *sys_var_hint;
};
class Opt_hints_qb : public Opt_hints {
 uint select_number;
 PT_qb_level_hint *subquery_hint, *semijoin_hint;
 Mem_root_array<PT_qb_level_hint *> join_order_hints;
 //...
};
class Opt_hints_table : public Opt_hints {
 Glob_index_key_hint index;
 Compound_key_hint index_merge;
 Compound_key_hint skip_scan;
 // ...
};
`

### 对优化器行为的影响方式

不同的 Hint 对优化器的干预方式是不同的（详见附录）。大体上，可以分为开关型、枚举型和复杂 Hint 三类。

#### 开关型

开关型影响方式就是直接启用特定的代码路径。大部分 hint 都是开关型的。下面是开关型查找函数。查找时会考虑两级继承逻辑（称为 Applicable Scopes ，详见[社区文档](https://dev.mysql.com/doc/refman/8.0/en/optimizer-hints.html#optimizer-hints-overview) ），不过，从上级对象继承干预方式的情况是几乎是没有的。Index merge 因为涉及多个索引，在处理上会特别一些。

`hint_table_state() // 表级 hint 状态
hint_key_state() // 索引级 hint 状态
compound_hint_key_enabled() // 主要用于检测 index merge 涉及的索引是否被禁掉
idx_merge_hint_state() // 用于表访问路径是否强制为 index merge
`

举例来说， 视图合并的决策点是在 SELECT_LEX::merge_derived() 函数里，在这里根据该引用位置（占位表）所匹配的 MERGE hint ，来决定启用相关代码路径：

`SELECT_LEX::resolve_placeholder_tables
 SELECT_LEX::merge_derived
 hint_table_state // 是否启用视图合并
`

#### 枚举型

枚举型相对于开关型，主要是支持多种状态。例如 semijoin 有两个决策点，在第一个决策点会调用 Opt_hints_qb::semijoin_enabled() 决定是否启用 semijion，在第二个决策点会查采用什么 semijoin 策略，调用 Opt_hints_qb::sj_enabled_strategies() 获得具体 semijoin 策略（ FIRSTMATCH, LOOSESCAN 或 DUPSWEEDOUT ），然后设置到 NESTED_JOIN 运行时结构中。

`SELECT_LEX::resolve_subquery
 SELECT_LEX::semijoin_enabled
 Opt_hints_qb::semijoin_enabled // 是否启用 semijoin

JOIN::optimize
 JOIN::make_join_plan
 SELECT_LEX::update_semijoin_strategies
 Opt_hints_qb::sj_enabled_strategies // 获取具体的 semijoin 策略，更新 NESTED_JOIN
`

#### 复杂型

其他 Hint 处理会相对复杂一点，不过，处理逻辑都包装在对应的函数中了。

干预连接顺序的 Hint 会在决策点调用 Opt_hints_qb::apply_join_order_hints() ，根据 hint 参数设置连接表的依赖关系，即修改 JOIN_TAB::dependent 表依赖位图，增加额外的依赖关系（表的相对顺序）。基于代价优化阶段产生的表序，会遵守设定的依赖关系。

`JOIN::optimize
 JOIN::make_join_plan
 Opt_hints_qb::apply_join_order_hints
 set_join_hint_deps // 修改表依赖位图 JOIN_TAB::dependent ，增加额外的相对顺序关系
 Optimize_table_order::choose_table_order() // 基于代价确定表序时，遵守已设定相对顺序
`

干预索引选择的 Hint 会在决策点调用 Opt_hints_qb::adjust_table_hints() 和 Opt_hints_table::update_index_hint_maps() 修改 TABLE 结构里的候选索引位图。在优化过程中，候选索引位图决定了哪些索引是可用的。

`SELECT_LEX::setup_tables
 Opt_hints_qb::adjust_table_hints // 查找索引干预决策
 Opt_hints_table::update_index_hint_maps // 修改候选索引位图 TABLE::keys_in_use_for_query 等
`

## 系统价值

### 现状

虽然已经一统江湖，但 MySQL Hints 仍然有不少需要完善的地方。例如视图的支持还是很欠缺的，对特定场景下的名字处理有歧义，也有一些决策点并没有覆盖到。不过，这些都可以在新一代 hints 的基础框架上逐步完善。

### 前景

由于优化器存在理论上的不确定性，简单直接的干预方式通常是有效的，这就可以构成稳定性系统的基础。比如说，将一个业务负载的执行计划全部记录下来，而在系统环境变化时（如系统升级或刷新统计信息）只考虑这些性能已知的执行计划，这样就可以减少升级带来的执行计划变差的风险。在通常是手工干预的机制上建立自动系统，在完整性和处理效率等方面会有很多挑战，不过，在大规模部署场景下也是值得尝试的。

## 附录

### 优化器参考文献

[1] Chaudhuri, Surajit. “An overview of query optimization in relational systems.” Proceedings of the seventeenth ACM SIGACT-SIGMOD-SIGART symposium on Principles of database systems. 1998.

[2] Selinger, P. Griffiths, et al. “Access path selection in a relational database management system.” Proceedings of the 1979 ACM SIGMOD international conference on Management of data. 1979.

[3] Is Query Optimization A “Solved” Problem? http://wp.sigmod.org/?p=1075#reference

### MySQL Hints 设计文档

[WL#3996: Add more hints](https://dev.mysql.com/worklog/task/?id=3996) (5.7)

[WL#8016: Parser for optimizer hints](https://dev.mysql.com/worklog/task/?id=8016)

[WL#8017: Infrastructure for Optimizer Hints](https://dev.mysql.com/worklog/task/?id=8017)

[WL#9158: Join Order Hints](https://dev.mysql.com/worklog/task/?id=9158) (8.0)

[WL#681: Hint to temporarily set session variable for current statement](https://dev.mysql.com/worklog/task/?id=681) (8.0)

[WL#9307: Enabling merging a derived table or view through a optimizer hint](https://dev.mysql.com/worklog/task/?id=9307) (8.0)

[WL#9467: Resource Groups](https://dev.mysql.com/worklog/task/?id=9467) (8.0)

[WL#9167: Index merge hints](https://dev.mysql.com/worklog/task/?id=9167) (8.0)

[WL#11322: SUPPORT LOOSE INDEX RANGE SCANS FOR LOW CARDINALITY](https://dev.mysql.com/worklog/task/?id=11322) (8.0)

[WL#8241: Hints for Join Buffering and Batched Key Access](https://dev.mysql.com/worklog/task/?id=8241) (5.7)

[WL#8243: Index Level Hints for MySQL 5.7](https://dev.mysql.com/worklog/task/?id=8243) (5.7)

[WL#8244: Hints for subquery strategies](https://dev.mysql.com/worklog/task/?id=8244) (5.7)

[WL#3527: Extend IGNORE INDEX so places where index is ignored can be specified](https://dev.mysql.com/worklog/task/?id=3527) (5.1)

[WL#2241 Implement hash join](https://dev.mysql.com/worklog/task/?id=2241) (8.0.18)

[WL#13538 Add index hints based on new hint infrastructure](https://dev.mysql.com/worklog/task/?id=13538) (8.0.20)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)