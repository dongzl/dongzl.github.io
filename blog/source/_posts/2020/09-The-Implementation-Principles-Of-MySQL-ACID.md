---
title: MySQL 数据库事务 ACID 的实现原理
date: 2020-03-13 21:39:26
cover: https://gitee.com/dongzl/article-images/raw/master/cover/mysql_study.png
# author information, multiple authors are set to array
# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨在通过一道面试题总结 MySQL 数据库事务 ACID 的实现原理。

categories: 
  - 数据库
tags: 
  - MySQL
  - 事务
---

## 面试题

> MySQL 数据库的原子性和持久性怎么保证？

> 技术关键点：通过 undo log 保证原子性；通过 redo log 保证持久性。

## 原理分析

首先介绍一下 MySQL 事务的 ACID 特性：

- 原子性（Atomicity）：事务中所有操作作为一个整体像原子一样不可分割，要么全部成功，要么全部失败；

- 一致性（Consistency）：事务的执行结果必须使数据从一个一致性状态到另一个一致性状态。一致性状态是指：a. 系统的状态满足数据的完整性约束；b. 系统的状态反映数据库本应描述现实世界的真实状态，比如转账前后两个账户的金额总和应该保持不变；

- 隔离性（Isolation）：并发执行的事务不会互相影响，其对数据库的影响和他们串行执行时一样。比如多个用户同时往一个账户转账，最后账户的结果应该和他们按先后次序转账的结果一样；

- 持久性（Durability）：事务一旦提交，其对数据库的更新就是持久的。任何事务或系统故障都不会导致数据丢失。

### 事务的特点

事务的根本追求：数据一致性

可能会对事务一致性造成破坏的原因：

- 事务的并发执行

- 事务故障或系统故障

避免事务一致性被破坏的技术手段：

- 并发控制技术（保证事务隔离性，防止事务并发执行破坏数据的一致性）

- 日志恢复技术（保证事务的原子性和持久性，防止事务故障或系统故障破坏数据一致性）

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/09-The-Implementation-Principles-Of-MySQL-ACID/The-Implementation-Principles-Of-MySQL-ACID_01.png">

### 事务的实现原理

- 原子性：通过 undo log 来实现

- 持久性：通过 redo log 来实现

- 隔离性：读写锁 + MVCC 来实现

- 一致性：通过 原子性 + 隔离性 + 持久性 来实现

**undo log 作用以及实现原理**

作用：保证事务原子性；实现多版本并发控制（MVCC）

原理：在操作任何数据之前，先将数据备份到 undo log，然后进行数据的修改，如果出现了错误或者 ROLLBACK 回滚事务，可以利用 undo log 中的备份将数据回复到事务开始之前的状态，undo log 是逻辑日志，可以理解为：

- 当 delete 一条记录时，undo log 中会记录一条对应的 insert 记录；
- 当 insert 一条记录时，undo log 中会记录一条对应的 delete 记录；
- 当 update 一条记录时，它记录一条对应相反的 update 记录。

**redo log 作用以及实现原理**

作用：保证事务的持久性

原理：redo log 记录的是新数据的备份。在事务提交前，只要将 redo log 持久化即可，不需要立即将数据持久化（预写式日志：WAL）。当系统崩溃时，虽然数据没有持久化，但是 redo log 已经持久化。系统可以根据 redo log 的内容，将所有的数据恢复到最新的状态。

**事务的隔离性**

事务具有隔离性，理论上来说事务之间的执行不应该互相影响，其对数据库的影响应该和串行执行时一样。

然而完全的隔离级别会导致系统并发性能很低，降低对资源的利用率，因此对事务的隔离性要求会放宽，这也会一定程度上造成对数据库一致性要求降低。

SQL 标准定义的事务的隔离级别：

- 读未提交（READ UNCOMMITTED）：对事务处理没有任何限制，不推荐

- 读已提交（READ COMMITTED）：Oracle 数据库默认的隔离级别

- 可重复读（REPEATABLE READ）：MySQL 数据库默认隔离级别

- 串行化（SERIALIZABLE）：并发性能最低，不推荐

不同的隔离级别可能导致的并发异常：

