# MySQL · 源码阅读 · 非阻塞异步C API简介

**Date:** 2022/04
**Source:** http://mysql.taobao.org/monthly/2022/04/04/
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

 ## MySQL · 源码阅读 · 非阻塞异步C API简介 
 Author: 杭枫 

 ## 概述
MySQL从8.0.16版本引入了非阻塞的异步C API接口，可以与 MySQL服务器进行非阻塞通信。原先的同步阻塞式接口在发起某个调用后，客户端必须等待这个调用返回结果才能继续往下执行，而异步非阻塞接口则不需要等待调用返回结果，在发起调用后，客户端可以继续执行后续操作，服务端通过某种反馈机制通知客户端调用是否完成，因此能够提高系统的整体响应速度和吞吐。

## MySQL C API接口
无论使用阻塞接口还是非阻塞接口，MySQL client在请求DB查询的时候，通常需要经过建立连接，执行SQL，获取结果集，关闭连接等步骤。非阻塞接口一般和阻塞接口相互对应，参数基本相同，通常通过net_async_status
枚举值返回异步执行状态。

 枚举返回值
 描述

 NET_ASYNC_COMPLETE
 异步操作已完成

 NET_ASYNC_NOT_READY
 异步操作仍在进行中

 NET_ASYNC_ERROR
 异步操作发生错误

 NET_ASYNC_COMPLETE_NO_MORE_RESULTS
 对于mysql_next_result_nonblocking()调用，表示没有更多结果

通常，要使用异步接口，需要执行以下操作：

* 重复调用该函数，直到其不再返回NET_ASYNC_NOT_READY状态。
* 检查最终状态是表示成功完成 ( NET_ASYNC_COMPLETE) 还是错误 ( NET_ASYNC_ERROR)。

### 调用模式
以下示例展示了一些异步接口的调用模式。

* 如果需要在操作进行时执行其他处理

`enum net_async_status status;

status = function(args);
while (status == NET_ASYNC_NOT_READY) {
 /* perform other processing */
 other_processing ();
 /* invoke same function and arguments again */
 status = function(args);
}
if (status == NET_ASYNC_ERROR) {
 /* call failed; handle error */
} else {
 /* call successful; handle result */
}
`

* 如果在操作过程中不需要进行其他处理

```
enum net_async_status status;

while ((status = function(args)) == NET_ASYNC_NOT_READY)
 ; /* empty loop */
if (status == NET_ASYNC_ERROR) {
 /* call failed; handle error */
} else {
 /* call successful; handle result */
}

```

* 如果不关心函数成功/失败

```
while (function (args) != NET_ASYNC_COMPLETE)
 ; /* empty loop */

```

### 建立连接

 C API接口
 描述
 返回值

 mysql_real_connect
 同步建立和MySQL服务端连接
 MYSQL句柄或null

 mysql_real_connect_nonblocking
 异步建立和MySQL服务端连接
 enum net_async_status

mysql_real_connect_nonblocking和mysql_real_connect的参数完全一样, 区别在于mysql_real_connect_nonblocking异步返回net_async_status枚举值，mysql_real_connect同步返回MYSQL句柄，如果连接失败则返回nullptr。

### 执行SQL

 C API接口
 描述
 返回值

 mysql_real_query
 同步执行SQL
 如果执行成功，返回0。如果出现错误，返回非0值

 mysql_real_query_nonblocking
 异步执行SQL
 enum net_async_status

mysql_real_query_nonblocking和mysql_real_query的参数完全一样, 区别在于mysql_real_query_nonblocking异步返回net_async_status枚举值，mysql_real_query同步返回int类型的返回值，如果执行成功，返回0。如果出现错误，返回非0值。

### 获取查询结果集

 C API接口
 描述
 参数
 返回值

 mysql_store_result
 同步获取查询结果集
 MYSQL *mysql
 分配1个MYSQL_RES结构，并将结果置于该结构中。

 mysql_store_result_nonblocking
 异步获取查询结果集
 MYSQL *mysql，MYSQL_RES **result
 enum net_async_status

