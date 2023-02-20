---
title: 深入探究 MySQL 数据库 Performance Schema
date: 2023-02-17 20:03:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/mysql_study.png

# author information, multiple authors are set to array
# single author
author:
  - nick: Ankit Kapoor
    link: https://www.percona.com/blog/author/ankit-kapoor/

# post subtitle in your index page
subtitle: 本文是一篇翻译文章，在这篇博客文章中，我们将学习如何借助 `performance schema` 中 `Instrument` 工具来排查 `MySQL` 数据库的一些问题。

categories: 
  - 数据库

tags: 
  - MySQL
  - Performance Schema
---

> 原文链接：https://www.percona.com/blog/deep-dive-into-mysqls-performance-schema/

最近我正在与一位客户合作，我们工作的重点是对客户的多个 `MySQL` 数据库节点进行性能审计。我们开始研究 `performance schema` 的一些统计数据。在工作中客户提出了两个有趣的问题：他如何才能充分利用 `performance schema`，他又如何找到他需要的东西？我意识到理解 `performance schema` 的内部实现，并知道如何有效地利用它是非常重要的。这个博客的目的是让每个人都能够更容易理解 `performance schema`。

`performance schema` 是 `MySQL` 中的一个引擎，我们可以使用 `SHOW ENGINES` 很方便查看它是否已被启用。它完全建立在各种的工具集（也可以称为事件名称）之上，这些工具集（事件名称）分别服务于不同目的。

`Instrument` 是 `performance schema` 的主要组成部分，当我们想调查一个问题及其出现的根本原因时，它非常有用；下面我列出了一些示例（但不限于如下内容）：

- **1、哪个 `IO` 操作导致 `MySQL` 变慢？**
- **2、进程/线程主要在等待哪个文件？**
- **3、查询在每个执行阶段需要花费时间，或者 `alter` 命令将花费多少时间？**
- **4、哪个进程消耗了大部分内存或如何确定内存泄漏的原因？**

> 1. Which IO operation is causing MySQL to slow down?
> 2. Which file a process/thread is mostly waiting for?
> 3. At which execution stage is a query taking time, or how much time will an alter command will take?
> 4. Which process is consuming most of the memory or how to identify the cause of memory leakage?

## 就 performance schema 而言，什么是 Instrument？

`Instrument` 是将 **wait**、**IO**、**SQL**、**binlog**、**file** 等不同组件组合到一起。如果我们将这些组件组合起来，它们将成为帮助我们解决不同问题的非常有用的工具。例如，**wait/io/file/sql/binlog** 是提供二进制日志文件有关阻塞等待和 `I/O` 详细信息的工具之一。`Instrument` 从左边读取，组件之间使用“/”分隔符进行分割。我们添加到 `Instrument` 中的组件越多，它就会变得越复杂、越具体，即 `Instrument` 越长，它就越复杂。

我们可以在所使用的 `MySQL` 版本 `setup_instruments` 表中找到所有可用的 `Instrument`。值得注意的是，每个版本的 `MySQL` 都有不同数量的 `Instrument`。

```sql
select count(1) from performance_schema.setup_instruments;

+----------+
| count(1) |
+----------+

|     1269 |

+----------+
```

为了便于理解，`Instrument` 可以分为如下所示的七个不同的部分。**我这里使用的 `MySQL` 版本是 `8.0.30`**。在早期版本中，我们只能使用其中四个部分，因此如果使用不同或者较低 `MySQL` 版本，我们可能会看到不同类型的 `Instrument`。

```sql
select distinct(substring_index(name,'/',1)) from performance_schema.setup_instruments;
 
+-------------------------------+
| (substring_index(name,'/',1)) |
+-------------------------------+
| wait                          |
| idle                          |
| stage                         |
| statement                     |
| transaction                   |
| memory                        |
| error                         |
+-------------------------------+
 
7 rows in set (0.01 sec)
```

- **Stage** - 以 `stage` 开头的 `Instrument` 提供所有查询的执行阶段，如读取数据、发送数据、修改表、检查查询缓存等等。例如 `stage/sql/altering table`；
- **Wait** - 以 `wait` 开头的 `Instrument` 放在这里，像互斥锁等待、文件等待、`I/O` 等待和表等待。这个 `Instrument` 的一个示例 **wait/io/file/sql/map**；
- **Memory** - 以 `memory` 开头的 `Instrument` 提供有关每个线程内存使用情况的信息，例如 **memory/sql/MYSQL_BIN_LOG**；
- **Statement** - 以 `statement` 开头的 `Instrument` 提供有关 `SQL` 类型和存储过程的信息；
- **Idle** - 提供有关套接字连接的信息和与连接相关线程的信息；
- **Transaction** - 提供与事务相关的信息并且只有一种 `Instrument`；
- **Error** - `Error` 只有一种 `Instrument`，它提供用户操作过程中产生的错误信息，该 `Instrument` 没有附加其他组件。

