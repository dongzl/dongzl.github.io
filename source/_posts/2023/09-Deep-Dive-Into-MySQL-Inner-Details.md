---
title: 深入探索 MySQL 内部实现原理
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
subtitle: 本文将深入探索 MySQL 数据库内部实现原理，并描述各个模块在实际工作中发挥的作用。

categories:
- 数据库

tags:
- MySQL
---

> 原文链接（请科学上网）：https://medium.com/@bb8s/mysql-from-5000ft-above-to-inner-details-i-6a81186064de

## 初学者眼中的 MySQL

对于我们许多初学者来说，`MySQL` 就是这样的：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/01.webp" style="width:100%"/>

一切看起来都很简单，但是 `MySQL` 如何在后台处理 `SQL` 请求呢？换句话说，工程师和数据科学家编写的 `SQL` 查询语句通常都是纯文本字符串内容，并发送到 `MySQL` 的。那么 `MySQL` 是如何解析这个字符串并知道要查找哪个数据表以及要获取哪些记录呢？

## 连接池

就像我们此时正在浏览这个页面一样，`Web` 浏览器（`chrome`、`safari`）会保持与 `Medium` 网站的连接；同样我们的应用服务器需要通过网络连接到 `MySQL` 服务器，然后发送 `SQL` 查询文本内容，连接池通常用于管理网络连接。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/02.webp" style="width:100%"/>

连接池允许重用已有的网络连接，避免在创建新连接时所产生初始化和断开连接时释放资源所产生的成本开销。另外还可以将用户认证内置到这一层，拒绝未经授权的连接访问数据库。

通常每个连接都映射到一个线程，当处理一个 `SQL` 查询请求时，应用服务器的线程会从连接池中取出一个连接，并向 `MySQL` 服务器发送一个请求，`MySQL` 服务器中的另一个线程将接收到 `SQL` 字符串格式的请求，并执行后续操作步骤。

那么后续操作步骤是什么？

## SQL 解析

`MySQL` 服务器需要了解查询语句试图要做什么，它是要尝试读取一些数据、更新数据或删除数据？

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/03.webp" style="width:100%"/>

接收到查询请求后，首先需要对 `SQL` 内容进行解析，主要工作是将其本质上文本格式的内容转换为 `MySQL` 内部二进制结构的组合，方便优化器程序进行优化操作。

## 查询优化器

在 `MySQL` 执行查询之前，它会确定如何完成查询，即选择最好的查询方法。

> 例如，您要出去参加一次大型家庭旅行，每个人都坐在车里准备离开，但是你突然发现忘了带20瓶水，你很快想起所有的瓶装水都在储藏室里，但是需要尽快把它们放到你的车上，因为其他人都在等你；你开始思考，每次可以手拿4瓶，来回跑5次，或者你也可以随身带一个箱子，把20瓶都装在箱子里，一起带上车不用再来回。这就是优化器所做的事情，它分析满足请求的所有不同方法，并选择其中最优的方法。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/04.webp" style="width:100%"/>

让我们看一个简单的 `SQL` 查询：

```SQL
SELECT name FROM employee_table WHERE employee_id = 1;
```

假设 `employee_table` 有 `10K` 条员工记录，至少有两种方法（或者是正式术语中的两个执行计划）：

- **执行计划一**：扫描 `name` 列中的所有姓名，对于每个姓名，检查其对应的 `employee_id` 是否为 `1`，如果 `employee_id = 1` 则返回该姓名；
- **执行计划二**：使用主键索引查找 `employee_id = 1` 的记录，并返回该记录名称内容。

方法二几乎总是会执行地更快，优化器会选择使用方法二，下一步就是真正的执行计划。

## 执行引擎

执行引擎将调用存储引擎的 `API` 来执行查询优化器所选择的执行计划。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/05.webp" style="width:100%"/>

## 存储引擎

很多软件系统都可以分为计算层和存储层。计算层的效率在很大程度上往往取决于数据在存储层中的组织方式。在本节中，让我们深入了解 `MySQL` 的存储引擎，以及为加快存储引擎的读写速度而精心设计的许多优化。

`MySQL` 可以集成许多种不同类型的存储引擎。对于不同的使用场景，每一种存储引擎都有各自的优缺点。换句话说存储引擎可以看作是一个接口，有不同的底层实现，比如有 `InnoDB`、`MyISAM`、`Memory`、`CSV`、`Archive`、`Merge`、`Blackhole`。

`InnoDB` 毫无无疑是使用最广泛的存储引擎，这是 `MySQL 5.5` 版本以后的默认设置。

