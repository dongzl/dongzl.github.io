---
title: MySQL 中 COUNT(*) 和 COUNT(col) 对比
date: 2023-02-04 15:37:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/mysql_study_2.png
# author information, multiple authors are set to array
# single author
author:
- nick: Denis Subbota
  link: https://www.percona.com/blog/author/denis-subbota/

# post subtitle in your index page
subtitle: 在这篇博客文章中，我们将对比在 MySQL 数据库中 COUNT(*) 和 COUNT(col) 的不同之处。

categories:
- 数据库

tags:
- MySQL
---

> 原文链接：https://www.percona.com/blog/count-vs-countcol-in-mysql/

如果我们观察一下大家是如何使用 `COUNT(*)` 和 `COUNT(col)` 的，大多数人都认为这两个操作是等价的，使用哪个只是个人的喜好不同；其实恰恰相反，这两个函数操作在查询性能甚至查询结果上是有很大的差异的。此外，我们还发现在 `InnoDB` 和 `MyISAM` 引擎上执行也存在很大差异。

> 注意：所有的测试都是在 MySQL 的 8.0.30 版本上完成的，在后台，我运行了每一个查询三到五次，以确保所有这些查询操作都完全在缓冲池（对于 InnoDB）或文件系统（对于 MyISAM）缓存中完成。

## InnoDB 引擎中 COUNT 函数

让我们观察一下 `InnoDB` 引擎的以下一系列示例：

```SQL
CREATE TABLE count_innodb (
  id int(10) unsigned NOT NULL AUTO_INCREMENT,
  val_with_nulls int(11) default NULL,
  val_no_null int(10) unsigned NOT NULL,
  PRIMARY KEY idx (id)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;

(mysql) > select count(*) from count_innodb;
+----------+
| count(*) |
+----------+
| 10000000 |
+----------+
1 row in set (0.38 sec)

(mysql) > select count(val_no_null) from count_innodb;
+--------------------+
| count(val_no_null) |
+--------------------+
|           10000000 |
+--------------------+
1 row in set (0.38 sec)
```

在 `InnoDB` 存储引擎中，我们可以发现，通过 `COUNT(*)` 和 `COUNT(val_no_null)` 操作来获取表的行数是需要一定时间的，在后面的内容我们还会提到，在获取 `COUNT(*)` 的结果方面，`MyISAM` 引擎比 `InnoDB` 引擎要快得多。

为什么我们不能缓存实际的行数呢？`InnoDB` 引擎并没有在内部保存表行数，因为在并发的事务场景下，在同一时刻执行的结果可能是不同的。因此，`SELECT` `COUNT(*)` 语句仅统计当前事务可见的行。顺便说一句，我们可以使用 `MySQL` 的 `information_schema` 立即获得有关表的大致行数：

```SQL
(mysql) >  select table_rows from information_schema.tables where table_name='count_innodb';
+------------+
| TABLE_ROWS |
+------------+
|    9980586 |
+------------+
1 row in set (0.00 sec)
```

正如我们所看到的，这个执行结果并不是准确的。但是在有些场景下粗略的统计结果就足够了。

下面我们来看一下 `COUNT(val_with_nulls)` 操作：

```SQL
(mysql) >  select count(val_with_nulls) from count_innodb;
+-----------------------+
| count(val_with_nulls) |
+-----------------------+
|               9990001 |
+-----------------------+
1 row in set (2.14 sec)
```

在这里我们看到，`COUNT(*)` 与 `COUNT(val_with_nulls)` 的执行结果存在差异。

为什么会这样呢？因为 `val_with_nulls` 字段被没有被定义为 `NOT NULL`，所以这个字段上存在一部分 `NULL` 值，`MySQL` 必须要执行全表扫描找到这些 `NULL` 值的行并在结果中排除掉，所以第二个查询执行结果不同。

因此 `COUNT(*)` 和 `COUNT(col)` 查询不仅可能具有显着的性能差异，而且还会提出不同的问题。

下面我们来看下面一系列查询，我们将对比一下，在带有 `WHERE` 条件的查询中，`InnoDB` 引擎是如何处理 `COUNT(*)`, `COUNT(val_no_null)`, `COUNT(val_with_nulls)` 的：

```SQL
(mysql) >  select count(*) from count_innodb where id<1000000;
+----------+
| count(*) |
+----------+
|   980000 |
+----------+
1 row in set (0.30 sec)

(mysql) > explain select count(*) from count_innodb where id<1000000\G
*************************** 1. row ***************************
  select_type: SIMPLE
        table: count_innodb
         type: range
possible_keys: PRIMARY
          key: PRIMARY
         rows: 1955802
     filtered: 100.00
        Extra: Using where; Using index

(mysql) >  select count(val_no_null) from count_innodb where id<1000000;
+--------------------+
| count(val_no_null) |
+--------------------+
|             980000 |
+--------------------+
1 row in set (0.33 sec)

(mysql) >  explain select count(val_no_null) from count_innodb where id<1000000\G
*************************** 1. row ***************************
  select_type: SIMPLE
        table: count_innodb
         type: range
possible_keys: PRIMARY
          key: PRIMARY
         rows: 2013804
     filtered: 100.00
        Extra: Using where
```