事务的隔离级别 | 脏读 |  不可重复读 | 幻读
-|-|-|-
读未提交（READ UNCOMMITTED） | YES | YES | YES
读已提交（READ COMMITTED）  |    | YES | YES
可重复读（REPEATABLE READ） |    |     | YES
串行化（SERIALIZABLE）     |     |     |   

设置事务的隔离级别操作：

```SQL
// 查看 MySQL 事务是否自动提交
show session variables like 'autocommit';
select @@autocommit;

// MySQL 关闭事务自动提交
set session autocommit=0;

// 查看 MySQL 当前事务隔离级别
SELECT @@tx_isolation;

// 修改 MySQL 事务隔离级别
// 设置read uncommitted级别：
set session transaction isolation level read uncommitted;

// 设置read committed级别：
set session transaction isolation level read committed;

// 设置repeatable read级别：
set session transaction isolation level repeatable read;

// 设置serializable级别：
set session transaction isolation level serializable;
```

**事务的隔离性实现原理：锁**

MySQL 锁分类：

- 共享锁（读锁）：数据只能读取，不能更新；

- 排他锁（写锁）：执行写入操作时，其他事务不能读取。

锁粒度：锁定对象的大小就是锁的粒度：记录 / 表。

基于锁的并发流程控制：

- 事务根据自己对数据项进行的操作类型申请相应的锁（读申请共享锁，写申请排他锁）

- 申请锁的请求被发送给锁管理器。锁管理器根据当前数据项是否已经有锁以及申请的和持有的锁是否冲突决定是否为该请求授予锁；

- 若锁被授予，则申请锁的事务可以继续执行；若被拒绝，则申请锁的事务将进行等待，知道锁被其他事务释放。

## 面试题

> MySQL 中 InnoDB 存储引擎和 MyISAM 存储引擎的区别？

[Mysql 中 MyISAM 和 InnoDB 的区别有哪些？](https://www.zhihu.com/question/20596402?sort=created)

 特征| MyISAM |  InnoDB 
-|-|-
索引类型 | 非聚簇索引 | 聚簇索引
支持事务 | 否    | 是
支持表锁 | 是    | 是
支持行锁 | 否    | 是
支持外键 | 否    | 是
支持全文索引 | 是 | 是（5.6版本后支持）
适合操作类型 | 大量select |  大量 inset/delete/update 
文件组织形式 | .frm / .ibd | .MYD / .MYI / .frm


**其他区别内容：**

- MyISAM：.frm文件存储表定义；数据文件的扩展名为.MYD(MYData)；索引文件的扩展名是.MYI (MYIndex)。

- InnoDB：.frm文件存储表定义；.ibd 文件和 .ibdata 文件：这两种文件都是存放InnoDB 数据的文件，之所以用两种文件来存放 文件：这两种文件都是存放InnoDB 的数据，是因为 文件：这两种文件都是存放InnoDB 的数据存储方式能够通过配置来决定是使用共享表空间存放存储数据，还是用独享表空间存放存储数据。独享表空间存储方式使用 .ibd 文件，并且每个表一个 .ibd 文件；共享表空间存储方式使用 .ibdata 文件，所有表共同使用一个 .ibdata 文件。

- MyISAM：允许没有任何索引和主键的表存在，索引都是保存行的地址。

- InnoDB：如果没有设定主键或者非空唯一索引，就会自动生成一个6字节的主键(用户不可见)，数据是主索引的一部分，附加索引保存的是主索引的值。

- MyISAM： 保存有表的总行数，如果 select count() from table; 会直接取出出该值。

- InnoDB： 没有保存表的总行数，如果使用 select count(*) from table; 就会遍历整个表，消耗相当大，但是在加了 wehre 条件后，MyISAM 和 InnoDB 处理的方式都一样。

{% pdf https://gitee.com/dongzl/article-images/raw/master/2020/09-The-Implementation-Principles-Of-MySQL-ACID/The-Implementation-Principles-Of-MySQL-ACID_02.pdf %}

- 参考资料

- [详细分析MySQL事务日志(redo log和undo log)](https://www.cnblogs.com/f-ck-need-u/archive/2018/05/08/9010872.html)

- [Mysql 中 MyISAM 和 InnoDB 的区别有哪些？](https://www.zhihu.com/question/20596402?sort=created)
