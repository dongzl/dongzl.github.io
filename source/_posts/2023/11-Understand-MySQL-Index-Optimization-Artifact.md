---
title: （进行中）探索 MySQL 索引优化神器
date: 2023-04-22 10:24:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/mysql_index_optimize.png

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

表 test2 是 UNION 关键字之后的查询，所以它被标识为 UNION，表 test1 是主表，被标识为 PRIMARY。而 `<union1,2>` 表示 `id=1` 和 `id=2` 的表并集，结果被标记为 `UNION RESULT`。

所以 UNION 和 UNION RESULT 通常是成对出现的。

### table 列

该列的值表示输出行所引用的表名，如前面的：`test1`、`test2` 等。

但它也可以是以下值之一：

- `<unionM,N>`：`M` 和 `N` 并集操作的行记录和记录 `id`；
- `<derivedN>`：用于与此行关联的派生表结果 id 的值 N。派生表可能来自（例如）FROM 子句中的子查询；
- `<subqueryN>`：子查询的结果，其 id 值为 N。

### partitions 列

此列的值表示匹配查询记录结果的分区。

### type 列

该列的值表示连接类型，是索引执行情况的重要指标。

这包含以下类型：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/11.png" style="width:100%"/>

执行结果从最好到最差的顺序是从上到下。

我们需要关注以下类型：

```shell
system > const > eq_ref > ref > range > index > all
```

```shell
# test2 table structure
id    code    name
1     001     city1
```

在 `code` 字段上建立一个普通索引。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/12.png" style="width:100%"/>

下面我们一一看看几种常见的连接类型是如何出现的。

#### 1. System

这种类型只需要数据库表中的一条数据，是 `const` 类型的特例，一般不会出现。

#### 2. Const

通过一个索引可以找到数据，一般用在以主键或唯一索引为条件的查询 SQL 语句中。

```sql
explain select * from test2 where id=1;
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/13.png" style="width:100%"/>

#### 3. Eq_ref

通常用于主键或唯一索引扫描。

```sql
explain select * from test2 t1 inner join test2 t2 on t1.id=t2.id;
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/14.png" style="width:100%"/>

const 和 eq_ref 都是对主键或唯一索引的扫描，那这两种类型有什么区别？

答案：const 只会被索引一次，eq_ref 的主键与数据记录的主键匹配。由于表中有多条数据，一般情况下，需要对数据进行多次索引才能全部匹配。

#### 4. Ref

常用于非主键所以和唯一索引扫描。

```sql
explain select * from test2 where code = '001';
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/15.png" style="width:100%"/>

#### 5. Range

通常用于范围查询，例如：`between...and` 或者是 `in` 操作。

```sql
explain select * from test2 where id between 1 and 2;
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/16.png" style="width:100%"/>

#### 6. Index

全索引扫描。

```sql
explain select code from test2;
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/17.png" style="width:100%"/>

##### 7. All

全表扫描。

```sql
explain select *  from test2;
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/11-Understand-MySQL-Index-Optimization-Artifact/18.png" style="width:100%"/>