下面列出了这七个组件的 `Instrument` 总数，我们可以仅以这些名称起始来识别这些 `Instrument`。

```sql
select distinct(substring_index(name,'/',1)) as instrument_name,count(1) from performance_schema.setup_instruments group by instrument_name;
 
+-----------------+----------+
| instrument_name | count(1) |
+-----------------+----------+
| wait            |      399 |
| idle            |        1 |
| stage           |      133 |
| statement       |      221 |
| transaction     |        1 |
| memory          |      513 |
| error           |        1 |
+-----------------+----------+
```

## 如何找到你需要的 `Instrument`

我清楚地记得有位客户问我，既然有成千上万种 `Instrument` 可供选择，他如何才能找到他需要的那一种。正如我之前提到的，`Instrument` 是从左到右阅读的，我们可以找出我们需要的 `Instrument`，然后找到它各自代表的性能指标。

例如，如果我们需要观察 `MySQL` 实例的 `redo` 日志（日志文件或 `WAL` 文件）的性能，需要检查线程/连接在写入数据之前，是否需要等待 `redo` 日志文件刷新到磁盘，如果需要等待，将会等待多长时间？

```shell
select * from setup_instruments where name like '%innodb_log_file%';

+-----------------------------------------+---------+-------+------------+------------+---------------+
| NAME                                    | ENABLED | TIMED | PROPERTIES | VOLATILITY | DOCUMENTATION |
+-----------------------------------------+---------+-------+------------+------------+---------------+
| wait/synch/mutex/innodb/log_files_mutex | NO      | NO    |            |          0 | NULL          |
| wait/io/file/innodb/innodb_log_file     | YES     | YES   |            |          0 | NULL          |
+-----------------------------------------+---------+-------+------------+------------+---------------+
```

在这里显示我有两个用于 `redo` 日志文件的工具。一个是关于 `redo` 日志文件的互斥锁统计信息，第二个是关于 `redo` 日志文件的 `IO` 等待统计信息。

示例二，我们需要找出可以计算花费时间的操作或工具，即批量更新需要多少时间。以下是所有可以帮助我们进行定位的 `Instrument`。

```shell
select * from setup_instruments where PROPERTIES='progress';        
 
+------------------------------------------------------+---------+-------+------------+------------+---------------+
| NAME                                                 | ENABLED | TIMED | PROPERTIES | VOLATILITY | DOCUMENTATION |
+------------------------------------------------------+---------+-------+------------+------------+---------------+
| stage/sql/copy to tmp table                          | YES     | YES   | progress   |          0 | NULL          |
| stage/sql/Applying batch of row changes (write)      | YES     | YES   | progress   |          0 | NULL          |
| stage/sql/Applying batch of row changes (update)     | YES     | YES   | progress   |          0 | NULL          |
| stage/sql/Applying batch of row changes (delete)     | YES     | YES   | progress   |          0 | NULL          |
| stage/innodb/alter table (end)                       | YES     | YES   | progress   |          0 | NULL          |
| stage/innodb/alter table (flush)                     | YES     | YES   | progress   |          0 | NULL          |
| stage/innodb/alter table (insert)                    | YES     | YES   | progress   |          0 | NULL          |
| stage/innodb/alter table (log apply index)           | YES     | YES   | progress   |          0 | NULL          |
| stage/innodb/alter table (log apply table)           | YES     | YES   | progress   |          0 | NULL          |
| stage/innodb/alter table (merge sort)                | YES     | YES   | progress   |          0 | NULL          |
| stage/innodb/alter table (read PK and internal sort) | YES     | YES   | progress   |          0 | NULL          |
| stage/innodb/alter tablespace (encryption)           | YES     | YES   | progress   |          0 | NULL          |
| stage/innodb/buffer pool load                        | YES     | YES   | progress   |          0 | NULL          |
| stage/innodb/clone (file copy)                       | YES     | YES   | progress   |          0 | NULL          |
| stage/innodb/clone (redo copy)                       | YES     | YES   | progress   |          0 | NULL          |
| stage/innodb/clone (page copy)                       | YES     | YES   | progress   |          0 | NULL          |
+------------------------------------------------------+---------+-------+------------+------------+---------------+
```

上述 `Instrument` 是可以进行跟踪定位的工具。

## 如何准备 `Instrument` 来解决性能问题

要利用这些 `Instrument`，首先需要启用它们来收集 `performance schema` 日志相关数据。除了记录运行线程的信息外，还可以维护此类线程的历史记录（`statement` / `stages` 或任何特定操作）。我们查看一下默认情况下所使用版本的数据库中启用了多少 `Instrument`。我没有明确启用任何其他工具。

