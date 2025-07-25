# MySQL · 引擎介绍 · Sphinx源码剖析(三)

**Date:** 2017/10
**Source:** http://mysql.taobao.org/monthly/2017/10/06/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 10
 ](/monthly/2017/10)

 * 当期文章

 PgSQL · 特性分析 · MVCC机制浅析
* MySQL · 性能优化· CloudDBA SQL优化建议之统计信息获取
* MySQL · 引擎特性 · InnoDB mini transation
* MySQL · 特性介绍 · 一些流行引擎存储格式简介
* MSSQL · 架构分析 · 从SQL Server 2017发布看SQL Server架构的演变
* MySQL · 引擎介绍 · Sphinx源码剖析(三)
* PgSQL · 内核开发 · 如何管理你的 PostgreSQL 插件
* MySQL · 特性分析 · 数据一样checksum不一样
* PgSQL · 应用案例 · 经营、销售分析系统DB设计之共享充电宝
* MySQL · 捉虫动态 · 信号处理机制分析

 ## MySQL · 引擎介绍 · Sphinx源码剖析(三) 
 Author: 雕梁 

 在本节中我会介绍Sphinx在构建索引之前做的一些事情，主要是从mysql拉取数据保存，然后分词排序保存到内存等等一系列的操作。下面是几个相关指令

` sql_query = \
 SELECT id, group_id, UNIX_TIMESTAMP(date_added) AS date_added, \
 title, content \
 FROM documents
 sql_query_range = SELECT MIN(id),MAX(id) FROM documents
 sql_range_step = 1000
`

其中sql_query是sphinx每次从mysql拉取数据的sql，而sql_query_range则是取得需要从mysql拉取的数据条目，而sql_rang_step则是表示每次从mysql拉取多少数据。sql_rang_range执行分两种情况，第一种是第一次拉取数据的时候，第二种是当当前的range数据读取完毕之后。

首先来看CSphSource_SQL::NextDocument函数，这个函数的主要作用是从mysql读取数据然后切分保存，首先我们来看读取数据这一部分，这里步骤很简单，就是执行对应的sql，然后判断当前range的数据是否读取完毕，如果读取完毕则继续执行sql_query_rang(RunQueryStep)。这里要注意的是，sphinx读取数据是一条一条的读取然后执行的.

` do
 {
 // try to get next row
 bool bGotRow = SqlFetchRow ();

 // when the party's over...
 while ( !bGotRow )
 {
 // is that an error?
 if ( SqlIsError() )
 {
 sError.SetSprintf ( "sql_fetch_row: %s", SqlError() );
 m_tDocInfo.m_uDocID = 1; // 0 means legal eof
 return NULL;
 }

 // maybe we can do next step yet?
 if ( !RunQueryStep ( m_tParams.m_sQuery.cstr(), sError ) )
 {
 // if there's a message, there's an error
 // otherwise, we're just over
 if ( !sError.IsEmpty() )
 {
 m_tDocInfo.m_uDocID = 1; // 0 means legal eof
 return NULL;
 }

 } else
 {
 // step went fine; try to fetch
 bGotRow = SqlFetchRow ();
 continue;
 }

 SqlDismissResult ();

 // ok, we're over
 ARRAY_FOREACH ( i, m_tParams.m_dQueryPost )
 {
 if ( !SqlQuery ( m_tParams.m_dQueryPost[i].cstr() ) )
 {
 sphWarn ( "sql_query_post[%d]: error=%s, query=%s",
 i, SqlError(), m_tParams.m_dQueryPost[i].cstr() );
 break;
 }
 SqlDismissResult ();
 }

 m_tDocInfo.m_uDocID = 0; // 0 means legal eof
 return NULL;
 }

 // get him!
 m_tDocInfo.m_uDocID = VerifyID ( sphToDocid ( SqlColumn(0) ) );
 m_uMaxFetchedID = Max ( m_uMaxFetchedID, m_tDocInfo.m_uDocID );
 } while ( !m_tDocInfo.m_uDocID );
`

