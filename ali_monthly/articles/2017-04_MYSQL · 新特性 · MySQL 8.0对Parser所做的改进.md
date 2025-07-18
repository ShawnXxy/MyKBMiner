# MYSQL · 新特性 · MySQL 8.0对Parser所做的改进

**Date:** 2017/04
**Source:** http://mysql.taobao.org/monthly/2017/04/02/
**Images:** 4 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 04
 ](/monthly/2017/04)

 * 当期文章

 MySQL · 源码分析 · MySQL 半同步复制数据一致性分析
* MYSQL · 新特性 · MySQL 8.0对Parser所做的改进
* MySQL · 引擎介绍 · Sphinx源码剖析（二）
* PgSQL · 特性分析 · checkpoint机制浅析
* MySQL · 特性分析 · common table expression
* PgSQL · 应用案例 · 逻辑订阅给业务架构带来了什么？
* MSSQL · 应用案例 · 基于内存优化表的列存储索引分析Web Access Log
* TokuDB · 捉虫动态 · MRR 导致查询失败
* HybridDB · 稳定性 · HybridDB如何优雅的处理Out Of Memery问题
* MySQL · 捉虫动态 · 5.7 mysql_upgrade 元数据锁等待

 ## MYSQL · 新特性 · MySQL 8.0对Parser所做的改进 
 Author: 令猴 

 ## 背景介绍
众所周知，MySQL Parser是利用C/C++实现的开源yacc/lex组合，也就是 GNU bison/flex。Flex负责生成tokens， Bison负责语法解析。开始介绍MySQL 8.0的新特新之前，我们先简单了解一下通用的两种Parser。一种是Bottom-up parser，另外一种是Top-down parser。

## Bottom-up parser
Bottom-up解析是从parse tree底层开始向上构造，然后将每个token移进（shift），进而规约（reduce）为较大的token，最终按照语法规则的定义将所有token规约（reduce）成为一个token。移进过程是有先后顺序的，如果按照某种顺序不能将所有token规约为一个token，解析器将会回溯重新选定规约顺序。如果在规约（reduce）的过程中出现了既可以移进生成一个新的token，也可以规约为一个token，这种情况就是我们通常所说的shift/reduce conflicts.

## Top-down parser
Top-down解析是从parse tree的顶层开始向下构造历。这种解析的方法是假定输入的解析字符串是符合当前定义的语法规则，按照规则的定义自顶开始逐渐向下遍历。遍历的过程中如果出现了不满足语法内部的逻辑定义，解析器就会报出语法错误。

如果愿意详细了解这两种parser的却别，可以参考https://qntm.org/top。

## MySQL8.0对parser所做的改进
Bison是一个bottom-up的parser。但是由于历史原因，MySQL的语法输入是按照Top-down的方式来书写的。这样的方式导致MySQL的parser语法上有包含了很多的reduce/shift conflicts；另外由于一些空的或者冗余的规则定义也使得的MySQL parser越来越复杂。为了应对未来越来越多的语法规则，以及优化MySQL parser的解析性能，MySQL 8.0对MySQL parser做了非常大的改进。当前的MySQL 8.0.1 Milestone release的代码中对于Parser的改进仍未全部完成，还有几个相关的worklog在继续。

改进之后，MySQL parser可以达到如下状态：

1. MySQL parser将会成为一个不涉及状态信息（即：不包含执行状态的上下文信息）的bottom-up parser；
2. 减少parse tree上的中间节点，减少冗余规则
3. 更少的reduce/shift conflicts
4. 语法解析阶段，只包含以下简单操作：
 * 创建parse tree node
* 返回解析的最终状态信息
* 有限的访问系统变量
5. MySQL parser执行流程将会由

SQL input -> lex. scanner -> parser -> AST (SELECT_LEX, Items etc) -> executor

变成

SQL input -> lex. scanner -> parser -> parse tree -> AST -> executor

下面我们通过看一个MySQL 8.0 中对SELECT statement所做的修改来看一下MySQL parser的改进。

SELECT statement可以说是MySQL中用处非常广泛的一个语句，比如CREATE VIEW, SELECT, CREATE TABLE, UNION, SUBQUERY等操作。 通过下图我们看一下MySQL8.0之前的版本是如何支持这些语法规则的。
![5.7-select.png](.img/c8f0fd77db0b_242b8cea4fe0a44f09eb7a7ac5e4fec3.png)

MySQL8.0中对于这些语法规则的支持如下图：
![select-8.0.png](.img/0f0733e49432_360d921151c34a4dc893ec6920d9a3ac.png)