就像我们使用的笔记本电脑一样，`InnoDB` 将数据存储在内存和磁盘上。从上层的角度来看，当写入 `InnoDB` 时，数据总是先写入内存的缓存空间中，然后再持久化到磁盘。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/06.webp" style="width:100%"/>

`InnoDB` 将内存分为两部分：

- 缓冲池；
- 日志缓冲区。

缓冲池对于 `InnoDB` 来说非常重要。`MySQL` 在处理查询时通常速度非常快，原因是数据实际上是存储在内存中并对外提供的服务的（在大多数情况下并不是从磁盘读取数据，这与许多人的想法恰恰相反），这个内存组件就是缓冲池。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/07.webp" style="width:100%"/>

## 缓冲池

通常来说宿主机 `80%` 的内存会分配给缓冲池使用，更大的内存可以缓存更多的数据到内存中，可以使读取速度更快。

缓冲池除了将数据简单地放在内存中，还针对数据在内存中的组织形式进行了精心设计，以加快数据库的读写速度，让我们来进一步深入了解这个详细设计。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/08.webp" style="width:100%"/>

## 页

类似于图书馆组织书籍的方式，使用 `ID` 标识书籍，并按字母或数字顺序放置在图书馆书架上，缓冲池也以类似的排序方式组织数据。

`InnoDB` 将缓冲池分成很多数据页，还包括一个 `change buffer`（后面会说明）。所有数据页都以双向链表的形式链接起来，也就是说我们可以很容易地从当前页到达下一页，或者从当前页到达上一页。

那么数据如何在页中存储的？

## 用户记录

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/09.webp" style="width:100%"/>

在一个数据页中包括如下内容：

- 指向上一个数据页的指针；
- 指向下一个数据页的指针；
- 用户记录；
- 其他属性信息。

指向上一个数据页和下一个数据页的指针简单来说就是一个指针，用户记录是存储每**行**数据的位置，每行都有一个指向下一行的 `next` 指针，形成一个单向链表。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/10.webp" style="width:100%"/>

那么问题来了。

我们知道关系型数据库中的一行记录通常由一个主键字段和许多其他字段组成，这些记录通常按主键排序，以便在查找具有特定主键的记录时可以使用二分查找等算法来快速检索数据，降低延迟；但是如果每次我们向用户记录添加新数据时，`InnoDB` 都需要重新排列所有记录（行）以保持有序，这将是一个非常耗时的操作。

实际上用户记录中的行是按插入顺序排列的，添加新记录只是意味着附加到用户记录的末尾，所需的主键顺序是通过 `next` 指针实现的，每行的 `next` 指针根据主键顺序指向下一个逻辑行，而不是内存中物理的下一行。

现在有一个新问题，前面我们提到不需要遍历所有记录就可以找到具有特定主键的目标数据，但是如果所有的行本质上都是一个单向链表，单向链表的特性决定了我们只能遍历整个链表才能找到特定的行，我们知道遍历链表是 `O(n)` 时间复杂度的操作，这是非常耗时的，那么 `InnoDB` 如何让它更快的呢？还记得我们之前提到的 `other fields` 吗？它们正是用于此目的。

## 下界与上界

这两个属性分别代表数据页中存储记录最大和最小的行，也就是说两者构成了一个 `min-max filter`。通过 `O(1)` 时间复杂度算法检查这两个字段，`InnoDB` 可以决定要查找的行是否存储在指定数据页中。

假设我们的主键是数值类型，并且我们的某个数据页的 `supremum = 1`，`infimum = 99`，如果我们试图查找 `primary key = 101` 的行，那么显然在 `O(1)` 时间内，`InnoDB` 就能够决定它不在这个数据页中，并且马上会转到另一个数据页继续查找。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/11.webp" style="width:100%"/>

如果要查找的记录不在某个数据页中，`InnoDB` 会跳过该页，但是如果该行记录在这个数据页中，那么 `InnoDB` 是否仍然需要遍历整个链表呢？答案是否定的，`other fields` 信息会再次派上用场。

## 页目录

顾名思义，它就像一本书的**目录**。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/09-Deep-Dive-Into-MySQL-Inner-Details/12.webp" style="width:100%"/>

页目录存储指向行块的指针，假设我们有第 `1` 行到第 `n` 行存储在用户记录中，页目录将有指向第 `1` 行、第 `7` 行、第 `13` 行……的指针，块大小为 `6`，这种设计与跳表结构非常相似。

正如我们现在能够想象的那样，`InnoDB` 首先检查数据页目录，然后快速确定要查找哪个数据块，然后利用指针跳转到单向链表中数据块对应的起始行，从这个位置开始遍历。这样 `InnoDB` 就避免了遍历整个列表，只需要遍历一个小得多的子列表就可以检索到数据。