```shell
select count(*) from setup_instruments where ENABLED='YES';

+----------+
| count(*) |
+----------+
|      810 |
+----------+

1 row in set (0.00 sec)
```

下面的查询列出了前 `30` 个启用的 `Instrument`，它们将在表中进行日志记录。

```shell
select * from performance_schema.setup_instruments where enabled='YES' limit 30;


+---------------------------------------+---------+-------+------------+------------+---------------+
| NAME                                  | ENABLED | TIMED | PROPERTIES | VOLATILITY | DOCUMENTATION |
+---------------------------------------+---------+-------+------------+------------+---------------+
| wait/io/file/sql/binlog               | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/binlog_cache         | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/binlog_index         | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/binlog_index_cache   | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/relaylog             | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/relaylog_cache       | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/relaylog_index       | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/relaylog_index_cache | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/io_cache             | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/casetest             | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/dbopt                | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/ERRMSG               | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/select_to_file       | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/file_parser          | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/FRM                  | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/load                 | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/LOAD_FILE            | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/log_event_data       | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/log_event_info       | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/misc                 | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/pid                  | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/query_log            | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/slow_log             | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/tclog                | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/trigger_name         | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/trigger              | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/init                 | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/SDI                  | YES     | YES   |            |          0 | NULL          |
| wait/io/file/sql/hash_join            | YES     | YES   |            |          0 | NULL          |
| wait/io/file/mysys/proc_meminfo       | YES     | YES   |            |          0 | NULL          |
+---------------------------------------+---------+-------+------------+------------+---------------+
```

正如我之前提到的，还可以维护事件的历史记录。例如，如果我们正在进行负载测试并希望分析查询完成后的性能，则需要激活以下事件（如果尚未激活）。

```shell
select * from performance_schema.setup_consumers;
 
+----------------------------------+---------+
| NAME                             | ENABLED |
+----------------------------------+---------+
| events_stages_current            | YES     |
| events_stages_history            | YES     |
| events_stages_history_long       | YES     |
| events_statements_cpu            | YES     |
| events_statements_current        | YES     |
| events_statements_history        | YES     |
| events_statements_history_long   | YES     |
| events_transactions_current      | YES     |
| events_transactions_history      | YES     |
| events_transactions_history_long | YES     |
| events_waits_current             | YES     |
| events_waits_history             | YES     |
| events_waits_history_long        | YES     |
| global_instrumentation           | YES     |
| thread_instrumentation           | YES     |
| statements_digest                | YES     |
+----------------------------------+---------+
```

注意：上面列出的前 `15` 条记录的意思是不言而喻的，但最后一条与摘要相关事件作用是允许记录 `SQL` 语句的摘要文本。我的意思是摘要内容，将相似的查询分组并显示它们的性能，这是通过哈希算法完成的。

比如说我们想分析在查询哪个阶段花费了大量的时间，我们需要使用以下语句启用相应的日志记录。

```shell
MySQL> update performance_schema.setup_consumers set ENABLED='YES' where NAME='events_stages_current';

Query OK, 1 row affected (0.00 sec)

Rows matched: 1  Changed: 1  Warnings: 0
```

## 如何充分利用 `performance schema`

现在我们知道什么是 `Instrument`、如何启用它们以及我们要存储的数据量，是时候了解如何使用这些 `Instrument` 了。为了更容易理解，我从测试用例中提取了一些 `Instrument` 的输出，因为有超过一千种 `Instrument`，我们的测试不可能覆盖所有。

