---
title: Redis 面试常见问题知识点总结
date: 2020-03-22 21:24:37
cover: https://gitee.com/dongzl/article-images/raw/master/cover/redis_study.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨总结在面试中 Redis 的缓存雪崩、缓存穿透、缓存击穿等问题涉及的知识内容。

categories: 
  - 架构设计

tags: 
  - Redis
---

## 背景描述

在面试过程中，`缓存雪崩`、`缓存穿透`、`缓存击穿`、`分布式锁` 等问题是 Redis 的常见问题，本文根据 [马士兵教育](http://www.mashibing.com/) 公开课内容整理而成，主要总结了上述面试题一些回答思路。

## 总结内容

### 缓存雪崩

Redis 中的热点数据集中过期导致 MySQL 在某一时刻承受很大的压力，也有可能是因为 Redis 服务器宕机所致。

- 数据集中过期：在数据过期时间后加上一个随机值，不要让数据同时过期；
  
- Redis 宕机问题：a、设置多级缓存，b、搭建 Redis 集群，防止单点问题。

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/18-Redis-Interview-knowledge/Redis-Interview-knowledge-01.jpg">

### 缓存穿透

如果客户端发送的请求，查询的数据，在 Redis 中都查不到，那么缓存就失去意义了。这种情况多数是由恶意用户伪造请求参数，导致 Redis 缓存查不到数据而失效。

- BloomFilter（布隆过滤器）；
- 使用分布式锁解决缓存穿透，对于缓存中不存在的数据，在访问 MySQL 数据库时需要抢占分布式锁。

### 缓存击穿

Redis 中有一条热点数据，过期之后，MySQL 承接了大量的请求。

一般这种情况在实际工作中出现较少，很少会只有一条热点数据。

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/18-Redis-Interview-knowledge/Redis-Interview-knowledge-03.jpg">

### 分布式锁

Redis 实现分布式锁的一些问题：

- 死锁 --> 有过期时间 --> 乱入锁 --> 增大有效期时间 --> 效率低，吞吐量下降；

- 资源浪费。

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/18-Redis-Interview-knowledge/Redis-Interview-knowledge-02.png">

## 参考资料

- [阿里一面：关于【缓存穿透、缓存击穿、缓存雪崩、热点数据失效】问题的解决方案](https://juejin.im/post/5c9a67ac6fb9a070cb24bf34)