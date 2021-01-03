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
| transientScan(tableName [, rowType])             |使用指定的类型在 [TransientTable](https://calcite.apache.org/javadocAggregate/org/apache/calcite/schema/TransientTable.html) 对象上创建 [TableScan](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/TableScan.html) 对象，（如果没有指定类型，将会使用最近使用到的关系表达式类型）|
| values(fieldNames, value...) </br> values(rowType, tupleList) |创建 [Values](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Values.html) 对象|
| filter([variablesSet, ] exprList) </br> filter([variablesSet, ] expr...) |在给定谓词的 AND 上创建 [Filter](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Filter.html)；如果指定了变量集合，则谓词可以引用到这些变量|
| project(expr...) </br> project(exprList [, fieldNames]) | 创建一个 [Project](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Project.html) 对象。如果要覆盖默认名称，需要使用别名包装表达式，或指定 fieldNames 参数。|
| projectPlus(expr...) </br> projectPlus(exprList) | 保留原始字段并附加给定表达式的项目变体。|
| projectExcept(expr...) </br> projectExcept(exprList) | 保留原始字段并删除给定表达式的项目变体。|
| permute(mapping) | 创建一个 [Project](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Project.html) 对象，并使用 mapping 参数排列属性。|
| convert(rowType [, rename]) | 创建一个将字段转换为给定类型的 [Project](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Project.html)，还可以重命名它们。|
| aggregate(groupKey, aggCall...) </br> aggregate(groupKey, aggCallList) | 创建一个 [Aggregate](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Aggregate.html)。|
| distinct() | 创建一个可以消除重复记录的 [Aggregate](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Aggregate.html) 对象。|
| pivot(groupKey, aggCalls, axes, values) | 添加枢轴操作，该操作是通过为带有度量和值的每种组合列生成一个聚合来实现。 |
| sort(fieldOrdinal...) </br> sort(expr...) </br> sort(exprList) | 创建一个 [Sort](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Sort.html) 对象。</br> 在第一种形式中，字段序号从0开始，负序号表示下降。例如，-2表示字段1降序。</br> 在其他形式中，可以将表达式包装为nullsFirst或nullsLast。 |
| sortLimit(offset, fetch, expr...) </br> sortLimit(offset, fetch, exprList) | 创建一个具备 offset 和 limit 功能的 [Sort](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Sort.html) 对象|
| limit(offset, fetch) | 创建一个不带排序功能的 [Sort](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Sort.html) 对象，只具备 offset 和 limit 功能。 |
| exchange(distribution) | 创建一个 [Exchange](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Exchange.html)。 |
| sortExchange(distribution, collation) | 创建一个 [SortExchange](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/SortExchange.html) 对象|
| correlate(joinType, correlationId, requiredField...) </br> correlate(joinType, correlationId, requiredFieldList) | 创建两个最新关系表达式的 [Correlate](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Correlate.html)，其中变量名称和左关系必需的字段表达式。 |
| join(joinType, expr...) </br> join(joinType, exprList) </br> join(joinType, fieldName...) | 创建两个最新关系表达式的 [Join](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Join.html)。</br> 第一种形式联接一个布尔表达式（使用 AND 组合多个条件）。</br> 最后一种形式连接在命名属性上；连接每一边都必须有所有的属性。 |
| semiJoin(expr) | 使用 SEMI 连接类型创建两个最新关系表达式的 [Join](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Join.html)。|
| antiJoin(expr) | 使用 ANTI 连接类型创建两个最新关系表达式的 [Join](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Join.html)。|
| union(all [, n]) | 创建 n（默认是两个） 个最新关系表达式的 [Union](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Union.html)。|
| intersect(all [, n]) | 创建 n（默认是两个） 个最新关系表达式的 [Intersect](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Intersect.html) |
| minus(all) | 创建两个最新关系表达式的 [Minus](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Minus.html) |
| repeatUnion(tableName, all [, n]) | 创建与两个最新关系表达式的 [TransientTable](https://calcite.apache.org/javadocAggregate/org/apache/calcite/schema/TransientTable.html) 关联的 [RepeatUnion](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/RepeatUnion.html)，最大迭代次数为n（默认为-1，即无限制）。|
| snapshot(period) | 创建给定快照周期的 [Snapshot](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Snapshot.html)。|
| match(pattern, strictStart, strictEnd, patterns, measures, after, subsets, allRows, partitionKeys, orderKeys, interval) | 创建一个 [Match](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/Match.html)。 |

### 参数类型

- `expr`, `interval` [RexNode](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rex/RexNode.html)

- `expr...`, `requiredField...` [RexNode](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rex/RexNode.html) 数组

- `exprList`, `measureList`, `partitionKeys`, `orderKeys`, `requiredFieldList` 可迭代的 [RexNode](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rex/RexNode.html)

- `fieldOrdinal` 其行内字段的序数（从0开始）

- `fieldName` 属性的名称，在其行内是唯一的

- `fieldName...` 字符串类型数组

- `fieldNames` 可迭代的字符串

- `rowType` [RelDataType](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/type/RelDataType.html)

- `groupKey` [RelBuilder.GroupKey](https://calcite.apache.org/javadocAggregate/org/apache/calcite/tools/RelBuilder.GroupKey.html)

- `aggCall...` [RelBuilder.AggCall](https://calcite.apache.org/javadocAggregate/org/apache/calcite/tools/RelBuilder.AggCall.html) 的数组

- `aggCallList` 可迭代的 [RelBuilder.AggCall](https://calcite.apache.org/javadocAggregate/org/apache/calcite/tools/RelBuilder.AggCall.html)

- `value...` 对象数组

- `value` 对象

- `tupleList` 可迭代的 List 集合的 [RexLiteral](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rex/RexLiteral.html)

- `all`, `distinct`, `strictStart`, `strictEnd`, `allRows` boolean

- `alias` 字符串

- `correlationId` [CorrelationId](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/CorrelationId.html)

- `variablesSet` 可迭代的 [CorrelationId](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/CorrelationId.html)

- `varHolder` [RexCorrelVariable](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rex/RexCorrelVariable.html) 对象中的 [Holder](https://calcite.apache.org/javadocAggregate/org/apache/calcite/util/Holder.html)

- `patterns` 一个 Map 对象，key 是字符串，value 是 [RexNode](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rex/RexNode.html) 对象

- `subsets` 一个 Map 对象，key 是字符串，value 是排序好的字符串集合。

- `distribution` [RelDistribution](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/RelDistribution.html)

- `collation` [RelCollation](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/RelCollation.html)

- `operator` [SqlOperator](https://calcite.apache.org/javadocAggregate/org/apache/calcite/sql/SqlOperator.html)

- `joinType` [JoinRelType](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rel/core/JoinRelType.html)

构建器方法执行各种优化，包括：

- `project` 如果要求按顺序投影所有列，则返回其输入

- `filter` 展平条件（因此 AND 和 OR 可能有两个以上的孩子结点），简化了（将 x = 1 AND TRUE 转换为 x = 1）

- 如果我们应用 `sort` 然后又使用了 `limit`，则效果就像您调用了 `sortLimit`。

有一些注释方法可以将信息添加到堆栈的顶部关系表达式中：

|  方法                                             | 描述                                           |
| :----------------------------------------------- | :--------------------------------------------- |
| as(alias) | 将表别名分配给堆栈上的顶级关系表达式 |
| variable(varHolder) | 创建一个引用顶部关系表达式的相关变量 |

### 堆栈方法

|  方法                                             | 描述                                           |
| :----------------------------------------------- | :--------------------------------------------- |
| build() | 将最新创建的关系表达式弹出堆栈 |
| push(rel) | 将关系表达式压入堆栈。上面的诸如 `scan` 之类的关系方法会调用此方法，但是用户代码通常不会 |
| pushAll(collection) | 将关系表达式的集合推入堆栈 |
| peek() | 返回最近放入堆栈的关系表达式，但不删除它 |

### 标量表达式方法

如下一些方法将返回标量表达式 [RexNode](https://calcite.apache.org/javadocAggregate/org/apache/calcite/rex/RexNode.html)

它们中的许多方法会使用堆栈的内容。例如，`field("DEPTNO")`返回对刚刚添加到堆栈中的关系表达式的 `DEPTNO` 字段的引用。

|  方法                                             | 描述                                           |
| :----------------------------------------------- | :--------------------------------------------- |
| literal(value) | 常量 |
| field(fieldName) | 按名称引用栈顶部的关系表达式的字段 |
| field(fieldOrdinal) | 按序引用栈顶部的关系表达式的字段 |
| field(inputCount, inputOrdinal, fieldName) | 按名称引用第（`inputCount - inputOrdinal`）个关系表达式的字段 |
| field(inputCount, inputOrdinal, fieldOrdinal) | 按顺序引用第（`inputCount - inputOrdinal`）个关系表达式的字段 |
| field(inputCount, alias, fieldName) | 通过表别名和字段名称引用从堆栈顶部开始的最多的 `inputCount - 1` 个元素 |
| field(alias, fieldName) | 通过表别名和字段名称引用最顶层关系表达式的字段 |
| field(expr, fieldName) | 按名称引用记录值表达式的字段 |
| field(expr, fieldOrdinal) | 按属性顺序引用记录值表达式的字段 |
| fields(fieldOrdinalList) | 按顺序引用输入字段的表达式列表 |
| fields(mapping) | 通过给定映射引用输入字段的表达式列表 |
| fields(collation) | 表达式列表，`exprList`，这样 `sort(exprList)` 将复制排序规则 |
| call(op, expr...) </br> call(op, exprList) | 调用函数或运算符 |
| and(expr...) </br> and(exprList) | 逻辑与。展平嵌套的 `AND`，并优化涉及 `TRUE` 和 `FALSE` 的情况。 |
| or(expr...) </br> or(exprList) | 逻辑或。展平嵌套的 `OR`，并优化涉及 `TRUE` 和 `FALSE` 的情况。 |
| not(expr) | 逻辑非 |
| equals(expr, expr) | 等于 |
| isNull(expr) | 检测某个表达式是否为空 |
| isNotNull(expr) | 检测某个表达式是否非空 |
| alias(expr, fieldName) | 重命名表达式（仅作为 `project` 的参数有效）|
| cast(expr, typeName) </br> cast(expr, typeName, precision) </br> cast(expr, typeName, precision, scale) | 转换一个表达式为指定类型 |
| desc(expr) | 将排序方向更改为降序（仅作为 `sort` 或 `sortLimit` 的参数有效） |
| nullsFirst(expr) | 将排序顺序更改为 `null first`（仅作为 `sort` 或 `sortLimit` 的参数有效）|
| nullsLast(expr) | 将排序顺序更改为 `null last`（仅作为 `sort` 或 `sortLimit` 的参数有效）|
| cursor(n, input) | 引用具有 `n` 个输入的 `TableFunctionScan` 的 `input`（从 `0` 开始）关系输入（请参见 `functionScan`）|

### 模式方法

如下方法返回用于匹配的模式。

|  方法                                             | 描述                                           |
| :----------------------------------------------- | :--------------------------------------------- |
| patternConcat(pattern...) | 级联模式 |
| patternAlter(pattern...) | 备用模式 |
| patternQuantify(pattern, min, max) | 量化模式 |
| patternPermute(pattern...) | 排列模式 |
| patternExclude(pattern) | 排除模式 |

### key 分组方法

如下的方法将返回 [RelBuilder.GroupKey](https://calcite.apache.org/javadocAggregate/org/apache/calcite/tools/RelBuilder.GroupKey.html) 对象。

|  方法                                             | 描述                                           |
| :----------------------------------------------- | :--------------------------------------------- |
| groupKey(fieldName...) </br> groupKey(fieldOrdinal...) </br> groupKey(expr...) </br> groupKey(exprList) | 创建给定表达式的 group key |
| groupKey(exprList, exprListList) | 创建具有分组集的给定表达式的 `group key` |
| groupKey(bitSet [, bitSets]) | 创建给定输入列的 `group key`，如果指定了 `bitSets` 参数，则具有多个分组集 |

### 聚合方法

如下方法将会返回 [RelBuilder.AggCall](https://calcite.apache.org/javadocAggregate/org/apache/calcite/tools/RelBuilder.AggCall.html) 对象。

|  方法                                             | 描述                                           |
| :----------------------------------------------- | :--------------------------------------------- |
| aggregateCall(op, expr...) </br> aggregateCall(op, exprList)| 创建对给定聚合函数的调用 |
| count([ distinct, alias, ] expr...) </br> count([ distinct, alias, ] exprList)| 创建一个对 `COUNT` 聚合函数的调用 |
| countStar(alias) | 创建一个对 `COUNT(*)` 聚合函数的调用 |
| sum([ distinct, alias, ] expr) | 创建一个对 `SUM` 聚合函数的调用 |
| min([ alias, ] expr) | 创建一个对 `MIN` 聚合函数的调用 |
| max([ alias, ] expr) | 创建一个对 `MAX` 聚合函数的调用 |

要进一步修改 `AggCall`，需要调用其方法：

|  方法                                             | 描述                                           |
| :----------------------------------------------- | :--------------------------------------------- |
| approximate(approximate) | 允许近似值作为 `approximate` 的聚合结果 |
| as(alias) | 为该表达式指定列别名（请参见 `SQL` `AS`） |
| distinct() | 在执行聚合操作前消除重复值（参见 `SQL` `DISTINCT`） |
| distinct(distinct) | 如果 distinct 为 true，在执行聚合操作前消除重复记录 |
| filter(expr) | 在执行聚合操作前过滤行记录（参见 `SQL` `FILTER` `(WHERE ...)`） |
| sort(expr...) </br> sort(exprList) | 在执行聚合操作前排序行记录（参见 `SQL` `WITHIN` `GROUP`）|
