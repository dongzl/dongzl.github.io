---
title: MySQL 数据库索引的实现原理
date: 2020-03-14 13:52:04
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/mysql_study.png
# author information, multiple authors are set to array
# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨在通过一道面试题总结 MySQL 数据库索引的实现原理。

categories: 
  - 数据库
tags: 
  - MySQL
  - 索引
---

## 面试题

> 1、MySQL 数据库索引分类。
> 2、MySQL 数据库 InnoDB 存储引擎的底层数据结构。
> 3、为什么底层使用 B+ 树而不用 B 树

## MySQL 架构

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/10-The-Implementation-Principles-Of-MySQL-Index/The-Implementation-Principles-Of-MySQL-Index-03.png">

## 索引的分类

MySQL 索引分五种类型：主键索引、唯一索引、普通索引和全文索引、组合索引。通过给字段添加索引可以提高数据的读取速度，提高项目的并发能力和抗压能力。

- 主键索引：主键是一种唯一性索引，但它必须指定为 PRIMARY KEY，每个表只能有一个主键；

- 唯一索引：索引列的所有值都只能出现一次，即必须唯一，但是值可以为空；

- 普通索引：基本的索引类型，值可以为空，没有唯一性的限制；

- 全文索引：全文索引的索引类型为 FULLTEXT，全文索引可以在 varchar、char、text 类型的列上创建；（使用较少，一般都使用专门的搜索框架，例如 ElasticSearch）

- 组合索引：多列值组成一个索引，专门用于组合搜索。

## MySQL 索引实现原理分析

**为什么没有使用 Hash 表的索引格式**

使用类似于 JDK 中 HashMap 的数据结构来存储索引数据，有如下特点：

- 优点：查询速度快，Hash 数据结构查询时间复杂度为 O(1);

- 缺点：
  - 1、需要将数据文件 load 到内存，比较耗费内存空间；
  - 2、Hash 快速查询只适合等值查询，对于范围查询效率低下，在实际工作中范围查询的场景比较多。

**为什么没有使用二叉树和红黑树索引格式**

无论是二叉树还是红黑树，都会因为树的深度过深而造成 IO 次数变多，影响数据读取效率。

**为什么没有使用 B 树索引格式**

B 树特点：

1、所有键值分布在整棵树中；
2、搜索有可能在非叶子节点结束，在关键字全集内做一次查找，性能逼近二分查找；
3、每个节点最多拥有 m 个子树；
4、根节点至少有 2 个子树；
5、分支节点至少拥有 m / 2 棵子树（除根节点和叶子节点外都是分支节点）；
6、所有叶子节点都在同一层、每个节点最多可以有 m - 1 个 key，并且以升序排列。

B 树缺点：

- 每个节点都有 key，同时也包含 data，而每个页存储空间是有限的，如果 data 比较大的话会导致每个节点存储的 key 数量变小；

- 当存储的数据量很大的时候会导致深度较大，增大查询时磁盘 IO 次数，进而影响查询性能。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/10-The-Implementation-Principles-Of-MySQL-Index/The-Implementation-Principles-Of-MySQL-Index-01.png">

**MySQL B+ 树索引格式实现原理**

B+ 树是在 B 树的基础上做的一种优化，优化如下：

- B+ 树每个节点可以包含更多的节点（非叶子节点不在存储数据），这样做的原因有两个，第一是为了降低树的高度，第二个原因是将数据范围变为多个区间，区间越多，数据检索越快；

- 非叶子节点存储 key ，叶子节点存储 key 和数据；

- 叶子节点两两指针互相连接（符合磁盘的预读特性），顺序查询性能更高。

**MySQL InnoDB--B+ 树，叶子节点直接存储数据**

- InnoDB 是通过 B+ 树结构对主键创建索引，在叶子节点中存储记录数据，如果没有主键，就选择唯一键，如果没有唯一键，那么会生成一个 6 位的 row_id 来作为主键；

- 如果创建索引的键是其他字段，那么在叶子节点中存储的是该记录的主键，然后在通过主键索引找到对应的记录，这个过程叫做回表。

**MySQL MyISAM--B+ 树，叶子节点存储表中数据的地址**

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/10-The-Implementation-Principles-Of-MySQL-Index/The-Implementation-Principles-Of-MySQL-Index-02.png">

