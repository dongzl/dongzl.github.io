---
title: Redis 知识思维导图总结
date: 2021-07-05 20:17:16
cover: https://gitee.com/dongzl/article-images/raw/master/cover/redis_study.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨在对 Redis 知识内容进行梳理总结，然后根据思维导图逐步深入学习，最终达到精通 Redis 地步。

categories: 
  - 架构设计

tags: 
  - Redis
---

## 背景介绍

最近在极客时间上学习了[《Redis核心技术与实战》](https://time.geekbang.org/column/intro/100056701)这门课程，真的意识到虽然使用 Redis 好多年了（在京东使用的 KV 缓存系统叫做 JimDB），但是仅仅停留在了使用阶段，至于说用熟用好完全谈不上，所以结合这门课程还有相关的书籍，准备系统学习一下 Redis，现在还是使用入门水平，学习的目标就是落到简历上达到可以自信的写到”精通 Redis“，虽然目标很功利，但是只要能实现目标，功利小手段也是可以的。

## 思维导图

<img src="https://gitee.com/dongzl/article-images/raw/master/2021/07-Redis-Knowledge-Summary/Redis-Knowledge-Summary-01.png" style="width:400px"/>

## 知识汇总

### 数据类型

#### 基础数据类型

- string

- list

- hash

- set

- sorted set

对于 Redis 中五种基本类型数据结构，需要掌握每种数据类型的特征、使用场景以及底层的存储结构。

#### 高级数据类型

- HyperLogLog

- Geo

- Bitmaps

对于 Redis 中三种高级数据类型结构，需要掌握每种数据类型的使用场景以及该特定使用场景下的优势。

### 持久化

- RDB

- AOF

系统梳理 Redis 中 RDB 和 AOF 两种数据持久化方式的特征和使用方式。

### Redis 高可用架构

- Redis 主从复制

- Redis Cluster

- Redis Sentinel

系统学习 Redis 中高可用架构的使用场景、实现原理以及在使用中需要注意的问题。

### Redis 优化

- 故障诊断

- 内存优化

系统学习 Redis 中故障诊断、内存优化相关知识内容，这一部分是用好 Redis 的关键。

### 其他知识学习

- Redis 事务机制

- Redis 发布订阅

- Redis stream

- Redis 脚本

- Redis 管道技术

掌握每种技术的使用场景、使用特征，通过与其他技术框架类比的方式分析 Redis 实现这些特征的优势以及不足。

## 总结

这篇文章算是一个流水账的开篇，算是自己在这里立下一个 flag，通过一段时间的学习，彻底搞定 Redis，达到简历上”精通“水平，做到对 Redis 知识 `360°` 无死角。