---
title: （进行中）Redis 集群高可用和数据持久化
date: 2023-04-15 13:39:53
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/redis_ha_dp.png

# author information, multiple authors are set to array
# single author
author:
- nick: BB8 StaffEngineer
  link: https://medium.com/@bb8s
- nick: 董宗磊
  link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文将深入探索 Redis 集群如何实现高可用和数据持久化。

categories:
- 架构设计

tags:
- Redis
---

> 原文链接：https://medium.com/@bb8s/how-redis-cluster-achieves-high-availability-and-data-persistence-8cdc899764e8

对于 Redis 集群等有状态分布式系统而言，系统的高可靠性需要有两个内在要求：

- **持久性**：也就是即使集群中的某个实例发生故障也不会丢失数据。`Redis` 集群提供两种持久化机制：`RDB（Redis Database）` 时间点快照和 `AOF（Append Only File）` 日志来保证数据的持久性；
- **高可用性**：尽管出现网络故障、机器故障等，`Redis` 服务仍然保持可用。

首先让我们研究一下 `Redis` 集群如何实现持久化，这是在每个实例级别发生的。

## Redis 持久化

`Redis` 通常被广泛用作基于内存的 `KV` 缓存。如果某个 `Redis` 实例在生产环境中出现不可用，通常意味着该节点上的所有内存数据都会丢失。一种直接的解决方案是允许事故发生，并依靠底层数据库（如 `MySQL`）继续提供数据，保证服务可用；虽然这种方法在理论上是可以的，但是在实际生产环境中却很少被采用。缓存的意外故障会导致数据库的流量突然变大，很容易使数据库过载。此外，即使数据库可以处理突然的过高负载，但是来自数据库的数据请求的读取延迟通常比从缓存读取要高很多。对于大多数业务场景来说，这种延迟增加几乎是不可接受的。在 `Redis` 缓存层的持久化特性，可以大大减少直接访问数据库的场景。

`Redis` 提供了两种持久化机制：

- `AOF` 日志；
- `RDB` 快照。

## 什么是 AOF 日志

我们知道 `MySQL` 等关系型数据库通常使用预写日志（`WAL`）来避免发生故障时的数据丢失。写操作数据首先追加到一个 `WAL`，然后在应用于内存缓冲区和磁盘。`Redis` 使用了不同的方案。在 `Redis` 中，写入操作首先应用于内存中的数据存储，只有当此步操作成功时，该操作才会持久化到日志中，也称为后写日志 (`WBL`)；如果写入内存数据存储失败，`Redis` 将不会写入日志，直接向客户端返回错误。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/10-How-Redis-Cluster-Achieves-High-Availability-Data-Persistence/01.webp" style="width:100%"/>

使用 `WBL` 的优势是：

- 不会产生验证数据更改（写入）请求的开销。来自客户端的写请求可能是无效的并且永远不会成功。因此始终需要请求验证或健全性检查。在 WBL 的情况下，写入内存步骤用于此验证目的，因此在写入日志时不需要额外的健全性检查。