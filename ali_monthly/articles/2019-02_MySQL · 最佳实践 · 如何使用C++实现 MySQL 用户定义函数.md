# MySQL · 最佳实践 · 如何使用C++实现 MySQL 用户定义函数

**Date:** 2019/02
**Source:** http://mysql.taobao.org/monthly/2019/02/08/
**Images:** 1 images downloaded

---

数据库内核月报

 [
 # 数据库内核月报 － 2019 / 02
 ](/monthly/2019/02)

 * 当期文章

 POLARDB · 性能优化 · 敢问路在何方 — 论B+树索引的演进方向（中）
* MySQL · 引擎特性 · Inspecting the Content of a MySQL Histogram
* Database · 原理介绍 · Snapshot Isolation 综述
* MSSQL · 最佳实践 · 数据库备份加密
* MySQL · 引擎特性 · The design of mysql8.0 redolog
* MySQL · 源码分析 · 8.0 Functional index的实现过程
* PgSQL · 源码解析 · Json — 从使用到源码
* MySQL · 最佳实践 · 如何使用C++实现 MySQL 用户定义函数
* MySQL · 最佳实践 · MySQL多队列线程池优化
* PgSQL · 应用案例 · PostgreSQL 时间线修复

 ## MySQL · 最佳实践 · 如何使用C++实现 MySQL 用户定义函数 
 Author: 荣生 

 ## 什么是用户定义函数(UDF, User-Defined Functions)

在MySQL中，可以通过UDF扩充MySQL的功能，加入一个新的SQL函数类似于内置的函数（如，ABS() or CONCAT()等。UDF使用C/C++实现， 编译成动态库文件（Linux对应.so文件），可以使用 CREATE FUNCTION动态加载到 mysqld服务进程里，使用DROP FUNCTION从mysqld 服务进程里移除。本文在MySQL 8.0上首先对MySQL UDF的接口进行了介绍，然后给出了一个简单的例子。 通过使用 UDF：

* 可以返回 string, integer 或 real 类型的值或作为函数参数
* 能够定义简单函数或聚集函数（aggregate function），本文只讲解简单函数，聚集函数请参考MySQL用户手册

## 如何实现MySQL UDF

MySQL UDF必须使用C/C++实现，同时要求操作系统必须支持动态装载，如果使用了mysqld中已经存在的符号， 那么链接动态库的时候必须得使用链接选项 -rdynamic。

### UDF接口

为了定义UDF，需要为每个UDF生成对应C/C++函数，为了下文描述方便，我们用“xxx”表示函数名，用大写的XXX()表示一个SQL 函数调用，用小写的xxx()表示一个C/C++函数。下面是实现一个 SQL 函数XXX()所需要定义的C/C++函数。

#### xxx()

主函数，在SQL调用函数XXX()时最终会调用到这里，SQL的数据类型和C/C++的数据类型对应关系如下：

 SQL类型
 C/C++ 类型

 STRING
 char *

 INTEGER
 long long

 REAL
 double

这些数据类型用于函数的返回值和函数参数。

函数定义如下：

* 对于SQL函数的返回值是STRING的 (这个函数原型同样适用于SQL函数返回类型是DECIMAL)

`char * xxx(UDF_INIT *initid, UDF_ARGS *args,
 char *result, unsigned long *length,
 char *is_null, char *error);
`

* 对于返回值是INTEGER的

```
long long xxx(UDF_INIT *initid, UDF_ARGS *args,
 char *is_null, char *error);

```

* 对于返回值是REAL的

```
double xxx(UDF_INIT *initid, UDF_ARGS *args,
 char *is_null, char *error);

```

#### xxx_init()

xxx()函数的初始化函数，这个函数的作用包括：

* 检查传入XXX()函数的参数个数
* 检验传入XXX()的参数的数据类型，而且它还可以让MySQL将传入XXX()的参数转成xxx()需要的数据类型
* 分配xxx()函数需要的内存
* 指定返回值的最大长度
* 指定返回值是REAL的函数的返回值的精度
* 指定返回值是不是NULL

函数原型如下：

`bool xxx_init(UDF_INIT *initid, UDF_ARGS *args, char *message);
`

#### xxx_deinit()

xxx()的析构函数，用于释放初始化函数分配的内存或做其它清理工作，这个函数是可选的。 函数原型如下：

`void xxx_deinit(UDF_INIT *initid);
`

#### UDF执行流程

当在一个SQL语句中调用XXX()时，MySQL首先调用xxx_init()函数做必要的初始化工作，比如：参数检查、内存分配等。 如果xxx_init()返回错误，则主函数xxx()和析构函数xxx_deinit()不会被调用，整个语句会报错退出。如果xxx_init() 执行成功MySQL会调用主函数xxx()，通常情况下会每行数据调用一次，依赖于XXX()在SQL语句中的位置。当所有的主函数xxx() 都调用完成后，MySQL会调用对应的析构函数xxx_deinit()做必要的清理工作。

注意，以上所有的函数都要求是线程安全的。同时如果是用C++实现的，那么在定义的函数开头必须要加上 extern “C”，以便 MySQL可以找到相应的符号。

#### UDF实现相关数据结构说明

