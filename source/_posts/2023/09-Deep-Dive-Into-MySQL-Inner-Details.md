---
title: （待完成）深入探索 MySQL 内部细节
date: 2023-04-04 10:24:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/mysql_inner_details.png

# author information, multiple authors are set to array
# single author
author:
- nick: BB8 StaffEngineer
  link: https://medium.com/@bb8s
- nick: 董宗磊
  link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文深入探索 Redis 集群分片算法和架构方案，并对 Redis 中常见的热 Key 和 大 Key 问题，给出几种解决思路。

categories:
- 数据库

tags:
- MySQL
---

> 原文链接：https://medium.com/@bb8s/mysql-from-5000ft-above-to-inner-details-i-6a81186064de

## 初学者眼中的 MySQL

对于我们许多初学者来说，`MySQL` 就是这样的：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/01.webp" style="width:100%"/>

一切看起来都很简单。但是 `MySQL` 如何在后台处理 `SQL` 查询呢？换句话说，工程师和数据科学家编写的 `SQL` 查询语句通常作为纯文本字符串发送到 `MySQL`。`MySQL` 是如何解析这个字符串并知道要查找哪个表以及要获取哪些行记录呢？

## 连接池

就像我们此时正在浏览这个页面一样，`Web` 浏览器（`chrome`、`safari`）会保持与 `Medium` 网站的连接；同样我们的应用服务器需要连接到 `MySQL` 服务器，然后发送 `SQL` 查询文本内容，连接池通常用于管理网路连接。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/02.webp" style="width:100%"/>

连接池允许重用已有的网络连接，避免在创建新连接时所产生启动和清理开销成本。另外还可以将用户认证内置到这一层，拒绝未经授权的连接访问数据库。

通常每个连接都映射到一个线程。当处理一个 `SQL` 查询请求时，应用服务器的线程会从连接池中取出一个连接，并向 `MySQL` 服务器发送一个请求。`MySQL` 服务器中的另一个线程将接收到 `SQL` 字符串格式的请求，并执行后续操作步骤。

那么后续操作步骤是什么？

## SQL 解析

`MySQL` 服务器需要了解查询语句试图要做什么。它是要尝试读取一些数据、更新数据或删除数据？

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/03.webp" style="width:100%"/>

接收收到查询请求后，首先需要对 `SQL` 内容进行解析，主要工作是将其从本质上的文本格式内容转换为 `MySQL` 内部二进制结构的组合，方便优化器程序进行优化操作。

## 查询优化器

在 `MySQL` 执行查询之前，它会确定如何完成查询，即选择最好的查询方法。

> 例如，您要出去参加一次大型家庭旅行，每个人都坐在车里准备离开，但是你突然发现忘了带20瓶水，你很快想起所有的瓶装水都在储藏室里，但是需要尽快把它们放到你的车上，因为其他人都在等你；你开始思考，每次可以手拿4瓶，来回跑5次，或者你也可以随身带一个箱子，把20瓶都装在箱子里，一起带上车不用再来回。这就是优化器所做的事情，它分析满足请求的所有不同方法，并选择其中最优的方法。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/04.webp" style="width:100%"/>

让我们看一个简单的 `SQL` 查询：

```SQL
SELECT name FROM employee_table WHERE employee_id = 1;
```

假设 `employee_table` 有 `10K` 条员工记录。至少有两种方法（或者是正式术语中的两个执行计划）：

- **执行计划一**：扫描 `name` 列中的所有姓名，对于每个姓名，检查其对应的 `employee_id` 是否为 `1`，如果 `employee_id = 1` 则返回该姓名；
- **执行计划二**：使用主键索引查找 `employee_id = 1` 的记录，并返回该记录名称。

方法二几乎总是会执行地更快，优化器会选择使用方法二，下一步就是真正的执行计划。

## 执行引擎

执行引擎将调用存储引擎的 `API` 来执行查询优化器所选择的计划。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/05.webp" style="width:100%"/>

## 存储引擎

很多软件系统都可以分为计算层和存储层。计算层的效率在很大程度上往往取决于数据在存储层中的组织方式。在本节中，让我们深入了解 `MySQL` 的存储引擎，以及为加快存储引擎的读写速度而精心设计的许多优化。

