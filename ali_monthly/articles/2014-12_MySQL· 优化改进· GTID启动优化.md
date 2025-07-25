# MySQL·　优化改进· GTID启动优化

**Date:** 2014/12
**Source:** http://mysql.taobao.org/monthly/2014/12/09/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2014 / 12
 ](/monthly/2014/12)

 * 当期文章

 MySQL · 性能优化 · 5.7 Innodb事务系统
* MySQL · 踩过的坑 · 5.6 GTID 和存储引擎那会事
* MySQL · 性能优化 · thread pool 原理分析
* MySQL · 性能优化 · 并行复制外建约束问题
* MySQL · 答疑释惑 · binlog event有序性
* MySQL · 答疑释惑 · server_id为0的Rotate
* MySQL · 性能优化 · Bulk Load for CREATE INDEX
* MySQL · 捉虫动态·Opened tables block read only
* MySQL·　优化改进· GTID启动优化
* TokuDB · TokuDB · Binary Log Group Commit with TokuDB

 ## MySQL·　优化改进· GTID启动优化 
 Author: 

 **背景**

GTID 可以说是 MySQL 5.6 版本的一个杀手级特性，它给主备复制带来了极大的便利，RDS只读实例就是强依赖于这个特性。然而GTID在给我们带来便利的同时，也埋下了许多坑，最近几期内核月报中GTID的频繁出现也说明了这一点，对其我们可以说是既爱又恨。

GTID 并不是免费午餐，要使用它是要有代价的，为了保证GTID这个体系能够运转起来，需要做许多相关的工作，比如binlog里每个事务要多记 gtid_event 这种事件、写binlog的时候要生成 gtid、要维护几个GTID集合(logged, purged, owned)、THD类要多加GTID的成员变量等等，这些对性能和资源开销方面都有影响。

官方的最新代码中加入了一个关于GTID的优化，是在mysqld启动的时候，加快 gtid_set 初始化的速度，详见revno: 6110。关于GITD集合，最重要的有2个，一个是 gtid_executed， 另一个是gtid_purged，很多数据库运维相关的操作都要和这2个集合打交道，前者对应当前实例已经执行过的事务集合，后者对应已经执行过，但是已经不在binlog中的事务集合。mysqld 正常运行时，这2个集合是在内存中持续更新的，可是重启的时候，需要初始化这2个集合，因为并没有专门的地方记录这2个集合，初始化是通过读取binlog进行的。

**优化分析**

mysqld 是通过对 binlog.index 中记录的 binlog 文件做2次遍历来实现初始化的，第一次是从后向前，即从最新的binlog开始，到最老的binlog，对每个binlog从头到尾读一遍，初始化 gtid_executed 集合；第二次是从前往后，同样对每个binlog从头到尾读一遍，用来初始化gtid_purged 集合。每一遍的最好情况都是只读一个binlog文件，对gtid_executed 集合来说，只需要最新的binlog就行了，因为每个binlog开始会记录 previous_gitd_set，这个集合加上当前binlog内部记录的 gtid_event，就是所有已经执行的，也即 gtid_executed； 对gitd_purged来说，理想情况更简单，只需要读最老binlog文件的头部的previous_gtid event即可，文件里面的 gtid_event 根本不需要。

最坏情况是什么呢，就是一堆binlog文件里，只有其中一个文件里有gtid，其它都没有，这样的话，对于2遍扫描，都需要扫到这个binlog，才能确定这2个集合。

比如 a b c D e f 这几个，每个对应一个binlog文件，其中只有D含有gtid，其它的都没有，这样的话，每一遍的扫描都要读到文件D才能确定。

官方的优化是，不管什么情况下，每一遍的扫描，最多只读一个文件，不会再多读，如果最新和最老的文件都没有gtid，就把gtid_executed和gtid_purged设为空。

**优化场景**

下面我们来看下，这个优化有没有用 。
我们还是用 a b c d e f 这几个表示binlog文件，小写表示文件没有包含gtid，大写表示有。

开始没有开gtid，后来开了：a, b, c, d, e, F 这样的模式，gtid_executed 只读 F，gtid_purged 只读a, 前者是F全集，后者是空的，如果没有这个优化的话，gtid_executed 也是读一个文件，gtid_purged 要从a读到F，最终还是空的，优化是有效果的
开始有，后来没有：A, b, c, d, e, f，这种情况下 gtid_executed 集合被初始化成空集，gtid_purged 也是空集，初始化结果是错的
开始没有，中间有最后也没有：a, b, c, D, e, f 这种情况，2个集合都被初始化成空的，结果也是错的
一直有：A，B，C，D，E，F，这种本来就是最好情况，本来每次遍历就只读一个文件的，加不加这个优化都一样
其它情况可以自己推算下

总的来说这个优化是比较鸡肋的，有的情况下还会算错，官方的优化 patch 带了个开关控制，默认是关的，这个只是对个别场景比较适合，比如上面的场景1。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)