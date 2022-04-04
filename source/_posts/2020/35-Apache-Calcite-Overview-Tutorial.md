---
title: Apache Calcite 概览(二)--使用教程
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

> 原文地址：https://calcite.apache.org/docs/tutorial.html

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
- 使用 `TranslatableTable` 接口，来实现更高级别的 `Table`，它可以使用执行计划规则来转换关系运算符。

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

（`JDBC` 专家们请注意：`sqlline` 的 `!tables` 命令的实现是在后台执行 `DatabaseMetaData.getTables()`。它还有其他查询 `JDBC` 元数据的命令，例如：`!columns` 和 `!describe`。）

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

在这个过程中包括几部分内容。首先，我们基于域模型文件中的 `schema` 工厂类来定义一个 `schema`。然后 `schema` 工厂会负责创建一个 `schema`，同时会为 `schema` 创建几张表，每一张表都知道如何通过扫描 `CSV` 文件来获取数据。最后，在 `Calcite` 解析完查询语句并生成使用这些表的执行计划后，`Calcite` 会在执行查询时调用这些表以读取数据。现在，让我们通过一个示例来更详细地了解这些步骤。

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

这个域模型定义了一个名为 `SALES` 的简单 `schema`。这个 `schema` 是由 [org.apache.calcite.adapter.csv.CsvSchemaFactory](https://github.com/apache/calcite/blob/master/example/csv/src/main/java/org/apache/calcite/adapter/csv/CsvSchemaFactory.java) 插件类提供功能，这个插件类是 `calcite-example-csv` 工程的一部分代码，并且这个类实现了 [SchemaFactory](https://calcite.apache.org/javadocAggregate/org/apache/calcite/schema/SchemaFactory.html) 接口，它的 `create` 方法实例化一个 `schema`，从域模型文件中传入 `directory` 参数：

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

在模型数据的驱动下，`schema` 工厂类会实例化一个名为 `SALES` 的简单的 `schema`。这个 `schema` 对象是 [`org.apache.calcite.adapter.csv.CsvSchema`](https://github.com/apache/calcite/blob/master/example/csv/src/main/java/org/apache/calcite/adapter/csv/CsvSchema.java) 类的实例对象，这个类同时实现了 `Calcite` 框架中的 [`Schema`](https://calcite.apache.org/javadocAggregate/org/apache/calcite/schema/Schema.html) 接口。

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

这个 `schema` 扫描了整个目录下面所有以 `.csv` 结尾的文件，并为他们创建相对应的表。根据这个原理，在 `sales` 目录下包含 `EMPS.csv` 和 `DEPTS.csv` 文件，这些文件会成为表 `EMPS` 和 `DEPTS` 的数据内容。

## schemas 下的表和视图

注意：我们不需要在域模型中定义有关数据表的任何内容；`schema` 会自动生成这些表。

我们可以使用 `schema` 中指定的数据表的属性创建除自动创建数据表以外的其他的数据表。

让我们看看如何创建一种重要且有用的表类型，即视图。

当我们在执行查询操作时，视图看起来和数据表的作用是一样的，但是视图并不存储任何数据。它通过执行查询得出其结果。在计划查询时会扩展视图，因此查询计划器通常可以执行优化操作，例如从 `SELECT` 子句中删除最终结果中未使用的表达式。

下面是定义一个视图使用的模式：

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
      },
      tables: [
        {
          name: 'FEMALE_EMPS',
          type: 'view',
          sql: 'SELECT * FROM emps WHERE gender = \'F\''
        }
      ]
    }
  ]
}
```

与普通数据表或自定义数据表不同，`type: 'view'` 标识的 `FEMALE_EMPS` 就是一个视图的定义。需要注意，对于 `JSON` 内容，通常以反斜杠对视图定义中的单引号进行转义。

`JSON` 编写长字符串并不友好，因此 `Calcite` 支持另一种语法。如果我们定义的视图具有较长的 `SQL` 语句，则可以提供行列表而不是单个字符串：

```json
{
  name: 'FEMALE_EMPS',
  type: 'view',
  sql: [
    'SELECT * FROM emps',
    'WHERE gender = \'F\''
  ]
}
```

这样我们就定义了一个视图，我们可以在查询中使用这个视图，就如图在查询中使用数据表一样：

```shell
sqlline> SELECT e.name, d.name FROM female_emps AS e JOIN depts AS d on e.deptno = d.deptno;
+--------+------------+
|  NAME  |    NAME    |
+--------+------------+
| Wilma  | Marketing  |
+--------+------------+
```

## 用户自定义数据表

用户自定义数据表是由用户定义的代码驱动的表。这些表不一定只存在于用户自定义的 `schema` 中。

下面内容是在 `model-with-custom-table.json` 文件中定义的示例：

```json
{
  version: '1.0',
  defaultSchema: 'CUSTOM_TABLE',
  schemas: [
    {
      name: 'CUSTOM_TABLE',
      tables: [
        {
          name: 'EMPS',
          type: 'custom',
          factory: 'org.apache.calcite.adapter.csv.CsvTableFactory',
          operand: {
            file: 'sales/EMPS.csv.gz',
            flavor: "scannable"
          }
        }
      ]
    }
  ]
}
```

我们可以按通用的方式进行数据表的查询操作：

```shell
sqlline> !connect jdbc:calcite:model=src/test/resources/model-with-custom-table.json admin admin
sqlline> SELECT empno, name FROM custom_table.emps;
+--------+--------+
| EMPNO  |  NAME  |
+--------+--------+
| 100    | Fred   |
| 110    | Eric   |
| 110    | John   |
| 120    | Wilma  |
| 130    | Alice  |
+--------+--------+
```

这是一个普通的 `schema`，在这个 `schema` 中包括一个由 `org.apache.calcite.adapter.csv.CsvTableFactory` 类提供的用户自定义的数据表，该类实现了 `Calcite` 框架中 `TableFactory` 接口。该类的 `create` 方法实例化 `CsvScannableTable` 对象，并从模型文件中传入 `file` 参数：

```java
public CsvTable create(SchemaPlus schema, String name, Map<String, Object> map, RelDataType rowType) {
  String fileName = (String) map.get("file");
  final File file = new File(fileName);
  final RelProtoDataType protoRowType = rowType != null ? RelDataTypeImpl.proto(rowType) : null;
  return new CsvScannableTable(file, protoRowType);
}
```

实现自定义数据表通常是实现自定义 `schema` 的一种更简单的选择。两种方法最终都可以创建类似的 `Table` 接口实现，但是对于自定义数据表，我们不需要实现元数据发现功能。 （`CsvTableFactory` 会像 `CsvSchema` 一样创建一个 `CsvScannableTable` 对象，但是定义的数据表不会在文件系统中扫描以 `.csv` 结尾的文件。）

自定义数据表需要创建者做更多的工作（创建者需要显式指定每个表及其文件），但也可以给创建者更多的控制权（例如，为每个表提供不同的参数）。

## 域模型中使用注释

域模型中可以包含注释内容，语法格式为 `/* ... */` 或者是 `//`：

