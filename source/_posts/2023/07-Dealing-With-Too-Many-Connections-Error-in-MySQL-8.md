---
title: MySQL 8 中如何处理 Too Many Connections 错误
date: 2023-03-18 20:03:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/mysql_study.png

# author information, multiple authors are set to array
# single author
author:
- nick: Michael Villegas
  link: https://www.percona.com/blog/author/michael-villegas/

# post subtitle in your index page
subtitle: 本文将用示例演示在 MySQL 8 中如何处理 Too Many Connections 错误。

categories:
- 数据库

tags:
- MySQL
---

> 原文链接：https://www.percona.com/blog/dealing-with-too-many-connections-error-in-mysql-8/

在做 `DBA` 的这些年里，我处理过各种各样的数据库问题。我遇到的最常见问题之一就是众所周知的错误——`ERROR 1040 (08004): Too many connections` 相关的一些问题；关于这个错误已经有很多文章了，尽管如此用户还是不断掉入这个问题的陷阱，这可能是因为数据库配置不当、应用程序组件发生变化，或者只是因为应用程序中的连接突然增加所导致的。在某些时候，我们可能都会在工作中遇到这个问题，而且可能还不止一次遇到。这篇文章的主要目的是研究 `MySQL 8` 新提供的 `administrative connections` 特性，使用这些管理连接可以在出现这个问题时无需重启数据库实例。

## 默认行为

