# 故障分析 | 有效解决 MySQL 行锁等待超时问题【建议收藏】

**原文链接**: https://opensource.actionsky.com/20200421-mysql/
**分类**: MySQL 新特性
**发布时间**: 2020-04-21T00:36:55-08:00

---

作者：xuty
本文来源：原创投稿
*爱可生开源社区出品，原创内容未经授权不得随意使用，转载请联系小编并注明来源。
**一、背景**
- `#### 20191219 10:10:10,234 | com.alibaba.druid.filter.logging.Log4jFilter.statementLogError(Log4jFilter.java:152) | ERROR |  {conn-10593, pstmt-38675} execute error. update operation_service set offlinemark = ? , resourcestatus = ?  where RowGuid = ?`
- `com.mysql.jdbc.exceptions.jdbc4.MySQLTransactionRollbackException: Lock wait timeout exceeded; try restarting transaction`
上述这个错误，接触 MySQL 的同学或多或少应该都遇到过，专业一点来说，这个报错我们称之为**锁等待超时**。
根据锁的类型主要细分为：- 行锁等待超时
当 SQL 因为等待行锁而超时，那么就为行锁等待超时，常在多并发事务场景下出现。
- 元数据锁等待超时
当 SQL 因为等待元数据锁而超时，那么就为元数据锁等待超时，常在 DDL 操作期间出现。
本文仅介绍如何有效解决行锁等待超时，因为大多数项目都是此类错误，元数据锁等待超时则不涉及讲解。
**二、行锁的等待**
在介绍如何解决行锁等待问题前，先简单介绍下这类问题产生的原因。
产生原因简述：当多个事务同时去操作（增删改）某一行数据的时候，MySQL 为了维护 ACID 特性，就会用锁的形式来防止多个事务同时操作某一行数据，避免数据不一致。只有分配到行锁的事务才有权力操作该数据行，直到该事务结束，才释放行锁，而其他没有分配到行锁的事务就会产生行锁等待。如果等待时间超过了配置值（也就是  innodb_lock_wait_timeout 参数的值，个人习惯配置成 5s，MySQL 官方默认为 50s），则会抛出行锁等待超时错误。
![](.img/36c8d3c6.png)											
如上图所示，事务 A 与事务 B 同时会去 Insert 一条主键值为 1 的数据，由于事务 A 首先获取了主键值为 1 的行锁，导致事务 B 因无法获取行锁而产生等待，等到事务 A 提交后，事务 B 才获取该行锁，完成提交。
这里强调的是行锁的概念，虽然事务 B 重复插入了主键，但是在获取行锁之前，事务一直是处于行锁等待的状态，只有获取行锁后，才会报主键冲突的错误。
当然这种 Inster 行锁冲突的问题比较少见，只有在大量并发插入场景下才会出现，项目上真正常见的是 update&delete 之间行锁等待，这里只是用于示例，原理都是相同的。
**三、产生的原因**
根据我之前接触到的此类问题，大致可以分为以下几种原因：
**1. 程序****中非数据库交互操作导致事务挂起**将接口调用或者文件操作等这一类非数据库交互操作嵌入在 SQL 事务代码之中，那么整个事务很有可能因此挂起（接口不通等待超时或是上传下载大附件）。
**2. 事务中包含性能较差的查询 SQL**事务中存在慢查询，导致同一个事务中的其他 DML 无法及时释放占用的行锁，引起行锁等待。
**3. 单个事务中包含大量 SQL**通常是由于在事务代码中加入 for 循环导致，虽然单个 SQL 运行很快，但是 SQL 数量一大，事务就会很慢。
**4. 级联更新 SQL 执行时间较久**这类 SQL 容易让人产生错觉，例如：`update A set ... where ...in (select B)` 这类级联更新，不仅会占用 A 表上的行锁，也会占用 B 表上的行锁，当 SQL 执行较久时，很容易引起 B 表上的行锁等待。
**5. 磁盘问题导致的事务挂起**极少出现的情形，比如存储突然离线，SQL 执行会卡在内核调用磁盘的步骤上，一直等待，事务无法提交。
综上可以看出，如果事务长时间未提交，且事务中包含了 DML 操作，那么就有可能产生行锁等待，引起报错。
**四、定位难点**
当 web 日志中出现行锁超时错误后，很多开发都会找我来排查问题，这里说下问题定位的难点！
1. MySQL 本身不会主动记录行锁等待的相关信息，所以无法有效的进行事后分析。
2. 锁争用原因有多种，很难在事后判断到底是哪一类问题场景，尤其是事后无法复现问题的时候。
3. 找到问题 SQL 后，开发无法有效从代码中挖掘出完整的事务，这也和公司**框架-产品-项目**的架构有关，需要靠 DBA 事后采集完整的事务 SQL 才可以进行分析。
**五、常用方法**
先介绍下个人通常是如何解决此类问题的， 这里问题解决的前提是问题可以复现，只要不是突然出现一次，之后再也不出现，一般都是可以找到问题源头的。
这里问题复现分为两种情景：
1. 手动复现只要按照一定的操作，就可以复现报错，这种场景较简单！
2. 随机复现不知道何时会突然报错，无法手动复现，这种场景较难！
下面先写下统一的模拟场景，用于复现行锁超时问题，便于大家理解：
- `--表结构`
- `CREATE TABLE `emp` (`
- `  `id` int(11) NOT NULL,`
- `  KEY `idx_id` (`id`)`
- `) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4`
- 
- `从1~100w插入100w行记录。`
- 
- `--测试过程：`
- `事务1：`
- `start transaction;`
- `delete from emp where id = 1;`
- `select * from emp where id in (select id from emp);   -->模拟慢查询，执行时间很久，事务因此一直不提交，行锁也不释放.`
- `commit;`
- 
- `事务2：`
- `start transaction;`
- `delete from emp where id < 10;   --> 处于等待id=1的行锁状态，当达到行锁超时时间（这里我配置了超时时间为 5s）后，返回行锁超时报错`
- `rollback;`
5.1 手动复现场景
这个场景通常只需要通过 innodb 行锁等待脚本就可以知道当前 MySQL 的 innodb 行锁等待情况，例如我们一边模拟上述报错场景（模拟页面操作），另一边使用脚本查询（需要在超时之前查询，否则超时报错后就看不到了）。- `/*innodb 行锁等待脚本*/`
- `SELECT r.trx_mysql_thread_id waiting_thread,r.trx_query waiting_query,`
- `concat(timestampdiff(SECOND,r.trx_wait_started,CURRENT_TIMESTAMP()),'s') AS duration,`
- `b.trx_mysql_thread_id blocking_thread,t.processlist_command state,b.trx_query blocking_current_query,e.sql_text blocking_last_query`
- `FROM information_schema.innodb_lock_waits w`
- `JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id`
- `JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id`
- `JOIN performance_schema.threads t on t.processlist_id = b.trx_mysql_thread_id`
- `JOIN performance_schema.events_statements_current e USING(thread_id)`
![](.img/cf3d90f8.png)											
如上我们可以看到事务 2 的线程 id 为 76，已经被事务 1，也就是线程 id 为 75 的事务阻塞了 3s，并且可以看到事务 1 当前执行的 SQL 为一个 SELECT。这里也解释了很多开发经常问我的，为什么 SELECT 也会阻塞其他会话？
如果遇到这种情况，那么处理其实非常简单。需要优化这个 SELECT 就好了，实在优化不了，把这个查询扔到事务外就可以了，甚至都不需要挖掘出整个事务。
上述这个问题模拟，其实就是对应第三节问题产生原因中的第二点**（事务中包含性能较差的查询 SQL）**，下面我们把第一点**（程序中非数据库交互操作导致事务挂起）**也模拟下，对比下现象。
我们只需要将事务 1 的过程改成如下即可。- `事务1：`
- `start transaction;`
- `delete from emp where id = 1;`
- `select * from emp where id in (select id from emp);`
- `等待60s(什么都不要做)             --> 模拟接口调用超时，事务夯住，随后再执行commit。`
- `commit;`
再次用脚本查看，可以看到现象是有所不同的，不同点在于，阻塞事务处于 sleep 状态，即事务当前并不在跑 SQL。从 DBA 的角度看，这类现象八成就可以断定是代码在事务中嵌入了其他的交互操作导致的事务挂起（另外也有可能是网络问题导致的事务僵死），因为程序并不像人，它不会偷懒，不会出现事务执行到一半，休息一会再提交一说。
![](.img/8cf98d42.png)											
如果是这类现象的问题，因为本质并不是由于 SQL 慢导致的事务挂起，所以必须要到代码里去找到对应的点，看下到底是在做什么交互操作卡住了。
这里就需要开发去排查代码才可以找到源头，但是唯一可用的信息就是该事务最后执行的一条 SQL，也就是上图中最后一列，从我之前的经验来看（绝大时候），开发很难单从这一条 SQL 就可以找到代码里具体位置，尤其是当这条 SQL 是一条很常见的 SQL，就更为困难！
当面对这种情况，就需要 DBA 去挖掘出这个事务执行过的所有 SQL，然后再让开发去排查代码，这样难度应该就小多了。这里就需要用到 MySQL 的 general_log，该日志用于记录 MySQL 中所有运行过的 SQL。
- `--查看general_log是否开启，及文件名`
- `mysql> show variables like '%general_log%';`
- `+------------------+--------------------------------------+`
- `| Variable_name    | Value                                |`
- `+------------------+--------------------------------------+`
- `| general_log      | OFF                                  |`
- `| general_log_file | /data/mysql_data/192-168-188-155.log |`
- `+------------------+--------------------------------------+`
- 
- `--暂时开启general_log`
- `mysql> set global general_log = 1;`
- `Query OK, 0 rows affected (0.00 sec)`
- 
- `--暂时关闭general_log`
- `mysql> set global general_log = 0;`
- `Query OK, 0 rows affected (0.00 sec)`
开启 general_log 后，手动复现的时候通过 innodb 行锁等待脚本查询结果中的线程 ID，去 general_log 找到对应的事务分析即可，如下：
![](.img/e11274f0.png)											
![](.img/b7f116e9.png)											
根据线程 ID 可以很轻易的从 general_log 中找到对应时间点的事务操作（实际场景下可能需要通过管道命令过滤）。如上图所示，事务 1 与事务 2 的全部 SQL 都可以找到，再通过这些 SQL 去代码中找到对应的位置即可，比如上图中线程 ID 为 111 的事务，执行 `select * from emp where id in (select id from emp)` 后到真正提交，过了 1min 左右，原因要么就是这条 SQL 查询慢，要么就是代码在执行其他交互操作。> PS：general_log 由于会记录所有 SQL，所以对 MySQL 性能影响较大，且容易暴涨，所以只在问题排查时暂时开启，问题排查后，请及时关闭！
5.2 随机复现场景
相较于手动复现场景，这种场景因为具有随机性，所以无法一边模拟报错，一边通过脚本查询到具体的阻塞情况，因此需要通过其他方式来监控 MySQL 的阻塞情况。
我一般是通过在 Linux 上后台跑监控脚本（innodb_lock_monitor.sh）来记录 MySQL 阻塞情况，脚本如下：- `#!/bin/bash`
- 
- `#账号、密码、监控日志`
- `user="root"`
- `password="Gepoint"`
- `logfile="/root/innodb_lock_monitor.log"`
- 
- `while true`
- `do`
- `        num=`mysql -u${user} -p${password} -e "select count(*) from information_schema.innodb_lock_waits" |grep -v count``
- `        if [[ $num -gt 0 ]];then`
- `            date  >> /root/innodb_lock_monitor.log`
- `            mysql -u${user} -p${password} -e  "SELECT r.trx_mysql_thread_id waiting_thread,r.trx_query waiting_query, \`
- `concat(timestampdiff(SECOND,r.trx_wait_started,CURRENT_TIMESTAMP()),'s') AS duration,\`
- `b.trx_mysql_thread_id blocking_thread,t.processlist_command state,b.trx_query blocking_query,e.sql_text \`
- `FROM information_schema.innodb_lock_waits w \`
- `JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id \`
- `JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id \`
- `JOIN performance_schema.threads t on t.processlist_id = b.trx_mysql_thread_id \`
- `JOIN performance_schema.events_statements_current e USING(thread_id) \G " >> ${logfile}`
- `        fi`
- `        sleep 5`
- `done`
再次查看- `--使用 nohup 命令后台运行监控脚本`
- `[root@192-168-188-155 ~]# nohup sh innodb_lock_monitor.sh  &`
- `[2] 31464`
- `nohup: ignoring input and appending output to ‘nohup.out’`
- 
- `--查看 nohup.out 是否出现报错`
- `[root@192-168-188-155 ~]# tail -f nohup.out`
- `mysql: [Warning] Using a password on the command line interface can be insecure.`
- `mysql: [Warning] Using a password on the command line interface can be insecure.`
- `mysql: [Warning] Using a password on the command line interface can be insecure.`
- 
- `--定时查看监控日志是否有输出(没有输出的话，这个日志也不会生成哦！)`
- `[root@192-168-188-155 ~]# tail -f innodb_lock_monitor.log`
- `Wed Feb  5 11:30:11 CST 2020`
- `*************************** 1. row ***************************`
- ` waiting_thread: 112`
- `  waiting_query: delete from emp where id < 10`
- `       duration: 3s`
- `blocking_thread: 111`
- `          state: Sleep`
- ` blocking_query: NULL`
- `       sql_text: select * from emp where id in (select id from emp)`
当监控日志有输出阻塞信息时，后续解决方案就和之前的手动复现场景一致。- 如果是事务卡在慢 SQL，那么就需要优化 SQL。
- 如果是事务挂起，那么就通过 general_log 分析事务，然后找到具体的代码位置。
> PS：问题排查完成后，请及时关闭后台监控进程，通过 `kill+pid` 的方式直接关闭即可！
**六、Performance_Schema**
之前的方法感觉不是很方便，因为 general_log 需要访问服务器，且过滤分析也较难，需要一定的 MySQL 基础及 Linux 基础才适用，因此想寻找一种更为简便的方法。
6.1 方法介绍
个人想法是利用 MySQL 5.5 开始提供的 performance_schema 性能引擎来进行分析，Performance_Schema 是 MySQL 提供的在系统底层监视 MySQL 服务器性能的一个特性，其提供了大量监控项，包括：锁、IO、事务、内存使用等。
**介绍下主要原理：**
1. 主要用的表有 2 张 events_transactions_history_long 和 events_statements_history_long。2. transactions_history_long 会记录历史事务信息，events_statements_history_long 则记录历史 SQL。3. 从 transactions_history_long 中得到回滚事务的线程 ID，再根据时间范围去筛选出可疑的事务，最后从 events_statements_history_long 得到事务对应的 SQL，从中排查哪个为源头。
**优点：**
1. 不需要通过 general_log 来获取事务 SQL。2. 不需要监控脚本来获取到行锁等待情况。3. 只需要访问 MySQL 就可以实现，而不需要访问服务器。4. 性能开销较小，且不会暴涨，因为是循环覆盖写入的。5. 可以知道每条 SQL 的运行时长。
**缺点：**
1. history_long 相关表默认保留记录有限，可能会把有用的数据刷掉，尤其是在 SQL 运行较多的系统。2. 如果要加大 history_long 相关表的最大保留行数，需要重启 MySQL，无法在线修改参数。3. history_long 相关表记录中的时间均为相对时间，也就是距离 MySQL 启动的时长，看起来不是很方便。4. history_long 相关表不会主动记录行锁等待的信息，所以只能通过先根据时间范围刷选出可疑的事务，再进一步分析，不如脚本监控定位的准。- `/*开启performance_schema相关监控项，需要提前开启performance_schema*/`
- `UPDATE performance_schema.setup_instruments SET ENABLED = 'YES', TIMED = 'YES' where name = 'transaction';`
- `UPDATE performance_schema.setup_consumers SET ENABLED = 'YES' where name like '%events_transactions%';`
- `UPDATE performance_schema.setup_consumers SET ENABLED = 'YES' where name like '%events_statements%';`
- 
- 
- `/*查看回滚事务SQL，确认是否是日志里报错的事务*/`
- `SELECT a.THREAD_ID`
- `    ,b.EVENT_ID`
- `    ,a.EVENT_NAME`
- `    ,CONCAT (b.TIMER_WAIT / 1000000000000,'s') AS trx_druation`
- `    ,CONCAT (a.TIMER_WAIT / 1000000000000,'s') sql_druation`
- `    ,a.SQL_TEXT,b.STATE,a.MESSAGE_TEXT`
- `FROM performance_schema.events_statements_history_long a`
- `JOIN performance_schema.events_transactions_history_long b ON a.THREAD_ID = b.THREAD_ID`
- `    AND (a.NESTING_EVENT_ID = b.EVENT_ID OR a.EVENT_ID = b.NESTING_EVENT_ID)`
- `WHERE b.autocommit = 'NO' AND a.SQL_TEXT IS NOT NULL AND b.STATE = 'ROLLED BACK'`
- 
- `/*查看该时间段内可疑事务即超过5s的事务SQL，这里默认innodb_lock_wait_timeout为5s*/`
- `SELECT a.THREAD_ID`
- `    ,b.EVENT_ID`
- `    ,a.EVENT_NAME`
- `    ,CONCAT (b.TIMER_WAIT / 1000000000000,'s') AS trx_druation`
- `    ,CONCAT (a.TIMER_WAIT / 1000000000000,'s') sql_druation`
- `    ,a.SQL_TEXT,b.STATE,a.MESSAGE_TEXT,a.ROWS_AFFECTED,a.ROWS_EXAMINED,a.ROWS_SENT`
- `FROM performance_schema.events_statements_history_long a`
- `JOIN performance_schema.events_transactions_history_long b ON a.THREAD_ID = b.THREAD_ID`
- `    AND (a.NESTING_EVENT_ID = b.EVENT_ID OR a.EVENT_ID = b.NESTING_EVENT_ID)`
- `WHERE b.autocommit = 'NO' AND SQL_TEXT IS NOT NULL AND b.STATE = 'COMMITTED'`
- `    AND b.TIMER_WAIT / 1000000000000  > 5`
- `    AND b.TIMER_START < (SELECT TIMER_START FROM performance_schema.events_transactions_history_long`
- `        WHERE THREAD_ID = 70402  /*上述SQL查询结果中的线程ID*/`
- `        AND EVENT_ID = 518)      /*上述SQL查询结果中的事件ID*/`
- `    AND b.TIMER_END > ( SELECT TIMER_END FROM performance_schema.events_transactions_history_long`
- `        WHERE THREAD_ID = 70402  /*上述SQL查询结果中的线程ID*/`
- `        AND EVENT_ID = 518)     /*上述SQL查询结果中的事件ID*/`
- `ORDER BY a.THREAD_ID`
6.2 测试模拟
如果是用这种方法的话，那么就不需要分手动复现还是随机复现了，操作方法都是一样的，下面模拟下如何操作：1. 首先通过上述方法开启 performance_schema 相关监控项，会直接生效，无需重启 MySQL。2. 然后复现问题，这里最好是手动复现（因为复现后如果没有及时查看，监控数据可能就会被刷掉），不行的话就只能等待随机复现了。3. 问题复现后通过上述脚本查询是否存在回滚事务（即因为行锁超时回滚的事务）。
![](.img/b2b64dcf.png)											
4. 然后根据回滚事务的线程 ID 和事件 ID，带入到最后一个脚本中，查看可疑事务，进行分析。
![](.img/1bc1f2c4.png)											
这里由于是测试环境模拟，所以结果非常了然，项目上实际输出结果可能有很多，需要一一分析事务是否有问题！
**七、总结**
实际测试后，发现通过 performance_schema 来排查行锁等待超时问题限制其实也比较多，而且最后的分析也是一门技术活，并不如一开始想象的那么简单，有点事与愿违了。
通过 performance_schema 排查问题最难处理的有 3 点：1. 时间问题，相对时间如何转换为绝对时间，这个目前一直找不到好的方法。2. 不会主动记录下行锁等待的信息，所以只能通过时间节点刷选后进一步分析。3. 记录被刷问题，因为是内存表，设置很大容易内存溢出，设置很小就容易被很快刷掉。