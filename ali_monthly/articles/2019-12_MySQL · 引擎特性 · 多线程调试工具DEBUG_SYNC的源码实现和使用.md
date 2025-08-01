# MySQL · 引擎特性 · 多线程调试工具DEBUG_SYNC的源码实现和使用

**Date:** 2019/12
**Source:** http://mysql.taobao.org/monthly/2019/12/04/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2019 / 12
 ](/monthly/2019/12)

 * 当期文章

 MySQL · 引擎特性 · 动态元信息持久化
* MySQL · 引擎特性 · Binlog encryption 浅析
* MySQL · 代码阅读 · MYSQL开源软件源码阅读小技巧
* MySQL · 引擎特性 · 多线程调试工具DEBUG_SYNC的源码实现和使用
* MySQL · 引擎特性 · InnoDB Parallel read of index

 ## MySQL · 引擎特性 · 多线程调试工具DEBUG_SYNC的源码实现和使用 
 Author: YuanZhen 

 ## 背景介绍
在MySQL的开发过程中为了验证某个需要多线程之间配合的功能时，就需要有一种机制使开发人员能够控制每个线程的执行流程，完成多个线程之间的配合，验证特殊并发逻辑下代码处理的正确性。MySQL 提供了DEBUG_SYNC 功能，就是让开发者可以在MySQL服务器代码中通过DEBUG_SYNC宏定义同步点的。你可以在代码中加入你希望定义的同步点。

## 使用方式
DEBUG_SYNC的功能，默认是关闭的。除非在启动的时候指定了–debug-sync-timeout[=N] 选项，N是可选的，可以指定也可以不指定。不指定的话，默认是300秒。

`#define DEBUG_SYNC_DEFAULT_WAIT_TIMEOUT 300
`
这个选项是启动时变量，为了能在测试中使用DEBUG_SYNC功能，必须在启动的时候指定–debug-sync-timeout[=N] 选项。这个参数有两个作用：
1) 其一是指定wait_for一个同步点的最大等待时间（单位：秒），若超过这个时间就会timeout；
2) 另一个是打开／关闭DEBUG_SYNC功能的选项，当其后的参数N为0时，就关闭了DEBUG_SYNC功能。

## 代码片段解析
其核心代码主要通过定义的宏DEBUG_SYNC做为入口，其定义如下：

`#define DEBUG_SYNC(_thd_, _sync_point_name_) \
 do { \
 if (unlikely(opt_debug_sync_timeout)) \
 debug_sync(_thd_, STRING_WITH_LEN(_sync_point_name_)); \
 } while (0)
`
这个宏的源码实现，主要是通过定义一个同步点（这个同步点是通过这里定义的名字来表示的_sync_point_name_），线程就会在这个同步点执行定义的行为动作，比如是在这个同步点发信号给等在其他同步点的线程、还是等在某个定义的事件上。在DEBUG_SYNC目前同步点的行为只定义了给其它同步点发信号、和等在某个信号上。其实现的主要数据结构如下：

`struct st_debug_sync_action {
 ulong activation_count = 0; /* max(hit_limit, execute) */
 ulong hit_limit = 0; /* hits before kill query */
 ulong execute = 0; /* executes before self-clear */
 ulong timeout = 0; /* wait_for timeout */
 String signal; /* signal to emit */
 String wait_for; /* signal to wait for */
 String sync_point; /* sync point name */
 bool need_sort = false; /* if new action, array needs sort */
 bool clear_event = false; /* do not clear signal if false */
};
`
而其功能的实现主要就是通过debug_sync_execute函数来实现的。以下是在debug_sync中调用debug_sync_find和debug_sync_execute的代码片段。

` if (ds_control->ds_active &&
 (action = debug_sync_find(ds_control->ds_action, ds_control->ds_active,
 sync_point_name, name_len)) &&
 action->activation_count) {
 /* Sync point is active (action exists). */
 debug_sync_execute(thd, action);

 /* Statistics. */
 ds_control->dsp_executed++;

 /* If action became inactive, remove it to shrink the search array. */
 if (!action->activation_count) debug_sync_remove_action(ds_control, action);
 }
`
首先在debug_sync_find里通过二分查找是否有同步点的要执行的行为动作，若是找到的话，就通过debug_sync_execute函数去执行。在debug_sync_execute根据定义的同步点执行次数，去判断是否达到了执行的次数，若没有达到执行的次数，则会在每次都会等这个event的信号。

