# MySQL · 引擎介绍 · Sphinx源码剖析（一）

**Date:** 2016/11
**Source:** http://mysql.taobao.org/monthly/2016/11/05/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 11
 ](/monthly/2016/11)

 * 当期文章

 PgSQL · 特性分析 · 金融级同步多副本分级配置方法
* MySQL · myrocks · myrocks之事务处理
* MySQL · TokuDB · rbtree block allocator
* MySQL · 引擎特性 · Column Compression浅析
* MySQL · 引擎介绍 · Sphinx源码剖析（一）
* PgSQL · 特性分析 · PostgreSQL 9.6 如何把你的机器掏空
* PgSQL · 特性分析 · PostgreSQL 9.6 让多核并行起来
* MSSQL · 最佳实战 · 巧用COLUMNS_UPDATED获取数据变更
* PgSQL · GIS应用 · 物流, 动态路径规划
* PgSQL · 特性分析· JIT 在数据仓库中的应用价值

 ## MySQL · 引擎介绍 · Sphinx源码剖析（一） 
 Author: 雕梁 

 ## 介绍

Sphinx是一个全文索引引擎，他被设计为可以非常简单方便的与各种数据库(mysql,PG…)进行交互。它提供了两种读取接口，a) sphinx自己实现的mysql协议的接口, SphinxQL。b) 各种语言客户端的接口,也就是native搜索API. c) 也可以直接通过mysql server的一个存储引擎插件来访问, SphinxSE.

接下来我们会有一些列文章来分析Sphinx的设计以及源码实现。

本篇是第一篇，主要是简要的介绍Sphinx的源码结构，设计以及索引文件的构成。我们当前分析的代码版本是sphinx-2.3.2-beta.tar.gz.

## 源码结构

sphinx的源码结构比较简单，下面主要是一些比较重要的目录:

* api 这个目录主要是包含了各种sphinx的native客户端.
* config 这个目录包含了configure需要的一些文件.
* cmake 这个目录包含了cmake构建需要的一些模块(当前Sphinx支持两种构建模式).
* mysqlse 这个目录包含了SphinxSE
* src 这个目录就是最主要的源码目录。
* src/http search服务的http接口

Sphinx最终会生成4个可执行文件，分别是:

* indexer 主要是操作索引文件，比如合并索引，重新构建索引等等
* indextool dump索引的一些信息，比如统计信息等
* searchd 搜索服务
* spelldump 拼写检查的工具

我们先来看Sphinx源码的CmakeFiles（src/CMakeLists.txt) ：

`set (LIBSPHINX_SRCS sphinx.cpp sphinxexcerpt.cpp
 sphinxquery.cpp sphinxsoundex.cpp sphinxmetaphone.cpp
 sphinxstemen.cpp sphinxstemru.cpp sphinxstemcz.cpp
 sphinxstemar.cpp sphinxutils.cpp sphinxstd.cpp
 sphinxsort.cpp sphinxexpr.cpp sphinxfilter.cpp
 sphinxsearch.cpp sphinxrt.cpp sphinxjson.cpp
 sphinxaot.cpp sphinxplugin.cpp sphinxudf.c
 sphinxqcache.cpp sphinxrlp.cpp)
set (INDEXER_SRCS indexer.cpp)
set (INDEXTOOL_SRCS indextool.cpp)
set (SEARCHD_SRCS searchd.cpp searchdha.cpp http/http_parser.c searchdhttp.cpp)
set (SPELLDUMP_SRCS spelldump.cpp)
...
add_library (libsphinx STATIC ${LIBSPHINX_SRCS} ${HEADERS} ${GHEADERS})
`

通过上面的构建文件我们可以看到4个可执行文件对应4个源文件(除了搜索服务，searchdha.cpp是分布式搜索，searchdhttp.cpp是搜索服务的http接口实现).剩下的源代码都会被编译为一个libsphinx的库.

因此下面简单介绍下libsphinx的几个文件主要作用:

* sphinx.cpp 核心的文件，一些核心功能的实现都在这里，比如读写索引文件，比如搜索的核心方法
* sphinxexcerpt.cpp 生成excerpt
* sphinxquery.cpp 处理query
* sphinxstemen.cpp sphinxstemru.cpp sphinxstemcz.cpp sphinxstemar.cpp 各种语言的解析器
* sphinxutils.cpp 一些工具函数，比如读写文件，日志，动态库等
* sphinxstd.cpp 库函数，实现了很多基本数据结构，比如 list/vector 等
* sphinxexpr.cpp 处理搜索的query
* sphinxfilter.cpp 处理query的filter
* sphinxsearch.cpp 核心的搜索处理函数
* sphinxrt.cpp rt index的实现
* sphinxjson.cpp json的处理

## 压缩格式

以RT索引为例，sphinx会有配置来决定内存中的索引大小(rt_mem_limit)，超过这个大小后，sphinx将会把内存索引刷新到磁盘中。接下来我们就来看sphinx的索引的含义以及原始格式。

在Sphinx中所有的索引最终都是被压缩的，压缩算法比较简单，要么是delta encoding, 要么是VLB(variable length byte string):

