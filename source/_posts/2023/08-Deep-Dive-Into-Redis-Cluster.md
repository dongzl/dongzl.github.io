---
title: 深入探索 Redis 集群：分片算法和分片架构
date: 2023-03-24 20:03:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/redis_server.png

# author information, multiple authors are set to array
# single author
author:
- nick: BB8 StaffEngineer
  link: https://medium.com/@bb8s

# post subtitle in your index page
subtitle: 本文将通过 Istio、eBPF 和 RSocket Broker 技术深入探索 Service Mesh 解决方案。

categories:
- 架构设计

tags:
- Redis
---

> 原文链接：https://medium.com/@bb8s/deep-dive-into-redis-cluster-sharding-algorithms-and-architecture-a093a92e97af

Redis 是在许多公司中被广泛使用的缓存框架。由于所有的操作都是在内存中完成，所以单个 Redis 实例可以处理百万级的读/写 QPS。除了作为类似于 memcached 一样的简单的内存缓存外，Redis 还提供了很多其它特性。例如，在许多场景中，Redis 同样可以用于持久化存储的工具，因为 Redis 内置了 AOF（append only file） 机制，可以将每一次的写操作持久化到磁盘。 AOF 有两种运行模式，可以在设置 Redis 时进行配置。

- 同步 AOF：同步 AOF 的意思是每一次写操作都需要成功持久化到磁盘后才能像客户端返回操作结果（也被称作是**阻塞写**）。在这种模式下，Redis 不再是纯粹的 `in-mem` 存储。同步 AOF 能够确保数据持久化，但是写入操作延迟会比较高，这一点是需要权衡的。在我们的生产环境中观察到同步 AOF 的单个写入操作平均需要 `1s` 完成。
- 异步 AOF：在这种模式下，写入操作是非阻塞的，这意味着 `Redis` 仍然可以被客户端视为是 `in-mem` 的，只是内存中的数据会在后台定期持久化到磁盘，这对客户端是透明的。这种模式提供了非常低的写入延迟，因为它所有的客户端操作都是在内存中完成的，但这种模式需要权衡数据丢失。如果一个 Redis 节点发生故障或者重启，写入到这个 Redis 节点但是还没有持久化到磁盘的数据就将会永久性丢失。

## Redis 集群

随着业务发展，有可能数据量、读写 QPS 很快就会超过单个 Redis 实例的容量。我们需要使用 Redis 集群来存储键值数据，在集群中，Key 被分布到多个实例。通常情况下，可以通过三种方式将 Key 分片到多个实例：

### 客户端分片

> 假设我们有一个 ad-ranking 引擎，它本质上是一种推荐服务，它从广告数据库中召回相关广告并对每个 ad_request 的广告进行排名。排名引擎需要检索每个广告的实时出价，以便进行广告拍卖，由于业务对请求延迟高度敏感，因此广告的所有实时出价都经过计算并预加载到 Redis 集群中；Ad-ranking 引擎在其二进制文件中内置了 Redis 客户端，以便于从 Redis 集群中读取数据。

在客户端分片模式中，`Redis` 客户端包括了分片和路由逻辑。换句话说，它是一个非常厚重的客户端。这种架构的优点是不依赖任何中间件。只有两方：`Redis` 客户端和 `Redis` 节点。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/08-Deep-Dive-Into-Redis-Cluster/01.png" style="width:100%"/>

### 具有集中代理（又名中间件）的服务器端分片

中间件被用作代理。来自 `ad-ranking` 引擎的请求流量将打到代理中间件。中间件代理包含分片和路由逻辑用来确定访问哪个 `Redis` 实例来检索数据。在这种情况下，`Redis` 客户端可以做地非常轻量，因为它不再包含分片和路由逻辑。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/08-Deep-Dive-Into-Redis-Cluster/02.png" style="width:100%"/>

### 分散的服务器端分片