`if (action->execute) {
 …
 action->execute--;
 …
 /** 如果本线程也需要等待某个信号，它首先把自己在processlist表里的状态设置成等待状态，为了能让其它线程能及时的看到*/
 if (action->wait_for.length()) {
 …
 debug_sync_thd_proc_info(thd, ds_control->ds_proc_info);
 }

 /* 如果定义了需要唤醒的同步点，就需要把这些唤醒的同步点设置成signaled，加入到全局变量中，然后唤醒其它等待线程*/
 if (action->signal.length()) {
 …
 /**把这些唤醒的同步点设置成signaled，加入到全局变量中*/
 if (!s.empty()) add_signal_event(&s);

 /* 唤醒等待同步点读线程*/
 mysql_cond_broadcast(&debug_sync_global.ds_cond);
 }

 /* 然后自己再等待在自己定义的事件上，等待被唤醒*/
 if (action->wait_for.length()) {
 …
 while (!is_signalled(&wait_for) && !thd->killed &&
 opt_debug_sync_timeout) {
 error = mysql_cond_timedwait(&debug_sync_global.ds_cond,
 &debug_sync_global.ds_mutex, &abstime);

 …
 }

 /* 如果定义了 CLAER行为则清除等待事件，以后再执行到此不必再等待该事件 */
 if (action->clear_event) clear_signal_event(&wait_for);
 }
 /* 如果定义了 HIT_LIMIT行为，则达到了指定的次数，会返回错误消息，并kill这个线程 */
 if (action->hit_limit) {
 if (!--action->hit_limit) {
 thd->killed = THD::KILL_QUERY;
 my_error(ER_DEBUG_SYNC_HIT_LIMIT, MYF(0));
 }
 。。。
 }

 …

}
`

## 用法简介
### 在源代码中定义一个同步点
在源码中使用的例子如下所示，开发者可以在任意的位置加入同步点，并给同步点命名，这样这个同步点就可以在接下来的测试案例中使用了。

` open_tables(...)

 DEBUG_SYNC(thd, "after_open_tables");

 lock_tables(...)
`
### 在测试场景中使用同步点
测试场景使用的语法，可以参考 [https://dev.mysql.com/doc/internals/en/syntax-debug-sync-values.html](https://dev.mysql.com/doc/internals/en/syntax-debug-sync-values.html)。在测试场景中，同步点的使用主要有以下几种情况：
1）SET DEBUG_SYNC=‘sync point name SIGNAL signal name WAIT_FOR signal name 是最常用的方法。
比如：
SET DEBUG_SYNC= ‘after_open_tables SIGNAL opened WAIT_FOR flushed’; 大部分情况下同步点都是未激活状态，当对整个同步点请求某个行为时就激活了这个同步点。比如上面这个例子，当执行到同步点after_open_tables后会向等待opened事件发送信号同时等在flushed时间上时，就激活了after_open_tables同步点。

2）SET DEBUG_SYNC= ‘after_open_tables SIGNAL a,b,c WAIT_FOR flushed’;
这中用法和1）的主要区别就是一次唤醒多个事件a、b、c，其它和1）相同

3）SET DEBUG_SYNC= ‘WAIT_FOR flushed NO_CLEAR_EVENT’;
默认情况下, 当等待线程收到唤醒的信号后，就会从全局信号中把这个信号清除。但如果等待这个信号的线程有多个的时候，就不能其中一个线程被唤醒后马上清除它，这样就需要在等待线程，在等待信号上指定NO_CLEAR_EVENT。直到所有的等待线程都唤醒了，然后再通过SET DEBUG_SYNC= ‘RESET’; 去清除这个event的唤醒信号。

4）SET DEBUG_SYNC= ‘name SIGNAL sig EXECUTE 3’;
一般情况下，等待线程被激活执行完后，马上就清除唤醒等待线程的信号。为了不立马清除激活信号，我们可以通过关键字EXECUTE指定执行的次数，执行完指定的次数后，才清除激活信号。比如这个例子指定了执行3次、每执行完一次这个数字就会减1，直到减到0为止。

5） SET DEBUG_SYNC= ‘name WAIT_FOR sig TIMEOUT 10 EXECUTE 2’;
在MySQL启动的时候，可以通过参数debug-sync-timeout指定一个等待事件的超时时间，也可以通过TIMEOUT 关键字为每个等待事件单独指定超时时间。这个例子就是等待线程最长等待10秒，若超过10秒还没收到唤醒等待事件的信号，就会超时不再等待了。

6）SET DEBUG_SYNC= ‘name SIGNAL sig EXECUTE 2 HIT_LIMIT 3’;
如果你想在执行完指定的次数后，返回一个错误消息并且中断这个线程的话，可以通过HIT_LIMIT来指定。这个例子中就是在执行完3次后，会返回一个错误消息并且中断这个查询。

7）SET DEBUG_SYNC= ‘name CLEAR’;
这个是可以在任何时候都清除name指定的同步点，不管它执行了还是没执行。

## 参考
[https://dev.mysql.com/doc/internals/en/debug-sync-facility.html](https://dev.mysql.com/doc/internals/en/debug-sync-facility.html)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)