# MySQL · 源码分析 · Range (Min-Max Tree)结构分析

**Date:** 2021/06
**Source:** http://mysql.taobao.org/monthly/2021/06/03/
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

 ## MySQL · 源码分析 · Range (Min-Max Tree)结构分析 
 Author: daoke 

 ## 概述
条件查询被广泛的使用在SQL查询中，复杂条件是否能在执行过程中被优化，比如恒为true或者false的条件，可以合并的条件。另外，由于索引是MySQL访问数据的基本方式，已经追求更快的访问方式，SARGable这个概念已经被我们遗忘了，因为他已经成为默认必要的方法（Search ARGument ABLE）。MySQL如何组织复杂条件并计算各个Ranges所影响到的对应可以使用的索引的代价和使用索引的不同快速方式，从而选出最优的计划，另外，MySQL分区表如何有效的进行条件剪枝？这都离不开一个结构mm tree。下面就让我们来揭开它的真实面纱。

我们在解决客户问题时
本文章的例子如下：

`create table tmp_sel_arg(kp1 int, kp2 int, kp3 int, kp4 int);
create index ind_tmp_sel_arg on tmp_sel_arg(kp1, kp2, kp3);
select * from tmp_sel_arg where 
 (kp1 < 1 AND kp2=5 AND (kp3=10 OR kp3=12)) OR 
 (kp1=2 AND (kp3=11 OR kp3=14)) OR 
 (kp1=3 AND (kp3=11 OR kp3=14));
`
源代码路径：./sql/opt_range.cc

## SEL_TREE 图结构
SEL_TREE结构的定义，记录选择的表上所有可选择的索引的图结构，针对索引的类森林数组

`class SEL_TREE { 
 Mem_root_array<SEL_ROOT *> keys;
 Key_map keys_map; /* bitmask of non-NULL elements in keys */
 enum Type { IMPOSSIBLE, ALWAYS, MAYBE, KEY, KEY_SMALLER } type;
 .......
}

IMPOSSIBLE: if keys[i]->type == SEL_ROOT::Type::IMPOSSIBLE for some i, then type == SEL_TREE::IMPOSSIBLE. Rationale: if the predicate for one of the indexes is always false, then the full predicate is also always false.

ALWAYS: if either (keys[i]->is_always()) or (keys[i] == NULL) for all i, then type == SEL_TREE::ALWAYS. Rationale: the range access method will not be able to filter out any rows when there are no range predicates that can be used to filter on any index.

KEY: There are range predicates that can be used on at least one index.

KEY_SMALLER: There are range predicates that can be used on at least one index. In addition, there are predicates that cannot be directly utilized by range access on key parts in the same index. These unused predicates makes it probable that the row estimate for range access on this index is too pessimistic.
`
## SEL_ROOT 图结构
索引Range的图结构，针对于索引列的森林树结构

`A graph of (possible multiple) key ranges, represented as a red-black binary tree. There are three types (see the Type enum); if KEY_RANGE, we have zero or more SEL_ARGs, described in the documentation on SEL_ARG.

class SEL_ROOT {
 /**
 Used to indicate if the range predicate for an index is always
 true/false, depends on values from other tables or can be
 evaluated as is.
 */
 enum class Type {
 /** The range predicate for this index is always false. */
 IMPOSSIBLE,
 /**
 There is a range predicate that refers to another table. The
 range access method cannot be used on this index unless that
 other table is earlier in the join sequence. The bit
 representing the index is set in JOIN_TAB::needed_reg to
 notify the join optimizer that there is a table dependency.
 After deciding on join order, the optimizer may chose to rerun
 the range optimizer for tables with such dependencies.
 */
 MAYBE_KEY,
 /**
 There is a range condition that can be used on this index. The
 range conditions for this index in stored in the SEL_ARG tree.
 */
 KEY_RANGE
 } type;
 SEL_ARG *root; // The root node of the tree
}
`

## SEL_ARG 图结构
记录了索引列的RB Tree结构，有两种表示形式，内部link和RB tree，各个keypart通过next_key_part相连

`class SEL_ARG {
 ......
 Field *field{nullptr};
 uchar *min_value, *max_value; // Pointer to range
 ......
 SEL_ARG *left, *right; /* R-B tree children */
 SEL_ARG *next, *prev; /* Links for bi-directional interval list */
 SEL_ARG *parent{nullptr}; /* R-B tree parent (nullptr for root) */
 /*
 R-B tree of intervals covering keyparts consecutive to this
 SEL_ARG. See documentation of SEL_ARG GRAPH semantics for details.
 */
 SEL_ROOT *next_key_part{nullptr};
 ......
}
`
### Red-Black树结构
通过下面例子来展示内部的RB树结构如图：

`(kp1 < 1 AND kp2=5 AND (kp3=10 OR kp3=12)) OR     (kp1=2 AND (kp3=11 OR kp3=14)) OR 
   (kp1=3 AND (kp3=11 OR kp3=14));
Where / and \ denote left and right pointers and ... denotes next_key_part pointers to the root of the R-B tree of intervals for consecutive key parts.

 tree->keys[0]
 0x7f59cf68a588
 SEL_ROOT::Type::KEY_RANGE 0x7f59cf68aa90
 use_count = 1, elements = 3 SEL_ROOT::Type::KEY_RANGE
 || --> use_count = 1, elements = 2
 || : ||
 \/ : \/
 0x7f59cf68ac40 : 0x7f59cf68aa10
 min_item=max_item=2 BLACK : min_item=max_item=11 BLACK 
 +-------+ : +--------+
 | kp1=2 |.............. | kp3=11 |
 +-------+ +--------+
 / \ \ 
 0x7f59cf68a508 0x7f59cf68af88 0x7f59cf68ab28
 max_item=1 RED min_item=max_item=3 RED min_item=max_item=14 RED
 +-------+ +-------+ +--------+
 | kp1<1 | | kp1=3 | | kp3=14 |
 +-------+ +-------+ +--------+
 : :
 ...... .......
 : :
 SEL_ROOT::Type::KEY_RANGE SEL_ROOT::Type::KEY_RANGE
 use_count = 1, elements = 1 use_count = 1, elements = 2
 || ||
 \/ \/
 0x7f59cf68a8f8 0x7f59cf68ad58
 min_item=max_item=5 BLACK min_item=max_item=11 BLACK
 +-------+ +--------+
 | kp2=5 | | kp3=11 |
 +-------+ +--------+
 . \ 
 ...... 0x7f59cf68ae70
 . min_item=max_item=14 RED 
 0x7f59cf68a6a8 +--------+
 SEL_ROOT::Type::KEY_RANGE | kp3=14 |
use_count = 1, elements = 2 +--------+
 ||
 \/ 
 0x7f59cf68a628
min_item=max_item=10 BLACK 
 +--------+
 | kp3=10 | 
 +--------+ 
 \ 
 0x7f59cf68a740
 min_item=max_item=12 RED 
 +--------+
 | kp3=12 |
 +--------+
`
### 内部的双向链表
SEL_ARG一组对象合成一个SEL_ROOT的图结构，内部通过SEL_ARG::next/prev来关联同一个索引列条件”OR”，通过next_key_part来关联不同索引列条件的”AND”

