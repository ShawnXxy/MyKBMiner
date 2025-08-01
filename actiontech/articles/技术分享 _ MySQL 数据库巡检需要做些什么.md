# 技术分享 | MySQL 数据库巡检需要做些什么？

**原文链接**: https://opensource.actionsky.com/%e6%8a%80%e6%9c%af%e5%88%86%e4%ba%ab-mysql-%e6%95%b0%e6%8d%ae%e5%ba%93%e5%b7%a1%e6%a3%80%e9%9c%80%e8%a6%81%e5%81%9a%e4%ba%9b%e4%bb%80%e4%b9%88%ef%bc%9f/
**分类**: MySQL 新特性
**发布时间**: 2022-01-12T23:44:01-08:00

---

作者：陈俊聪
中移信息基础平台部数据库团队成员，主要负责 MySQL、TiDB、Redis、clickhouse 等开源数据库的维护工作。
本文来源：原创投稿
*爱可生开源社区出品，原创内容未经授权不得随意使用，转载请联系小编并注明来源。
接触 MySQL 数据库 7 年了，专职做 MySQL 数据库运维工作也有 6 个年头了，这 6 年来呆了三家公司，做过很多次数据库巡检工作，从一开始是网上下载个巡检模板应付工作，草草了事，到后来使用公司专门的数据库巡检模板做巡检，经验越来越多，逐渐形成了自己的套路，或称为方法论。今天，想写下这篇文章，把我的这些个人的经验和想法总结下来，也为了证明，即使巡检那么小的一件事，只要你愿意，也能得出个最佳实践。
> 
最佳实践的意义是什么？并不是所有人都对 MySQL 那么熟悉，最佳实践以文档的形式沉淀下来，可以有效避免犯错，也能最大限度的避免因人员流失而带来的巡检质量降低。
我认为巡检有好几种分类，他们的侧重点各不相同，以下是我的分类。
## 按巡检方式来划分
按巡检方式来划分的话，巡检分为人肉巡检、脚本化巡检、平台化巡检。
人肉巡检也称为手工巡检，在管理的数据库服务器数量少，巡检频率低的情况下，是可以这么做。上去服务器执行 `df -h` 、`mysql -u` 等命令检查服务器、数据库的运行状况，查看监控(zabbix、Prometheus 等)来分析。这种方法除了不效率之外，还有一个缺点，就是非常依赖巡检人员的技能水平，不同的工程师去巡检，可能结论会不一样。
脚本化巡检，这个阶段其实也就是把巡检的命令打包做成一个脚本，工程师登录服务器一台台执行脚本，当然了，如果公司允许的话，可以采用 ansible 等批量运维工具，批量跑脚本巡检，脚本生成 html 报表或 csv 格式的数据，供您分析和汇总。有了这个方法后，我们的经验是，一个人可以在一天内，轻松地在 2000 个实例上做巡检并完成巡检报告。这个方法相对人肉巡检方法在效率上有了质的飞跃。他仍然有缺点，就是他仍然需要耗费人力，仍然需要部分依赖巡检人员的技能水平去分析报告和汇总。
平台化巡检，我们的数据库管理平台解决租户的三大核心诉求（高可用管理、备份恢复管理、监控告警管理）之外，还提供了平台化巡检功能，定时触发巡检生成巡检报告，并且利用融合了我们运维经验的健康度评分模型算法，去给实例打分，对于不满 60 分的实例我们需要马上关注，自动通知数据库管理员，并且自动和智能地分析存在的问题。有了它，极大地减少数据库运维人员登录生产环境的次数，并且不再依赖巡检人员的技能水平，做到巡检的绝对标准化。
## 按时间维度来划分
按时间维度来划分，数据库巡检分为日常巡检和节前巡检。
日常巡检，就是每天都做的巡检，这个巡检最简单办法就是看监控，主要留意 warning 或以上级别的告警，监控基本上可以覆盖百分之 95% 以上的日常巡检需求。
节前巡检，指的是特殊日期的巡检工作，要比日常巡检要深度。狭义上指的是国家法定节假日放假前（例如五一、端午、清明、中秋、国庆、过年等等）的巡检。广义上也包含了双十一等重大活动前的保障级巡检。节前巡检，对于我所在的数据库运维团队来说，主要关注的就是，过节时可能会有部分业务有流量高峰，并且过节时由于人员放假，人员不足，需要提前巡检发现未来的问题，提前解决。而作为运维，主要关注的是数据库可用性，所以节前巡检的检查核心如下:
- 
系统层面
CPU
- 
RAM
- 
磁盘空间
- 
应用层面( MySQL 实例)
实例状态
- 
高可用状态
- 
复制状态
- 
监控状态
- 
VIP 状态
这里我展开说明一下。
CPU ，通过监控，去寻找高 CPU 水位的实例，确认是否和日常监控曲线一致，判断是否有异常，是否有优化的需要。
RAM，通过监控找到 RAM 使用率超过 80% 的实例，检查是否有内存不够用的问题，该扩容的扩容，防止 OOM。这里一并查找是否有实例使用到 swap，如果内存充裕而使用到 swap，那很大概率是因为未正确设置 numa 或 vm.swappiness 导致的。
磁盘空间，这个是节前巡检中的重点，平时磁盘使用率的告警线是 80%，而巡检时我们应该去获取大于 70% 磁盘使用率的实例，提前扩容，避免放个春节(7天) 磁盘有告警的风险，这个时候回家过年还加班大家都不好受。
实例状态，一般来说就是检查 mysqld 的存活，有条件的话可以分析其是否健康。(至于怎么判断其是否健康，这里不扩展了)
高可用状态，通过巡检证明数据库是&#8221;可切换&#8221;状态。例如 MHA 架构的话，要检查这三个脚本执行的结果。
`masterha_check_ssh --conf=/etc/masterha/app.cnf
masterha_check_repl --conf=/etc/masterha/app.cnf
masterha_check_status --conf=/etc/masterha/app.cnf
`
复制状态，实际上上述高可用状态检查，一般也会检查复制状态。但有一些复制不在高可用范畴，所以这里要多嘴提一下。异步复制、半同步复制、延迟复制、双向复制、级联复制状态的检查，看是否正常，容灾的 DTS 复制是否正常，DTS 的高可用是否正常等等。
VIP 状态，我们有一些实例是有双网络冗余链路的，这些实例会有双 VIP 。如果这时掉了那个冗余网络链路的 VIP 对业务是没有感知的。所以我们应该有一个巡检机制，这个可以开发一个探活任务去定时探测和告警。
## 按巡检程度来划分
按巡检程度来划分，分为普通巡检和深度巡检。上述的节前巡检虽然比日常巡检要深度，但也只是普通巡检，不能算是深度巡检。那什么样的巡检才算是深度巡检呢？
**节前巡检，我们的关注点在于运维，在于数据库可用性，那么深度巡检必然关注点要向外扩展，要关注用户体验，要关注性能，这样的巡检就是深度巡检。**
所以深度巡检是解决什么问题呢？深度巡检的目的是对日常巡检和节前巡检的补充，让数据库在未来日子里不单只可用，并且可靠和跑得更快。
**我认为，深度巡检 = 可用性巡检 + 可靠性巡检 + 性能巡检 + 分析和建议**
## 可用性巡检
在前面提及的节前巡检，已经大量检查了数据库的可用性，但那些都是从运维角度、从服务、从实例级别来衡量的，从应用角度、从业务角度，其实这个可用性检查是可以扩展的，例如在深度巡检里，我们会检查租户每张表的自增键使用情况，租户常见的自增键类型是 int unsigned 和 int signed，前者是无符号 int 类型，范围是(-2147483648,2147483647)，后者是有符号 int 类型 (0,4294967295)，开发人员在建表的时候更常见是直接定义 int，没有规定是 unsigned 还是 signed，那么默认就是 unsigned 了，而没有使用更理想的  int signed 值，这样自增键可用的范围就会少了一倍。再加上，自增键在插入时并不是连续的，和你插入的方式，参数设置(innodb_autoinc_lock_mode、auto_increment_increment)有关，自增键在插入前就会分配，所以一旦插入失败事务回滚，这个自增 id 也会自然浪费掉。所以 int unsigned 的上限虽然看起来很高，有 21 亿之多，但由于刚才说的原因，很有可能你的 table 里只有 10 亿行数据时，自增键就满了。自增键满了，你这张表就不可写入了，这就是业务层面的不可用。
我们在生产实践中就遇到过这样的情况，某业务是负责某商城交易账单数据分析的，其账单的日志表，每天入库入表的记录数一般为 500 万条，高峰时可以达到 900 万条以上，当时这张表采用的是 int unsigned 自增 id 作为主键，在业务上线不到 9 个月自增主键就用完了。解决办法就是修改自增利类型，从  int unsigned 修改为 bigint signed，我们知道 MySQL 修改主键列类型是锁表的，只能读不能写，所以当时这个业务受损了，DDL 花了 6 个小时。
所以深度巡检，需要对这些情况，做可用性巡检的扩展，更多的可用性巡检，读者可以自行补充。
## 可靠性巡检
在说性能巡检之前，我想补充一下，可靠性巡检，前面提到的节前巡检有大量的检查可用性了，但可用性是否等于可靠性呢，这里有很多人会混淆，他们并不相等。可用性指的是 Availability，一般是高可用要解决的问题，而可靠性指的是 Reliability ，在数据库里一般指的是数据不错、不丢和数据副本的一致性。
节前巡检，已经包含了不少数据库可靠性检查，例如高可用检查中的&#8221;可切换&#8221;检查，复制状态检查。但这里并不是万无一失的，在这里我提出深度巡检需要做 &#8220;核心参数检查&#8221;。
这里的&#8221;核心参数检查&#8221;包括三方面
- 
检查数据库里的参数是否满足我们的交维规范要求的核心参数列表
- 
检查主备数据库参数是否一致
- 
检查数据库运行参数和配置文件(my.cnf)参数是否一致
检查数据库里的参数是否满足我们的交维规范要求的核心参数列表，这个其实是历史遗留问题，因为我们大多数数据库本身不来自于我们的部署交付，而是各业务部门交接给我们的，对于这些新交接的实例，务必检查核心参数，才能保证数据不错不丢和主从数据一致性，相关参数包含并不仅限于以下这些:
`binlog_format = row
binlog_row_image = full
gtid_mode = on
enforce-gtid-consistency = on
innodb_doublewrite = on
innodb_flush_log_at_trx_commit = 1
log_bin = mysql-bin
master_info_repository = table
sync_binlog = 1
... 
`
实际上我们检查的参数高达 80 个。
检查主备数据库参数是否一致，这个主要是避免主备切换后，有使用上的不一致。这里也会检查一些务必设置不一致的参数，例如 server_id ，反正目的只有一个，就是检查主备的参数，保证他们正常。
检查数据库运行参数和配置文件(my.cnf)参数是否一致。很多人以为持久化的配置文件一定会和运行参数一致，这个没必要检查，这就错了，在 MySQL 5.7 或之前，没有办法修改参数同时持久化配置文件，所以修改参数通常都是分两步，先在数据库里 set global 参数=值，然后登陆服务器修改 my.cnf 配置文件，因不是原子操作，那么运维人员就有犯错的可能，千万不要相信人，人总是会犯错的。之前我们就发生过好几次运行参数和持久化配置文件不一致产生的故障。例如，动态修改 MySQL 的 innodb_buffer_pool_size = 128G，然后忘记持久化到配置文件了。当时数据库发生了crash，之后被高可用组件拉回 mysqld 实例，发现性能很差，这个排查了半天，居然是 innodb_buffer_pool_size 被还原了默认值 128M ！
还有一个案例，在 mysql 5.6 年代，当时硬件性能不行加上没有好的并行复制技术，从库容易因为 io 瓶颈而复制延迟，临时解决方法是从库设置 sync_binlog=0、innodb_flush_log_at_trx_commit = 0 来追延迟，待延迟追平后，修改回 &#8220;双1&#8243;。这个时候 DBA 很容易忘记去执行修改回 &#8220;双1&#8243;操作。如果这个时候有个数据库实例级故障，造成主从切换，那么这个时候就有丢失数据的风险了。
另外，某些租户是持有 super 权限，可以修改数据库的参数，但他们是没有服务器权限，如果这些租户修改的参数涉及了我们认为的核心参数，造成这个核心参数的运行参数和配置文件(my.cnf)参数不一致，那就有可能埋了雷，后续引发数据库可靠性甚至是可用性问题了。
&#8220;核心参数检查&#8221; 是可靠性巡检的一个例子，更多的可靠性巡检，读者可以自行补充。
## 性能巡检
性能巡检上，就有很多细小的项目了，我这里介绍一些常见的。
**1、是否存在没有主键的表。**
MySQL 的玩法就是需要有主键，最好是业务无关的 int signed 自增主键，具体为什么请出门右拐看 &#8220;开发规范&#8221;，他是如何影响性能的，网上有大量的文章，这里我就没必要过多赘述了。
**2、SQL 性能优化**
首先，巡检报告里，可以列出 top 10 慢查询，让租户去优化 SQL。
其次，在报告里可以抓取提供一些执行全表扫描次数 TOP 30 的 SQL 给租户，因为有些SQL 他的执行计划本身其实就有问题的，这些 SQL可能当前跑得很快，但没有评估过数据量增长，当表变得越来越大时，达到一个阀值时，线上可能就会爆发 CPU 100% 的性能问题，成为爆发性杀手级慢查询。
再次，索引方面，可以关注冗余索引、无效索引、索引区分度等信息。冗余索引意思是数据库里有重复的索引，对于a、b、c列的联合索引 idx_a_b_c，他其实同时等于拥有了(a、b、c)、(a、b)、(a) 三个索引的能力，如果这时候你再创建一个 idx_a，idx_a_b 索引，那他们就是冗余索引了，这些索引应该删掉，因为索引是占存储空间的，并且索引不是越多越好，维护索引是有开销的，他影响了 DML 语句的性能，所以用不到的索引就应该删除。同样的，无效索引，就是从未使用过的索引，巡检报告应该把这些索引列出来，开发去评估一下这些索引是否未来都不会用到，用不到就应该删除他们。索引区分度用于评估列的值是否足够分散，值越多越适合建立索引，如果是性别列，只有男女两个值，是不适用创建索引的。区分度越接近 1 ，表示区分度越高；低于 0.1 ，则说明区分度较差，开发者应该重新评估 SQL 语句涉及的字段，选择区分度高的多个字段创建索引。
**3、是否有 MyISAM 存储引擎表**
MyISAM 基本没有好处，我之前写过一篇文章《交维规范讲解系列——为什么我们禁止使用 MyISAM 存储引擎》来说过这个问题。实际上我们不应该还要检查是否存在 MyISAM 表，99% 的场景应该直接禁用他，请与我一样通过参数防止  MyISAM 表的创建，当然还有很多存储引擎我都建议不要用了，innodb 才是永远的王者，参考的参数如下:
`disabled_storage_engines=ARCHIVE,BLACKHOLE,EXAMPLE,FEDERATED,MEMORY,MERGE,NDB,MyISAM`
> 
因为 MySQL 5.7 版本仍然有 10 张元数据表使用 MyISAM 存储引擎，可能会影响数据库的升级，如果 5.7 版本使用上了这个参数，运维人员要注意在做数据库升级变更前先禁用此参数，升级完后加回来，参考 https://mp.weixin.qq.com/s/O9UtGskB3IydkEEMscEo1Q。8.0 版本则无这个问题。
我们的目标是，检查到 MyISAM 表后，尽量进行整改，加上上述这个参数，哈。
**4、TOP 10 大表**
大表在做全表扫描时非常耗费性能，在 DDL 方面的话更是灾难，大于 100G 的表都应该评估一下，为什么会有那么大的表？为什么会放在 MySQL 上，是否可以放到 TiDB 上？ 是否可以拆分为小表，水平拆还是纵向拆？归档，冷热分离？
建议：MySQL 单实例的存储空间大小应控制在 500G，单表行数控制在 1000 万行以内、大小在 30G 以内，单表字段 50 个以内，单表索引 5 个以内。
> 
这是以前的建议，仅供参考。随着硬件的提升，我最新的观点是 MySQL 实例 2T 以内，单表体积 100G 以内我都可以接受。当然了，我是从运维角度考虑，性能角度的话主要是看业务是否能接受。
## 分析和建议
暂时列那么多，也就是抛砖引玉，性能巡检的目的是出具尽量多的数据给租户自行做性能分析，这里有 SQL 相关的，有非 SQL 相关的，至于对这些数据的加工和分析方面，我们的报告主要是对非 SQL 相关的加以文字说明，给出建议，SQL 相关的，我们不是这方面的专家，里面有很多门道有很多小技巧，这个交给租户级别高的开发人员去分析优化。
以上就是我个人对 MySQL 数据库巡检需要做什么的总结，欢迎指正。