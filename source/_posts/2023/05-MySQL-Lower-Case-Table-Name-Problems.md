---
title: （待完成）表不存在：MySQL 中 lower_case_table_names 问题探究
date: 2023-03-10 20:03:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/mysql_study.png

# author information, multiple authors are set to array
# single author
author:
- nick: Bhuvanes Waran
  link: https://www.percona.com/blog/author/bhuvanes-waran/

# post subtitle in your index page
subtitle: 本文是一篇翻译文章，在这篇博客文章中，我们将学习如何借助 `performance schema` 中 `Instrument` 工具来排查 `MySQL` 数据库的一些问题。

categories:
- 数据库

tags:
- MySQL
- Performance Schema
---

> 原文链接：https://www.percona.com/blog/table-doesnt-exist-mysql-lower_case_table_names-problems/

在 [Managed Services](https://www.percona.com/services/managed-services) 系统中，我们有很多客户，由于每个客户都有不同的系统环境和配置，因此在他们的环境中工作总是很有趣。在这篇博文中，我将展示我们在删除表时遇到的一个问题及其解决方法。

在客户的生产环境（`MySQL 5.7`）中有一个需要被删除的表都有一个标识，表名称以 `#` 符号开头。我认为我们可以很容易地使用引号或反引号来指定要删除的表。但实际效果并没有像我预期的那样，我才知道为什么客户将可以删除的表进行了标识。

以下示例重现了该问题。这里显示了一张表，但是我们却无法看到它的结构，也无法删除它。

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

```shell
[root@centos12 percona]# ls -lrth
total 112K
-rw-r-----. 1 mysql mysql   65 Dec 20 02:44 db.opt
-rw-r-----. 1 mysql mysql 8.4K Dec 20 02:51 @0023Tableau_01_bw_F2DD_test.frm
-rw-r-----. 1 mysql mysql  96K Dec 20 02:51 @0023Tableau_01_bw_F2DD_test.ibd
[root@centos12 percona]# pwd
/var/lib/mysql/percona
[root@centos12 percona]# 
```
