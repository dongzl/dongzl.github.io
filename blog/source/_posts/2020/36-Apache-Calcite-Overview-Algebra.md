---
title: Apache Calcite 概览(三)--代数
date: 2020-11-17 10:32:56
cover: https://gitee.com/dongzl/article-images/raw/master/cover/calcite_study.png
# author information, multiple authors are set to array
# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文是对 Apache Calcite 官方文档的中文翻译和学习。

categories: 
  - 数据库

tags: 
  - Apache
  - Calcite
---

> 原文地址：https://calcite.apache.org/docs/algebra.html

## 代数

关系代数是 `Calcite` 的核心。每个查询都可以表示为一棵关系运算符树。我们可以从 `SQL` 转换为关系代数，也可以直接构建关系运算符树。

计划器规则使用保留语义的数学恒等式来转换表达式树。例如，如果过滤器未引用其他输入的列，则将过滤器推入内部联接的输入中是有效的。

`Calcite` 通过将计划程序规则重复应用于关系表达式来优化查询。基于成本模型指导处理过程，计划器引擎生成具有与原始语义相同但成本较低的替代表达式。

计划处理程序都是可扩展的。我们可以添加自定义的关系运算符、计划器规则、成本模型和统计信息。

## 代数构造器

最简单的构造关系表达式的方式是使用代数构造器工具类，[`RelBuilder`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/tools/RelBuilder.html)。代码示例如下：

### 表扫描

```java
final FrameworkConfig config;
final RelBuilder builder = RelBuilder.create(config);
final RelNode node = builder.scan("EMP").build();
System.out.println(RelOptUtil.toString(node));
```

