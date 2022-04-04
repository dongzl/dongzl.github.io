---
title: MySQL 主从复制原理及配置
date: 2020-03-15 11:04:01
cover: https://gitee.com/dongzl/article-images/raw/master/cover/mysql_study.png
# author information, multiple authors are set to array
# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨在总结 MySQL 数据库主从复制原理及配置。

categories: 
  - 数据库
tags: 
  - MySQL
  - 主从复制
---

## MySQL 主从复制原理

### 什么是 MySQL 的主从复制

MySQL 主从复制是指数据可以从一个 MySQL 数据库服务器主节点复制到一个或多个从节点。MySQL 默认采用异步复制方式，这样从节点不用一直访问主服务器来更新自己的数据，数据的更新可以在远程连接上进行，从节点可以复制主数据库中的所有数据库或者特定的数据库，或者特定的表。

### 为什么需要主从复制

- 提高数据库读写性能，提升系统吞吐量

在业务复杂的系统中，如果有一条 SQL 语句的执行需要锁表，导致 MySQL 暂时不能提供读的服务，那么就很影响运行中的业务，使用主从复制，让主库负责写，从库负责读，这样即使主库出现了锁表的情景，通过读从库也可以保证业务的正常运作。

- 做数据库热备

- 架构扩展需要

业务量越来越大，I/O 访问频率过高，单机无法满足，此时做多库的存储，降低磁盘I/O 访问的频率，提升整个数据库性能。

### MySQL 的复制原理

**原理**：

- master 服务器将数据的改变记录二进制 binlog 日志，当 master 上的数据发生改变时，则将其改变写入二进制日志中；

- slave 服务器会在一定时间间隔内对 master 二进制日志进行探测其是否发生改变，如果发生改变，则开始一个 I/OThread 请求 master 二进制事件；

- 同时主节点为每个 I/O 线程启动一个 dump 线程，用于向其发送二进制事件，并保存至从节点本地的中继日志中，从节点将启动 SQL 线程从中继日志中读取二进制日志，在本地重放，使得其数据和主节点的保持一致，最后 I/OThread 和 SQLThread 将进入睡眠状态，等待下一次被唤醒。

**也就是**：
- 从库会生成两个线程,一个 I/O 线程,一个 SQL 线程；

- I/O 线程会去请求主库的 binlog，并将得到的 binlog 写到本地的 relay-log（中继日志）文件中；主库会生成一个 log dump 线程,用来给从库 I/O 线程传 binlog；

- SQL 线程，会读取 relay log 文件中的日志，并解析成sql语句逐一执行。

**注意**：
- master 将操作语句记录到 binlog 日志中，然后授予 slave 远程连接的权限（master 一定要开启 binlog 二进制日志功能；通常为了数据安全考虑，slave 也开启binlog功能）；

- slave 开启两个线程：IO 线程和 SQL 线程。其中：IO 线程负责读取 master 的 binlog 内容到中继日志 relay log 里；SQL 线程负责从 relay log 日志里读出 binlog 内容，并更新到 slave 的数据库里，这样就能保证 slave 数据和 master 数据保持一致了；

- MySQL 复制至少需要两个 MySQL 的服务，当然 MySQL 服务可以分布在不同的服务器上，也可以在一台服务器上启动多个服务；

- MySQL复制最好确保 master 和 slave 服务器上的 MySQL 版本相同（如果不能满足版本一致，那么要保证 master 主节点的版本低于 slave 从节点的版本）；

- master 和 slave 两节点间时间需同步。

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/12-MySQL-Master-Slave-Replication/MySQL-Master-Slave-Replication-01.jpeg">

**具体步骤**：

- 从库通过手工执行 change master to 语句连接主库，提供了连接的用户一切条件（user 、password、port、ip），并且让从库知道，二进制日志的起点位置（file名 position 号）start slave；

- 从库的 IO 线程和主库的 dump 线程建立连接；

- 从库根据 change master to 语句提供的 file 名和 position 号，IO 线程向主库发起 binlog 的请求；

- 主库 dump 线程根据从库的请求，将本地 binlog 以 events 的方式发给从库IO 线程；

