---
title: MySQL 数据库 Lock 知识总结
date: 2020-03-19 22:13:22
cover: https://gitee.com/dongzl/article-images/raw/master/cover/mysql_study.png
# author information, multiple authors are set to array
# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨在总结 MySQL 数据库 InnoDB 和 MyISAM 存储引擎锁相关知识内容。

categories: 
  - 数据库
tags: 
  - MySQL
  - Lock
---

## MySQL 中锁的基本知识

**锁是计算机协调多个进程或线程并发访问某一资源的机制。** 在数据库中，除传统的计算资源（如 CPU、RAM、I/O 等）的争用以外，数据也是一种供许多用户共享的资源。如何保证数据并发访问的一致性、有效性是所有数据库必须解决的一个问题，锁冲突也是影响数据库并发访问性能的一个重要因素。从这个角度来说，锁对数据库而言显得尤其重要，也更加复杂。

​相对其他数据库而言，MySQL 的锁机制比较简单，其最显著的特点是不同的 **存储引擎** 支持不同的锁机制。比如，MyISAM 和 MEMORY 存储引擎采用的是表级锁（table-level locking）；InnoDB 存储引擎既支持行级锁（row-level locking），也支持表级锁，但默认情况下是采用行级锁。

- **表级锁**：开销小，加锁快；不会出现死锁；锁定粒度大，发生锁冲突的概率最高，并发度最低； 

- **行级锁**：开销大，加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低，并发度也最高。  

​从上述特点可见，很难笼统地说哪种锁更好，只能就具体应用的特点来说哪种锁更合适。仅从锁的角度来说：**表级锁**更适合于以查询为主，只有少量按索引条件更新数据的应用，如Web应用；而**行级锁**则更适合于有大量按索引条件并发更新少量不同数据，同时又有并发查询的应用，如一些在线事务处理（OLTP）系统。 

## MySQL 中锁的分类

- 共享锁：Shared Locks（简称 S 锁，属于行锁）

- 排他锁：Exclusive Locks（简称 X 锁，属于行锁）

- 意向共享锁：Intention Shared Locks（简称 IS 锁，属于表锁）

- 意向排他锁：Intention Exclusive Locks（简称 IX 锁，属于表锁）

- 自增锁 AUTO-INC Locks

### 共享锁

共享锁就是多个事务对于同一数据可以共享一把锁，都能访问到数据库，但是只能读不能修改；

```sql
#事务A：
select * from student where id = 1 lock in share mode;

#事务B:
select * from student where id = 1; --(读取数据没问题)

#事务B:
update student set name='hehe' where id  =1;
--注意：无法修改会卡死，当事务A提交事务之后，会立刻修改成功
```
### 排他锁

排它锁不能与其他锁并存，如一个事务获取了一个数据行的排它锁，其他事务就不能再获取改行的锁，只有当前获取了排它锁的事务可以对数据进行读取和修改。`delete、update、insert` 默认是排他锁：

```sql
#事务A:
select * from student where id = 1 for update;

#事务B:
select * from student where id = 1 for update;

select * from student where id = 1 lock in share mode;

-- 注意：事务B操作的时候回卡死，提交事务立马成功。
```

### 意向共享锁和意向排他锁

- 意向共享锁：表示事务准备给数据行加入共享锁，也就是说一个数据行在加共享锁之前必须先取得该表的 IS 锁;

- 意向排他锁：表示事务准备给数据行加入排它锁，也就是说一个数据行加排它锁之前必须先取得该表的 IX 锁。

**PS. 意向锁是InnoDB数据操作之前自动加的，不需要用户干预。**

### 自增锁

针对自增列自增长的一个特殊的表级别锁

```sql
SHOW VARIABLES LIKE 'innodb_autoinc_lock_mode';
--默认值1 代表连续，事务未提交则id永久丢失
```

## MyISAM 表锁

MySQL 的表级锁有两种模式：**表共享读锁（Table Read Lock）**和**表独占写锁（Table Write Lock）**。  

对 MyISAM 表的读操作，不会阻塞其他用户对同一表的读请求，但会阻塞对同一表的写请求；对 MyISAM 表的写操作，则会阻塞其他用户对同一表的读和写操作；MyISAM 表的读操作与写操作之间，以及写操作之间是串行的！ 

建表语句：

```sql
CREATE TABLE `mylock` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `NAME` varchar(20) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;

INSERT INTO `mylock` (`id`, `NAME`) VALUES ('1', 'a');
INSERT INTO `mylock` (`id`, `NAME`) VALUES ('2', 'b');
INSERT INTO `mylock` (`id`, `NAME`) VALUES ('3', 'c');
INSERT INTO `mylock` (`id`, `NAME`) VALUES ('4', 'd');
```

**MyISAM写锁阻塞读的案例：**

​当一个线程获得对一个表的写锁之后，只有持有锁的线程可以对表进行更新操作。其他线程的读写操作都会等待，直到锁释放为止。

