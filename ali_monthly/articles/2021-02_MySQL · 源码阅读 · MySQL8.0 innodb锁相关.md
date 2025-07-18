# MySQL · 源码阅读 · MySQL8.0 innodb锁相关

**Date:** 2021/02
**Source:** http://mysql.taobao.org/monthly/2021/02/08/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2021 / 02
 ](/monthly/2021/02)

 * 当期文章

 PolarDB · 特性分析 · Explain Format Tree 详解
* MySQL · 源码阅读 · InnoDB Export/Import Tablespace解析
* MySQL · 源码解析 · MySQL 8.0.23 Hypergraph Join Optimizer代码详解
* MySQL · 性能优化 · InnoDB 事务 sharded 锁系统优化
* DataBase · 社区动态 · 数据库中的表达式
* MySQL · 源码分析 · Group by优化逻辑代码分析
* MySQL · 源码阅读 · X-plugin的传输协议
* MySQL · 源码阅读 · MySQL8.0 innodb锁相关
* PolarDB · 优化改进 · 使用窗口聚合函数来将子查询解关联

 ## MySQL · 源码阅读 · MySQL8.0 innodb锁相关 
 Author: yixiong 

 ## 背景
innodb里面的mutex常见的实现是PolicyMutex<TTASEventMutex，信号量底层使用是os_event_t

## 代码分析

#### os_event_t
`struct os_event {
void set();
int64_t reset();
void wait_low();
void broadcast();
private:
bool m_set;
int64_t signal_count;
os_cond_t cond_var;
EventMutex mutex;
os_cond_t cond_var;
}
`

1. set函数 如果m_set是false，调用broadcast函数
2. reset函数 设置m_set = false, 返回signal_count
3. wait_low函数 m_set == false 并且signal_count == reset_sig_count 才进入wait, 保证不会死锁
4. broadcast函数 m_set设置true, signal_count计数器+1，唤醒其他等待者

wait操作: 先调用reset函数，然后用返回的reset_sig_count作为参数，调用wait_low函数 

signal操作: 调用set函数

#### PolicyMutex
先看PolicyMutex的主要结构

`template <typename MutexImpl>
struct PolicyMutex {
private:
MutexImpl m_impl;
public:
void enter();
void exit();
void init();
}
`
init函数负责初始化，加锁是enter函数，解锁是exit函数，具体的实现都是通过m_impl来实现

#### TTASEventMutex
在看TTASEventMutex的主要结构

`struct TTASEventMutex {
public:
void init();
void exit();
void enter();
bool try_lock();
private:
std::atomic_bool m_lock_word;
std::atomic_bool m_waiters;
os_event_t m_event;
MutexPolicy m_policy;
}
`
1. m_lock_word 判断是否加锁成功
2. m_waiters 当前锁有多少个等待者
3. m_event 线程等待内部实现使用
4. m_policy 统计使用

exit函数

1. m_lock_word设置false
2. 如果有waiter就调用signal函数唤醒其他等待者

enter函数功能就是加锁，成功返回，否则就一直等待，具体内部实现：

1. 第一步会优先判断m_lock_word原子变量cas操作能否成功，成功就说明加锁成功了，否则说明有其他人持有锁
2. 调用spin_and_try_lock函数，内部实现死循环执行下面步骤： 

 2.1. 先尝试max_spins次对m_lock_word变量执行cas操作，如果成功就返回 

 2.2. 没有成功就先尝试执行yield函数，放弃cpu占用 

 2.3. 调用wait函数，内部实现：

 ` 2.3.1. 先调用sync_array_get_and_reserve_cell从wait_array获取一个cell，m_waiters设置为true
 2.3.2. 尝试4次m_lock_word原子变量cas操作，如果成功就返回
 2.3.3. 调用sync_array_wait_event等待信号量唤醒
`

#### sync_array_t
```
struct sync_array_t {
ulint n_reserved; //正在使用的cell个数
ulint n_cells; //数组分配大小 
sync_cell_t *cells; //数组
ulint next_free_slot; //除了free list以外，下一个可以用的cell
ulint first_free_slot; //free list链表头, 复用cell里面的line字段作为next指针
}

```

sync_array_init 初始化sync_wait_array 二维数组，第一维大小1，第二维大小100k。 

sync_array_reserve_cell 从sync_wait_array 里面获取一个free cell，极限情况全部cell被占用就返回nullptr 

sync_array_free_cell 放回cell到free list 

sync_array_wait_event 等待信号量唤醒 

#### GenericPolicy
`struct GenericPolicy {
latch_id_t m_id;
/** Number of spins trying to acquire the latch. */
uint32_t m_spins;

/** Number of waits trying to acquire the latch */
uint32_t m_waits;

/** Number of times it was called */
uint32_t m_calls;
}
`
每次加锁都会更新里面相关字段,因此通过GenericPolicy可以看到锁竞争激烈程度

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)