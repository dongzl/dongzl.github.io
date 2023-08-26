---
title: 微服务架构中的SAGA模式是什么？它能够解决哪些问题？
date: 2023-08-20 11:04:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/saga_pattern.png

# author information, multiple authors are set to array
# single author
author:
- nick: Soma
  link: https://medium.com/@somasharma_81597
- nick: 董宗磊
  link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: SAGA 是微服务中一种非常重要的架构模式，它常用于解决分布式系统中数据一致性问题。

categories:
- 架构设计

tags:
- SAGA
---

> 原文链接：https://medium.com/javarevisited/what-is-saga-pattern-in-microservice-architecture-which-problem-does-it-solve-de45d7d01d2b

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/22-What-SAGA-Pattern-Microservice-Architecture/01.webp"/>

朋友们大家好，如果大家正在从事 `Java` 微服务相关岗位的工作，或者正在准备微服务岗位的 `Java` 开发人员面试，那么我们必须要准备 `SAGA` 模式的相关知识。`SAGA` 是一种重要的微服务模式，它的目的是解决微服务架构中长事务的问题。这也是流行的[微服务面试问题](https://medium.com/javarevisited/50-microservices-interview-questions-for-java-programmers-70a4a68c4349)之一，经常被用来考察经验丰富的开发人员。

由于微服务架构将我们的应用程序分解为多个小的应用程序，单个请求也被分解为多个请求，所以有可能会出现部分请求成功，部分请求失败的现象，在这种情况下，保证数据的一致性是很困难的。

如果我们正在处理实际业务场景数据，例如亚马逊上的订单服务，那么我们必须妥善处理这种情况，保证如果出现付款失败，那么库存将恢复到原始状态并且订单不会被发货。

在这篇文章中，我们将解释什么是 `SAGA` 模式？它的作用是什么、能够解决什么问题以及微服务架构中 `SAGA` 模式的优缺点。

顺便说一下，如果大家正在准备 `Java` 开发岗位的面试，那么也可以看看我之前发布的文章，比如 [25 个 Java 高级问题](https://medium.com/javarevisited/200-coursera-plus-discount-and-best-new-year-deals-for-developers-in-2023-eb2b682575)、[25 个 Spring 框架问题](https://medium.com/javarevisited/25-spring-framework-interview-questions-for-1-to-3-years-experienced-java-programmers-567f268ed897)、[20 个面试中常见的 SQL 查询](https://medium.com/javarevisited/20-sql-queries-for-programming-interviews-a7b5a7ea8144)、[50 个微服务问题](https://medium.com/javarevisited/50-microservices-interview-questions-for-java-programmers-70a4a68c4349)、[60 个关于树的数据结构问题](https://medium.com/javarevisited/top-60-tree-data-structure-coding-interview-questions-every-programmer-should-solve-89c4dbda7c5a)、[15 个系统设计问题](https://medium.com/javarevisited/7-system-design-problems-to-crack-software-engineering-interviews-in-2023-13a518467c3e)，以及 [35 个核心 Java 问题](https://medium.com/javarevisited/top-10-java-interview-questions-for-3-to-4-years-experienced-programmers-c4bf6d8b5e7b)。

## 什么是SAGA设计模式？它解决什么问题？

`SAGA`（或 `Saga`）模式是一种微服务设计模式，它常用于处理分布式系统中的数据一致性问题。它提供了一种处理由多个步骤组成的长事务的解决方案，其中每个步骤都是一个独立的数据库操作。

`SAGA` 模式的主要思路是将事务的所有步骤捕获在数据库中，以便在发生故障时系统可以将事务回滚到其初始状态。

`SAGA` 模式解决了分布式系统中维护数据一致性的问题，在分布式系统中很难保证事务中的所有操作都以原子方式执行，尤其是在出现故障的情况下。

`SAGA` 模式的一个经典案例是电子商务交易，例如在 `Amazon` 或 `FlipKart` 下单，下单后从客户的账户中扣除货款，并将商品在库存中扣除。

如果其中某个步骤出现失败，则需要回滚之前的步骤以确保数据一致性。例如，如果付款失败，则需要取消扣除商品的库存。`SAGA` 模式解决在涉及多个步骤的事务中出现部分成功部分失败可能会导致的数据一致性问题。

这是某个微服务架构图，用来演示 `SAGA` 模式的工作原理：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/22-What-SAGA-Pattern-Microservice-Architecture/02.jpeg"/>

<hr />

## 微服务架构中 SAGA 设计模式的优缺点

每当我们学习一种设计模式时，我们都应该了解它的优缺点，因为它可以帮助我们更好地理解这种模式，并帮助我们决定何时在应用程序中使用它们：

以下是微服务中 `SAGA` 模式的一些优缺点：

**优点**：

以下是在微服务架构中使用 `SAGA` 设计模式的一些优点：

1. 比较方便处理跨多个微服务的复杂事务；
2. 可以优雅地处理系统故障并确保数据一致性；
3. 提高系统的弹性和稳定性；
4. 避免数据不一致和丢失更新问题；
5. 提供清晰明确的交易补偿流程。

**缺点**：

以下是在微服务架构中使用 SAGA 设计模式的一些缺点：

1. 实施和维护困难，程序监控和调试也比较困难；
2. 我们将需要存储和管理 `Saga` 状态，这会产生额外的开销；
3. 由于需要管理跨多个微服务的事务，它还会产生额外的性能开销；
4. 由于微服务之间需要进行多次交互，应用程序响应延迟会增加；
5. 跨不同微服务的 `Saga` 并没有标准化实现，未来如果像 `Spring Cloud` 或 `Quarks` 这样的框架原生支持 `Saga` 模式，那就完美了。

## 如何在微服务架构中实现 SAGA 模式？

`SAGA` 模式可以通过在微服务架构中将复杂的业务事务拆分为多个更小的独立步骤或服务来实现。

每个步骤都会与其所对应的微服务进行通信以完成事务的一部分，如果某个步骤失败，系统将会启动补偿机制以撤消之前的步骤。

协调这些步骤可以使用数据库、消息队列或其它协调服务来存储事务的状态并触发补偿机制来实现，这样系统可以确保最终一致性并优雅地处理系统故障。

如果我们想知道是否有某个 `Java` 微服务框架可以提供对 `SAGA` 模式的支持？很遗憾的是并没有特定的微服务框架为 `SAGA` 模式提供直接支持。

不过我们可以使用 `Apache Camel` 等框架或者与 `Spring` 集成以及 `Apache Kafka`、事件溯源和消息驱动架构等技术来实现 `SAGA` 模式。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/22-What-SAGA-Pattern-Microservice-Architecture/03.webp"/>

<hr />

感谢大家阅读这篇的文章，这就是**微服务架构中的 `SAGA` 设计模式**。它是一种复杂但重要的设计模式，从面试的角度来说学习这种模式非常重要。

即使大家并没有在项目中实现 `SAGA` 模式，它同样值得关注，因为管理分布式事务和数据一致性的问题是一个真实存在的问题，作为经验丰富的开发人员，我们应该知道如何在微服务架构中处理这个问题。
