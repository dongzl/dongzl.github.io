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
