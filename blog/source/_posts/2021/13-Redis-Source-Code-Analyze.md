---
title: 《Redis 源码剖析与实战》学习笔记
date: 2021-07-27 20:35:25
cover: https://gitee.com/dongzl/article-images/raw/master/cover/redis_study.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文是极客时间《Redis 源码剖析与实战》课程学习笔记整理。

categories: 
  - 架构设计

tags: 
  - Redis
---

## 开篇词 | 阅读Redis源码能给你带来什么？

阅读源码带来的受益：

- 第一，从原理到源码，学习源码阅读方法，培养源码习惯，掌握学习主动权。
- 第二，学习良好的编程规范和技巧，写出高质量的代码。
- 第三，举一反三，学习计算机系统设计思想，实现职业能力进阶。

<img src="https://static001.geekbang.org/resource/image/d4/8f/d4d4dea857ee82360a0bb5f4cb847f8f.jpg?wh=2000x1125" style="width:500px"/>

高效阅读代码的要点：

- 先从整体上掌握源码的结构。
- 一定要有目标牵引和原理支撑。
- 要做到先主线逻辑再分支细节。

<img src="https://static001.geekbang.org/resource/image/59/35/5975c57d9ac404fe3a774ea28a7ac935.jpg?wh=2238x811" style="width:500px"/>

## 01 | 带你快速攻略Redis源码的整体架构

Redis 源码学习需要掌握的两方面内容：

- **代码的目录结构和目录划分**，目的是理解 `Redis` 代码的整体架构，以及所包含的代码功能类别；

- **系统功能模块与对应代码文件**，目的是了解 `Redis` 实例提供的各项功能及其相应的实现文件，以便后续深入学习。

### Redis 目录结构

- deps 目录：Redis 依赖的第三方代码库；

<img src="https://static001.geekbang.org/resource/image/42/c7/4278463fb96f165bf41d6a97ff3216c7.jpg?wh=1945x726" style="width:500px"/>

- src 目录：Redis 所有功能模块的代码文件，也是 Redis 源码的重要组成部分；

<img src="https://static001.geekbang.org/resource/image/d7/26/d7ac6b01af49047409db5d9e16b6e826.jpg?wh=2187x487" style="width:500px"/>

- tests 目录：单元测试（unit 子目录），Redis Cluster 功能测试（cluster 子目录）、哨兵功能测试（sentinel 子目录）、主从复制功能测试（integration 子目录）；

<img src="https://static001.geekbang.org/resource/image/cc/5e/ccb2feae193e4911cc68a0ccb755ac5e.jpg?wh=2250x1111" style="width:500px"/>

- utils 目录：

<img src="https://static001.geekbang.org/resource/image/3b/b2/3b7933e5f1740ccdc3870ee554faf4b2.jpg?wh=2250x1039" style="width:500px"/>

### Redis 源码全景图

<img src="https://static001.geekbang.org/resource/image/59/35/5975c57d9ac404fe3a774ea28a7ac935.jpg?wh=2238x811" style="width:500px"/>

### 服务器实例代码结构

<img src="https://static001.geekbang.org/resource/image/51/df/514e63ce6947d382fe7a3152c1c989df.jpg?wh=2250x882" style="width:500px"/>

### 数据库数据类型与操作

<img src="https://static001.geekbang.org/resource/image/0b/57/0be4769a748a22dae5760220d9c05057.jpg?wh=2000x1125" style="width:500px"/>

<img src="https://static001.geekbang.org/resource/image/15/f0/158fa224d6a49c7d4702ce3f07dbeff0.jpg?wh=1938x768" style="width:500px"/>

### 高可靠性和高可扩展性

- 数据持久化实现：`rdb.h/rdb.c` 和 `aof.c`;

- 主从复制功能实现：`replication.c`。

### 总结

**数据类型：**
- String（t_string.c、sds.c、bitops.c）
- List（t_list.c、ziplist.c）
- Hash（t_hash.c、ziplist.c、dict.c）
- Set（t_set.c、intset.c）
- Sorted Set（t_zset.c、ziplist.c、dict.c）
- HyperLogLog（hyperloglog.c）
- Geo（geo.c、geohash.c、geohash_helper.c）
- Stream（t_stream.c、rax.c、listpack.c）

**全局：**
- Server（server.c、anet.c）
- Object（object.c）
- 键值对（db.c）
- 事件驱动（ae.c、ae_epoll.c、ae_kqueue.c、ae_evport.c、ae_select.c、networking.c）
- 内存回收（expire.c、lazyfree.c）
- 数据替换（evict.c）
- 后台线程（bio.c）
- 事务（multi.c）
- PubSub（pubsub.c）
- 内存分配（zmalloc.c）
- 双向链表（adlist.c）

**高可用&集群：**
- 持久化：RDB（rdb.c、redis-check-rdb.c)、AOF（aof.c、redis-check-aof.c）
- 主从复制（replication.c）
- 哨兵（sentinel.c）
- 集群（cluster.c）

**辅助功能：**
- 延迟统计（latency.c）
- 慢日志（slowlog.c）
- 通知（notify.c）
- 基准性能（redis-benchmark.c）

