---
title: （进行中）Difference between Hibernate, JPA, and Spring Data JPA?
date: 2013-06-03 12:33:45
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/hibernate_jpa.png

# author information, multiple authors are set to array
# single author
author:
- nick: Soma
  link: https://medium.com/@somasharma_81597
- nick: 董宗磊
  link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文将通过 Istio、eBPF 和 RSocket Broker 技术深入探索 Service Mesh 解决方案。

categories:
- 架构设计

tags:
- Hibernate
- JPA
- Spring Data JPA
---

> 原文链接（请科学上网）：https://medium.com/javarevisited/difference-between-hibernate-jpa-and-spring-data-jpa-7df55717692f

大家好，如果您正在准备 Java 开发人员面试，那么除了准备 Core Java、Spring Boot 和微服务之外，您还应该准备 ORM 框架、Hibernate、JPA 和 Spring Data JPA 之类的东西，比如 Hibernate、JPA 之间的区别, 和 Spring Data JPA?，这也是 Java 面试的热门问题之一。

在上一篇文章中，我分享了[JWT、OAuth 和 SAML](https://medium.com/javarevisited/difference-between-jwt-oauth-and-saml-for-authentication-and-authorization-in-web-apps-75b412754127)、[Kafka 与 RabbitMQ](https://medium.com/javarevisited/difference-between-rabbitmq-apache-kafka-and-activemq-65e26b923114) 以及 [REST 与 GraphQL 与 gRPC](https://medium.com/javarevisited/difference-between-rest-graphql-and-grpc-10ac365462b8) 之间的区别，在本文中，我将分享我对 Hibernate、JPA 和 Spring Data JPA 的看法，从 Java 应用程序访问数据库的三个流行框架。

在开发与数据库交互的 Java 应用程序时，开发人员通常依赖框架和 API 来简化和简化使用持久层的过程。尽管 JDBC 来自 Java，但由于 API 的精心设计和设计不佳，它并不总是最佳选择。

在 Java 中管理对象关系映射 (ORM) 的三个流行选项是 Hibernate、JPA（Java Persistence API）和 Spring Data JPA。虽然这些框架有一些相似之处，但了解它们的独特特性、实现和优势至关重要。

在本文中，您将了解 Hibernate、JPA 和 Spring Data JPA 之间的区别。我们将探讨它们的定义、实现、持久性 API、数据库支持、事务管理、查询语言、缓存机制、配置选项、与 Spring 的集成以及其他功能。

通过了解这些区别，您可以在为项目选择最合适的框架并利用每个选项提供的功能时做出明智的决定。

无论您是开始使用 ORM 框架的 Java 开发人员，还是希望加强对 Hibernate、JPA 和 Spring Data JPA 的理解的人，本文旨在提供有价值的见解和比较，以帮助您应对 Java 应用程序中管理持久性的复杂性。

让我们首先详细检查每个框架并揭示使它们与众不同的独特特征。

顺便说一下，如果您正在准备 Java 开发人员面试，您还可以查看我之前发布的关于 [21 个软件设计模式问题](https://medium.com/javarevisited/21-software-design-pattern-interview-questions-and-answers-b7d1774b5dd2)、[10 个基于微服务场景的问题](https://medium.com/javarevisited/top-10-microservices-problem-solving-questions-for-5-to-10-years-experienced-developers-3391e4f6b591)、20 个来自面试的 SQL 查询、50 个微服务问题、60 个树数据结构问题、15 个系统问题Design Questions，以及 35 道 Core Java Questions 和 21 道 Lambda and Stream questions，其中包含大量常见问题，可以帮助您更好地准备面试。

而且，如果您还不是 Medium 会员，那么我强烈建议您加入 Medium 并阅读我的其他会员专用文章以准备面试。您可以在这里加入 Medium

## 1. Hibernate：强大的 ORM 框架

与 Spring 框架一起，Hibernate 可能是最受 Java 开发人员欢迎的框架。 Hibernate 是一个功能强大且广泛使用的 ORM 框架，它提供了一组广泛的功能，用于将 Java 对象映射到关系数据库。

它提供自己的 JPA 规范实现，使其成为管理 Java 应用程序持久性的综合解决方案。 Hibernate 通过其方言支持各种数据库，允许开发人员无缝地使用不同的数据库系统。

借助 Hibernate，Java 开发人员可以使用 XML 配置文件、注释甚至基于 Java 的配置来定义他们的持久性映射。它提供了自己的查询语言，称为 Hibernate 查询语言 (HQL)，允许开发人员使用面向对象的语法编写数据库查询，从而更轻松地处理复杂的关系并执行高效的数据检索。

Hibernate 的主要优势之一是它对缓存机制的支持。它同时提供[一级和二级缓存](https://javarevisited.blogspot.com/2017/03/difference-between-first-and-second-level-cache-in-Hibernate.html)，可以通过减少数据库往返次数来显着提高数据库操作的性能。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/16-Difference-Between-Hibernate-JPA-Spring-Data-JPA/01.png"/>

<hr />

## 2. JPA：Java 持久化 API

另一方面，JPA 是一种规范，它为 Java 中的 ORM 定义了一组标准 API。它旨在为对象关系映射提供一致且与供应商无关的方法。但是，JPA 本身不提供实现；它需要使用底层实现。

开发人员可以从各种 JPA 实现中进行选择，例如 Hibernate、EclipseLink 或 Apache OpenJPA 等。这些实现遵循 JPA 规范并提供它们自己独特的功能和优化。

JPA 的查询语言 JPQL（Java Persistence Query Language）类似于 Hibernate 的 HQL，允许开发人员使用基于实体的对象模型编写数据库查询。这种抽象简化了查询过程并促进了不同 JPA 实现之间的可移植性。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/16-Difference-Between-Hibernate-JPA-Spring-Data-JPA/02.png"/>

## 3. Spring Data JPA：简化 JPA 开发

Spring Data JPA 建立在 JPA 规范之上，并提供了一种简化且直观的方法来处理 Spring 应用程序中的持久层。它提供了比 JPA 更高级别的抽象，减少了样板代码并提供了便利的功能，例如存储库抽象和查询支持。

借助 Spring Data JPA，Java 开发人员可以将存储库定义为接口，利用 Spring 强大的依赖注入功能自动生成必要的 CRUD（创建、读取、更新、删除）操作。它还支持通过根据特定命名约定定义方法名称来创建自定义查询，从而无需手动编写 JPQL 或 SQL 查询。

Spring Data JPA 与 Spring 生态系统无缝集成，允许开发人员利用其他 Spring 功能，例如事务管理、依赖项注入和声明式缓存。它还提供了一个内聚且高效的解决方案，用于在利用标准化 JPA API 的同时管理 Spring 应用程序中的持久性。

这是一个很好的图表，它显示了 Spring Data JPA 与使用 EntityManager 的 Raw JPA 相比如何工作：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/16-Difference-Between-Hibernate-JPA-Spring-Data-JPA/03.png"/>

<hr />

## 何时使用 JPA、Hibernate 或 Spring Data JPA？

在 Hibernate、JPA 和 Spring Data JPA 之间做出选择时，您需要考虑各种因素。 Hibernate 提供了一套全面的功能，包括它自己的查询语言和缓存机制。 

JPA 提供了一种具有多种实现选择的标准化方法，提供了跨不同 ORM 框架的可移植性。另一方面，Spring Data JPA 简化了 Spring 生态系统中的 JPA 开发，提供了存储库抽象和查询支持。

框架的选择取决于项目的具体要求、所需的控制和灵活性级别以及开发团队的熟悉程度和专业知识。评估性能、数据库兼容性、易用性以及与现有 Spring 应用程序的集成等因素有助于做出明智的决策。

如果您真正了解这些差异，那么您只能为您的项目选择正确的技术，这就是了解 Hibernate、JPA 和 Spring 之间差异的原因 Data JPA 对于寻求有效解决方案来管理其应用程序持久性的 Java 开发人员来说至关重要。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/16-Difference-Between-Hibernate-JPA-Spring-Data-JPA/04.png"/>

<hr />

## Hibernate、JPA 和 Spring Data JPA 之间有什么区别？

现在我们已经对什么是 Hibernate、JPA 和 Spring Data JPA 及其功能有了清晰的认识，是时候看看它们之间的真正区别了。以下是 Hibernate、JPA 和 Spring Data JPA 以功能列表格式的比较：

### 1. 定义

Hibernate 是提供对象关系映射功能的 Robust ORM 框架。虽然 JPA 是 Java 中的 ORM 规范，但定义了一组用于持久化的标准 API，而 Spring Data JPA 提供了对 JPA 的简化抽象，提供了额外的功能和存储库抽象。

### 2. 实现

Hibernate 提供自己的 JPA 规范实现，而 JPA 需要使用底层实现，例如 Hibernate、EclipseLink 或 OpenJPA。 Spring Data JPA 构建在 JPA 之上，需要符合 JPA 的实现，例如 Hibernate 或 EclipseLink。

### 3. 持久化 API

同样，JPA 为 Java 中的 ORM 定义了一组标准 API，而 Hibernate 为持久性操作提供了自己的 API，而 Spring Data JPA 构建在 JPA API 之上并添加了额外的功能，例如存储库抽象和查询支持。

### 4. 数据库支持

如果 JPA 数据库支持取决于所使用的 JPA 实现，它可能具有特定的数据库支持。 Hibernate 确实通过其方言支持各种数据库，而 Spring Data JPA 也取决于所使用的 JPA 实现，提供与不同数据库的兼容性。

### 5. 事务管理

Hibernate 带有自己的事务管理功能，而 JPA 和 Spring Data JPA 都依赖于用于事务管理的 JPA 实现。

### 6. 查询语言

Hibernate 提供了用于编写面向对象查询的 Hibernate 查询语言 (HQL)。 JPA 为数据库查询定义了 Java 持久性查询语言 (JPQL)，而 Spring Data JPA 使用 JPQL 作为查询语言，类似于 JPA。

### 7. 缓存

Hibernate 提供一级和二级缓存机制。虽然 JPA 再次依赖于所使用的 JPA 实现，它可能提供缓存功能，而 Spring Data JPA 依赖于用于缓存支持的 JPA 实现。

### 8. 配置

Hibernate 支持使用 XML、注释或基于 Java 的方法进行配置。 JPA 还支持使用 XML、注释或基于 Java 的方法进行配置，而 Spring Data JPA 支持使用 XML、注释或基于 Java 的方法进行配置。

### 9. 一体化

Hibernate可以独立使用，也可以与Spring集成使用。 JPA 与任何符合 JPA 的实现一起工作，包括与 Spring 和 Spring Data 的集成 JPA 是 Spring Data 系列的一部分，旨在与 Spring 应用程序集成。

### 10， 其他特性

Hibernate 提供超出 JPA 指定的额外功能，而 JPA 仅提供一组标准化的 API。另一方面，Spring Data JPA 还在 JPA 之上提供额外的存储库抽象和查询支持。

此列表清楚地总结了 Hibernate、JPA 和 Spring Data JPA 之间每个功能的区别，但是，如果您想查看表格格式的区别，这里有一个很好的表格，它突出显示了 Hibernate、JPA 和 Spring Data JPA 之间的关键区别技术：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/16-Difference-Between-Hibernate-JPA-Spring-Data-JPA/05.png"/>

<hr />

## 如何在面试中处理问题？

当在 Java 开发人员面试中遇到有关 Hibernate、JPA 和 Spring Data JPA 之间差异的问题时，提供全面准确的答案至关重要。您可以考虑以下方法：

1. 首先解释 Hibernate 和 JPA 之间的关系，强调 Hibernate 是 J PA 规范的实现，提供额外的特性和增强。
2. 阐明 Hibernate 是一个强大的 ORM 框架，它提供了 JPA 定义之外的高级功能。 
3. 强调当使用Hibernate时，你本质上是在使用JPA，因为Hibernate完全支持JPA规范。 
4. 继续讨论 Spring Data JPA 作为 Spring Data 项目的一部分，它简化了使用 JPA 的数据库访问。 
5. 5.突出Spring Data JPA的优点，例如存储库抽象、查询方法和动态查询生成。 
6. 解释 Spring Data JPA 如何减少样板代码并简化数据库交互。 
7. 讨论 Hibernate 提供的附加功能，包括缓存、延迟加载、HQL 和 Criteria API。 
8. 强调 Spring Data JPA 建立在 JPA 和 Hibernate 之上，提供更高级别的抽象和便利。 
9. 最后总结要点并重申理解这些差异对于有效应用程序开发的重要性。

在面试中，不仅要展示这些技术的理论知识，还要展示实际经验和对如何在现实场景中使用它们的理解，这一点至关重要。根据特定的项目要求，举例说明您何时以及为何选择其中一个，可以进一步证明您的理解。 

<hr />

## 总结

这就是 Hibernate、JPA 和 Spring Data JPA 技术之间的区别。了解 Hibernate、JPA 和 Spring Data JPA 之间的差异对于面试准备和构建具有高效数据库交互的健壮 Java 应用程序至关重要。 

Hibernate 作为一个强大的 ORM 框架，实现了 JPA 规范并提供了额外的特性。 Spring Data JPA 通过在 JPA 和 Hibernate 之上提供存储库抽象和动态查询生成来简化数据库访问。

通过展示这些区别并突出实际用例，候选人可以用他们在使用这些技术方面的知识和专长给面试官留下深刻印象。

这是我认为每个 Java 开发人员都应该准备的一个问题，但如果你想要更多，你还可以准备微服务问题，例如 [API 网关和负载均衡器之间的区别](https://medium.com/javarevisited/difference-between-api-gateway-and-load-balancer-in-microservices-8c8b552a024)、[SAGA 模式](https://medium.com/javarevisited/what-is-saga-pattern-in-microservice-architecture-which-problem-does-it-solve-de45d7d01d2b)、[如何在微服务中管理事务](https://medium.com/javarevisited/how-to-manage-transactions-in-distributed-systems-and-microservices-d66ff26b405e)以及 [SAGA 和 CQRS 模式之间的区别](https://medium.com/javarevisited/difference-between-saga-and-cqrs-design-patterns-in-microservices-acd1729a6b02), 他们在采访中很受欢迎。

而且，如果您不是 Medium 会员，那么我强烈建议您加入 Medium 并阅读来自真实领域的伟大作者的精彩故事。您可以在这里加入 Medium 