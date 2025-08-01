# MySQL · 捉虫动态 · binlog重放失败

**Date:** 2014/10
**Source:** http://mysql.taobao.org/monthly/2014/10/03/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2014 / 10
 ](/monthly/2014/10)

 * 当期文章

 MySQL · 5.7重构 · Optimizer Cost Model
* MySQL · 系统限制 · text字段数
* MySQL · 捉虫动态 · binlog重放失败
* MySQL · 捉虫动态 · 从库OOM
* MySQL · 捉虫动态 · 崩溃恢复失败
* MySQL · 功能改进 · InnoDB Warmup特性
* MySQL · 文件结构 · 告别frm文件
* MariaDB · 新鲜特性 · ANALYZE statement 语法
* TokuDB · 主备复制 · Read Free Replication
* TokuDB · 引擎特性 · 压缩

 ## MySQL · 捉虫动态 · binlog重放失败 
 Author: 

 **背景**

在 MySQL 日常维护中，要回滚或者恢复数据，我们经常会用 binlog 来在数据库上重放，执行类似下面的语句：

`mysqlbinlog mysql-bin.000001 | mysql -hxxxx -Pxx -u
`
最近遇到了这样一个问题，在重放 binlog 时，mysqld 报这样的错

`ERROR 1064 (42000) at line 25: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'DELIMITER ;
`

**分析**

上面的错是说语法不对，难道是 binlog 写错了，为了方便查看，先把 mysqlbinlog 解析结果保存到一个文件

`mysqlbinlog mysql-bin.000001 > abc.sql
`
然后打开 abc.sql 文件，会看到这样的语句

`"CREATE TABLE t_binlog_sbr(a int)^@"
`
最后面的奇怪的 “^@” 这是啥呢，我们用二进制方式打开文件后，发现这个其实是1个字节，值是 00，被显示成 “^@”了。

为啥后面会多 1 个 0 呢，后来发现是用户在用 MySQL C API 时用错了，具体是这个函数 mysql_real_query，基原型是

`int mysql_real_query(MYSQL *mysql, const char *stmt_str, unsigned long length)
`
详细说明参考[这里](http://dev.mysql.com/doc/refman/5.6/en/mysql-real-query.html)，length 参数表示 stmt_str 的长度，所以正常的调用应该是这样的：

`mysq_real_query(mysql, sql, strlen(sql))
`
可是用户在使用时多加了个1，变成这样

`mysq_real_query(mysql, sql, strlen(sql) + 1)
`
最终导致记录的 binlog 后面多了个 ‘\0’。 这个问题只在 statement 格式有，row 格式没有。

**解决方法**

有同学会问，+1 可以，+2、 +3 呢，这个是不可以的，>=2 的都是不行的，语句发过来后，mysqld 在 parse_sql 阶段直接报错返回了，后面就不会执行了。

1. 修改代码

 mysq_real_query(mysql, sql, strlen(sql) + 1) 这种用法是不对的，但是 MySQL 却允许，虽然这么用是不对的，但是为了兼容性，最好还是允许这种使用方式，但是在写binlog的时候做个判断，长度是不是写错了，错了的话纠正过来，在 THD::binlog_query 里面改。
2. 5.6 版本加参数

 如果是用 5.6 版本的 mysql client 的话，在重放时出错提示信息不一样，是类似下面这样的，更加友好，这个错误是 mysql client 报的，不是mysqld报的：

 `ERROR at line 24: ASCII '\0' appeared in the statement, but this is not allowed unless option --binary-mode is enabled and mysql is run in non-interactive mode. Set --binary-mode to 1 if ASCII '\0' is expected....
` 
 5.6 版本的 mysql client 多了一个参数 –binary-mode，允许语句里有 ‘\0’，所以如果是用5.6的话，就可以不用修改代码，重放binlog时这样做就可以了：

 ```
mysqlbinlog mysql-bin.000001 | mysql --binary-mode -hxxxx -Pxx -u

```

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)