(我们可以在 [RelBuilderExample.java](https://github.com/apache/calcite/blob/master/core/src/test/java/org/apache/calcite/examples/RelBuilderExample.java) 类中查看完整的代码示例，包括一些其它示例程序。)

代码会打印输出：

```java
LogicalTableScan(table=[[scott, EMP]])
```

上述代码已经创建了对 `EMP` 表的扫描。等效于如下 `SQL`：

```sql
SELECT * FROM scott.EMP;
```

### 添加项目

我们来增加一个项目，等价于如下 `SQL`：

```sql
SELECT ename, deptno FROM scott.EMP;
```

我们只需要在调用 `build` 方法之前先调用 `project` 方法：

```java
final RelNode node = builder
  .scan("EMP")
  .project(builder.field("DEPTNO"), builder.field("ENAME"))
  .build();
System.out.println(RelOptUtil.toString(node));
```

程序的输出结果是：

```java
LogicalProject(DEPTNO=[$7], ENAME=[$1])
  LogicalTableScan(table=[[scott, EMP]])
```

对 `builder.field` 的两次调用创建了简单表达式，这些表达式从输入的关系表达式中返回字段，即调用 `scan` 方法创建的 `TableScan` 输入。

`Calcite` 按顺序将返回的字段转换为字段引用，分别为 `$7` 和 `$1`。

### 添加过滤器和聚合操作

包含聚合和过滤操作的查询：

```java
final RelNode node = builder
  .scan("EMP")
  .aggregate(builder.groupKey("DEPTNO"),
      builder.count(false, "C"),
      builder.sum(false, "S", builder.field("SAL")))
  .filter(
      builder.call(SqlStdOperatorTable.GREATER_THAN,
          builder.field("C"),
          builder.literal(10)))
  .build();
System.out.println(RelOptUtil.toString(node));
```

等价的 SQL 语句为：

```sql
SELECT deptno, count(*) AS c, sum(sal) AS s
FROM emp
GROUP BY deptno
HAVING count(*) > 10
```

输出的结果为：

```java
LogicalFilter(condition=[>($1, 10)])
  LogicalAggregate(group=[{7}], C=[COUNT()], S=[SUM($5)])
    LogicalTableScan(table=[[scott, EMP]])
```

### 推入和弹出操作

构造器方法使用堆栈存储某一步生成的关系表达式，并将其作为输入传递给下一步。这允许产生关系表达式的方法来负责创建构造器。

大多数时间里，我们只会使用堆栈中的 `build()` 方法，来获取最后的关系表达式，即关系表达式树的根节点。

有些场景下，堆栈会变得比较深，容易使我们产生困惑。为了使事情简单明了，我们也可以从堆栈中删除表达式。例如，在这里我们正在建立一个复杂的联接：

```
.
               join
             /      \
        join          join
      /      \      /      \
CUSTOMERS ORDERS LINE_ITEMS PRODUCTS
```

我们分三个阶段进行构建。将中间结果存储在左右变量中，并在需要创建最终 `Join` 的时候使用 `push()` 方法将它们放回堆栈中：

```java
final RelNode left = builder
  .scan("CUSTOMERS")
  .scan("ORDERS")
  .join(JoinRelType.INNER, "ORDER_ID")
  .build();

final RelNode right = builder
  .scan("LINE_ITEMS")
  .scan("PRODUCTS")
  .join(JoinRelType.INNER, "PRODUCT_ID")
  .build();

final RelNode result = builder
  .push(left)
  .push(right)
  .join(JoinRelType.INNER, "ORDER_ID")
  .build();
```

## 转换规则

默认的 `RelBuilder` 会创建没有约定的逻辑 `RelNode`。但是我们可以通过 `adoptConvention()` 方法来切换使用不同的约定规则：

```java
final RelNode result = builder
  .push(input)
  .adoptConvention(EnumerableConvention.INSTANCE)
  .sort(toCollation)
  .build();
```

在这种场景下，我们在输入 `RelNode` 的顶部创建一个 `EnumerableSort`。

### 属性的名称和序号

我们可以通过名称或者是序号来索引某个属性。

序号是从 `0` 开始。每个运算符都保证其输出字段的出现顺序。例如，`Project` 返回由每个标量表达式生成的字段。

运算符中的属性名称需要确保是唯一的，但是有时这意味着名称并不完全符合我们的预期。例如，当我们通过 `DEPTNO` 字段对 `EMP` 和 `DEPT` 进行关联，其中一个输出字段将被称为 `DEPTNO`，另一个将被称为 `DEPTNO_1`。

一些关系表达式中提供的方法能够使我们更好地控制字段名称：

- `project` 允许我们使用 `alias(expr, fieldName)` 来包装表达式。它会自动删除包装器，但是会保留建议的名称（只要这个名称是唯一的）。
- `values(String[] fieldNames, Object... values)` 接收一个属性名称的数组参数，如果数组的所有元素都为空，则构建器将自动生成唯一名称。

如果表达式投影到某个输入字段，或者对输入字段进行强制转换，它将使用该输入字段的名称。

一旦分配了唯一的字段名称，这些名称将是不可变的。如果我们有特定的 `RelNode` 实例，我们可以保持字段名称不变。实际上，整个关系表达式都是不可变的。

如果一个关系表达式通过多个重写规则（参见 RelOptRule），那么结果表达式的字段名称可能看起来与原始表达式不太相似。在这种情况下最好通过序号来索引字段。

当我们构建关系表达式来接收多个输入，我们需要构造

当我们构建一个接受多个输入的关系表达式时，我们也需要在构建时将字段引用考虑在内的。建立连接条件时，这种情况最常发生。

假设我们构造一个 `EMP` 和 `DEPT` 的关联操作，`EMP` 有 `8` 个属性：`EMPNO`、`ENAME`、`JOB`、`MGR`、`HIREDATE`、`SAL`、`COMM`、`DEPTNO`；`DEPT` 有 `3` 个属性：`DEPTNO`、`DNAME`、`LOC`。在 `Calcite` 内部，`Calcite` 将这些字段表示为具有 `11` 个字段的组合输入行中的偏移量：左侧输入的第一个属性是 `#0`（以 0 为基准，请牢记），右侧输入的第一个属性是 `#8`。

但是通过构建器的 API，我们可以指定输入的字段。为了索引到 `SAL`，内部的属性是 `#5`，可以写做 `builder.field(2, 0, "SAL")`、`builder.field(2, "EMP", "SAL")` 或者是 `builder.field(2, 0, 5)`。含义是“两个输入，`#0` 输入中 `#5` 属性”。（为什么需要知道有两个输入？因为它们存储在堆栈中；输入 `#1` 在堆栈的顶部，输入 `#0` 在堆栈的底部。如果我们不告诉构建器这是两个输入，那么它将不知道输入 `#0` 栈有多深。）

类似的，为了索引 `DNAME` 属性，内部属性是 `#9`（8 + 1），可以写做 `builder.field(2, 1, "DNAME")`、`builder.field(2, "DEPT", "DNAME")` 或者是 `builder.field(2, 1, 1)`。

## 递归查询

警告：当前的 `API` 是实验性的，如有更改，恕不另行通知。 `SQL` 递归查询，例如生成序列 `1、2、3，... 10`：

```sql
WITH RECURSIVE aux(i) AS (
  VALUES (1)
  UNION ALL
  SELECT i+1 FROM aux WHERE i < 10
)
SELECT * FROM aux
```

可以使用对 `TransientTable` 和 `RepeatUnion` 的扫描操作来生成：

```java
final RelNode node = builder
  .values(new String[] { "i" }, 1)
  .transientScan("aux")
  .filter(
      builder.call(
          SqlStdOperatorTable.LESS_THAN,
          builder.field(0),
          builder.literal(10)))
  .project(
      builder.call(
          SqlStdOperatorTable.PLUS,
          builder.field(0),
          builder.literal(1)))
  .repeatUnion("aux", true)
  .build();
System.out.println(RelOptUtil.toString(node));
```

输出结果：

```java
LogicalRepeatUnion(all=[true])
  LogicalTableSpool(readType=[LAZY], writeType=[LAZY], tableName=[aux])
    LogicalValues(tuples=[[{ 1 }]])
  LogicalTableSpool(readType=[LAZY], writeType=[LAZY], tableName=[aux])
    LogicalProject($f0=[+($0, 1)])
      LogicalFilter(condition=[<($0, 10)])
        LogicalTableScan(table=[[aux]])
```

## API 概述

### 关系运算符

如下的一些方法能够创建关系表达式（[RelNode](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/RelNode.html)），将其压入堆栈，然后返回 `RelBuilder`。

|  方法                                             | 描述                                           |
| :----------------------------------------------- | :--------------------------------------------- |
| scan(tableName)                                  | 创建 [TableScan](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/TableScan.html) 对象。 |
| functionScan(operator, n, expr...) </br> functionScan(operator, n, exprList) | 创建 `n` 个最新关系表达式的 [TableFunctionScan](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/TableFunctionScan.html)。|
| transientScan(tableName [, rowType])             ||
| values(fieldNames, value...) </br> values(rowType, tupleList)                       ||
| filter([variablesSet, ] exprList) </br> filter([variablesSet, ] expr...)                 ||


Creates a TableScan on a TransientTable with the given type (if not specified, the most recent relational expression’s type will be used).