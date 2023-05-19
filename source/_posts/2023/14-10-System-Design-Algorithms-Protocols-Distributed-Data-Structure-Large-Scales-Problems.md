---
title: （进行中）解决大规模问题的 10 个系统设计算法、协议和分布式数据结构
date: 2023-05-20 20:03:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/microservices_algorithms.png

# author information, multiple authors are set to array
# single author
author:
- nick: Soma
  link: https://medium.com/@somasharma_81597

# post subtitle in your index page
subtitle: 解决大规模分布式系统问题的 10 个必备系统设计算法和分布式数据结构。

categories:
- 架构设计

tags:
- JWT
- OAuth
- SAML
---

> 原文链接：https://medium.com/javarevisited/10-system-design-algorithms-protocols-and-distributed-data-structure-to-solve-large-scales-40bd24d9a57f

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/14-10-System-Design-Algorithms-Protocols-Distributed-Data-Structure-Large-Scales-Problems/02.png" style="width:100%"/>

朋友们大家好，如果您正在准备系统设计面试，那么您应该关注的一件事情：学习不同的系统设计算法以及这些算法能够在分布式系统和微服务中解决的问题。

在过去的文章中，我分享了[7 个系统设计问题](https://medium.com/javarevisited/7-system-design-problems-to-crack-software-engineering-interviews-in-2023-13a518467c3e)和[10 个系统设计基本概念](https://medium.com/javarevisited/top-10-system-design-concepts-every-programmer-should-learn-54375d8557a6)两篇文章；在本文中，我将继续分享每个开发人员都应该学习的 `10` 个系统设计算法。

事不宜迟，以下是可用于解决大规模分布式系统问题的 `10` 种系统设计算法和分布式数据结构：

1. 一致性哈希（Consistent Hashing）
2. MapReduce
3. 分布式哈希表（Distributed Hash Tables (DHT)）
4. 布隆过滤器（Bloom Filters）
5. 两阶段提交（Two-phase commit (2PC)）
6. Paxos
7. Raft
8. Gossip 协议（Gossip protocol）
9. Chord
10. CAP 理论（CAP theorem）

这些算法和分布式数据结构只是用于解决大规模分布式系统问题的众多技术中的几个例子。

## 面向程序开发人员的 10 大分布式数据结构和系统设计算法

深入地理解这些算法，以及理解在不同的场景中如何有效地应用这些算法，对一名开发人员很重要。

因此，让我们来深入研究其中的每一个算法，研究它们是什么、它们是如何工作的以及何时使用它们。

## 1. 一致性哈希

[一致性哈希](https://medium.com/javarevisited/what-is-consistent-hashing-what-problem-does-it-solve-9c161fc6147d)是分布式系统中用于在多个节点之间有效分发数据的技术。当在系统中添加或删除节点时，它可以最小化系统在节点之间迁移的数据量。

一致性哈希背后的基本原理是使用哈希函数将每条数据映射到系统中的某个节点。每个节点都分配了一段哈希值范围值，哈希值映射到该范围内的的任何数据都会被分配到该节点。

当从系统中添加或删除节点时，需要将分配给该节点的数据迁移到另一个节点，这个过程是通过使用虚拟节点的概念来实现的，并不是直接为每个物理节点分配一系列哈希值，而是为每个物理节点分配了多个虚拟节点，每个虚拟节点都被分配了一段唯一的哈希值范围，任何哈希值映射到该范围内的数据都被分配给虚拟节点相对应的物理节点。

当从系统中添加或删除节点时，只有受影响的虚拟节点上的数据需要重新分配，并且分配给这些虚拟节点的所有数据都将被迁移到另外一个节点，这就允许系统动态高效地进行扩展，从而避免在每次添加或删除节点时需要重新分配数据。

总的来说，**一致性哈希提供了一种在分布式系统中的多个节点之间分发数据的简单而有效的方法**。它通常用于大型分布式系统，例如内容分发网络和分布式数据库，用来提供高可用性和可扩展性。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/14-10-System-Design-Algorithms-Protocols-Distributed-Data-Structure-Large-Scales-Problems/01.png" style="width:100%"/>

<hr />

## 2. Map reduce

`MapReduce` 是一种用于在分布式系统中处理海量数据集的编程模型和框架。它的思想最初由谷歌提出，目前广泛应用于许多大数据处理系统，例如 `Apache Hadoop`。

`MapReduce` 背后的基本思想是将海量数据集切割成较小的数据块，并将这些数据块分布在集群中的多个节点上，多个节点之间并行处理数据，处理过程分为两个阶段：**Map阶段**和**Reduce阶段**。

在 `Map` 阶段，输入数据集由一组 `Map` 函数并行处理，每个 `Map` 函数都将一组键值对作为输入，并生成一组中间键值对作为输出，然后将这些中间键值对按键排序和分区，并发送到 `Reduce` 阶段进行处理。

在 `Reduce` 阶段，`Map` 阶段生成的中间键值对被一组 `Reduce` 函数并行处理，每个 `Reduce` 函数都将一个键和一组值作为输入，并生成输出一组键值对。

下面是一个示例，说明如何使用 `MapReduce` 计算大型文本文件中单词的出现频率：

1. **Map阶段**：每个 `Map` 函数读取输入文件的一个数据块并输出一组中间键值对，其中键是某个单词，值是该词在数据块中出现的次数；
2. **Shuffle阶段**：对生成的中间键值对按键排序和分区，以便将相同单词分组到一起；
3. **Reduce阶段**：每个 `Reduce` 函数都将某个单词和该单词出现次数作为输入，并输出一个键值对，其中键是该单词，值是输入文件中该单词出现的总次数。

`MapReduce` 框架负责计算的并行处理、数据分布处理和机器故障容错，这使它能够高效可靠地处理海量数据集，即使是在普通的商用硬件集群上也是如此。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/14-10-System-Design-Algorithms-Protocols-Distributed-Data-Structure-Large-Scales-Problems/02.png" style="width:100%"/>

<hr />

## 3. 分布式哈希表（DHT）

分布式哈希表（`DHT`）是一种分布式系统，可提供散列的键值存储，它常用于 `P2P` 网络场景，通过可扩展和容错的方式存储和检索信息。

在 `DHT` 中，每个参与计算的节点存储的是所有键值对的子集，并使用映射函数将键分配到某个节点，这使得某个节点仅通过查询一小部分节点就可以定位到与给定键关联的值，而且查询的通常是那些映射空间中接近给定键的所在节点的键。

`DHT` 提供了几个优秀的特性，例如自组织、容错、负载均衡和高效路由。`DHT` 通常用于 `P2P` 文件共享系统、内容分发网络和分布式数据库等场景。

一种非常流行的 `DHT` 算法是 `Chord` 协议算法，它使用基于环的拓扑结构和一致性的哈希函数将密钥分配给某个节点；另一个广泛使用的 `DHT` 算法是 `Kademlia` 协议算法，它使用二叉树树状结构来定位存储给定密钥的节点。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/14-10-System-Design-Algorithms-Protocols-Distributed-Data-Structure-Large-Scales-Problems/03.png" style="width:100%"/>

<hr />

## 4. 布隆过滤器

布隆过滤器是一种概率数据结构，通常用于高效的测试集合中是否存在某个元素，它是由 `Burton Howard Bloom` 于 `1970` 年提出的。布隆过滤器是一种节省存储空间的概率数据结构，用于测试元素在集合中是否存在，它使用位数组和一组哈希函数来存储和检查集合中是否存在某个元素。

将元素添加到布隆过滤器的过程首先将元素传入给一组散列函数，这些散列函数返回位数组中的一组索引位置，然后将位数组中对应索引位置的值设置为 `1`。

为了检查集合中是否存在某个元素，首先对该元素应用相同的哈希函数，并在位数组中检查哈希函数结果对应的索引位置的值，如果对应索引位置的所有值都为 `1`，则认为该元素可能存在于集合中；但是如果对应索引位置的某个值 `0`，则认为该元素不存在于集合中。

由于布隆过滤器使用哈希函数来索引位数组，因此存在误报的可能性，即过滤器可能判断认为某个元素存在于集合中，而实际上它不存在；但是可以通过调整位数组的大小和使用的哈希函数的数量来控制误报出现的概率，假阴性率，即布隆过滤器无法识别集合中实际存在的元素的概率为零。

布隆过滤器广泛应用于网络、数据库和网络缓存等各个领域，用来高效地测试集合中是否存在某个成员。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/14-10-System-Design-Algorithms-Protocols-Distributed-Data-Structure-Large-Scales-Problems/04.png" style="width:100%"/>

## 5. 2阶段提交

[2阶段提交（2PC）](https://medium.com/javarevisited/difference-between-saga-pattern-and-2-phase-commit-in-microservices-e1d814e12a5a)是一种用于保证分布式系统中事务的原子性和一致性的协议，它能够保证参与事务的所有节点一起提交或回滚的思想。

两阶段提交协议分两个步骤：

1. **Prepare阶段**：在 `prepare` 阶段，协调者向所有参与者发送消息，要求它们准备提交事务；每个参与者都会回复一条响应消息，表明它是否做好提交事务准备，如果某个参与者还未准备就绪，它会回复一条消息，表明它无法参与事务；
2. **Commit阶段**：如果所有参与者都已经做好提交准备，协调者会向所有节点发送一条消息，要求他们提交事务，每个参与者提交事务后会向协调者发送确认消息，如果某个参与者无法提交事务，它会回滚事务并向协调者发送一条消息，表明它已回滚事务。

如果协调者收到所有参与者的确认消息，它会向所有节点发送一条消息，表明事务已提交；如果协调器收到某个参与者的回滚消息，它会向所有节点发送一条消息，指示事务已回滚。

[2阶段提交协议](https://medium.com/javarevisited/difference-between-saga-pattern-and-2-phase-commit-in-microservices-e1d814e12a5a)能够确保分布式系统中的所有节点事务结果是一致的，即使出现故障也是如此；然而它也有一些缺点，包括延迟较高和死锁的可能性，此外它需要一个协调者节点，这可能引起单点故障。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/14-10-System-Design-Algorithms-Protocols-Distributed-Data-Structure-Large-Scales-Problems/05.png" style="width:100%"/>

## 6. Paxos

`Paxos` 是一种分布式共识算法，即使在某个节点出现故障的情况下，也允许一组节点就某个结果值达成一致。它是由 `Leslie Lamport` 于 `1998` 年提出，现已成为分布式系统的基本算法。

`Paxos` 算法旨在处理各种故障场景，包括消息丢失、重复、重排序和节点故障等等。该算法分两个阶段进行：**prepare阶段**和**accept阶段**。在**prepare阶段**，一个节点向所有其他节点发送一个 `prepare` 消息，要求他们确保不接受任何数值小于一定值的提议。

一旦大多数节点响应提案消息，节点就可以进入**accept阶段**，在**accept阶段**，节点向所有其他节点发送 `accept` 消息，提出某个值，如果大多数节点响应消息内容，则该值被视为已接受。

`Paxos` 是一种复杂的算法，它有多种变体和优化，例如 `Multi-Paxos`、`Fast Paxos` 等；这些变体目的是减少交换的消息数量，优化算法的延迟，并减少需要参与共识的节点数量。`Paxos` 被广泛应用于分布式数据库、分布式文件系统等需要高度容错性的分布式系统中。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/14-10-System-Design-Algorithms-Protocols-Distributed-Data-Structure-Large-Scales-Problems/06.png" style="width:100%"/>