`tree->keys[0] (SEL_ROOT::Type::KEY_RANGE)
 |
 | $ $
 | part=1 $ part=2 $ part=3
 | 0x7f59cf68a508 $ 0x7f59cf68a8f8 $ 0x7f59cf68a628
 | left(RED) $ root(BLACK) $ root(BLACK) 
 | +-------+ $ +-------+ $ +--------+
 | | kp1<1 |--------$------->| kp2=5 |-------$------>| kp3=10 |
 | +-------+ $ +-------+ $ +--------+
 | | $ $ |
 | | $ $ right(RED) 
 | | $ $ 0x7f59cf68a740 
 | | $ $ +--------+
 | | $ $ | kp3=12 |
 | | $ $ +--------+
 | | $ $
 | 0x7f59cf68ac40 $ $ 0x7f59cf68aa10 
 | root(BLACK) $ $ root(BLACK) 
 | +-------+ $ $ +--------+
 \------>| kp1=2 |-------$------------------------$------>| kp3=11 |
 +-------+ $ $ +--------+
 | $ $ | 
 | $ $ right(RED) 
 | $ $ 0x7f59cf68ab28 
 | $ $ +--------+
 | $ $ | kp3=14 |
 | $ $ +--------+
 | $ $ 
 right(RED) $ $ root(BLACK) 
 0x7f59cf68af88 $ $ 0x7f59cf68ad58 
 +-------+ $ $ +--------+
 | kp1=3 |-------$------------------------$------>| kp3=11 |
 +-------+ $ $ +--------+
 $ $ | 
 $ $ right(RED) 
 $ $ 0x7f59cf68ae70 
 $ $ +--------+
 $ $ | kp3=14 |
 $ $ +--------+
`
### SEL_ARG flag标识
SEL_ARGs flag一共有6个bit来记录不同含义。前四个是借鉴enum key_range_flags.

`flag = min_flag && max_flag
 /*
 The valid SEL_ARG representations are enumerated here:

 bit5 bit4 bit3 bit2 bit1 bit0
 NULL NULL NEAR NEAR NO NO
 flag MAX MIN MAX MIN MAX MIN notation expr
 0 0 0 0 0 0 0 [a, a] X == a
 0 0 0 0 0 0 0 [a, b] X >= a && X <= b
 4 0 0 0 1 0 0 (a, b] X > a && X <= b
 8 0 0 1 0 0 0 [a, b) X >= a && X < b
 C 0 0 1 1 0 0 (a, b) X > a && X < b
 1 0 0 0 - 0 1 b] X <= b
 2 0 0 - 0 1 0 [a, X >= a
 6 0 0 - 1 1 0 (a, X > a
 9 0 0 1 - 0 1 b) X < b

 10 0 1 - 0 0 0 [NULL, b] X IS NULL || X <= b
 18 0 1 - 0 1 0 [NULL, b) X IS NULL || X < b
 30 1 1 - 0 0 0 [NULL, NULL] X IS NULL
 14 0 1 0 1 0 0 (NULL, b] X <= b (nullable X)
 1C 0 1 1 1 0 0 (NULL, b) X < b (nullable X)
 16 - 1 - 1 1 0 (NULL, X IS NOT NULL
`

## 如何构造mm tree
### get_mm_tree (mm=min_max) 函数
Range分析模块，用于找到所有可能索引的mm tree，构造的ranges可能会比原有的条件范围更大，比如下面简单的两个索引列和条件：

`"WHERE fld1 > 'x' AND fld2 > 'y'"
`
这种场景，不论选择fld1的索引或者fld2的索引，可能读到的行数都比最终结果集多

`static SEL_TREE *get_mm_tree(RANGE_OPT_PARAM *param, Item *cond) {
 // Item And
 while itemlist 
 new_tree = get_mm_tree
 tree = tree_and(param, tree, new_tree);
 // Item Or
 while itemlist 
 new_tree = get_mm_tree
 tree = tree_or(param, tree, new_tree);
}
`

### get_full_func_mm_tree 函数
需要考虑所有等值列，构造多个mm trees
WHERE t1.a=t2.a AND t2.a > 10 ==> WHERE t1.a=t2.a AND t2.a > 10 AND t1.a > 10 
field->item_equal

`The class Item_equal is used to represent conjunctions of equality | sql/sql_gather.cc:1824: List_iterator_fast<Item_equal> li(cond_equal->current_level)$
 predicates of the form field1 = field2, and field=const in where | sql/sql_gather.cc:1825: Item_equal *item;
 conditions and on expressions.
`

A BETWEEN predicate : 
fi [NOT] BETWEEN c1 AND c2
AND j,k (f1j <=c AND f2k<=c)

IN predicates:
f IN (c1,…,cn)
c IN (c1,…,f,…,cn) –> never try to narrow the index scan

### get_func_mm_tree 函数
构造mm tree的入口函数

`switch(func type) {
 Item_func::NE_FUNC : get_ne_mm_tree
 Item_func::BETWEEN : get_mm_parts
 Item_func::IN_FUNC : get_func_mm_tree_from_in_predicate
 default : get_mm_parts // <, <=, =, >=, >, LIKE, IS NULL, IS NOT NULL and GIS functions.
}
`
### get_ne_mm_tree 函数
针对于 SEL_TREE for <> or NOT BETWEEN 条件构造mm tree
kp1 <> 1 => kp1 < 1 or kp1 > 1

tree_or(kp1 < 1 , kp1 > 1)

`(gdb) my sel tree
$k0 (SEL_TREE *) 0x7f7cf17c0cd0 [type=SEL_TREE::KEY,keys.m_size=1]
`--$k1 (SEL_ROOT *) 0x7f7cf17c0dc8 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=2]
 `--$k2 (SEL_ARG *) 0x7f7cf17c0d48 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=8 '\b', part=0 '\000', selectivity=1]
 | field = $k3 (Item_field *) 0x7f7cf04af268 field = test.tmp_sel_arg1.kp1
 | scope = ( -infinity, $k4 (Item_int *) 0x7f7cf04ae670 value = 1 )
 `--$k6 (SEL_ARG *) 0x7f7cf17c0e68 [color=SEL_ARG::RED, is_asc=true, minflag=4 '\004', maxflag=2 '\002', part=0 '\000', selectivity=1]
 | field = $k7 (Item_field *) 0x7f7cf04af268 field = test.tmp_sel_arg1.kp1
 | scope = ( $k8 (Item_int *) 0x7f7cf04ae670 value = 1, +infinity )
