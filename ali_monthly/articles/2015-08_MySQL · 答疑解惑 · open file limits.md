# MySQL · 答疑解惑 · open file limits

**Date:** 2015/08
**Source:** http://mysql.taobao.org/monthly/2015/08/07/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2015 / 08
 ](/monthly/2015/08)

 * 当期文章

 MySQL · 社区动态 · InnoDB Page Compression
* PgSQL · 答疑解惑 · RDS中的PostgreSQL备库延迟原因分析
* MySQL · 社区动态 · MySQL5.6.26 Release Note解读
* PgSQL · 捉虫动态 · 执行大SQL语句提示无效的内存申请大小
* MySQL · 社区动态 · MariaDB InnoDB表空间碎片整理
* PgSQL · 答疑解惑 · 归档进程cp命令的core文件追查
* MySQL · 答疑解惑 · open file limits
* MySQL · TokuDB · 疯狂的 filenum++
* MySQL · 功能分析 · 5.6 并行复制实现分析
* MySQL · 功能分析 · MySQL表定义缓存

 ## MySQL · 答疑解惑 · open file limits 
 Author: 冷香 

 ## 背景

最近在Aliyun RDS的环境上，有些用户碰到了打开文件句柄数过多的错误，查看用户实例的打开句柄个数，确实超过了系统设置的值，一旦出现了这种错误，将会带来连锁的各种错误(取决于当时正在操作什么类型的文件，以及什么操作)。下面，我们就一起来看一下MySQL在操作过程中，牵涉到文件打开和关闭的关键点，以及你一直以来可能存在的认识误区。

## 参数和名词

**关联参数**

我们先列一下几个关键的参数，不了解的可以先参考官方文档，我们假设在MySQL 5.6版本上，主要针对InnoDB表。

`open_file_limits
table_open_cache
table_definition_cache
innodb_file_per_table
innodb_open_files
`

open_file_limits的设置的值，mysqld会通过setrlimit系统调用来初始化本进程可以使用的最大文件句柄数。

而另外的四个参数的设置，对打开的文件数产生什么影响？我们稍后分析。

**关联名词**

下面列出的名称，是在MySQL源代码中出现的，能帮助我们更好的理解参数的设置：

* table：MySQL操作一张表时建的对象，table_open_cache参数控制缓存的对象就是table；
* table_share：MySQL对一张表的定义建的对象， table_definition_cache参数控制缓存的对象就是table_share；
* handler：引擎句柄，和table是一对一的关系，每一种引擎都实现自己的handler，这也是MySQL支持多引擎的关键；
* innobase_share：InnoDB引擎层对应的表定义对象；
* dict_table：InnoDB引擎对应的表结构定义；
* fil_tablespace：InnoDB引擎层对应的表空间，当设置innodb_file_per_table=ON的时候，每一个InnoDB表对应一个表空间；
* fil_node：表空间对应的节点，如果一个表空间对应多个文件，比如logfile，那么就tablespace和node就是一对多的关系。

对照着上面提到的参数和名称，下面来看几个场景。

## 3. 场景

我们把一个简单的select语句在MySQL中的操作分为三个过程，open/read/close。

**open过程**

当操作某张表的时候，比如select * from test;

1. 首先初始化一个table_share对象:
 * 如果对象在table_definition_cache中存在，就直接引用；
* 如果不存在，就打开test.frm，读取表结构的定义，创建table_share对象，然后就关闭了test.frm。所以，这一步只在table_definition_cache未命中的时候，才open/close frm文件，并不占用太久的句柄。
2. 然后创建table对象:
 * 如果在table_open_cache中存在unused_table， 就直接使用；
* 如果不存在，就会创建table对象，这里并不牵涉到文件操作。

server层所需要的对象已经创建完毕，下面是InnoDB层:

1. 首先创建InnoDB handler 和Innobase_share 对象，这一步仅仅是内存对象，不牵涉文件操作。
2. 然后load dict_table对象:
 * 如果在dictionary cache中存在dict_table，就直接引用；
