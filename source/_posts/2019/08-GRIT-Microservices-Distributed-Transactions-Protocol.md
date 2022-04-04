---
title: GRIT：一种微服务场景下分布式事务协议实现
date: 2019-10-29 09:42:47
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/mysql_study.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文描述了 eBay 技术人员最近发布的 GRIT 分布式事务协议的基本思想。（译）

categories: 
  - 架构设计

tags: 
  - 事务
---

> 原文链接：https://tech.ebayinc.com/engineering/grit-a-protocol-for-distributed-transactions-across-microservices/

> eBay 技术人员最近发布了一种称为 `GRIT` 分布式事务协议，该协议是为了解决跨多个底层数据库的微服务场景下分布式 ACID（原子性，一致性，隔离性，持久性）事务问题。

本文描述了在 **IEEE国际数据工程会议(ICDE) 2019** 上发布的 GRIT 协议的基本思想，<font color="red">并且提供了一个 JanusGraph 事务性存储后端的示例，该示例实现了 GRIT 协议的一部分功能</font>。这个示例关注的是仅有单一数据库的应用，但是正如我们所说，GRIT 可以支持由多个数据库组成的系统的 ACID 事务。

<!-- more -->

## 背景介绍

在 [微服务](https://en.wikipedia.org/wiki/Microservices) 架构中，应用程序可能会调用很多个微服务，通常这些微服务是用不同编程语言实现并且由不同的团队来维护，而且可能会使用多个数据库来实现微服务的功能。这种目前流行的架构为跨多个微服务的一致性分布式事务带来了新的挑战。支持微服务场景下的 ACID 事务是有实际需要的，但是在现有技术下实现起来是非常困难的，因为被设计用于单个数据库的分布式事务机制无法通过微服务架构轻松扩展到具有多个数据库的情况。

在涉及多个独立数据库的运行环境中，传统的两阶段提交（2PC）协议[1]从本质上来说是唯一一种能够实现分布式事务功能而不会给应用系统带来额外工作量的方式。然而，由于可能存在多个协调参与者而导致的较长调用链路以及在各个阶段所需要的资源锁定，所以它在横向扩展平台中工作的并不好。另一方面，引入第三方框架[2]（例如：Saga）执行事务日志的方式将导致应用程序产生复杂的事务补偿逻辑；并且由于已经部分执行成功的事务有可能是不可逆的，所以会对业务逻辑产生影响。

为了解决这些问题，我们开发了 [GRIT](https://ieeexplore.ieee.org/abstract/document/8731442)，一种用于全局一致性分布式事务的新协议，这个协议巧妙地结合了乐观并发控制（OCC）[3]、2PC 和 确定性数据库[4,5]的思想，首次实现了高性能，跨多个基础数据库微服务的全局一致性事务。

## GRIT: 一种分布式事务协议实现

下图描述了 GRIT 协议在两个数据库的微服务环境中的使用。GRIT 组件中包含了在图中中间部分显示的 GTM、GPL、DBTM、DBTL 和 LogPlayer 等组件。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2019/08-GRIT-Microservices-Distributed-Transactions-Protocol/GRIT-Protocol-for-Distributed-Transactions-across-Microservices6.png">

图中除去 GRIT 相关组件，剩余部分是一个横向扩展的两个数据库组成的微服务架构系统。它包括如下组成部分：

1. 应用程序：调用微服务来实现相关功能。
2. 微服务（实体服务）：为应用程序提供面向业务的服务以实现业务逻辑的功能模块。每个数据库可能要支持多个微服务，并且每个微服务可能彼此都是独立的。
3. 数据库服务：提供数据库 `读/写` 接口，可以直接访问数据库服务器。当支持事务时，它会缓存每次事务在执行阶段数据的 `读/写结果集`，并在提交时将其发送到 DBTM 用于解决冲突。
4. 数据库分片服务：数据库后台的存储服务，通过复制机制实现数据库高可用。

GRIT 包含的关键组件：

1. 全局事务管理器（GTM）：它用来协调跨多个数据库的全局事务。可以同时存在一个或多个 GTM。
2. 全局事务日志（GTL）：对于 GTM 来说，GTL 表示事务的请求队列。GTL 中事务的请求顺序决定了全局事务之间的相对可串行化顺序。全局事务日志数据的持久化是可选的。
3. 数据库事务管理器（DBTM）：DBTM 是每个数据库内部的事务管理器。它用来执行冲突检查并解决冲突问题，即本地事务提交就是在这里完成。
4. 数据库事务日志（DBTL）：DBTL 是每个数据库内部的事务日志，它用于记录与此数据库相关逻辑事务提交（包括单个数据库事务和多个数据库事务）。在 DBTL 中事务的顺序决定了整个数据库应用的可串行化顺序，包括 GTM 所保证的全局顺序。需要为每个日志条目分配一个日志序列号（LSN）。
5. 日志播放器（LogPlayer）：这个组件将用于更新数据的日志条目按顺序发送到后端存储服务器。每一个数据库服务按顺序使用日志条目数据进行逻辑事务提交。

为了更好的理解 GRIT 协议的实现细节，我们用下图来演示分布式事务的主要步骤：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2019/08-GRIT-Microservices-Distributed-Transactions-Protocol/GRIT-main-steps2.png">

在 GRIT 中，一个分布式事务要经历三个步骤：

1. 乐观执行（步骤 1-4）：应用程序通过调用微服务来执行业务逻辑，数据库服务捕捉事务的 `读/写结果集`。在这个环节没有实际的数据修改发生。
2. 逻辑提交（步骤 5-11）：一旦应用程序发送提交事务请求，每一个数据库服务节点将 `读/写结果集` 提交到它对应的 DBTM。DBTM 使用 `读/写结果集` 进行冲突检查，以实现本地提交决策。GTM 在收集到所有 DBTM 对事务的本地决策之后，做出全局提交决策。一旦事务的 `写结果集` 被持久化到所对应的数据库的日志存储（DBTL）中，事务就会进行逻辑提交。这涉及到 GTM 和 DBTM 之间的最小协调。
3. 物理应用（步骤 12-13）：日志播放器异步发送 DBTL 条目到后端存储服务器。数据修改发生在这个阶段。

总体来说，我们的方案避免了执行和提交过程中的悲观锁定，同时避免了物理提交等待。我们采用乐观锁的方案，通过利用逻辑提交日志，并使用确定性数据库技术将物理数据库更新操作从提交决策过程中移出，使提交过程更加高效，这类似于复制中的日志播放。

GRIT 能够为调用微服务的应用程序实现一致性的高吞吐量和可串行化的分布式事务，并保证最小范围协调性。GRIT 非常适合冲突较少的事务，并为那些需要复杂机制以在具有多个底层数据库的微服务之间实现一致性事务的应用程序提供了关键功能。

## 在单一数据库中应用 GRIT

正如你所了解到的，GRIT 协议包含两部分：<font color="red">其中一部分是用于 DBTM、DBTL 和 LogPlayer 操作的，这是每个数据库（或是每个数据库内部，可以是数据库的一组分区）提供的；另外一部分是用于 GTM 和 DBTM 操作的跨数据库协调。</font>在下图中，我们演示了使用了单个数据库的 GRIT 部分协议为 [JanusGraph](https://janusgraph.org/) 设计的事务图形存储后端（称为 NuGraphStore）。

下图显示了如何使用两个可用区域（AZ1 和 AZ2）部署来实现 NuGraphStore。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2019/08-GRIT-Microservices-Distributed-Transactions-Protocol/GRIT-Protocol-for-Distributed-Transactions-across-Microservices4.png">

JanusGraph 的 NuGraphStore 存储后端包含了以下几个组件：

- 存储插件（Storage plugin）：一个自定义的存储接口插件，用于 JanusGraph 数据库层、后端存储引擎和事务协议组件之间的接口。
- DBTM：为乐观并发控制提供关键地冲突检查。这是分布式事务协议 [GRIT](https://ieeexplore.ieee.org/abstract/document/8731442) 在单个数据库乐观并发控制中的组成部分。
- 日志存储：事务中的变更数据的复制日志存储。每个事务一个条目，由日志序列号（LSN）索引。它在传统数据系统中充当 WAL（预写日志）角色。LogStore 就是 GRIT 架构中的 DBTL 部分。
- LogPlayer：在后端服务器中异步的执行日志条目。
- 存储引擎：用于存储 JanusGraph 中的KCV（键-列-值）数据的后端存储引擎。它执行读取和变更操作，并支持 LSN 定义的快照。

当一个应用程序执行事务时，它可以从存储层读取数据，也可以向存储层写入数据。对于读取数据操作，存储插件直接和存储服务器进行交互（要读取的数据在事务的 `写结果集` 这种情况除外）。当应用程序在事务上下文环境中从存储层读取数据时，存储插件也会对 `读结果集` 进行追踪。每次读取中有用信息的是 <key, lsn> 键值对，其中 LSN 是反映读取键值时存储引擎状态的日志序列号。LSN 是一个事务中变更数据的日志索引条目。它由 LogStore 来分配，被设计用于后端数据库的快照。没有被查询到的 key 同样被记录成 `读结果集` 的一部分。和读操作不同，在写操作中，存储插件不会直接和存储服务器交互。相反，存储插件会为事务缓存它对应的 `写结果集` 中的写操作。

当事务提交之时，存储插件将提交请求以及为事务捕获的 `读/写结果集` 提交到 DBTM。DBTM 为 OCC 执行事务的标准冲突检查。如果没有冲突，它将把 `写结果集` 持久化到复制的日志存储（即它将 `写结果集` 发送到日志存储副本集，因此所有副本都保持完全相同的日志）此时，事务就完成了逻辑提交，DBTM 会向存储插件返回响应结果信息。日志播放器跟踪日志存储，并根据数据分布将日志条目播放到后端分片服务器。

值得指出的是，上面描述的内容只是一个基本设计，还有很多机会来提高它的性能和可用性。我们相信，在跨组件进行优化或使用 DBTM 的复制来实现更高的可靠性之前，使基础组件成熟稳定是更富有成效的。当然，还有很多不同的方式来捕捉 `读/写结果集`。对于一个键值存储，我们需要的最简单进行冲突检测的方式就是 <key, lsn> 键值对。然而，为了支持更复杂的系统应用，`读结果集` 可能是一定范围或者是限定条件的数据，正如这篇文章内描述的。[6]在本文撰稿时，NuGraphStore 也正在进行开源。

## 参考文献
[1] C. Mohan, Bruce Lindsay and R. Obermarck, “Transaction management in the R* distributed database management system” ACM Transactions on Database Systems (TODS), Volume 11 Issue 4, Dec. 1986, Pages 378 - 396.

[2] Pat Helland, “Life beyond Distributed Transactions: an Apostate’s Opinion”, CIDR 2007. 

[3] H.T. Kung, J.T. Robinson, “On Optimistic Methods for Concurrency Control”, ACM Transactions on Database Systems 6:2, June, 1981.

[4] Thomson, Alexander and Diamond, Thaddeus and Shao, Philip and Ren, Kun and Weng, Shu-Chun and Abadi, Daniel J, “Calvin: Fast distributed transactions for partitioned database systems”, SIGMOD 2012.

[5] Kun Ren, Alexander Thomson, Daniel Abadi, “An Evaluation of the Advantages and Disadvantages of Deterministic Database Systems”, VLDB 2014.

[6] Thomas Neumann, Tobias Mühlbauer, Alfons Kemper, “Fast Serializable Multi-Version Concurrency Control for Main-Memory Database Systems”, SIGMOD 2015

## GRIT: a Protocol for Distributed Transactions across Microservices

> eBay technologists recently showed off a distributed transaction protocol called GRIT, for distributed ACID (atomicity, consistency, isolation, durability) transactions across microservices with multiple underlying databases.

This article describes the basic ideas of the GRIT protocol, which was announced at the IEEE International Conference on Data Engineering (ICDE) 2019, and provides an example of using part of the protocol for implementing a transactional storage backend for JanusGraph. This example focuses on a system with only a single database, but as we said, GRIT can support ACID transactions for systems consisting of multiple databases.

## Background
In a [microservice](https://en.wikipedia.org/wiki/Microservices) architecture, an application may invoke multiple microservices, which are usually implemented in different application languages by different teams and may use multiple underlying databases to achieve their functionality. This popular architecture brings new challenges for consistent distributed transactions across multiple microservices. It is a real requirement to support ACID transactions in the context of microservices, but is very hard to achieve using existing technologies, since distributed transaction mechanisms designed for a single database cannot be easily extended to the cases with multiple databases through microservices.

In environments that involve multiple independent databases, the traditional two-phase commit (2PC) protocol[1] was essentially the only option for distributed transactions by the system without additional application effort. However, it does not work well in a scale-out platform due to long paths of potentially many coordinating participants and the locking required over the phases. On the other hand, using a transaction log executed by a framework[2] such as Saga will incur complex compensating logic by applications and may have business implications due to irreversible partially successful transactions. 

To address these issues, we developed [GRIT](https://ieeexplore.ieee.org/abstract/document/8731442), a novel protocol for globally consistent distributed transactions that cleverly combines ideas from optimistic concurrency control (OCC)[3], 2PC, and deterministic databases[4,5] to achieve, for the first time, high-performance, globally consistent transactions across microservices with multiple underlying databases.

## GRIT: a protocol for distributed transactions

The following diagram illustrates the GRIT protocol in a system of microservices with two databases. The GRIT components, including GTM, GTL, DBTM, DBTL, and LogPlayer, are shown in the center.

<img src="https://tech.ebayinc.com/assets/Uploads/Editor/GRIT-Protocol-for-Distributed-Transactions-across-Microservices6.png">

Without the GRIT components, the diagram represents a system of plain microservice architecture with two scale-out databases. They consist of the following:

1. Applications: invoke microservices to achieve their functionality.
2. Microservices (Entity Services): building blocks to provide business-oriented service for applications to implement business logic. Each DB may have support for multiple microservices, and each microservice is likely independent of the other.
3. DB Services: provide DB read/write interface and directly access DB servers. When supporting transactions, it also caches the read/write results of each transaction during the execution phase and sends them to its DBTM for conflict resolution at commit time.
4. DB shard servers: the backend storage servers for the database, usually replicated for high availability.

The key components of GRIT include:

1. Global Transaction Manager (GTM): It coordinates global transactions across multiple databases. There can be one or more GTMs.
2. Global Transaction Log (GTL): It represents the transaction request queue for a GTM. The order of transaction requests in a GTL determines the relative serializability order among global transactions. Persistence of GTLs is optional.
3. Database Transaction Manager (DBTM): The transaction manager at each database realm. It performs the conflict checking and resolution, i.e. local commit decision is located here.
4. Database Transaction Log (DBTL): The transaction log at each database realm that logs logically committed transactions that relate to this database (including single database transactions and multi-database transactions). The order of transactions in a DBTL determines the serializability order of the whole database system, including the global order dictated by the GTM. A log sequence number (LSN) is assigned to each log entry.
5. LogPlayer: This component sends log entries, in sequence, to the backend storage servers for them to apply the updates. Each DB server applies log entries of logically committed transactions in order.

For the purpose of understanding the details of the protocol, we use the following diagram to show the main steps for a distributed transaction.

<img src="https://tech.ebayinc.com/assets/Uploads/Editor/GRIT-main-steps2.png">

In GRIT, a distributed transaction goes through three phases:

1. Optimistic execution (steps 1-4): As the application is executing the business logic via microservices, the database services capture the read-set and write-set of the transaction. No actual data modification occurs at this phase.
2. Logical commit (steps 5-11): Once the application requests the transaction commit, the read-set and write-set at each database service point are submitted to its DBTM. The DBTM uses the read-set and write-set for conflict checking to achieve local commit decision. The GTM will make the global commit decision after collecting all the local decisions of DBTMs for the transaction. A transaction is logically committed once its write-sets are persisted in log stores (DBTLs) for databases involved. This involves minimum coordination between the GTM and the DBTMs.
3. Physical apply (steps 12-13): The log players asynchronously sends DBTL entries to backend storage servers. The data modification occurs at this phase.

Overall, our approach avoids pessimistic locking during both execution and commit process and avoids waiting for physical commit. We take the optimistic approach and also make the commit process very efficient by leveraging logical commit logs and moving the physical database changes out of the commit decision process with deterministic database technology, which is similar to log play in replication.

GRIT is able to achieve consistent high throughput and serializable distributed transactions for applications invoking microservices with minimum coordination. GRIT fits well for transactions with few conflicts and provides a critical capability for applications that otherwise need complex mechanisms to achieve consistent transactions across microservices with multiple underlying databases.

## Applying GRIT for a single database

As you can see, the GRIT protocol contains two parts: one for each database (or each database realm, which can be a set of partitions of a database) performed by DBTM, DBTL, and LogPlayer, and the other for cross-database coordination by GTM and DBTMs. In the following diagram, we illustrate the design of a transactional graph store backend (called NuGraphStore) for [JanusGraph](https://janusgraph.org/) using the part of GRIT protocol for a single database.

The following diagram shows how NuGraphStore is implemented with two availability zone (AZ1 and AZ2) deployment for illustration.

GRIT Protocol for Distributed Transactions across Microservices4

<img src="https://tech.ebayinc.com/assets/Uploads/Editor/GRIT-Protocol-for-Distributed-Transactions-across-Microservices4.png">

There are a few components involved in the NuGraphStore backend for JanusGraph:

- Storage plugin: a custom storage interface plugin to interface between the JanusGraph DB Layer and the backend storage engine and transaction protocol components.
- DBTM: performs the critical conflict checking for optimistic concurrency control. This is part of the [GRIT](https://ieeexplore.ieee.org/abstract/document/8731442) distributed transaction protocol on single databases performing OCC.
- LogStore: replicated log store for mutations from transactions. One entry for each transaction, indexed by Log Sequence Number (LSN). It acts as a WAL (write-ahead-log) in traditional database systems. The LogStore is the DBTL in our GRIT architecture. 
- LogPlayer: applies log entries to the backend servers asynchronously.
- Storage engine: backend storage engine to store KCV (Key-Column-Value) data from JanusGraph. It performs reads and mutations and supports snapshots defined by the LSN.

As an application is performing a transaction, it can read from the store and write to the store. For the read operations, the storage plugin directly communicates with the storage servers (except for reads that are found in the write-set for the transaction). The storage plugin also keeps track of the read-set as the application is reading from the store in the context of a transaction. The useful information for each read is the <key, lsn> pair, where lsn is the log sequence number reflecting the storage engine state when the key-value is read. An LSN is the log index of the entry for the mutations of a transaction. It is assigned by the LogStore and used to define the snapshot of the backend databases. A key being not found is also recorded as part of the read-set. Unlike reads, the storage plugin for writes does not directly communicate with the storage servers. Instead, the storage plugin buffers the writes in the corresponding write-set for the transaction.

When a transaction commits, the storage plugin submits the commit request with the read-set and write-set it has captured for the transaction to the DBTM. The DBTM performs the standard conflict checking for OCC for the transaction. If there is no conflict, it will persist the write-set to the replicated LogStore (i.e., it sends the write-set to the LogStore replica set, so all the replicas keep the exact same log). At this point, the transaction commit completes logically, and the DBTM responds back to the storage plugin. The LogPlayers tail the LogStores and play the log entries to the backend shard servers based on the data distribution.

It’s worth pointing out that the above description is a basic design with many opportunities to enhance for performance and reliability. It’s our belief that it is more productive to make the basic components mature before optimizing across the components or using replication for DBTM to achieve higher reliability. Also, there are different ways that we can capture the read-set and write-set. For a KV store, the simplest form we need for conflict checking is <key, lsn> pairs. To support more complex systems, however, the read-set may include ranges, or predicates, as described in.[6] As of this writing, NuGraphStore is going through the open source process.

## References
[1] C. Mohan, Bruce Lindsay and R. Obermarck, “Transaction management in the R* distributed database management system” ACM Transactions on Database Systems (TODS), Volume 11 Issue 4, Dec. 1986, Pages 378 - 396.

[2] Pat Helland, “Life beyond Distributed Transactions: an Apostate’s Opinion”, CIDR 2007. 

[3] H.T. Kung, J.T. Robinson, “On Optimistic Methods for Concurrency Control”, ACM Transactions on Database Systems 6:2, June, 1981.

[4] Thomson, Alexander and Diamond, Thaddeus and Shao, Philip and Ren, Kun and Weng, Shu-Chun and Abadi, Daniel J, “Calvin: Fast distributed transactions for partitioned database systems”, SIGMOD 2012.

[5] Kun Ren, Alexander Thomson, Daniel Abadi, “An Evaluation of the Advantages and Disadvantages of Deterministic Database Systems”, VLDB 2014.

[6] Thomas Neumann, Tobias Mühlbauer, Alfons Kemper, “Fast Serializable Multi-Version Concurrency Control for Main-Memory Database Systems”, SIGMOD 2015