`
### get_mm_parts 函数
通用的通过Item cond构造mm tree的方法

`for (every key part) {
 SEL_ROOT root = get_mm_leaf
 SEL_TREE tree.root = root
}
`

### get_func_mm_tree_from_in_predicate 函数
通过IN value构造mm tree

`select * from tmp_sel_arg1 where kp1 in (1, 2, 3, 4, 5) and kp2 < 4;
(gdb) my sel tree
$d0 (SEL_TREE *) 0x7f7cf17c0cd0 [type=SEL_TREE::KEY,keys.m_size=2]
|--$d1 (SEL_ROOT *) 0x7f7cf17c0dd0 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=5]
| `--$d2 (SEL_ARG *) 0x7f7cf17c0e70 [color=SEL_ARG::BLACK, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=0 '\000', selectivity=1]
| | field = $d3 (Item_field *) 0x7f7cf0444938 field = test.tmp_sel_arg1.kp1
| | equal = [ $d4 (Item_int *) 0x7f7cf0d96420 value = 2 ]
| |--$d6 (SEL_ARG *) 0x7f7cf17c0d50 [color=SEL_ARG::BLACK, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=0 '\000', selectivity=1]
| | | field = $d7 (Item_field *) 0x7f7cf0444938 field = test.tmp_sel_arg1.kp1
| | | equal = [ $d8 (Item_int *) 0x7f7cf0d96320 value = 1 ]
| | `--$d12 (SEL_ROOT *) 0x7f7cf17c1370 [type=SEL_ROOT::Type::KEY_RANGE, use_count=5, elements=1]
| | `--$d13 (SEL_ARG *) 0x7f7cf17c12f0 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=8 '\b', part=1 '\001', selectivity=1]
| | | field = $d14 (Item_field *) 0x7f7cf0444ab0 field = test.tmp_sel_arg1.kp2
| | | scope = ( -infinity, $d15 (Item_int *) 0x7f7cf0d96b70 value = 4 )
| |--$d18 (SEL_ARG *) 0x7f7cf17c10b0 [color=SEL_ARG::BLACK, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=0 '\000', selectivity=1]
| | | field = $d19 (Item_field *) 0x7f7cf0444938 field = test.tmp_sel_arg1.kp1
| | | equal = [ $d20 (Item_int *) 0x7f7cf0d96668 value = 4 ]
| | |--$d22 (SEL_ARG *) 0x7f7cf17c0f90 [color=SEL_ARG::RED, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=0 '\000', selectivity=1]
| | | | field = $d23 (Item_field *) 0x7f7cf0444938 field = test.tmp_sel_arg1.kp1
| | | | equal = [ $d24 (Item_int *) 0x7f7cf0d96558 value = 3 ]
| | | `--$d28 (SEL_ROOT *) 0x7f7cf17c1370 [type=SEL_ROOT::Type::KEY_RANGE, use_count=5, elements=1]
| | | `--$d29 (SEL_ARG *) 0x7f7cf17c12f0 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=8 '\b', part=1 '\001', selectivity=1]
| | | | field = $d30 (Item_field *) 0x7f7cf0444ab0 field = test.tmp_sel_arg1.kp2
| | | | scope = ( -infinity, $d31 (Item_int *) 0x7f7cf0d96b70 value = 4 )
| | |--$d34 (SEL_ARG *) 0x7f7cf17c11d0 [color=SEL_ARG::RED, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=0 '\000', selectivity=1]
| | | | field = $d35 (Item_field *) 0x7f7cf0444938 field = test.tmp_sel_arg1.kp1
| | | | equal = [ $d36 (Item_int *) 0x7f7cf0d96778 value = 5 ]
| | | `--$d40 (SEL_ROOT *) 0x7f7cf17c1370 [type=SEL_ROOT::Type::KEY_RANGE, use_count=5, elements=1]
| | | `--$d41 (SEL_ARG *) 0x7f7cf17c12f0 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=8 '\b', part=1 '\001', selectivity=1]
| | | | field = $d42 (Item_field *) 0x7f7cf0444ab0 field = test.tmp_sel_arg1.kp2
| | | | scope = ( -infinity, $d43 (Item_int *) 0x7f7cf0d96b70 value = 4 )
| | `--$d46 (SEL_ROOT *) 0x7f7cf17c1370 [type=SEL_ROOT::Type::KEY_RANGE, use_count=5, elements=1]
| | `--$d47 (SEL_ARG *) 0x7f7cf17c12f0 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=8 '\b', part=1 '\001', selectivity=1]
| | | field = $d48 (Item_field *) 0x7f7cf0444ab0 field = test.tmp_sel_arg1.kp2
| | | scope = ( -infinity, $d49 (Item_int *) 0x7f7cf0d96b70 value = 4 )
| `--$d52 (SEL_ROOT *) 0x7f7cf17c1370 [type=SEL_ROOT::Type::KEY_RANGE, use_count=5, elements=1]
| `--$d53 (SEL_ARG *) 0x7f7cf17c12f0 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=8 '\b', part=1 '\001', selectivity=1]
| | field = $d54 (Item_field *) 0x7f7cf0444ab0 field = test.tmp_sel_arg1.kp2
| | scope = ( -infinity, $d55 (Item_int *) 0x7f7cf0d96b70 value = 4 )
`--$d58 (SEL_ROOT *) 0x7f7cf17c1420 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=1]
 `--$d59 (SEL_ARG *) 0x7f7cf17c13a0 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=8 '\b', part=0 '\000', selectivity=1]
 | field = $d60 (Item_field *) 0x7f7cf0444ab0 field = test.tmp_sel_arg1.kp2
 | scope = ( -infinity, $d61 (Item_int *) 0x7f7cf0d96b70 value = 4 )
`
对于IN values

`foreach op->arguments
 tree_or (get_mm_parts)
`

NOT IN是有可能把内存oom的，因此not in的个数收到NOT_IN_IGNORE_THRESHOLD的控制，小于这个阈值，创建mmtree，否则返回null

