---
title: Apache Calcite 概览--使用教程
date: 2020-11-10 12:00:08
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

## 使用教程

这是一个分步教程，显示了如何构建和连接到 `Calcite` 框架。它使用一个简单的适配器，使 `CSV` 文件目录看起来像是包含表的结构。 `Calcite` 完成其余工作，并提供完整的 `SQL` 接口支持。

`Calcite-example-CSV` 是一个完整功能的 `Calcite` 适配器，它用来读取 [CVS（逗号分隔内容）](https://en.wikipedia.org/wiki/Comma-separated_values)格式的文本文件内容。值得注意的是，使用数百行的 `Java` 代码足以提供完整的 `SQL` 查询功能。

`CSV` 还可以用作构建适配器以转换为其它数据格式的模板。即使没有多少行代码，但是它也包含几个重要概念：

- 使用 `SchemaFactory` 和 `Schema` 接口实现用户自定义 `schema`；
- 以 `JSON` 文件域模型来声明 `schema`；
- 以 `JSON` 文件域模型来声明视图（`view`）；
- 使用 `Table` 接口实现用户自定义表（`table`）；
- 定义一个表中记录的类型；
- 使用 `ScannableTable` 接口，来实现一个简单的 `Table`，直接枚举所有行；
- 使用 `FilterableTable` 接口，来实现稍微高级一些的 `Table`，可以根据简单谓词过表达式滤掉一些行；
- 使用 `TranslatableTable` 接口，来时更高级别的 `Table`，它可以使用执行计划规则转换关系运算符。

## 下载 & 构建

我们需要准备 `Java（Java 8、9、10）`环境和 `Git` 环境：

```shell
$ git clone https://github.com/apache/calcite.git
$ cd calcite/example/csv
$ ./sqlline
```

## 首个查询示例

首先，我们使用 [`sqlline`](https://github.com/julianhyde/sqlline) 连接到 `Calcite`，`sqlline` 是 `Calcite` 项目中已经包含的 `SQL Shell` 工具。

```shell
$ ./sqlline
sqlline> !connect jdbc:calcite:model=src/test/resources/model.json admin admin
```

（如果你是运行在 `Windows` 环境，命令文件是 `sqlline.bat`。）

执行元数据查询结果：

```shell
sqlline> !tables
+------------+--------------+-------------+---------------+----------+------+
| TABLE_CAT  | TABLE_SCHEM  | TABLE_NAME  |  TABLE_TYPE   | REMARKS  | TYPE |
+------------+--------------+-------------+---------------+----------+------+
| null       | SALES        | DEPTS       | TABLE         | null     | null |
| null       | SALES        | EMPS        | TABLE         | null     | null |
| null       | SALES        | HOBBIES     | TABLE         | null     | null |
| null       | metadata     | COLUMNS     | SYSTEM_TABLE  | null     | null |
| null       | metadata     | TABLES      | SYSTEM_TABLE  | null     | null |
+------------+--------------+-------------+---------------+----------+------+
```

（`JDBC` 专家们请注意：`sqlline` 的 `!tables` 命令只是在后台执行 `DatabaseMetaData.getTables()`。它还有其他查询 `JDBC` 元数据的命令，例如：`!columns` 和 `!describe`。）

正如我们所看到的，在系统中存在 `5` 张表：`EMPS`、`DEPTS` 和 `HOBBIES` 表存在当前的 `SALES` 模式中，`COLUMNS` 和 `TABLES` 表存在系统的元数据模式中。系统表始终存在于 `Calcite` 中，但其他表由 `schema` 的特定实现提供；在这个查询结果中，`EMPS` 和 `DEPTS` 表基于 `resources/sales` 目录中的 `EMPS.csv` 和 `DEPTS.csv` 文件提供。

让我们对这些表执行一些查询操作，来显示 `Calcite` 已经提供的 `SQL` 的完整实现。首先，执行表扫描操作：

```shell
sqlline> SELECT * FROM emps;
+--------+--------+---------+---------+----------------+--------+-------+---+
| EMPNO  |  NAME  | DEPTNO  | GENDER  |      CITY      | EMPID  |  AGE  | S |
+--------+--------+---------+---------+----------------+--------+-------+---+
| 100    | Fred   | 10      |         |                | 30     | 25    | t |
| 110    | Eric   | 20      | M       | San Francisco  | 3      | 80    | n |
| 110    | John   | 40      | M       | Vancouver      | 2      | null  | f |
| 120    | Wilma  | 20      | F       |                | 1      | 5     | n |
| 130    | Alice  | 40      | F       | Vancouver      | 2      | null  | f |
+--------+--------+---------+---------+----------------+--------+-------+---+
```

再来执行 `JOIN` 和 `GROUP BY` 操作：

```shell
sqlline> SELECT d.name, COUNT(*)
. . . .> FROM emps AS e JOIN depts AS d ON e.deptno = d.deptno
. . . .> GROUP BY d.name;
+------------+---------+
|    NAME    | EXPR$1  |
+------------+---------+
| Sales      | 1       |
| Marketing  | 2       |
+------------+---------+
```

最后，`VALUES` 运算符生成一行简单数据，这是测试表达式和 `SQL` 内置函数的便捷方法：

```shell
sqlline> VALUES CHAR_LENGTH('Hello, ' || 'world!');
+---------+
| EXPR$0  |
+---------+
| 13      |
+---------+
```

`Calcite` 还具有许多其他的 `SQL` 功能。我们没有时间在这里一一介绍它们。用户可以编写更多查询来进行实验。

## Schema 发现

那么，`Calcite` 是如何发现这些表的呢？请记住，核心的 `Calcite` 功能并不知道有关 `CSV` 文件的任何内容。（作为“没有存储层的数据库”，`Calcite` 不知道任何文件格式。）我们是通过运行 `calcite-example-csv` 工程的代码来告知 `Calcite` 这些表的存在。

在这个过程中包括几部分内容。首先，我们基于域模型文件中的 `schema` 工厂类来定义一个 `schema`。然后 `schema` 工厂会负责创建一个 `schema`，同时 `schema` 会创建几张表，每一张表都知道如何通过扫描 CSV 文件来获取数据。最后，在 `Calcite` 解析完查询操作并生成使用这些表执行计划后，`Calcite` 会在执行查询时调用这些表以读取数据。现在，让我们通过一个示例来更详细地了解这些步骤。

在 `JDBC` 的连接字符串上，我们以 `JSON` 格式给出了模型的路径。这是一个模型示例：

```json
{
  version: '1.0',
  defaultSchema: 'SALES',
  schemas: [
    {
      name: 'SALES',
      type: 'custom',
      factory: 'org.apache.calcite.adapter.csv.CsvSchemaFactory',
      operand: {
        directory: 'sales'
      }
    }
  ]
}
```

这个域模型定义了一个名为 ‘SALES’ 的简单 schema。这个 schema 是由 [org.apache.calcite.adapter.csv.CsvSchemaFactory](https://github.com/apache/calcite/blob/master/example/csv/src/main/java/org/apache/calcite/adapter/csv/CsvSchemaFactory.java) 插件类提供功能，这个插件类是 calcite-example-csv 工程的一部分代码，并且这个类实现了 [SchemaFactory](https://calcite.apache.org/javadocAggregate/org/apache/calcite/schema/SchemaFactory.html) 接口，它的 create 方法实例化一个 schema，从域模型文件中传入 directory 参数：

```java
public Schema create(SchemaPlus parentSchema, String name, Map<String, Object> operand) {
  String directory = (String) operand.get("directory");
  String flavorName = (String) operand.get("flavor");
  CsvTable.Flavor flavor;
  if (flavorName == null) {
    flavor = CsvTable.Flavor.SCANNABLE;
  } else {
    flavor = CsvTable.Flavor.valueOf(flavorName.toUpperCase());
  }
  return new CsvSchema(
      new File(directory),
      flavor);
}
```

在模型数据的驱动下，`schema` 工厂类会实例化一个名为 `SALES` 的简单的 `schema`。这个 `schema` 对象是一个 [`org.apache.calcite.adapter.csv.CsvSchema`](https://github.com/apache/calcite/blob/master/example/csv/src/main/java/org/apache/calcite/adapter/csv/CsvSchema.java) 类的实例对象，这个类同时实现了 `Calcite` 框架中的 [`Schema`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/schema/Schema.html) 接口。

`schema` 的作用是包含一系列的 `table`。（它也可能包括一系列的子 `schema` 和 `table` 函数，但是这些是一些高级特性，`alcite-example-csv`并不支持这些特性）。`table` 实现了 `Calcite` 框架中的 [Table](https://calcite.apache.org/javadocAggregate/org/apache/calcite/schema/Table.html) 接口。`CsvSchema` 生成的 `table` 对象是 [CsvTable](https://github.com/apache/calcite/blob/master/example/csv/src/main/java/org/apache/calcite/adapter/csv/CsvTable.java) 及其子类的实例。

下面代码是 `CsvSchema` 类中的相关代码，它重写了 `AbstractSchema` 基类中的 `getTableMap()` 方法：

```java
protected Map<String, Table> getTableMap() {
  // Look for files in the directory ending in ".csv", ".csv.gz", ".json",
  // ".json.gz".
  File[] files = directoryFile.listFiles(
      new FilenameFilter() {
        public boolean accept(File dir, String name) {
          final String nameSansGz = trim(name, ".gz");
          return nameSansGz.endsWith(".csv")
              || nameSansGz.endsWith(".json");
        }
      });
  if (files == null) {
    System.out.println("directory " + directoryFile + " not found");
    files = new File[0];
  }
  // Build a map from table name to table; each file becomes a table.
  final ImmutableMap.Builder<String, Table> builder = ImmutableMap.builder();
  for (File file : files) {
    String tableName = trim(file.getName(), ".gz");
    final String tableNameSansJson = trimOrNull(tableName, ".json");
    if (tableNameSansJson != null) {
      JsonTable table = new JsonTable(file);
      builder.put(tableNameSansJson, table);
      continue;
    }
    tableName = trim(tableName, ".csv");
    final Table table = createTable(file);
    builder.put(tableName, table);
  }
  return builder.build();
}

/** Creates different sub-type of table based on the "flavor" attribute. */
private Table createTable(File file) {
  switch (flavor) {
  case TRANSLATABLE:
    return new CsvTranslatableTable(file, null);
  case SCANNABLE:
    return new CsvScannableTable(file, null);
  case FILTERABLE:
    return new CsvFilterableTable(file, null);
  default:
    throw new AssertionError("Unknown flavor " + flavor);
  }
}
```

这个 `schema` 扫描了整个目录下面所有以 `.csv` 结尾的文件，并为他们创建相对应的表。根据这种情况，在 `sales` 目录下包含 `EMPS.csv` 和 `DEPTS.csv` 文件，这些文件会成为表 `EMPS` 和 `DEPTS` 的内容。

