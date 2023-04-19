---
title: （进行中）探索 MySQL 索引优化神器
date: 2023-04-04 10:24:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/mysql_inner_details.png

# author information, multiple authors are set to array
# single author
author:
- nick: Dwen
  link: https://medium.com/@ddwen
- nick: 董宗磊
  link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文是一篇 MySQL 索引优化教程。

categories:
- 数据库

tags:
- MySQL
---

> 原文链接（请科学上网）：https://betterprogramming.pub/understand-the-mysql-index-optimization-artifact-d4d7c6eb31f3

随着用户数量和数据量的增长，慢查询可能是一个无法回避的问题。一般来说，如果出现慢查询，都会伴随着出现接口响应慢，接口超时等问题。

如果是在高并发场景，可能会导致数据库连接被打满，直接导致服务不可用。

慢查询会引起很多问题。那么我们该如何优化慢查询呢？

主要的解决方式有如下一些：

- 监控执行的 SQL，发送邮件和手机短信告警，方便快速定位慢查询 SQL；
- 开启数据库慢查询日志功能；
- 简化业务逻辑；
- 代码重构和优化；
- 异步处理；
- SQL 优化；
- 索引优化。

这篇文章我主要会关注索引优化，因为索引优化是解决慢查询 SQL 问题最有效的一种方式。

如何查看SQL索引的执行状态？

是的，通过在 SQL 语句前面添加 `explain` 关键字，我们可以查看 SQL 的执行计划。通过执行计划，我们可以清晰地看到表和索引的执行情况，索引是否使用，索引执行的顺序，使用索引的类型等等。

优化索引的步骤如下：

- 使用 explain 查看SQL执行计划；
- 确定哪些索引使用不当；
- 优化 SQL，可能需要多次优化 SQL 才能达到索引使用的最佳效果。

## Explain 是什么

我们来看看 MySQL 的官方文档是怎么描述 explain 的：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/01.png" style="width:100%"/>

<div style="color:DarkGray;font-size:14px;text-align:center;"> [Click to read documentation](https://dev.mysql.com/doc/refman/8.0/en/explain.html) </div>

### explain 语法

```sql
{EXPLAIN | DESCRIBE | DESC}
    tbl_name [col_name | wild]

{EXPLAIN | DESCRIBE | DESC}
    [explain_type]
    {explainable_stmt | FORCONNECTION connection_id}

explain_type: {
    EXTENDED
  | PARTITIONS
  | FORMAT = format_name
}

format_name: {
    TRADITIONAL
  | JSON
}

explainable_stmt: {
    SELECTstatement
  | DELETEstatement
  | INSERTstatement
  | REPLACEstatement
  | UPDATEstatement
}
```

用一个简单的SQL看看使用explain关键字的效果：

```sql
explain select * from test1;
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/02.png" style="width:100%"/>

从上图可以看出，执行结果中会显示 `12` 列信息。

每个列具体信息如下：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/03.png" style="width:100%"/>

说白了，我们需要了解这些列的具体含义，才能正常判断索引的使用情况。事不宜迟，让我们马上开始。

### id 列

该列的值为 `select` 查询中的序号，如 `1、2、3、4` 等，决定了表的执行顺序。

一条 SQL 的执行计划一般有三种情况：

- 相同 id；
- 不同 id；
- 相同 id 和 不同 id 同时出现。

那么，在这三个 `case` 中表的执行顺序是怎样的呢？

#### 1. 相同 id

```sql
explain select * from test1 t1 inner join test1 t2 on t1.id=t2.id
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/04.png" style="width:100%"/>

我们可以看到执行结果中的两条数据 id 是相同的，都是 `1`。

在这个场景中表的执行顺序是什么样的呢？

答案：从上到下开始执行，首先执行表 t1，接着执行表 t2.

#### 2. 不同 id

```sql
explain select * from test1 t1 where t1.id = (select id from  test1 t2 where  t2.id=2);
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/05.png" style="width:100%"/>

我们可以看到执行结果中的两条数据 id 是不同的，第一条数据 `1`，第二条数据是 `2`。

在这个场景中表的执行顺序是什么样的呢？

答案：序号大的会首先被执行。在这里将会从下到上开始执行，表 t2 将首先被执行，接着表 t1 将被执行。

#### 3. 相同 id 和 不同 id 同时出现

```sql
explain
select t1.* from test1 t1
inner join (select max(id) mid from test1 group by id) t2
on t1.id=t2.mid
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/06.png" style="width:100%"/>

我们在执行结果中看到了三条数据。前两条数据 id 相同，第三条数据 id 与前一条不同。

在这个场景中表的执行顺序是什么样的呢？

答案：先执行序号大的，从下往上执行。当序号相同时，从上往下执行。因此，此列中表的顺序是 `test1`、`t1`。

**注意**：有一个特殊的表名称，内容为 `<derived2>`，表示是派生表，文章后面会详细介绍。

### select_type 列

这一列表示 `select` 的类型，具体包括以下 `11` 种类型：

- `SIMPLE`：简单查询；
- `PRIMARY`：最外层查询；
- `UNION`：UNION 之后的第二个或以后的查询；
- `DEPENDENT UNION`：UNION 之后的第二个或后面的查询，取决于外部查询；
- `UNION RESULT`：UNION 的结果；
- `SUBQUERY`：第一个子查询；
- `DEPENDENT SUBQUERY`：第一个子查询，取决于外部查询；
- `DERIVED`：派生表；
- `MATERIALIZED`：物化子查询；
- `UNCACHEABLE SUBQUERY`：结果无法缓存的子查询；
- `UNCACHEABLE UNION`：无法缓存结果的 UNION 之后的第二个查询或后面的查询。

最常用的有以下几种类型。

- `SIMPLE`：简单的 `SELECT` 查询，不包含子查询和 `UNION` 操作；
- `PRIMARY`：复杂查询中最外层的查询，代表主要查询；
- `SUBQUERY`：包含在 SELECT 或 WHERE 列表中的子查询；
- `DERIVED`：FROM 列表中包含的子查询，即派生的；
- `UNION`：`UNION` 关键字之后的查询；
- `UNION RESULT`：UNION 操作之后从表中获取结果集。

让我们看一下这些 `SELECT` 类型是如何出现的？

#### 1. SIMPLE

```sql
explain select * from test1;
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/07.png" style="width:100%"/>

它只出现在简单的 `SELECT` 查询中，不包含子查询和 UNION 操作，这种类型比较直观，就不多说了。

#### 2. PRIMARY 和 SUBQUERY

```sql
explain select * from test1 t1 where t1.id = (select id from  test1 t2 where  t2.id=2);
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/08.png" style="width:100%"/>

我们看到在这个嵌套查询的 SQL 中，最外层的 `t1` 表是 PRIMARY 类型，最里面的子查询 `t2` 表是 SUBQUERY 类型。

#### 3. DERIVED

```sql
explain
select t1.* from test1 t1
inner join (select max(id) mid from test1 group by id) t2
on t1.id=t2.mid
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/09.png" style="width:100%"/>

最后一条记录是派生表，一般是 FROM 列表中包含的子查询，这里是 SQL 语句中的分组子查询。

#### 4. UNION and UNION RESULT

```sql
explain
select * from test1
union
select* from test2
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/10.png" style="width:100%"/>


