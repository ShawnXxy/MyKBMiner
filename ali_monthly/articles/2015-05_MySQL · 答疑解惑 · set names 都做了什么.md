# MySQL · 答疑解惑 · set names 都做了什么

**Date:** 2015/05
**Source:** http://mysql.taobao.org/monthly/2015/05/07/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 05
 ](/monthly/2015/05)

 * 当期文章

 MySQL · 引擎特性 · InnoDB redo log漫游
* MySQL · 专家投稿 · MySQL数据库SYS CPU高的可能性分析
* MySQL · 捉虫动态 · 5.6 与 5.5 InnoDB 不兼容导致 crash
* MySQL · 答疑解惑 · InnoDB 预读 VS Oracle 多块读
* PgSQL · 社区动态 · 9.5 新功能BRIN索引
* MySQL · 捉虫动态 · MySQL DDL BUG
* MySQL · 答疑解惑 · set names 都做了什么
* MySQL · 捉虫动态 · 临时表操作导致主备不一致
* TokuDB · 引擎特性 · zstd压缩算法
* MySQL · 答疑解惑 · binlog 位点刷新策略

 ## MySQL · 答疑解惑 · set names 都做了什么 
 Author: 襄洛 

 ## 背景
最近有同事问，set names 时会同时设置了3个session变量

`SET character_set_client = charset_name;
SET character_set_results = charset_name;
SET character_set_connection = charset_name;
`

就从变量名字来看，character_set_client 是设置客户端相关的字符集，character_set_results 是设置返回结果相关的字符集，character_set_connection 这个就有点不太明白了，这个有啥用呢？