我们可以看到在这两种情况下，查询的性能都是相同的，并且只有 `10%` 的差异；如果我们使用 `EXPLAIN` 更进一步观察 `COUNT(*)` 查询的执行计划，就会注意到执行计划中使用了索引。这意味着 `MySQL` 只需要使用索引，而不用获取其余表数据，就可能足以获得一张大表的行数。

我们可能希望使用已有索引的列来加快对大型表的查询。

那 `COUNT(val_with_nulls)` 会给我们带来什么惊喜吗？让我们看一下：

```SQL
(mysql) > select count(val_with_nulls) from count_innodb where id<1000000;
+-----------------------+
| count(val_with_nulls) |
+-----------------------+
|                970001 |
+-----------------------+
1 row in set (0.33 sec)

(mysql) > explain select count(val_with_nulls) from count_innodb where id<1000000\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: count_innodb
   partitions: NULL
         type: range
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: NULL
         rows: 1955802
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```

没有任何惊喜；我们可以看到，在所有 `COUNT(*)`、`COUNT(val_no_null)` 和 `COUNT(val_with_nulls)` 查询中，性能都相当均匀。

## MyISAM 引擎中 COUNT 函数

下面我们来看一下在 `MyISAM` 引擎中的 `COUNT()` 函数操作：

```SQL
CREATE TABLE count_myisam (
  id int(10) unsigned NOT NULL,
  val_with_nulls int(11) default NULL,
  val_no_null int(10) unsigned NOT NULL,
  KEY idx (id)
) ENGINE=MyISAM DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

(mysql) > select count(*) from count_myisam;
+----------+
| count(*) |
+----------+
| 10000000 |
+----------+
1 row in set (0.00 sec)

(mysql) > select count(val_no_null) from count_myisam;
+--------------------+
| count(val_no_null) |
+--------------------+
|           10000000 |
+--------------------+
1 row in set (0.00 sec)
```

我们看到了飞快的执行速度！

由于这是一个 `MyISAM` 引擎表，在引擎内部缓存了表的行数，这就是 `MyISAM` 引擎的工作方式；这就是为什么它可以立即返回 `COUNT(*)` 和 `COUNT(val_no_null)` 的查询结果。

请注意引擎之间的区别：`InnoDB` 是一个事务存储引擎，`MyISAM` 是一个非事务存储引擎。

```SQL
(mysql) > select count(val_with_nulls) from count_myisam;
+-----------------------+
| count(val_with_nulls) |
+-----------------------+
|               9990001 |
+-----------------------+
1 row in set (14.18 sec)
```

但是当涉及到 `MyISAM` 表的 `COUNT(val_with_nulls)` 时，我们可以看到比 `InnoDB` 慢了 `7` 倍；多么巨大的差异。此外，我们可以看到 `COUNT(val_with_nulls)` 的相同行为，因为 `NULL` 值显然不会被考虑。 在这种情况下，`MySQL` 优化器做得很好，只有在需要时才进行全表扫描，因为列值可能为 `NULL`。

现在，让我们尝试使用 `WHERE` 子句对 `MyISAM` 表进行更多查询：

```SQL
(mysql) >  select count(*) from count_myisam where id<1000000;
+----------+
| count(*) |
+----------+
|  1001237 |
+----------+
1 row in set (0.41 sec)

(mysql) >  explain select count(*) from count_myisam where id<1000000 \\G
*************************** 1. row ***************************
  select_type: SIMPLE
        table: count_myisam
         type: range
possible_keys: idx
          key: idx
         rows: 1041561
     filtered: 100.00
        Extra: Using where; Using index

(mysql) > select count(val_no_null) from count_myisam where id<1000000;
+--------------------+
| count(val_no_null) |
+--------------------+
|            1001237 |
+--------------------+
1 row in set (2.55 sec)

(mysql) >  explain select count(val_no_null) from count_myisam where id<1000000\\G
*************************** 1. row ***************************
  select_type: SIMPLE
        table: count_myisam
         type: range
possible_keys: idx
          key: idx
         rows: 1041561
     filtered: 100.00
        Extra: Using index condition; Using MRR
(mysql) >  select count(val_with_nulls) from count_myisam where id<1000000;
+-----------------------+
| count(val_with_nulls) |
+-----------------------+
|               1000281 |
+-----------------------+
1 row in set (2.55 sec)

(mysql) >  explain select count(val_with_nulls) from count_myisam where id<1000000\\G
*************************** 1. row ***************************
  select_type: SIMPLE
        table: count_myisam
         type: range
possible_keys: idx
          key: idx
         rows: 1041561
     filtered: 100.00
        Extra: Using index condition; Using MRR
```

