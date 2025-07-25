# MySQL · 系统限制 · text字段数

**Date:** 2014/10
**Source:** http://mysql.taobao.org/monthly/2014/10/02/
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

 ## MySQL · 系统限制 · text字段数 
 Author: 

 **背景**

　　当用户从oracle迁移到MySQL时，可能由于原表字段太多建表不成功，这里讨论一个问题：一个InnoDB表最多能建多少个text字段。

　　我们后续的讨论基于创建表的语句形如：create table t(f1 text, f2 text, …, fN text)engine=innodb;

**默认配置**

　　在默认配置下，上面的建表语句，N取值范围为[1, 1017]。 为什么是1017这个“奇怪”的数字。实际上单表的最大列数目是1024-1，但是由于InnoDB会增加三个系统内部字段（主键ID、事务ID、回滚指针），因此需要减3。而用于记录系统字典表也受1023的限制，又需要再增加三个该表的系统字段，因此每个表的最大字段数是1023-3*2。

**插入异常**

　　上述描述说明的是表能够创建成功的最大字段数。但是这样的表是“插入不安全”的。我们知道text的长度上限是64k。而往上表中插入一行，每个字段长度为7，就会报错：Row size too large (> 8126).

　　一个page是16k，空page扣掉页信息占用空间是16252，需要除以2，原因是每个page至少要包含两个记录。

　　也就是说，虽然可以创建一个包含1017个text字段的表，但是很容易碰到插入失败。

**如何保证插入安全**

　　上面的表结构，在保证插入安全的情况下，N的最大值是多少？text在存储的时候，当超过768字节的时候，剩余部分会保存在另外的页面（off-page），因此每个字段占用的最大空间为768+20+2=788. 20字节存储最短剩余部分的位置（SPACEID+PAGEID+OFFSET）。2字节存储本地实际长度。

　　因此N最大值为lower(8126/790)=10。

　　如果我们想在创建的表的时候，保证创建的表中的text字段都能安全的达到64k上限（而不是等插入的时候才发现），那么需要将默认为OFF的innodb_strict_mode设置为ON，这样在建表时会先做判断。

　　但是，在设置为严格模式后，上述建表语句的最大N却并非10.

ROW_FORMAT

　　在off-page存储时，本地占用790个字节，是基于默认的ROW_FORMAT，即为COMPACT，此时插入安全的N上限为10。

　　而在InnoDB新格式Barracuda支持下，Dynamic格式的off-page存储时，在local保存的上限不再是768，而是20个字节。这样每个字段在数据页里面占用的最大值是40byte，再需要一个额外的字节存储实际的本地长度，因此每个text最大占用41字节。

　　实际上很容易测试在严格模式下，建表的最大N为196. 以下为N=197时计算过程：

　　每行记录预留header 5个字节。

　　每个bit保存是否允许null，需要 upper(197/8)=25个字节。

　　三个系统保留字段 6+6+7=19.

　　因此总占用空间 5+25+19+41*197=8126！

　　也就是说，当N=197时，刚好长度为8126，而代码中实现是 if(rec_max_size >= page_rec_max) reutrn(error).

　　就这么不巧！

**作为补充**

　　有经验的读者可以联想到，如果我们的表中自己定义一个int型主键呢？此时系统不需要额外增加主键，因此整个表结构比之前少2字节。

　　也就是说，建表语句修改为: create table t(id int primary key, f1 text, f2 text, …, fN text)engine=innodb;

　　则此时的N上限能达到197。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)