## 概念说明
通过[官方文档](http://dev.mysql.com/doc/refman/5.6/en/charset-connection.html)来看:

1. character_set_client 是指客户端发送过来的语句的编码;
2. character_set_connection 是指mysqld收到客户端的语句后，要转换到的编码；
3. 而 character_set_results 是指server执行语句后，返回给客户端的数据的编码。

对人来说，能够理解的是各种各样的符号，而对计算机来说，只能理解二进制，二进制和符号之间的对应关系就是编码。不同地域国家都有自己的一套符号集合，每个都各自用一组二进制数字表示，从而形成了不同的编码，字符集就可以看作是编码和符号的对应关系集合。同一个二进制数在不同的字符集下可能对应完全不一样的字符，如在GBK字符集中，`C4E3` 对应的是`你`，而在big5字符集中对应的是`斕`，而 `你`在unicode中的编码是`4F60`，在[Collation-Charts](http://collation-charts.org/) 这个网站有字符集和编码对应关系图，可以非常直观地看到不同编码下二进制数和符号的对应关系。

set names 设置的3个变量就是设置mysqld和客户端通信时，mysqld应该如何解读client发来的字符，以及返回给客户端什么样的编码。

## 实验测试

环境如下：

`mysql> show variables like 'character%';
+--------------------------+-------------------------------------+
| Variable_name | Value |
+--------------------------+-------------------------------------+
| character_set_client | utf8 |
| character_set_connection | utf8 |
| character_set_database | utf8 |
| character_set_filesystem | binary |
| character_set_results | utf8 |
| character_set_server | utf8 |
| character_set_system | utf8 |
`

server端的3个编码设置都是utf8。
另外，客户端是标准 mysql client，使用的编码是utf8，和sever端编码是一致的。

建一张表作为测试

`CREATE TABLE t1(id INT, name VARCHAR(200) CHARSET utf8) engine=InnoDB;

INSERT INTO t1 VALUES(0, '你好');
mysql> SELECT id, name, hex(name) FROM t1;
+------+--------+--------------+
| id | name | hex(name) |
+------+--------+--------------+
| 0 | 你好 | E4BDA0E5A5BD |
+------+--------+--------------+
`

下面我们分别改变这3个值，来看下结果会有什么变化

**Case 1 只改变 character_set_client**

`SET character_set_client=gbk;
INSERT INTO t1 VALUES(1, '你好');
mysql> SELECT id, name, hex(name) FROM t1;
+------+-----------+--------------------+
| id | name | hex(name) |
+------+-----------+--------------------+
| 0 | 你好 | E4BDA0E5A5BD |
| 1 | 浣犲ソ | E6B5A3E78AB2E382BD |
+------+-----------+--------------------+
2 rows in set (0.00 sec)
`

可以看到返回的数据已经乱码了，并且数据库里存的确实和第一条记录不一样。

**case 2 只改变 character_set_connection**

`SET names utf8;
SET character_set_connection = gbk;
INSERT INTO t1 VALUES(2, '你好');

mysql> SELECT id, name, hex(name) FROM t1;
+------+-----------+--------------------+
| id | name | hex(name) |
+------+-----------+--------------------+
| 0 | 你好 | E4BDA0E5A5BD |
| 1 | 浣犲ソ | E6B5A3E78AB2E382BD |
| 2 | 你好 | E4BDA0E5A5BD |
+------+-----------+--------------------+
3 rows in set (0.00 sec)
`

**case 3 只改变 character_set_results**

`SET names utf8;
SET character_set_results = gbk;
INSERT INTO t1 VALUES(3, '你好');

mysql> select id, name, hex(name) from t1;
+------+--------+--------------------+
| id | name | hex(name) |
+------+--------+--------------------+
| 0 | | E4BDA0E5A5BD |
| 1 | 你好 | E6B5A3E78AB2E382BD |
| 2 | | E4BDA0E5A5BD |
| 3 | | E4BDA0E5A5BD |
+------+--------+--------------------+
4 rows in set (0.00 sec)
`

再改回原样，看下结果

`SET names utf8;
mysql> SELECT id, name, hex(name) FROM t1;
+------+-----------+--------------------+
| id | name | hex(name) |
+------+-----------+--------------------+
| 0 | 你好 | E4BDA0E5A5BD |
| 1 | 浣犲ソ | E6B5A3E78AB2E382BD |
| 2 | 你好 | E4BDA0E5A5BD |
| 3 | 你好 | E4BDA0E5A5BD |
+------+-----------+--------------------+
4 rows in set (0.00 sec)
`

## 分析

我们先理下字符集在整个过程中是怎样变化的，然后再分析上面的case

客户发送请求时：

`A1 客户端发送出语句(总是以utf8)------> A2 sever收到语句解析(按character_set_client指定编码)
 |
 v
A4 数据进入mysqld内部存储<--------- A3 sever判断是否需要转换编码(以character_set_connection 目标编码)
`

server返回结果时：

`B1 server返回结果(按character_set_results 指定编码) ----->B2客户端解析编码显示(总是以utf8)
`
A3步是否需要转换编码，代码中的逻辑是这样的，在sql_yacc.yy文件中：

` LEX_STRING tmp;
 THD *thd= YYTHD;
 const CHARSET_INFO *cs_con= thd->variables.collation_connection;
 const CHARSET_INFO *cs_cli= thd->variables.character_set_client;
 uint repertoire= thd->lex->text_string_is_7bit &&
 my_charset_is_ascii_based(cs_cli) ?
 MY_REPERTOIRE_ASCII : MY_REPERTOIRE_UNICODE30;
 if (thd->charset_is_collation_connection ||
 (repertoire == MY_REPERTOIRE_ASCII &&
 my_charset_is_ascii_based(cs_con)))
 tmp= $1;
 else
 {
 if (thd->convert_string(&tmp, cs_con, $1.str, $1.length, cs_cli))
 MYSQL_YYABORT;
 }
 $$= new (thd->mem_root) Item_string(tmp.str, tmp.length, cs_con,
 DERIVATION_COERCIBLE,
 repertoire);
 if ($$ == NULL)
 MYSQL_YYABORT;
`
如果 `character_set_client` 和 `character_set_connection` 一样，或者当前的字符编码是和ASCII兼容，并且都是ASCII范围内的，就不转换，其它情况就转。

对于case1
实际上客户端发过来是UTF8的，但A2步骤server认为客户端的编码是GBK的，就按GBK来解析，同时满足A3步骤的转换条件，所以就误将UTF8编码认为是GBK，然后又给转成了UTF8。
`你好`的UTF8编码是 `E4BDA0E5A5BD` 6个字节，每个字符3个字节，按GBK来解析的话，因为GBK是固定2个字节，就认为有3个字符，然后转成UTF8，虽然UTF8是变长的，但是这里的3个GBK字符按值都是要占3个字节的，转出来一共9个字节。所以case1看到的实际存储的值一共9个字节，比原来的大。
在返回时，是按UTF8返回的，因为存了3个UTF8字符，所以客户端看到的就是3个。

对于case2
A2步骤没问题，问题是出在A3，按照转换逻辑，此时需要把UTF8转成GBK，这里因为`character_set_client`是正确的，所以转换的源不会识别错，转换成GBK自然也不会错，后面存储成UTF8时，再从GBK转成UTF8，也没错，因为UTF8和GBK字符集里都包含 ‘你’和’好’，所以相互转换也不会出错，只是多了2次转换。

对于case3
错在返回字符集设置的和客户端不匹配，在返回时，server将所有字符转成GBK的，结果客户端一根筋的认为是UTF8，就解析错了。
比较有意思的是第二条记录，即case1错误插进去的，显示出来是对的。
为什么呢，因为在case1中存的时候，是按 `UTF8->强制解析为GBK->然后转为UTF8` 这个逻辑存下去的，而返回的时候，因为server会将存的UTF8又给转回GBK，然后客户端又拿着这个GBK误以为是UTF8解析，实际上是case1的逆向过程，虽然2个方向都是错的，最终显示是好的，所谓的负负得正吧，哈哈。

对于case2 ，数据从客户端进入server的时候，多做了2次转换，最终显示还是对的，但不是所有场景都是这样，如下面这种

`set names utf8;
set character_set_connection = latin1;
INSERT INTO t1 VALUES(4, '你好');
set names utf8;
mysql> SELECT id, name, hex(name) FROM t1;
+------+-----------+--------------------+
| id | name | hex(name) |
+------+-----------+--------------------+
| 0 | 你好 | E4BDA0E5A5BD |
| 1 | 浣犲ソ | E6B5A3E78AB2E382BD |
| 2 | 你好 | E4BDA0E5A5BD |
| 3 | 你好 | E4BDA0E5A5BD |
| 4 | ?? | 3F3F |
+------+-----------+--------------------+
5 rows in set (0.00 sec)
`

为什么呢，因为在 UTF8转latin1时，信息丢失了，latin1字符编码所能表达的字符集是远小于utf8的，`你` 和 `好`就不在其中，这2个字符在转换中被转成了 `?` 和 `?`，之后存储转换成UTF8时，`?`只有一个字节`3F`，还原回去还是 `3F`。

## 总结

`character_set_client` 和 `character_set_results` 是一定要和客户端一致，不要依赖于负负得正，`character_set_connection` 设置和`character_set_client` 不一致，有丢失数据的风险，所以尽量也一致，总之这3个值就是要一样，还要和客户端一致，所以才有了 set names 这个快捷命令。关于为啥要有 `character_set_connection` 这一步转换，笔者目前还没看出来，以后理解了再更新，如果读者朋友知道的话，请不吝赐教。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)