`const uint NOT_IN_IGNORE_THRESHOLD = 1000; // If we have t.key NOT IN (null, null, ...) or the list is too long
select * from tmp_sel_arg1 where kp1 not in (1, 2, 3, 4, 5)
(gdb) my st tree
$b0 (SEL_TREE *) 0x7f7cf040acd0 [type=SEL_TREE::KEY,keys.m_size=1]
`--$b1 (SEL_ROOT *) 0x7f7cf040adc8 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=6]
 `--$b2 (SEL_ARG *) 0x7f7cf040ae68 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=8 '\b', part=0 '\000', selectivity=1]
 | field = $b3 (Item_field *) 0x7f7cf0446438 field = test.tmp_sel_arg1.kp1
 | scope = ( $b4 (Item_int *) 0x7f7cf04add58 value = 5, $b5 (Item_int *) 0x7f7cf04add58 value = 5 )
 |--$b6 (SEL_ARG *) 0x7f7cf040ad48 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=8 '\b', part=0 '\000', selectivity=1]
 | | field = $b7 (Item_field *) 0x7f7cf0446438 field = test.tmp_sel_arg1.kp1
 | | scope = ( -infinity, $b8 (Item_int *) 0x7f7cf04add58 value = 5 )
 `--$b11 (SEL_ARG *) 0x7f7cf040b0a8 [color=SEL_ARG::RED, is_asc=true, minflag=4 '\004', maxflag=8 '\b', part=0 '\000', selectivity=1]
 | field = $b12 (Item_field *) 0x7f7cf0446438 field = test.tmp_sel_arg1.kp1
 | scope = ( $b13 (Item_int *) 0x7f7cf04add58 value = 5, $b14 (Item_int *) 0x7f7cf04add58 value = 5 )
 |--$b15 (SEL_ARG *) 0x7f7cf040af88 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=8 '\b', part=0 '\000', selectivity=1]
 | | field = $b16 (Item_field *) 0x7f7cf0446438 field = test.tmp_sel_arg1.kp1
 | | scope = ( $b17 (Item_int *) 0x7f7cf04add58 value = 5, $b18 (Item_int *) 0x7f7cf04add58 value = 5 )
 `--$b21 (SEL_ARG *) 0x7f7cf040b1c8 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=8 '\b', part=0 '\000', selectivity=1]
 | field = $b22 (Item_field *) 0x7f7cf0446438 field = test.tmp_sel_arg1.kp1
 | scope = ( $b23 (Item_int *) 0x7f7cf04add58 value = 5, $b24 (Item_int *) 0x7f7cf04add58 value = 5 )
 `--$b26 (SEL_ARG *) 0x7f7cf040b2e8 [color=SEL_ARG::RED, is_asc=true, minflag=4 '\004', maxflag=2 '\002', part=0 '\000', selectivity=1]
 | field = $b27 (Item_field *) 0x7f7cf0446438 field = test.tmp_sel_arg1.kp1
 | scope = ( $b28 (Item_int *) 0x7f7cf04add58 value = 5, +infinity )
`
记录NOT IN的变量，is_negated = true
t.key NOT IN (c1, c2, …), c{i} are constants. 
=>
($MIN<t.key<c1) OR (c1<t.key<c2) OR (c2<t.key<c3) OR … (*), $MIN is either “-inf” or NULL

` 1. Get a SEL_TREE for "(-inf|NULL) < X < c_0" interval.
 tree = get_mm_parts(param, op, field, Item_func::LT_FUNC, value_item);
 2. Get a SEL_TREE for "-inf < X < c_i" interval
 tree2 = get_mm_parts(param, op, field, Item_func::LT_FUNC, value_item);
 3. Change all intervals to be "c_{i-1} < X < c_i
 tree = tree_or(param, tree, tree2);
 4. Get the SEL_TREE for the last "c_last < X < +inf" interval
 tree2 = get_mm_parts(param, op, field, Item_func::GT_FUNC, value_item);
 tree = tree_or(param, tree, tree2);
`
### get_mm_leaf 函数
构造SEL_ROOT结构

### tree_and 函数
遍历所有keys，通过key_and合并 tree1和tree2

`for (uint idx = 0; idx < param->keys; idx++) {
 key_and // Produce a SEL_ARG graph that represents "key1 AND key2"
}
`

### tree_or 函数
遍历所有keys，通过key_or合并 tree1和tree2
a) 可能产生简单的range (in tree->keys[]) 
b) 可能产生index merge range(in tree->merges)

`for (uint idx = 0; idx < param->keys; idx++) {
 key_or
}
`

### key_and 函数
尽可能合并 SEL_ARG “key1 AND key2” .
kp1 > 1 and kp1 < 5 (sel_arg1(min_value=1) and sel_arg2(max_value=5))
—-> 1 < kp1 < 5 (new_sel_arg(min_value=1, max_value=5))

kp1 > 1 and kp1 > 5 (sel_arg1(min_value=1) and sel_arg2(min_value=5))
—-> kp1 > 5 (new_sel_arg(min_value=5))

### key_or 函数
尽可能合并 SEL_ARG “key1 OR key2” .
=> expr1 OR expr2.
对于重叠的子范围，递归调用key_or：
(1) ( 1 < kp1 < 10 AND 1 < kp2 < 10 )
(2) ( 2 < kp1 < 20 AND 4 < kp2 < 20 )
key_or( 1 < kp2 < 10, 4 < kp2 < 20 ) => 1 < kp2 < 2

### Range 概念
` Notation for illustrations used in the rest of this function:

 Range: [--------]
 ^ ^
 start stop

 Two overlapping ranges:
 [-----] [----] [--]
 [---] or [---] or [-------]

 Ambiguity: ***
 The range starts or stops somewhere in the "***" range.
 Example: a starts before b and may end before/the same place/after b
 a: [----***]
 b: [---]

 Adjacent ranges:
 Ranges that meet but do not overlap. Example: a = "x < 3", b = "x >= 3"
 a: ----]
 b: [----
`

比较函数cmp_xxx_to_yyy {xxx|yyy = min|max}
find_range–>cmp_min_to_min

` initialize cur_key1 to the latest range in key1 that starts the
 same place or before the range in cur_key2 starts

 cur_key2: [------]
 key1: [---] [-----] [----]
 ^
 cur_key1
 
 Used to describe how two key values are positioned compared to
 each other. Consider key_value_a.<cmp_func>(key_value_b):

 -2: key_value_a is smaller than key_value_b, and they are adjacent
 -1: key_value_a is smaller than key_value_b (not adjacent)
 0: the key values are equal
 1: key_value_a is bigger than key_value_b (not adjacent)
 2: key_value_a is bigger than key_value_b, and they are adjacent

 Example: "cmp= cur_key1->cmp_max_to_min(cur_key2)"

 cur_key2: [-------- (10 <= x ... )
 cur_key1: -----] ( ... x < 10) => cmp==-2
 cur_key1: ----] ( ... x < 9) => cmp==-1
 cur_key1: ------] ( ... x <= 10) => cmp== 0
 cur_key1: --------] ( ... x <= 12) => cmp== 1
 (cmp == 2 does not make sense for cmp_max_to_min()) 
