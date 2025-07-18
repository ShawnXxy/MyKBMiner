# MySQL · 捉虫动态 · Count(Distinct) ERROR

**Date:** 2014/08
**Source:** http://mysql.taobao.org/monthly/2014/08/03/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2014 / 08
 ](/monthly/2014/08)

 * 当期文章

 MySQL · 参数故事 · timed_mutexes
* MySQL · 参数故事 · innodb_flush_log_at_trx_commit
* MySQL · 捉虫动态 · Count(Distinct) ERROR
* MySQL · 捉虫动态 · mysqldump BUFFER OVERFLOW
* MySQL · 捉虫动态 · long semaphore waits
* MariaDB · 分支特性 · 支持大于16K的InnoDB Page Size
* MariaDB · 分支特性 · FusionIO特性支持
* TokuDB · 性能优化 · Bulk Fetch
* TokuDB · 数据结构 · Fractal-Trees与LSM-Trees对比
* TokuDB·社区八卦·TokuDB团队

 ## MySQL · 捉虫动态 · Count(Distinct) ERROR 
 Author: 

 **背景**

　　MySQL现行版本中存在一个count(distinct)语句返回结果错误的bug，表现为，实际结果存在值，但是用count(distinct)统计后返回的是0。

`drop table if exists tb;
set tmp_table_size=1024;
create table tb(id int auto_increment primary key, v varchar(32)) charset=gbk;
insert into tb(v) values("aaa");
insert into tb(v) (select v from tb);
insert into tb(v) (select v from tb);
insert into tb(v) (select v from tb);
insert into tb(v) (select v from tb);
insert into tb(v) (select v from tb);
insert into tb(v) (select v from tb);
insert into tb(v) (select v from tb);
insert into tb(v) (select v from tb);
update tb set v=concat(v, id);
select count(distinct v) from tb;
返回0

上述中update语句的目的是将所有的v值设为各不相同。
`

**原因分析**

　　Count(distinct f)的语义就是计算字段f的去重总数，计算流程大致如下：

　　流程一:

1. 构造一个unique集合A1（用tree实现） 2、 对每个值都试图插入集合A1中 3、 若和A1中现有item重复则直接跳过，不重复则插入并+1 4、 完成后计算集合中元素个数。

　　细心的同学会看到上面的语句中有一个set tmp_table_size的过程，集合A1并不能无限扩大，大小上限为tmp_table_size。若超过则上述流程变为

　　流程二：

1. 构造一个unique 集合A1 2、 插入item过程中若大小超过tmp_table_size，则将A1暂时写到文件中，再构造集合A2 3、 重复步骤2直到所有的item插入完成 因此若item很多则可能重复生成多个集合A1～An。 4、 对A1～An作合并操作。由于只是每个集合A保证unique，因此需要做类似归并排序的操作（实际上不需要排序，只是扫一遍） 5、 因此合并操作需要一个临时内存，长度为n，单元大小为key_length （key大小）。这个临时内存，用的也是tmp_table_size定义的大小。实际上在合并过程中还需要长为key_length的预留空间作临时内存保存。因此需要的空间为 (n+1)*key_length。 6、 在进行合并前会判断tmp_table_size >=(n+1)*key_length， 不满足则直接放弃合并。其结果就是返回为0。

**案例分析**

　　以上面这个case为例。字段v的单key大小为65 (65 = 32*2+1) 加上tree节点字占空间24字节共89字节。单个集合只能放11个item （1024/89）， 因此n为 24 （24>=256/11）, 在合并时需要 (24+1)*65= 1625字节的临时空间，大于1024，放弃合并。

Sql_big_tables

　　实际上在最初处理这个问题时，DBA同学发现社区也有人讨论这个bug，并且指出在set sql_big_tables=on的时候，执行count(distinct)就能正确返回结果。原因就是在sql_big_tables=on的情况下，构造集合的方式是直接生成一个临时表，全部插入后直接计算临时表的大小作为结果，整个过程与tmp_table_size无关。

**解决方法**

　　运维上，set sql_big_tables是一个方法，不过会影响性能。调高tmp_table_size算是正招。当然本质上这是一个bug。 　　代码上，对于已经走到合并操作的这个逻辑，如果tmp_table_size不够，应该直接申请新的临时空间用于合并，完成后释放。虽然会造成临时征用内存，不过以现有的逻辑来看，临时征用的内存已经不少了.

　　另外一种时间换空间的方法，就是作多次合并。

　　相比之下第一种改造比较简单安全。该bug在RDS MySQL 5.5 中已经修复。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)