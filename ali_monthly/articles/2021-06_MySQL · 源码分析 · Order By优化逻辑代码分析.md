# MySQL · 源码分析 · Order By优化逻辑代码分析

**Date:** 2021/06
**Source:** http://mysql.taobao.org/monthly/2021/06/04/
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

 ## MySQL · 源码分析 · Order By优化逻辑代码分析 
 Author: xavier.zxm 

 ## 概述
近期在处理线上问题的时候遇到了一些关于Filesort的问题，在分析问题的过程中对代码进行了一些梳理，在此做一些总结，希望对同学们在学习mysql的代码过程中能有所帮助。以下分析基于Mysql-8.0.24代码。

## 优化阶段
涉及的函数：

`JOIN::optimize_distinct_group_order
JOIN::test_skip_sort
test_if_order_by_key
test_if_skip_sort_order
`

相关变量：

`JOIN::order
JOIN::simple_order
JOIN::skip_sort_order
`
同时Order会受index，group by， distinct，window的影响，例如

1. order是group list的前缀，且window和rollup不影响顺序时，order会被优化掉， 在group list上加上顺序要求
2. distinct查询，没有group list、window、sum函数，order都在投影中，会尝试是否可以使用索引来完成去重排序
3. group by操作可能会产生filesort，用来进行流式group by
4. distinct操作可能会通过filesort来完成
5. window function如果有partition/order by要求，会在window function的输入table上进行filesort

相关环境变量：
|变量名|默认值|描述|
|:-|:-|:-|
|max_length_for_sort_data|4096 [4, 8MB]| sorted records的最大字节数，用来决定filesort时是否是padon，8.0.18 之后已经不再使用|
|max_sort_length|1024 [4, 8MB]| 大对象（TEXT）， 如果是pad空格的字符集，参与排序的长度|
|sort_buffer_size|256KB [32KB, ULONG_MAX] | sort buffer的大小，每个排序线程一个|

### JOIN::test_skip_sort
检查是否可以通过索引来避免filesort的总入口，在JOIN::optimizer中调用

1. group list
2. 不是BIG_RESULT或JSON Aggregation， 是GROUP MIN MAX优化
 1. simple group并且不是distinct转化的group by
 call -> test_if_skip_sort_order
3. order list
4. simple order或skip sort order 且不是window sort
 call -> test_if_skip_sort_order

### test_if_skip_sort_order
检查是否可以通过使用索引来避免filesort
条件：

1. single row result, EQ_REF/CONST/SYSTEM， 可以skip
2. 如果order by的列是全文索引函数， 要求order by只能有这一个列， 且必须是降序，这时候是否能使用全文索引替代filesort取决于下面两个条件：
 
 表的访问方式是全文索引， 且使用的函数和order by的函数必须相同， 可以skip
3. 否则，没有单表条件，select limit是用户指定的且小于全文索引函数名中的行数， 可以skip

 order by必须是表列并且属于同一个索引，如果不是REF使用的索引，则计算可能的代价是否比REF更低，并尝试替换索引。 如果时REF使用的索引，检测是否可以满足Order的方向要求，如果可以满足，则省略filesort操作，由索引扫描保证顺序。

### test_if_order_by_key
检查是否可以通过索引来进行排序

1. order list都是表列
2. partition表必须是索引的全部列
3. 可以满足索引的顺序要求

## 执行阶段
涉及的函数：

`filesort()

`

### filesort
filesort执行逻辑总入口， 主要流程如下：

1. 初始化排序参数， 确定是否可以使用addon，避免回表，涉及的函数包括：
 `Sort_param::decide_addon_fields
Sort_param::try_to_pack_addons
Filesort::get_addon_fields
`
2. 确定是否可以使用优先级队列排序避免排序过程中落盘（写tempfile）, 函数为：
 ```
check_if_pq_applicable

```
3. 初始化优先级队列（初始化tempfile）
4. 读取数据并排序（tempfile需要多轮merge），排序完成后如果是rowid，需要回表
5. 去重和limit会在排序过程中应用，用来减少中间排序的数据量

filesort主要代码：

`bool filesort() {

 param->init_for_filesort(); // init filesort parameter

 if (check_if_pq_applicable()) { // check using priority queue
 pq.init() // init priority queue
 }
 
 read_all_rows() // read all rows from sorted table
 
 if (num_trunk == 0) {
 save_index() // save sorted result to buffer, deal rowid
 } else {
 merge_many_buffer() // merge trunks
 merge_index() // merge last 15 trunks
 }
}

`

**判断Addon的逻辑** 

根据排序列和表需要返回的列，判断排序方式是addon还是rowid，并计算sort record length，filesort认为回表代价总是更大的，会尽量采用addon的方式， 8.0.24不再额外单独限制max_sort_length, 但是新引入了一个限制，如果pack_field导致额外需要存储的内容超过10字节（描述变长列的变量0）， 则不采用addon方式。 其他不能addon的条件有：

1. 全文索引过滤后的filesort，只能使用rowid
2. force_sort_position，只能使用rowid
3. 包含blob列，且pack长度大于70000，只能使用rowid

**check_if_pq_applicable, 判断是否可以使用优先级队列排序**

1. 需要有limit
2. 不能是去重
3. limit的值需要小于UINT_MAX - 2（priority queue的最大容量）
4. 最大行长度需要小于0xFFFFFFFF（不能是无边界的行）
5. 排序的数据需要在sort buffer中可以放下
6. limit rows小于排序总数据量的1/3 (认为优先级队列比filesort只有1个trunk的代价更大)
7. 排序的总数据量放不下时，limit的数据量需要能在sort buffer中放下

**优先级队列排序过程**

1. 初始化优先级队列，大小为排序行数 + 2
2. 读取表的全部数据，并放入优先级队列(内存中）

**filesort排序过程**

1. 初始化tempfile
2. 读取表的全部数据，每次读取一行，构建sortkey后copy进入trunk， trunk满后对trunk的数据进行快排，如果数据不足一个trunk，则返回， 保留数据在内存中，否责trunk数据写入tempfile，知道全部数据读取完
3. 构建另一个trempfile， 两个tempfile来回倒， 做trunk合并，直到trunk数量少15个， 合并中使用的是merge sort，每次合并7个trunk
4. 对最后不足15个trunk通过priority queue进行合并
5. 排序后的数据保存在tempfile中

**返回结果**

1. 如果是addon，排序后直接返回结果
2. 如果是rowid，排序后会通过rowid读取记录后返回

**去重操作**
merge trunk的时候，保留上一个sort key，如果sort key不发生变化，则是重复数据跳过，否则是新的数据。更新last_sort_key

**limit操作**
limit作用于filesort的方式

1. prioriyty
2. 最多只放limit数量的rows
3. 通过rowid读取record时，最多只读区limit行的数据
4. filesort
5. trunk在写入文件时，如果limit > trunk中的rows， 只写limit数量的行
6. 合并trunk时，只需要从待合并的trunk中读取limit的行数
7. 通过rowid读取record时，最多只读区limit行的数据

## Order By的并行
Parallel Query提供了Order By的并行执行方式，可能的执行方式包括：

1. Serail, 在leader上串行执行
2. Two Phase Gather， worker上先并行排序， 然后在Leader上进行Merge排序
具体执行方式取决于计划的代价。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)