# MySQL · 插件分析 · Connection Control

**Date:** 2020/05
**Source:** http://mysql.taobao.org/monthly/2020/05/08/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2020 / 05
 ](/monthly/2020/05)

 * 当期文章

 Database · 技术方向 · 下一代云原生数据库详解
* Database · 理论基础 · 高性能B-tree索引
* Database · 理论基础 · ARIES/IM (一)
* AliSQL · 引擎特性 · Fast Query Cache 介绍
* MySQL · 源码分析 · 8.0 · DDL的那些事
* MySQL · 内核分析 · InnoDB Buffer Pool 并发控制
* MySQL · 源码分析 · 内部 XA 和组提交
* MySQL · 插件分析 · Connection Control
* MySQL · 引擎特性 · 基于GTID复制实现的工作原理

 ## MySQL · 插件分析 · Connection Control 
 Author: 巴彦 

 本文基于mysql 8.0.13.

### 插件介绍

MySQL 5.6.35 开始提供Connnection Control 插件；

如果客户端在连续失败登陆一定次数后，那么此插件可以给客户端后续登陆行为的响应增加一个延迟。该插件可以防止恶意暴力破解MySQL账户。该插件包含以下2个组件：

`- CONNECTION_CONTROL：检查mysql的刚建立连接的响应是否需要延迟，并且提供一些系统变量和状态参数；方便用户配置插件和查看此插件基本的状态。
- CONNECTION_CONTROL_FAILED_LOGIN_ATTEMPTS：提供了一个INFORMATION_SCHEMA类型的表，用户在此表中可以查看更详细关于登陆失败连接的信息。
`

### 基本使用

#### 插件的安装与卸载

安装可以通过配置文件静态安装，也可以在MySQL中动态安装。

**静态安装**

`-- 配置文件增加以下配置
[mysqld]
plugin-load-add = connection_control.so
`

**动态安装**

`-- 插件动态安装启用
mysql> INSTALL PLUGIN CONNECTION_CONTROL SONAME 'connection_control.so';
mysql> INSTALL PLUGIN CONNECTION_CONTROL_FAILED_LOGIN_ATTEMPTS SONAME 'connection_control.so';

-- 验证是否正常安装
mysql> SHOW PLUGINS;
`

**卸载**

`-- 插件卸载
UNINSTALL PLUGIN CONNECTION_CONTROL;
UNINSTALL PLUGIN CONNECTION_CONTROL_FAILED_LOGIN_ATTEMPTS;
`