|                           session1                           |                         session2                          |
| :----------------------------------------------------------: | :-------------------------------------------------------: |
|       获取表的write锁定<br />lock table mylock write;        |                                                           |
| 当前session对表的查询，插入，更新操作都可以执行<br />select * from mylock;<br />insert into mylock values(5,'e'); | 当前session对表的查询会被阻塞<br />select * from mylock； |
|                释放锁：<br />unlock tables；                 |          当前session能够立刻执行，并返回对应结果          |


**MyISAM读阻塞写的案例：**

一个 session 使用 `lock table` 给表加读锁，这个 session 可以锁定表中的记录，但更新和访问其他表都会提示错误，同时，另一个 session 可以查询表中的记录，但更新就会出现锁等待。

|                           session1                           |                           session2                           |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
|        获得表的read锁定<br />lock table mylock read;         |                                                              |
|   当前session可以查询该表记录：<br />select * from mylock;   |   当前session可以查询该表记录：<br />select * from mylock;   |
| 当前session不能查询没有锁定的表<br />select * from person;<br />Table 'person' was not locked with LOCK TABLES | 当前session可以查询或者更新未锁定的表<br />select * from person;<br />insert into person values(1,'zhangsan'); |
| 当前session插入或者更新表会提示错误<br />insert into mylock values(6,'f');<br />Table 'mylock' was locked with a READ lock and can't be updated<br />update mylock set name='aa' where id = 1;<br />Table 'mylock' was locked with a READ lock and can't be updated | 当前session插入数据会等待获得锁<br />insert into mylock values(6,'f'); |
|                  释放锁<br />unlock tables;                  |                       获得锁，更新成功                       |

**注意：MyISAM 在执行查询语句之前，会自动给涉及的所有表加读锁，在执行更新操作前，会自动给涉及的表加写锁，这个过程并不需要用户干预，因此用户一般不需要使用命令来显式加锁，上例中的加锁时为了演示效果。**

**MyISAM 的并发插入问题：**

MyISAM 表的读和写是串行的，这是就总体而言的，在一定条件下，MyISAM 也支持查询和插入操作的并发执行

|                           session1                           |                           session2                           |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
|   获取表的read local锁定<br />lock table mylock read local;   |                                                              |
| 当前session不能对表进行更新或者插入操作<br />insert into mylock values(6,'f');<br />Table 'mylock' was locked with a READ lock and can't be updated<br />update mylock set name='aa' where id = 1;<br />Table 'mylock' was locked with a READ lock and can't be updated |    其他session可以查询该表的记录<br />select* from mylock    |
| 当前session不能查询没有锁定的表<br />select * from person;<br />Table 'person' was not locked with LOCK TABLES | 其他session可以进行<font color="red">插入</font>操作，但是<font color="red">更新</font>会阻塞<br />update mylock set name = 'aa' where id = 1; |
|          当前session不能访问其他session插入的记录；          |                                                              |
|                  释放锁资源：unlock tables;                   |               当前session获取锁，更新操作完成                |
|           当前session可以查看其他session插入的记录           |                                                              |

 可以通过检查 `table_locks_waited` 和 `table_locks_immediate `状态变量来分析系统上的表锁定争夺：

```sql
mysql> show status like 'table%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| Table_locks_immediate | 352   |
| Table_locks_waited    | 2     |
+-----------------------+-------+
--如果 Table_locks_waited 的值比较高，则说明存在着较严重的表级锁争用情况。
```

## InnoDB锁

可以通过检查 `Innodb_row_lock` 状态变量来分析系统上的行锁的争夺情况：

```sql
mysql> show status like 'innodb_row_lock%';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| Innodb_row_lock_current_waits | 0     |
| Innodb_row_lock_time          | 18702 |
| Innodb_row_lock_time_avg      | 18702 |
| Innodb_row_lock_time_max      | 18702 |
| Innodb_row_lock_waits         | 1     |
+-------------------------------+-------+
--如果发现锁争用比较严重，如InnoDB_row_lock_waits和InnoDB_row_lock_time_avg的值比较高
```

**InnoDB的行锁模式及加锁方法**：

- **共享锁（s）**：又称读锁。允许一个事务去读一行，阻止其他事务获得相同数据集的排他锁。若事务 T 对数据对象 A 加上 S 锁，则事务 T 可以读 A 但不能修改 A，其他事务只能再对 A 加 S 锁，而不能加 X 锁，直到 T 释放 A 上的 S 锁，这保证了其他事务可以读 A，但在 T 释放 A 上的 S 锁之前不能对 A 做任何修改；
​
- **排他锁（x）**：又称写锁。允许获取排他锁的事务更新数据，阻止其他事务取得相同的数据集共享读锁和排他写锁。若事务 T 对数据对象 A 加上 X 锁，事务 T 可以读 A 也可以修改 A，其他事务不能再对 A 加任何锁，直到 T 释放 A 上的锁。