* 如果不存在，InnoDB会读取系统表空间(ibdata)的SYS_TABLES表，读取InnoDB记录的表结构定义，同时还会读取SYS_INDEXES, SYS_COLUMN, SYS_FOREIGN 等和表关联的定义。
注：这里会牵涉到数据字典的读取操作，但因为ibdata文件从系统启动的时候，就一直处在打开状态，并且不能关闭，所以这里也没有打开新的文件操作。
3. 最后load test表空间，在fil_system的缓存中查找: 如果存在，就直接使用，如果不存在，会读取SYS_DATAFILES系统字典表，并打开第一个文件，这里只有一个文件test.ibd，读取segment header验证space id，验证成功，就创建了file_tablespace和fil_node对象。
注：这里会打开test.ibd文件，验证完tablespace id就会关闭。

**结论**

这里我们发现，在读取一张表的之前的open过程，虽然有open file的动作，但都是用于初始化定义、结构等信息。所以table_open_cache， table_definition_cache并不会对open_file_limits有什么影响。而innodb_file_per_table的设置，只是增加了open file超过limit的几率，并不会有直接的影响。

**read 过程**

open完成后，当select需要扫描BTree结构上的某一个leaf page，而buffer pool未命中的时候，会发起IO操作:

InnoDB会通过fil_node对象里的file handler进行IO操作，但这里每次open file的时候，会进行innodb_open_files判断。如果当前InnoDB打开的文件数超过了innodb_open_files，就会强制关闭一些文件，在fil_system全局结构中有一个LRU链表，这里保存了所有打开的用户表空间文件句柄，并且当前没有任何IO操作。系统可以安全的关闭一些文件句柄，以满足innodb_open_files的需要。

**结论**
对于InnoDB来说，innodb_open_files设置了一个安全的file limit，除非InnoDB发起的并发IO请求数过多，并且分散在不同的表空间上。

**close过程**

当语句完成后，会进行close动作：

1. 如果当前table cache大小没有超过table_open_cache，就把table缓存到cache中；
2. 如果table_share的ref_count，就是table引用次数减到0，说明没有table引用，并且超过了table_definition_cache，就从cache中删除；
3. 同样，innobase_share也根据ref_count来判断是否要缓存。InnoDB层的dict_table也缓存在dictionary cache中。

**结论**
close的过程，并没有文件的关闭动作，而仅仅是内存对象的缓存或者销毁的动作。

总体来看这三个过程：InnoDB所有和文件相关的对象、fil_tablespace、fil_node和语句、事务等一些生命周期并没有什么关系。所以语句的并发，事务的大小等等因素都不会引起文件打开数过多。

**recovery过程**

recovery的过程中，因为无法判断要应用的redo，所以会load fil_space一遍，会打开所有的ibd文件一遍，进行一次读取。但同样受限innodb_open_files的控制，同时打开文件数不能超过这个值。

不过5.7已经添加了一种新的redo类型MLOG_FILE_NAME ，来[优化recovery的过程](http://mysqlserverteam.com/innodb-crash-recovery-improvements-in-mysql-5-7/)。

**系统启动**
在系统启动的过程中，会初始化ibdata和logfile文件，这两类文件，是永久打开的，不受innodb_open_files限制，但数量有限。

从上面来看，InnoDB并不会引起这么明显的open files过多的问题，那问题究竟出现在哪里？

## 问题

在MySQL实例中还存在其他文件，比如用户连接创建的socket、binlog 文件、relay log文件、slow log等log文件。socket由max_connections来控制，log文件的打开数量有限。所以问题回到了MyISAM表上面，对于用户创建的MyISAM分区表，open的过程中，会把MYD文件全部打开，当分区过多的时候，open files数量就急剧上升，导致超过limit值。

## flush table会关闭打开的文件吗？

flush table操作，会把table cache中未使用的table close掉，前面我们看到close的操作，并不会产生文件关闭操作。不过MyISAM实现的handler的close函数，会把打开的文件句柄给关闭掉，所以flush table能够缓解open files过多的问题，但仅限于MyISAM，而InnoDB的文件打开/关闭逻辑并不受影响。

## 总结
所以，对于用户实例的open file limit的设置问题，需要计算好连接数、系统文件、表文件等文件，另外建议使用InnoDB表来避免open files 暴涨的问题。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)