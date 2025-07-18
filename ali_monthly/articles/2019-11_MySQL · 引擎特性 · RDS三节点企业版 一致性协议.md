# MySQL · 引擎特性 · RDS三节点企业版 一致性协议

**Date:** 2019/11
**Source:** http://mysql.taobao.org/monthly/2019/11/06/
**Images:** 8 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2019 / 11
 ](/monthly/2019/11)

 * 当期文章

 MySQL · 最佳实践 · 今天你并行了吗？---洞察PolarDB 8.0之并行查询
* MySQL · 新特征 · MySQL 哈希连接实现介绍
* MySQL · 最佳实践 · 性能分析的大杀器—Optimizer trace
* PgSQL · 未来特性调研 · TDE
* Database · 理论基础 · Multi-ART
* MySQL · 引擎特性 · RDS三节点企业版 一致性协议
* MySQL · 引擎特性 · RDS三节点企业版 Learner 只读实例

 ## MySQL · 引擎特性 · RDS三节点企业版 一致性协议 
 Author: 甄平 

 本文介绍三节点企业版如何在AliSQL的基础上集成X-Paxos一致性协议，来实现高可用强一致的特性。

## 背景介绍

RDS 5.7三节点企业版是孵化于阿里巴巴集团内部的高可用、强一致，支持全球部署的数据库产品。该产品从2017年在阿里巴巴集团自有业务推广，平稳支持多年双十一。经过2年的内部打磨，该版本在2019年7月正式上线公有云售卖。相比RDS 5.6三节点版本，我们对内核进行的全新的设计，特别是一致性协议方面。

三节点企业版的核心是一致性协议。在5.7的版本，我们把阿里巴巴自研的一致性协议库X-Paxos集成到AliSQL中，在100%兼容MySQL的基础上，实现了数据库的自动选主，日志同步，数据强一致，在线配置变更等功能。X-Paxos采用了unique proposer的Multi-Paxos实现方案，同时又做了很多创新性的功能和性能优化，是一个更具生产环境实用意义的一致性协议。

## 节点角色

熟悉Paxos论文的人都知道，整个Paxos算法中包含三种角色：Proposer、Accepter和Learner。在X-Paxos中，节点的角色分为四类：

 角色
 同步日志
 投票权
 状态机回放
 读写状态
 Paxos角色映射

 Leader
 1
 1
 1
 rw
 Proposer / Accepter / Learner

 Follower
 1
 1
 1
 ro
 Proposer / Accepter / Learner

 Logger
 1
 1
 0
 -
 Accepter / Learner

 Learner
 1
 0
 1
 ro
 Learner

整个一致性协议的持久存储分两块：日志和状态机。日志代表了对状态机的更新操作，状态机存放了外部业务读写的实际数据。

Leader是集群中唯一可读写的节点。它给集群所有节点发送新写入的日志，达成多数派后允许提交，并回放到本地的状态机。众所周知，标准的Paxos存在活锁的问题（livelock），即两个Proposer交替发起Prepare请求，导致每一轮Prepare的Accept请求都失败，提案编号不断递增，陷入死循环永远达不成一致。因此业界的最佳实践是选取一个主Proposer，来保证算法的活性。另一方面，针对数据库场景，只允许主Proposer发起提案，简化了事务的冲突处理，保证了高性能。这个主Proposer被称之为Leader。

Follower是灾备节点，用于收集Leader发送的日志，并负责把达成多数派的日志回放到状态机。当Leader发生故障时，集群中的剩余节点会选一个新的Follower升级成Leader接受读写请求。

Logger是一种特殊类型的Follower，不对外提供服务。Logger做两件事：存储最新的日志用于Leader的多数派判定；选主阶段行使投票权。Logger不回放状态机，会定期清理老旧的日志，占用极少的计算和存储资源。因此，基于Leader/Follower/Logger的部署方式，三节点相比双节点高可用版，只额外增加很少的成本。

Learner没有投票权，不参加多数派的计算，仅从Leader同步已提交的日志，并回放到状态机。在实际使用中，我们把Learner作为只读副本，用于应用层的读写分离。此外，X-Paxos支持Learner和Follower之间的节点变更，基于这个功能可以实现故障节点的迁移和替换。

![](.img/0245d944100c_2019-11-zhenpin-paxos-role.png)

## 集群管理

三节点企业版支持丰富的集群变更和配置管理功能，列举如下：