`

典型的例子和处理逻辑：

`Some typical examples:

1. explain select * from tmp_sel_arg where kp1 between 1 and 10 or kp1 between 0 and 20;

 cur_key2: [--------]
 key1: [****--] [----] [-------]
 ^
 cur_key1

2. explain select * from tmp_sel_arg where kp1 between 1 and 10 or kp1 between 10 and 20;
This is the case:
 cur_key2: [-------]
 cur_key1: [----]

 Result:
 cur_key2: [-------------] => inserted into key1 below
 cur_key1: => deleted 
 
(gdb) my sel key1
$p0 (SEL_ROOT *) 0x7f7cf17c0f48 [type=SEL_ROOT::Type::KEY_RANGE, use_count=0, elements=1]
`--$p1 (SEL_ARG *) 0x7f7cf17c0ec8 [color=SEL_ARG::BLACK, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=0 '\000', selectivity=1]
 | field = $p2 (Item_field *) 0x7f7cf0444df0 field = test.tmp_sel_arg.kp1
 | scope = [ $p3 (Item_int *) 0x7f7cf0d96340 value = 1, $p4 (Item_int *) 0x7f7cf0d96440 value = 10 ]
(gdb) my sel key2
$q0 (SEL_ROOT *) 0x7f7cf17c1220 [type=SEL_ROOT::Type::KEY_RANGE, use_count=0, elements=1]
`--$q1 (SEL_ARG *) 0x7f7cf17c11a0 [color=SEL_ARG::BLACK, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=0 '\000', selectivity=1]
 | field = $q2 (Item_field *) 0x7f7cf0444f68 field = test.tmp_sel_arg.kp1
 | scope = [ $q3 (Item_int *) 0x7f7cf0d969a0 value = 10, $q4 (Item_int *) 0x7f7cf0d96aa0 value = 20 ]
 8838 SEL_ROOT *new_key = key_or(param, key1, key2);