`MySQL` 可以与集成许多不同类型的存储引擎。对于不同的使用场景，每一种存储引擎都有各自的优缺点。换句话说存储引擎可以看作是一个接口，有不同的底层实现。比如有 `InnoDB`、`MyISAM`、`Memory`、`CSV`、`Archive`、`Merge`、`Blackhole`。

`InnoDB` 毫无无疑是使用最广泛的存储引擎。这是 `MySQL 5.5` 版本以后的默认设置。

就像我们使用的笔记本电脑一样，`InnoDB` 将数据存储在内存和磁盘上。从上层的角度来看，当写入 `InnoDB` 时，数据总是先写入内存的缓存空间中，然后再持久化到磁盘。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/06.webp" style="width:100%"/>

`InnoDB` 将内存分为两部分：

- 缓冲池；
- 日志缓冲区。

缓冲池对于 `InnoDB` 来说非常重要。`MySQL` 在处理查询时通常速度非常快，原因是数据实际上是存储在内存中并对外提供的服务的（在大多数情况下并不是从磁盘，与许多人的想法恰恰相反），这个内存组件就是缓冲池。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/07.webp" style="width:100%"/>

## 缓冲池

通常来说宿主机 `80%` 的内存会分配给缓冲池使用，更大的内存可以缓存更多的数据到内存中，可以使读取速度更快。

缓冲池除了将数据简单地放在内存中，还针对数据在内存中的组织方式进行了精心设计，以加快数据库的读写速度，让我们来进一步深入了解这个详细设计。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/08.webp" style="width:100%"/>

## 页

类似于图书馆组织书籍的方式，使用 `ID` 标识书籍，并按字母或数字顺序放置在图书馆书架上，缓冲池也以类似的排序方式组织数据。

`InnoDB` 将缓冲池分成很多页，还包括一个 `change buffer`（后面会说明）。所有页都以双向链表的形式链接起来，也就是说我们可以很容易地从当前页到达下一页，或者从当前页到达上一页。

那么数据如何在页中存储的？

## 用户记录

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/09.webp" style="width:100%"/>

在一个数据页中包括如下内容：

- 指向上一个页的指针；
- 指向下一个页的指针；
- 用户记录；
- 其他属性信息。

指向上一页和下一页的指针简单来说就是一个指针，用户记录是存储每****数据的位置，每行都有一个指向下一行的 `next` 指针，形成一个单向链表。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/10.webp" style="width:100%"/>

那么问题来了。

我们知道关系型数据库中的一行记录通常由一个主键和许多其他字段组成，这些记录通常按主键排序，以便在查找具有特定主键的记录时可以使用二分查找等算法来快速检索数据降低延迟；但是如果每次我们向用户记录添加新数据时，`InnoDB` 都需要重新排列所有记录（行）以保持有序，这将是一个非常慢的操作。

实际上用户记录中的行是按插入顺序排列的，添加新记录只是意味着附加到用户记录的末尾，所需的主键顺序是通过 `next` 指针实现的，每行的 `next` 指针根据主键顺序指向下一个逻辑行，而不是内存中物理的下一行。

现在有一个新问题。前面我们提到不需要遍历所有记录就可以找到具有特定主键的目标数据，但是如果所有的行本质上都是一个单向链表，单向链表的特性决定了我们只能遍历整个链表才能找到特定的行，我们知道遍历列表是需要 O(n) 时间时间复杂度，这是非常慢的，那么 `InnoDB` 如何让它更快的呢？还记得我们之前提到的 `other fields` 吗？它们正是用于此目的。

## 下界与上界

这两个字段分别代表数据页中存储记录最大和最小的行，也就是说两者构成了一个 `min-max filter`。通过 O(1) 时间复杂度内算法检查这两个字段，`InnoDB` 可以决定要查找的行是否存储在指定数据页中。

假设我们的主键是数值类型，并且我们的某个数据页的 `supremum = 1`，`infimum = 99`。如果我们试图查找 `primary key = 101` 的行，那么显然在 `O(1)` 时间内，`InnoDB` 就能够决定它不在这个数据页中，并且马上会转到另一个数据页继续查找。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/11.webp" style="width:100%"/>

如果要查找的记录不在某个数据页中，`InnoDB` 会跳过该页，但是如果该行记录在这个数据页中，那么 `InnoDB` 是否仍然需要遍历整个链表呢？答案是否定的，`other fields` 信息会再次派上用场。


