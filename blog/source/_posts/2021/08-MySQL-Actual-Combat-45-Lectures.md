---
title: MySQL 实战45讲--基础篇
date: 2021-07-08 09:16:39
cover: https://gitee.com/dongzl/article-images/raw/master/cover/mysql_geek_time.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文是对极客时间《MySQL 实战45讲》课程学习的读书笔记总结。

categories: 
  - 数据库

tags: 
  - Redis
---

# 开篇词

## 开篇词 | 这一次，让我们一起来搞懂MySQL

<!-- ![](https://static001.geekbang.org/resource/image/b7/c2/b736f37014d28199c2457a67ed669bc2.jpg) -->

<hr/>

# 基础篇

## 01 | 基础架构：一条SQL查询语句是如何执行的？

MySQL 基本架构示意图：

<img src="https://static001.geekbang.org/resource/image/0d/d9/0d2070e8f84c4801adbfa03bda1f98d9.png" style="width:400px"/>

- MySQL 分为 Server 层和存储引擎层；
- Server 层包括连接器、查询缓存、分析器、优化器、执行器等，存储引擎层负责数据的存储和提取；
- 不同的存储引擎共用一个 Server 层；

### 连接器

- 负责跟客户端建立连接、获取权限、维持和管理连接；
- wait_timeout=8小时，连接器自动断开没动静的客户端连接。

### 查询缓存

- 查询缓存往往弊大于利：表上有更新操作就会导致查询缓存失效；
- MySQL 8.0 版本已经移除查询缓存功能。

### 分析器

- 词法分析：识别关键字、表名、列名；
- 语法分析：识别语法是否正确。

### 优化器

- 索引选择、表关联顺序；

### 执行器

- 执行 SQL 语句；

### 问题整理

1、如果表 T 中没有字段 k，而你执行了这个语句 select * from T where k=1, 那肯定是会报“不存在这个列”的错误： “Unknown column ‘k’ in ‘where clause’”。

这个是在分析器阶段报出的错误。

2、为什么权限检查不是在分析器阶段，而是要到执行器之前检查？

执行过程中可能会有触发器这种在运行时才能确定的过程，分析器工作结束后的 precheck 是不能对这种运行时涉及到的表进行权限校验的，所以需要在执行器阶段进行权限检查。

<hr/>

## 02 | 日志系统：一条SQL更新语句是如何执行的？

WAL 的全称是 Write-Ahead Logging，它的关键点就是先写日志，再写磁盘。

InnoDB 的 redo log 是固定大小的，从头开始写，写到末尾就又回到开头循环写。

<img src="https://static001.geekbang.org/resource/image/16/a7/16a7950217b3f0f4ed02db5db59562a7.png" style="width:400px"/>

有了 redo log，InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力成为 cash-safe。

redo log -- MySQL InnoDB 存储引擎特有的

binlog -- MySQL Server 层日志（归档日志）

先有 MySQL Server 层，先有 binlog，binlog 无法做到 cash-safe；InnoDB 存储引擎后出现，通过 redo log 实现 cash-safe。

MySQL binlog 和 InnoDB redo log 的区别：

- redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。

- redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。

- redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

redo log 日志的写入采用两阶段提交方式，写入 redo log 但是处于 prepare 阶段，写入 binlog，commit 提交事务，完成 redo log 写入。

- innodb_flush_log_at_trx_commit 这个参数设置成 1 的时候，表示每次事务的 redo log 都直接持久化到磁盘。

- sync_binlog 这个参数设置成 1 的时候，表示每次事务的 binlog 都持久化到磁盘。

### 问题整理

1、有了 redo log，binlog 的存在是否多余呢？

不多余，binlog 有存在意义。

- redo log 只有 InnoDB 有，其他存储引擎没有；

- redo log 是循环写的，无法持久保存，binlog 的“归档”功能，redo log 是不具备的。

2、如何理解 binlog 是”逻辑日志“，redo log 是”物理日志“？

逻辑日志可以给别的数据库，别的引擎使用，已经大家都讲得通这个“逻辑”；
物理日志就只有“我”自己能用，别人没有共享我的“物理格式”。

3、需要掌握的知识点问题：

1. redo log 的概念是什么？为什么会存在？
2. 什么是WAL(write-ahead log)机制, 好处是什么？
3. redo log 为什么可以保证 crash-safe 机制？
4. binlog 的概念是什么, 起到什么作用, 可以做 crash-safe 吗？
5. binlog 和 redo log 的不同点有哪些？
6. 物理一致性和逻辑一致性各应该怎么理解？
7. 执行器和 InnoDB 在执行 update 语句时候的流程是什么样的？
8. 如果数据库误操作, 如何执行数据恢复？
9. 什么是两阶段提交, 为什么需要两阶段提交, 两阶段提交怎么保证数据库中两份日志间的逻辑一致性（什么叫逻辑一致性）？
10. 如果不是两阶段提交, 先写 redo log 和先写 binlog 两种情况各会遇到什么问题？

那么在什么场景下，一天一备会比一周一备更有优势呢？或者说，它影响了这个数据库系统的哪个指标？

好处是“最长恢复时间”更短。在一天一备的模式里，最坏情况下需要应用一天的 binlog。比如，你每天 0 点做一次全量备份，而要恢复出一个到昨天晚上 23 点的备份。一周一备最坏情况就要应用一周的 binlog 了。系统的对应指标就是 @尼古拉斯·赵四 @慕塔 提到的 RTO（恢复目标时间）。当然这个是有成本的，因为更频繁全量备份需要消耗更多存储空间，所以这个 RTO 是成本换来的，就需要你根据业务重要性来评估了。

<hr/>

## 03 | 事务隔离：为什么你改了我还看不见？

在 MySQL 中事务是在引擎层实现的，MySQL 是一个支持多引擎的数据库系统，但是不是所有的引擎都支持事务。

ACID（Atomicity、Consistency、Isolation、Durability，即原子性、一致性、隔离性、持久性）。

SQL 标准隔离级别：读未提交（read uncommitted）、读提交（read committed）、可重复读（repeatable read）和串行化（serializable ）。

- 读未提交是指，一个事务还没提交时，它做的变更就能被别的事务看到。
- 读提交是指，一个事务提交之后，它做的变更才会被其他事务看到。
- 可重复读是指，一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。当然在可重复读隔离级别下，未提交变更对其他事务也是不可见的。
- 串行化，顾名思义是对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行。

为实现数据库的隔离级别，数据库内部会创建视图，访问的时候以视图的逻辑结果为准。在“可重复读”隔离级别下，这个视图是在事务启动时创建的，整个事务存在期间都用这个视图。在“读提交”隔离级别下，这个视图是在每个 SQL 语句开始执行的时候创建的。

### 事务隔离的实现

在 MySQL 中，实际上每条记录在更新的时候都会同时记录一条回滚操作。记录上的最新值，通过回滚操作，都可以得到前一个状态的值。

<img src="https://static001.geekbang.org/resource/image/d9/ee/d9c313809e5ac148fc39feff532f0fee.png" style="width:400px"/>

回滚日志的删除：当系统里没有比这个回滚日志更早的 read-view 的时候，回滚日志会被删除。

为什么不建议使用长事务？

长事务意味着系统里面会存在很老的事务视图。由于这些事情随时可能访问数据库里的任何数据，所以这个事务提交之前，数据库里面他可能用到的回滚记录都必须保留，这就会导致占用大量的存储空间。

除了对回滚段的影响，长事务还占用锁资源，也可能拖垮整个库。

如何查询数据库长事务？

```SQL
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60
```

### 事务的启动方式

- 显式启动事务语句， begin 或 start transaction。配套的提交语句是 commit，回滚语句是 rollback。

- set autocommit=0，这个命令会将这个线程的自动提交关掉。

建议使用方式：建议总是使用 set autocommit=1, 通过显式语句的方式来启动事务；如果减少与数据库交互次数，可以使用 commit work and chain 语法。

### 问题整理

你现在知道了系统里面应该避免长事务，如果你是业务开发负责人同时也是数据库负责人，你会有什么方案来避免出现或者处理这种情况呢？

- 首先，从应用开发端来看：
  - 确认是否使用了 set autocommit=0。这个确认工作可以在测试环境中开展，把 MySQL 的 general_log 开起来，然后随便跑一个业务逻辑，通过 general_log 的日志来确认。一般框架如果会设置这个值，也就会提供参数来控制行为，你的目标就是把它改成 1。

  - 确认是否有不必要的只读事务。有些框架会习惯不管什么语句先用 begin/commit 框起来。我见过有些是业务并没有这个需要，但是也把好几个 select 语句放到了事务中。这种只读事务可以去掉。
  
  - 业务连接数据库的时候，根据业务本身的预估，通过 SET MAX_EXECUTION_TIME 命令，来控制每个语句执行的最长时间，避免单个语句意外执行太长时间。（为什么会意外？在后续的文章中会提到这类案例）

- 其次，从数据库端来看：
  - 监控 information_schema.Innodb_trx 表，设置长事务阈值，超过就报警 / 或者 kill；
  - Percona 的 pt-kill 这个工具不错，推荐使用；
  - 在业务功能测试阶段要求输出所有的 general_log，分析日志行为提前发现问题；
  - 如果使用的是 MySQL 5.6 或者更新版本，把 innodb_undo_tablespaces 设置成 2（或更大的值）。如果真的出现大事务导致回滚段过大，这样设置后清理起来更方便。

<hr/>

## 04 | 深入浅出索引（上）

索引的出现就是为了提高数据的查询效率，就像书的目录一样。

### 常见的索引模型

1、哈希表

<img src="https://static001.geekbang.org/resource/image/0c/57/0c62b601afda86fe5d0fe57346ace957.png" style="width:400px"/>

- 哈希表这种结构适用于只有等值查询的场景；

2、有序数组

<img src="https://static001.geekbang.org/resource/image/bf/49/bfc907a92f99cadf5493cf0afac9ca49.png" style="width:400px"/>

- 有序数组在等值查询和范围查询场景中的性能就都非常优秀。
  - 等值查询：使用二分法，时间复杂度 O(log(N))。
  - 范围查询：使用二分法，定位查询起点，进行范围查询。

- 有序数组索引适用于静态存储引擎，如果需要在有序数组中间插入一个记录就必须得挪动后面所有的记录，成本太高。

3、二叉搜索树

<img src="https://static001.geekbang.org/resource/image/04/68/04fb9d24065635a6a637c25ba9ddde68.png" style="width:400px"/>

二叉搜索树的特点是：父节点左子树所有结点的值小于父节点的值，右子树所有结点的值大于父节点的值。

查找某个结点的时间复杂度：O(log(N))。

新增某个结点的时间复杂度：O(log(N))，为了维持平衡二叉树。

问题点：

1、如何计算一棵二叉树的高度，比如有100W个结点二叉树高度，高度是20。
2、以 InnoDB 的一个整数字段索引为例，N 差树的这个 N 差不多是 1200。

### InnoDB 的索引模型

在 MySQL 中，索引是在存储引擎层实现的，所以并没有统一的索引标准，即不同存储引擎的索引的工作方式并不一样。而即使多个存储引擎支持同一种类型的索引，其底层的实现也可能不同。

在 InnoDB 中，表都是根据主键顺序以索引的形式存放的，这种存储方式的表称为索引组织表。

InnoDB 使用了 B+ 树索引模型，所以数据都是存储在 B+ 树中的；每一个索引在 InnoDB 里面对应一棵 B+ 树。

<img src="https://static001.geekbang.org/resource/image/dc/8d/dcda101051f28502bd5c4402b292e38d.png" style="width:400px"/>

- 主键索引的叶子节点存的是整行数据。在 InnoDB 里，主键索引也被称为聚簇索引（clustered index）。

- 非主键索引的叶子节点内容是主键的值。在 InnoDB 里，非主键索引也被称为二级索引（secondary index）。

- 如果语句是 select * from T where ID=500，即主键查询方式，则只需要搜索 ID 这棵 B+ 树；

- 如果语句是 select * from T where k=5，即普通索引查询方式，则需要先搜索 k 索引树，得到 ID 的值为 500，再到 ID 索引树搜索一次。这个过程称为回表。

基于非主键索引的查询需要多扫描一棵索引树。因此，我们在应用中应该尽量使用主键查询。

### 索引维护

B+ 树为了维护索引有序性，在插入新值的时候需要做必要的维护。

在中间插入数据，需要逻辑上挪动后面的数据，空出一个位置。如果插入数据的数据页已满，会触发页分裂。

- 页分裂会影响数据库的性能；
- 页分裂还会影响数据页的利用率。

删除数据可能会触发页合并，当相邻两个页由于删除了数据，利用率很低之后，会将数据页做合并。合并的过程，可以认为是分裂过程的逆过程。

自增主键 vs 业务唯一字段主键

自增主键由于具备单调自增性，所以不会涉及挪动其他记录，也不会出现页分裂问题。

业务唯一字段做主键，插入顺序不固定，可能会出现挪动其他记录，触发页分裂。

自增主键占用空间比较小，业务唯一字段可能占用空间比较大；主键长度越小，普通索引的叶子节点就越小，普通索引占用的空间也就越小。

从性能和存储空间方面考量，自增主键往往是更合理的选择。

- 自增主键不会触发其他记录挪动，也不会触发页分裂；

- 自增主键长度比较小，普通索引占用空间也就越小。

业务唯一字段做主键的场景：KV场景。

- 只有一个索引；
- 该索引必须是唯一索引。

### 问题总结

1、“N 叉树”的 N 值在 MySQL 中是可以被人工调整的么？

- 通过改变 key 值来调整

N 叉树中非叶子节点存放的是索引信息，索引包含 Key 和 Point 指针。Point 指针固定为 6 个字节，假如 Key 为 10 个字节，那么单个索引就是 16 个字节。如果 B+ 树中页大小为 16K，那么一个页就可以存储 1024 个索引，此时 N 就等于 1024。我们通过改变 Key 的大小，就可以改变 N 的值；

- 改变页的大小

页越大，一页存放的索引就越多，N就越大。数据页调整后，如果数据页太小层数会太深，数据页太大，加载到内存的时间和单个数据页查询时间会提高，需要达到平衡才行。

2、请问没有主键的表，有一个普通索引。怎么回表？

没有主键的表，innodb会给默认创建一个Rowid做主键

### 链接地址

- [B+Tree index structures in InnoDB](https://blog.jcole.us/2013/01/10/btree-index-structures-in-innodb/)

- [一个案例彻底弄懂如何正确使用 mysql inndb 联合索引](https://mengkang.net/1302.html)

- [InnoDB一棵B+树可以存放多少行数据？](https://www.cnblogs.com/leefreeman/p/8315844.html)

<hr/>

