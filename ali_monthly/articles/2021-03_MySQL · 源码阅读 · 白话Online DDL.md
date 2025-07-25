# MySQL · 源码阅读 · 白话Online DDL

**Date:** 2021/03
**Source:** http://mysql.taobao.org/monthly/2021/03/06/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2021 / 03
 ](/monthly/2021/03)

 * 当期文章

 MySQL · 引擎特性 · InnoDB Faster truncate/drop table space
* MySQL · 源码阅读 · Decimal 的实现方法
* PolarDB · 最佳实践 · 并行查询优化器的应用实践
* PolarDB · 引擎特性 · 物理复制热点页优化
* DataBase · 引擎特性 · OLAP/HTAP列式存储引擎概述
* MySQL · 源码阅读 · 白话Online DDL

 ## MySQL · 源码阅读 · 白话Online DDL 
 Author: 翊云 

 ## 发展历程

MySQL Online DDL 功能从 5.6 版本开始正式引入，发展到现在的 8.0 版本，经历了多次的调整和完善。本文主要就 Online DDL 的发展过程，以及各版本的区别进行总结。其实早在 MySQL 5.5 版本中就加入了 INPLACE DDL 方式，但是因为实现的问题，依然会阻塞 INSERT、UPDATE、DELETE 操作，这也是 MySQL 早期版本长期被吐槽的原因之一。

在 MySQL 5.6 中，官方开始支持更多的 ALTER TABLE 类型操作来避免数据拷贝，同时支持了在线上 DDL 的过程中不阻塞 DML 操作，真正意义上的实现了 Online DDL。然而并不是所有的 DDL 操作都支持在线操作，后面会附上 MySQL 官方文档对于 DDL 操作的总结。到了 MySQL 5.7，在 5.6 的基础上又增加了一些新的特性，比如：增加了重命名索引支持，支持了数值类型长度的增大和减小，支持了 VARCHAR 类型的在线增大等。但是基本的实现逻辑和限制条件相比 5.6 并没有大的变化。MySQL 8.0 对 DDL 的实现重新进行了设计，其中一个最大的改进是 DDL 操作支持了原子特性。另外，Online DDL 的 ALGORITHM 参数增加了一个新的选项：INSTANT，只需修改数据字典中的元数据，无需拷贝数据也无需重建表，同样也无需加排他 MDL 锁，原表数据也不受影响。整个 DDL 过程几乎是瞬间完成的，也不会阻塞 DML。

关于 MySQL 8.0 原子 DDL 的介绍可以参考：http://mysql.taobao.org/monthly/2020/05/05/ 。

## 各版本支持

