# MySQL · 答疑释惑· lower_case_table_names 使用问题

**Date:** 2015/03
**Source:** http://mysql.taobao.org/monthly/2015/03/07/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 03
 ](/monthly/2015/03)

 * 当期文章

 MySQL · 答疑释惑· 并发Replace into导致的死锁分析
* MySQL · 性能优化· 5.7.6 InnoDB page flush 优化
* MySQL · 捉虫动态· pid file丢失问题分析
* MySQL · 答疑释惑· using filesort VS using temporary
* MySQL · 优化限制· MySQL index_condition_pushdown
* MySQL · 捉虫动态·DROP DATABASE外键约束的GTID BUG
* MySQL · 答疑释惑· lower_case_table_names 使用问题
* PgSQL · 特性分析· Logical Decoding探索
* PgSQL · 特性分析· jsonb类型解析
* TokuDB ·引擎机制· TokuDB线程池

 ## MySQL · 答疑释惑· lower_case_table_names 使用问题 
 Author: 

 **背景**

在MySQL中，表是和操作系统中的文件对应的，而文件名在有的操作系统下是区分大小写的（比如linux），有的是不区分大小写（比如Windows），表名与文件名的大小写对应关系，MySQL 是通过[lower_case_table_names](http://dev.mysql.com/doc/refman/5.6/en/server-system-variables.html#sysvar_lower_case_table_names) 这个变量来控制的。

这个变量的有效取值是0，1，2，按照[官方文档](http://dev.mysql.com/doc/refman/5.5/en/identifier-case-sensitivity.html) 的解释：

0表示，表在文件系统存储的时候，对应的文件名是按建表时指定的大小写存的，MySQL 内部对表名的比较也是区分大小写的； 
1表示，表在文件系统存储的时候，对应的文件名都小写的，MySQL 内部对表名的比较是转成小写的，即不区分大小写； 
2表示，表在文件系统存储的时候，对应的文件名是按建表时指定的大小写存的，但是 MySQL 内部对表名的比较是转成小写的，即不区分大小写。

0适用于区分大小写的系统，1都适用，2适用于不区分大小写的系统。

如果在开始使用MySQL选定了一个合适的值后，就不要改变，不然的话在之后使用中就会出现问题。

**问题描述**

这里给出一个在使用过程中改变 lower_case_table_names 导致 drop database 失败的案例。因为lower_case_table_names是个只读变量，只能在启动时指定参数设置值，或者 gdb 挂上去直接改内存。

首先在启动 mysqld 的时候，指定 lower_case_table_names = 0，我们执行这样的语句：

`create database db1;
use db1;
create table t1(a int) engine = InnoDB; 
create table t2(a int) engine = MyISAM; 
create table T3(a int) engine = InnoDB; 
create table T4(a int) engine = MyISAM;
`

查看对应数据库目录下的表文件：

`$ls db1
db.opt t1.frm t1.ibd t2.frm t2.MYD t2.MYI T3.frm T3.ibd T4.frm T4.MYD T4.MYI
`

然后重启mysqld，指定 lower_case_table_names =1，执行删除db1

`mysql&gt; drop database db1;
ERROR 1010 (HY000): Error dropping database (can&#039;t rmdir &#039;./db1&#039;, errno: 39)
`

可以看到删库语句执行失败，我们再看下数据库目录下的表文件

`$ls a
T3.frm T4.frm T4.MYD T4.MYI
`

可以看到，大写的 T3 和 T4 表没有被删掉，为什么呢？

**问题分析**

mysqld 在执行 drop database 操作的时候，是调用 mysql_rm_db 这个函数，在删除时先把db下的所有表都删掉，然后再把db删掉。为了找出对应db下的所有表，mysqld 是通过遍历数据库目录下的文件来做的，具体是用 find_db_tables_and_rm_known_files 这个函数，遍历数据库目录下的所有文件，然后构造出要 drop 的table列表，然而在构造删除列表过程中，会有这样一个判断:

`if (lower_case_table_names)
table_list-&gt;table_name_length= my_casedn_str(files_charset_info,
table_list-&gt;table_name);
`

意思就是如果lower_case_table_names非0的话，就把 table_name 转成小写的，T3 和 T4 就被转成 t3 和 t4，这样生成的 table_list 中的对应的表是 t1,t2,t3,t4。之后拿着这样的 table_list 通过 mysql_rm_table_no_locks 一个个删表，这样就只把t1,t2 给删了，t3和t4不存在，并且删表时的逻辑是带有 if exists 的，所以也不会报错。

在list表都删除完后，调用rm_dir_w_symlink来删除db目录，此时db1目录下还有 T3 T4 对应的文件，这个函数会调用系统的 rmdir 函数，而当目录非空的时候，rmdir是执行失败的。

所以我们看到最终的错误提示 Error dropping database (can't rmdir './db1', errno: 39)

**建议**

上面的问题是改变 lower_case_table_names 导致 drop database 失败，其实还有许多其它的因为lower_case_table_names值改变导致的问题，比如主备库本来这个值本来是一致的，如果只改主库的值的话，就会导致备库复制中断，报找不到表的问题，或者本来是不区分大小写的，应用里的写的SQL语句有大写表名，也有小写表名，之后改成区分大小写，就会导致应用出错。

所以建议是:

* 不要轻易的改变lower_case_table_names的值，如果真要改的话，要先检查下已有的表是否有大小写的问题，保证目前的表名和要改的模式是一致的，比如从区分大小写改为不区分大小写，那就不应该有大写表存在，如果有的话，要先把大写表rename成小写的，如果本来有共存同名的大写表和小写表，就要想办法去掉一个。
* 应用不要依赖于 mysql 的表名转换机制，应用里的sql语句应该和表名一致，在不区分大小写的时候，应用里对同一个表的使用，不要既有大写表名，也有小写表名。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)