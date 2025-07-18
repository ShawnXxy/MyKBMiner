# MySQL · 特性分析 · MySQL的预编译功能

**Date:** 2018/04
**Source:** http://mysql.taobao.org/monthly/2018/04/07/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2018 / 04
 ](/monthly/2018/04)

 * 当期文章

 MySQL · 引擎特性 · InnoDB 表空间加密
* MongoDB · myrocks · mongorocks 引擎原理解析
* MySQL · 引擎特性 · InnoDB 数据页解析
* MySQL · MyRocks · TTL特性介绍
* MySQL · 源码分析 · 协议模块浅析
* MSSQL · 最佳实践 · 如何监控备份还原进度
* MySQL · 特性分析 · MySQL的预编译功能
* MySQL · 特性分析 · (deleted) 临时空间
* MySQL · RocksDB · WAL(WriteAheadLog)介绍
* PgSQL · 应用案例 · 相似文本识别与去重

 ## MySQL · 特性分析 · MySQL的预编译功能 
 Author: xunchen 

 ## 背景
目前大部分关系型数据库执行sql的过程如下

1. 对SQL语句进行词法和语义解析，生成抽象语法树
2. 优化语法树，生成执行计划
3. 按照执行计划执行，并返回结果

绝大部分的常用SQL语句都可以被分解成静态部分和动态部分。静态部分主要包括sql语句的关键字（如DML,DDL等）以及数据库的对象及其相关信息（如表名，视图名，字段名等）。动态部分主要是由数据里的存储的数据构成。一个稳定运行的数据库中执行的所有sql语句，如果我们只关注静态部分，而忽略动态部分（以问号或者占位符对动态部分进行替换）。那么将会发现该系统执行的SQL语句的数量非常有限，只是相同的sql被反复的执行。这些反复执行的sql语句都用相同的执行计划。如果能让sql语句共享执行计划，将极大的提高执行的效率。很多主流的关系型数据库都支持sql语句以绑定变量的方式来共享执行计划。即编译一次，执行多次。遗憾的是mysql并不支持绑定变量。但是MySQL的预编译功能可以达到和绑定变量相同的效果。

## 开启MySQL的预编译功能
1.先建一张下面的表名为mytab

`CREATE TABLE `mytab` (
 `a` int(11) DEFAULT NULL,
 `b` varchar(20) DEFAULT NULL,
 UNIQUE KEY `ab` (`a`,`b`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8

`
2.通过PREPARE stmt_name FROM preparable_stmt 对下面的sql进行预编译

`
mysql> prepare sqltpl from 'insert into mytab select ?,?';
Query OK, 0 rows affected (0.00 sec)
Statement prepared

`
3.使用 EXECUTE stmt_name [USING @var_name [, @var_name] …] 来执行预编译语句

`mysql> set @a=999,@b='hello';
Query OK, 0 rows affected (0.00 sec)

mysql> execute ins using @a,@b;
Query OK, 1 row affected (0.01 sec)
Records: 1 Duplicates: 0 Warnings: 0

mysql> select * from mytab;
+------+-------+
| a | b |
+------+-------+
| 999 | hello |
+------+-------+
1 row in set (0.00 sec)
`
在MySQL中预编译语句作用域是session级，参数max_prepared_stmt_count可以控制全局最大的存储的预编译语句的数量。

当预编译条数已经达到阈值时可以看到MySQL会报错，如下。

`mysql> set @@global.max_prepared_stmt_count=1;
Query OK, 0 rows affected (0.00 sec)

mysql> prepare sel from 'select * from t';
ERROR 1461 (42000): Can't create more than max_prepared_stmt_count statements (current value: 1)
`

## 利用MySQL JDBC进行预编译
上面介绍了直接在MySQL上通过sql命令进行预编译/缓存sql语句。接下来我们以MySQL Java驱动Connector/J（版本5.1.45）测试通过MySQL驱动进行预编译。
###开启服务端预编译和客户端本地缓存
JDBC的连接串如下

`jdbc:mysql://localhost/test?useServerPrepStmts=true&cachePrepStmts=true,
`
并用下面的程序向表中插入两条记录

`public class PreparedStatementTest {
 public static void main(String[] args) throws Throwable {
 Class.forName("com.mysql.jdbc.Driver");

 String url = "jdbc:mysql://localhost/test?useServerPrepStmts=true&cachePrepStmts=true";
 try (Connection con = DriverManager.getConnection(url, "root", null)) {
 insert(con, 123, "abc");
 insert(con, 321, "def");
 }
 }

 private static void insert(Connection con, int arg1, String arg2) throws SQLException {
 String sql = "insert into mytab select ?,?";
 try (PreparedStatement statement = con.prepareStatement(sql)) {
 statement.setInt(1, arg1);
 statement.setString(2, arg2);
 statement.executeUpdate();
 }
 }
}
`
将会在mysql的后台日志中发现以下内容

`
2018-04-19T14:11:09.060693Z 45 Prepare insert into mytab select ?,?
2018-04-19T14:11:09.061870Z 45 Execute insert into mytab select 123,'abc'
2018-04-19T14:11:09.086018Z 45 Execute insert into mytab select 321,'def'

`

### 性能测试
我们来做一个简易的性能测试。首先写个存储过程向表中初始化大约50万条数据，然后使用同一个连接做select查询(查询条件走索引)。

`CREATE PROCEDURE init(cnt INT)
 BEGIN
 DECLARE i INT DEFAULT 1;
 TRUNCATE t;
 INSERT INTO mytab SELECT 1, 'stmt 1';
 WHILE i <= cnt DO
 BEGIN
 INSERT INTO t SELECT a+i, concat('stmt ',a+i) FROM mytab;
 SET i = i << 1;
 END;
 END WHILE;
 END;
mysql> call init(1<<18);
Query OK, 262144 rows affected (3.60 sec)

mysql> select count(0) from t;
+----------+
| count(0) |
+----------+
| 524288 |
+----------+
1 row in set (0.14 sec)

`

```

public static void main(String[] args) throws Throwable {
 Class.forName("com.mysql.jdbc.Driver");

 String url = "";

 long start = System.currentTimeMillis();
 try (Connection con = DriverManager.getConnection(url, "root", null)) {
 for (int i = 1; i <= (1<<19); i++) {
 query(con, i, "stmt " + i);
 }
 }
 long end = System.currentTimeMillis();

 System.out.println(end - start);
}
private static void query(Connection con, int arg1, String arg2) throws SQLException {
 String sql = "select a,b from t where a=? and b=?";
 try (PreparedStatement statement = con.prepareStatement(sql)) {
 statement.setInt(1, arg1);
 statement.setString(2, arg2);
 statement.executeQuery();
 }
}

```

以下几种情况，经过3测试取平均值，情况如下：

本地预编译：65769 ms
本地预编译+缓存：63637 ms
服务端预编译：100985 ms
服务端预编译+缓存：57299 ms
本地预编译加不加缓存其实差别不是太大，服务端预编译不加缓存性能明显会降低很多，但是服务端预编译加缓存的话性能还是会比本地好很多。主要原因是服务端预编译不加缓存的话本身prepare也是有开销的，另外多了大量的round-trip。

## 小结
经过实际测试，对于频繁使用的语句，使用服务端预编译+缓存效率还是能够得到可观的提升的。但是对于不频繁使用的语句，服务端预编译本身会增加额外的round-trip，因此在实际开发中可以视情况定夺使用本地预编译还是服务端预编译以及哪些sql语句不需要开启预编译等。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)