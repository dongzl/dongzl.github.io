---
title: REST、GraphQL 和 gRPC 之间的差异
date: 2023-06-16 15:14:51
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/RabbitMQ_Kafka_ActiveMQ.png

# author information, multiple authors are set to array
# single author
author:
- nick: Soma
  link: https://medium.com/@somasharma_81597
- nick: 董宗磊
  link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨在帮助读者了解 REST、GraphQL 和 gRPC（微服务和 Web 应用程序中客户端服务器通信的三种主要协议）之间的主要区别。

categories:
- web开发

tags:
- REST
- GraphQL
- gRPC
---

> 原文链接（请科学上网）：https://medium.com/javarevisited/difference-between-rest-graphql-and-grpc-10ac365462b8

朋友们大家好，如果大家正在通过学习 `Spring Boot` 和微服务知识来准备 `Java` 开发岗位的面试，我建议还应该准备一下 `REST`、`GraphQL` 和 `gRPC` 等内容，例如 **`REST`、`GraphQL` 和 `gRPC` 之间的差异**，这也是 `Java` 面试中的热门问题之一。

在上一篇文章中，我分享了 [JWT、OAuth 和 SAML 之间的差异](https://medium.com/javarevisited/difference-between-jwt-oauth-and-saml-for-authentication-and-authorization-in-web-apps-75b412754127)；在本文中我将分享我对 `REST`、`GraphQL` 和 `gRPC` 这三种流行的用于构建 `Web` `API` 的通信协议的一些思考。

它们通常用于不同的系统组件通过网络进行相互通信，例如[使用 REST 可以在微服务之间进行同步通信](https://medium.com/javarevisited/how-microservices-communicates-with-each-other-synchronous-vs-asynchronous-communication-pattern-31ca01027c53)。这些协议都有各自的优缺点，了解它们之间的差异不仅对于面试很重要，而且对于实际的项目中选择正确的协议也很重要。

在本文中，我们将了解 `REST`、`GraphQL` 和 `gRPC` 之间的差异。我将解释每个协议背后的核心概念、它们的优点和缺点，并提供一些使用每种协议的实际场景。在本文结束时，我们应该能够更好地理解每种协议最能够满足我们的项目要求。

我们首先从一些简单的介绍开始，然后将会深入研究每一种协议，然后再总结它们之间的差异，以便大家清晰地了解它们的优缺点以及何时使用它们。

[REST](https://en.wikipedia.org/wiki/Representational_state_transfer) 即表述性状态转移，它是一种流行的协议，通常用于创建 `Web` 服务，这些服务通过 `HTTP` 公开数据和开放功能。它基于 `HTTP` 协议和一组约束，用来定义如何识别和定位某个资源以及如何对这些资源执行相关操作。

另一方面，[GraphQL](https://graphql.org/) 是 `Facebook` 开发的 `API` 查询语言。它允许客户端准确指定他们需要的数据，并且服务器仅响应该数据。

`GraphQL` 的出现是为了解决 `REST` 的缺点和限制，因此它提供了一种更灵活、更高效的从服务器获取数据的方式，因为客户端可以在单个请求中请求多个资源。

[gRPC](https://grpc.io/) 是一种用于创建 `API` 的高性能开源协议。它使用 **`Google` 的 `Protocol Buffers`** 作为数据格式，并提供对数据流和双向通信的支持。`gRPC` 因其性能和对多种编程语言的支持，经常用于微服务架构。

现在我们知道它们是什么，接下来我们深入研究它们。

顺便说一下，如果大家正在准备 `Java` 开发岗位的面试，还可以查看我之前发布的关于 [21 个软件设计模式问题](https://medium.com/javarevisited/21-software-design-pattern-interview-questions-and-answers-b7d1774b5dd2)、[10 个基于微服务场景的问题](https://medium.com/javarevisited/top-10-microservices-problem-solving-questions-for-5-to-10-years-experienced-developers-3391e4f6b591)、[20 个 SQL 查询面试题](https://medium.com/javarevisited/20-sql-queries-for-programming-interviews-a7b5a7ea8144)、[50 个微服务问题](https://medium.com/javarevisited/50-microservices-interview-questions-for-java-programmers-70a4a68c4349)、[60 个关于树的数据结构问题](https://medium.com/javarevisited/top-60-tree-data-structure-coding-interview-questions-every-programmer-should-solve-89c4dbda7c5a)、[15 个系统设计问题](https://medium.com/javarevisited/7-system-design-problems-to-crack-software-engineering-interviews-in-2023-13a518467c3e)、[35 个 Java 核心问题](https://medium.com/javarevisited/top-10-java-interview-questions-for-3-to-4-years-experienced-programmers-c4bf6d8b5e7b) 和 [21 个 Lambda 和 Stream 问题](https://medium.com/javarevisited/21-lambda-and-stream-interview-questions-for-java-programmers-38d7e83b5cac) 等文章，这些文章包含大量常见问题，可以帮助大家更好地准备面试。

## 什么是 REST？什么时候使用它？

正如我所说，REST（表述性状态转移）是一种用于设计分布式应用程序的架构风格，尤其是基于 Web 的 API。 RESTful API 使用 HTTP 方法（例如 GET、POST、PUT、DELETE）对 URL（统一资源定位器）标识的资源执行 CRUD（创建、读取、更新、删除）操作。

如果您了解 HTTP，那么您就了解 REST。

REST 还依赖于无状态的客户端-服务器架构，其中来自客户端的每个请求都包含服务器完成请求所需的所有信息，而无需维护会话状态。

以下是 REST 是不错选择的一些场景：

1. 当您需要通过 API 公开数据和服务时，因为 REST 是一种流行且完善的协议，用于创建可供其他应用程序和服务轻松使用的 API。
2. 当您因为 REST 依赖于标准的 HTTP 方法和数据格式而需要支持多种平台和编程语言时，它可以被多种编程语言和平台使用。
3. 当您需要支持缓存时，因为REST支持缓存，这可以提高性能并减少网络流量。
4. 当您需要构建简单、轻量级的 API 时
5. 当您需要支持大量资源时

此外，了解 HTTP 方法对于设计 REST API 非常重要

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/19-Difference-Between-REST-GraphQL-gRPC/01.png"/>

总体而言，REST 是一种灵活且广泛采用的协议，对于许多类型的 API 来说都是不错的选择。但是，它可能不是所有场景的最佳选择，尤其是那些需要实时更新或更复杂的查询和数据操作的场景。在这些情况下，其他协议（例如 GraphQL 或 gRPC）可能更合适。

<hr />

## 什么是 GraphQL？什么时候使用它？

GraphQL 是一种 API 查询语言，由 Facebook 于 2012 年开发，并于 2015 年作为开源项目发布。它最初是为了解决 REST 的限制和缺点而创建的。

GraphQL 允许客户端定义他们需要的数据的结构，并且服务器可以准确地响应该数据，而没有任何不必要的数据。它通常用作 RESTful API 的替代方案，特别是在客户端需要对返回的数据进行细粒度控制的情况下。

以下是 GraphQL 是不错选择的一些场景：

1. 当您想要减少网络流量时，因为 GraphQL 允许客户端准确指定他们需要的数据，这可以减少通过网络传输的不必要的数据量。
2. 当您需要支持各种客户端时，因为 GraphQL 支持强类型查询，这可用于确保客户端以他们理解的格式接收正确的数据。
3. **当您需要支持实时更新**时，因为 GraphQL 支持通过订阅进行实时更新，这允许客户端在更新可用时立即接收更新。
4. 当您需要支持复杂的查询和数据操作时：因为GraphQL允许客户端使用简单的语法执行复杂的查询和数据操作操作，例如过滤、排序和聚合。
5. 当您需要支持版本控制时，因为 GraphQL 通过允许客户端指定他们在请求中使用的模式版本来支持版本控制，这可以让您在模式随着时间的推移而发展时更轻松地保持向后兼容性。

总的来说，GraphQL 是一个强大而灵活的协议，对于数据细粒度控制和实时更新很重要的场景来说，它是一个不错的选择。然而，**它可能比 RESTful API 需要更多的设置和配置**，特别是当您使用多种编程语言或平台时。

这也是一个很好的图表，突出**显示了 REST 和 GraphQL 查询**之间的区别：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/19-Difference-Between-REST-GraphQL-gRPC/02.png"/>

<hr />

## 什么是 gRPC？什么时候使用它？

现在让我们看看什么是 gRPC 以及它提供什么？ gRPC 是 Google 开发的一个高性能、开源的远程过程调用 (RPC) 框架。它使用Protocol Buffers作为接口描述语言，支持多种编程语言，可以轻松构建跨不同平台和环境的分布式系统。

以下是 gRPC 是不错选择的一些场景：

1. **当您需要高性能和高效率时**，因为 gRPC 使用二进制协议并支持流式传输，这可以使其比其他协议更快、更高效，特别是在高延迟或低带宽连接上。
2. 当您需要支持多种编程语言时，因为 gRPC 支持多种编程语言，包括 Java、C、Python 和 Go，可以轻松构建跨不同平台和环境的分布式系统。
3. 当您需要支持实时更新时，因为 gRPC 支持双向流，这允许服务器实时向客户端发送更新。
4. **当您需要处理大量数据时**，因为 gRPC 使用 Protocol Buffers，它比 JSON 或 XML 等其他数据格式更高效、更紧凑，使其成为处理大量数据的不错选择。
5. 当您需要构建微服务或分布式系统时，因为 gRPC 提供了强大而灵活的框架，用于构建可以水平扩展并处理大量流量的微服务和分布式系统。

总体而言，gRPC 是一个强大且高效的协议，对于性能、效率和实时更新很重要的场景来说，它是一个不错的选择。但是，与 RESTful API 等其他协议相比，它可能需要更多的设置和配置，特别是当您使用多种编程语言或平台时。

这是一个很好的图表，突出显示了 REST、gRPC 和 GraphQL 请求之间的区别

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/19-Difference-Between-REST-GraphQL-gRPC/03.png"/>

<font color=DarkGray size=2>image_credit — [https://medium.com/@LadyNoBug/grpc-v-s-rest-v-s-others-5d8b6eaa61df](https://medium.com/@LadyNoBug/grpc-v-s-rest-v-s-others-5d8b6eaa61df)</font>

<hr />

## REST、gRPC 和 GraphQL 之间的区别

现在您已经了解什么是 REST、gRPC 和 GraphQL 以及它们的工作原理，以下是 REST、GraphQL 和 gRPC 之间的主要区别（以点格式表示），记住它们的主要特征以及何时在项目中使用它们：

### REST：

- 代表代表性状态转移
- 使用 HTTP 方法（GET、POST、PUT、DELETE）执行 CRUD 操作
- 以结构化格式发送数据，通常是 JSON 或 XML
- 不同资源可以有多个端点
- 客户端收到响应中指定的所有数据，即使他们不需要全部数据
- 支持缓存，但管理起来可能很复杂
- 完善且广泛采用，提供大量工具和文档

### GraphQL：

- 允许客户准确指定他们需要什么数据，并仅接收该数据
- 使用单个端点访问多个资源
- 拥有自己的查询语言，允许复杂的数据获取和操作
- 可以支持通过订阅实时更新
- 在某些情况下比 REST 更高效，特别是对于带宽有限的移动设备
- 与 REST 相比，缓存可以更细粒度且更易于管理
- 比 REST 需要更多的设置和配置，并且可能需要更多的专业知识才能有效使用

### gRPC：
- 代表带有 Google 协议缓冲区的远程过程调用 (RPC)；
- 使用二进制数据代替 HTTP 进行通信；
- 支持流数据实时更新；
- 使用协议缓冲区进行序列化，这比 JSON 或 XML 更高效
- 可以跨不同的编程语言使用
- 专为微服务之间的高性能、低延迟通信而设计
- 比 REST 需要更多的设置和配置，并且可能需要更多的专业知识才能有效使用
- 互操作性可能不如 REST 或 GraphQL，因为它不基于 HTTP

这里还有一个很好的表格，突出显示了 REST、GraphQL 和 REST 之间的区别，您可以使用它来快速复习：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/19-Difference-Between-REST-GraphQL-gRPC/04.png"/>

还值得注意的是，**这些协议并不相互排斥，并且可以组合使用它们以利用它们的不同优势**。

例如，您可能对大多数 API 使用 REST，但对某些资源密集型查询使用 GraphQL，或者使用 gRPC 在微服务之间进行通信，同时对外部 API 客户端使用 REST 或 GraphQL。

<hr />

## 总结

这就是 REST、GraphQL 和 gRPC 技术之间的差异。简而言之，REST 是一种用于创建 Web 服务的流行协议，受到 HTTP 的启发并充分利用 HTTP 提供的功能，而 GraphQL 是一种查询语言，允许客户端准确指定他们需要从服务器获取哪些数据。它的创建是为了解决 REST 的缺点，因此如果您正在努力维护 REST API，那么它绝对是一个可行的选择。

另一方面，gRPC 是一种高性能、开源协议，常用于微服务架构中。这些协议中的每一个都有不同的用途，并且它们都可以一起使用，为 Web 应用程序提供全面且高效的通信系统。

我相信这是每个​​ Java 开发人员都应该准备的问题，但如果您想要更多，您还可以准备微服务问题，例如 [API 网关和负载均衡器之间的区别](https://medium.com/javarevisited/difference-between-api-gateway-and-load-balancer-in-microservices-8c8b552a024)、[SAGA 模式](https://medium.com/javarevisited/what-is-saga-pattern-in-microservice-architecture-which-problem-does-it-solve-de45d7d01d2b)、[如何管理微服务中的事务](https://medium.com/javarevisited/how-to-manage-transactions-in-distributed-systems-and-microservices-d66ff26b405e)以及 [SAGA 和 CQRS 模式之间的区别](https://medium.com/javarevisited/difference-between-saga-and-cqrs-design-patterns-in-microservices-acd1729a6b02)，他们在采访中很受欢迎。