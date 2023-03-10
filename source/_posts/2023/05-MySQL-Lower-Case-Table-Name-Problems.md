---
title: 表不存在：MySQL 中 lower_case_table_names 问题探究
date: 2023-03-10 20:03:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/mysql_study_2.png

# author information, multiple authors are set to array
# single author
author:
- nick: Bhuvanes Waran
  link: https://www.percona.com/blog/author/bhuvanes-waran/

# post subtitle in your index page
subtitle: 在这篇博客文章中，我们将排查一个由于 lower_case_table_names 配置变更引起的问题。

categories:
- 数据库

tags:
- MySQL
---

> 原文链接：https://www.percona.com/blog/table-doesnt-exist-mysql-lower_case_table_names-problems/

在 [Managed Services](https://www.percona.com/services/managed-services) 系统中，我们有很多客户，由于每个客户都有不同的系统和配置环境，因此在他们的环境中工作总是会发现一些有趣事情。在这篇博文中，我将展示在删除表时遇到的一个问题及其解决方法。

在客户的生产环境（`MySQL 5.7`）中需要被删除的表都有一个标识，表名称以 `#` 符号开头。我认为可以很容易地使用引号或反引号来指定要删除的表。但实际效果并没有像我预期的那样，我才知道为什么客户将可以删除的表进行了标识。

以下示例重现了该问题。示例中显示有一张表，但是我们无法查看表结构，也无法删除它。

```sql
mysql> show tables;
+--------------------------+
| Tables_in_percona        |
+--------------------------+
| #Tableau_01_bw_F2DD_test |
+--------------------------+
1 row in set (0.00 sec)

mysql> show create table `#Tableau_01_bw_F2DD_test`G
ERROR 1146 (42S02): Table 'percona.#tableau_01_bw_f2dd_test' doesn't exist
mysql> 
mysql> drop table `#Tableau_01_bw_F2DD_test`;
ERROR 1051 (42S02): Unknown table 'percona.#tableau_01_bw_f2dd_test'
mysql> 
mysql> drop table '#Tableau_01_bw_F2DD_test';
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''#Tableau_01_bw_F2DD_test'' at line 1
mysql> 
mysql> drop table "#Tableau_01_bw_F2DD_test";
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '"#Tableau_01_bw_F2DD_test"' at line 1
mysql> 
```

检查 `.ibd` 和 `.frm` 文件时，这些文件都是存在的，并且磁盘上物理文件没有问题。

```sql
[root@centos12 percona]# ls -lrth
total 112K
-rw-r-----. 1 mysql mysql   65 Dec 20 02:44 db.opt
-rw-r-----. 1 mysql mysql 8.4K Dec 20 02:51 @0023Tableau_01_bw_F2DD_test.frm
-rw-r-----. 1 mysql mysql  96K Dec 20 02:51 @0023Tableau_01_bw_F2DD_test.ibd
[root@centos12 percona]# pwd
/var/lib/mysql/percona
[root@centos12 percona]# 
```

我认为这个问题是由于 `#` 标识引起的，所以我想创建一张带有 `#` 标识的表然后尝试删除它。但是令人惊讶的是我们可以创建表也可以删除它。但是，我们仍然未能删除客户提供的数据表。

```sql
mysql> show tables;
+--------------------------+
| Tables_in_percona        |
+--------------------------+
| #Tableau_01_bw_F2DD_test |
| #tableau_01__test        |
+--------------------------+
2 rows in set (0.00 sec)

mysql> Drop table `#Tableau_01__Test`;
Query OK, 0 rows affected (0.01 sec)

mysql> show tables;
+--------------------------+
| Tables_in_percona        |
+--------------------------+
| #Tableau_01_bw_F2DD_test |
+--------------------------+
1 row in set (0.00 sec)

mysql> Drop table `#Tableau_01_bw_F2DD_test`;
ERROR 1051 (42S02): Unknown table 'percona.#tableau_01_bw_f2dd_test'
mysql> show create table `#Tableau_01_bw_F2DD_test`G
ERROR 1146 (42S02): Table 'percona.#tableau_01_bw_f2dd_test' doesn't exist
mysql> 
mysql> show global variables like '%lower_case_table%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| lower_case_table_names | 1     |
+------------------------+-------+
1 row in set (0.00 sec)
```

在这里我们注意到一件事——我们以大写字母创建数据表，而在显示表时它显示的是小写字母。 这给了我们检查 `lower_case_table_names` 配置的启示，它被设置为 `1`（在 `Unix` 中默认为 `0`）。
所以我想尝试将这个配置设置为某个值场景下中创建一张表，然后再将它设置为另一个值时将表删除。

`lower_case_table_names` 属性值及其行为是：

- **0**：数据表和数据库名称使用 `CREATE TABLE` 或 `CREATE DATABASE` 语句中指定的字母大小写存储在磁盘上。名称比较会区分大小写。如果在文件名不区分大小写的系统（例如 `Windows` 或 `MacOS`）上运行 `MySQL`，则不应将此变量设置为 `0`。如果在不区分大小写的文件系统上使用 `−−lower-case-table-names=0` 强制此配置设置为 `0` 并使用不同的字母大小写访问 `MyISAM` 表名，可能会导致索引文件损坏。

- **1**：表名都以小写形式存储在磁盘上，名称比较不区分大小写。`MySQL` 在存储和查找时将所有表名转换为小写。此行为也适用于数据库名称和表别名。

- **2**：表和数据库名称使用 `CREATE TABLE` 或 `CREATE DATABASE` 语句中指定的字母大小写存储在磁盘上，但 `MySQL` 在查找时将它们转换为小写字母。名称比较不区分大小写。这仅适用于不区分大小写的文件系统！ `InnoDB` 表名和视图名以小写形式存储，和 `lower_case_table_names=1` 配置效果一致。

### 场景一：lower_case_table_names=0 时创建表，lower_case_table_names=1 时删除表

设置 `lower_case_table_names=0` 并创建名称包含大写字母的数据表和数据库。

```sql
mysql> use percona;
Database changed
mysql> create table `#Table1_test2`(ct INT primary key);
Query OK, 0 rows affected (0.02 sec)

mysql> create table `#table1_test2`(ct INT primary key);
Query OK, 0 rows affected (0.01 sec)

mysql> create table `Table1_test3`(ct INT primary key);
Query OK, 0 rows affected (0.02 sec)

mysql> create table `table1_test2`(ct INT primary key);
Query OK, 0 rows affected (0.01 sec)

mysql> show tables;
+-------------------+
| Tables_in_percona |
+-------------------+
| #Table1_test2     |
| #table1_test2     |
| Table1_test3      |
| table1_test2      |
+-------------------+
4 rows in set (0.00 sec)

mysql> show global variables like '%lower_case_table%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| lower_case_table_names | 0     |
+------------------------+-------+
1 row in set (0.00 sec)

mysql> create database Test_database;
Query OK, 1 row affected (0.00 sec)

mysql> show global variables like '%lower_case_table%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| lower_case_table_names | 0     |
+------------------------+-------+
1 row in set (0.00 sec)
```

为了修改 `lower_case_table_names` 值为 `1`，我不仅修改了配置文件，而且重启了 `MySQL` 服务。当 `lower_case_table_names` 设置为 `1` 时，可以看到第一次删除表 `#Table1_test2` 是成功的并且有结果显示，但是删除表 `#table1_test2` 失败了，并且没有显示到 `show tables` 列表中。这是由于设置不区分大小写，因为无论我们在哪里使用大写字母，它都只会以小写字母进行查找。这就是在删除表 `#Table1_test2` 时表 `#table1_test2` 被删除的原因。

我们无法使用 `Test_database` 数据库，因为它是以大写形式创建的。因此，无论数据库中存在什么样的表，我们都将无法访问。简而言之，当设置 `lower_case_table_names=1` 时，大写字母出现在表和数据库中时，它们将被视为小写字母。

```sql
mysql> use percona;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+-------------------+
| Tables_in_percona |
+-------------------+
| #Table1_test2     |
| #table1_test2     |
| Table1_test3      |
| table1_test2      |
+-------------------+
4 rows in set (0.00 sec)

mysql> Drop table `#Table1_test2`;
Query OK, 0 rows affected (0.01 sec)

mysql> drop table `#table1_test2`;
ERROR 1051 (42S02): Unknown table 'percona.#table1_test2'
mysql> drop table Table1_test3;
ERROR 1051 (42S02): Unknown table 'percona.table1_test3'
mysql> drop table table1_test2;
Query OK, 0 rows affected (0.01 sec)

mysql> show tables;
+-------------------+
| Tables_in_percona |
+-------------------+
| #Table1_test2     |
| Table1_test3      |
+-------------------+
2 rows in set (0.00 sec)

mysql> drop table `#Table1_test2`;
ERROR 1051 (42S02): Unknown table 'percona.#table1_test2'
mysql> show global variables like '%lower_case_table%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| lower_case_table_names | 1     |
+------------------------+-------+
1 row in set (0.00 sec)

mysql> show schemas;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| Test_database      |
| mysql              |
| percona            |
| performance_schema |
| sys                |
+--------------------+
6 rows in set (0.00 sec)

mysql> use Test_database;
ERROR 1049 (42000): Unknown database 'test_database'
mysql> use test_database;
ERROR 1049 (42000): Unknown database 'test_database'
mysql> use `Test_database`;
ERROR 1049 (42000): Unknown database 'test_database'
```

### 场景二：lower_case_table_names=1 时创建表，lower_case_table_names=0 时删除表

当我们尝试创建名称包含大写字母的表和数据库，最终只创建了小写字母内容。`#table1_test2` 表创建失败，提示表已存在错误，因为 `#Table1_test2` 的第一个建表语句表名称被转换为小写并创建了表 `#table1_test2`。
创建表 `Table1_test3` 成功时也是如此，创建名称为 `table1_test3` 的表，再次创建表 `table1_test3` 时提示失败。

```sql
mysql> create database lower1_to_Lower0;
Query OK, 1 row affected (0.00 sec)

mysql> show schemas;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| Test_database      |
| lower1_to_lower0   |
| mysql              |
| percona            |
| performance_schema |
| sys                |
+--------------------+
7 rows in set (0.00 sec)
mysql> use lower1_to_lower0;
Database changed
mysql> create table `#Table1_test2`(ct INT primary key);
Query OK, 0 rows affected (0.02 sec)

mysql> create table `#table1_test2`(ct INT primary key);
ERROR 1050 (42S01): Table '#table1_test2' already exists
mysql> create table `Table1_test3`(ct INT primary key);
Query OK, 0 rows affected (0.02 sec)

mysql> create table `table1_test3`(ct INT primary key);
ERROR 1050 (42S01): Table 'table1_test3' already exists
mysql> show tables;
+----------------------------+
| Tables_in_lower1_to_lower0 |
+----------------------------+
| #table1_test2              |
| table1_test3               |
+----------------------------+
2 rows in set (0.00 sec)
```

要将 `lower_case_table_names` 的值从 `1` 更改为 `0`，我只是更改了配置中的值并重新启动了 `MySQL` 服务。当 `lowercase_table_name=0` 时，我们能够删除表和数据库，因为数据库和表不是用大写字母创建的。

```sql
mysql> show global variables like '%lower_case_table%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| lower_case_table_names | 0     |
+------------------------+-------+
1 row in set (0.00 sec)


mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| Test_database      |
| lower1_to_lower0   |
| mysql              |
| percona            |
| performance_schema |
| sys                |
+--------------------+
7 rows in set (0.00 sec)

mysql> use lower1_to_lower0;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+----------------------------+
| Tables_in_lower1_to_lower0 |
+----------------------------+
| #table1_test2              |
| table1_test3               |
+----------------------------+
2 rows in set (0.00 sec)

mysql> drop table `#table1_test2`;
Query OK, 0 rows affected (0.01 sec)

mysql> drop table `table1_test3`;
Query OK, 0 rows affected (0.01 sec)

mysql> show tables;
Empty set (0.01 sec)

mysql> drop database lower1_to_lower0;
Query OK, 0 rows affected (0.00 sec)

mysql> show schemas;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| Test_database      |
| mysql              |
| percona            |
| performance_schema |
| sys                |
+--------------------+
6 rows in set (0.00 sec)
```

场景一是我们在客户端环境中遇到的场景。客户很久以前在 `lower_case_table_names=0` 时创建了该表，一段时间后他们将配置更改为 `lower_case_table_names=1`。所以我们经过批准，先设置 `lower_case_table_names=0` 并删除了表，然后再将配置恢复为 `1`。由于这个配置修改不是动态的，因此我们需要重启服务。

### 总结

我不建议将 `lower_case_table_names` 设置为 `1` 或 `0`，因为这是基于应用程序的要求。但在将配置从 `0` 更改为 `1` 之前，需要仔细检查是否有大写的表或数据库。如果存在的话，需要将它们转换为小写，否则那些名称包含大写的表和数据库将无法访问。将配置从 `1` 更改为 `0` 时，我们没有遇到问题，因为不允许以大写字母创建表和数据库，并且完全忽略大小写仅以小写字母创建。
在 `MySQL 8.0` 中，我们无法在创建数据库实例后更改 `lower_case_table_names` 配置值，因为此变量的值会影响数据字典表的定义，所以在服务器初始化后将不能被修改。
