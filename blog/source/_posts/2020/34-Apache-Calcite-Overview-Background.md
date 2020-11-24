---
title: Apache Calcite 概览(一)--背景介绍
date: 2020-11-09 09:27:38
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

> 原文地址：https://calcite.apache.org/docs/index.html

## 背景介绍

`Apache Calcite` 是一个动态数据管理框架。

它包括了一个典型数据库管理系统的许多组成部分，但是它却省略了一些关键功能：数据存储、处理数据的算法以及用于存储元数据的存储库。

`Calcite` 故意回避存储和数据处理的业务功能。正如我们将看到的，这使它成为在应用程序与一个或多个数据存储和数据处理引擎之间进行功能优化的绝佳选择。它也是构建数据库的理想基础：只需要再添加数据就可以实现数据库的功能。

为了说明这一点，让我们创建一个 `Calcite` 的空白实例，然后向其中增加一些数据。

```java
public static class HrSchema {
  public final Employee[] emps = 0;
  public final Department[] depts = 0;
}
Class.forName("org.apache.calcite.jdbc.Driver");
Properties info = new Properties();
info.setProperty("lex", "JAVA");
Connection connection = DriverManager.getConnection("jdbc:calcite:", info);
CalciteConnection calciteConnection = connection.unwrap(CalciteConnection.class);
SchemaPlus rootSchema = calciteConnection.getRootSchema();
Schema schema = new ReflectiveSchema(new HrSchema());
rootSchema.add("hr", schema);
Statement statement = calciteConnection.createStatement();
ResultSet resultSet = statement.executeQuery("select d.deptno, min(e.empid) from hr.emps as e join hr.depts as d on e.deptno = d.deptno group by d.deptno having count(*) > 1");
print(resultSet);
resultSet.close();
statement.close();
connection.close();
```

数据库在哪里？在这里是没有数据库的。数据库连接（`connection`）一直是空白的直到通过 `new ReflectiveSchema` 方式注册一个 `Java` 对象来作为一个 `schema`，并收集到 `emps` 和 `depts` 属性注册为表（`table`）。

`Calcite` 自己并不拥有数据，它甚至都没有自己定义的数据格式。这个示例使用的是内存数据库集合，并使用诸如 `groupBy` 之类的运算符对数据进行处理，并从 `linq4j` 库中进行连接操作。但是 `Calcite` 还可以处理其他格式的数据，例如 `JDBC`。在第一个示例中，替换

```java
Schema schema = new ReflectiveSchema(new HrSchema());
```
为
```java
Class.forName("com.mysql.jdbc.Driver");
BasicDataSource dataSource = new BasicDataSource();
dataSource.setUrl("jdbc:mysql://localhost");
dataSource.setUsername("username");
dataSource.setPassword("password");
Schema schema = JdbcSchema.create(rootSchema, "hr", dataSource, null, "name");
```

`Calcite` 将在 `JDBC` 中执行相同的查询。对于应用程序，数据和 `API` 完全相同，但是背后实现逻辑却大不相同。`Calcite` 使用优化器规则将 `JOIN` 和 `GROUP BY` 操作推入源数据库去执行。

前面的内存中操作和 `JDBC` 操作是两个非常相似的示例程序。`Calcite` 可以处理任何数据源内容和数据格式。如果要添加一个数据源，我们需要实现一个适配器（`adapter`），并且告知 `Calcite` 框架它应该将数据源中的哪些集合视为“表”（`table`）。

为了使用一些更高级别的功能，我们可以自己编写优化器规则。优化器规则允许 `Calcite` 通过新的格式来访问数据，允许我们注册新的运算符（例如一种更好的连接（`join`）算法），并且允许 `Calcite` 进行优化如何将查询转换到到运算符上。`Calcite` 会将我们自定义的规则和运算符与内置的规则和运算符合并到一起，使用基于成本的优化器，并生成有效的执行计划。

### 实现适配器

在 `example/csv` 目录下的子项目提供了一个 `CSV` 适配器的实现，该适配器可以完全在应用程序中使用，并且这个示例也足够简单，如果我们要实现自己的适配器功能，这是一个很好的示例模板。

有关使用 `CSV` 适配器和编写其他适配器的信息，请参见[教程](https://calcite.apache.org/docs/tutorial.html)。

有关使用其他适配器以及使用 `Calcite` 的更多信息，请参见 [HOWTO](https://calcite.apache.org/docs/howto.html)。

## 目前状态

`Calcite` 框架目前已支持如下一些特征：

- 查询解析器、验证器和优化器；
- 支持读取 `JSON` 格式的对象模型；
- 支持许多标准函数和聚合函数；
- 针对 `Linq4j` 和以 `JDBC` 为后端的 `JDBC` 查询；
- `Linq4j` 作为前端；
- SQL 特征：`SELECT`、`FROM`（包括 `JOIN` 语法）、`WHERE`、`GROUP BY`（包括 `GROUPING SETS`）、聚合函数（包括 `COUNT(DISTINCT …)` 和 `FILTER`）、`HAVING`、`ORDER BY`（包括 `NULLS FIRST / LAST`）、结合操作（`UNION`、`INTERSECT`、`MINUS`）、子查询（包括相关子查询）、窗口聚合、`LIMIT`（`Postgres` 语法）；更多更详细内容参考链接 [SQL reference](https://calcite.apache.org/docs/reference.html)；
- 本地或远程的 `JDBC` 驱动；参加链接 [Avatica](https://calcite.apache.org/avatica/docs/index.html)；
- 支持多种类型的[适配器](https://calcite.apache.org/docs/adapter.html)。