去中心化分片是`Redis` 官方提供的集群解决方案。集群中的每个节点都在本地维护一份**路由表**的副本，并通过 `gossip` 协议交互不断更新自己的路由表。换句话说，不再有中心化的代理；恰恰相反每个 `Redis` 节点都包含充当代理所需的全部元数据信息。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/08-Deep-Dive-Into-Redis-Cluster/03.png" style="width:100%"/>

一个请求可能会打到任意一个 Redis 节点；每个节点都知道集群中的所有其他节点的网络地址、存储的密钥等。处理请求的节点首先会检查请求需要的数据是在本地节点还是在其他节点，如果数据存储在其它节点，则将请求重定向到合适的节点。

## 分片算法

### 一致性哈希

一致性哈希是一个经典的分片算法。本质上所有的内容，包括 Redis 集群中的每个节点，每一条记录的 Key，都映射到一个 `0~2³²-1`（或 `0~2⁶⁴-1`）的哈希环上。当客户端想要检索一条 k-v 记录时，它首先计算 Key 的散列，然后在哈希环上顺时针找到下一个 `Redis` 节点，并从该节点获取数据。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/08-Deep-Dive-Into-Redis-Cluster/04.png" style="width:100%"/>

当**添加/删除**节点时，一致性哈希能够确保只有相邻节点需要迁移数据，而集群中的其它大多数节点不受影响。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/08-Deep-Dive-Into-Redis-Cluster/05.png" style="width:100%"/>

一致性哈希有一个很大的缺点：数据分片不均衡或者说是数据偏斜。通常一个缓存集群可能只有几十到几百个节点。在极端情况下，假设只有 `2` 个节点，可能大部分键值数据都位于节点 `A` 上，而只有少量位于节点 `B` 上。即使对于最初分片均衡的集群，由于系统演进，例如添加/删除节点、Key 过期、添加新 Key 等原因，集群最终都可能会出现不平衡的分片。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/08-Deep-Dive-Into-Redis-Cluster/06.png" style="width:100%"/>

为了是分片尽量平衡，常用的方法包括：

- 引入微分片的概念。例如，每个物理节点映射到 `n` 个虚拟节点。如果集群有 `m` 个节点，那么总共会有 `m*n` 个虚拟节点，虚拟节点越多，数据分布在统计上就越均衡；
- 定期重新分片。例如，每个月重新计算每个分片上的键分布，并在所有节点上执行全局数据迁移，使集群恢复到分片均衡的状态。

### Redis 集群中使用的哈希槽分片

`Redis` 集群没有使用上面提到的一致性哈希。相反使用的是哈希槽，系统中的所有键被散列成 `0~16383` 的整数范围，计算公式是 `slot = CRC16(key) & 16383`。每个节点负责存储一组槽，以及与槽关联的键值对。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/08-Deep-Dive-Into-Redis-Cluster/07.png" style="width:100%"/>

本质上哈希槽是另外一层抽象。它解耦数据记录（键值对）和节点。每个节点只需要知道应该在其上存储哪些槽，槽被编码为 `bit` 数组。该数组的大小为 `16384/8 = 2048` 字节。换句话说，一个 `2048` 字节的数组包含节点的所有信息，能够确定 `16384` 个槽中的特定槽是否存储在这个节点上。例如，对于槽 `1`，节点只需要检查第二位是否为 `1`（因为槽索引从 `0` 开始），这是一个时间复杂度为 O(1) 的操作，执行非常高效。

添加一层哈希槽能够使得节点的**添加/删除**操作变得非常简单。假设我们有一个由 `3` 个节点 `A`、`B` 和 `C` 组成的集群。每个节点将存储一定范围的槽：

> Node A: slot 0–5460
> Node B: slot 5461–10922
> Node C: slot 10923–16383