- 从库 IO 线程接收 binlog events，并存放到本地 relay-log 中，传送过来的信息，会记录到 `master.info` 中；

- 从库 SQL 线程应用 relay-log，并且把应用过的记录到 `relay-log.info` 中，默认情况下，已经应用过的 relay 会自动被清理 purge。

### MySQL 主从复制的形式

- 一主一从

- 主主复制

- 一主多从

- 多主一从

- 级联复制

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/12-MySQL-Master-Slave-Replication/MySQL-Master-Slave-Replication-02.png">

### MySQL 主从复制延时分析

MySQL 的主从复制都是单线程的操作，主库对所有 DDL 和 DML 产生的日志写进 binlog，由于 binlog 是顺序写，所以效率很高，slave 的 SQL thread 线程将主库的 DDL 和 DML 操作事件在 slave 中重放。DML 和 DDL 的 IO 操作是随机的，不是顺序，所以成本要高很多，另一方面，由于 SQL thread 也是单线程的，当主库的并发较高时，产生的 DML 数量超过 slave 的 SQL thread 所能处理的速度，或者当 slave 中有大型 query 语句产生了锁等待，那么延时就产生了。

**解决方案**：

- 业务的持久层实现采用分库架构，mysql 服务可以水平扩展，分散压力；
- 单个库读写分离，一主多从，主写从读，分散压力；这样从库压力可能会比主库高，保护主库。
- 服务的基础架构在业务系统和mysql之间加入memcache或者redis 的cache层，降低mysql读压力。
- 不同业务的mysql物理上放在不同的机器，分散压力。
- 使用比主库更好的硬件设备作为slave，mysql压力小，延迟自然会变小。
- 使用更加强劲的硬件设备。

## MySQL 主从复制安装配置

### 基础设置准备

```shell
#操作系统：
centos6.5
#mysql版本：
5.7
#两台虚拟机：
node1:192.168.85.111（主）
node2:192.168.85.112（从）
```

### 安装 MySQL 数据库

```shell
```

### 在两台数据库中分别创建数据库

```sql
--注意两台必须全部执行
create database test;
```

### 在主（node1）服务器进行如下配置：

```shell
#修改配置文件，执行以下命令打开mysql配置文件
vi /etc/my.cnf
#在mysqld模块中添加如下配置信息
log-bin=master-bin #二进制文件名称
binlog-format=ROW  #二进制日志格式，有row、statement、mixed三种格式，row指的是把改变的内容复制过去，而不是把命令在从服务器上执行一遍，statement指的是在主服务器上执行的SQL语句，在从服务器上执行同样的语句。MySQL默认采用基于语句的复制，效率比较高。mixed指的是默认采用基于语句的复制，一旦发现基于语句的无法精确的复制时，就会采用基于行的复制。
server-id=1		   #要求各个服务器的id必须不一样
binlog-do-db=test   #同步的数据库名称
```

### 配置从服务器登录主服务器的账号授权

```sql
--授权操作
set global validate_password_policy=0;
set global validate_password_length=1;
grant replication slave on *.* to 'root'@'%' identified by '123456';
--刷新权限
flush privileges;
```

### 从服务器的配置

```shell
#修改配置文件，执行以下命令打开mysql配置文件
vi /etc/my.cnf
#在mysqld模块中添加如下配置信息
log-bin=master-bin	#二进制文件的名称
binlog-format=ROW	#二进制文件的格式
server-id=2			#服务器的id
```

### 重启主服务器的mysqld服务

```shell
#重启mysql服务
service mysqld restart
#登录mysql数据库
mysql -uroot -p
#查看master的状态
show master status；
```

### 重启从服务器并进行相关配置

```shell
#重启mysql服务
service mysqld restart
#登录mysql
mysql -uroot -p
#连接主服务器
change master to master_host='192.168.85.11',master_user='root',master_password='123456',master_port=3306,master_log_file='master-bin.000001',master_log_pos=154;
#启动slave
start slave
#查看slave的状态
show slave status\G(注意没有分号)
```

> 内容来源：[马士兵教育](http://www.mashibing.com/) 公开课内容知识整理

