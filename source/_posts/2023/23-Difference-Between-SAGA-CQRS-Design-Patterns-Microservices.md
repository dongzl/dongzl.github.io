---
title: 微服务中 SAGA 和 CQRS 设计模式的差异？
date: 2023-08-27 15:57:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/SAGA_vs_CQRS.png

# author information, multiple authors are set to array
# single author
author:
- nick: Soma
  link: https://medium.com/@somasharma_81597
- nick: 董宗磊
  link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 一个大家不能错过的微服务热门面试题。

categories:
- 架构设计

tags:
- 微服务
- SAGA
- CQRS
---

> 原文链接：https://medium.com/javarevisited/difference-between-saga-and-cqrs-design-patterns-in-microservices-acd1729a6b02

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/23-Difference-Between-SAGA-CQRS-Design-Patterns-Microservices/01.webp"/>

朋友们大家好，如果大家不只是听说过微服务架构中的 `SAGA` 模式和 `CQRS` 模式，还想知道它们到底是什么以及何时使用它们，那么大家看这篇文章就对了。`SAGA` 和 `CQRS` 模式之间的差异也是 [Java 面试](https://medium.com/javarevisited/top-10-java-interview-questions-for-3-to-4-years-experienced-programmers-c4bf6d8b5e7b)中常见的问题之一。

`SAGA` 模式和 `CQRS` 模式是微服务架构中使用广泛的设计模式，但是它们有不同的使用场景。

`SAGA` 模式和 `CQRS` 模式之间的主要区别在于，`SAGA` 模式侧重于保证系统的**事务一致性**，而 `CQRS` 模式侧重于系统的**可扩展性和性能**，我们将在本文中分析更多有关它们的差异。

在前面的文章中，我在面向初学者和经验丰富的开发人员所分享的 [50 个微服务面试问题](https://medium.com/javarevisited/50-microservices-interview-questions-for-java-programmers-70a4a68c4349)和[10 个基本的微服务设计原则](https://medium.com/javarevisited/10-microservices-design-principles-every-developer-should-know-44f2f69e960f)等文章中都提到过 `SAGA` 设计模式；在这篇文章中，我将分享两者之间的差异以及何时在微服务架构中使用 `SAGA` 模式和 `CQRS` 模式。

<hr />

## 什么是微服务架构中的 SAGA 设计模式？

`SAGA` 设计模式常用于管理跨多个服务的事务，它能够确保事务中被涉及到的多个服务都正确地提交或回滚其所负责的事务部分，即使其它服务出现失败也是如此。

例如，在电子商务应用系统中，当用户发起在线下单时，这涉及支付处理、订单管理和商品配送等多种服务。

> SAGA 模式可用于确保如果这些服务中的任何一个出现失败，事务都会被回滚，用户的钱会被退还，不会受到损失。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/23-Difference-Between-SAGA-CQRS-Design-Patterns-Microservices/02.webp"/>

<hr />

## 什么是微服务中的 CQRS 模式？它能够解决什么问题？

`CQRS` （`Command Query Responsibility Segregation`）设计模式常用于将应用程序的读写操作分离到不同的模型中，它允许不同的模型针对各自的功能进行优化。

例如，考虑一个银行应用系统，用户可以查看账户余额、交易历史记录并进行转账。读取操作（例如查看账户余额）可以针对高可用性和低延迟进行优化，而写入操作（例如进行转账操作）可以针对数据一致性和准确性进行优化。
    
> 通过分离读写操作，我们可以为每个操作选择适当的存储模型和处理方式。

下面是微服务架构中 `CQRS` 模式的架构：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/23-Difference-Between-SAGA-CQRS-Design-Patterns-Microservices/03.webp"/>

综上所述，`SAGA` 模式和 `CQRS` 模式是微服务架构中两种适用于不同场景的设计模式。

`SAGA` 模式适用于管理跨多个服务的事务，而 `CQRS` 模式适用于将应用程序的读写操作分离到不同的模型中。

<hr />

## 微服务架构中 SAGA 和 CQRS 设计模式的区别？

现在我们已经熟悉了 `SAGA` 和 `CQRS` 模式之间的基本区别，下面让我们继续分析它们之间的其它差异。下面是微服务架构中 `SAGA` 模式和 `CQRS` 模式之间的对比分析：

### 1. 它们能够解决什么问题？

`SAGA` 模式是一种用于管理分布式事务顺序的设计模式，每个事务可能需要更新多个服务中的数据；而 `CQRS` 模式是一种分离系统中的读写操作的设计模式，每个操作使用独立的模型。

### 2. 功能

`SAGA` 模式用于在某个事务需要更新多个微服务数据时，确保跨多个微服务的数据一致性；而 `CQRS` 模式通过分离读写操作模型并分别优化来提高系统的可扩展性、性能和可用性。

### 3. 工作机制

`SAGA` 模式使用一系列补偿机制来协调事务，如果发生故障，这些补偿机制将回滚前一个事务所做的变更；另一方面，`CQRS` 模式使用不同的模型分离读写操作，写入模型针对数据更新进行优化，而读取模型针对查询和数据检索进行优化。

### 4. 一致性与可扩展性

`SAGA` 模式是一种在分布式系统中维护数据一致性的解决方案，数据会分布在多个微服务中；另一方面，`CQRS` 模式是一种通过分离读写操作并为每个操作使用不同模型来提升系统扩展性的解决方案。

### 5. 复杂性
   
`SAGA` 模式的实现和管理可能很复杂，因为它需要协调跨多个服务的多个事务；而 `CQRS` 模式可以通过分离读写操作，并为每个操作模型进行单独优化来简化复杂系统的管理工作。

### 6.使用场景

`SAGA` 模式最适合于在某个业务事务中，需要更新多个微服务数据的场景；而 `CQRS` 模式最适合于使用大量读写操作并且需要单独进行优化的场景。

`SAGA` 模式通常用于**需要强事务一致性**的系统，例如金融系统；而 `CQRS` 模式则用于需要高可扩展性和高性能的系统，例如社交媒体平台。

### 7. 实施

`SAGA` 模式可以使用 `choreography` 或者 `orchestration` 方法来实现，而 `CQRS` 模式通常使用 `event sourcing` 来实现。

这就是**微服务架构中 `SAGA` 模式和 `CQRS` 设计模式的区别**。`SAGA` 模式和 `CQRS` 模式之间的主要区别在于 `SAGA` 模式侧重于系统的**事务一致性**，而 `CQRS` 模式侧重于系统的*可扩展性*和*高性能*。

同样重要并且要注意的是，这两种模式可以在微服务架构中一起使用来实现不同的目的。

事实上，`SAGA` 模式和 `CQRS` 模式可以在微服务架构中一起使用，来实现强大的事务一致性以及可扩展性和高性能。但是，这可能会增加系统的额外复杂性，因此在使用 `SAGA` 模式或 `CQRS` 模式之前需要分析使用的场景和它们各自的优缺点。

顺便说一下，`SAGA` 模式和 `CQRS` 模式只是[众多常用的微服务设计模式](https://medium.com/javarevisited/top-10-microservice-design-patterns-for-experienced-developers-f4f5f782810e)中的两个，大家可以自己学习，我将来还会分享更多设计模式的文章。