本文数据全部来自 MySQL 官方文档，此处进行一个集中的整理和总结：
[https://dev.mysql.com/doc/refman/5.6/en/innodb-online-ddl-operations.html](https://dev.mysql.com/doc/refman/5.6/en/innodb-online-ddl-operations.html)
[https://dev.mysql.com/doc/refman/5.7/en/innodb-online-ddl-operations.html](https://dev.mysql.com/doc/refman/5.7/en/innodb-online-ddl-operations.html)
[https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html)

 **操作**
 **版本**
 **INSTANT**
 **INPLACE**
 **重建表**
 **并发 DML**
 **仅修改元数据**

 **二级索引**

 创建二级索引
 MySQL 8.0
 No
 Yes
 No
 Yes
 No

 MySQL 5.7
 
 Yes
 No
 Yes
 No

 MySQL 5.6
 
 Yes
 No
 Yes
 No

 删除索引
 MySQL 8.0
 No
 Yes
 No
 Yes
 Yes

 MySQL 5.7
 
 Yes
 No
 Yes
 Yes

 MySQL 5.6
 
 Yes
 No
 Yes
 Yes

 重命名索引
 MySQL 8.0
 No
 Yes
 No
 Yes
 Yes

 MySQL 5.7
 
 Yes
 No
 Yes
 Yes

 MySQL 5.6

 增加全文索引
 MySQL 8.0
 No
 Yes*
 No*
 No
 No

 MySQL 5.7
 
 Yes*
 No*
 No
 No

 MySQL 5.6
 
 Yes*
 No*
 No
 No

 增加空间索引
 MySQL 8.0
 No
 Yes
 No
 No
 No

 MySQL 5.7
 
 Yes
 No
 No
 No

 MySQL 5.6

 修改索引类型
 MySQL 8.0
 Yes
 Yes
 No
 Yes
 Yes

 MySQL 5.7
 
 Yes
 No
 Yes
 Yes

 MySQL 5.6
 
 Yes
 No
 Yes
 Yes

 **主键**

 增加主键
 MySQL 8.0
 No
 Yes*
 Yes*
 Yes
 No

 MySQL 5.7
 
 Yes*
 Yes*
 Yes
 No

 MySQL 5.6
 
 Yes*
 Yes*
 Yes
 No

 删除主键
 MySQL 8.0
 No
 No
 Yes
 No
 No

 MySQL 5.7
 
 No
 Yes
 No
 No

 MySQL 5.6
 
 No
 Yes
 No
 No

 重建主键
 MySQL 8.0
 No
 Yes
 Yes
 Yes
 No

 MySQL 5.7
 
 Yes
 Yes
 Yes
 No

 MySQL 5.6
 
 Yes
 Yes
 Yes
 No

 **列操作**

 新增列
 MySQL 8.0
 Yes*
 Yes
 No*
 Yes*
 No

 MySQL 5.7
 
 Yes
 Yes
 Yes*
 No

 MySQL 5.6
 
 Yes
 Yes
 Yes*
 No

 删除列
 MySQL 8.0
 No
 Yes
 Yes
 Yes
 No

 MySQL 5.7
 
 Yes
 Yes
 Yes
 No

 MySQL 5.6
 
 Yes
 Yes
 Yes
 No

 重命名列
 MySQL 8.0
 No
 Yes
 No
 Yes*
 Yes

 MySQL 5.7
 
 Yes
 No
 Yes*
 Yes

 MySQL 5.6
 
 Yes
 No
 Yes*
 Yes

 调整列顺序
 MySQL 8.0
 No
 Yes
 Yes
 Yes
 No

 MySQL 5.7
 
 Yes
 Yes
 Yes
 No

 MySQL 5.6
 
 Yes
 Yes
 Yes
 No

 修改列默认值
 MySQL 8.0
 Yes
 Yes
 No
 Yes
 Yes

 MySQL 5.7
 
 Yes
 No
 Yes
 Yes

 MySQL 5.6
 
 Yes
 No
 Yes
 Yes

 修改列数据类型
 MySQL 8.0
 No
 No
 Yes
 No
 No

 MySQL 5.7
 
 No
 Yes
 No
 No

 MySQL 5.6
 
 No
 Yes
 No
 No

 扩展 VARCHAR 长度
 MySQL 8.0
 No
 Yes
 No
 Yes
 Yes

 MySQL 5.7
 
 Yes
 No
 Yes
 Yes

 MySQL 5.6

 删除列默认值
 MySQL 8.0
 Yes
 Yes
 No
 Yes
 Yes

 MySQL 5.7
 
 Yes
 No
 Yes
 Yes

 MySQL 5.6
 
 Yes
 No
 Yes
 Yes

 修改自增值
 MySQL 8.0
 No
 Yes
 No
 Yes
 No*

 MySQL 5.7
 
 Yes
 No
 Yes
 No*

 MySQL 5.6
 
 Yes
 No
 Yes
 No*

 修改列为空
 MySQL 8.0
 No
 Yes
 Yes*
 Yes
 No

 MySQL 5.7
 
 Yes
 Yes*
 Yes
 No

 MySQL 5.6
 
 Yes
 Yes*
 Yes
 No

 修改列为非空
 MySQL 8.0
 No
 Yes*
 Yes*
 Yes
 No

 MySQL 5.7
 
 Yes*
 Yes*
 Yes
 No

 MySQL 5.6
 
 Yes*
 Yes*
 Yes
 No

 修改列 ENUM 值
 MySQL 8.0
 Yes
 Yes
 No
 Yes
 Yes

 MySQL 5.7
 
 Yes
 No
 Yes
 Yes

 MySQL 5.6
 
 Yes
 No
 Yes
 Yes

 **表操作**

 修改 ROW_FORMAT
 MySQL 8.0
 No
 Yes
 Yes
 Yes
 No

 MySQL 5.7
 
 Yes
 Yes
 Yes
 No

 MySQL 5.6
 
 Yes
 Yes
 Yes
 No

 修改 KEY_BLOCK_SIZE
 MySQL 8.0
 No
 Yes
 Yes
 Yes
 No

 MySQL 5.7
 
 Yes
 Yes
 Yes
 No

 MySQL 5.6
 
 Yes
 Yes
 Yes
 No

 指定字符集
 MySQL 8.0
 No
 Yes
 Yes*
 No
 No

 MySQL 5.7
 
 Yes
 Yes*
 No
 No

 MySQL 5.6
 
 Yes
 Yes*
 No
 No

 修改字符集
 MySQL 8.0
 No
 No
 Yes*
 No
 No

 MySQL 5.7
 
 No
 Yes*
 No
 No

 MySQL 5.6
 
 No
 Yes
 No
 No

 OPTIMIZE 表
 MySQL 8.0
 No
 Yes*
 Yes
 Yes
 No

 MySQL 5.7
 
 Yes*
 Yes
 Yes
 No

 MySQL 5.6
 
 Yes*
 Yes
 Yes
 No

 重命名表
 MySQL 8.0
 Yes
 Yes
 No
 Yes
 Yes

 MySQL 5.7
 
 Yes
 No
 Yes
 Yes

 MySQL 5.6
 
 Yes
 No
 Yes
 Yes

结合上面的表格，对 MySQL 当前 DDL 的执行模式总结如下：

INSTANT DDL 是 MySQL 8.0 引入的新功能，当前支持的范围较小，包括：

* 修改二级索引类型
* 新增列
* 修改列默认值
* 修改列 ENUM 值
* 重命名表

在执行 DDL 操作时，MySQL 内部对于 ALGORITHM 的选择策略是：如果用户显式指定了 ALGORITHM，那么使用用户指定的选项；如果用户未指定，那么如果该操作支持 INPLACE 则优先选择 INPLACE，否则选择 COPY；当前不支持 INPLACE 的操作主要有：

* 删除主键
* 修改列数据类型
* 修改表字符集

我们常说的 Online DDL，其实是从 DML 操作的角度描述的，如果 DDL 操作不阻塞 DML 操作，那么这个 DDL 就是 Online 的。当前非 Online 的 DDL 其实已经比较少了，主要有：

* 新增全文索引
* 新增空间索引
* 删除主键
* 修改列数据类型
* 指定表字符集
* 修改表字符集

更多详细的示例请参考上面的官方文档的地址。

## 几个问题

最后讨论几个非常容易混淆的问题：

1. Online DDL 不会锁表，可以随意的执行。
2. 支持 INPLACE 算法的 DDL 一定是 Online 的。
3. 对于支持 INPLACE 算法的 DDL，DDL 操作是原地修改数据，不需要额外的数据空间。

### Q1: Online DDL 会不会锁表

Online DDL 会不会锁表？要回答这个问题，首先要明确“锁表”的含义。很多 MySQL 用户经常在表无法正常的进行 DML 时就觉得是锁表了，这种说法其实过于宽泛，实际上能够影响 DML 操作的锁至少包括以下几种（默认为 InnoDB 表）：

* MDL 锁
* 表锁
* 行锁
* GAP 锁

其中除了 MDL 锁是在 Server 层加的之外，其它三种都是在 InnoDB 层加的。具体的加锁逻辑不在此进行展开，但是需要明确一点：所有的操作（不管是 DDL 还是 DML 还是查询语句）都需要先拿 Server 层的 MDL 锁，然后再去拿 InnoDB 层的某个需要的锁。一个 DDL 的基本过程是这样的：

1. 首选，在开始进行 DDL 时，需要拿到对应表的 MDL X 锁，然后进行一系列的准备工作；
2. 然后将 MDL X 锁降级为 MDL S 锁，进行真正的 DDL 操作；
3. 最后再次将 MDL S 锁升级为 MDL X 锁，完成 DDL 操作，释放 MDL 锁；

所以在真正执行 DDL 操作期间，确实是不会“锁表”的，但是如果在第一阶段拿 MDL X 锁时无法正常获取，那就可能真的会“锁表了”。一个简单的例子如下：

`# session 1
select sleep(300) from mytest.t1;

# session 2
optimize table mytest.t1;

# session 3
select * from mytest.t1;

`

session 1 模拟了一个慢查询，然后 session 2 开始进行 DDL 操作，无法拿到 MDL X 锁，处于等到中。此时 session 3 需要执行一个查询，发现无法执行。实际上，在 session 1 结束前，表 t1 的所有操作都无法进行了，也可以说表 t1 “锁表”了。MySQL 5.7/8.0 可以在开启 performance_schema 的情况下直接查询 metadata_locks 表。阿里云 RDS 5.6 版本新增了 I_S.MDL_INFO 表，提供 MDL 的查询。

`MySQL [performance_schema]> select * from metadata_locks where OBJECT_NAME = 't1';
+-------------+---------------+-------------+-------------+-----------------------+----------------------+---------------+-------------+-------------------+-----------------+----------------+
| OBJECT_TYPE | OBJECT_SCHEMA | OBJECT_NAME | COLUMN_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE | LOCK_DURATION | LOCK_STATUS | SOURCE | OWNER_THREAD_ID | OWNER_EVENT_ID |
+-------------+---------------+-------------+-------------+-----------------------+----------------------+---------------+-------------+-------------------+-----------------+----------------+
| TABLE | mytest | t1 | NULL | 140730442220576 | SHARED_READ | TRANSACTION | GRANTED | sql_parse.cc:5916 | 1083 | 24 |
| TABLE | mytest | t1 | NULL | 140730576178368 | SHARED_NO_READ_WRITE | TRANSACTION | PENDING | sql_parse.cc:5916 | 1091 | 3 |
| TABLE | mytest | t1 | NULL | 140730374843168 | SHARED_READ | TRANSACTION | PENDING | sql_parse.cc:5916 | 1092 | 3 |
+-------------+---------------+-------------+-------------+-----------------------+----------------------+---------------+-------------+-------------------+-----------------+----------------+
3 rows in set (0.00 sec)

`

明确了上面的概念之后，再回到我们的问题，Online DDL 是不是不锁表？如果非要回答，那么只能说，Online DDL 并不是绝对安全，更不是可以随意的执行。线上操作还是需要在业务低峰期谨慎操作。

### Q2: 支持 INPLACE 算法的 DDL 一定是 Online 的

从概念上来说，INPLACE 和 Online 是两个不同维度的事情。COPY 和 INPLACE 指的是 DDL 内部的执行逻辑，可以简单的理解成：COPY 是在 Server 层的操作，INPLACE 是在 InnoDB 层的操作。而用户更加关心 Online 与否，通常只与一个问题有关：是否允许并发 DML。两个基本结论：

1. COPY 算法执行的 DDL 肯定不是 Online 的；
2. INPLACE 算法执行的 DDL 不一定是 Online 的；

### Q3: INPLACE DDL 需不需要额外的数据空间

前面我们提到过，MySQL 内部对于 DDL 的 ALGORITHM 有两种选择：INPLACE 和 COPY（8.0 新增了 INSTANT，但是使用范围较小）。COPY 算法理解起来相对简单一点：创建一张临时表，然后将原表的数据拷贝到临时表中，最后再用临时表替换原表。对于上面的步骤，由于需要将原表的数据拷贝到临时表中，所以肯定需要消耗额外的数据空间。

那么对于支持 INPLACE 算法的 DDL，是不是不需要额外的数据空间？答案是：需要。其实之所以会问这个问题，还是因为对 INPLACE 本身的理解出现了偏差。简单来说：INPLACE 描述的是表，而不是数据文件。只要不创建临时表，那么都是 INPLACE 的。

实际上，很多 INPLACE DDL 都会重建表（会创建临时数据文件），所以都会需要额外的数据空间，例如：

* 增加主键
* 重建主键
* 新增列（8.0 支持 INSTANT DDL，不需要）
* 删除列
* 调整列顺序
* 删除列默认值
* 增加列默认值
* 修改表的 ROW_FORMAT
* OPTIMIZE 表

## 总结

本文主要是对 MySQL Online DDL 进行了一个简单的整理和总结，更多关于 MySQL 内部实现细节和源码的分析，请关注后续文章。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)