更多关于插接件安装/卸载的信息 [请点击](https://dev.mysql.com/doc/refman/8.0/en/plugin-loading.html)

#### 插件参数

* **connection_control_failed_connections_threshold**：失败登陆次数达到此值后触发延迟。
 
 值域：[0, INT_MAX32(2147483647)]，0表示关闭此功能。
* 默认值：3

 **connection_control_max_connection_delay**：登陆发生延迟时，延迟的最大时间；此值必须大于等于**connection_control_min_connection_delay**。
 * 值域：[1, INT_MAX32(2147483647)]
* 默认值：INT_MAX32
* 单位：毫秒

 **connection_control_min_connection_delay**：登陆发生延迟时，延迟的最小时间，此值必须小于等于**connection_control_max_connection_delay**。
 * 值域：[1000, INT_MAX32(2147483647)]
* 默认值：1000
* 单位：毫秒

### 基本原理

* Connection Control 插件通过订阅MYSQL_AUDIT_CONNECTION_CLASSMASK 来处理 MYSQL_AUDIT_CONNECTION_CONNECT(完成认证后触发)和MYSQL_AUDIT_CONNECTION_CHANGE_USER(完成COM_CHANGE_USER RPC后触发)子事件；通过这两种子事件的处理来检查给客户端发送回包时是否需要延迟。
* Connection Control 插件通过 LF hash来存储不同账户的失败登陆信息。LF hash中的key为**user@host **，这里的**user**与**host**将遵循以下条件：
 
 如果在MySQL的security context有proxy user信息，那么这个信息将用于**user**与**host**；
* 否则，查看security context是否有priv_user 和 priv_host信息，如果有则用于**user**与**host**；
* 否则，将security context中已经连接的user 和 host信息用于**user**与**host**。

 LF hash的更新：对于每次失败的登陆通过**user@host **的key值对其value加1；对于每次成功的登陆，如果需要延迟，处理完延迟后将**user@host **从LF hash中删除。
 为什么在达到**connection_control_failed_connections_threshold**失败登陆次数后的第一次成功登陆需要延迟？
 * 这其实还是出于对攻击者开销的考虑；如果成功登陆后马上返回，不需要延迟，那么攻击者就可以使用更少的连接数，进一步攻击者所消耗的资源就会更少；为了增加攻击者的开销，在连续失败登陆后的第一次成功登陆，还是会产生延迟。

 具体延迟的时间如何计算？
 * 一旦连续的失败登陆次数超过设定阈值，那么就会产生延迟，并且延迟随着失败次数增加而增加，上限为**connection_control_max_connection_delay**；具体的计算方式如下：
* MIN ( MAX((failed_attempts - threshold), MIN_DELAY), MAX_DELAY)

### 实现分析

从上一小节的基本原理我们知道Connection Control插件主要是通过订阅处理MYSQL_AUDIT_CONNECTION_CONNECT与MYSQL_AUDIT_CONNECTION_CHANGE_USER事件来实现的。

主要处理流程如下：

`//创建一个新线程，处理新连接
handle_connection() in connection_handler_per_thread.cc
|
| //准备工作
->thd_prepare_connection() in sql_connect.cc
 | 
 | //进行登陆操作
 ->login_connection() in sql_connect.cc
 |
 | //对此连接的有效性进行验证
 ->check_connection() in sql_connect.cc
 |
 | //验证登陆
 ->acl_authenticate() in sql_authentication.cc
 |
 | //对登陆连接事件进行处理
 ->mysql_audit_notify() in sql_audit.cc
 |
 | //对登陆连接事件进行处理，并获得错误码
 ->mysql_audit_notify() in sql_audit.cc
 |
 | //获取需要处理登陆事件的插件
 ->mysql_audit_acquire_plugins() in sql_audit.cc
 |
 | //将连接事件分发，并按照需求是都获取插件处理的返回值
 ->event_class_dispatch_error() in sql_audit.cc
 |
 | //将连接事件分发
 ->event_class_dispatch() in sql_audit.cc
 |
 | // 调用插件的相关处理函数处理连接事件
 ->plugins_dispatch() in sql_audit.cc
 |
 | //检查当前插件是否需要处理此事件
 ->check_audit_mask（）in sql_audit.cc
 |
 | //connection_control处理连接事件
 ->connection_control_notify() in connection_control.cc
 |
 | //依次遍历订阅了连接事件的订阅者处理此事件
 ->notify_event() in connection_control_coordinator.cc
 |
 | //处理连接事件
 ->notify_event() in connection_delay.cc
`

下面我们主要看一下最终Connection Control插件是怎么处理连接事件的。

`
/**
 @brief Handle a connection event and if requried,
 wait for random amount of time before returning.
 We only care about CONNECT and CHANGE_USER sub events.
 @param [in] thd THD pointer
 @param [in] coordinator Connection_event_coordinator
 @param [in] connection_event Connection event to be handled
 @param [in] error_handler Error handler object
 @returns status of connection event handling
 @retval false Successfully handled an event.
 @retval true Something went wrong.
 error_buffer may contain details.
*/

bool Connection_delay_action::notify_event(
 MYSQL_THD thd, Connection_event_coordinator_services *coordinator,
 const mysql_event_connection *connection_event,
 Error_handler *error_handler) {
 
 ...

 // 只关注CONNECT与CHANGE_USER事件
 if (subclass != MYSQL_AUDIT_CONNECTION_CONNECT &&
 subclass != MYSQL_AUDIT_CONNECTION_CHANGE_USER)
 DBUG_RETURN(error);

 RD_lock rd_lock(m_lock);

 int64 threshold = this->get_threshold();

 // 拿到当前阈值检查阈值是否有效，DISABLE_THRESHOLD=0
 if (threshold <= DISABLE_THRESHOLD) DBUG_RETURN(error);

 int64 current_count = 0;
 bool user_present = false;
 Sql_string userhost;

 make_hash_key(thd, userhost);

 DBUG_PRINT("info", ("Connection control : Connection event lookup for: %s",
 userhost.c_str()));

 // 获取到当前失败登陆的次数
 user_present = m_userhost_hash.match_entry(userhost, (void *)&current_count)
 ? false
 : true;

 // 如果失败次数超过阈值，无论这次连接成功与否，都需要延迟
 // 同时更新统计信息
 if (current_count >= threshold || current_count < 0) {
 
 ulonglong wait_time = get_wait_time((current_count + 1) - threshold);

 if ((error = coordinator->notify_status_var(
 &self, STAT_CONNECTION_DELAY_TRIGGERED, ACTION_INC))) {
 error_handler->handle_error(
 ER_CONN_CONTROL_STAT_CONN_DELAY_TRIGGERED_UPDATE_FAILED);
 }

 // 在产生延迟时，需要释放读写锁，以减少锁的粒度
 // 防止阻塞对于IS table的数据访问
 rd_lock.unlock();
 conditional_wait(thd, wait_time);
 rd_lock.lock();
 }

 if (connection_event->status) {
 
 // 如果此次登陆失败，那么更新LF Hash
 if (m_userhost_hash.create_or_update_entry(userhost)) {
 error_handler->handle_error(
 ER_CONN_CONTROL_FAILED_TO_UPDATE_CONN_DELAY_HASH, userhost.c_str());
 error = true;
 }
 } else {
 
 // 如果此次登陆成功并且LF Hash中有数据，那么就删除LF Hash中的数据
 if (user_present) {
 (void)m_userhost_hash.remove_entry(userhost);
 }
 }

 DBUG_RETURN(error);
}

`

### 小结

1，通过分析Connection Control处理流程与具体实现，我们可以知道插件是如何来处理连接事件的。

2，该插件虽然可以防止恶意暴力破解MySQL账户，但是可能会浪费MySQL的资源；

```
- 比如如果短时间内有大量的恶意攻击，该插件虽然可以防止破解mysql账户，但是会消耗主机资源(每一个连接创建一个线程)；
- 如果这里使用了线程池，虽然可以避免消耗主机资源，但是等线程池中的线程被消耗光，再有新连接来就会拒绝服务。

```

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)