(gdb) my sel new_key
$r0 (SEL_ROOT *) 0x7f7cf17c0f48 [type=SEL_ROOT::Type::KEY_RANGE, use_count=0, elements=1]
`--$r1 (SEL_ARG *) 0x7f7cf17c0ec8 [color=SEL_ARG::BLACK, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=0 '\000', selectivity=1]
 | field = $r2 (Item_field *) 0x7f7cf0444df0 field = test.tmp_sel_arg.kp1
 | scope = [ $r3 (Item_int *) 0x7f7cf0d96340 value = 1, $r4 (Item_int *) 0x7f7cf0d96aa0 value = 20 ] 

3. explain select * from tmp_sel_arg where kp1 between 1 and 10 and kp2 > 1 or kp1 between 10 and 20 and kp2 > 1;
Adjacent ranges with equal next_key_part. Merge like this:

 This is the case:
 cur_key2: [------]
 cur_key1: [-----]

 Result:
 cur_key2: [------]
 cur_key1: [-------------] ? TODO:daoke.wangc why key2 is not deleted
 
$u0 (SEL_TREE *) 0x7f7cf040ac90 [type=SEL_TREE::KEY,keys.m_size=1]
`--$u1 (SEL_ROOT *) 0x7f7cf040af48 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=2]
 `--$u2 (SEL_ARG *) 0x7f7cf040aec8 [color=SEL_ARG::BLACK, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=0 '\000', selectivity=1]
 | field = $u3 (Item_field *) 0x7f7cf04b09e0 field = test.tmp_sel_arg.kp1
 | scope = [ $u4 (Item_int *) 0x7f7cf04af218 value = 10, $u5 (Item_int *) 0x7f7cf04af318 value = 20 ]
 |--$u6 (SEL_ARG *) 0x7f7cf040b470 [color=SEL_ARG::RED, is_asc=true, minflag=0 '\000', maxflag=8 '\b', part=0 '\000', selectivity=1]
 | | field = $u7 (Item_field *) 0x7f7cf04b09e0 field = test.tmp_sel_arg.kp1
 | | scope = [ $u8 (Item_int *) 0x7f7cf04ae6f0 value = 1, $u9 (Item_int *) 0x7f7cf04af218 value = 10 )
 | `--$u12 (SEL_ROOT *) 0x7f7cf040b060 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=1]
 | `--$u13 (SEL_ARG *) 0x7f7cf040afe0 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=2 '\002', part=1 '\001', selectivity=1]
 | | field = $u14 (Item_field *) 0x7f7cf04b0b58 field = test.tmp_sel_arg.kp2
 | | scope = ( $u15 (Item_int *) 0x7f7cf04aed50 value = 1, +infinity )
 `--$u19 (SEL_ROOT *) 0x7f7cf040b570 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=1]
 `--$u20 (SEL_ARG *) 0x7f7cf040b4f0 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=2 '\002', part=1 '\001', selectivity=1]
 | field = $u21 (Item_field *) 0x7f7cf04b0b58 field = test.tmp_sel_arg.kp2
 | scope = ( $u22 (Item_int *) 0x7f7cf04aed50 value = 1, +infinity ) 
 
4. explain select * from tmp_sel_arg where kp1 between 30 and 50 and kp2 > 1 or kp1 between 60 and 120 and kp2 > 20 or kp1 between 10 and 100 and kp2 > 1;

 cur_key2: [****----------------------*******]
 key1: [--] [----] [---] [-----] [xxxx]
 ^ ^ ^
 first last different next_key_part
 
 Result:
 cur_key2: [****----------------------*******]
 [--] [----] [---] => deleted from key1
 key1: [**------------------------***][xxxx]
 ^ ^
 cur_key1=last different next_key_part
 
$ab0 (SEL_TREE *) 0x7f7cf17c0c90 [type=SEL_TREE::KEY,keys.m_size=1]
`--$ab1 (SEL_ROOT *) 0x7f7cf17c0f48 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=3]
 `--$ab2 (SEL_ARG *) 0x7f7cf17c1860 [color=SEL_ARG::BLACK, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=0 '\000', selectivity=1]
 | field = $ab3 (Item_field *) 0x7f7cf0446718 field = test.tmp_sel_arg.kp1
 | scope = [ $ab4 (Item_int *) 0x7f7cf0d96ee8 value = 60, $ab5 (Item_int *) 0x7f7cf0444cd0 value = 100 ]
 |--$ab6 (SEL_ARG *) 0x7f7cf17c0ec8 [color=SEL_ARG::RED, is_asc=true, minflag=0 '\000', maxflag=8 '\b', part=0 '\000', selectivity=1]
 | | field = $ab7 (Item_field *) 0x7f7cf0446228 field = test.tmp_sel_arg.kp1
 | | scope = [ $ab8 (Item_int *) 0x7f7cf0444bd0 value = 10, $ab9 (Item_int *) 0x7f7cf0d96ee8 value = 60 )
 | `--$ab12 (SEL_ROOT *) 0x7f7cf17c1060 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=1]
 | `--$ab13 (SEL_ARG *) 0x7f7cf17c0fe0 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=2 '\002', part=1 '\001', selectivity=1]
 | | field = $ab14 (Item_field *) 0x7f7cf04463a0 field = test.tmp_sel_arg.kp2
 | | scope = ( $ab15 (Item_int *) 0x7f7cf0d96a20 value = 1, +infinity )
 |--$ab18 (SEL_ARG *) 0x7f7cf17c12b8 [color=SEL_ARG::RED, is_asc=true, minflag=4 '\004', maxflag=0 '\000', part=0 '\000', selectivity=1]
 | | field = $ab19 (Item_field *) 0x7f7cf0446718 field = test.tmp_sel_arg.kp1
 | | scope = ( $ab20 (Item_int *) 0x7f7cf0444cd0 value = 100, $ab21 (Item_int *) 0x7f7cf0444028 value = 120 ]
 | `--$ab24 (SEL_ROOT *) 0x7f7cf17c1450 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=1]
 | `--$ab25 (SEL_ARG *) 0x7f7cf17c13d0 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=2 '\002', part=1 '\001', selectivity=1]
 | | field = $ab26 (Item_field *) 0x7f7cf0446890 field = test.tmp_sel_arg.kp2
 | | scope = ( $ab27 (Item_int *) 0x7f7cf0444588 value = 20, +infinity )
 `--$ab30 (SEL_ROOT *) 0x7f7cf17c1960 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=1]
 `--$ab31 (SEL_ARG *) 0x7f7cf17c18e0 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=2 '\002', part=1 '\001', selectivity=1]
 | field = $ab32 (Item_field *) 0x7f7cf0446890 field = test.tmp_sel_arg.kp2
 | scope = ( $ab33 (Item_int *) 0x7f7cf0445230 value = 1, +infinity ) 

5. with next_key_part and not
 This is the case:
 cur_key2: [-------]
 cur_key1: [---------]

 Result:
 cur_key2: deleted 
 cur_key1: [------------]
 
explain select * from tmp_sel_arg where kp1 between 5 and 15 or kp1 between 10 and 30; 
$ae0 (SEL_TREE *) 0x7f7cf040ac90 [type=SEL_TREE::KEY,keys.m_size=1]
`--$ae1 (SEL_ROOT *) 0x7f7cf040af48 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=1]
 `--$ae2 (SEL_ARG *) 0x7f7cf040aec8 [color=SEL_ARG::BLACK, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=0 '\000', selectivity=1]
 | field = $ae3 (Item_field *) 0x7f7cf0444df0 field = test.tmp_sel_arg.kp1
 | scope = [ $ae4 (Item_int *) 0x7f7cf0d96340 value = 5, $ae5 (Item_int *) 0x7f7cf0d96aa0 value = 30 ]
 
 This is the case:
 cur_key2: [-------]
 cur_key1: [---------]

 Result:
 cur_key2: [---]
 cur_key1: [---------]
 
explain select * from tmp_sel_arg where kp1 between 5 and 15 and kp2 > 1 or kp1 between 10 and 30 and kp2 > 1; 
$ad0 (SEL_TREE *) 0x7f7cf17c0c90 [type=SEL_TREE::KEY,keys.m_size=1]
`--$ad1 (SEL_ROOT *) 0x7f7cf17c0f48 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=2]
 `--$ad2 (SEL_ARG *) 0x7f7cf17c0ec8 [color=SEL_ARG::BLACK, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=0 '\000', selectivity=1]
 | field = $ad3 (Item_field *) 0x7f7cf04b09e0 field = test.tmp_sel_arg.kp1
 | scope = [ $ad4 (Item_int *) 0x7f7cf04af218 value = 10, $ad5 (Item_int *) 0x7f7cf04af318 value = 30 ]
 |--$ad6 (SEL_ARG *) 0x7f7cf17c1470 [color=SEL_ARG::RED, is_asc=true, minflag=0 '\000', maxflag=8 '\b', part=0 '\000', selectivity=1]
 | | field = $ad7 (Item_field *) 0x7f7cf04b09e0 field = test.tmp_sel_arg.kp1
 | | scope = [ $ad8 (Item_int *) 0x7f7cf04ae6f0 value = 5, $ad9 (Item_int *) 0x7f7cf04af218 value = 10 )
 | `--$ad12 (SEL_ROOT *) 0x7f7cf17c1060 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=1]
 | `--$ad13 (SEL_ARG *) 0x7f7cf17c0fe0 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=2 '\002', part=1 '\001', selectivity=1]
 | | field = $ad14 (Item_field *) 0x7f7cf04b0b58 field = test.tmp_sel_arg.kp2
 | | scope = ( $ad15 (Item_int *) 0x7f7cf04aed50 value = 1, +infinity )
 `--$ad19 (SEL_ROOT *) 0x7f7cf17c1570 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=1]
 `--$ad20 (SEL_ARG *) 0x7f7cf17c14f0 [color=SEL_ARG::BLACK, is_asc=true, minflag=4 '\004', maxflag=2 '\002', part=1 '\001', selectivity=1]
 | field = $ad21 (Item_field *) 0x7f7cf04b0b58 field = test.tmp_sel_arg.kp2
 | scope = ( $ad22 (Item_int *) 0x7f7cf04aed50 value = 1, +infinity )
`

### 调用堆栈
通过range生成mm tree是在优化阶段进行的，目的是计算代价，选择更优quick访问路径。