mysql_store_result_nonblocking比mysql_store_result多一个输入参数，用于保存返回查询结果集的指针，mysql_store_result_nonblocking异步返回net_async_status枚举值，mysql_store_result同步将查询的全部结果读取到客户端，分配1个MYSQL_RES结构，并将结果置于该结构中。如果查询未返回结果集，mysql_store_result()将返回Null指针。

### 获取下一个查询结果集

 C API接口
 描述
 返回值

 mysql_next_result
 同步获取下一个查询结果集
 如果执行成功并有多个结果集，返回0; 如果执行成功但没有多个结果集，返回-1; 如果出现错误，返回>0的值。

 mysql_next_result_nonblocking
 异步获取下一个查询结果集
 enum net_async_status

mysql_next_result_nonblocking和mysql_next_result的输入参数完全一样，区别在于mysql_next_result_nonblocking异步返回net_async_status枚举值，mysql_next_result同步返回int类型的返回值。如果存在多个查询结果，mysql_next_result()将读取下一个查询结果，并将状态返回给应用程序。如果前面的查询返回了结果集，必须为其调用mysql_free_result()。如果mysql_next_result()返回错误，将不执行任何其他语句，也不会获取任何更多的结果。

### 获取结果集的下一行

 C API接口
 描述
 参数
 返回值

 mysql_fetch_row
 同步获取结果集的下一行
 MYSQL_RES *result
 下一行的MYSQL_ROW结构。如果没有更多要检索的行或出现了错误，返回NULL。

 mysql_fetch_row_nonblocking
 异步获取结果集的下一行
 MYSQL_RES *result, MYSQL_ROW *row
 enum net_async_status

mysql_fetch_row_nonblocking异步返回net_async_status枚举值，并将下一行MYSQL_ROW结构指针保存在输入参数row中。在mysql_store_result()之后使用时，如果没有要检索的行，mysql_fetch_row()返回NULL。
行内值的数目由mysql_num_fields(result)给出。如果行中保存了调用mysql_fetch_row()返回的值，将按照row[0]到row[mysql_num_fields(result)-1]，访问这些值的指针。行中的NULL值由NULL指针指明。
可以通过调用mysql_fetch_lengths()来获得行中字段值的长度。对于空字段以及包含NULL的字段，长度为0。通过检查字段值的指针，能够区分它们。如果指针为NULL，字段为NULL，否则字段为空。

### 释放结果集分配的内存

 C API接口
 描述
 返回值

 mysql_free_result
 同步释放结果集分配的内存
 无

 mysql_free_result_nonblocking
 异步释放结果集分配的内存
 enum net_async_status

mysql_free_result_nonblocking异步释放结果集分配的内存，通过net_async_status枚举值反应执行状态。
mysql_free_result没有返回值。完成对结果集的操作后，必须调用mysql_free_result()或mysql_free_result_nonblocking释放结果集使用的内存。释放完成后，不能访问结果集。

## 示例
### 准备工作
`CREATE DATABASE db;
USE db;
CREATE TABLE test_table (id INT NOT NULL);
INSERT INTO test_table VALUES (10), (20), (30);

CREATE USER 'testuser'@'localhost' IDENTIFIED BY 'testpass';
GRANT ALL ON db.* TO 'testuser'@'localhost';
`
创建一个名为async_app.cc包含以下程序的文件。根据需要调整连接参数。