上面的代码我们可以看到一个很关键的字段m_uDocID,这个字段表示当前doc的id(因此数据库的表设计必须有这个id字段).

读取完毕数据之后，开始处理读取的数据，这里会按照字段来切分，主要是将对应的数据库字段保存到索引fielld

` // split columns into fields and attrs
 for ( int i=0; i<m_iPlainFieldsLength; i++ )
 {
 // get that field
 #if USE_ZLIB
 if ( m_dUnpack[i]!=SPH_UNPACK_NONE )
 {
 DWORD uUnpackedLen = 0;
 m_dFields[i] = (BYTE*) SqlUnpackColumn ( i, uUnpackedLen, m_dUnpack[i] );
 m_dFieldLengths[i] = (int)uUnpackedLen;
 continue;
 }
 #endif
 m_dFields[i] = (BYTE*) SqlColumn ( m_tSchema.m_dFields[i].m_iIndex );
 m_dFieldLengths[i] = SqlColumnLength ( m_tSchema.m_dFields[i].m_iIndex );
 }
`

紧接着就是处理attribute，后续我们会详细介绍attribute，现在我们只需要知道它是一个类似二级索引的东西(不进入全文索引).

` switch ( tAttr.m_eAttrType )
 {
 case SPH_ATTR_STRING:
 case SPH_ATTR_JSON:
 // memorize string, fixup NULLs
 m_dStrAttrs[i] = SqlColumn ( tAttr.m_iIndex );
 if ( !m_dStrAttrs[i].cstr() )
 m_dStrAttrs[i] = "";

 m_tDocInfo.SetAttr ( tAttr.m_tLocator, 0 );
 break;
..................................
 default:
 // just store as uint by default
 m_tDocInfo.SetAttr ( tAttr.m_tLocator, sphToDword ( SqlColumn ( tAttr.m_iIndex ) ) ); // FIXME? report conversion errors maybe?
 break;
 }
`

然后我们来看Sphinx如何处理得到的数据,核心代码在 RtIndex_t::AddDocument中，这个函数主要是用来分词(IterateHits中)然后保存数据到对应的数据结构,而核心的数据结构是RtAccum_t，也就是最终sphinx在写索引到文件之前，会将数据保存到这个数据结构，这里要注意一般来说sphinx会保存很多数据，然后最后一次性提交给索引引擎来处理.而索引引擎中处理的就是这个数据结构.因此最终会调用RtAccum_t::AddDocument.

这里需要注意两个地方，第一个是m_dAccum这个域，这个域是一个vector，而这个vector里面保存了CSphWordHit这个结构，我们来看这个结构的定义

` struct CSphWordHit
 {
 SphDocID_t m_uDocID; ///< document ID
 SphWordID_t m_uWordID; ///< word ID in current dictionary
 Hitpos_t m_uWordPos; ///< word position in current document
 };
`

可以看到其实这个结构也就是保存了对应分词的信息.

然后我们来看核心代码，这里主要是便利刚才从mysql得到的数据，去重然后保存数据.

` int iHits = 0;
 if ( pHits && pHits->Length() )
 {
 CSphWordHit tLastHit;
 tLastHit.m_uDocID = 0;
 tLastHit.m_uWordID = 0;
 tLastHit.m_uWordPos = 0;

 iHits = pHits->Length();
 m_dAccum.Reserve ( m_dAccum.GetLength()+iHits );
 for ( const CSphWordHit * pHit = pHits->First(); pHit<=pHits->Last(); pHit++ )
 {
 // ignore duplicate hits
 if ( pHit->m_uDocID==tLastHit.m_uDocID && pHit->m_uWordID==tLastHit.m_uWordID && pHit->m_uWordPos==tLastHit.m_uWordPos )
 continue;

 // update field lengths
 if ( pFieldLens && HITMAN::GetField ( pHit->m_uWordPos )!=HITMAN::GetField ( tLastHit.m_uWordPos ) )
 pFieldLens [ HITMAN::GetField ( tLastHit.m_uWordPos ) ] = HITMAN::GetPos ( tLastHit.m_uWordPos );

 // accumulate
 m_dAccum.Add ( *pHit );
 tLastHit = *pHit;
 }
 if ( pFieldLens )
 pFieldLens [ HITMAN::GetField ( tLastHit.m_uWordPos ) ] = HITMAN::GetPos ( tLastHit.m_uWordPos );
 }
`

