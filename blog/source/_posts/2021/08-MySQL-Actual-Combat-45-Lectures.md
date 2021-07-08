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

- 
- 