`select * from tmp_sel_arg where (kp1=1 and kp2=2 and kp3=3) or        (kp1=1 and kp2=2 and kp3=4) or        (kp1=1 and kp2=3 and kp3=5) or        (kp1=1 and kp2=3 and kp3=6);
#3 0x000000000322ab92 in test_quick_select (thd=0x7f59d160b000, keys_to_use=..., prev_tables=0, limit=18446744073709551615, force_quick_range=false,
 interesting_order=ORDER_NOT_RELEVANT, tab=0x7f59cf4dd690, cond=0x7f59d0ee3740, needed_reg=0x7f59cf4dd6e0, quick=0x7f5ad0efa738)
 at /flash11/daoke.wangc/PolarDB_80/sql/opt_range.cc:4106
#4 0x0000000003384764 in get_quick_record_count (thd=0x7f59d160b000, tab=0x7f59cf4dd690, limit=18446744073709551615)
 at /flash11/daoke.wangc/PolarDB_80/sql/sql_optimizer.cc:5980
#5 0x0000000003383c40 in JOIN::estimate_rowcount (this=0x7f59cf4dbc68) at /flash11/daoke.wangc/PolarDB_80/sql/sql_optimizer.cc:5713
#6 0x000000000338202c in JOIN::make_join_plan (this=0x7f59cf4dbc68) at /flash11/daoke.wangc/PolarDB_80/sql/sql_optimizer.cc:5123
#7 0x0000000003375d5b in JOIN::optimize (this=0x7f59cf4dbc68) at /flash11/daoke.wangc/PolarDB_80/sql/sql_optimizer.cc:688
#8 0x0000000003424076 in SELECT_LEX::optimize (this=0x7f59cf6ad968, thd=0x7f59d160b000) at /flash11/daoke.wangc/PolarDB_80/sql/sql_select.cc:1619
#9 0x00000000034223d2 in Sql_cmd_dml::execute_inner (this=0x7f59cf4db3f0, thd=0x7f59d160b000) at /flash11/daoke.wangc/PolarDB_80/sql/sql_select.cc:753
#10 0x0000000003421d53 in Sql_cmd_dml::execute (this=0x7f59cf4db3f0, thd=0x7f59d160b000) at /flash11/daoke.wangc/PolarDB_80/sql/sql_select.cc:631
#11 0x00000000033a7a71 in mysql_execute_command (thd=0x7f59d160b000, first_level=true) at /flash11/daoke.wangc/PolarDB_80/sql/sql_parse.cc:4897
#12 0x00000000033aa369 in mysql_parse (thd=0x7f59d160b000, parser_state=0x7f5ad0efc6f0, force_primary_storage_engine=false)
 at /flash11/daoke.wangc/PolarDB_80/sql/sql_parse.cc:5722
#13 0x000000000339eaeb in dispatch_command (thd=0x7f59d160b000, com_data=0x7f5ad0efd1b0, command=COM_QUERY) at /flash11/daoke.wangc/PolarDB_80/sql/sql_parse.cc:1873
#14 0x000000000339cd7f in do_command(THD*, std::function<bool (THD*, COM_DATA const*, enum_server_command)>*) (thd=0x7f59d160b000, dispatcher=0x0)
 at /flash11/daoke.wangc/PolarDB_80/sql/sql_parse.cc:1335
#15 0x000000000339cf1d in do_command (thd=0x7f59d160b000) at /flash11/daoke.wangc/PolarDB_80/sql/sql_parse.cc:1372
#16 0x00000000035f4930 in handle_connection (arg=0x7f5ad6180ec0) at /flash11/daoke.wangc/PolarDB_80/sql/conn_handler/connection_handler_per_thread.cc:316
#17 0x000000000506e2f3 in pfs_spawn_thread (arg=0x7f5ad628aa20) at /flash11/daoke.wangc/PolarDB_80/storage/perfschema/pfs.cc:2879
#18 0x00007f5af4027e25 in start_thread () from /lib64/libpthread.so.0
#19 0x00007f5af291df1d in clone () from /lib64/libc.so.6
`

### test_quick_select 函数
test_quick_select用来根据范围选择索引是否有很快的代价最低的访问方式

步骤如下:

`0. prepare potential ranges scan index/keyparts
1. setup_range_conditions
 tree = get_mm_tree(&param, cond);
2. Fix the selectivity for SEL_ARGs by histogram
 fix_sel_tree_selectivity(&param, tree);
3. Try to construct a QUICK_GROUP_MIN_MAX_SELECT
 group_trp = get_best_group_min_max(&param, tree, &best_cost);
4. Try to construnct a QUICK_SKIP_SCAN_SELECT
 skip_scan_trp = get_best_skip_scan(&param, tree, force_skip_scan);
5. Get best 'range' plan and prepare data for making other plans
 range_trp = get_key_scans_params(&param, tree, false, true, &best_cost)
6. Get best non-covering ROR-intersection plan and prepare data for building covering ROR-intersection.
 rori_trp = get_best_ror_intersect(&param, tree, &best_cost, true)
7. Try creating index_merge/ROR-union scan.
 new_conj_trp = get_best_disjunct_quick(&param, imerge, &best_cost);
8. If we got a read plan, create a quick select from it.
 qck = best_trp->make_quick(&param, true)
`

### get_key_scans_params 函数
获取最佳的range扫描方式

`for (idx = 0; idx < param->keys; idx++) {
 key = tree->keys[idx];
 check_quick_select
 find best trp
}
`

### check_quick_select 函数
根据已知索引key，遍历对应SEL_ARG mm tree，计算所有range索引扫描的rows用来计算代价

`ha_innobase::multi_range_read_info_const
for every range in key RB tree
 get the next interval in the R-B tree
 rows += ha_innobase/ha_innopart::records_in_range
 n_ranges++
RANGE_SEQ_IF seq_if = {sel_arg_range_seq_init, sel_arg_range_seq_next, 0, 0};
MRR range sequence, SEL_ARG* implementation: SEL_ARG graph traversal context
 Consider a query with these range predicates:
 (kp0=1 and kp1=2 and kp2=3) or
 (kp0=1 and kp1=2 and kp2=4) or
 (kp0=1 and kp1=3 and kp2=5) or
 (kp0=1 and kp1=3 and kp2=6)

 1) sel_arg_range_seq_next() is called the first time
 - traverse the R-B tree (see SEL_ARG) to find the first range
 - returns range "1:2:3"
 - values in stack after this: stack[1, 1:2, 1:2:3]
 2) sel_arg_range_seq_next() is called second time
 - keypart 2 has another range, so the next range in
 keypart 2 is appended to stack[1] and saved
 in stack[2]
 - returns range "1:2:4"
 - values in stack after this: stack[1, 1:2, 1:2:4]
 3) sel_arg_range_seq_next() is called the third time
 - no more ranges in keypart 2, but keypart 1 has
 another range, so the next range in keypart 1 is
 appended to stack[0] and saved in stack[1]. The first
 range in keypart 2 is then appended to stack[1] and
 saved in stack[2]
 - returns range "1:3:5"
 - values in stack after this: stack[1, 1:3, 1:3:5]
 4) sel_arg_range_seq_next() is called the fourth time
 - keypart 2 has another range, see 2)
 - returns range "1:3:6"
 - values in stack after this: stack[1, 1:3, 1:3:6]
`