```json
{
  version: '1.0',
  /* Multi-line
     comment. */
  defaultSchema: 'CUSTOM_TABLE',
  // Single-line comment.
  schemas: [
    ..
  ]
}
```

（注释内容并不是标准 `JSON` 的一部分，但却是一个有益的扩展。）

## 使用计划器规则优化查询

到目前为止我们所看到的 `table` 的实现类都不包含大量的数据。但是如果是我们自定义的 `table`，可能包含数百列，并且包数百万的数据，我们当然希望系统不会为每个查询检索所有数据。我们希望 `Calcite` 框架与适配器协商使用并找到一种更有效的访问数据的方法。

这种协商是查询优化的一种简单形式。 `Calcite` 框架可以通过添加计划器规则来支持查询优化。计划器规则的运行方式是在查询分析树中根据某个模式进行内容匹配（例如，某种表顶部的项目），然后用实现优化的一组新节点替换树中匹配的节点。

和模式（`schema`）、数据表（`table`）类似，计划器规则也是可扩展的。所以，如果我们希望通过 `SQL` 语句来访问已经存储的数据，我们首先需要自定义模式（`schema`）或者数据表（`table`），然后定义我们自己的计划器规则来提高数据的访问效率。

为了了解这一点，让我们使用计划器规则来访问 `CSV` 文件中的一部分列。让我们针对两个非常相似的 `schema` 运行相同的查询：

```shell
sqlline> !connect jdbc:calcite:model=src/test/resources/model.json admin admin
sqlline> explain plan for select name from emps;
+-----------------------------------------------------+
| PLAN                                                |
+-----------------------------------------------------+
| EnumerableCalcRel(expr#0..9=[{inputs}], NAME=[$t1]) |
|   EnumerableTableScan(table=[[SALES, EMPS]])        |
+-----------------------------------------------------+
sqlline> !connect jdbc:calcite:model=src/test/resources/smart.json admin admin
sqlline> explain plan for select name from emps;
+-----------------------------------------------------+
| PLAN                                                |
+-----------------------------------------------------+
| EnumerableCalcRel(expr#0..9=[{inputs}], NAME=[$t1]) |
|   CsvTableScan(table=[[SALES, EMPS]])               |
+-----------------------------------------------------+
```

