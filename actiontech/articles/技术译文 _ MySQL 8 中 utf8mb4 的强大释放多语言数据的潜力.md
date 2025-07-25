# 技术译文 | MySQL 8 中 utf8mb4 的强大：释放多语言数据的潜力

**原文链接**: https://opensource.actionsky.com/%e6%8a%80%e6%9c%af%e8%af%91%e6%96%87-mysql-8-%e4%b8%ad-utf8mb4-%e7%9a%84%e5%bc%ba%e5%a4%a7%ef%bc%9a%e9%87%8a%e6%94%be%e5%a4%9a%e8%af%ad%e8%a8%80%e6%95%b0%e6%8d%ae%e7%9a%84%e6%bd%9c%e5%8a%9b/
**分类**: MySQL 新特性
**发布时间**: 2023-12-11T01:32:32-08:00

---

在现代 Web 应用程序世界中，支持多种语言和字符集变得越来越重要。随着全球化的兴起，存储和处理多语言数据的需求变得至关重要。MySQL 作为最流行的关系数据库管理系统之一，认识到了这一需求，并在其 8.0 版本中引入了 utf8mb4。在这篇文章中，我们将通过实际示例探讨 utf8mb4 及其在 MySQL 8 中的优势。
> 作者：Arunjith Aravindan
本文原文：[https://www.percona.com/blog/the-power-of-utf8mb4-in-mysql-8-0-unleashing-the-full-potential-of-multilingual-data/](https://www.percona.com/blog/the-power-of-utf8mb4-in-mysql-8-0-unleashing-the-full-potential-of-multilingual-data/)
本文约 2000 字，预计阅读需要 7 分钟。
 
![](https://opensource.actionsky.com/wp-content/uploads/2023/12/🐉.png)
# 了解 utf8mb4
在深入探讨 utf8mb4 的好处之前，我们先澄清一下 utf8mb4 代表什么。在 MySQL 中，“utf8”是指支持 Unicode 字符集的字符编码，每个字符最多使用三个字节。然而，MySQL 中原始的 utf8 实现并没有涵盖所有 Unicode 字符。另一方面，utf8mb4 是 utf8 的修改版本，它支持完整的 Unicode 字符集，包括表情符号和其他补充字符，每个字符最多使用四个字节。
MySQL 中原始的 utf8 实现仅支持基本多文种平面（BMP）中的字符，大约占所有 Unicode 字符的 90%。另一方面，utf8mb4 支持整个 Unicode 字符集，包括表情符号和其他补充字符。它通过每个字符最多使用四个字节而不是 utf8 使用的三个字节来实现此目的。
下表显示了 utf8 和 utf8mb4 之间的区别：
| 特征 | UTF8 | utf8mb3 | utf8mb4 |
| --- | --- | --- | --- |
| 每个字符的最大字节数 | 3 | 3 | 4 |
| 支持的字符 | 基本多文种平面 (BMP) | BMP | BMP + 辅助平面 |
| MySQL 默认 | Yes | Yes | Yes（MySQL 8.0 开始) |
| 状态 | 已弃用 | 已弃用 | 未弃用 |
*注意：历史上，MySQL 使用字符集 utf8 作为 utf8mb3 的别名。但是，从 MySQL 8.0.28 开始，utf8mb3 仅在 SHOW 语句的输出和信息架构表中引用该字符集时使用。未来，utf8 有望成为 utf8mb4 的参考。为了避免任何歧义，建议在引用该字符集时显式指定 utf8mb4。*
如您所见，**utf8、utf8mb3 和 utf8mb4 之间的主要区别在于每个字符的最大字节数。** utf8 和 utf8mb3 只能存储 BMP 中的字符，而 utf8mb4 还可以存储补充平面（Supplementary Plane）中的字符。这意味着 utf8mb4 可以支持更广泛的字符，包括表情符号、数学符号和其他特殊字符。
这三个字符集之间的另一个区别是它们在 MySQL 中的默认状态。utf8 是 MySQL 5.7 及更早版本中的默认字符集，而 utf8mb3 是 MySQL 8.0 中的默认字符集。但是，utf8mb4 是 MySQL 8.0.28 及更高版本中的默认字符集。
最后，MySQL 8.0 中已弃用 utf8 和 utf8mb3。这意味着它们最终将从 MySQL 中删除，因此建议使用 utf8mb4 代替。
因此，如果您需要存储所有 Unicode 字符，包括表情符号和其他补充字符，那么您应该使用 utf8mb4。但是，如果您只需要存储 BMP 中的字符，那么 utf8 可能就足够了。
以下是使用 MySQL 表和查询对 utf8 和 utf8mb4 进行比较的示例：
# 对比示例
## MySQL 5.7
`mysql> select version();
+-----------+
| version() |
+-----------+
| 5.7.42-46 |
+-----------+
`
### Table
```
mysql> CREATE TABLE users (
id INT AUTO_INCREMENT PRIMARY KEY,
name VARCHAR(255) CHARACTER SET utf8,
email VARCHAR(255) CHARACTER SET utf8
);
Query OK, 0 rows affected (0.03 sec)
```
```
mysql> show create table usersG
*************************** 1. row ***************************
Table: users
Create Table: CREATE TABLE `users` (
`id` int(11) NOT NULL AUTO_INCREMENT,
`name` varchar(255) CHARACTER SET utf8 DEFAULT NULL,
`email` varchar(255) CHARACTER SET utf8 DEFAULT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
1 row in set (0.01 sec)
```
在用户表中插入三行数据，包括 emoji 表情符号。
`mysql> INSERT INTO users (name, email) VALUES
('Arun Jith', 'arunjith@example.com'),
('Jane Doe', 'janedoe@example.com'),
('𝌆', 'emoji@example.com');
ERROR 1366 (HY000): Incorrect string value: 'xF0x9Dx8Cx86' for column 'name' at row 3
mysql>
`
遇到的错误消息 `ERROR 1366 (HY000): Incorrect string value: ‘xF0x9Dx8Cx86’ for column ‘name’ at row 3,` 第 3 行的 `name` 字段的字符编码存在问题。用户表尝试将 Unicode 字符 `𝌆` 插入 `name` 字段时发生错误。
`mysql> INSERT INTO users (name, email) VALUES
('Arun Jith', 'arunjith@example.com'),
('Jane Doe', 'janedoe@example.com')
;
Query OK, 2 rows affected (0.00 sec)
Records: 2  Duplicates: 0  Warnings: 0
`
## MySQL 8.0
```
mysql> select version();
+-------------------------+
| version()               |
+-------------------------+
| 8.0.33-0ubuntu0.22.04.2 |
+-------------------------+
```
### Table
```
CREATE TABLE users (
id INT AUTO_INCREMENT PRIMARY KEY,
name VARCHAR(255) CHARACTER SET utf8,
email VARCHAR(255) CHARACTER SET utf8
);
```
```
mysql> show create table usersG
*************************** 1. row ***************************
Table: users
Create Table: CREATE TABLE `users` (
`id` int NOT NULL AUTO_INCREMENT,
`name` varchar(255) CHARACTER SET utf8mb3 COLLATE utf8mb3_general_ci DEFAULT NULL,
`email` varchar(255) CHARACTER SET utf8mb3 COLLATE utf8mb3_general_ci DEFAULT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.00 sec)
```
该表的 `name` 和 `email` 字段均使用 utf8mb3 字符集。这意味着该表可以存储 BMP 中的所有字符，但不能存储表情符号或其他补充字符。
### Query
`INSERT INTO users (name, email) VALUES
('Arun Jith', 'arunjith@example.com'),
('Jane Doe', 'janedoe@example.com'),
('𝌆', 'emoji@example.com');
`
与前面的示例一样的错误（ `ERROR 1366 (HY000): Incorrect string value: ‘xF0x9Dx8Cx86’ for column ‘name’ at row 3,`）。
`mysql> INSERT INTO users (name, email) VALUES
-> ('Arun Jith', 'arunjith@example.com'),
-> ('Jane Doe', 'janedoe@example.com'),
-> ('𝌆', 'emoji@example.com');
ERROR 1366 (HY000): Incorrect string value: 'xF0x9Dx8Cx86' for column 'name' at row 3
`
```
mysql> INSERT INTO users (name, email) VALUES
-> ('Arun Jith', 'arunjith@example.com'),
-> ('Jane Doe', 'janedoe@example.com')
-> ;
Query OK, 2 rows affected (0.00 sec)
Records: 2  Duplicates: 0  Warnings: 0
```
此查询将前两行数据插入用户表中。前两行包含简单的文本数据，而第三行包含emoji 表情符号。表情符号将无法正确存储在数据库中，因为 utf8 字符集无法存储 emoji 表情符号。
### Output
`mysql> SELECT * FROM users;
+----+-----------+----------------------+
| id | name      | email                |
+----+-----------+----------------------+
|  4 | Arun Jith | arunjith@example.com |
|  5 | Jane Doe  | janedoe@example.com  |
+----+-----------+----------------------+
2 rows in set (0.00 sec)
`
此查询将从用户表中选择两行。查询的输出将是用户表中所有行的列表，包括每个用户的 `ID`、`name`、`email`。第三行有表情符号的无法存储，插入时出错，因为 utf8 字符集无法存储表情符号。
### Table
为了确保正确存储表情符号，让我们使用 utf8mb4 字符集创建表列。之后，我们可以继续检查表情符号插入是否正确。
`mysql> CREATE TABLE users (
->   id INT AUTO_INCREMENT PRIMARY KEY,
->   name VARCHAR(255) CHARACTER SET utf8mb4,
->   email VARCHAR(255) CHARACTER SET utf8mb4
-> );
Query OK, 0 rows affected (0.03 sec)
`
```
mysql> show create table usersG
*************************** 1. row ***************************
Table: users
Create Table: CREATE TABLE `users` (
`id` int NOT NULL AUTO_INCREMENT,
`name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL,
`email` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
1 row in set (0.00 sec)
```
### Query
```
INSERT INTO users (name, email) VALUES
('Arun Jith', 'arunjith@example.com'),
('Jane Doe', 'janedoe@example.com'),
('𝌆', 'emoji@example.com');
```
```
mysql> INSERT INTO users (name, email) VALUES
-> ('Arun Jith', 'arunjith@example.com'),
-> ('Jane Doe', 'janedoe@example.com'),
-> ('𝌆', 'emoji@example.com');
Query OK, 3 rows affected (0.01 sec)
Records: 3  Duplicates: 0  Warnings: 0
```
该表对 `name` 和 `email` 列均使用 utf8mb4 字符集。这意味着该表可以存储完整 Unicode 字符集中的所有字符，包括 emoji 表情符号和其他补充字符。
此查询将三行数据插入用户表中。前两行包含简单的文本数据，而第三行包含表情符号。表情符号将正确存储在数据库中，因为 utf8mb4 字符集可以存储表情符号。
### Output
`mysql> SELECT * FROM users;
+----+-----------+----------------------+
| id | name      | email                |
+----+-----------+----------------------+
|  1 | Arun Jith | arunjith@example.com |
|  2 | Jane Doe  | janedoe@example.com  |
|  3 | 𝌆         | emoji@example.com    |
+----+----------+-----------------------+
3 rows in set (0.00 sec)
`
此查询将从用户表中选择所有行。查询的输出将是用户表中所有行的列表，包括每个用户的 `ID`、`name`、`email`。表情符号将被存储为表情符号，因为 utf8mb4 字符集可以存储表情符号。
# 总结
如您所见，utf8mb4 字符集可以存储完整 Unicode 字符集中的所有字符，包括表情符号和其他补充字符。这使得它成为存储复杂文本数据、文本搜索和比较的不错选择。另一方面，utf8 字符集只能存储 BMP 中的字符。这意味着它无法存储表情符号或其他补充字符。
**一般来说，建议所有新应用程序都使用 utf8mb4。这将确保您的数据可以正确存储和处理，无论它包含什么字符。**