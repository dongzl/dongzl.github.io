---
title: 使用连接控制插件实现 MySQL 安全连接
date: 2023-05-25 20:03:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/mysql_security_plugins.png

# author information, multiple authors are set to array
# single author
author:
- nick: Snehal Bhavsar
  link: https://www.percona.com/blog/author/snehal-bhavsar/
- nick: 董宗磊
  link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 解决大规模分布式系统问题的 10 个必备系统设计算法和分布式数据结构。

categories:
- 数据库

tags:
- sharding

---

> 原文链接：https://www.percona.com/blog/mysql-connection-security-with-connection-control-plugins/

作为数据库管理员，您是否遇到过数据库被暴力破解的情况？对 `MySQL` 中的用户账户发起暴力攻击。`MySQL` 根据提供验证结果返回成功或失败，这两种情况下所花费的验证时间几乎是相同的。因此，攻击者可以快速对 `MySQL` 用户账户发起暴力攻击，并可以尝试许多不同的密码。

根据密码学，暴力攻击包括攻击者尝试许多密码或密码短语，希望最终能猜对。攻击者系统地检查所有可能的密码和密码短语，直到找到正确的密码。

不仅仅是暴力攻击在进行；`IT` 行业最近发现分布式拒绝服务（`DDoS`）攻击在稳步增加。您是否也成为 `3306` 端口上此类连接流的目标？今天，我们想带你通过一个特殊的 一种插件，即 [connection_control 插件](https://dev.mysql.com/doc/refman/8.0/en/connection-control.html)！它在 `MySQL 8.0` 中引入，并向后移植到 `MySQL 5.7` 和 `MySQL 5.6`。

## 什么是连接控制插件

连接控制插件库允许管理员在指定次数的连续登录尝试失败后增加服务器对连接的响应延迟。

使用连接控制插件的想法是配置 MySQL 服务器，以便服务器延迟响应。除非服务器回复，否则未经授权的用户或客户端不知道密码是否正确。因此，如果攻击者通过生成多个连接请求来攻击服务器，则此类连接必须处于活动状态，直到服务器回复为止。引入延迟会使攻击者更难，因为现在资源被占用以确保连接请求处于活动状态。这种技术可以减缓对 MySQL 用户帐户的暴力攻击。

### 插件库包含两个插件：

- [CONNECTION_CONTROL](https://dev.mysql.com/doc/refman/5.7/en/connection-control.html)：CONNECTION_CONTROL 检查传入的连接尝试并根据需要向服务器响应添加延迟。该插件还公开了允许配置其操作的系统变量和提供基本监控信息的状态变量；
- [CONNECTION_CONTROL_FAILED_LOGIN_ATTEMPTS](https://dev.mysql.com/doc/refman/5.7/en/information-schema-connection-control-failed-login-attempts-table.html)：CONNECTION_CONTROL_FAILED_LOGIN_ATTEMPTS 实现了一个 INFORMATION_SCHEMA 表，该表公开了有关失败连接尝试的更详细的监视信息。

## 如何安装连接控制插件

要在运行时加载插件，请使用这些语句，根据需要为您的平台调整 .so 后缀。在这里，我将使用 Percona Server for MySQL 5.7.36 对其进行测试：

```sql
mysql> INSTALL PLUGIN CONNECTION_CONTROL SONAME 'connection_control.so';
INSTALL PLUGIN CONNECTION_CONTROL_FAILED_LOGIN_ATTEMPTS SONAME 'connection_control.so';
Query OK, 0 rows affected (0.01 sec)
Query OK, 0 rows affected (0.00 sec)
```

或者，您可以在 my.cnf 中安装插件。在 MySQL 配置文件 (/etc/my.cnf) 的 [mysqld] 选项组下添加这些选项：

```shell
[mysqld]

plugin-load-add=connection_control.so
connection-control=FORCE_PLUS_PERMANENT
connection-control-failed-login-attempts=FORCE_PLUS_PERMANENT
```

现在让我们更深入地了解这些配置选项中的每一个：

- `plugin-load-add=connection_control.so` – 每次启动服务器时加载 connection_control.so 库；
- `connection_control=FORCE_PLUS_PERMANENT` – 防止服务器在没有 CONNECTION_CONTROL 插件的情况下运行，如果插件未成功初始化，则服务器启动失败；
- `connection-control-failed-login-attempts=FORCE_PLUS_PERMANENT` – 防止服务器在没有 CONNECTION_CONTROL_FAILED_LOGIN_ATTEMPTS 插件的情况下运行，如果插件未成功初始化，则服务器启动失败。

要验证插件安装，请重新启动服务器并检查 INFORMATION_SCHEMA.PLUGINS 表或使用 SHOW PLUGINS 语句：

```sql
mysql> SELECT PLUGIN_NAME, PLUGIN_STATUS
      FROM INFORMATION_SCHEMA.PLUGINS
      WHERE PLUGIN_NAME LIKE 'connection%';
+------------------------------------------+---------------+
| PLUGIN_NAME                              | PLUGIN_STATUS |
+------------------------------------------+---------------+
| CONNECTION_CONTROL                       | ACTIVE        |
| CONNECTION_CONTROL_FAILED_LOGIN_ATTEMPTS | ACTIVE        |
+------------------------------------------+---------------+
```

### 配置连接控制阈值

现在，让我们使用这些服务器参数为失败的连接尝试配置服务器响应延迟。我们将连续失败连接的阈值暂定为三个，并添加至少一秒的连接延迟。

```sql
mysql> SET GLOBAL connection_control_failed_connections_threshold = 3;
SET GLOBAL connection_control_min_connection_delay = 1000;  
SET GLOBAL connection_control_min_connection_delay = 90000;
```

或者，要在运行时设置和保留变量，请使用以下语句：

```sql
mysql> SET PERSIST connection_control_failed_connections_threshold = 3;
SET PERSIST connection_control_min_connection_delay = 1000;
```

此外，您可以将这些选项添加到 MySQL 配置文件 (/etc/my.cnf) 的 [mysqld] 选项组下，以便稍后根据需要进行调整。

```shell
Shell
[mysqld]

connection_control_failed_connections_threshold=3
connection_control_min_connection_delay=1000 
connection_control_max_connection_delay=2147483647
```

让我们更详细地讨论这些变量中的每一个：

- [connection_control_failed_connections_threshold](https://dev.mysql.com/doc/refman/8.0/en/connection-control-variables.html#sysvar_connection_control_failed_connections_threshold)：在服务器为后续连接尝试添加延迟之前允许帐户的连续失败连接尝试次数；
- [connection_control_min_connection_delay](https://dev.mysql.com/doc/refman/8.0/en/connection-control-variables.html#sysvar_connection_control_min_connection_delay)：超过阈值的连接失败的最小延迟（以毫秒为单位）；
- [connection_control_max_connection_delay](https://dev.mysql.com/doc/refman/8.0/en/connection-control-variables.html#sysvar_connection_control_max_connection_delay)：超过阈值的连接失败的最大延迟（以毫秒为单位）。

## 测试和监控连接

### 第一个窗口：

请注意，这里我们将添加一个最小一秒的延迟并将失败的连接阈值设置为三。最大连接延迟设置为 90 秒。

```sql
mysql> SET GLOBAL connection_control_failed_connections_threshold = 3;
SET GLOBAL connection_control_min_connection_delay = 1000;
set global connection_control_max_connection_delay=90000;

Query OK, 0 rows affected (0.00 sec)
Query OK, 0 rows affected (0.00 sec)
Query OK, 0 rows affected (0.00 sec)

mysql> show variables like '%connection_control%';
+-------------------------------------------------+-------+
| Variable_name                                   | Value |
+-------------------------------------------------+-------+
| connection_control_failed_connections_threshold | 3     |
| connection_control_max_connection_delay         | 90000 |
| connection_control_min_connection_delay         | 1000  |
+-------------------------------------------------+-------+
3 rows in set (0.00 sec)
```

尝试获取这些变量的值：

```sql
mysql> select * from information_schema.connection_control_failed_login_attempts;
Empty set (0.00 sec)

mysql> show global status like 'connection_control_%';
+------------------------------------+-------+
| Variable_name                      | Value |
+------------------------------------+-------+
| Connection_control_delay_generated | 0     |
+------------------------------------+-------+
1 row in set (0.00 sec)
```

最初，没有失败的连接尝试，也没有产生连接延迟，因为我们在这里重新配置了参数。因此，您可以看到 information_schema.connection_control_failed_login_attempts 和零 Connection_control_delay_generated 的空集。

### 第二个窗口：

通过以上设置，我测试了 53 个假连接的暴力破解。

打开另一个终端并以 root 用户身份执行这些错误的连接，每次都指定错误的密码。

```shell
[root@ip-xxx-xx-xx-xx ~]# for i in `seq 1 53`;   do time mysql mysql  -uroot -p”try_an_incorrect_password” 
-h xxx.xx.x.x 2>&1 >/dev/null | grep meh ;   done
0.093
0.092
0.093
1.093
2.093
3.105
4.093
5.093
…
…
45.092
46.093
47.093
48.093
49.092
50.093
```

## 这些连接发生了什么？

- 在MySQL进程列表中，每个连接都会处于“Waiting in connection_control plugin”状态。
- 在第三次连接尝试后，每个连接都会经历小而明显的延迟，并且会持续增加，直到您进行最后一次尝试。对于随后的每次失败尝试，延迟都会增加一秒，直到达到最大限制。这意味着如果在 3 次登录失败后建立第 50 个连接，则第 51 个连接将花费 51 秒，第 52 个连接将再次花费 52 秒，依此类推。这意味着延迟持续增加，直到达到 connection_control_max_connection_delay。因此，自动暴力攻击工具将不再有用，因为它们将面临持续的延迟。

### 第一个窗口

现在切换回第一个终端并重新检查变量的值。

connection_control 开始监视所有失败的连接尝试并跟踪每个用户连续失败的连接尝试。

直到连续失败的尝试次数小于阈值，即在我们的例子中为 3 次，用户不会遇到任何延迟。这应该避免在用户错误输入他/她的密码的真实情况下的延迟。

在这里你可以注意到 Connection_control_delay_generated 的状态现在是 50，CONNECTION_CONTROL_FAILED_LOGIN_ATTEMPTS 是 53。

```shell
mysql> show global status like 'connection_control_%';
+------------------------------------+-------+
| Variable_name                      | Value |
+------------------------------------+-------+
| Connection_control_delay_generated | 50    |
+------------------------------------+-------+
1 row in set (0.00 sec)

mysql> SELECT FAILED_ATTEMPTS FROM INFORMATION_SCHEMA.CONNECTION_CONTROL_FAILED_LOGIN_ATTEMPTS;
+-----------------+
| FAILED_ATTEMPTS |
+-----------------+
|              53 |
+-----------------+
1 row in set (0.00 sec)
```

## 当您想成功/真正登录时会发生什么？

请注意，服务器将继续为所有后续失败连接和第一次成功连接引入此类延迟。因此，在 53 次登录失败后第一次成功登录尝试产生的延迟是 53 秒。假设 MySQL 在第一次成功连接后没有添加任何延迟，在这种情况下，攻击者将得到一个指示，延迟意味着密码错误，因此在等待特定时间后释放挂起的连接。因此，如果您在 N 次不成功尝试后尝试建立一个成功连接，您肯定会在第一次成功登录尝试时遇到 N 秒的延迟。

```shell
[root@ip-xxx-xx-xx-xx ~]# date; mysql -uroot -p’correct_password’ -hxxx.xx.x.x -e "select now();";date
Tue Apr 18 06:27:36 PM UTC 2023
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------------+
| now()               |
+---------------------+
| 2023-04-18 18:28:29 |
+---------------------+
Tue Apr 18 06:28:29 PM UTC 2023
```

## 哪个用户造成了这种暴力攻击？

您还可以确定这些失败的连接尝试来自哪个用户或主机。

```shell
mysql> select * from information_schema.connection_control_failed_login_attempts;
+-----------------------+-----------------+
| USERHOST              | FAILED_ATTEMPTS |
+-----------------------+-----------------+
| 'root'@'xxx-xx-xx-xx' |              53 |
+-----------------------+-----------------+
1 row in set (0.00 sec)
```

## 如何重置失败的连接阈值

如果您想重置这些计数器，您只需再次为变量 connection_control_failed_connections_threshold 赋值：

```shell
mysql> SET GLOBAL connection_control_failed_connections_threshold = 4;
Query OK, 0 rows affected (0.00 sec)

# Now you can see the values are reset!
mysql> select * from information_schema.connection_control_failed_login_attempts;
Empty set (0.00 sec)

mysql> show global status like 'connection_control_%';
+------------------------------------+-------+
| Variable_name                      | Value |
+------------------------------------+-------+
| Connection_control_delay_generated | 0     |
+------------------------------------+-------+
1 row in set (0.00 sec)
```

## 总结

MySQL 连接控制对于限制暴力攻击或不合时宜的 TCP 连接带来的不便非常有用。 Percona Server for MySQL 也支持这些插件。默认情况下，它未启用，但您可以执行相同的步骤来启用该插件并为您的环境设置安全连接。

总而言之，该插件提供以下功能：

- 一种配置阈值的方法，在该阈值之后触发增加的延迟
- 能够使用最小和最大限制配置服务器回复的延迟
- 一个信息模式视图，用于查看与失败的连接尝试相关的监控信息

请查看我们关于如何确保数据库安全的最新博客：

- [Keep Your Database Secure With Percona Advisors](https://www.percona.com/blog/keep-your-database-secure-with-percona-advisors/)
- [Improving MySQL Password Security with Validation Plugin](https://www.percona.com/blog/improving-mysql-password-security-with-validation-plugin/)
- [MySQL 8: Account Locking](https://www.percona.com/blog/mysql-8-account-locking/)
- [Brute-Force MySQL Password From a Hash](https://www.percona.com/blog/brute-force-mysql-password-from-a-hash/)
- [Version 2 Advisor Checks for PMM 2.28 and Newer](https://docs.percona.com/percona-monitoring-and-management/details/develop-checks/checks-v2.html?_gl=1*x9v1q5*_gcl_au*NDUzNDMxODk0LjE2ODQ5MzEzMzE.)
