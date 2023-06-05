---
title: Hibernate、JPA 和 Spring Data JPA 之间的区别
date: 2023-06-03 12:33:45
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/hibernate_jpa.png

# author information, multiple authors are set to array
# single author
author:
- nick: Soma
  link: https://medium.com/@somasharma_81597
- nick: 董宗磊
  link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文主要介绍 Hibernate、JPA 和 Spring Data JPA 之间的区别。

categories:
- web开发

tags:
- Hibernate
- JPA
- Spring Data JPA
---

> 原文链接（请科学上网）：https://medium.com/javarevisited/difference-between-hibernate-jpa-and-spring-data-jpa-7df55717692f

朋友们大家好，如果大家正在准备 `Java` 开发岗位面试，那么除了准备 `Core Java`、`Spring Boot` 和微服务之外，我们还应该准备一些其他内容，比如 `ORM` 内容，`Hibernate`、`JPA` 和 `Spring Data JPA` 这些框架知识，例如 `Hibernate`、`JPA` 和 `Spring Data JPA` 之间的区别？这也是 `Java` 面试的热门问题之一。

在前面的文章中，我分享了[JWT、OAuth 和 SAML 之间的区别](https://medium.com/javarevisited/difference-between-jwt-oauth-and-saml-for-authentication-and-authorization-in-web-apps-75b412754127)、[Kafka 与 RabbitMQ之间的区别](https://medium.com/javarevisited/difference-between-rabbitmq-apache-kafka-and-activemq-65e26b923114) 以及 [REST、GraphQL 与 gRPC 之间的区别](https://medium.com/javarevisited/difference-between-rest-graphql-and-grpc-10ac365462b8) 这几篇文章；在本文中我将分享我对 `Hibernate`、`JPA` 和 `Spring Data JPA` 的理解，学习 `Java` 应用程序访问数据库的三个流行框架。

在开发与数据库交互的 `Java` 应用程序时，开发人员通常依赖某些框架和 `API` 来简化数据持久化的过程；虽然 `JDBC` 来自 `Java`，但是由于 `API` 的过度设计或者设计不足，`JDBC` 并不一定是最佳选择。

在 `Java` 中处理对象关系映射 (`ORM`) 的三个主流框架是 `Hibernate`、`JPA`（`Java Persistence API`）和 `Spring Data JPA`。虽然这些框架有一些相似之处，但是了解它们的特性差异、实现原理和优缺点是非常重要的。

在本文中，我们将学习 `Hibernate`、`JPA` 和 `Spring Data JPA` 之间的区别，将会探讨它们的定义、实现、持久性 `API`、数据库支持、事务管理、查询语言、缓存机制、配置选项、与 `Spring` 框架的集成以及其他功能特性。

通过了解这些区别，我们可以为项目选择最合适的框架，并在利用每种框架提供的特性时做出最佳的选择。

无论我们是开始使用 `ORM` 框架的 `Java` 开发人员，还是希望加强对 `Hibernate`、`JPA` 和 `Spring Data JPA` 的理解，本文旨在提供深入的研究和功能比较，来帮助大家应对 `Java` 应用程序开发中数据持久化的复杂性。

首先我们深入研究每个框架并揭示出使它们与众不同的独特功能。

顺便说一下，如果大家正在准备 `Java` 开发岗位的面试，还可以查看我之前发布的关于 [21 个软件设计模式问题](https://medium.com/javarevisited/21-software-design-pattern-interview-questions-and-answers-b7d1774b5dd2)、[10 个基于微服务场景的问题](https://medium.com/javarevisited/top-10-microservices-problem-solving-questions-for-5-to-10-years-experienced-developers-3391e4f6b591)、[20 个面试中常见的 SQL 查询](https://medium.com/javarevisited/20-sql-queries-for-programming-interviews-a7b5a7ea8144)、[50 个微服务问题](https://medium.com/javarevisited/50-microservices-interview-questions-for-java-programmers-70a4a68c4349)、[60 个关于树的数据结构问题](https://medium.com/javarevisited/top-60-tree-data-structure-coding-interview-questions-every-programmer-should-solve-89c4dbda7c5a)、[15 个系统问题](https://medium.com/javarevisited/7-system-design-problems-to-crack-software-engineering-interviews-in-2023-13a518467c3e)、[35 个 Java 核心问题](https://medium.com/javarevisited/top-10-java-interview-questions-for-3-to-4-years-experienced-programmers-c4bf6d8b5e7b) 和 [21 个 Lambda 和 Stream 问题](https://medium.com/javarevisited/21-lambda-and-stream-interview-questions-for-java-programmers-38d7e83b5cac)，其中包含大量常见问题，可以帮助大家更好地准备面试。

## 1. Hibernate：强大的 ORM 框架

与 `Spring` 框架一起，`Hibernate` 可能是最受 `Java` 开发人员欢迎的框架。`Hibernate` 是一个功能强大且广泛使用的 `ORM` 框架，它提供了一组广泛的功能，用于将 `Java` 对象映射到关系数据库。

它内部提供了 `JPA` 规范实现，使其成为管理 `Java` 应用程序数据持久化的综合解决方案。`Hibernate` 通过数据库方言支持各种数据库，允许开发人员无缝地迁移到各种不同的数据库系统。

借助 `Hibernate`，*`Java` 开发人员可以使用 `XML` 配置文件、注释或者基于 `Java` 的配置来定义对象的持久化映射*。它提供了自己的查询语言，称为 **Hibernate 查询语言（HQL）**，允许开发人员使用面向对象的语法编写数据库查询，从而更轻松地处理复杂的数据关系并执行高效的数据检索。

`Hibernate` 的主要优势之一是它对缓存机制的支持，它同时支持[一级和二级缓存](https://javarevisited.blogspot.com/2017/03/difference-between-first-and-second-level-cache-in-Hibernate.html)，可以通过减少数据库交互次数来显着提高操作数据库的性能。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/16-Difference-Between-Hibernate-JPA-Spring-Data-JPA/01.png"/>

<hr />

## 2. JPA：Java 持久化 API

另一方面，`JPA` 其实是一种**规范**，它为 `Java` 中的 `ORM` 定义了一组标准 `API`；它旨在为对象关系映射定义一套统一的且与具体框架无关的接口方法，但是 `JPA` 本身不提供接口实现，它需要依赖具体底层实现。

开发人员可以从多种 `JPA` 框架实现中进行选择，例如 `Hibernate`、`EclipseLink` 或 `Apache OpenJPA` 等，这些框架的实现遵循 `JPA` 规范并提供各自独特的功能，并进行内部优化。

`JPA` 的查询语言 `JPQL`（`Java Persistence Query Language`）类似于 `Hibernate` 的 `HQL`，允许开发人员使用基于实体的对象模型编写数据库查询，这种抽象简化了查询过程并促进了不同 `JPA` 框架实现之间的可移植性。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/16-Difference-Between-Hibernate-JPA-Spring-Data-JPA/02.png"/>

## 3. Spring Data JPA：简化 JPA 开发

`Spring Data JPA` 建立在 `JPA` 规范之上，并提供了一种简化且直观的方法来处理 `Spring` 应用程序中的数据持久化操作。它提供了**比 `JPA` 更高级别的抽象**，减少了样板代码并提供了便捷的功能，例如抽象存储和查询支持。

借助 `Spring Data JPA`，`Java` 开发人员可以将存储库定义为接口，利用 `Spring` 强大的依赖注入能力自动生成必要的 `CRUD`（创建、读取、更新、删除）操作，它还支持根据特定命名约定定义的方法名称来创建自定义查询，从而无需手工编写 `JPQL` 或 `SQL` 查询。

`Spring Data JPA` 与 `Spring` 生态系统无缝集成，允许开发人员利用其他 `Spring` 能力，例如事务管理、依赖项注入和声明式缓存。它还提供了一套内聚且高效的解决方案，用于在利用标准化 `JPA` `API` 的同时管理 `Spring` 应用程序中的持久性。

下图是一张很好的图表，它显示了 `Spring Data JPA` 与使用 `EntityManager` 的 `Raw JPA` 相比是如何工作的：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/16-Difference-Between-Hibernate-JPA-Spring-Data-JPA/03.png"/>

<hr />

## 何时使用 JPA、Hibernate 或 Spring Data JPA？

在 `Hibernate`、`JPA` 和 `Spring Data JPA` 之间做出选择时，我们需要考虑多种因素。`Hibernate` 提供了一套全面的功能，包括它定制的查询语言和缓存机制。 

`JPA` 提供了一种具有多种实现选择的**标准化方法**，提供了跨不同 `ORM` 框架的可移植性。另一方面，`Spring Data JPA` 简化了 `Spring` 生态系统中的 `JPA` 实现，提供了抽象存储和查询支持。

框架的选择取决于项目的具体要求、所需控制的层级和框架的灵活性，以及开发团队的熟悉程度和专业知识。**性能评估**、**数据库兼容性**、**易用性**以及与**现有 Spring 应用程序的集成**等因素有助于做出最佳的选择。

如果我们真正了解这些框架之间的差异，我们可以为项目选择正确的技术框架，这就是了解 `Hibernate`、`JPA` 和 `Spring Data JPA` 之间差异的原因，对于寻求高效解决方案来管理其应用程序数据持久化的 `Java` 开发人员来说这是至关重要。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/16-Difference-Between-Hibernate-JPA-Spring-Data-JPA/04.png"/>

<hr />

## Hibernate、JPA 和 Spring Data JPA 之间有什么区别？

现在我们已经对什么是 `Hibernate`、`JPA` 和 `Spring Data JPA` 以及各个框架的功能有了清晰的认识，是时候研究一下它们之间的具体区别了。以下是 `Hibernate`、`JPA` 和 `Spring Data JPA` 功能列表比较：

### 1. 定义

`Hibernate` 是具备对象关系映射能力的强大的 `ORM` 框架，虽然 `JPA` 是 `Java` 中的 `ORM` 规范，但是只定义了一组用于持久化的标准化 `API`，而 `Spring Data JPA` 提供了对 `JPA` 的简化抽象，提供了额外的能力和抽象存储。

### 2. 实现

`Hibernate` 提供自己的 `JPA` 规范实现，而使用 `JPA` 需要依赖具体实现，例如 `Hibernate`、`EclipseLink` 或 `OpenJPA`；`Spring Data JPA` 构建在 `JPA` 之上，需要符合 `JPA` 的实现，例如 `Hibernate` 或 `EclipseLink`。

### 3. 持久化 API

此外，`JPA` 为 `Java` 中的 `ORM` 定义了一组标准 `API`，而 `Hibernate` 为数据持久化操作提供了自己的 `API`，而 `Spring Data JPA` 是构建在 `JPA` `API` 之上并添加了额外的特性，例如抽象存储和查询支持。

### 4. 数据库支持

`JPA` 对数据库支持取决于所使用的 `JPA` 具体实现框架，框架可能具有特定的数据库支持；`Hibernate` 是通过数据库方言支持各种数据库，而 `Spring Data JPA` 也取决于所使用的 `JPA` 实现，提供与不同数据库的兼容性。

### 5. 事务管理

`Hibernate` 带有自己的事务管理功能，而 `JPA` 和 `Spring Data JPA` 都依赖于 `JPA` 具体实现框架的事务管理能力。

### 6. 查询语言

`Hibernate` 提供了用于编写面向对象查询的 `Hibernate` 查询语言（`HQL`），`JPA` 为数据库查询定义了 `Java` 持久性查询语言（`JPQL`），而 `Spring Data JPA` 使用 `JPQL` 作为查询语言，类似于 `JPA`。

### 7. 缓存

`Hibernate` 提供一级和二级缓存机制，`JPA` 同样依赖于所使用的 `JPA` 具体实现框架，它可能提供缓存功能，而 `Spring Data JPA` 同样依赖于 `JPA` 具体实现来完成对缓存的支持。

### 8. 配置

`Hibernate` 支持使用 `XML`、注释或基于 `Java` 的方法进行配置，`JPA` 同样支持使用 `XML`、注释或基于 `Java` 的方法进行配置，而 `Spring Data JPA` 也支持使用 `XML`、注释或基于 `Java` 的方法进行配置。

### 9. 一体化

`Hibernate` 可以独立使用，也可以与 `Spring` 框架集成使用；`JPA` 可以与任何符合 `JPA` 规范的框架一起使用，包括与 `Spring` 和 `Spring Data JPA` 的集成构成了 `Spring Data` 系列的一部分，旨在便于与 `Spring` 应用程序集成。

### 10. 其他特性

`Hibernate` 提供超出 `JPA` 规范的额外特性，而 `JPA` 仅提供一组标准化的 `API`；另一方面，`Spring Data JPA` 还在 `JPA` 之上提供额外的抽象存储和查询支持。

这个清单清晰地总结了 `Hibernate`、`JPA` 和 `Spring Data JPA` 之间每个功能的区别；但是如果我们想通过表格形式查看各个框架的区别，这里还有一个非常不错的总结表格，它突出显示了 `Hibernate`、`JPA` 和 `Spring Data JPA` 之间的关键技术差异：

| 特性           | Hibernate            | JPA                               | Spring Data JPA                         |
|--------------|----------------------|-----------------------------------|-----------------------------------------|
| 定义           | `ORM`（对象关系映射）框架      | `Java` 中 `ORM` 规范 | 基于 `JPA` 的简化和抽象 |
| 实现           | 自身提供实现 | 仅定义规范，依赖具体实现 | 构建在 `JPA` 之上，提供额外的特性 |
| 持久化 `API`    | `Hibernate` 提供内置 `API` | 为 `ORM` 定义一系列标准化 `API` | 构建在 `JPA` 的 `API` 之上，提供更多功能支持 |
| 数据库支持        | 通过数据库方言支持不同数据库 | 依赖于使用的具体 `JPA` 框架实现 | 依赖于使用的具体 `JPA` 框架实现 |
| 事务管理         | 提供内置的事务管理 | 依赖于使用的具体 `JPA` 框架实现 | 依赖于使用的具体 `JPA` 框架实现 |
| 查询语言         | `Hibernate` 查询语言（`HQL`） | `JPQL`（`Java` 持久化查询语言） | `JPQL`（`Java` 持久化查询语言） |
| 缓存           | 提供一级和二级缓存 | 依赖于使用的具体 `JPA` 框架实现 | 依赖于使用的具体 `JPA` 框架实现 |
| 配置           | `XML`、注解或者是基于 `Java` 配置 | `XML`、注解或者是基于 `Java` 配置 | `XML`、注解或者是基于 `Java` 配置 |
| 集成           | 可以独立使用或者与 `JPA` 一起使用 | 与任何符合 `JPA` 规范实现一起工作 | 与任何符合 `JPA` 规范实现一起工作 |
| `Spring` 集成  | 可以与 `Spring` 集成一起使用 | 可以与 `Spring` 集成一起使用 | `Spring Data` 家族的一款产品，可以与 `Spring` 集成一起使用 |
| 其他特性         | 提供额外功能特性，超过 `JPA` 规范 | 无 | 提供额外的抽象存储和查询支持 |

<hr />

## 如何在面试中处理问题？

当在 `Java` 开发岗位面试中遇到有关 `Hibernate`、`JPA` 和 `Spring Data JPA` 之间差异对比的问题时，提供全面准确的答案至关重要。我们可以考虑如下思路：

1. 首先解释 `Hibernate` 和 `JPA` 之间的关系，强调 `Hibernate` 不仅是 `JPA` 规范的实现，而且提供额外的特性和功能增强；
2. 阐明 `Hibernate` 是一个强大的 `ORM` 框架，它提供了 `JPA` 定义之外的高级特性；
3. 强调当使用 `Hibernate` 时，本质上是在使用 `JPA`，因为 `Hibernate` 完全支持 `JPA` 规范；
4. 继续探讨 `Spring Data JPA` 作为 `Spring Data` 项目的一部分，它简化了使用 `JPA` 对数据库地访问；
5. 突出 `Spring Data JPA` 的优点，例如抽象存储、查询方法和动态查询处理；
6. 解释 `Spring Data JPA` 是如何减少样板代码并简化数据库操作交互；
7. 讨论 `Hibernate` 提供的附加功能，包括缓存、延迟加载、`HQL` 和标准化 `API`；
8. 强调 `Spring Data JPA` 构建在 `JPA` 和 `Hibernate` 之上，提供更高级别的抽象和易用性；
9. 最后总结要点并重点强调理解这些差异对于高效开发应用程序的重要性。

在面试中，不仅要**展示这些技术的理论知识，还要展示实际工作经验和对如何在现实场景中使用这些框架给出一些理解**，这一点至关重要。根据特定的项目要求，举例说明何时以及为何选择使用其中某一个框架，可以进一步证明我们的理解。

<hr />

## 总结

这就是 `Hibernate`、`JPA` 和 `Spring Data JPA` 技术之间的区别。了解 `Hibernate`、`JPA` 和 `Spring Data JPA` 之间的差异对于面试准备和构建具有高效数据库交互的健壮 `Java` 应用程序至关重要。 

`Hibernate` 作为一个强大的 `ORM` 框架，实现了 `JPA` 规范并提供了额外的特性。`Spring Data JPA` 通过在 `JPA` 和 `Hibernate` 之上提供抽象存储和生成动态查询来简化数据库访问。

通过展示这些区别并突出实际用例，候选人可以用他们在使用这些技术方面的知识和专长给面试官留下深刻印象。

这是我认为每个 `Java` 开发人员都应该准备的一个问题，但如果你想要更多，你还可以准备微服务问题，例如 [API 网关和负载均衡器之间的区别](https://medium.com/javarevisited/difference-between-api-gateway-and-load-balancer-in-microservices-8c8b552a024)、[SAGA 模式](https://medium.com/javarevisited/what-is-saga-pattern-in-microservice-architecture-which-problem-does-it-solve-de45d7d01d2b)、[如何在微服务中管理事务](https://medium.com/javarevisited/how-to-manage-transactions-in-distributed-systems-and-microservices-d66ff26b405e)以及 [SAGA 和 CQRS 模式之间的区别](https://medium.com/javarevisited/difference-between-saga-and-cqrs-design-patterns-in-microservices-acd1729a6b02), 他们在采访中很受欢迎。
