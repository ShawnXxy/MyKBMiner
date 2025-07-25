# MySQL · 源码分析·  跟着MySQL 8.0 学 C++：scope_guard

**Date:** 2021/10
**Source:** http://mysql.taobao.org/monthly/2021/10/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2021 / 10
 ](/monthly/2021/10)

 * 当期文章

 MySQL · 引擎特性 · 庖丁解InnoDB之UNDO LOG
* 数据库系统 · 事物并发控制 · Two-phase Lock Protocol
* MySQL · 源码分析 · BLOB字段UPDATE流程分析
* MySQL · 源码分析· 跟着MySQL 8.0 学 C++：scope_guard
* MySQL · 源码分析 · CSV 引擎详解

 ## MySQL · 源码分析· 跟着MySQL 8.0 学 C++：scope_guard 
 Author: 甄平 

 ## 背景简介

 MySQL source code now permits and uses C++11 features. —- MySQL 8.0.0 (2016-09-12）
MySQL now can be compiled using C++14. —- MySQL 8.0.16 (2019-04-25)
MySQL now can be compiled using C++17. —- MySQL 8.0.27 (2021-10-19)

MySQL 8.0近段时间GA了8.0.17版本，正式支持了C++17的编译，很高兴看到官方开始逐步抛弃5.x时代的老包袱，在代码风格上开始拥抱一些新的东西。数据库作为一个复杂系统，依赖大量的协作开发和软件工程管理，代码的可读性和可维护性至关重要。MySQL 8.0在这方面向前一步，包括重构代码、使用STL容器、优化宏定义、std::thread等。本文向大家介绍一下8.0中的scope_guard功能。

## scope_guard是什么
Scope_guard顾名思义，针对某个scope的一个guard。假设我们有如下一段代码逻辑：

`if (⟨action⟩) {
 if (!⟨next⟩) {
 ⟨rollback⟩
 }
 ⟨cleanup⟩
}
`
以上代码非常健壮，包含了错误处理（rollback）和资源清理（cleanup）。不过缺点老生常谈，如果嵌套更多if条件，代码会变得臃肿。一般解法是用RAII，RAII自动管理cleanup，并用try-catch或类似方法统一处理rollback，来规避多层嵌套：

`class RAII {
 RAII() { ⟨action⟩ }
 ~RAII() { ⟨cleanup⟩ }
};
...
RAII raii;
try {
 ⟨next⟩
} catch (...) {
 ⟨rollback⟩
 throw;
}
`
我们的需求是，如果⟨next⟩出错，调用⟨rollback⟩，当任务结束，调用⟨cleanup⟩。scope_guard是一种轻量级的RAII，实现同样功能的伪代码如下，是不是简洁很多：

`⟨action⟩
auto g1 = scopeGuard([] { ⟨cleanup⟩ });
auto g2 = scopeGuard([] { ⟨rollback⟩ });
⟨next⟩
g2.dismiss();
`

## MySQL中的scope_guard
MySQL里面scope_guard的代码很简单，以下摘抄自8.0源码中的include/scope_guard.h。

`template <typename TLambda>
class Scope_guard {
 public:
 Scope_guard(const TLambda &rollback_lambda)
 : m_committed(false), m_rollback_lambda(rollback_lambda) {}
 Scope_guard(const Scope_guard<TLambda> &) = delete;
 Scope_guard(Scope_guard<TLambda> &&moved)
 : m_committed(moved.m_committed),
 m_rollback_lambda(moved.m_rollback_lambda) {
 moved.m_committed = true;
 }
 ~Scope_guard() {
 if (!m_committed) {
 m_rollback_lambda();
 }
 }

 inline void commit() { m_committed = true; }

 inline void rollback() {
 if (!m_committed) {
 m_rollback_lambda();
 m_committed = true;
 }
 }

 private:
 bool m_committed;
 const TLambda m_rollback_lambda;
};

template <typename TLambda>
Scope_guard<TLambda> create_scope_guard(const TLambda rollback_lambda) {
 return Scope_guard<TLambda>(rollback_lambda);
}
`
简单分析下这段代码。最下面的函数create_scope_guard是个模板函数，接受const TLambda类型的参数，创建一个Scope_guard对象。Scope_guard类中的m_committed，控制m_rollback_lambda在生命周期内最多允许调用一次。从析构函数和rollback()中可以看出，TLambda类型需要实现operator()，因此应该是一个function或者functor。Scope_guard的行为是除了显式调用commit()，最终在生命周期结束之前会执行一次rollback_lambda。commit的功能类似于前一节伪代码的dismiss，允许某些逻辑放弃lambda的回调。此外create_scope_guard函数本身就包括Scope_guard对象的一个作用域，因此Scope_guard类中实现move构造函数就显得非常必要。

### 案例一：资源管理
那么Scope_guard具体有什么用，我们可以从MySQL里面看出一些端倪。第一个典型的场景在sql/xa.cc中的find_trn_for_recover_and_check_its_state函数。这个函数比较短，我删除了一些无关的debug代码和注释，剩余摘录如下：

`static std::shared_ptr<Transaction_ctx>
find_trn_for_recover_and_check_its_state(THD *thd,
 xid_t *xid_for_trn_in_recover,
 XID_STATE *xid_state) {
 if (!xid_state->has_state(XID_STATE::XA_NOTR)) {
 my_error(ER_XAER_RMFAIL, MYF(0), xid_state->state_name());
 return nullptr;
 }

 mysql_mutex_lock(&LOCK_transaction_cache);
 auto grd =
 create_scope_guard([]() { mysql_mutex_unlock(&LOCK_transaction_cache); });

 auto foundit = transaction_cache.find(to_string(*xid_for_trn_in_recover));
 if (foundit == transaction_cache.end()) {
 my_error(ER_XAER_NOTA, MYF(0));
 return nullptr;
 }

 const XID_STATE *xs = foundit->second->xid_state();
 if (!xs->get_xid()->eq(xid_for_trn_in_recover) || !xs->is_in_recovery()) {
 my_error(ER_XAER_NOTA, MYF(0));
 return nullptr;
 }
 if (thd->in_active_multi_stmt_transaction()) {
 my_error(ER_XAER_RMFAIL, MYF(0), xid_state->state_name());
 return nullptr;
 }

 return foundit->second;
}
`
互斥锁LOCK_transaction_cache用来保护transaction_cache的并发访问。原则上，lock执行后，代码最后4处return，都应该调用unlock释放锁。通过给create_scope_guard传一个带有unlock逻辑的lambda表达式，借助Scope_guard的析构函数，锁释放就被自动处理了。由于MySQL中的mutex都是采用内部的mysql_mutex_t类型，跨平台且集成了performance_schema的性能诊断，无法直接使用C++标准库中的std::lock_guard，为mysql_mutex_t单独实现RAII又涉及面太广，create_scope_guard就是这里的瑞士军刀。

### 案例二：错误处理
另一个样例来源于sql/sql_tmp_table.cc中的create_tmp_table函数。该函数超级长，简化逻辑如下：

`TABLE *create_tmp_table(THD *thd, Temp_table_param *param,
 const mem_root_deque<Item *> &fields, ORDER *group,
 bool distinct, bool save_sum_fields,
 ulonglong select_options, ha_rows rows_limit,
 const char *table_alias) {
 ... // skip 73 lines of code

 table->init_tmp_table(thd, share, &own_root, param->table_charset,
 table_alias, reg_field, blob_field, false);

 auto free_tmp_table_guard = create_scope_guard([table] {
 close_tmp_table(table);
 free_tmp_table(table);
 });

 ... // skip 641 lines of code

 free_tmp_table_guard.commit();

 return table;
}
`
这段代码是Scope_guard在错误处理方面的典型应用。代码需求是，当函数异常退出的时候，执行close_tmp_table和free_tmp_table的回滚操作；如果函数成功执行，则直接返回这个table对象。MySQL的很多代码由于历史演进背景，以及基础软件不可逃避的复杂性本质，很多函数称得上是“又臭又长”。我统计了一下，create_tmp_table这个函数总共有732行，在init_tmp_table和最后的return table中间，竟然有16个return nullptr的异常逻辑。感谢Scope_guard，否则这16个异常逻辑都要补上回滚操作的两行代码。更不用提未来会有其他开发者在中间的600多行中添加了新的错误处理逻辑，一不小心很容易出错。

### 案例三：状态维护
MySQL的InnoDB存储引擎代码也有Scope_guard的应用，不过在类定义上做了一些改动：

`class bool_scope_guard_t {
 bool *m_active;
 public:
 explicit bool_scope_guard_t(bool &active) : m_active(&active) {
 *m_active = true;
 }
 ~bool_scope_guard_t() {
 if (m_active != nullptr) {
 *m_active = false;
 m_active = nullptr;
 }
 }
 bool_scope_guard_t(bool_scope_guard_t const &) = delete;
 bool_scope_guard_t &operator=(bool_scope_guard_t const &) = delete;
 bool_scope_guard_t &operator=(bool_scope_guard_t &&) = delete;
 bool_scope_guard_t(bool_scope_guard_t &&old) {
 m_active = old.m_active;
 old.m_active = nullptr;
 }
};
`
bool_scope_guard_t确保了一个bool类型的变量在某段作用域内始终是true（不过其实bool_scope_guard_t也可以复用Scope_guard那个大类）。这个类使用在如下代码中：

`struct row_prebuilt_t {
 ... // skip some code

 private:
 /** Set to true iff we are inside read_range_first() or read_range_next() */
 bool m_is_reading_range;

 public:
 bool is_reading_range() const { return m_is_reading_range; }

 class row_is_reading_range_guard_t : private ut::bool_scope_guard_t {
 public:
 explicit row_is_reading_range_guard_t(row_prebuilt_t &prebuilt)
 : ut::bool_scope_guard_t(prebuilt.m_is_reading_range) {}
 };

 row_is_reading_range_guard_t get_is_reading_range_guard() {
 return row_is_reading_range_guard_t(*this);
 }
}

int ha_innobase::read_range_first(const key_range *start_key,
 const key_range *end_key, bool eq_range_arg,
 bool sorted) {
 auto guard = m_prebuilt->get_is_reading_range_guard();
 return handler::read_range_first(start_key, end_key, eq_range_arg, sorted);
}

int ha_innobase::read_range_next() {
 auto guard = m_prebuilt->get_is_reading_range_guard();
 return (handler::read_range_next());
}
`
row_prebuilt_t这个struct包含m_is_reading_range标记位，is_reading_range()可以判断当前是否在执行read_range_first或read_range_next。对应的这两个ha_innobase接口通过get_is_reading_range_guard获取private继承自bool_scope_guard_t的row_is_reading_range_guard_t，来管理m_is_reading_range的状态。至于is_reading_range具体到InnoDB中是用来做什么的，这其实是8.0修复的一个历史bug（Bug #29508068）：对于SELECT…FOR UPDATE在PK/UK范围扫描遍历的过程中，会一直加next key lock包括第一个不满足条件的记录，相当于在最后一条记录上多加了没必要的锁。这段代码是这个bug修复的一部分。

对于这个“状态维护”的应用场景，我举一个更直观的例子。很多系统都会有后台GC的任务，比如单独起一个线程每隔几秒调用一次gc_work()。假设我需要一个监控项gc_running标记当前是否在GC过程中，可以这么简化实现：

`// garbage collection
bool gc_running;
void gc_work() {
 gc_running = true;
 auto guard = create_scope_guard([]() { gc_running = false; });
 ... // do something
 return;
}
// invoke gc_work() periodically
`
​

## scope_guard的小扩展
MySQL中的Scope_guard算是一个初级版，如果想更适配C++11风格的话，可以用另一种写法（来自参考资料[1]）：

`template<typename Fun>
class ScopeGuard {
 Fun f_;
 bool active_;
 public:
 ScopeGuard(Fun f) : f_(std::move(f)), active_(true) {}
 ~ScopeGuard() { if (active_) f_(); }
 void dismiss() { active_ = false; }

 ScopeGuard() = delete;
 ScopeGuard(const ScopeGuard &) = delete;
 ScopeGuard& operator=(const ScopeGuard &) = delete;
 ScopeGuard(ScopeGuard&&rhs) : f_(std::move(rhs.f_)), active_(rhs.active_) {
 rhs.dismiss();
 }
};

template<typename Fun>
ScopeGuard<Fun> scopeGuard(Fun f) {
 return ScopeGuard<Fun>(std::move(f));
}

namespace detail {
 enum class ScopeGuardOnExit {};
 template<typename Fun>
 ScopeGuard<Fun> operator+(ScopeGuardOnExit, Fun&& fn)
 {
 return ScopeGuard<Fun>(std::forward < Fun > (fn));
 }
}

// Helper macro
#define SCOPE_EXIT \
auto ANONYMOUS_VARIABLE(SCOPE_EXIT_STATE) =::detail::ScopeGuardOnExit()+[&]()

#define CONCATENATE_IMPL(s1,s2) s1##s2
#define CONCATENATE(s1,s2) CONCATENATE_IMPL(s1,s2)

#ifdef __COUNTER__
#define ANONYMOUS_VARIABLE(str) CONCATENATE(str,__COUNTER__)
#else
#define ANONYMOUS_VARIABLE(str) CONCATENATE(str,__LINE__)
#endif
`
最上面主要的Class和MySQL中的类似，忽略变量名的差异，改动有两块：1. 模板参数从const引用变成std::move传进来，减少不必要的对象销毁；2. 通过=delete的语法禁用默认构造函数、copy构造函数和copy赋值操作，防止误用。之前MySQL的版本，创建scope_guard需要调用create_scope_guard返回一个变量，即使之后代码完全不再用到，也得专门起个名字，是比较麻烦的。新的代码新加了一些宏定义的黑科技来解决这个问题，如果要读懂的话，你要了解宏定义展开、宏定义中的##语法、operator+操作符重载、编译器的预定义宏`__COUNTER__`等知识，大家可以自行分析。

简化后的一种代码样例如下：

`void fun() {
 char name[] = "/tmp/test.xxx";
 auto fd = mkstemp(name);
 SCOPE_EXIT { fclose(fd); unlink(name); };
 auto buf = malloc(1024 * 1024);
 SCOPE_EXIT { free(buf); };

 ... use fd and buf ...
}
`

## 总结
本质上来讲，scope_guard相当于把很多工作交给了编译器，如错误处理的延迟回调、资源释放、提供RAII。对此还有一个新的概念叫Declarative Control Flow。如果想看更丰富的ScopeGuard写法可以参考[2][3]。如何借助scope_guard提高复杂代码的可维护性，是个很有趣的问题，感兴趣的朋友可以从参考资料[4][5]中找到更多的灵感。

## 参考资料
[1] ScopeGuard C++11基础版 [https://github.com/joker-eph/ScopeGuard11/blob/master/ScopeGuard.hpp](https://github.com/joker-eph/ScopeGuard11/blob/master/ScopeGuard.hpp) 

[2] ScopeGuard C++11完整版 [https://github.com/Neargye/scope_guard](https://github.com/Neargye/scope_guard) 

[3] ScopeGuard folly版 [https://github.com/facebook/folly/blob/main/folly/ScopeGuard.h](https://github.com/facebook/folly/blob/main/folly/ScopeGuard.h) 

[4] C++ and Beyond 2012: Andrei Alexandrescu - [Systematic Error Handling in C++](https://channel9.msdn.com/Shows/Going+Deep/C-and-Beyond-2012-Andrei-Alexandrescu-Systematic-Error-Handling-in-C) 

[5] CppCon 2015: Andrei Alexandrescu “[Declarative Control Flow](https://www.youtube.com/watch?v=WjTrfoiB0MQ&ab_channel=CppCon)”

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)