* Leader节点主动切换
* 加减Learner节点
* Follower降级成Learner、Learner升级为Follower
* 修改节点的选举权重
* 修改Learner节点的复制拓扑
* 修改日志发包的配置模式（Pipelining、Batching、压缩、加密）
* 高性能异步模式

## 日志

首先回顾MySQL双节点高可用版本的复制模式。其中Master节点负责写入binary log，并提交事务。Slave节点通过IO线程从Master节点发起dump协议拉取binary log，并存储到本地的relay log中。最后由Slave节点的SQL线程负责回放relay log。

双节点复制模式可以用下图表示：

![](.img/ef669278a4d3_2019-11-zhenpin-ms.png)

一般情况下，Slave节点还需要开启log-slave-updates来保证从库也可以为下游提供日志同步，因此Slave线程除了relay log，还会有一份冗余的binary log。

三节点企业版创新性的整合了binary log和relay log，实现了统一的consensus log，节省了日志存储的成本。当某个节点是Leader的时候，consensus log扮演了binary log的角色；同理当某个节点被切换成Follower/Learner时，consensus log扮演了relay log的角色。X-Paxos一致性协议层接管consensus log的同步逻辑，同时提供对外的接口来实现日志写入和状态机回放。新的consensus log基于一致性协议和State Machine Replication理论，保证了多个节点之间的数据一致性。此外，三节点企业版日志的实现遵循了MySQL binary log的标准，可以无缝兼容aliyun DTS、Canal等业内常用的binlog增量订阅工具。

三节点复制模式如下图所示：

![](.img/2939df839363_2019-11-zhenpin-lfl.png)

## 状态机

三节点企业版的状态机实现改造了MySQL原有事务提交的流程。

MySQL组提交（Group Commit）相关的技术文章网上有很多，原有Group Commit分为三个阶段：flush stage、sync stage、commit stage。对于Leader节点，三节点企业版修改了其中commit stage的实现方式。所有进入commit stage的事务会被统一推送到一个异步队列中，进入quorum决议的判定阶段，等待事务日志同步到多数节点上，满足quorum条件的事务才允许commit。另外，Leader上consensus log的本地写入和日志同步可以并行执行，保证了高性能。

对于Follower节点，SQL线程读取consensus log，开始等待Leader的通知。Leader会定期同步给Follower每一条日志的提交状态，达成多数派的日志会被分发给worker线程并行执行。

Learner节点相对Follower的逻辑更加简单，一致性协议保证了它不会接收到未提交的日志，SQL线程不用等待任何条件，只需分发最新的日志给worker线程即可。此外，三节点企业版使用特殊版本的Xtrabackup进行实例备份和恢复。我们基于X-Paxos的snapshot接口改进了Xtrabackup，支持创建带有一致性位点的物理备份快照，可以十分快捷的孵化一个全新的Learner节点，并加入到集群中提供读能力的扩展。

![](.img/a1c18ab87405_2019-11-zhenpin-commit.png)

## 部署模式

### 同城三副本

同城三副本是公有云上默认的部署模式。比较传统的双机房主备高可用版，三节点在满足高可用强一致特性的基础上，基本不增加存储成本：

* 三节点单机房不可用场景下数据0丢失，秒级切换，主备有丢数据的风险；
* 三节点和主备都只存储两份状态机数据；三节点存储三份consensus log日志，而主备版本常态化有两份binary log和一份relay log，总量基本持平。

![](.img/34a121c7e701_2019-11-zhenpin-3node.png)

### 跨域五副本

对于跨域容灾场景，我们推荐跨域五副本的架构。相比简单的搭建跨域三副本，五副本有以下优势：

* 和跨域三副本一样有Region级别的容灾能力，链路上仅有少量性能损耗；
* 通过增加一个Follower和Logger节点，实现单机房故障下的同城容灾，对用户端友好；
* 通过X-Paxos的选举权重功能，可实现定制化的region切换顺序。

![](.img/71206d3f973d_2019-11-zhenpin-5node.png)

## 总结

随着当前互联网的发展，云上客户对数据安全越来越重视，大量行业对数据存储有跨机房跨地域的需求。RDS 5.7三节点企业版是基于阿里巴巴内部自研技术的沉淀，针对数据质量要求较高的用户，在云上推出的数据库解决方案。此外，对于RDS 5.7高可用版的老用户，也支持一键升级三节点。

购买方式：

![](.img/b185de0e122b_2019-11-zhenpin-buy.png)

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)