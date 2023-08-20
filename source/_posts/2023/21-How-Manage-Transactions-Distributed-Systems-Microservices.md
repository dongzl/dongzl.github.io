---
title: 如何实现分布式系统和微服务中的事务管理？
date: 2023-08-13 21:23:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/transactions_distributed.png

# author information, multiple authors are set to array
# single author
author:
- nick: Soma
  link: https://medium.com/@somasharma_81597
- nick: 董宗磊
  link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 在本文中，我们将学习实现分布式系统和微服务中事务管理的三种种方案：两阶段提交、SAGA 和事件溯源机制。

categories:
- 架构设计

tags:
- 两阶段提交
- SAGA
- 事件溯源
---

> 原文链接（请科学上网）：https://medium.com/javarevisited/what-is-saga-pattern-in-microservice-architecture-which-problem-does-it-solve-de45d7d01d2b

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/21-How-Manage-Transactions-Distributed-Systems-Microservices/01.webp"/>

朋友们大家好，如果大家正在准备微服务相关的面试题，那么我们可能同样需要知道*如何管理分布式系统或微服务中的事务*？这个问题是[流行的微服务面试问题](https://medium.com/javarevisited/50-microservices-interview-questions-for-java-programmers-70a4a68c4349)之一，难道不是吗，它也是重要的问题之一。

在前面的文章中，我分享了[基本的微服务设计原则和最佳实践](https://medium.com/javarevisited/10-microservices-design-principles-every-developer-should-know-44f2f69e960f)以及流行的[微服务设计模式](https://medium.com/javarevisited/top-10-microservice-design-patterns-for-experienced-developers-f4f5f782810e)，在本文中，我将回答这个问题，并告诉大家在分布式系统中管理事务的不同方法。

如果我们曾经在[微服务架构](https://medium.com/javarevisited/difference-between-microservices-and-monolithic-architecture-for-java-interviews-af525908c2d5)的系统场景中工作过，那么我们就应该知道某个业务场景的事务通常会跨越多个微服务，这就是为什么确保数据一致性和管理事务是非常具有挑战性的工作。

> 事务处理不当可能会导致数据不一致、丢失数据更新或数据重复，这可能会给业务带来严重后果。

因此，拥有强大的事务管理系统以确保系统的可靠性是至关重要。

在本文中，我们将**探讨管理分布式系统和微服务中事务的各种策略和最佳实践**。我们将讨论与分布式事务相关的挑战以及如何克服这些挑战，包括两阶段提交和 `SAGA` 模式。

此外，我们还将介绍流行的微服务相关的框架（例如 `Spring Cloud` 和 `Apache Kafka`）所提供的不同事务管理机制。

阅读完本文后，我们将能够清楚地了解如何管理分布式系统中的事务并确保数据的可靠性和一致性。

## 管理分布式系统和微服务中事务的三种方法？

正如我所说，由于系统的分布式特性，管理分布式系统和微服务中的事务可能是一项复杂的工作。在现实世界中，对于多个服务需要协作和共享数据的分布式系统中，事务对于确保数据一致性和完整性是至关重要的。

管理分布式系统中事务的最流行方案之一是使用**两阶段提交协议（`2PC`）**。在该协议中，**事务协调者**负责确保事务中的所有参与者都同意提交或回滚事务。

协调者首先向所有参与者发送“准备提交”消息，如果所有参与者都做出正确响应，则协调者向所有参与者发送“提交”消息。

如果任何参与者做出否定响应，协调者将向所有参与者发送“中止”消息，并进行事务回滚。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/21-How-Manage-Transactions-Distributed-Systems-Microservices/02.gif"/>

但是，**两阶段提交可能会成为性能瓶颈**，并可能导致系统可用性降低。另一种方法是使用**基于补偿的事务协议**。在该协议中，事务的每个参与者都有责任在事务失败时补偿事务所产生的任何影响。

这种方法比 `2PC` 协议更灵活，速度更快，可扩展性更强，但需要仔细设计补偿逻辑。

用于管理分布式系统中事务的其它技术还包括[Saga 模式](https://medium.com/javarevisited/what-is-saga-pattern-in-microservice-architecture-which-problem-does-it-solve-de45d7d01d2b)和事件溯源机制。[Saga](https://medium.com/javarevisited/difference-between-saga-and-cqrs-design-patterns-in-microservices-acd1729a6b02)是一种事务模式，它使用一系列本地事务来实现全局事务。

另一方面，**事件溯源**会将系统状态的所有更改存储为事件序列，而不仅仅是当前状态，这种方式可以简化事务管理流程。

在微服务架构中，必须确保正确定义每个微服务的事务边界，来避免出现多个服务之间的数据更新不一致问题。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/21-How-Manage-Transactions-Distributed-Systems-Microservices/03.webp"/>

使用事务管理器也是很有必要的，它可以跨多个微服务协调分布式事务。

简而言之，管理分布式系统和微服务中的事务可能很具有挑战性，但是使用适当的事务管理技术，例如 `2PC`、基于补偿的协议、`Saga` 和事件溯源机制，可以帮助我们确保分布式系统中的数据一致性和完整性。

现在我们已经了解了一些基础知识，下面让我们更深入地学习这些技术。

### 什么是两阶段提交？它是如何管理分布式事务的？

两阶段提交（`2PC`）是一种分布式事务协议，用于管理跨多个分布式系统的事务。它确保参与分布式事务的所有节点都达成一致，提交或回滚事务。

在分布式事务中，协调者节点负责管理事务，并负责协调所有参与事务的节点。

**协调者向所有参与者发送准备消息，以确认他们做好准备提交事务**。如果所有参与者都已做好准备，协调者会发送提交消息，所有参与者都会提交事务。

另一方面，**如果某个参与者没有做好准备或者做出了否定响应，协调者会向所有参与者发送回滚消息**，所有参与者都会回滚事务。

两阶段提交协议确保即使在出现通信故障或系统崩溃时，事务也可以在分布式环境中正常提交或者回滚。两阶段提交协议被广泛应用于分布式数据库、分布式消息系统和其他分布式应用程序。

例如，考虑一个具有多个分支机构的银行系统，每个分支机构都有自己的数据库。当客户在一个分支机构执行交易时，需要与其他分支机构的数据库同步以保持数据一致性。

在这种情况下，我们就可以考虑使用两阶段提交协议来确保事务在所有分支机构的数据库上一致地进行提交或者回滚。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/21-How-Manage-Transactions-Distributed-Systems-Microservices/04.webp"/>

<br />

### 什么是微服务中的 SAGA 设计模式？它是如何管理分布式事务的？

`SAGA` 是 “[Saga Pattern](https://medium.com/javarevisited/what-is-saga-pattern-in-microservice-architecture-which-problem-does-it-solve-de45d7d01d2b)” 的缩写，是一种协助管理微服务架构中的分布式事务的设计模式。在微服务环境中，不同的服务可以拥有自己的数据库和独立的事务，`SAGA` 模式提供了一种确保跨多个服务的事务一致性的方法。

[SAGA](https://medium.com/javarevisited/what-is-saga-pattern-in-microservice-architecture-which-problem-does-it-solve-de45d7d01d2b) 将一个长时间运行的事务分解为多个较小的事务，每个事务都可以独立提交或者回滚。这些较小的事务由 `SAGA` 协调器进行编排，该协调器负责以正确的顺序执行程序并在发生故障时进行补偿操作。

`SAGA` 模式可以通过两种方式实现：`choreography-based` 和 `orchestration-based`。在 `choreography-based` 实现的 `SAGA` 系统中，每个服务负责自己的事务，并且通过服务之间的事件驱动通信来完成协调。

而在 `orchestration-based` 实现的 `SAGA` 系统中，是由 `SAGA` 协调器负责协调事务并控制 `SAGA` 的整个流程。

下图很好地解释了分布式微服务中的 `Saga` 模式：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/21-How-Manage-Transactions-Distributed-Systems-Microservices/05.webp"/>

<br />

### 什么是微服务中的事件溯源模式？它是如何管理分布式事务的？

事件溯源是微服务架构中使用的一种设计模式，它会将系统状态持久化为一系列事件，而不仅仅是系统的当前状态。

*它通过提供可靠且可扩展的方式来处理复杂的业务流程，帮助管理分布式事务。*

在事件溯源中，对系统状态所做的所有变更都将作为事件被捕获并保存到事件日志中。事件日志作为系统状态的真实来源，允许在需要时轻松回滚或重放事件。**这种方法确保对系统的所有变更都是可跟踪、可审计的，并且可以在必要时进行回滚**。

事件溯源的主要优点之一是它可以创建松散耦合的服务，这些服务可以彼此独立运行。每个服务都可以根据需要消费事件或者是生成事件，而不必关心其它服务的实现细节。

在管理分布式事务方面，**事件源提供了一种方法来确保事务中涉及的每个服务都可以可靠地提交或回滚变更内容**。

当事务涉及多个服务时，**每个服务可以发布一个事件来表明它所负责的事务是否已成功完成**。

然后，其它服务可以使用这些事件并执行相应的动作，确保在所有涉及的服务中一致地提交或回滚事务。

总的来说，事件溯源是一种在管理微服务架构中的分布式事务的强大模式，提供了一种可扩展且可靠的方式来处理复杂的业务流程。

下图很好地解释了事件溯源模式：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/21-How-Manage-Transactions-Distributed-Systems-Microservices/06.webp"/>

<br />

### Spring Cloud 如何管理分布式事务

`Spring Cloud` 提供了一些用于微服务架构中分布式事务管理的能力：

1. **服务注册和发现**
   `Spring Cloud` 所提供的服务注册和发现功能允许多个服务之间轻松地发现彼此的存在并进行通信；它通过提供服务注册和发现的中心化组件来帮助管理分布式事务。
2. **熔断**
   `Spring Cloud` 的熔断器模式用于防止分布式系统中出现级联故障；熔断器模式通过隔离特定服务中的故障，防止故障影响系统中的其它服务从而保障整个系统的可用性。
3. **分布式追踪**
   `Spring Cloud` 的分布式跟踪功能用于跟踪多个微服务之间的事务；这有助于识别事务中问题出现的根源从而轻松进行功能调试。
4. **配置管理**
   `Spring Cloud` 提供了一个集中的配置管理系统，用于管理跨多个微服务的配置；这使得系统可以轻松地实现更新和扩展，这有助于分布式系统中的事务管理。

总体而言，`Spring Cloud` 提供了一系列工具和功能，有助于管理微服务架构中的分布式事务，从而更轻松地维护系统的一致性和可靠性。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/21-How-Manage-Transactions-Distributed-Systems-Microservices/07.webp"/>

## 总结

以上内容就是如何在分布式系统中管理事务。总之，微服务和分布式系统中的事务管理是一项具有挑战性的任务，处理起来并不容易，但是对于确保整个系统的数据完整性和一致性却是至关重要的。

了解各种事务管理模式（例如两阶段提交、`SAGA` 和事件溯源）并为系统的特定场景选择适当的方法是至关重要的。每种模式都有其优点和缺点，选择某种模式应基于系统的特定要求和规范。

虽然**两阶段提交确保了一致性**，但是它**存在单点故障，并且性能可能会受到影响**；另一方面，**SAGA 和事件溯源提供了更好的可扩展性和可用性**，但需要额外的实现和维护工作。

因此，*对于开发人员和架构师来说，仔细分析和设计微服务和分布式系统中的事务管理功能是至关重要的，以确保系统性能最佳并保证数据完整性*。

伴随着微服务和分布式系统的不断普及，事务管理面临的挑战还会不断出现，新的模式和技术也将会不断涌现；紧跟该领域的最新发展对于开发高效、健壮的微服务和分布式系统是非常重要的。