正如我们所看到的，即使是带有 `WHERE` 条件的查询，`COUNT(*)` 和 `COUNT(col)` 的性能也会有很大不同。事实上，这个示例显示了 `5` 倍的性能差异，因为所有数据都在内存中命中（对于目前场景，由于是使用 `MyISAM` 引擎，数据缓存发生在文件系统缓存级别）。如果 `IO` 工作负载很大，在这种情况下，我们可以看到甚至 `100` 倍的性能差异。

`COUNT(*)` 查询能够使用覆盖索引，而 `COUNT(col)` 是不行的。当然，我们可以将索引扩展为 `(id, val_with_nulls)`，再次查询就可以使用覆盖索引；但只有在无法更改查询语句（例如：它是第三方应用程序）或列名出于某种原因出现在查询中，并且需要计算非 `NULL` 值的行数时，我才会使用此解决方法。

值得注意的是在这种情况下，`MySQL` 优化器在优化查询方面做得不够好。我们可能会注意到 `(val_with_nulls)` 列不为空，因此 `COUNT(val_with_null)` 与 `COUNT(*)` 查询结果相同；因此可以使用覆盖索引进行查询操作，但是它不会，在这种情况下，两个查询都必须执行行读取。

> **我认为原文这里有误，正确的内容应该是：值得注意的是在这种情况下，`MySQL` 优化器在优化查询方面做得不够好。我们可能会注意到 `(val_no_null)` 列不为空，因此 `COUNT(val_no_null)` 与 `COUNT(*)` 查询结果相同；因此可以使用覆盖索引进行查询操作，但是它不会，在这种情况下，两个查询都必须执行行读取。**

> 原文内容：It is worth to note in this case, MySQL Optimizer does not do a good job of optimizing the query. One could notice (val_with_nulls) column is not null, so COUNT(val_with_nulls) is the same as COUNT(*), and so the query could be run as an index-covered query. It does not, and both queries have to perform row reads in this case.

```SQL
(mysql) >  alter table count_myisam drop key idx, add key idx (id,val_with_nulls);
Query OK, 10000000 rows affected (1 min 38.71 sec)
Records: 10000000  Duplicates: 0  Warnings: 0

(mysql) >  select count(val_with_nulls) from count_myisam where id<1000000;
+-----------------------+
| count(val_with_nulls) |
+-----------------------+
|               1000281 |
+-----------------------+
1 row in set (0.42 sec)

(mysql) >  select count(*) from count_myisam where id<1000000;
+----------+
| count(*) |
+----------+
|  1000762 |
+----------+
1 row in set (0.56 sec)
```

正如我们看到的，与没有索引的 `COUNT(val_with_nulls)` 相比，扩展索引有助于提高 `COUNT(val_with_nulls)` 查询空值的性能，大约有 `7` 倍提升。但是，我们也发现 `COUNT(*)` 变慢了大约 `0.6` 倍，这可能是因为在这种情况下索引长度大约扩大为原来两倍。

最后，我想消除一些关于 `COUNT(0)` 和 `COUNT(1)` 的错觉。

```SQL
(mysql) >  select count(1) from count_innodb where id<1000000;
+----------+
| count(1) |
+----------+
|   980000 |
+----------+
1 row in set (0.30 sec)

(mysql) >  select count(0) from count_innodb where id<1000000;
+----------+
| count(0) |
+----------+
|   980000 |
+----------+
1 row in set (0.30 sec)

(mysql) > explain select count(1) from count_innodb where id<1000000\G
*************************** 1. row ***************************
  select_type: SIMPLE
        table: count_innodb
         type: range
possible_keys: PRIMARY
          key: PRIMARY
         rows: 1955802
     filtered: 100.00
        Extra: Using where; Using index

(mysql) >  explain select count(0) from count_innodb where id<1000000\G
*************************** 1. row ***************************
  select_type: SIMPLE
        table: count_innodb
         type: range
possible_keys: PRIMARY
          key: PRIMARY
         rows: 1955802
     filtered: 100.00
        Extra: Using where; Using index
```

正如我们所看到的，实际查询的性能和 `EXPLAIN` 查询计划的结果都是相同的；在 `COUNT()` 函数的括号内放什么数字并不重要，它可以是我们想要的任何数字，并且根据性能和查询的实际输出结果来看，它与 `COUNT(*)` 完全等价。
