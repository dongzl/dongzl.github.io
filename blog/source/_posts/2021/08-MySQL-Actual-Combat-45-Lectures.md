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