请注意为了生成模拟负载，我使用了 `sysbench`（如果不熟悉它，可以阅读[此处](https://github.com/akopytov/sysbench)文档）工具，它可以使用以下内容创建读写流量：

```lua
lua : oltp_read_write.lua
 
Number of tables : 1
 
table_Size : 100000
 
threads : 4/10 
 
rate - 10

```

举个例子，思考一下如果我们想知道内存被使用的情况，为了找出这一点，我们可以在与内存相关的表中执行以下查询。

```shell
select * from memory_summary_global_by_event_name order by SUM_NUMBER_OF_BYTES_ALLOC desc limit 3\G;
 
*************************** 1. row ***************************
                  EVENT_NAME: memory/innodb/buf_buf_pool
                 COUNT_ALLOC: 24
                  COUNT_FREE: 0
   SUM_NUMBER_OF_BYTES_ALLOC: 3292102656
    SUM_NUMBER_OF_BYTES_FREE: 0
              LOW_COUNT_USED: 0
          CURRENT_COUNT_USED: 24
             HIGH_COUNT_USED: 24
    LOW_NUMBER_OF_BYTES_USED: 0
CURRENT_NUMBER_OF_BYTES_USED: 3292102656
   HIGH_NUMBER_OF_BYTES_USED: 3292102656
*************************** 2. row ***************************
                  EVENT_NAME: memory/sql/THD::main_mem_root
                 COUNT_ALLOC: 138566
                  COUNT_FREE: 138543
   SUM_NUMBER_OF_BYTES_ALLOC: 2444314336
    SUM_NUMBER_OF_BYTES_FREE: 2443662928
              LOW_COUNT_USED: 0
          CURRENT_COUNT_USED: 23
             HIGH_COUNT_USED: 98
    LOW_NUMBER_OF_BYTES_USED: 0
CURRENT_NUMBER_OF_BYTES_USED: 651408
   HIGH_NUMBER_OF_BYTES_USED: 4075056
*************************** 3. row ***************************
                  EVENT_NAME: memory/sql/Filesort_buffer::sort_keys
                 COUNT_ALLOC: 58869
                  COUNT_FREE: 58868
   SUM_NUMBER_OF_BYTES_ALLOC: 2412676319
    SUM_NUMBER_OF_BYTES_FREE: 2412673879
              LOW_COUNT_USED: 0
          CURRENT_COUNT_USED: 1
             HIGH_COUNT_USED: 13
    LOW_NUMBER_OF_BYTES_USED: 0
CURRENT_NUMBER_OF_BYTES_USED: 2440
   HIGH_NUMBER_OF_BYTES_USED: 491936
 
Above are the top three records, showing where the memory is getting mostly utilized.
```

`memory/innodb/buf_buf_pool` 这个 `Instrument` 与缓冲池相关，我们可以从 `SUM_NUMBER_OF_BYTES_ALLOC` 字段了解到缓冲池被分配了 `3GB` 空间。另一个对我们来说也很重要的数据是 `CURRENT_COUNT_USED`，它告诉我们当前使用了多少数据块，一旦工作完成，该列的值将被修改。看这条记录的统计数据，`3GB` 的消耗不是问题，即使 `MySQL` 非常频繁地使用缓冲池（例如，写入数据、加载数据、修改数据等）。但是当我们遇到内存泄漏问题或缓冲池未被使用时，问题就来了，在这种情况下，该 `Instrument` 对分析问题非常有用。

再看第二个 `Instrument`，`memory/sql/THD::main_mem_root` 占用了 `2G` 空间，这个 `Instrument` 跟 `SQL` 相关（应该从最左边开始读）。`THD::main_mem_root` 是一种线程类型。让我们试着了解这个 `Instrument`：

**THD** 代表线程

**main_mem_root** 是 `MEM_ROOT` 的一种类型。 `MEM_ROOT` 是一种为线程分配内存空间的结构体，这些线程用于解析查询、生成执行计划期间、执行嵌套查询/子查询期间或者其他执行查询分配资源操作。现在，在我们的例子中我们想要查看 **线程/主机** 内存消耗情况，以便我们可以进一步优化查询。在进一步深入学习之前，让我们先了解第三种 `Instrument`，这是一种用于搜索的重要 `Instrument`。

**memory/sql/filesort_buffer::sort_keys** —— 正如我之前提到的，`Instrument` 名称应该从左侧开始阅读。这是一个和 `SQL` 内存分配有关的 `Instrument`，该 `Instrument` 中的下一个组件是 **filesort_buffer::sort_keys**，它负责对数据进行排序（它可以是一个缓冲区，用于存储数据并进行排序，这方面的各种示例很多，比如索引创建或常见的 `order by` 子句）。

是时候深入分析哪个连接正在使用内存了。为了找出这一点，我使用了表 **memory_summary_by_host_by_event_name** 并过滤出来自我的应用程序服务器的记录。

```shell
select * from memory_summary_by_host_by_event_name where HOST='10.11.120.141' order by SUM_NUMBER_OF_BYTES_ALLOC desc limit 2\G;

*************************** 1. row ***************************
                        HOST: 10.11.120.141
                  EVENT_NAME: memory/sql/THD::main_mem_root
                 COUNT_ALLOC: 73817
                  COUNT_FREE: 73810
   SUM_NUMBER_OF_BYTES_ALLOC: 1300244144
    SUM_NUMBER_OF_BYTES_FREE: 1300114784
              LOW_COUNT_USED: 0
          CURRENT_COUNT_USED: 7
             HIGH_COUNT_USED: 39
    LOW_NUMBER_OF_BYTES_USED: 0
CURRENT_NUMBER_OF_BYTES_USED: 129360
   HIGH_NUMBER_OF_BYTES_USED: 667744
*************************** 2. row ***************************
                        HOST: 10.11.120.141
                  EVENT_NAME: memory/sql/Filesort_buffer::sort_keys
                 COUNT_ALLOC: 31318
                  COUNT_FREE: 31318
   SUM_NUMBER_OF_BYTES_ALLOC: 1283771072
    SUM_NUMBER_OF_BYTES_FREE: 1283771072
              LOW_COUNT_USED: 0
          CURRENT_COUNT_USED: 0
             HIGH_COUNT_USED: 8
    LOW_NUMBER_OF_BYTES_USED: 0
CURRENT_NUMBER_OF_BYTES_USED: 0
   HIGH_NUMBER_OF_BYTES_USED: 327936
```

主机 `11.11.120.141` 是我执行查询的应用程序主机，通过事件 **memory/sql/THD::main_mem_root** 显示，这台主机已消耗超过 `1G` 内存（内存总和）。现在我们已经知道这台主机目前内存消耗情况，我们就可以进一步挖掘以找出嵌套查询或子查询之类的查询语句，然后尝试对其进行优化。

同样，如果我们看到在执行查询时 **filesort_buffer::sort_keys** 分配的内存也超过了1G（内存总会）。这些 `Instrument` 工具向我们展示了一些带有排序操作（例如：`order by` 子句）查询语句参考指标。

### Time to join all dotted lines

让我们尝试在文件排序使用大部分内存的情况之一中找出罪魁祸首线程。第一个查询帮助我们找到主机和事件名称（工具）：

```shell
select * from memory_summary_by_host_by_event_name order by SUM_NUMBER_OF_BYTES_ALLOC desc limit 1\G;
 
*************************** 1. row ***************************
                        HOST: 10.11.54.152
                  EVENT_NAME: memory/sql/Filesort_buffer::sort_keys
                 COUNT_ALLOC: 5617297
                  COUNT_FREE: 5617297
   SUM_NUMBER_OF_BYTES_ALLOC: 193386762784
    SUM_NUMBER_OF_BYTES_FREE: 193386762784
              LOW_COUNT_USED: 0
          CURRENT_COUNT_USED: 0
             HIGH_COUNT_USED: 20
    LOW_NUMBER_OF_BYTES_USED: 0
CURRENT_NUMBER_OF_BYTES_USED: 0
   HIGH_NUMBER_OF_BYTES_USED: 819840
```

这是我的应用程序主机，让我们找出正在执行的用户及其各自的线程 `ID`。

```shell
select * from memory_summary_by_account_by_event_name where HOST='10.11.54.152' order by SUM_NUMBER_OF_BYTES_ALLOC desc limit 1\G;

*************************** 1. row ***************************
                        USER: sbuser
                        HOST: 10.11.54.152
                  EVENT_NAME: memory/sql/Filesort_buffer::sort_keys
                 COUNT_ALLOC: 5612993
                  COUNT_FREE: 5612993
   SUM_NUMBER_OF_BYTES_ALLOC: 193239513120
    SUM_NUMBER_OF_BYTES_FREE: 193239513120
              LOW_COUNT_USED: 0
          CURRENT_COUNT_USED: 0
             HIGH_COUNT_USED: 20
    LOW_NUMBER_OF_BYTES_USED: 0
CURRENT_NUMBER_OF_BYTES_USED: 0
   HIGH_NUMBER_OF_BYTES_USED: 819840

select * from memory_summary_by_thread_by_event_name where EVENT_NAME='memory/sql/Filesort_buffer::sort_keys' order by SUM_NUMBER_OF_BYTES_ALLOC desc limit 1\G;

*************************** 1. row ***************************
                   THREAD_ID: 84
                  EVENT_NAME: memory/sql/Filesort_buffer::sort_keys
                 COUNT_ALLOC: 565645
                  COUNT_FREE: 565645
   SUM_NUMBER_OF_BYTES_ALLOC: 19475083680
    SUM_NUMBER_OF_BYTES_FREE: 19475083680
              LOW_COUNT_USED: 0
          CURRENT_COUNT_USED: 0
             HIGH_COUNT_USED: 2
    LOW_NUMBER_OF_BYTES_USED: 0
CURRENT_NUMBER_OF_BYTES_USED: 0
   HIGH_NUMBER_OF_BYTES_USED: 81984
```

现在我们查到了用户及其线程 `ID` 的全部详细信息，让我们查一下这个线程正在执行哪种查询。

```shell
select * from events_statements_history where THREAD_ID=84 order by SORT_SCAN desc\G;

*************************** 1. row ***************************
              THREAD_ID: 84
               EVENT_ID: 48091828
           END_EVENT_ID: 48091833
             EVENT_NAME: statement/sql/select
                 SOURCE: init_net_server_extension.cc:95
            TIMER_START: 145083499054314000
              TIMER_END: 145083499243093000
             TIMER_WAIT: 188779000
              LOCK_TIME: 1000000
               SQL_TEXT: SELECT c FROM sbtest2 WHERE id BETWEEN 5744223 AND 5744322 ORDER BY c
                 DIGEST: 4f764af1c0d6e44e4666e887d454a241a09ac8c4df9d5c2479f08b00e4b9b80d
            DIGEST_TEXT: SELECT `c` FROM `sbtest2` WHERE `id` BETWEEN ? AND ? ORDER BY `c`
         CURRENT_SCHEMA: sysbench
            OBJECT_TYPE: NULL
          OBJECT_SCHEMA: NULL
            OBJECT_NAME: NULL
  OBJECT_INSTANCE_BEGIN: NULL
            MYSQL_ERRNO: 0
      RETURNED_SQLSTATE: NULL
           MESSAGE_TEXT: NULL
                 ERRORS: 0
               WARNINGS: 0
          ROWS_AFFECTED: 0
              ROWS_SENT: 14
          ROWS_EXAMINED: 28
CREATED_TMP_DISK_TABLES: 0
     CREATED_TMP_TABLES: 0
       SELECT_FULL_JOIN: 0
 SELECT_FULL_RANGE_JOIN: 0
           SELECT_RANGE: 1
     SELECT_RANGE_CHECK: 0
            SELECT_SCAN: 0
      SORT_MERGE_PASSES: 0
         SORT_RANGE: 0
              SORT_ROWS: 14
          SORT_SCAN: 1
          NO_INDEX_USED: 0
     NO_GOOD_INDEX_USED: 0
       NESTING_EVENT_ID: NULL
     NESTING_EVENT_TYPE: NULL
    NESTING_EVENT_LEVEL: 0
           STATEMENT_ID: 49021382
               CPU_TIME: 185100000
       EXECUTION_ENGINE: PRIMARY
```

我在这里仅根据 `rows_scan`（代表的是表扫描）粘贴了一条记录，我们也可以在案例中找到类似的其他查询，然后尝试通过创建索引或其他一些合适的解决方案来优化查询。

**案例二**

我们来吃尝试找出表锁定的情况，例如：有哪些锁？读锁还是写锁等？已经在用户表上加锁多长时间（以**皮秒**为单位显示）。

我们先给一张表加上写锁：

```shell
mysql> lock tables sbtest2 write;

Query OK, 0 rows affected (0.00 sec)
```

```shell
mysql> show processlist;

+----+--------+---------------------+--------------------+-------------+--------+-----------------------------------------------------------------+------------------+-----------+-----------+---------------+
| Id | User   | Host                | db                 | Command     | Time   | State                                                           | Info             | Time_ms   | Rows_sent | Rows_examined |
+----+--------+---------------------+--------------------+-------------+--------+-----------------------------------------------------------------+------------------+-----------+-----------+---------------+
|  8 | repl   | 10.11.139.171:53860 | NULL               | Binlog Dump | 421999 | Source has sent all binlog to replica; waiting for more updates | NULL             | 421998368 |         0 |             0 |
|  9 | repl   | 10.11.223.98:51212  | NULL               | Binlog Dump | 421998 | Source has sent all binlog to replica; waiting for more updates | NULL             | 421998262 |         0 |             0 |
| 25 | sbuser | 10.11.54.152:38060  | sysbench           | Sleep       |  65223 |                                                                 | NULL             |  65222573 |         0 |             1 |
| 26 | sbuser | 10.11.54.152:38080  | sysbench           | Sleep       |  65222 |                                                                 | NULL             |  65222177 |         0 |             1 |
| 27 | sbuser | 10.11.54.152:38090  | sysbench           | Sleep       |  65223 |                                                                 | NULL             |  65222438 |         0 |             0 |
| 28 | sbuser | 10.11.54.152:38096  | sysbench           | Sleep       |  65223 |                                                                 | NULL             |  65222489 |         0 |             1 |
| 29 | sbuser | 10.11.54.152:38068  | sysbench           | Sleep       |  65223 |                                                                 | NULL             |  65222527 |         0 |             1 |
| 45 | root   | localhost           | performance_schema | Sleep       |   7722 |                                                                 | NULL             |   7722009 |        40 |           348 |
| 46 | root   | localhost           | performance_schema | Sleep       |   6266 |                                                                 | NULL             |   6265800 |        16 |          1269 |
| 47 | root   | localhost           | performance_schema | Sleep       |   4904 |                                                                 | NULL             |   4903622 |         0 |            23 |
| 48 | root   | localhost           | performance_schema | Sleep       |   1777 |                                                                 | NULL             |   1776860 |         0 |             0 |
| 54 | root   | localhost           | sysbench           | Sleep       |    689 |                                                                 | NULL             |    688740 |         0 |             1 |
| 58 | root   | localhost           | NULL               | Sleep       |     44 |                                                                 | NULL             |     44263 |         1 |             1 |
| 59 | root   | localhost           | sysbench           | Query       |      0 | init                                                            | show processlist |         0 |         0 |             0 |
+----+--------+---------------------+--------------------+-------------+--------+-----------------------------------------------------------------+------------------+-
```

现在思考这样一种情况，我们不知道该会话存在，并且正在尝试读取该表数据并需要等待[元数据锁定](https://dev.mysql.com/doc/refman/5.6/en/metadata-locking.html)。在这种情况下，我们需要借助与锁相关的 `Instrument`（找出哪个会话正在锁定该表），例如：**wait/table/lock/sql/handler**（`table_handles` 表负责存储表锁相关的 `Instrument`）：

```shell
mysql> select * from table_handles where object_name='sbtest2' and OWNER_THREAD_ID is not null;

+-------------+---------------+-------------+-----------------------+-----------------+----------------+---------------+----------------+
| OBJECT_TYPE | OBJECT_SCHEMA | OBJECT_NAME | OBJECT_INSTANCE_BEGIN | OWNER_THREAD_ID | OWNER_EVENT_ID | INTERNAL_LOCK | EXTERNAL_LOCK  |
+-------------+---------------+-------------+-----------------------+-----------------+----------------+---------------+----------------+
| TABLE       | sysbench      | sbtest2     |       140087472317648 |             141 |             77 | NULL          | WRITE EXTERNAL |
+-------------+---------------+-------------+-----------------------+-----------------+----------------+---------------+----------------+
```

```shell
mysql> select * from metadata_locks;

+---------------+--------------------+------------------+-------------+-----------------------+----------------------+---------------+-------------+-------------------+-----------------+----------------+
| OBJECT_TYPE   | OBJECT_SCHEMA      | OBJECT_NAME      | COLUMN_NAME | OBJECT_INSTANCE_BEGIN | LOCK_TYPE            | LOCK_DURATION | LOCK_STATUS | SOURCE            | OWNER_THREAD_ID | OWNER_EVENT_ID |
+---------------+--------------------+------------------+-------------+-----------------------+----------------------+---------------+-------------+-------------------+-----------------+----------------+
| GLOBAL        | NULL               | NULL             | NULL        |       140087472151024 | INTENTION_EXCLUSIVE  | STATEMENT     | GRANTED     | sql_base.cc:5534  |             141 |             77 |
| SCHEMA        | sysbench           | NULL             | NULL        |       140087472076832 | INTENTION_EXCLUSIVE  | TRANSACTION   | GRANTED     | sql_base.cc:5521  |             141 |             77 |
| TABLE         | sysbench           | sbtest2          | NULL        |       140087471957616 | SHARED_NO_READ_WRITE | TRANSACTION   | GRANTED     | sql_parse.cc:6295 |             141 |             77 |
| BACKUP TABLES | NULL               | NULL             | NULL        |       140087472077120 | INTENTION_EXCLUSIVE  | STATEMENT     | GRANTED     | lock.cc:1259      |             141 |             77 |
| TABLESPACE    | NULL               | sysbench/sbtest2 | NULL        |       140087471954800 | INTENTION_EXCLUSIVE  | TRANSACTION   | GRANTED     | lock.cc:812       |             141 |             77 |
| TABLE         | sysbench           | sbtest2          | NULL        |       140087673437920 | SHARED_READ          | TRANSACTION   | PENDING     | sql_parse.cc:6295 |             142 |             77 |
| TABLE         | performance_schema | metadata_locks   | NULL        |       140088117153152 | SHARED_READ          | TRANSACTION   | GRANTED     | sql_parse.cc:6295 |             143 |            970 |
| TABLE         | sysbench           | sbtest1          | NULL        |       140087543861792 | SHARED_WRITE         | TRANSACTION   | GRANTED     | sql_parse.cc:6295 |             132 |            156 |
+---------------+--------------------+------------------+-------------+-----------------------+----------------------+---------------+-------------+-------------------+-----------------+----------------+
```

通过查询结果我们知道 `ID` 为 `141` 的线程在 `sbtest2` 表上持有锁**SHARED_NO_READ_WRITE**，因此我们可以采取一些处理措施，例如我们可以根据实际情况，提交会话或者终止会话。我们需要从线程表中找到相应的 `processlist_id` 来杀死它。

```shell
mysql> kill 63;

Query OK, 0 rows affected (0.00 sec)
```

```shell
mysql> select * from table_handles where object_name='sbtest2' and OWNER_THREAD_ID is not null;
 
Empty set (0.00 sec)
```

**案例三**

在某些情况下，我们需要找出 `MySQL` 服务器在哪里出现等待而花费了大部分时间，以便我们可以进一步采取措施：

```sql
mysql> select * from events_waits_history order by TIMER_WAIT desc limit 2\G;

*************************** 1. row ***************************
            THREAD_ID: 88
             EVENT_ID: 124481038
         END_EVENT_ID: 124481038
           EVENT_NAME: wait/io/file/sql/binlog
               SOURCE: mf_iocache.cc:1694
          TIMER_START: 356793339225677600
            TIMER_END: 420519408945931200
           TIMER_WAIT: 63726069720253600
                SPINS: NULL
        OBJECT_SCHEMA: NULL
          OBJECT_NAME: /var/lib/mysql/mysqld-bin.000009
           INDEX_NAME: NULL
          OBJECT_TYPE: FILE
OBJECT_INSTANCE_BEGIN: 140092364472192
     NESTING_EVENT_ID: 124481033
   NESTING_EVENT_TYPE: STATEMENT
            OPERATION: write
      NUMBER_OF_BYTES: 683
                FLAGS: NULL
*************************** 2. row ***************************
            THREAD_ID: 142
             EVENT_ID: 77
         END_EVENT_ID: 77
           EVENT_NAME: wait/lock/metadata/sql/mdl
               SOURCE: mdl.cc:3443
          TIMER_START: 424714091048155200
            TIMER_END: 426449252955162400
           TIMER_WAIT: 1735161907007200
                SPINS: NULL
        OBJECT_SCHEMA: sysbench
          OBJECT_NAME: sbtest2
           INDEX_NAME: NULL
          OBJECT_TYPE: TABLE
OBJECT_INSTANCE_BEGIN: 140087673437920
     NESTING_EVENT_ID: 76
   NESTING_EVENT_TYPE: STATEMENT
            OPERATION: metadata lock
      NUMBER_OF_BYTES: NULL
                FLAGS: NULL

2 rows in set (0.00 sec)
```

在上面的例子中，**bin log file** 操作已经等待了很长时间（`timer_wait` 单位是皮秒），在等待 `mysqld-bin.000009` 文件来完成 `IO` 操作。这可能是由于多种原因，例如存储空间已满。接下来的记录显示了我之前解释的示例二的详细信息。

### 还有什么？

为了让工作更轻松，能够更方便地监控这些 `Instrument`，[Percona Monitoring and Management(PMM)](https://docs.percona.com/percona-monitoring-and-management/index.html) 扮演着重要的角色。例如，请参见下面的快照信息。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/03-Deep-Dive-into-MySQL-Performance-Schema/01.png" />

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/03-Deep-Dive-into-MySQL-Performance-Schema/02.png" />

我们仅使用这些图表，几乎可以将所有的 `Instrument` 通过配置成进行监控，而不是使用 `SQL` 进行查询。如果要熟悉这些内容，请查看 PMM [示例](https://pmm2demo.percona.com/graph/d/mysql-performance-schema/mysql-performance-schema-details?from=now-12h&to=now&var-interval=$__auto_interval_interval&var-environment=All&var-node_name=&var-crop_host=&var-service_name=pxc57-2-mysql&var-region=&var-cluster=PXCCluster1&var-node_id=&var-agent_id=&var-service_id=%2Fservice_id%2F03d49df4-3870-460b-a5e4-94647b56a99d&var-az=&var-node_type=All&var-node_model=&var-replication_set=All&var-version=&var-service_type=All&var-database=All&var-username=All&var-schema=All&orgId=1&refresh=1m)。

显然，了解 `performance schema` 对我们有很大帮助，但同时启用这个特性会产生额外成本并影响性能。因此在许多场景下，[Percona Toolkit](https://docs.percona.com/percona-toolkit/) 能够在不影响数据库性能的情况下发挥很大有用。例如：`pt-index-usage`、`pt-online-schema-change`、`pt-query-digest`等工具。

**几点重要说明**

1. 历史表会在一段时间后加载，而不是立即加载。只有在一个线程活动完成之后。
2. 启用所有工具可能会影响您的 MySQL 的性能，因为我们正在启用对这些内存表的更多写入。此外，它还会对您的预算施加额外的资金。因此仅根据要求启用。
3. PMM 包含大部分仪器，也可以根据您的要求配置更多。
4. 您不需要记住所有表的名称。您可以只使用 PMM 或使用连接来创建查询。本文将整个概念散列成更小的块，因此没有使用任何连接，以便读者可以理解它。
5. 启用多种仪器的最佳方法是在暂存环境中优化您的发现，然后转移到生产环境中。

### 总结

在对 `MySQL` 服务器的行为进行故障排除时，性能模式非常有用。您需要找出您需要的仪器。如果您仍在为性能而苦恼，请随时联系我们，我们将非常乐意为您提供帮助。
