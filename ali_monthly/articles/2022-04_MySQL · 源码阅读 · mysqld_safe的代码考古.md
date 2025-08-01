# MySQL · 源码阅读 ·  mysqld_safe的代码考古

**Date:** 2022/04
**Source:** http://mysql.taobao.org/monthly/2022/04/03/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2022 / 04
 ](/monthly/2022/04)

 * 当期文章

 MySQL · 源码阅读 · 数据库的扫描方法
* MariaDB · 功能特性 · 无DDL延迟的主备复制
* MySQL · 源码阅读 · mysqld_safe的代码考古
* MySQL · 源码阅读 · 非阻塞异步C API简介
* MySQL · InnoDB · Instant DDL扩展

 ## MySQL · 源码阅读 · mysqld_safe的代码考古 
 Author: zhenping 

 ## Part 1

mysqld_safe是一个跟随mysql安装包一起发布的bash脚本，源码目录在`scripts/mysqld_safe.sh`。核心功能就是启动mysqld，在mysqld进程故障（比如crash）之后，自动探测并重启实例。参考[官方文档](https://dev.mysql.com/doc/refman/8.0/en/mysqld-safe.html)的说明，mysqld_safe是在Linux部署mysql数据库的推荐方法，执行命令大致如下：

`mysqld_safe --defaults-file=file_name <options> <mysqld_options>
`

运行完之后，在bash上执行ps，能看到有一个mysqld_safe进程和一个mysqld进程，mysqld_safe会自动为mysqld准备一系列的参数，包括my.cnf的地址、basedir、错误日志、端口等。这些都可以从ps的命令行输出中查看到。

## Part 2

mysqld_safe目前有1000多行，不过核心逻辑就200行，都是围绕mysqld进程和$pid_file展开。$pid_file存放在my.cnf中配置的pid-file路径上，是一个普通的文本文件，里面存放了创建者的pid，也就是对应的mysqld进程。mysqld_safe依赖pid精确判断是否需要重启。
围绕涉及到的各个文件操作，简化版的mysqld_safe逻辑可以描述如下（参考8.0.28）：

`准备一系列参数和路径，包括最后要拼接在mysqld命令后面的defaults-file、basedir、pid-file、socket等参数
if ($pid_file文件存在) {
 if (pid对应的进程存在 && 该进程名字是mysqld) {
 "A mysqld process already exists"
 报错退出
 }
 删除$pid_file // 说明是老的mysqld生成的
 if ($pid_file文件存在) {
 "Fatal error: Can't remove the pid file: $pid_file. Please remove the file manually and start $0 again; mysqld daemon not started" // 文件删失败了
 报错退出
 }
 同上，尝试删除socket文件 // my.cnf中socket配置的路径
 同上，尝试删除$pid_file.shutdown文件
}
"Starting $MYSQLD daemon with databases from $DATADIR"
while true { // 核心逻辑的主循环
 启动mysqld // 正常启动成功后，mysqld_safe就会等在这里
 if (返回值 == 16) {
 dont_restart_mysqld=false
 "Restarting mysqld..."
 } else {
 dont_restart_mysqld=true
 }

 if (dont_restart_mysqld) {
 if ($pid_file文件不存在) {
 // 说明是normal shutdown，pid文件会在mysqld退出时自动被删掉
 break; // 跳出while循环
 } else {
 从$pid_file读取pid
 if (pid进程存在) {
 "A mysqld process with pid=$PID is already running. Aborting!!"
 报错退出
 }
 }
 }

 if (存在$pid_file.shutdown文件) {
 "$pid_file.shutdown present. The server will not restart."
 break;
 }

 判断$fast_restart变量，做一些限速 // 细节暂时省略

 if (启动mysqld_safe时没配置--skip-kill-mysqld选项) { // 正常都是不配的
 $numofproces = 统计当前使用了$pid_file路径的mysqld进程数
 "Number of processes running now: $numofproces"
 while (循环$numofproces) {
 获取其中一个mysqld进程的pid
 if (kill -9 该进程) { // 发SIGKILL
 "$MYSQLD process hanging, pid $T - killed"
 } else {
 break;
 }
 }
 }
 删除$pid_file、socket文件、$pid_file.shutdown
 "mysqld restarted"
}
删除$pid_file.shutdown文件
"mysqld from pid file $pid_file ended"
删除$safe_pid文件 // 似乎是毫无意义的一段代码
`

## Part 3

整个代码的理解，主要涉及了一些commit历史“考古”的工作和bash脚本的写法。

* 如何确定某个pid属于一个running的进程，scripts/CMakeLists.txt下面有CHECK_PID的定义。通过kill -0返回值确定。如果系统不支持signal 0，就发SIGCONT，本身这俩类型的signal发给一个运行的进程是没副作用的。

`EXECUTE_PROCESS(COMMAND sh -c "kill -0 $$"
 OUTPUT_QUIET ERROR_QUIET RESULT_VARIABLE result)
IF(result MATCHES 0)
 SET(CHECK_PID "kill -0 $PID > /dev/null 2> /dev/null")
ELSE()
 SET(CHECK_PID "kill -s SIGCONT $PID > /dev/null 2> /dev/null")
ENDIF()
`

* socket file的地址是这么算的。其中`:-`符号是bash中变量默认值的用法，`${a:-b}`的意思就是如果a存在且不为空，返回a，否则返回b。

```
safe_mysql_unix_port=${mysql_unix_port:-${MYSQL_UNIX_PORT:-@MYSQL_UNIX_ADDR@}}

```

* “$numofproces = 统计当前使用了$pid_file路径的mysqld进程数”，这个是判断当前是否存在mysqld_safe负责的mysqld还hang在那里。ps中的ww是为了打印完成的命令，否则默认会限制window size；排除grep自己；过滤mysqld和pid这俩路径；grep中的>是匹配一个空格；grep -c可以返回行数。

```
ps xaww | grep -v "grep" | grep "$ledir/$MYSQLD\>" | grep -c "pid-file=$pid_file"

```

* 统计完$numofproces之后，要kill hanging的mysqld，用了如下几行脚本找pid。PROC前半段类似上一条，最后sed -n ‘$p’是取出最后一行。随后的for循环，相当于把PROC当做空格分割的数组，取出第一列，也就是pid。其实这么搞复杂了，还不如直接用awk…

```
PROC=`ps xaww | grep "$ledir/$MYSQLD\>" | grep -v "grep" | grep "pid-file=$pid_file" | sed -n '$p'`

for T in $PROC
do
 break
done

```

* 代码逻辑中有几处处理$pid_file.shutdown文件的。据考证是以前mysql.init里面用的历史遗留代码，现在已经废弃了。很早之前官方就都替换成给DB发SIGTERM信号，走normal shutdown了。所以$pid_file.shutdown这些目前其实是废代码。
* 脚本中涉及到删除文件的地方，比如删除$pid_file、socket file，都加了非symbolic links的限制（`! -h "$pid_file"`）。这个是为了规避symbolic links可能产生的privilege escalation的风险。以前有很多符号链接漏洞攻击的案例，感兴趣细节的可以Google。

`if [ ! -h "$pid_file" ]; then
 rm -f "$pid_file"
 if test -f "$pid_file"; then
 log_error "Fatal error: Can't remove the pid file: $pid_file. Please remove the file manually and start $0 again; mysqld daemon not started"
 exit 1
 fi
fi
`

* “判断$fast_restart变量，做一些限速”，实际逻辑很简单，统计每次“启动mysqld”这一步的开始时间和结束时间，连续多次（代码里是5次）间隔时间小于1s，就sleep 1s后再跳到下一轮循环。这个优化是为了防止mysqld一直拉不起来，比如有非法参数，导致mysqld_safe一直疯狂尝试重启，占用100%的cpu。
* 为啥会有“返回值 == 16”条件下dont_restart_mysqld=false的逻辑？因为是支持通过SQL请求发restart命令来重启mysqld的，代码里有`#define MYSQLD_RESTART_EXIT 16`。只要有父进程管理mysqld（参考sql_restart_server.cc的代码is_mysqld_managed），restart命令会走SIGUSR2的信号处理函数，返回的值是MYSQLD_RESTART_EXIT。
* mysqld_safe还支持设malloc-lib，在脚本里，转换成LD_PRELOAD设置在mysqld启动的命令中。常见的用法是改成jemalloc，在my.cnf里加上如下配置：

`[mysqld_safe]
malloc-lib=/path/libjemalloc.so
`

## Part 4

之前线上还出现过一个bug，当一台机器的某个mysqld故障之后，会出现同宿主机的其他mysqld被自动重启一遍，非常诡异。最后排查下来就和mysqld_safe有关。云环境mysqld都是混布的，通过K8S这样的技术去做隔离。最开始的时候我们容器技术做的不完善，各个mysqld的文件系统是隔离的，但是进程权限没隔离。导致的情况就是从每个容器里面看，能看到所有的mysqld进程，且–pid-file上都是配置的同一目录。

`PROC=`ps xaww | grep "$ledir/$MYSQLD\>" | grep -v "grep" | grep "pid-file=$pid_file" | sed -n '$p'`
`
结果mysqld_safe通过如上命令找进程的时候，就把别的不属于自己管理的mysqld kill了…之后修复方法就是把进程权限的隔离做上去，这样从一个容器就看不到其他容器里跑的mysqld了。

**大概就总结这些**

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)