* delta encoding
 
 主要用来保存递增的一个序列，每一个元素都保存和前一个元素的差值。这种压缩更高效，结果更小(比起VLB) 比如：

` source-sequence = 3, 5, 7, 11, 13, 17, ...
 delta-encoded = 3, 2, 2, 4, 2, 4, ...
`

* VLB
 
 将一个固定大小(34/64)的整数值转换为一个字符串，每个字节分为高1位和低七位，最高位表示当前是否解析结束，低7位表示压缩的值，原理很简单，那么就是对于大多数整数来说,不需要完整的8个字节(或者4个字节)来表示一个整数，因为没有那么大。而由于是每次移动7位，那么对于最高位为1的情况也可以处理(因为每次移动完毕的最高位都是无意义的值)。
* 例子:

```
 source-value = 0x37
 encoded-value = 0x37

 source-value = 0x12345
 encoded-value = 0x84 0xC6 0x45
 // 0x84 == ( ( 0x12345>>14 ) & 0x7F ) | 0x80
 // 0xC6 == ( ( 0x12345>>7 ) & 0x7F ) | 0x80
 // 0x45 == ( ( 0x12345>>0 ) & 0x7F )

```

* 下面我们来看代码，对应的函数是CSphReader::UnzipInt以及CSphWriter::ZipInt，其中前一个是解压缩，后一个是压缩。
先来看压缩, DWORD可以简单地认为是int64_t,代码比较简单，首先先计算可以用几个字节来保存当前的数(iBytes),然后循环的保存每个字节:

```
void CSphWriter::ZipInt ( DWORD uValue )
{
 int iBytes = 1;

 DWORD u = ( uValue>>7 );
 while ( u )
 {
 u >>= 7;
 iBytes++;
 }

 while ( iBytes-- )
 PutByte (
 ( 0x7f & ( uValue >> (7*iBytes) ) )
 | ( iBytes ? 0x80 : 0 ) );
}

```

然后是解压缩，刚好和压缩相反，也就是通过最高位来判断是否结束解压缩，然后通过左移以及累加来不断地计算原有的值。

`DWORD CSphReader::UnzipInt () { SPH_VARINT_DECODE ( DWORD, GetByte() ); }

#define SPH_VARINT_DECODE(_type,_getexpr) \
 register DWORD b = _getexpr; \
 register _type res = 0; \
 while ( b & 0x80 ) \
 { \
 res = ( res<<7 ) + ( b & 0x7f ); \
 b = _getexpr; \
 } \
 res = ( res<<7 ) + b; \
 return res;
`

## 索引文件介绍

在介绍索引文件之前，我们先介绍几个概念：

* document
 
 每一条数据也就是一个document。

 word
 * 这里word表示一个单词，也就是Sphinx分词器处理完一条document之后，所得到的分词。

 hits
 * 也就是一个word在一条document中的频率。

 attribute
 * 一些扩展的字段，主要是为了做一些过滤（filter）。

然后来看索引文件的简单介绍.

* 然后我们来看索引的种类以及格式，在sphinx中，每一个索引都包含了下面几个文件：
 
 sph文件 保存了索引的头文件，主要是一些索引元信息
 
 实现在WriteHeader/LoadHeader中。

 spi文件 保存了wordlist,也就是索引文件中最核心的一个文件。
 * 也就是通过spi文件可以迅速的从一个keywords(word)映射到一堆document list。下面就是spi文件的格式(dict=keywords)：

`byte dummy = 0x01
keyword[] keyword_blocks
keyword is:
 byte keyword_editcode
 byte[] keyword_delta
 if keyword_editcode == 0:
 assert keyword_delta = { 0 }
 return block_end
 zint doclist_offset
 zint num_docs
 zint num_hits
 if num_docs >= DOCLIST_HINT_THRESH:
 byte doclist_sizehint
 if ver >= 31 and num_docs > SKIPLIST_BLOCK:
 zint skiplist_pos
 zint skiplist_len

if min_infix_len > 0:
 tag "infix-entries"
 infix_entry[] infix_hash_entries

checkpoint[] checkpoints
checkpoint is:
 dword keyword_len
 byte[] keyword [ keyword_len ]
 qword dict_offset

if min_infix_len > 0:
 tag "infix-blocks"
 infix_block[] infix_hash_blocks

tag "dict-header"
zint num_checkpoints
zint checkpoints_offset
zint infix_codepoint_bytes
zint infix_blocks_offset
`

* 文件生成是在cidxHit中。
* spa文件 保存了attribute
* sps文件 单独保存string类型的attribute值
* spd文件 保存了document list
 
 所有的document id都保存在这个这个文件中，也就是通过spi文件得到document list的信息后，可以迅速在spd文件中定位document list。

 spe文件 保存了skip list
 spk文件 保存了 kill list
 spm文件 保存了MVA 值
 spp文件 保存了hit list。
 * 保存了一个word在document中的所有出现的位置。也就是给定一个document 和一个keywords，这个文件将会返回所有的匹配位置(在当前的document中).

其中spp/spi/spd/spa/spe文件的生成都在RtIndex_t::SaveDiskDataImpl中实现。

在下一篇文章，我们会详细的介绍每个索引文件的生成。

## 参考文档

http://sphinxsearch.com/docs/current.html

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)