## MySQL 索引的一些其他内容

### 回表

如果 MySQL 中的索引不是主键索引，在使用这个索引进行数据查询时，需要现在这个索引的 B+ 树中找到叶子节点存储的主键 ID，然后根据主键 ID 到主键索引的 B+ 树中查找到最终记录，这个过程就叫做回表。

### 最左匹配原则

在创建联合索引时，如果在查询条件中包含索引的最左列，那么这个索引可以匹配到，如果不包含最左列，这个索引无法匹配，这个称做最左原则。

比如某个表有联合索引，索引字段为 (column1, column2)

```SQL
a、select * from table where column1 = 'c1' and column2 = 'c2';

b、select * from table where column2 = 'c2' and column1 = 'c1';

c、select * from table where column1 = 'c1';

d、select * from table where column2 = 'c2';
```

上述 a、b、c 三个查询可以使用该索引，查询 d 无法使用该索引。

知识补充，如果上述四条SQL语句都希望走索引，需要创建两个索引：

组合索引: (column1, column2), 和 column2 单列索引；

或者是组合索引: (column2, column1), 和 column1 单列索引；

具体选择哪种方案，需要对比 column1、column2 字段类型，在满足查询优化情况尽量减少磁盘空间占用。

PS.
```SQL
## 范围查询只能用到 column2 列上索引，column2 上用不到
select * from table where column1 > 'c1' and column2 = 'c2';
```

### 索引覆盖

如果索引中包含查询结果需要的全部字段，那么将不需要在回表查询该记录的所有数据，这个过程就是索引覆盖，覆盖索引在查询计划中表现为：`using index`。

### 索引下推

例如有 user_table 表，表上有 (user_name, age) 联合索引
如果现在有一个需求，查出名称中以“张”开头且年龄小于等于 10 的用户信息：

```SQL
select * from user_table where user_name like '张%' and age > 10;
```

这条查询语句有两种执行可能：

- 根据 (user_name, age) 联合索引查询所有满足名称以“张”开头的索引，然后回表查询出相应的全行数据，然后再筛选出满足年龄小于等于10的用户数据。

- 根据 (user_name, age) 联合索引查询所有满足名称以“张”开头的索引，然后直接再筛选出年龄小于等于10的索引，之后再回表查询全行数据。

很明显，后一种方式需要回表查询的全行数据比较少，这就是 MySQL 的索引下推。

### 聚簇索引与非聚簇索引

聚簇索引并不是一种索引类型，而是一种数据存储方式，指的是数据行跟相邻的键值紧凑的存储在一起。费聚簇索引是指数据文件和索引文件分开存放。

MySQL 数据库中 InnoDB 存储引擎，B+ 树索引可以分为聚簇索引（也称聚集索引，clustered index）和辅助索引（有时也称非聚簇索引或二级索引，secondary index，non-clustered index）。这两种索引内部都是B+树，聚集索引的叶子节点存放着一整行的数据。

InnoDB 中的主键索引是一种聚簇索引，非聚簇索引都是辅助索引，像复合索引、前缀索引、唯一索引。

InnoDB 使用的是聚簇索引，MyISAM 使用的是非聚簇索引。

**聚簇索引的优缺点**：

优点：

- 数据访问更快，因为聚簇索引将索引和数据保存在同一个B+树中，因此从聚簇索引中获取数据比非聚簇索引更快；

- 聚簇索引对于主键的排序查找和范围查找速度非常快。

缺点：

- 插入速度严重依赖于插入顺序，按照主键的顺序插入是最快的方式，否则将会出现页分裂，严重影响性能。因此，对于InnoDB表，我们一般都会定义一个自增的ID列为主键

- 更新主键的代价很高，因为将会导致被更新的行移动。因此，对于 InnoDB 表，我们一般定义主键为不可更新。

- 二级索引访问需要两次索引查找，第一次找到主键值，第二次根据主键值找到行数据。

## 参考资料

- [MySQL 索引篇之覆盖索引、联合索引、索引下推](https://www.jianshu.com/p/bdc9e57ccf8b)
- [索引下推（5.6版本+）](https://blog.csdn.net/mccand1234/article/details/95799942)
- [MySQL优化面试](https://zhenganwen.top/posts/433a3305/)


