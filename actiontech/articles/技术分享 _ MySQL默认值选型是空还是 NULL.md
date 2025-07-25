# 技术分享 | MySQL默认值选型（是空，还是 NULL）

**原文链接**: https://opensource.actionsky.com/20190710-mysql/
**分类**: MySQL 新特性
**发布时间**: 2019-07-10T01:07:09-08:00

---

如果对一个字段没有过多要求，是使用“”还是使用 NULL，一直是个让人困惑的问题。即使有前人留下的开发规范，但是能说清原因的也没有几个。NULL 是“”吗？在辨别 NULL 是不是空的这个问题上，感觉就像是在证明 1 + 1 是不是等于 2。
在 MySQL 中的 NULL 是一种特殊的数据。一个字段是否允许为 NULL，字段默认值是否为 NULL。
主要有如下几种情况：
| 字段类型 | 表定义中设置方式 | 字段值 |
| --- | --- | --- |
| 数值类型 (INT/BIGINT) | Default NULL / Default 0 | NULL / NUM |
| 字符类型 (CHAR/VARCHAR) | Default NULL / Default &#8221; / Default &#8216;ab&#8217; | NULL / &#8221; / String |
**1. NULL 与空字符存储上的区别**
表中如果允许字段为 NULL，会为每行记录分配 NULL 标志位。NULL 除了在每行的行首存有 NULL 标志位，实际存储不占有任何空间。如果表中所有字段都是非 NULL，就不存在这个标示位了。网上有一些验证 MySQL 中 NULL 存储方式的文章，可以参考下。
**2. NULL使用上的一些问题。**
数值类型，对一个允许为NULL的字段进行min、max、sum、加减、order by、group by、distinct 等操作的时候。字段值为非 NULL 值时，操作很明确。如果使用 NULL， 需要清楚的知道如下规则：
**数值类型，以 INT 列为例**
**1) 在 min / max / sum / avg 中 NULL 值会被直接忽略掉，如下是测试结果，可能 min / max / sum 还比较可以理解，但 avg 真的是你想要的结果吗？**
CREATE TABLE `t1` (
`id` int(16) NOT NULL AUTO_INCREMENT,
`name` varchar(20) DEFAULT NULL,
`number` int(11) DEFAULT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8;
select * from t1;
+------+----------+--------+
| id | name | number |
+------+----------+--------+
| 1 | zhangsan | NULL |
| 2 | lisi | NULL |
| 3 | wangwu | 0 |
| 4 | zhangliu | 4 |
+------+----------+--------+
select max(number) from t1;
+-------------+
| max(number) |
+-------------+
| 4 |
+-------------+
select min(number) from t1;
+-------------+
| min(number) |
+-------------+
| 0 |
+-------------+
select sum(number) from t1;
+-------------+
| sum(number) |
+-------------+
| 4 |
+-------------+
select avg(number) from t1;
+-------------+
| avg(number) |
+-------------+
| 2.0000 |
+-------------+
**2) 对 NULL 做加减操作,如 1 + NULL，结果仍是 NULL**
select 1+NULL;
+--------+
| 1+NULL |
+--------+
| NULL |
+--------+
**3) order by 以升序检索字段的时候 NULL 会排在最前面（倒序相反）**
select * from t1 order by number;
+----+----------+--------+
| id | name | number |
+----+----------+--------+
| 1 | zhangsan | NULL |
| 2 | lisi | NULL |
| 3 | wangwu | 0 |
| 4 | zhangliu | 4 |
+----+----------+--------+
select * from t1 order by number desc;
+----+----------+--------+
| id | name | number |
+----+----------+--------+
| 4 | zhangliu | 4 |
| 3 | wangwu | 0 |
| 1 | zhangsan | NULL |
| 2 | lisi | NULL |
+----+----------+--------+
#### 4) group by / distinct 时，NULL 值被视为相同的值
select distinct(number) from t1;
+--------+
| number |
+--------+
| NULL |
| 0 |
| 4 |
+--------+
select number,count(*) from t1 group by number;
+--------+----------+
| number | count(*) |
+--------+----------+
| NULL | 2 |
| 0 | 1 |
| 4 | 1 |
+--------+----------+
**字符类型，在使用 NULL 值的时候，也需要格外注意**
**1) **字段是字符时，你无法一目了然的区分这个值到底是 NULL ，还是字符串 &#8216;NULL&#8217;
insert into t1 (name,number) values ('NULL',5);
insert into t1 (number) values (6);
select * from t1 where number in (5,6);
+----+------+--------+
| id | name | number |
+----+------+--------+
| 5 | NULL | 5 |
| 6 | NULL | 6 |
+----+------+--------+
select name is NULL from t1 where number=5;
+--------------+
| name is NULL |
+--------------+
| 0 |
+--------------+
select name is NULL from t1 where number=6;
+--------------+
| name is NULL |
+--------------+
| 1 |
+--------------+
**2) 统计包含 NULL 字段的值，NULL 值不包括在里面**
select count(*) from t1;
+----------+
| count(*) |
+----------+
| 6 |
+----------+
select count(name)from t1;
+-------------+
| count(name) |
+-------------+
| 5 |
+-------------+
select * from t1 where name is null;
+----+------+--------+
| id | name | number |
+----+------+--------+
| 6 | NULL | 6 |
+----+------+--------+
**3) 如果你用 length 去统计一个 VARCHAR 的长度时，NULL 返回的将不是数字**
select length(name) from t1 where name is null;
+--------------+
| length(name) |
+--------------+
| NULL |
+--------------+
**总结：**
NULL 本身是一个特殊值，MySQL 采用特殊的方法来处理 NULL 值。从理解肉眼判断，操作符运算等操作上，可能和我们预期的效果不一致。可能会给我们项目上的操作不符合预期。
你必须要使用 IS NULL / IS NOT NULL 这种与普通 SQL 大相径庭的方式去处理 NULL。
尽管在存储空间上，在索引性能上可能并不比空值差，但是为了避免其身上特殊性，给项目带来不确定因素，**因此建议默认值不要使用 NULL**。
**近期社区动态**
![](https://opensource.actionsky.com/wp-content/uploads/2019/08/海报.jpg)