通过如上两个图的对比，显然MySQL8.0的parser清爽了许多。当然我们也清晰的看到MySQL8.0中对于MySQL parser所做的改进。相同的语法规则只有一处定义，消除了过去版本中按照top-down方式书写的冗余语法定义。当然通过这样的简化也可以看到实际的效果， shift/reduce conflicts也减少了很多：
![conflicts.png](.img/2b883f23c6bc_9820b146bbfa5a61a67e0722d3f1a8e7.png)

下面我们看看MySQL 8.0是如何将所有的SELECT statement操作定义为一个Query specification，并为所有其他操作所引用的：

Parse tree上所有的node都定义为Parse_tree_node的子类。Parse_tree_node的结构体定义如下：

`typedef Parse_tree_node_tmpl<Parse_context> Parse_tree_node; 
template<typename Context>
class Parse_tree_node_tmpl
{
...
private:
 /*
 False right after the node allocation. The contextualize/contextualize_
 function turns it into true.
 */
#ifndef DBUG_OFF
 bool contextualized;
#endif//DBUG_OFF
 /*
 这个变量是由于当前仍旧有未完成的相关worklog，parser的refactor还没有彻底完成。当前的parser中还有一部分上下文依赖的关系没有独立出来。
 等到整个parse refactor完成之后该变量就会被移除。
 */
 bool transitional; 
public:
 /*
 Memory allocation operator are overloaded to use mandatory MEM_ROOT
 parameter for cheap thread-local allocation.
 Note: We don't process memory allocation errors in refactored semantic
 actions: we defer OOM error processing like other error parse errors and
 process them all at the contextualization stage of the resulting parse
 tree.
 */
 static void *operator new(size_t size, MEM_ROOT *mem_root) throw ()
 { return alloc_root(mem_root, size); }
 static void operator delete(void *ptr,size_t size) { TRASH(ptr, size); }
 static void operator delete(void *ptr, MEM_ROOT *mem_root) {}

protected:
 Parse_tree_node()
 {
#ifndef DBUG_OFF
 contextualized= false;
 transitional= false;
#endif//DBUG_OFF
 }

public:
 ...

 /*
 True if contextualize/contextualized function has done:
 */
#ifndef DBUG_OFF
 bool is_contextualized() const { return contextualized; }
#endif//DBUG_OFF

 /*
 这个函数是需要被所有子类继承的，所有子类需要定义属于自己的上下文环境。通过调用子类的重载函数，进而初始化每个Parse tree node。
 */
 virtual bool contextualize(THD *thd);

 /**
 my_parse_error() function replacement for deferred reporting of parse
 errors

 @param thd current THD
 @param pos location of the error in lexical scanner buffers
 */
 void error(THD *thd) const;
};

`

当前MySQL8.0的源码中执行流程为：

`mysql_parse
|
parse_sql
|
MYSQLparse
|
Parse_tree_node::contextualize() /* 经过Bison进行语法解析之后生成相应的Parse tree node。然后调用contextualize对Parse tree node进行上下文初始化。
 初始化上下文后形成一个AST(Abstract Syntax Tree)节点。*/
`
接下来我们以SELECT statement来看一下PT_SELECT_STMT::contexualize()做些什么工作：