​MySQL InnoDB引 擎默认的修改数据语句：**update, delete, insert都会自动给涉及到的数据加上排他锁，select 语句默认不会加任何锁类型**，如果加排他锁可以使用 `select …for update` 语句，加共享锁可以使用 `select … lock in share mode` 语句。**所以加过排他锁的数据行在其他事务种是不能修改数据的，也不能通过for update和lock in share mode锁的方式查询数据，但可以直接通过select …from…查询数据，因为普通查询没有任何锁机制。** 

**InnoDB行锁实现方式**：

InnoDB 行锁是通过给**索引**上的索引项加锁来实现的，这一点 MySQL 与 Oracle 不同，后者是通过在数据块中对相应数据行加锁来实现的。InnoDB 这种行锁实现特点意味着：只有通过索引条件检索数据，InnoDB 才使用行级锁，**否则，InnoDB将使用表锁！**  

- 在不通过索引条件查询的时候，InnoDB 使用的是表锁而不是行锁

```sql
create table tab_no_index(id int,name varchar(10)) engine=innodb;
insert into tab_no_index values(1,'1'),(2,'2'),(3,'3'),(4,'4');
```

|                           session1                           |                           session2                           |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| set autocommit = 0;<br />select * from tab_no_index where id = 1; | set autocommit = 0;<br />select * from tab_no_index where id = 2; |
|      select * from tab_no_index where id = 1 for update;      |                                                              |
|                                                              |     select * from tab_no_index where id = 2 for update;      |

session1 只给一行加了排他锁，但是 session2 在请求其他行的排他锁的时候，会出现锁等待。原因是在没有索引的情况下，InnoDB 只能使用表锁。

- 创建带索引的表进行条件查询，InnoDB 使用的是行锁

```sql
create table tab_with_index(id int,name varchar(10)) engine=innodb;
alter table tab_with_index add index id(id);
insert into tab_with_index values(1,'1'),(2,'2'),(3,'3'),(4,'4');
```

|                           session1                           |                           session2                           |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| set autocommit = 0;<br />select * from tab_with_indexwhere id = 1; | set autocommit = 0;<br />select * from tab_with_indexwhere id =2; |
|     select * from tab_with_indexwhere id = 1 for update;      |                                                              |
|                                                              |     select * from tab_with_indexwhere id = 2 for update;     |

- 由于 MySQL 的行锁是针对索引加的锁，不是针对记录加的锁，所以虽然是访问不同行的记录，但是依然无法访问到具体的数据

```sql
alter table tab_with_index drop index id;
insert into tab_with_index  values(1,'4');
```

|                           session1                           |                           session2                           |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
|                       set autocommit = 0;                  |                       set autocommit = 0;                       |
| select * from tab_with_index where id = 1 and name='1' for update; |                                                              |
|                                                              | select * from tab_with_index where id = 1 and name='4' for update;<br />虽然 session2 访问的是和 session1 不同的记录，但是锁的是具体的表，所以需要等待锁 |

## 总结

**对于MyISAM的表锁，主要讨论了以下几点：** 
- 共享读锁（S）之间是兼容的，但共享读锁（S）与排他写锁（X）之间，以及排他写锁（X）之间是互斥的，也就是说读和写是串行的；  

- 在一定条件下，MyISAM 允许查询和插入并发执行，我们可以利用这一点来解决应用中对同一表查询和插入的锁争用问题；

- MyISAM 默认的锁调度机制是写优先，这并不一定适合所有应用，用户可以通过设置 `LOW_PRIORITY_UPDATES` 参数，或在 `INSERT、UPDATE、DELETE` 语句中指定 `LOW_PRIORITY` 选项来调节读写锁的争用。 

- 由于表锁的锁定粒度大，读写之间又是串行的，因此，如果更新操作较多，MyISAM 表可能会出现严重的锁等待，可以考虑采用 InnoDB 引擎来减少锁冲突。

**对于InnoDB表，本文主要讨论了以下几项内容：** 

- InnoDB 的行锁是基于索引实现的，如果不通过索引访问数据，InnoDB 会使用表锁；
- 在不同的隔离级别下，InnoDB 的锁机制和一致性读策略不同；

在了解 InnoDB 锁特性后，用户可以通过设计和 SQL 调整等措施减少锁冲突和死锁，包括：

- 尽量使用较低的隔离级别；精心设计索引，并尽量使用索引访问数据，使加锁更精确，从而减少锁冲突的机会；

- 选择合理的事务大小，小事务发生锁冲突的几率也更小；

- 给记录集显式加锁时，最好一次性请求足够级别的锁。比如要修改数据的话，最好直接申请排他锁，而不是先申请共享锁，修改时再请求排他锁，这样容易产生死锁；

- 不同的程序访问一组表时，应尽量约定以相同的顺序访问各表，对一个表而言，尽可能以固定的顺序存取表中的行。这样可以大大减少死锁的机会；

- 尽量用相等条件访问数据，这样可以避免间隙锁对并发插入的影响；不要申请超过实际需要的锁级别；除非必须，查询时不要显示加锁；

- 对于一些特定的事务，可以使用表锁来提高处理速度或减少死锁的可能。