我们上面提到的块，官方称之为 `Slot`。一个页目录会有多个槽，每个槽都有一个指针指向用户记录中的一行数据。

每次向用户记录插入新数据时，`InnoDB` 也会同时更新页目录以使两者保持一致，`InnoDB` 会每 `6` 行创建一个槽。

## 索引

`InnoDB` 使用 `B+` 树（读作**B加树**）进行数据存储。`MySQL` 支持两种类型的索引：**主键索引**和**二级索引**。一旦我们理解了数据页的概念，索引就变得显而易见了。

> 主键索引的叶子节点存储“page”
> 
> 相反二级索引的叶子节点存储主键信息，当查询命中二级索引时，首先从二级索引中检索主键值，一旦我们知道了主键值，就可以使用主键索引检索目标数据行。

至于为什么使用 `B+` 树而不是 `B-` 树（`B` 树，不是读作 `B` 减树）：

- `B+` 树减少了所需要的 `IO` 操作；
- 查询效率更稳定；
- 可以更好地支持范围查询。

<hr />

所有这些优秀的设计，例如用户记录之间使用单向链表、`infimum-supremum` 和页目录，主要目的是加快数据读取。那当我们想要更新一行记录时会发生什么？现在让我们研究一下 `MySQL` 是如何处理数据更新的。

## 更新数据

当我们向 `MySQL` 中插入一行记录，比如 `id = 100`，假如我们需要马上再更新这一行数据，由于这行数据刚刚插入可能还保存在缓冲池中。

但是如果我们等待相当长一段时间，然后再更新这一行，那么 `id = 100` 的这行记录很可能已经不在缓冲池中了。

因为内存有限，我们不能一直将插入的数据保存在内存中，假如这样内存很快就会被装满。清除策略，通常是说 `ttl(time-to-live)`，用于清理内存空间，例如，`Redis` 通常用作缓存，`Redis` 中的 `ttl` 表示键被删除（只标记为已删除且显示为不可读，然后实际由后台进程将其批量删除）。对于 `MySQL` 数据需要持久化，清除的意思是将数据持久化到磁盘并从内存中删除。

因此当尝试更新一行记录时，如果该行记录不在缓冲池中，`InnoDB` 会将数据从磁盘加载到缓冲池，这里的问题是 `InnoDB` 不能只加载 `id = 100` 这一行记录，它需要加载包含该行的整个数据页，当整个数据页加载到缓冲池时，就可以更新指定数据行了。

到目前为止好像一切看起来都不错，但这只是描述了对主键索引的更新，那么二级索引呢？

## 变更缓冲区

假设我们更新了 `id = 100` 行的某些字段信息，如果在其中一个字段上建立二级索引，那么在更新字段信息时需要同时更新二级索引。但是如果包含二级索引的数据页不在缓冲池中，那么 `InnoDB` 是否也会将数据页加载到缓冲池中呢？

假如二级索引并不是马上被使用到，如果每次更新相关字段的时候都立即将数据加载到缓冲池中，就会产生很多随机的磁盘 `I/O` 操作，这必然会拖慢 `InnoDB`。

这就是为什么 `Change Buffer` 被设计用来**延迟**更新二级索引的原因。

当二级索引需要更新时，而包含它的数据页不在缓冲池中时，更新的数据将会被暂时存储在变更缓冲区中。稍后如果数据页被加载到缓冲池中（由于主键索引的更新），`InnoDB` 会将变更缓冲区中保存的变更数据合并到缓冲池的数据页中。

不过这种变更缓冲区设计有一个缺点，如果变更缓冲区已经积累了大量的临时变更数据，如果我们要一次性全部合并到 `Buffer Pool` 中，可能需要几个小时才能完成！！并且在合并的过程中，会产生大量的磁盘 `I/O` 操作，占用 `CPU` 周期，影响 `MySQL` 的整体性能。

所以这里进一步做了一些权衡，我们之前提到当包含二级索引的页面加载到缓冲池时会触发合并操作。在 `InnoDB` 设计中，为避免出现可能需要几个小时的大型合并操作，合并操作也可以由其他事件触发，这些事件包括**事务**、**服务器关闭**和**服务器重启**等。

## 自适应哈希索引

自适应哈希索引旨在与缓冲区一起工作，自适应哈希索引使 `InnoDB` 存储引擎的性能更像内存数据库。自适应哈希索引可以通过 `innodb_adaptive_hash_index` 变量开启，或在服务器启动时通过 `--skip-innodb-adaptive-hash-index` 关闭。