`class PT_select_stmt : public Parse_tree_node
{
 bool contextualize(Parse_context *pc)
 {
 // 这里初始化Parse_tree_node
 if (super::contextualize(pc))
 return true;

 pc->thd->lex->sql_command= m_sql_command;

 // 调用PT_query_specification来进行上下文初始化
 return m_qe->contextualize(pc) ||
 contextualize_safe(pc, m_into);
 }
private:
 PT_query_expression *m_qe；//通过m_qe来引用query_expression
}

class PT_query_expression : public Parse_tree_node
{
 ...
 bool contextualize(Parse_context *pc)
 {
 // 判断是否需要独立的名空间
 pc->select->set_braces(m_parentheses || pc->select->braces);
 m_body->set_containing_qe(this);
 if (Parse_tree_node::contextualize(pc) ||
 // 初始化SELECT主体上下文
 m_body->contextualize(pc))
 return true;
 // 这里会初始化ORDER, LIMIT子句
 if (!contextualized && contextualize_order_and_limit(pc))
 return true;

 // 这里会对SELECT表达式里包含的存储过程或者UDF继续进行上下文初始化
 if (contextualize_safe(pc, m_procedure_analyse))
 return true;

 if (m_procedure_analyse && pc->select->master_unit()->outer_select() != NULL)
 my_error(ER_WRONG_USAGE, MYF(0), "PROCEDURE", "subquery");

 if (m_lock_type.is_set && !pc->thd->lex->is_explain())
 {
 pc->select->set_lock_for_tables(m_lock_type.lock_type);
 pc->thd->lex->safe_to_cache_query= m_lock_type.is_safe_to_cache_query;
 }
 }
 ...
private： 
 bool contextualized;
 PT_query_expression_body *m_body; /* 这个类包含了SELECT语句的主要部分，select_list, FROM, GROUP BY, HINTs等子句。
 这里m_body变量其实是PT_query_expression_body的子类 PT_query_expression_body_primary */
 PT_order *m_order; // ORDER BY node
 PT_limit_clause *m_limit; // LIMIT node
 PT_procedure_analyse *m_procedure_analyse; //存储过程相关
 Default_constructible_locking_clause m_lock_type;
 bool m_parentheses;

}

class PT_query_expression_body_primary : public PT_query_expression_body
{
 {
 if (PT_query_expression_body::contextualize(pc) ||
 m_query_primary->contextualize(pc))
 return true;
 return false;
 }
private：
 PT_query_primary *m_query_primary; // 这里是SELECT表达式的定义类PT_query_specification的父类
}

// PT_query_specification是SELECT表达式的定义类，它定义了SELECT表达式中绝大部分子句
class PT_query_specification : public PT_query_primary
{
 typedef PT_query_primary super;
private:
 PT_hint_list *opt_hints;
 Query_options options;
 PT_item_list *item_list;
 PT_into_destination *opt_into1;
 Mem_root_array_YY<PT_table_reference *> from_clause; // empty list for DUAL
 Item *opt_where_clause;
 PT_group *opt_group_clause;
 Item *opt_having_clause;

bool PT_query_specification::contextualize(Parse_context *pc)
{
 if (super::contextualize(pc))
 return true;

 pc->select->parsing_place= CTX_SELECT_LIST;

 if (options.query_spec_options & SELECT_HIGH_PRIORITY)
 {
 Yacc_state *yyps= &pc->thd->m_parser_state->m_yacc;
 yyps->m_lock_type= TL_READ_HIGH_PRIORITY;
 yyps->m_mdl_type= MDL_SHARED_READ;
 } 
 if (options.save_to(pc))
 return true;
 
 // 这里开始初始化SELECT list项
 if (item_list->contextualize(pc))
 return true;
 // Ensure we're resetting parsing place of the right select
 DBUG_ASSERT(pc->select->parsing_place == CTX_SELECT_LIST);
 pc->select->parsing_place= CTX_NONE;

 // 初始化SELECT INTO子句
 if (contextualize_safe(pc, opt_into1))
 return true;

 // 初始化FROM子句
 if (!from_clause.empty())
 {
 if (contextualize_array(pc, &from_clause))
 return true;
 pc->select->context.table_list=
 pc->select->context.first_name_resolution_table=
 pc->select->table_list.first;
 }

 // 初始化WHERE条件
 if (itemize_safe(pc, &opt_where_clause) ||
 // 初始化GROUP子句 
 contextualize_safe(pc, opt_group_clause) ||
 // 初始化HAVING子句
 itemize_safe(pc, &opt_having_clause))
 return true;

 pc->select->set_where_cond(opt_where_clause);
 pc->select->set_having_cond(opt_having_clause);

 // 初始化HINTs
 if (opt_hints != NULL)
 {
 if (pc->thd->lex->sql_command == SQLCOM_CREATE_VIEW)
 { // Currently this also affects ALTER VIEW.
 push_warning_printf(pc->thd, Sql_condition::SL_WARNING,
 ER_WARN_UNSUPPORTED_HINT,
 ER_THD(pc->thd, ER_WARN_UNSUPPORTED_HINT),
 "CREATE or ALTER VIEW");
 }
 else if (opt_hints->contextualize(pc))
 return true;
 }
 return false;
}
`

综上我们以SELECT statement为例对MySQL8.0在MySQL parser方面所做的改进进行了简单介绍。这样的改进对于MySQL parser也许是一小步，但对于MySQL未来的可扩展确实是迈出了一大步。Parse tree独立出来，通过Parse tree再来构建AST，这样的方式下将简化MySQL对于Parse tree的操作，最大的受益者就是Prepared statement。等到MySQL parse的所有worklog完成之后，MySQL用户期盼多年的global prepared statement也就顺其自然实现了。

当然MySQL parser的改进让我们已经看到Oracle MySQL在对MySQL optimizier方面对于PARSER，optimizer， executor三个阶段的松解耦工作已经展开了。未来期待Optimizer生成的plan也可以像当前的parser一样成为一个纯粹的Plan，执行上下文与Plan也可以独立开来。只有到了executor阶段才生成相应的执行上下文。这样一来对于MySQL optimizer未来的可扩展势必会起到如虎添翼的作用。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)