1. UDF_INIT

 是参数initid的类型，该参数是3个函数都需要的，可以在xxx_init函数中初始化。该结构的主要成员如下：

 * bool maybe_null

 如果xxx()可以返回NULL，xxx_init函数需要把它设置成true，如果函数参数有maybe_null是true的，该值的默认值就是true。

 * unsigned int max_length

 返回值的最大长度。对于不同返回类型该值的默认值不同，对于STRING，默认值和和最长的函数参数相等。对于INTEGER， 默认值是21。如果是BLOB类型的，可以将它设置成65KB或16MB。

 * char *ptr

 一个透明的指针，UDF的实现可以自己根据需要使用。该指针一般在xxx_init()里分配内存，在xxx_deinit()里进行释放。

 * bool const_item

 如果xxx()函数总是返回相同的值，xxx_init()中可以把该值设置成true。
2. UDF_ARGS

 是参数args是数据类型，主要成员如下：

 * unsigned int arg_count

 SQL函数参数的个数，也是下面其他成员的数组长度。可以在xxx_init()函数里检查是否与预期一致，如：

 `if (args->arg_count != 2)
{
 strcpy(message, "XXX() requires two arguments");
 return 1;
}
` 

 * enum Item_result *arg_type

 是一个定义了每个参数类型的数组，每个元素可能的取值：TRING_RESULT, INT_RESULT, REAL_RESULT, 和DECIMAL_RESULT。 也可以通过它在xxx_init()里指定某个参数的数据类型，MySQL会将输入的参数强制转化为该类型。

 * char **args

 对于xxx_init()，当参数是常量时，比如 3、4*7-2或SIN(3.14) args->args[i]指向参数值，当参数是非常量时 args->args[i]为NULL；对于主函数xxx()总是指向参数的值，如果参数i为null，则args->args[i]为NULL。

 * 对于STRING_RESULT类型，args->args[i]指向对应的字符串，args->lengths[i]是字符串长度。
* 对于INT_RESULT类型，需要强制转化成long long:

 `long long int_val = *(long long *) args->args[i];
` 

 * 对于REAL_RESULT类型，需要转成double:

 ```
double real_val = *(double *) args->args[i]

```

 * unsigned long *lengths

 对于xxx_init()函数该数组包含每个参数的最大长度，对于xxx()函数为参数的实际长度。

 * char *maybe_null

 对于xxx_init()该成员表示对应的参数是否可以为null。

 * **attributes

 表示传入参数的参数名，参数名的长度在args->attribute_lengths[i]中。

#### UDF返回值及错误处理

如果有错误发生xxx_init()应该返回true，同时将错误消息保存在message参数中，message参数的buffer长度为 MYSQL_ERRMSG_SIZE(512)。对于long long和double的SQL函数的返回值通过主函数xxx()的返回值返回。字符串类型的SQL函数 如果字符串长度小于255,可以通过参数result参数返回，实际长度存在*length中，xxx()函数要返回result；如果要返回的字 符串长度大于255，需要自己分配内存并通过xxx()返回值返回。分配的内存需要在xxx_deinit里释放。可以通过设置*is_null = 1来表示SQL函数返回值为null。另外如果函数发生错误需要设置 *error = 1。

#### UDF的编译和安装

这里只讲Linux下编译和安装，编译可以使用如下命令：

`c++ -I$(MYSQL_INSTALLDIR)/include -fPIC -g -shared \
 -o $(MYSQL_INSTALLDIR)/lib/plugin/libmyudf.so myudf.cc
`

这里MYSQL_INSTALLDIR指的是MySQL的安装目录。编译完成后生成的目标动态库直接写到了MySQL的安装目录的plugin目录 下，mysqld只在这个目录上寻找UDF实现动态库。

## UDF函数的使用

使用mysql命令连接到MySQL server，执行以下查询在数据库中生成SQL函数

`CREATE FUNCTION myudf RETURNS INT SONAME 'libmyudf.so';
`

这里在的libmyudf.so是前面编译生成的动态库。可以通过系统表mysql.func和performance_schema下的user_defined_functions 来跟踪系统中已经安装的UDF。

## 一个简单的例子

`#include "mysql.h"
#include <sys/types.h> /* getpid() */
#include <unistd.h> /* getpid() */

extern "C" bool
mysqld_pid_init(UDF_INIT *initid __attribute__((unused)),
 UDF_ARGS *args __attribute__((unused)),
 char *message __attribute__((unused)))
{
 return false;
}

extern "C" long long
mysqld_pid(UDF_INIT *initid __attribute__((unused)),
 UDF_ARGS *args __attribute__((unused)),
 char *is_null __attribute__((unused)),
 char *error __attribute__((unused)))
{
 return getpid();
}

`

这个例子实现了一个简单的UDF：mysqld_pid()，该UDF可以拿到mysqld进程的PID，可以通过SQL语句select mysqld_pid()调用。 将以上例子拷贝到一个文件，比如mysqld_pid.cc然后进行编译：

`c++ -I$(MYSQL_INSTALLDIR)/include -fPIC -g -shared -o \
 $(MYSQL_INSTALLDIR)/lib/plugin/libmysqld_pid.so mysqld_pid.cc
`

最后通过mysql连接到数据库执行如下SQL语句，将mysqld_pid安装到数据库，用户就可以在SQL语句上使用了。

```
CREATE FUNCTION mysqld_pid RETURNS INT SONAME 'libmysqld_pid.so';

```

 阅读： - 

[![知识共享许可协议](.img/8232d49bd3e9_88x31.png)](http://creativecommons.org/licenses/by-nc-sa/3.0/)
本作品采用[知识共享署名-非商业性使用-相同方式共享 3.0 未本地化版本许可协议](http://creativecommons.org/licenses/by-nc-sa/3.0/)进行许可。

 [

 ](#0)