是什么导致了执行计划的不同？我们来查找一下原因。在 `smart.json` 域模型文件中，有一行额外的配置内容：

```shell
flavor: "translatable"
```

这个配置会导致在创建 `CsvSchema` 对象时使用 `flavor = TRANSLATABLE` 参数，添加这个参数后在调用 `createTable` 方法创建的是 [`CsvTranslatableTable`](https://github.com/apache/calcite/blob/master/example/csv/src/main/java/org/apache/calcite/adapter/csv/CsvTranslatableTable.java) 类实例对象而不是 `CsvScannableTable` 类实例对象。

<font color="red">`CsvTranslatableTable` 类实现 `TranslatableTable.toRel()` 方法来创建 `CsvTableScan` 对象。表扫描是查询运算符生成树的叶子节点。通常的实现是EnumerableTableScan 实现类，但我们创建了一个独特的子类型，该子类型将导致规则触发。</font>

下面的示例是完整的规则：

```java
public class CsvProjectTableScanRule extends RelRule<CsvProjectTableScanRule.Config> {
  /** Creates a CsvProjectTableScanRule. */
  protected CsvProjectTableScanRule(Config config) {
    super(config);
  }

  @Override public void onMatch(RelOptRuleCall call) {
    final LogicalProject project = call.rel(0);
    final CsvTableScan scan = call.rel(1);
    int[] fields = getProjectFields(project.getProjects());
    if (fields == null) {
      // Project contains expressions more complex than just field references.
      return;
    }
    call.transformTo(
        new CsvTableScan(
            scan.getCluster(),
            scan.getTable(),
            scan.csvTable,
            fields));
  }

  private int[] getProjectFields(List<RexNode> exps) {
    final int[] fields = new int[exps.size()];
    for (int i = 0; i < exps.size(); i++) {
      final RexNode exp = exps.get(i);
      if (exp instanceof RexInputRef) {
        fields[i] = ((RexInputRef) exp).getIndex();
      } else {
        return null; // not a simple projection
      }
    }
    return fields;
  }

  /** Rule configuration. */
  public interface Config extends RelRule.Config {
    Config DEFAULT = EMPTY
        .withOperandSupplier(b0 ->
            b0.operand(LogicalProject.class).oneInput(b1 ->
                b1.operand(CsvTableScan.class).noInputs()))
        .as(Config.class);

    @Override default CsvProjectTableScanRule toRule() {
      return new CsvProjectTableScanRule(this);
    }
}
```

规则的默认实例会在 `CsvRules` 类中持有：

```java
public abstract class CsvRules {
  public static final CsvProjectTableScanRule PROJECT_SCAN =
      CsvProjectTableScanRule.Config.DEFAULT.toRule();
}
```

在默认配置中调用的 `withOperandSupplier` 方法（在 `Config` 接口中的 `DEFAULT` 属性中调用）声明了关系表达式的一种模式，这种模式会触发规则的运行，如果计划器发现一个 `LogicalProject` 的唯一输入是没有任何输入的 `CsvTableScan` 对象，则它将触发该规则。

该规则可能会出现变体。例如，另一个规则实例可能会与 `CsvTableScan` 上的 `EnumerableProject` 匹配。

`onMatch` 方法会自动生成一个新的关系表达式并且调用 `RelOptRuleCall.transformTo()` 方法确保规则被成功触发。

## 查询优化过程

关于 `Calcite` 的查询计划器有多么智能，我们可以列举很多内容，但是我们这里不再赘述。智能的目的是减轻用户制定规则的负担。

首先，`Calcite` 并不会以一定的顺序触发规则。查询优化过程遵循一棵分支树的许多分支，就像下棋程序检查许多可能的移动顺序一样。如果规则 `A` 和规则 `B` 都与查询运算符树的给定部分匹配，`Calcite` 会触发这两个规则。

其次，`Calcite` 会根据使用成本在执行计划之间进行选择，但是成本模型并不能阻止规则的触发，而这在短时间内可能会消耗更大成本。

许多优化器有线性优化方案。面对如上所述的规则 `A` 和规则 `B` 之间的选择，优化器必须立刻给出选择。它可能有一种策略，例如“将规则 `A` 应用于整个树，然后将规则 `B` 应用于整个树”，或者应用基于成本的策略，并使用产生成本较小的规则。

`Calcite` 并不需要这种妥协。这使得组合各种规则变得很简单。例如，如果我们想结合使用识别物化视图的规则和要从 `CSV` 和 `JDBC` 源系统读取数据的规则，则只需给 `Calcite` 提供所有规则的集合并告诉它就可以了。

`Calcite` 确实使用基于成本模型。成本模型可以决定哪一个计划会被最终使用，并且有时还可以通过修剪搜索树来防止搜索空间膨胀，但是它绝不会强迫你在规则 `A` 和规则 `B` 之间进行选择。这一点非常重要，这很重要，因为它可以避免搜索到的局部成本最小值，其实在实际搜索空间中并不是最优搜索结果的问题。

同时成本模型是可插拔的，这一点我们也可以猜测到，基于成本模型的表和查询运算符统计信息也是如此。这可能是我们以后要讲到的内容。

## JDBC 适配器

`JDBC` 适配器可以将一个 `JDBC` 数据源中的 `schema` 映射为 `Calcite` 的 `schema`。

例如，下面的 `schema` 是从 `MySQL` 的 `foodmart` 数据库中适配到的内容：

```json
{
  version: '1.0',
  defaultSchema: 'FOODMART',
  schemas: [
    {
      name: 'FOODMART',
      type: 'custom',
      factory: 'org.apache.calcite.adapter.jdbc.JdbcSchema$Factory',
      operand: {
        jdbcDriver: 'com.mysql.jdbc.Driver',
        jdbcUrl: 'jdbc:mysql://localhost/foodmart',
        jdbcUser: 'foodmart',
        jdbcPassword: 'foodmart'
      }
    }
  ]
}
```

（那些使用 `Mondrian` `OLAP` 引擎的用户都会熟悉 `FoodMart` 数据库，因为它是 `Mondrian` 的主要测试数据。要加载测试数据内容，请遵循 [Mondrian的安装说明](https://mondrian.pentaho.com/documentation/installation.php#2_Set_up_test_data)。)

目前使用限制：`JDBC` 适配器目前仅能够下推数据表扫描操作；对于其它一些处理过程（过滤、连接、聚合等操作）都需要在 `Calcite` 内部完成。<font color="red">我们的目标是尽可能减少对源系统的处理，尽可能地翻译语法，数据类型和内置函数。</font>如果某个 `Calcite` 查询基于单个 `JDBC` 数据库中的表，那么原则上整个查询应转到该数据库上完成。如果查询的表来源于多个 `JDBC` 数据源，或者是 `JDBC` 数据源和非 `JDBC` 数据源的混合，则 `Calcite` 将使用它可以使用的最高效的分布式查询方法。

## 克隆 JDBC 适配器

克隆 `JDBC` 适配器会创建一个混合数据库。数据来源于一个 `JDBC` 数据库，但是在数据表第一次被访问时数据会被加载到内存表中。`Calcite` 根据这些内存表评估查询，实际上等同于数据库的缓存。

例如，下面的示例模型是从 `MySQL` 的 `foodmart` 数据库中读取数据表内容：

```json
{
  version: '1.0',
  defaultSchema: 'FOODMART_CLONE',
  schemas: [
    {
      name: 'FOODMART_CLONE',
      type: 'custom',
      factory: 'org.apache.calcite.adapter.clone.CloneSchema$Factory',
      operand: {
        jdbcDriver: 'com.mysql.jdbc.Driver',
        jdbcUrl: 'jdbc:mysql://localhost/foodmart',
        jdbcUser: 'foodmart',
        jdbcPassword: 'foodmart'
      }
    }
  ]
}
```

另一种技术是在现有 `schema` 基础上克隆新的 `schema`。我们可以在域模型中使用 `source` 属性来引用已经存在的 `schema`，示例如下：

```json
{
  version: '1.0',
  defaultSchema: 'FOODMART_CLONE',
  schemas: [
    {
      name: 'FOODMART',
      type: 'custom',
      factory: 'org.apache.calcite.adapter.jdbc.JdbcSchema$Factory',
      operand: {
        jdbcDriver: 'com.mysql.jdbc.Driver',
        jdbcUrl: 'jdbc:mysql://localhost/foodmart',
        jdbcUser: 'foodmart',
        jdbcPassword: 'foodmart'
      }
    },
    {
      name: 'FOODMART_CLONE',
      type: 'custom',
      factory: 'org.apache.calcite.adapter.clone.CloneSchema$Factory',
      operand: {
        source: 'FOODMART'
      }
    }
  ]
}
```

我们可以使用这种方法在任何一个 `schema` 的基础上来创建一个克隆的 `schema`，而不仅仅局限于 `JDBC`。

克隆适配器并非是最完美的解决方案。我们计划开发更复杂的缓存策略，以及内存表的更完整和有效的实现，但是目前，克隆 `JDBC` 适配器显示了可行的方法，并允许我们尝试我们的初始实现。

## 进一步讨论

当然，还有许多其他方法可以扩展 `Calcite`，在本教程中还没有一一介绍。[适配器规范](https://calcite.apache.org/docs/adapter.html)章节描述了所涉及到的 `API` 的内容。
