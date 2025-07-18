# MySQL · 物理备份 · Percona XtraBackup 备份原理

**Date:** 2016/03
**Source:** http://mysql.taobao.org/monthly/2016/03/07/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2016 / 03
 ](/monthly/2016/03)

 * 当期文章

 MySQL · TokuDB · 事务子系统和 MVCC 实现
* MongoDB · 特性分析 · MMAPv1 存储引擎原理
* PgSQL · 源码分析 · 优化器逻辑推理
* SQLServer · BUG分析 · Agent 链接泄露分析
* Redis · 特性分析 · AOF Rewrite 分析
* MySQL · BUG分析 · Rename table 死锁分析
* MySQL · 物理备份 · Percona XtraBackup 备份原理
* GPDB · 特性分析· GreenPlum FTS 机制
* MySQL · 答疑解惑 · 备库Seconds_Behind_Master计算
* MySQL · 答疑解惑 · MySQL 锁问题最佳实践

 ## MySQL · 物理备份 · Percona XtraBackup 备份原理 
 Author: xiangluo 

 ## 前言

[Percona XtraBackup](https://www.percona.com/software/mysql-database/percona-xtrabackup)（简称PXB）是 Percona 公司开发的一个用于 MySQL 数据库**物理热备**的备份工具，支持 MySQl（Oracle）、Percona Server 和 MariaDB，并且全部开源，真可谓是业界良心。我们 RDS MySQL 的物理备份就是基于这个工具做的。

项目的 blueprint 和 bug 讨论放在 [Launchpad](https://launchpad.net/percona-xtrabackup)，代码之前也放在 Launchpad，现在已经迁移到 [Github](https://github.com/percona/percona-xtrabackup) 啦，项目更新发布非常快，感兴趣的可以关注 :-)

本文会介绍下备份工具的工作原理，希望对大家有所帮助。

## 工具集

软件包安装完后一共有4个可执行文件，如下：

`usr
├── bin
│ ├── innobackupex
│ ├── xbcrypt
│ ├── xbstream
│ └── xtrabackup
`

其中最主要的是 `innobackupex` 和 `xtrabackup`，前者是一个 perl 脚本，后者是 C/C++ 编译的二进制。

`xtrabackup` 是用来备份 InnoDB 表的，不能备份非 InnoDB 表，和 mysqld server 没有交互；`innobackupex` 脚本用来备份非 InnoDB 表，同时会调用 `xtrabackup` 命令来备份 InnoDB 表，还会和 mysqld server 发送命令进行交互，如加读锁（FTWRL）、获取位点（SHOW SLAVE STATUS）等。简单来说，`innobackupex` 在 `xtrabackup` 之上做了一层封装。

一般情况下，我们是希望能备份 MyISAM 表的，虽然我们可能自己不用 MyISAM 表，但是 mysql 库下的系统表是 MyISAM 的，因此备份基本都通过 `innobackupex` 命令进行；另外一个原因是我们可能需要保存位点信息。

另外2个工具相对小众些，`xbcrypt` 是加解密用的；`xbstream` 类似于tar，是 Percona 自己实现的一种支持并发写的流文件格式。两都在备份和解压时都会用到（如果备份用了加密和并发）。

本文的介绍的主角是 `innobackupex` 和 `xtrabackup`。

## 原理

### 通信方式

2个工具之间的交互和协调是通过控制文件的创建和删除来实现的，主要文件有：

* xtrabackup_suspended_1
* xtrabackup_suspended_2
* xtrabackup_log_copied

举个栗子，我们来看备份时 xtrabackup_suspended_2 是怎么来协调2个工具进程的

1. `innobackupex` 在启动 `xtrabackup` 进程后，会一直等 `xtrabackup` 备份完 InnoDB 文件，方式就是等待 xtrabackup_suspended_2 这个文件被创建出来；
2. `xtrabackup` 在备完 InnoDB 数据后，就在指定目录下创建出这个文件，然后等这个文件被 `innobackupex` 删除；
3. `innobackupex` 检测到文件 xtrabackup_suspended_2 被创建出来后，就继续往下走；
4. `innobackupex` 在备份完非 InnoDB 表后，删除 xtrabackup_suspended_2 这个文件，这样就通知 `xtrabackup` 可以继续了，然后等 xtrabackup_log_copied 被创建；
5. `xtrabackup` 检测到 xtrabackup_suspended_2 文件删除后，就可以继续往下了。

是不是感觉有点不可思议，通过文件是否存在来控制进程，这种方式非常的不靠谱，因为非常容易被外部干扰，比如文件被别人误删掉，或者2个正在跑的备份控制文件误放在同一个目录下，就等着备份乱掉吧，但是 Percona 就是这么干的。

之所以这么搞，估计主要是因为 perl 和 C 二进制2个进程，没有既好用又方便的通信方式，搞个协议啥的太麻烦了。但是官方也觉得这种方式不靠谱，11年就搞了个 [blueprint](https://blueprints.launchpad.net/percona-xtrabackup/+spec/rewrite-innobackupex-in-c) 要用C重写 `innobackupex`，终于在[2.3 版本](https://www.percona.com/blog/2015/05/20/percona-xtrabackup-2-3-1-beta1-is-now-available/)实现了，`innobackupex` 功能全部集成到 `xtrabackup` 里面，只有一个 binary，另外为了使用上的兼容考虑，`innobackupex` 作为 `xtrabackup` 的一个软链。对于二次开发来说，2.3 摆脱了之前2个进程协作的负担，架构上明显要好于之前版本。考虑到 perl + C 这种架构的长期存在，大多数读者朋友也基本用的2.3之前版本，本文的介绍也是基于老的架构（2.2版本），但是原理和2.3是一样的，只是实现上的差别。

### 备份过程

整个备份过程如下图：

1. `innobackupex` 在启动后，会先 fork 一个进程，启动 `xtrabackup`进程，然后就等待 `xtrabackup` 备份完 ibd 数据文件；
2. `xtrabackup` 在备份 InnoDB 相关数据时，是有2种线程的，1种是 redo 拷贝线程，负责拷贝 redo 文件，1种是 ibd 拷贝线程，负责拷贝 ibd 文件；redo 拷贝线程只有一个，在 ibd 拷贝线程之前启动，在 ibd 线程结束后结束。`xtrabackup` 进程开始执行后，先启动 redo 拷贝线程，从最新的 checkpoint 点开始顺序拷贝 redo 日志；然后再启动 ibd 数据拷贝线程，在 `xtrabackup` 拷贝 ibd 过程中，`innobackupex` 进程一直处于等待状态（等待文件被创建）。
3. `xtrabackup` 拷贝完成idb后，通知 `innobackupex`（通过创建文件），同时自己进入等待（redo 线程仍然继续拷贝）;
4. `innobackupex` 收到 `xtrabackup` 通知后，执行`FLUSH TABLES WITH READ LOCK` (FTWRL)，取得一致性位点，然后开始备份非 InnoDB 文件（包括 frm、MYD、MYI、CSV、opt、par等）。拷贝非 InnoDB 文件过程中，因为数据库处于全局只读状态，如果在业务的主库备份的话，要特别小心，非 InnoDB 表（主要是MyISAM）比较多的话整库只读时间就会比较长，这个影响一定要评估到。
5. 当 `innobackupex` 拷贝完所有非 InnoDB 表文件后，通知 `xtrabackup`（通过删文件） ，同时自己进入等待（等待另一个文件被创建）；
6. `xtrabackup` 收到 `innobackupex` 备份完非 InnoDB 通知后，就停止 redo 拷贝线程，然后通知 `innobackupex` redo log 拷贝完成（通过创建文件）；
7. `innobackupex` 收到 redo 备份完成通知后，就开始解锁，执行 `UNLOCK TABLES`；
8. 最后 `innobackupex` 和 `xtrabackup` 进程各自完成收尾工作，如资源的释放、写备份元数据信息等，`innobackupex` 等待 `xtrabackup` 子进程结束后退出。

在上面描述的文件拷贝，都是备份进程直接通过操作系统读取数据文件的，只在执行 SQL 命令时和数据库有交互，基本不影响数据库的运行，在备份非 InnoDB 时会有一段时间只读（如果没有MyISAM表的话，只读时间在几秒左右），在备份 InnoDB 数据文件时，对数据库完全没有影响，是真正的热备。

InnoDB 和非 InnoDB 文件的备份都是通过拷贝文件来做的，但是实现的方式不同，前者是以page为粒度做的(`xtrabackup`)，后者是 cp 或者 tar 命令(`innobackupex`)，`xtrabackup` 在读取每个page时会校验 checksum 值，保证数据块是一致的，而 `innobackupex` 在 cp MyISAM 文件时已经做了flush（FTWRL），磁盘上的文件也是完整的，所以最终备份集里的数据文件都是写入完整的。

#### 增量备份

PXB 是支持增量备份的，但是只能对 InnoDB 做增量，InnoDB 每个 page 有个 LSN 号，LSN 是全局递增的，page 被更改时会记录当前的 LSN 号，page中的 LSN 越大，说明当前page越新（最近被更新）。每次备份会记录当前备份到的LSN（xtrabackup_checkpoints 文件中），增量备份就是只拷贝LSN大于上次备份的page，比上次备份小的跳过，每个 ibd 文件最终备份出来的是增量 delta 文件。

MyISAM 是没有增量的机制的，每次增量备份都是全部拷贝的。

增量备份过程和全量备份一样，只是在 ibd 文件拷贝上有不同。

### 恢复过程

如果看恢复备份集的日志，会发现和 mysqld 启动时非常相似，其实备份集的恢复就是类似 mysqld crash后，做一次 crash recover。

恢复的目的是把备份集中的数据恢复到一个一致性位点，所谓一致就是指原数据库某一时间点各引擎数据的状态，比如 MyISAM 中的数据对应的是 15:00 时间点的，InnoDB 中的数据对应的是 15:20 的，这种状态的数据就是不一致的。PXB 备份集对应的一致点，就是备份时FTWRL的时间点，恢复出来的数据，就对应原数据库FTWRL时的状态。

因为备份时 FTWRL 后，数据库是处于只读的，非 InnoDB 数据是在持有全局读锁情况下拷贝的，所以非 InnoDB 数据本身就对应 FTWRL 时间点；InnoDB 的 ibd 文件拷贝是在 FTWRL 前做的，拷贝出来的不同 ibd 文件最后更新时间点是不一样的，这种状态的 ibd 文件是不能直接用的，但是 redo log 是从备份开始一直持续拷贝的，最后的 redo 日志点是在持有 FTWRL 后取得的，所以最终通过 redo 应用后的 ibd 数据时间点也是和 FTWRL 一致的。

所以恢复过程只涉及 InnoDB 文件的恢复，非 InnoDB 数据是不动的。备份恢复完成后，就可以把数据文件拷贝到对应的目录，然后通过mysqld来启动了。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)