做完上面这些事情之后，就需要提交数据给索引处理引擎了，这里核心的代码都是在RtIndex_t::Commit中.

这个函数主要做两个事情，第一个提取出前面我们构造好的RtAccum_t,然后对于所有的doc进行排序，创建segment，也就是对应的索引块(ram chunk)，最后调用CommitReplayable来提交ram chunk到磁盘.

其实可以这么理解，保存在内存中的索引也就是segment,然后当内存的大小到达限制后就会刷新内存中的索引到磁盘.

` void RtIndex_t::Commit ( int * pDeleted, ISphRtAccum * pAccExt )
 {
 assert ( g_bRTChangesAllowed );
 MEMORY ( MEM_INDEX_RT );

 RtAccum_t * pAcc = AcquireAccum ( NULL, pAccExt, true );
 if ( !pAcc )
 return;

 ...................................
 pAcc->Sort();

 RtSegment_t * pNewSeg = pAcc->CreateSegment ( m_tSchema.GetRowSize(), m_iWordsCheckpoint );
 .............................................

 // now on to the stuff that needs locking and recovery
 CommitReplayable ( pNewSeg, pAcc->m_dAccumKlist, pDeleted );
 ......................................
 }
`

然后我们来看RtAccum_t::CreateSegment函数，这个函数用来将分词好的数据保存到ram chunk，这里需要注意两个数据结构分别是RtDoc_t和RtWord_t,这两个数据结构分别表示doc信息和分词信息.

结构很简单，后面的注释都很详细

` template < typename DOCID = SphDocID_t >
 struct RtDoc_T
 {
 DOCID m_uDocID; ///< my document id
 DWORD m_uDocFields; ///< fields mask
 DWORD m_uHits; ///< hit count
 DWORD m_uHit; ///< either index into segment hits, or the only hit itself (if hit count is 1)
 };

 template < typename WORDID=SphWordID_t >
 struct RtWord_T
 {
 union
 {
 WORDID m_uWordID; ///< my keyword id
 const BYTE * m_sWord;
 };
 DWORD m_uDocs; ///< document count (for stats and/or BM25)
 DWORD m_uHits; ///< hit count (for stats and/or BM25)
 DWORD m_uDoc; ///< index into segment docs
 };
`

然后来看代码，首先是初始化对应的写结构,可以看到都是会写到我们创建好的segment中.

` RtDocWriter_t tOutDoc ( pSeg );
 RtWordWriter_t tOutWord ( pSeg, m_bKeywordDict, iWordsCheckpoint );
 RtHitWriter_t tOutHit ( pSeg );
`

然后就是写数据了，这里主要是做一个聚合，也就是将相同的keyword对应的属性聚合起来.

` ARRAY_FOREACH ( i, m_dAccum )
 {
 .......................................
 // new keyword; flush current keyword
 if ( tHit.m_uWordID!=tWord.m_uWordID )
 {
 tOutDoc.ZipRestart ();
 if ( tWord.m_uWordID )
 {
 if ( m_bKeywordDict )
 {
 const BYTE * pPackedWord = pPacketBase + tWord.m_uWordID;
 assert ( pPackedWord[0] && pPackedWord[0]+1<m_pDictRt->GetPackedLen() );
 tWord.m_sWord = pPackedWord;
 }
 tOutWord.ZipWord ( tWord );
 }

 tWord.m_uWordID = tHit.m_uWordID;
 tWord.m_uDocs = 0;
 tWord.m_uHits = 0;
 tWord.m_uDoc = tOutDoc.ZipDocPtr();
 uPrevHit = EMPTY_HIT;
 }
 ..................
 }
`

这次就分析到这里，下次我们将会分析最核心的部分就是Sphinx如何刷新数据到磁盘.

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)