# MySQL · 源码分析 ·  mysql认证阶段漫游

**Date:** 2017/08
**Source:** http://mysql.taobao.org/monthly/2017/08/05/
**Images:** 2 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2017 / 08
 ](/monthly/2017/08)

 * 当期文章

 MySQL · 引擎特性 · Group Replication内核解析
* PgSQL · 特性介绍 · 列存元数据扫描介绍
* MySQL · 源码分析 · MySQL replication partial transaction
* MySQL · 特性分析 · 到底是谁执行了FTWL
* MySQL · 源码分析 · mysql认证阶段漫游
* MySQL · 源码分析 · 内存分配机制
* PgSQL · 源码分析 · PG 优化器中的pathkey与索引在排序时的使用
* MSSQL· 实现分析 · Extend Event日志文件的分析方法
* MySQL · 源码分析 · SHUTDOWN过程
* PgSQL · 应用案例 · HDB for PG特性(数据排盘与任意列高效率过滤)

 ## MySQL · 源码分析 · mysql认证阶段漫游 
 Author: santo 

 client发起一个连接请求, 到拿到server返回的ok包之间, 走三次握手, 交换了[不可告人]的验证信息, 这期间mysql如何完成校验工作?

## 过程(三次握手)

![没加滤镜的三次握手](.img/b1f777cf1688_4c8292a87c407fbe12c8272be36e3c22.png)

## 信息是如何加密的

### client:

hash_stage1 = sha1(password)
hash_stage2 = sha1(hash_stage1)
reply = sha1(scramble, hash_stage2) ^ hash_stage1

### server: (逻辑位于sql/password.c:check_scramble_sha1中, 下文亦有提及)

// mysql.user表中, 对应user的passwd实际上是hash_stage2
res1 = sha1(scramble, hash_stage2)
hash_stage1 = reply ^ res1
hash_stage2_reassured = sha1(hash_stage1)
再根据hash_stage2_reassured == hash_stage2(from mysql.user)是否一致来判定是否合法

## 涉事函数们

如图, client发起连接请求, server创建新的线程, 并进入acl_authenticate(5.7位于sql/auth/sql_authentication.cc, 5.6位于sql/sql_acl.cc)函数完成信息验证, 并把包里读出的信息更新到本线程.

流程堆栈:

`#0 parse_client_handshake_packet 
#1 server_mpvio_read_packet 
#2 native_password_authenticate
#3 do_auth_once 
#4 acl_authenticate 
#5 check_connection 
#6 login_connection 
#7 thd_prepare_connection
#8 do_handle_one_connection
`

接下来考察这些函数中做了哪些事.
check_connection(sql/sql_connect.cc)
当接收到client的建连接请求时, 进入check_connection, 先对连接本身上下文分析(socket, tcp/ip的v4/6 哇之类的)
当然你用very long的host连进来, 也会在这里被cut掉防止overflow. 
不合法的ip/host也会在这里直接返回, 如果环境ok, 就进入到acl_authenticate的逻辑中
acl_authenticate: 
初始化MPVIO_EXT, 用于保存验证过程的上下文; 字符集, 挑战码, …的坑位, 上锁, 根据command进行分派, (新建链接为COM_CONNECT

COM_CONNECT下会进入函数do_auth_once(), 返回值直接决定握手成功与否.
先对authentication plugin做判定, 咱们这里基于”mysql_native_password”的情况

`if (plugin)
 {
 st_mysql_auth *auth= (st_mysql_auth *) plugin_decl(plugin)->info;
 res= auth->authenticate_user(mpvio, &mpvio->auth_info); 
 ...
`

在mysql_native_password时会进入native_password_authenticate 逻辑:

` /* generate the scramble, or reuse the old one */
 if (mpvio->scramble[SCRAMBLE_LENGTH])
 create_random_string(mpvio->scramble, SCRAMBLE_LENGTH, mpvio->rand);

 /* send it to the client */
 if (mpvio->write_packet(mpvio, (uchar*) mpvio->scramble, SCRAMBLE_LENGTH + 1)) 
 DBUG_RETURN(CR_AUTH_HANDSHAKE);

 /* read the reply with the encrypted password */
 if ((pkt_len= mpvio->read_packet(mpvio, &pkt)) < 0) 
 DBUG_RETURN(CR_AUTH_HANDSHAKE);
 DBUG_PRINT("info", ("reply read : pkt_len=%d", pkt_len));
`

可见这里才生成了挑战码并发送到client, 再调用mpvio->read_packet等待client回包, 
进入server_mpvio_read_packet, 
这里的实现则调用常见的my_net_read读包, 
当拿到auth包时, 逻辑分派到parse_client_handshake_packet, 对包内容进行parse, 这里会根据不同client protocol, 去掉头和尾, 还对client是否设置了ssl做判定. 接着:

` if (mpvio->client_capabilities & CLIENT_SECURE_CONNECTION) 
 {
 /* 
 Get the password field.
 */
 passwd= get_length_encoded_string(&end, &bytes_remaining_in_packet,
 &passwd_len);
 }
 else 
 {
 /* 
 Old passwords are zero terminatedtrings.
 */
 passwd= get_string(&end, &bytes_remaining_in_packet, &passwd_len);
 }
 ...
`

在拿到了client发来的加密串(虽然叫passwd), 暂时存放在内存中, 返回native_password_authenticate, 
当判定为需要做password check时(万一有人不设置密码呢), 进入check_scramble, 这个函数中才实现了对密码的验证:

`// server decode回包中的加密信息
// 把上面提到的三个公式包在函数中
my_bool
check_scramble_sha1(const uchar *scramble_arg, const char *message,
 const uint8 *hash_stage2)
{
 uint8 buf[SHA1_HASH_SIZE];
 uint8 hash_stage2_reassured[SHA1_HASH_SIZE];

 /* create key to encrypt scramble */
 compute_sha1_hash_multi(buf, message, SCRAMBLE_LENGTH,
 (const char *) hash_stage2, SHA1_HASH_SIZE);
 /* encrypt scramble */
 my_crypt((char *) buf, buf, scramble_arg, SCRAMBLE_LENGTH);

 /* now buf supposedly contains hash_stage1: so we can get hash_stage2 */
 compute_sha1_hash(hash_stage2_reassured, (const char *) buf, SHA1_HASH_SIZE);

 return MY_TEST(memcmp(hash_stage2, hash_stage2_reassured, SHA1_HASH_SIZE));
}
`

native_password_authenticate拿到check_scamble的返回值, 返回OK, 
再返回到acl_authenticate, 讲mpvio中环境信息更新到线程信息THD中, successful login~

(所以可以魔改这块代码搞事, 密码什么的, 权限什么的….
(我就说说, 别当真

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)