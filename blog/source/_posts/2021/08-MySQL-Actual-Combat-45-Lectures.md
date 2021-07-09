---
title: MySQL 实战45讲
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

![](https://static001.geekbang.org/resource/image/0d/d9/0d2070e8f84c4801adbfa03bda1f98d9.png)

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

![](https://static001.geekbang.org/resource/image/16/a7/16a7950217b3f0f4ed02db5db59562a7.png)

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
6. 物理一致性和逻辑一直性各应该怎么理解？
7. 执行器和 InnoDB 在执行 update 语句时候的流程是什么样的？
8. 如果数据库误操作, 如何执行数据恢复？
9. 什么是两阶段提交, 为什么需要两阶段提交, 两阶段提交怎么保证数据库中两份日志间的逻辑一致性（什么叫逻辑一致性）？
10. 如果不是两阶段提交, 先写 redo log 和先写 binlog 两种情况各会遇到什么问题？