`#include <stdio.h>
#include <string.h>
#include <iostream>
#include <mysql.h>
#include <mysqld_error.h>

using namespace std;

/* change following connection parameters as necessary */
static const char * c_host = "localhost";
static const char * c_user = "testuser";
static const char * c_auth = "testpass";
static int c_port = 3306;
static const char * c_sock = "/usr/local/mysql/mysql.sock";
static const char * c_dbnm = "db";

void perform_arithmetic() {
 cout<<"dummy function invoked\n";
 for (int i = 0; i < 1000; i++)
 i*i;
}

int main(int argc, char ** argv)
{
 MYSQL *mysql_local;
 MYSQL_RES *result;
 MYSQL_ROW row;
 net_async_status status;
 const char *stmt_text;

 if (!(mysql_local = mysql_init(NULL))) {
 cout<<"mysql_init() failed\n";
 exit(1);
 }
 while ((status = mysql_real_connect_nonblocking(mysql_local, c_host, c_user,
 c_auth, c_dbnm, c_port,
 c_sock, 0))
 == NET_ASYNC_NOT_READY)
 ; /* empty loop */
 if (status == NET_ASYNC_ERROR) {
 cout<<"mysql_real_connect_nonblocking() failed\n";
 exit(1);
 }

 /* run query asynchronously */
 stmt_text = "SELECT * FROM test_table ORDER BY id";
 status = mysql_real_query_nonblocking(mysql_local, stmt_text,
 (unsigned long)strlen(stmt_text));
 /* do some other task before checking function result */
 perform_arithmetic();
 while (status == NET_ASYNC_NOT_READY) {
 status = mysql_real_query_nonblocking(mysql_local, stmt_text,
 (unsigned long)strlen(stmt_text));
 perform_arithmetic();
 }
 if (status == NET_ASYNC_ERROR) {
 cout<<"mysql_real_query_nonblocking() failed\n";
 exit(1);
 }

 /* retrieve query result asynchronously */
 status = mysql_store_result_nonblocking(mysql_local, &result);
 /* do some other task before checking function result */
 perform_arithmetic();
 while (status == NET_ASYNC_NOT_READY) {
 status = mysql_store_result_nonblocking(mysql_local, &result);
 perform_arithmetic();
 }
 if (status == NET_ASYNC_ERROR) {
 cout<<"mysql_store_result_nonblocking() failed\n";
 exit(1);
 }
 if (result == NULL) {
 cout<<"mysql_store_result_nonblocking() found 0 records\n";
 exit(1);
 }

 /* fetch a row synchronously */
 row = mysql_fetch_row(result);
 if (row != NULL && strcmp(row[0], "10") == 0)
 cout<<"ROW: " << row[0] << "\n";
 else
 cout<<"incorrect result fetched\n";

 /* fetch a row asynchronously, but without doing other work */
 while (mysql_fetch_row_nonblocking(result, &row) != NET_ASYNC_COMPLETE)
 ; /* empty loop */
 /* 2nd row fetched */
 if (row != NULL && strcmp(row[0], "20") == 0)
 cout<<"ROW: " << row[0] << "\n";
 else
 cout<<"incorrect result fetched\n";

 /* fetch a row asynchronously, doing other work while waiting */
 status = mysql_fetch_row_nonblocking(result, &row);
 /* do some other task before checking function result */
 perform_arithmetic();
 while (status != NET_ASYNC_COMPLETE) {
 status = mysql_fetch_row_nonblocking(result, &row);
 perform_arithmetic();
 }
 /* 3rd row fetched */
 if (row != NULL && strcmp(row[0], "30") == 0)
 cout<<"ROW: " << row[0] << "\n";
 else
 cout<<"incorrect result fetched\n";

 /* fetch a row asynchronously (no more rows expected) */
 while ((status = mysql_fetch_row_nonblocking(result, &row))
 != NET_ASYNC_COMPLETE)
 ; /* empty loop */
 if (row == NULL)
 cout <<"No more rows to process.\n";
 else
 cout <<"More rows found than expected.\n";

 /* free result set memory asynchronously */
 while (mysql_free_result_nonblocking(result) != NET_ASYNC_COMPLETE)
 ; /* empty loop */

 mysql_close(mysql_local);
}
`
运行程序：

`dummy function invoked
dummy function invoked
ROW: 10
ROW: 20
dummy function invoked
ROW: 30
No more rows to process.
`
## 限制
异步API存在如下一些限制：

* mysql_real_connect_nonblocking() 只能用于使用以下身份验证插件之一进行身份验证的帐户： mysql_native_password、 sha256_password或 caching_sha2_password.
* mysql_real_connect_nonblocking() 只能用于建立 TCP/IP 或 Unix 套接字文件连接。
* 这些语句不受支持，必须使用同步 C API 函数进行处理：LOAD DATA, LOAD XML.
* 传递给非阻塞操作的异步 C API 调用的输入参数可能会一直使用，因此在异步C API结束前不能重用这些输入参数。
* 异步 C API 函数不支持协议压缩。

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)