### Quick Read Plan 结构函数
```
/*
 Table rows retrieval plan. Range optimizer creates QUICK_SELECT_I-derived
 objects from table read plans.
 */
 class TABLE_READ_PLAN {
 ......
 }

/*
 Plan for a QUICK_RANGE_SELECT scan.
 TRP_RANGE::make_quick ignores retrieve_full_rows parameter because
 QUICK_RANGE_SELECT doesn't distinguish between 'index only' scans and full
 record retrieval scans.
 */

 class TRP_RANGE : public TABLE_READ_PLAN {
 ......
 SEL_ROOT *key;
 ......
 }

class TRP_ROR_INTERSECT : public TABLE_READ_PLAN {
 ......
}

 /*
 Plan for QUICK_ROR_UNION_SELECT scan.
 QUICK_ROR_UNION_SELECT always retrieves full rows, so retrieve_full_rows
 is ignored by make_quick.
 */

 class TRP_ROR_UNION : public TABLE_READ_PLAN {
 ......
 }

 /*
 Plan for QUICK_INDEX_MERGE_SELECT scan.
 QUICK_ROR_INTERSECT_SELECT always retrieves full rows, so retrieve_full_rows
 is ignored by make_quick.
 */

 class TRP_INDEX_MERGE : public TABLE_READ_PLAN {
 ......
 }

 /*
 Plan for a QUICK_GROUP_MIN_MAX_SELECT scan.
 */

 class TRP_GROUP_MIN_MAX : public TABLE_READ_PLAN {
 ......
 }

 /*
 Plan for a QUICK_SKIP_SCAN_SELECT scan.
 */

 class TRP_SKIP_SCAN : public TABLE_READ_PLAN {
 ......
 }

```

TRP_RANGE::make_quick()是根据最优的Range查询计划，执行范围快速查询确定执行计划，生成对应的quick实例，QEP->quick = TRP_RANGE::make_quick()。

## GDB分析工具
工具下载地址：https://github.com/cwang82566/mysql_debugging_tools

```
select * from tmp_sel_arg1 where (kp1=1 and kp2=2 and kp3=3) or        (kp1=1 and kp2=2 and kp3=4) or        (kp1=1 and kp2=3 and kp3=5) or        (kp1=1 and kp2=3 and kp3=6);
(gdb) my st tree
$c0 (SEL_TREE *) 0x7f7cf040acd0 [type=SEL_TREE::KEY,keys.m_size=2]
|--$c1 (SEL_ROOT *) 0x7f7cf040add0 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=3]
| `--$c2 (SEL_ARG *) 0x7f7cf040b800 [color=SEL_ARG::BLACK, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=0 '\000', selectivity=1]
| | field = $c3 (Item_field *) 0x7f7cf04c1508 field = test.tmp_sel_arg1.kp1
| | equal = [ $c4 (Item_int *) 0x7f7cf04af718 value = 2 ]
| |--$c6 (SEL_ARG *) 0x7f7cf040ad50 [color=SEL_ARG::RED, is_asc=true, minflag=4 '\004', maxflag=8 '\b', part=0 '\000', selectivity=1]
| | | field = $c7 (Item_field *) 0x7f7cf04c0728 field = test.tmp_sel_arg1.kp1
| | | scope = ( -infinity, $c8 (Item_int *) 0x7f7cf04ae760 value = 1 )
| | `--$c11 (SEL_ROOT *) 0x7f7cf040b3d0 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=1]
| | `--$c12 (SEL_ARG *) 0x7f7cf040b350 [color=SEL_ARG::BLACK, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=1 '\001', selectivity=1]
| | | field = $c13 (Item_field *) 0x7f7cf04c0aa0 field = test.tmp_sel_arg1.kp2
| | | equal = [ $c14 (Item_int *) 0x7f7cf04aea80 value = 5 ]
| | `--$c18 (SEL_ROOT *) 0x7f7cf040aef8 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=2]
| | `--$c19 (SEL_ARG *) 0x7f7cf040ae78 [color=SEL_ARG::BLACK, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=2 '\002', selectivity=1]
| | | field = $c20 (Item_field *) 0x7f7cf04c0e18 field = test.tmp_sel_arg1.kp3
| | | equal = [ $c21 (Item_int *) 0x7f7cf04aef48 value = 10 ]
| | `--$c24 (SEL_ARG *) 0x7f7cf040b040 [color=SEL_ARG::RED, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=2 '\002', selectivity=1]
| | | field = $c25 (Item_field *) 0x7f7cf04c1190 field = test.tmp_sel_arg1.kp3
| | | equal = [ $c26 (Item_int *) 0x7f7cf04af268 value = 12 ]
| |--$c30 (SEL_ARG *) 0x7f7cf04370c8 [color=SEL_ARG::RED, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=0 '\000', selectivity=1]
| | | field = $c31 (Item_field *) 0x7f7cf04c1f70 field = test.tmp_sel_arg1.kp1
| | | equal = [ $c32 (Item_int *) 0x7f7cf04b0520 value = 3 ]
| | `--$c36 (SEL_ROOT *) 0x7f7cf040b9a0 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=2]
| | `--$c37 (SEL_ARG *) 0x7f7cf040b920 [color=SEL_ARG::BLACK, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=2 '\002', selectivity=1]
| | | field = $c38 (Item_field *) 0x7f7cf04c22e8 field = test.tmp_sel_arg1.kp3
| | | equal = [ $c39 (Item_int *) 0x7f7cf04b0840 value = 11 ]
| | `--$c42 (SEL_ARG *) 0x7f7cf040bae8 [color=SEL_ARG::RED, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=2 '\002', selectivity=1]
| | | field = $c43 (Item_field *) 0x7f7cf04c2660 field = test.tmp_sel_arg1.kp3
| | | equal = [ $c44 (Item_decimal *) 0x7f7cf04b0b60 value = 14 ]
| `--$c48 (SEL_ROOT *) 0x7f7cf040b4f0 [type=SEL_ROOT::Type::KEY_RANGE, use_count=1, elements=2]
| `--$c49 (SEL_ARG *) 0x7f7cf040b470 [color=SEL_ARG::BLACK, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=2 '\002', selectivity=1]
| | field = $c50 (Item_field *) 0x7f7cf04c1880 field = test.tmp_sel_arg1.kp3
| | equal = [ $c51 (Item_int *) 0x7f7cf04afa38 value = 11 ]
| `--$c54 (SEL_ARG *) 0x7f7cf040b638 [color=SEL_ARG::RED, is_asc=true, minflag=0 '\000', maxflag=0 '\000', part=2 '\002', selectivity=1]
| | field = $c55 (Item_field *) 0x7f7cf04c1bf8 field = test.tmp_sel_arg1.kp3
| | equal = [ $c56 (Item_int *) 0x7f7cf04afd58 value = 14 ]
`--$c60 (SEL_ROOT *) 0x0 Non

```

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)