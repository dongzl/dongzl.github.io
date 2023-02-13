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
subtitle: 本文是一篇翻译文章，在这篇博客文章中，我们将学习如何在不使用连接器或外部库的情况下从头开始编写我们自己的原生 MySQL 客户端。

categories: 
  - 数据库

tags: 
  - MySQL
  - Performance Schema
---

> 原文链接：https://www.percona.com/blog/deep-dive-into-mysqls-performance-schema/

最近我正在与一位客户合作，我们工作的重点是对其多个 `MySQL` 数据库节点进行性能审计。我们开始研究 `performance schema` 的统计数据。在工作中，客户提出了两个有趣的问题：他如何才能充分利用 `performance schema`，他又如何找到他需要的东西？我意识到了解 `performance schema` 的意义，以及如何有效地利用它是非常重要的。这个博客应该让每个人都更容易理解 `performance schema`。

`performance schema` 是 `MySQL` 中的一个引擎，我们可以使用 `SHOW ENGINES` 很方便检查它是否已被启用。它完全建立在各种的工具集（也可以称为事件名称）之上，这些工具集（事件名称）分别服务于不同目的。

`Instrument` 是性能模式的主要部分，当我想调查一个问题及其根本原因时，它非常有用；下面我列出了一些示例（但不限于如下内容）：

- **1、哪个 `IO` 操作导致 `MySQL` 变慢？**
- **2、进程/线程主要在等待哪个文件？**
- **3、查询在哪个执行阶段需要时间，或者 `alter` 命令将花费多少时间？**
- **4、哪个进程消耗了大部分内存或如何确定内存泄漏的原因？**

> 1. Which IO operation is causing MySQL to slow down?
> 2. Which file a process/thread is mostly waiting for?
> 3. At which execution stage is a query taking time, or how much time will an alter command will take?
> 4. Which process is consuming most of the memory or how to identify the cause of memory leakage?

## 就 `performance schema` 而言，什么是 `Instrument`？

`Instrument` 是将 **wait**、**IO**、*SQL*、**binlog**、**file** 等不同组件组合到一起。如果我们将这些组件组合起来，它们将成为帮助我们解决不同问题的非常意义的工具。例如，**wait/io/file/sql/binlog** 是提供二进制日志文件有关阻塞等待和 `I/O` 详细信息的工具之一。`Instrument` 从左边读取，然后组件将添加分隔符“/”。我们添加到 `Instrument` 中的组件越多，它就会变得越复杂或越具体，即 `Instrument` 越长，它就越复杂。

我们可以在表 `setup_instruments` 下找到所使用的 `MySQL` 版本中所有可用的 `Instrument`。值得注意的是，每个版本的 `MySQL` 都有不同数量的 `Instrument`。

```shell
select count(1) from performance_schema.setup_instruments;

+----------+

| count(1) |

+----------+

|     1269 |

+----------+
```