我们知道数据库中允许创建的连接数是由 `max_connections` 参数定义的，这个参数的默认值是 `151`，并且支持动态调整，这意味着无需重启数据库实例就可以调整连接数。如果数据库中的连接数达到最大值，我们将看到这个糟糕的错误——`ERROR 1040 (08004): Too many connections`。重要的是还要记住一个开箱即用功能，`MySQL` 允许创建一个额外的连接，这个连接是为 `SUPER` 权限（已废弃[这里](https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html#priv_super)）或 `CONNECTION_ADMIN` 权限的用户保留的。

我准备用一个示例演示这个功能，对于这个示例需要创建一个 `max_connections=20` 的数据库实例，并创建三个用户，用户 `monitor1` 只有 `PROCESS` 权限，用户 `admin1` 拥有 `PROCESS` 和 `CONNECTION_ADMIN` 权限，最后一个用户 `admin2` 拥有超级用户权限（已废弃）。我们将演示 `MySQL` 在用户连接数达到最大值的情况下如何处理这些连接：

```sql
-- execute all 20 concurrent connections
sysbench oltp_read_write --table-size=1000000 --db-driver=mysql --mysql-host=localhost --mysql-db=sbtest --mysql-user=root --mysql-password="***" --num-threads=20 --time=0 --report-interval=1 run
-- test with user monitor1 
[root@rocky-test1 ~]# mysql -u monitor1 -p
Enter password:
ERROR 1040 (08004): Too many connections

-- test with user admin1
[root@rocky-test1 ~]# mysql -u admin1 -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or g.
Your MySQL connection id is 144
Server version: 8.0.29-21 Percona Server (GPL), Release 21, Revision c59f87d2854

Copyright (c) 2009-2022 Percona LLC and/or its affiliates
Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or 'h' for help. Type 'c' to clear the current input statement.

mysql> show grants;
+-----------------------------------------------+
| Grants for admin1@%                           |
+-----------------------------------------------+
| GRANT PROCESS ON *.* TO `admin1`@`%`          |
| GRANT CONNECTION_ADMIN ON *.* TO `admin1`@`%` |
+-----------------------------------------------+
2 rows in set (0.00 sec)

mysql> select count(1) from information_schema.processlist;
+----------+
| count(1) |
+----------+
|       22 |
+----------+
1 row in set (0.00 sec)


-- test with user admin2 
[root@rocky-test1 ~]# mysql -u admin2 -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or g.
Your MySQL connection id is 145
Server version: 8.0.29-21 Percona Server (GPL), Release 21, Revision c59f87d2854

Copyright (c) 2009-2022 Percona LLC and/or its affiliates
Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or 'h' for help. Type 'c' to clear the current input statement.

mysql> show grants;
+------------------------------------+
| Grants for admin2@%                |
+------------------------------------+
| GRANT SUPER ON *.* TO `admin2`@`%` |
+------------------------------------+
1 row in set (0.00 sec)

mysql> select count(1) from information_schema.processlist;
+----------+
| count(1) |
+----------+
|        1 |
+----------+
1 row in set (0.00 sec)
```

正如我们所演示的，`MySQL` 允许拥有 `CONNECTION_ADMIN` 或者 `SUPER` 权限的用户进行连接；所以当用户 `monitor1` 尝试进行连接时是不允许的，因为它没有被授予这些权限。一旦我们获得了对数据库的访问权限，我们就可以通过在线更改参数 `max_connections` 轻松增加连接数量，然后再继续排查导致连接数量不够问题的根本原因。重要的是要记住只是授予这些权限的用户连接才可以执行这个操作，所以不要轻易将这些权限授予某个用户，否则我们仍然可能被锁定在数据库之外。

```sql
– trying a second connection with user admin1

[root@rocky-test1 ~]# mysql -u admin1 -p
Enter password:
ERROR 1040 (HY000): Too many connections
```

通常当出现这个问题时，我们是无法访问 `MySQL` 数据库的，紧急的处理方式是重启数据库实例并处理由此所导致的一些问题，但是嘿......这会导致业务系统在正常运行时有几分钟时间无法连接到数据库；还有另一种获取数据库访问权限的方法，即使用 `GDB` 工具，但这个办法并一定是可行的，[Too many connections? No problem!](https://www.percona.com/blog/too-many-connections-no-problem/)是我过去写的一篇关于这个工具的文章，文章有点久了但这个办法仍然有效。

## Percona Server 为 MySQL 和 MariaDB 边注说明

`Percona Server for MySQL` 在 `MySQL 8.0.14` 之前的版本中，有另一种访问数据库实例的方法，与 `MySQL 8.0.14` 版本中引入的新功能类似，它是通过启用变量 [extra_port](https://docs.percona.com/percona-server/8.0/performance/threadpool.html#extra_port) 和 [extra_max_connections](https://docs.percona.com/percona-server/8.0/performance/threadpool.html#extra_max_connections) 来实现的，这些变量的使用超出了这篇博文的范围，这些变量的目的同样是在数据库的最大连接数已经达到上限的情况下允许继续连接到数据库。不过需要记住，这些变量已在 `MySQL 8.0.14` 版本中删除，如果在配置文件中出现这些变量，`MySQL` 数据库实例将无法正常启动并显示错误信息。与 `Percona Server for MySQL` 一样，`MariaDB` 数据库对这些变量也有类似的实现。`MariaDB` 的文档可以在[这里](https://mariadb.com/kb/en/thread-pool-system-status-variables/#extra_port)找到。

## 新特性

从 `MySQL 8.0.14` 版本开始，引入了新的 `Administrative Connections` 或 `Administrative Network Interface` 特性。这个特性允许通过管理端口连接到数据库，并且对管理连接的数量没有限制。这个特性与上面示例中显示的单个连接之间的区别在于，这个特性是通过一个不同的端口，并且它不会限制只有一个连接，而是在需要时允许创建多个连接；这个特性使我们在用户连接达到上限时还可以继续访问数据库，从而可以增加连接数量或终止一些应用程序的连接。

启用 `Administrative Connections` 最简单的方式是设置 `admin_address` 参数，这时管理连接将会监听的 `IP` 地址；例如，如果我们只允许本地连接，我们可以将这个参数设置为 `127.0.0.1`，或者如果我们想通过网络进行连接，我们可以将这个变参数定义为服务器的 `IP` 地址。这个参数不能够动态设置，这意味着修改这个参数需要重启数据库；默认情况下，此参数值为空，表示禁用管理连接功能。另一个相关参数是 `admin_port`，此参数定义 `MySQL` 为管理连接所提供的监听端口，此参数的默认值为 `33062`，定义这两个参数并重启数据库实例后，我们将会在错误日志中看到一条消息，显示管理接已准备好并等待连接：

```sql
2023-02-28T14:42:44.383663Z 0 [System] [MY-013292] [Server] Admin interface ready for connections, address: '127.0.0.1'  port: 33062
```

现在管理接口已配置就绪，我们需要定义可以访问此管理连接的用户。这些用户将需要拥有 `SERVICE_CONNECTION_ADMIN` 权限；否则将没有权限进行连接。按照我们的初始示例，我已将 `SERVICE_CONNECTION_ADMIN` 授予用户 `admin1` 但未授予用户 `admin2`。

```sql
mysql> show grants for admin1;
+------------------------------------------------------------------------+
| Grants for admin1@%                                                    |
+------------------------------------------------------------------------+
| GRANT PROCESS ON *.* TO `admin1`@`%`                                   |
| GRANT CONNECTION_ADMIN,SERVICE_CONNECTION_ADMIN ON *.* TO `admin1`@`%` |
+------------------------------------------------------------------------+
2 rows in set (0.00 sec)

mysql> show grants for admin2;
+------------------------------------+
| Grants for admin2@%                |
+------------------------------------+
| GRANT SUPER ON *.* TO `admin2`@`%` |
+------------------------------------+
1 row in set (0.00 sec)
```

测试与管理接口的连接，我们看到只允许用户 `admin1` 进行连接，而用户 `admin2` 因没有 `SERVICE_CONNECTION_ADMIN` 权限而被拒绝。此外，我们可以确认用户 `admin1` 用户连接到了 `33062` 端口，这是用于管理连接特性的端口。

```sql
-- testing user admin1

[root@rocky-test1 ~]# mysql -h 127.0.0.1 -P 33062 -u admin1 -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or g.
Your MySQL connection id is 23
Server version: 8.0.29-21 Percona Server (GPL), Release 21, Revision c59f87d2854

Copyright (c) 2009-2022 Percona LLC and/or its affiliates
Copyright (c) 2000, 2022, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or 'h' for help. Type 'c' to clear the current input statement.

mysql> s
--------------
mysql  Ver 8.0.29-21 for Linux on x86_64 (Percona Server (GPL), Release 21, Revision c59f87d2854)

Connection id:		23
Current database:
Current user:		admin1@localhost
SSL:			Cipher in use is TLS_AES_256_GCM_SHA384
Current pager:		stdout
Using outfile:		''
Using delimiter:	;
Server version:		8.0.29-21 Percona Server (GPL), Release 21, Revision c59f87d2854
Protocol version:	10
Connection:		127.0.0.1 via TCP/IP
Server characterset:	utf8mb4
Db     characterset:	utf8mb4
Client characterset:	utf8mb4
Conn.  characterset:	utf8mb4
<strong>TCP port:		33062</strong>
Binary data as:		Hexadecimal
Uptime:			50 min 27 sec

Threads: 3  Questions: 188  Slow queries: 0  Opens: 335  Flush tables: 3  Open tables: 269  Queries per second avg: 0.062
--------------

-- testing user admin2

[root@rocky-test1 ~]# mysql -h 127.0.0.1 -P 33062 -u admin2 -p
Enter password:
<strong>ERROR 1227 (42000): Access denied; you need (at least one of) the SERVICE_CONNECTION_ADMIN privilege(s) for this operation</strong>
```

## 总结

如果我们使用的是 `MySQL 8.0.14` 或更高版本，我们应用启用管理连接功能；正如示例所见开启这个能非常简单，使用这个新特性在触发 `ERROR 1040 (08004): Too many connections` 错误时允许 `DBA` 继续访问数据库。这个新特性不会影响正常的数据库性能，但是可以对 `DBA` 工作发挥很大作用。

我们应当考虑仅向管理员用户添加 `SERVICE_CONNECTION_ADMIN` 权限，而不是普通应用程序用户，这样做的目的是不要滥用此功能。如果我们还在使用较低版本的 `Percona Server for MySQL` 时，如果遇到最大连接问题，我们可以配置参数 `extra_port` 和 `extra_max_connections` 来访问数据库。

<hr />

# Too many connections? No problem!

> https://www.percona.com/blog/too-many-connections-no-problem/

你在生产中遇到过这种情况吗？

```sql
[percona@sandbox msb_5_0_87]$ ./use
ERROR 1040 (00000): Too many connections
```

刚刚发生在我们的一位客户身上。想知道我们是怎么处理的吗？

出于演示目的，我在此处使用沙箱环境（因为 `./use` 实际上正在执行 `MySQL CLI` 工具）。请注意这不是通用的最佳实践，而是类似于服务器恶意闯入式的黑客攻击。因此当这种情况发生在生产环境中时，我们需要处理的问题是：

- 如何快速重新获得对 `MySQL` 服务器的访问权限以便查看所有会话都在执行什么操作；
- 如何能够不重启应用程序的情况下连接到 `MySQL` 服务器。

下面方法是一个小窍门：

```sql
[percona@sandbox msb_5_0_87]$ gdb -p $(cat data/mysql_sandbox5087.pid) \
                                     -ex "set max_connections=5000" -batch
[Thread debugging using libthread_db enabled]
[New Thread 0x2ad3fe33b5c0 (LWP 1809)]
[New Thread 0x4ed19940 (LWP 27302)]
[New Thread 0x41a8b940 (LWP 27203)]
...
[New Thread 0x42ec5940 (LWP 1813)]
[New Thread 0x424c4940 (LWP 1812)]
0x00000035f36cc4c2 in select () from /lib64/libc.so.6
```

结果如下：

```sql
[percona@test9 msb_5_0_87]$ ./use 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 5.0.87-percona-highperf-log MySQL Percona High Performance Edition, Revision 61 (GPL)

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql [localhost] {msandbox} ((none)) > select @@global.max_connections;
+--------------------------+
| @@global.max_connections |
+--------------------------+
|                     5000 | 
+--------------------------+
1 row in set (0.00 sec)
```

[gdb magic](https://dom.as/tag/gdb/) 的能力需要归功于 [Domas](https://dom.as/)。

**几点注意事项**：
- 我们通常会为 `SUPER` 权限用户保留一个连接，但如果应用程序正在使用 `SUPER` 用户身份连接数据库（无论如何这不是一个好主意），这个方法就行不通了；
- 这个办法在 `5.0.87-percona-highperf` 上是有效的，但使用这个方案需要承担一些风险，并且在实际生产环境中使用之前需要进行充分测试；
- 这个示例假设配置的 `max_connections` 数量小于 `5000`。