- 假设我们需要向集群添加一个新的节点 `D`。在这种情况下，在 `A`、`B` 和 `C` 节点上的一些槽将会迁移到 `D` 节点。由于槽及其存储的键值记录是不可分割的，所以在跨节点迁移时是作为一个原子单元进行迁移的。
- 同样当我们从集群中移除 `A` 节点时，只需要把 `A` 中的槽迁移到 `B` 和 `C` 上即可，这样就可以安全移除节点 `A` 了。

总之，将哈希槽从一个节点移动到另一个节点不需要集群停止操作；添加和删除节点，或调整某个节点上哈希槽的百分比，不需要任何集群停机时间。

## 分片架构

### 原生 Redis 集群

原生集群的实现与上述算法完全相同。所有功能，例如**路由/分片**、**集群拓扑元数据**、**实例健康监控**等等都集成在集群中，没有其他依赖项，实例之间使用 `gossip` 协议来相互交互。

原生集群通常可以支持 `300 ~ 400` 个实例，每个实例可以处理 `8` 万 `QPS` 的读操作，集群总共可以处理 `2000 ~ 3000` 万 `QPS`。

但是，如果需要处理更高的 `QPS`，一旦集群超过 `400` 个实例，添加更多的 `Redis` 实例就不再是一个好主意。原因是随着集群中实例数量的增加，`gossip` 协议的资源占用也会迅速增加；当系统架构需要进一步扩展时，其他架构的使用更为广泛。

### Twemproxy + 原生 Redis 集群

下图展示了 `Twemproxy` 在多个 `Redis` 实例环境下工作的基本架构。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/08-Deep-Dive-Into-Redis-Cluster/08.png" style="width:100%"/>

`Twemproxy` 是前面描述具有集中代理的服务器端分片的架构示例。 `Twemproxy` 是由 `Twitter` 开源的。`Twemproxy` 可以接受来自多个客户端服务的请求，并将请求定向到底层的 `Redis` 节点等待响应，然后直接将响应结果返回给客户端。`Twemproxy` 还支持如下一些非常有用的功能：

- 自动删除故障 Redis 节点；
- 支持 Hashtag 能力。例如，如果我们想要确保一组 Key 被哈希到同一个 Redis 实例节点，我们就可以给这些 Key 设置相同的 `Hashtag`；
- 支持多种哈希算法。

`Twemproxy` 可以与原生 `Redis` 集群协同工作，如图：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/08-Deep-Dive-Into-Redis-Cluster/09.png" style="width:100%"/>

在这种情况下，`Twemproxy` 不再处理**路由/分片**，而是用于存储元数据，例如高级 `Redis` 集群拓扑、访问控制列表，并可用于监控常见的可能导致本机 `Redis` 集群出现故障的**热 Key**、**大 Key**问题等。**分片/路由**仍然由底层原生 `Redis` 集群处理。换句话说，一个 `Twemproxy` 节点维护与所有 `Redis` 节点的连接，并且可以向底层所有的 `Redis` 节点发送请求，接收请求的 `Redis` 节点将在需要时将请求重新路由到集群内的另一个 `Redis` 节点。

`AWS` `ElasticCache` 就是使用的这种架构。`ElasticCache` 由一个代理服务器和一个支持主从复制的 `Redis` 集群组成，其中 `primary` 节点主要用于数据写入，`replicas` 用于数据读取。

### 使用 Codis 进行集中分片

`Codis` 引入了 `group` 的概念，每个 `group` 包含一个 `Redis` 主节点（`Redis master`），和 `1` 到多个 `Redis` 副本节点（`Redis slave`），如果主节点挂掉，一个副本节点可以被提升为新的主节点。

`Codis` 同样使用了预分片机制，与原生的 `Redis` 集群的哈希槽功能类似，所有的 `Key` 被分布到 `1024` 个槽中，这意味着总共可以有多达 `1024` 个 `group`。路由信息（即元数据）存储在强一致性的数据存储框架中，例如 `Etcd` 或 `Zookeeper`。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/08-Deep-Dive-Into-Redis-Cluster/10.png" style="width:100%"/>
