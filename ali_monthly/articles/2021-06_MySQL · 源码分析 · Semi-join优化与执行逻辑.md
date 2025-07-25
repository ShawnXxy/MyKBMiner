# MySQL · 源码分析 · Semi-join优化与执行逻辑

**Date:** 2021/06
**Source:** http://mysql.taobao.org/monthly/2021/06/02/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2021 / 06
 ](/monthly/2021/06)

 * 当期文章

 MySQL · 性能优化 · Undo Log IO优化
* MySQL · 源码分析 · Semi-join优化与执行逻辑
* MySQL · 源码分析 · Range (Min-Max Tree)结构分析
* MySQL · 源码分析 · Order By优化逻辑代码分析
* MySQL · 内核特性 · Btree 顺序插入优化及问题
* MySQL · 内核特性 · 分区表下的多种索引类型

 ## MySQL · 源码分析 · Semi-join优化与执行逻辑 
 Author: 遥凌 

 ## Semi-join 语义

在MySQL中，Semi-join是专门针对SPJ IN/Exists子查询进行优化的一种join语义，起到了对外层表的过滤作用，通过将相关/非相关subquery unnesting为semi join来充分利用join reordering的灵活性，以期获取最高的执行效率，正因为如此，其实现非常灵活多变，共有4种不同的实现策略。 本文主要针对semi-join(antijoin处理类似），描述其整体处理流程和具体细节代码，虽然复杂，但MySQL对于semi-join的处理并不像对于group by/distinct的实现逻辑如此杂乱，还是比较有章可循，因此允许我们顺着rewrite -> optimize的代码流程进行分析。

本篇是对MySQL代码实现的分析，涉及比较多的细节，如果只需要大概了解semi-join是什么以及MySQL处理它的4种策略大致是什么，可以参看之前的月报：

[https://www.bookstack.cn/read/aliyun-rds-core/ecc50d80e6916bc3.md](https://www.bookstack.cn/read/aliyun-rds-core/ecc50d80e6916bc3.md)

代码版本 MySQL 8.0.18

## Notation

我们称在外层子查询当中的相关表outer table，ot1, ot2 … 这其中如果是在inner subquery的where condition中依赖的外表相关表，记为non-trivially correlated outer table，ct1, ct2… 外层query block中，不相关的table，记为non-correlated outer table, nt1, nt2…. 内层query block中的表，记为inner table, it1, it2….

## 代码流程

### Rewrite Phase

* SELECT_LEX::resolve_subquery

 收集unnesting subquery items 对subquery item进行resolve，收集能够unnesting为semi-join的所有subquery block，这里有很多的严格限制条件，基本来说就是只允许SPJ的subquery进行unnesting，具体条件可详见函数中的代码及注释。

 可以做unnesting，会把这个Item对象，加入到外层select_lex::sj_candidates中后续使用

 无法做unnesting，则调用select_transformer，尝试IN->EXIST的转换
* SELECT_LEX::flatten_subqueries

 做unnesting的rewrite，由于MySQL在一个query block中能够join的tables数是有限的(MAX_TABLES)，因此做unnesting的这些sj_candidates，也要有一个优先级决定的先后顺序，保证重要的先unnesting掉，后续如果table满了，则停止转换，优先级规则集合如下：

 相关 > 非相关
* inner tables多 > inner tables少
* 可以提前完成的subq > 晚完成的subq

 另外，由于rewrite这个phase本身是递归完成的，因此flatten的过程是自内到外，依次把下层的subq展开到外层qb中对每个subquery

 replace_subcondition: 替换掉外层query block中，IN item所对应的predicate，替换为对应Item_func_true/Item_func_false
 SELECT_LEX::convert_subquery_to_semijoin

将真正可以融合的(有table)，建立sj-nest这个TABLE_LIST对象，基本思路就是想将inner table放到外层的join list中，内层的(oe1, … oeM) = (ie1, …, ieM) / inner-cond 都放在外层对应的ON/WHERE条件上，这里分为3种情况:

1. subq_pred->embedding_join_nest->nested_join 存在

`... [LEFT] JOIN ( ... ) ON (subquery AND condition) ...
`

这种形式，sj-nest这个TABLE_LIST会放到JOIN后面的 ( … ) 当中

1. subq_pred->embedding_join_nest->outer_join 不为true

`... INNER JOIN tblX ON (subquery AND condition) ...
`

没有nested join（只有一个tblX）且是INNER JOIN，sj-nest直接append到tblX所在这层的join list中

1. subq_pred->embedding_join_nest->outer_join 为true

`... LEFT JOIN tbl ON (subquery AND condition) ... 
 ( tbl SJ (subq_tables) )
 | |
 |<---- wrap_nest --->|
`

没有nested join，是left join on tbl的形式，为保证正确性，需要把tbl替换为上面这个(wrap-nest)的TABLE_LIST对象，如下

`... LEFT JOIN ( tbl SJ (subq_tables) ) ON (on_expr AND subq_cond) ...
`

sj-nest是后续优化semi-join的一个重要结构，会用subq SELECT_LEX中的内容对其进行填充，填充内容如下：

* 把subq_select中的leaf_table这个list，链接到外层的leaf table/next local链上，这样后续才能join reordering
* 将subq中的相关条件，也放入sj_nest->nested_join->sj_outer_exprs/sj_nest->nested_join->sj_inner_exprs中，统一设置到外层的condtition中
* 设置nested_join->sj_depends_on/sj_corr_tables，sj_depends_on是ot + ct，而sj_corr_tables只表示了ct，这个在选择有效的join order时，会使用到
* SELECT_LEX::simplify_joins

 做join结构的简化和展平，outer->inner等，是通用的处理逻辑这里就不展开了，其中和semi-join相关的是，它会去掉嵌套在sj-nest中的任何子sj-nest，把他们展开到一个sj-nest结构中，保证MySQL不用去处理嵌套sj-nest的情况

### Optimize Phase

* JOIN::make_join_plan

 pull_out_semijoin_tables : 检查sj-nest中的function dependency(EQ_REF(outer_table))，对于这种table，从sj-nest中抽取出来，放到外层join nest->join_list中，对于EQ_REF，是保证能join到，且只能join到一条的，所以对于存在性语义来说，这个it表是没有用的，抽取出来后，sj-nest的相关字段都要调整，sj-nest将可能被标记为correlated（内层条件变为了相关条件）

 JOIN::set_semijoin_embedding

 设置每个join_tab->emb_sj_nest，为其table所在的sj-nest对象

 SELECT_LEX::update_semijoin_strategies

 设置每个sj_nest->nested_join->sj_enabled_strategies，为可以考虑的SJ策略

 optimize_semijoin_nests_for_materialization

 对每个sj-nest对象： 判断其是否可以做物化：

 * sj_corr_tables，相关子查询，不能物化
* semijoin_types_allow_materialization ，根据其sj_outer_exprs/sj_inner_exprs的类型，判断是否可以做materialized scan/lookup
* 如果可以物化，则调用Optimize_table_order::choose_table_order，对这个partial join list做join reordering，获取最优的执行顺序
 
 calculate_materialization_costs 计算物化的相关代价，包括cost + rowcount
* get_partial_join_cost 计算这部分join的cost/rowcount，估计物化后的distinct_rowcount，即去重之后行数

 Optimize_table_order::choose_table_order

 外层qb的join reordering过程，这里会处理所有semi-join的可能执行策略，计算其代价，并选择最优方案，核心函数是 advance_sj_state，关于greedy search的具体流程就不描述了，由于MySQL早期无法支持hash join，它对semi-join的实现方式更多的耦合了其原有的这种left-deep, nested-loop的执行方式，为了找到最优执行方式，需要尽量的允许不同的join order可以被考虑到，因此在reordering的过程中，具体就是best_access_path完成时，对semi-join的可能状态进行考量，我们focus在某个level（某个递归长度）选定一个table之后：

 * Optimize_table_order::advance_sj_state

 在POSITION中，包含了每种可能的sj strategy的状态变量,这个函数更新这些变量，
* 如果在当前的join prefix前提下，某种semi-join strategy所要求的结构可以被满足(所有需要的tables都已经在join prefix中)，对prefix中一定范围内的tables+positions(sj strategy所涉及的那些），重算cost + rowcount，替换掉原有的POSITION信息，并设置POSITION::sj_strategy

 **Firstmatch**

 POSITION::first_firstmatch_table : 表示第一个可能的first match table对象 POSITION::firstmatch_need_tables firstmatch : 需要的inner tables

 POSITION::first_firstmatch_rtbl : 优化中间状态，表示remaining_tables

 当所有sj_depends_on的outer table都在join prefix中，且当前table是第一个inner table，则标记进入FirstMatch的考虑范围，当所有Inner table也都在prefix中时，得到一个完整的duplicate_generating_range

 调用semijoin_firstmatch_loosescan_access_paths，重算整个range中的rowcount/cost
* **Loosescan**

 POSITION::first_loosescan_table : 执行loosescan的driving table
POSITION::loosescan_need_tables : 包括sj_inner_tables | sj_depends_on
当所有sj_corr_tables的outer table都在join prefix中，其余outer table在后面，当前是第一个sj inner table且当前table使用index时，标记进入LooseScan的考虑范围，当所有inner tables + outer tables都在join prefix之后，得到一个完整的 duplicate_generating_range

 调用semijoin_firstmatch_loosescan_access_paths, 重算整个range中的rowcount/cost
* **Materialize**

 semijoin_order_allows_materialization 判断要使用的物化策略： 必须所有inner tables在join prefix上紧邻在一起 基于heuristic，如果后续表中还有outer tables，则使用Scan

 调用semijoin_mat_scan_access_paths/semijoin_mat_lookup_access_paths，更新相关POSITION的cost/rowcount，这里可以利用上optimize_semijoin_nests_for_materialization中，已经得到的物化cost + rowcount这些
* **DuplicateWeedOut**

 POSITION::first_dupsweedout_table : 第一个sj inner table
POSITION::dupsweedout_tables : 包括sj_inner_tables | sj_depends_on
一旦当前table是first inner table，就可以开始考虑这个策略了（最为灵活），当所有sj_inner_tables + sj_depends_on outer tables都在join prefix当中时，我们得到一个有效的duplicate_generating_range

 调用semijoin_dupsweedout_access_paths，重算整个range内的rowcount/cost

 每种strategy的策略信息，会记录在对应POSITION上相关字段中 完成greedy_search之后

 Optimize_table_order::fix_semijoin_strategies

 在完成join order优化后，由于sj的策略是每递归到新level，添加一个新table时判断一次，有可能出现前后不同tab使用不同的策略情况，这里要从后->前的遍历(后的总是最新的) ，确定最终的策略

 Note : POSITION::sj_strategy 总是记在有效range的最后一个表上的，这个函数会将最终选中的strategy信息，记录到第一个inner table上，主要是n_sj_tables / sj_strategy字段，这里n_sj_tables不止是inner table的数量，而是整个duplicate_generating_range的tables数量，由第一个inner table + n_sj_tables，即可找到整个range的最后一个table

 JOIN::get_best_combination

 根据得到的best_positions，设置join_tabs，这里会调整sjm的相关结构，把inner tables放到primary tables的后面(tmp table之后），把sjm放到primary tables当中，并创建并创建Semijoin_mat_exec结构，放在表示sjm的那个JOIN_TAB上

 * JOIN::setup_semijoin_materialized_table

 在该materialize table上，设置必要的结构，并创建tmp table，使用sj_inner_exprs作为table的field list并标记distinct，这样在物化完成时也就完成了内表的去重

 create_keyuse_for_table

 利用Semijoin_mat_optimize::mat_fields + sj_outer_exprs 创建keyuse对象，做lookup时要使用 根据是lookup/scan，计算read_cost/row_fetched这些代价信息，记录在sjm所在的JOIN_TAB对应的POSITION上
* JOIN::set_semijoin_info

 在JOIN_TAB数组中，对于属于选定sj strategy的执行range中每个JOIN_TAB，设置m_first_sj_inner/m_last_sj_inner

### Execution Phase

在完成基本的优化后，最重要的函数就是setup_semijoin_dups_elimination，它会创建具体的semi-join执行结构，这个函数的注释中包含了非常重要的信息，描述了每种执行策略，各自可以产生怎样的QEP_TAB序列，这里的描述也将以此为基础，对各个策略的执行结构和必要函数/结构/字段做描述，并分析下MySQl是怎么保证semi-join结果正确性的

setup_semijoin_dups_elimination

* **SJ_OPT_MATERIALIZE_LOOKUP/SJ_OPT_MATERIALIZE_SCAN** ：物化策略的思路是对内表做去重，其可能的执行结构

 MaterializeLookup

`(ot|nt)* [ it (it)* ] (nt)*
+------+ +==========+ +---+
 (1) (2) (3)
`

所有inner table必须邻接排列，且在所有outer tables的后面

​ MaterializeScan

` (ot|nt)* [ it (it)* ] (ot|nt)*
 +------+ +==========+ +-----+
 (1) (2) (3)
`

所有inner table必须邻接排列，且在所有outer tables的前面 由于sj_strategy都标记在了第一个inner table上，而物化的inner table不在primary tables中，这里只有sjm table，因此这里无需做处理，具体执行时，在sjm所对应的QEP_TAB开始执行时，会先通过preprare_scan函数完成inner tables的物化+去重，后续无需特殊处理

* **SJ_OPT_LOOSE_SCAN** ：loosescan的思路也是对内表做去重

`(ot|ct|nt) [ loosescan_tbl (ot|nt|it)* it ] (ot|nt)*
+--------+ +===========+ +=============+ +------+
 (1) (2) (3) (4)
`

这里的要求是，所有non-trivially correlated outer tables必须都在inner tables的前面，这是必须的因为loosescan是对内表去重，如果有相关表在外层，它会决定内表的内容，因此必须要在于相关表join完成后再做去重，避免数据错误（如 本可以通过与ct做join过滤掉的tuple，却被保留下来参与了与nt/ot的join ) loosescan要求在第一个inner table上使用Index，对后续range（3）使用了first match策略，从而保证整个inner table范围内，做到了去重，在(3)的范围内，目前MySQL的实现是不允许有ot/nt的，虽然图示如此 相关执行结构： last_sj_tab，也就是整个range的最后一个tab的idx，设置给loose scan driving tab->match_tab driving tab的idx和last_sj_tab的idx，设置给last_sj_tab的firstmatch_return/match_tab loosescan_tbl->loosescan_buf/loosescan_tbl->loosescan_key_len，用来保存looscan keypart的buffer及其长度，用来在上层显式的skip掉重复index entry

* **SJ_OPT_FIRST_MATCH** ：其执行思路与loose scan有些类似，也是对内表做去重

` (ot|nt)* [ it ((it|nt)* it) ] (nt)*
 +------+ +==================+ +---+
 (1) (2) (3)
`

在inner table的range之内，是可以有nt表的，对于这种情况，MySQL会使用一种”split jump”的执行方式，即： 这里的序号1/2是第i行的意思 通过这种方式，nt1中的每行都还是正常被join到的，只是通过jump的方式保证了inner table不会有重复行 执行结构： firstmatch_return在每个jump range最后一个inner table上，记录跳回的range的起始tab idx match_tab 记录last inner table的位置，不随jump range变化

`ot -> [it1 -> nt1 -> it3 ]
=> 1 -> 1 -> 1 -> 1
<- jump
2 -> 1
2 -> 2
<- jump
..... 
<- jump
2 -> 1
2 -> 2 -> 1 -> 1
....
`

* **SJ_OPT_DUPS_WEEDOUT** ：DuplicateWeedout的思路是不同的，它基于外表做去重，对join order基本没有限制，只需要将join后的所有数据，针对outer table rowid做一次去重，保证每个outer table的一行row combination，只会join出一行结果，就足够了，这个去重是在first inner table上，创建一个tmp table，将去重使用的各个outer table rowid拼接起来作为distinct key，插入即可完成去重。

```
(ot|nt)* [ it ((it|ot|nt)* (it|ot))] (nt)*
+------+ +=========================+ +---+
 (1) (2) (3)

```

DuplicateWeedout的执行和MySQL nested-loop的执行方式有比较强的耦合： 

如果在range (2) 范围内没有使用join buffer，则从range (1)中输入进来的每一行row combination，可以通过一个have_confluent_row标记来简化执行，也就是判断如果prefix部分到来的是不是新的一行 row comibination，则由于已经join过了，所以不再进一步处理，只有当每次时新的行时，才做处理！这样保证了range (1) 范围内的outer table，是没有重复列的，而对于range (2)范围内的ot|nt，则需要将其row id加入到tmp table distinct key中来完成去重如果在range (2) 范围内使用了join buffer，上面所描述的方案将无法成立，因为没法保证每到来一个range (1) 中的 row combination，是新的一行数据因此只能将整个range (1) + range (2)的所有 ot+nt的rowid，作为distinct keypart，加入到tmp table中，完成去重 执行结构：create_sj_tmp_table: 根据SJ_TMP_TABLE::TAB数组中描述的需要记录rowid的各个table，创建SJ_TMP_TABLE对象，保存在first inner table->flush_weedout_table上，以及 range的最后一个inner table->check_weed_out_table上 create_duplicate_weedout_tmp_table： 创建实际去重的tmp table + distinct key，这个table保存在SJ_TMP_TABLE::tmp_table字段上。

## 总结

文中涉及的代码细节很多也比较复杂，需要大家结合实际代码来看。

另外在调试中发现了社区对于semi-join materialization代价信息的填充中的一个明显bug，具体问题是在完成sjm的优化后，需要将sjm table的代价信息存入到外层join序列的POSITION数组中，而MySQL选择了错误的position序列导致访问了不正确的结构，具体参看 [https://bugs.mysql.com/bug.php?id=103997](https://bugs.mysql.com/bug.php?id=103997)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)