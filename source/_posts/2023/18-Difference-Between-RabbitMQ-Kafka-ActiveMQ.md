---
title: RabbitMQ、Apache Kafka 和 ActiveMQ 之间的差异
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
subtitle: 本文主要对比 RabbitMQ、Apache Kafka 和 ActiveMQ 之间的使用场景和差异。

categories:
- web开发

tags:
- RabbitMQ
- Kafka
- ActiveMQ
---

> 原文链接（请科学上网）：https://medium.com/javarevisited/difference-between-rabbitmq-apache-kafka-and-activemq-65e26b923114

朋友们大家好，如果大家正在准备 `Java` 开发人员面试以及 `Spring Boot` 和微服务面试，还应该准备消息代理、`Kafka`、`RabbitMQ` 和 `ActiveMQ` 等框架知识，比如 `Kafka`、`RabbitMQ` 和 `ActiveMQ` 之间有什么差异？这也是 `Java` 面试的热门问题之一。

在我的上一篇文章中，我分享了[JWT、OAuth 和 SAML 之间的差异](https://medium.com/javarevisited/difference-between-jwt-oauth-and-saml-for-authentication-and-authorization-in-web-apps-75b412754127) 和 [REST、GraphQL 和 gRPC 之间的差异](https://medium.com/javarevisited/difference-between-rest-graphql-and-grpc-10ac365462b8) ；在这篇文章中，我将分享我对 `Kafka`、`RabbitMQ` 和 `ActiveMQ` 这三个流行的[异步消息](https://medium.com/javarevisited/how-microservices-communicates-with-each-other-synchronous-vs-asynchronous-communication-pattern-31ca01027c53)代理框架的一些理解。

消息系统在现代分布式架构中，在通过网络相互通信的分布式应用程序和服务中扮演着至关重要的角色。消息系统可以使消息发送方和消息接收方解耦，从而实现[异步通信](https://medium.com/javarevisited/how-microservices-communicates-with-each-other-synchronous-vs-asynchronous-communication-pattern-31ca01027c53) 。`RabbitMQ`、`Apache Kafka` 和 `ActiveMQ` 是业界使用的三个流行消息队列框架。在本文中，我们将讨论 `RabbitMQ`、`Apache Kafka` 和 `ActiveMQ` 之间的差异。

顺便说一下，如果大家正在准备 `Java` 开发人员面试，还可以查看我之前发布的关于 [21 个软件设计模式问题](https://medium.com/javarevisited/21-software-design-pattern-interview-questions-and-answers-b7d1774b5dd2)、[10 个基于微服务场景的问题](https://medium.com/javarevisited/top-10-microservices-problem-solving-questions-for-5-to-10-years-experienced-developers-3391e4f6b591)、[20 个 SQL 查询面试题](https://medium.com/javarevisited/20-sql-queries-for-programming-interviews-a7b5a7ea8144)、[50 个微服务问题](https://medium.com/javarevisited/50-microservices-interview-questions-for-java-programmers-70a4a68c4349)、[60 个关于树的数据结构问题](https://medium.com/javarevisited/top-60-tree-data-structure-coding-interview-questions-every-programmer-should-solve-89c4dbda7c5a)、[15 个系统设计问题](https://medium.com/javarevisited/7-system-design-problems-to-crack-software-engineering-interviews-in-2023-13a518467c3e)、[35 个 Java 核心问题](https://medium.com/javarevisited/top-10-java-interview-questions-for-3-to-4-years-experienced-programmers-c4bf6d8b5e7b) 和 [21 个 Lambda 和 Stream 问题](https://medium.com/javarevisited/21-lambda-and-stream-interview-questions-for-java-programmers-38d7e83b5cac) 等文章，这些文章包含大量常见问题，可以帮助大家更好地准备面试。

<hr />

## 什么是 RabbitMQ，可以在什么地方使用它？

`RabbitMQ` 是一个实现高级消息队列协议 (`AMQP`) 标准的开源消息代理框架。它是用 `Erlang` 语言编写的，具备可插拔的体系结构，可以轻松进行扩展。

`RabbitMQ` 支持多种消息传输模式，例如**发布/订阅**、**请求/回复**和**点对点**模式，并且它具有一组强大的功能，例如**消息确认机制**、**消息路由**和**消息队列**。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/18-Difference-Between-RabbitMQ-Kafka-ActiveMQ/01.png"/>

<hr />

## 什么是 Apache Kafka，可以在什么地方使用它？

`Apache Kafka` 是一个开源分布式事件流平台，最初由 `LinkedIn` 公司开发。`Kafka` 是用 `Scala` 和 `Java` 语言编写的，旨在处理大规模流式数据。

`Kafka` 使用**发布/订阅消息传输模型**，并针对高吞吐量、低延迟和容错进行了优化。

`Kafka` 支持持久化的消息传递模型，这意味着消息可以存储在磁盘上并且可以多次重放。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/18-Difference-Between-RabbitMQ-Kafka-ActiveMQ/02.png"/>

<hr />

## 什么是 ActiveMQ，可以在什么地方使用它？

`Apache ActiveMQ` 是一个开源消息代理框架，它实现了 `Java` 消息服务（`JMS`）的 `API`。`ActiveMQ` 是用 `Java` 语言编写的，具有可插拔的体系结构，可以轻松实现扩展。

`ActiveMQ` 支持多种消息传输模式，例如**点对点**、**发布/订阅**和**请求/回复**模式，并且它具有一组强大的功能，例如**消息确认机制**、**消息路由**和**消息队列**。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/18-Difference-Between-RabbitMQ-Kafka-ActiveMQ/03.gif"/>

<hr />

## RabbitMQ、Apache Kafka 和 ActiveMQ 之间的差异

现在大家已经对 `RabbitMQ`、`ActiveMQ` 和 `Apache Kafka` 有了清晰地认识，是时候找出它们之间从消息传输模型到性能之间的差异了。以下是 `Apache Kafka`、`RabbitMQ` 和 `ActiveMQ` 之间的主要差异：

### 1. 消息模型

`RabbitMQ` 和 `ActiveMQ` 都支持 `JMS` 的 `API`，这意味着它们遵循传统的消息传输模型，消息被发送到队列或主题并由一个或多个消费者消费。

另一方面，*`Kafka` 使用发布/订阅消息传输模型*，将消息发布到主题并由一个或多个订阅者消费。

`RabbitMQ` 和 `ActiveMQ` 使用的是传统消息传输模型，非常适合需要严格排序和可靠消息传递的应用程序。

另一方面，`Kafka` 使用的发布/订阅消息模型更适合流式数据，需要实时处理数据的场景。

下图是一张很好的对比图，它突出了 `Kafka` 和 `RabbitMQ` 之间的架构差异：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/18-Difference-Between-RabbitMQ-Kafka-ActiveMQ/04.png"/>

<hr />

## 2. 可扩展性

可扩展性是消息传输系统的基本要求，尤其是在处理海量数据时。`RabbitMQ` 和 `ActiveMQ` 都具备可扩展性，但它们实现可扩展性的方法并不不同。

`RabbitMQ` 使用集群的方式来实现可扩展性，其中多个 `RabbitMQ` `broker` 彼此连接形成一个集群。消息分布在整个集群中，消费者可以连接到集群中的任意一个代理来消费消息。`RabbitMQ` 还支持联邦，允许多个 `RabbitMQ` 集群连接在一起。

`ActiveMQ` 使用网络代理方法来实现可扩展性，其中多个 `ActiveMQ` 代理连接形成一个网络。消息分布在整个网络中，消费者可以连接到网络中的任何代理节点来消费消息。`ActiveMQ` 还支持**主/从**复制，为消息代理节点提供高可用性。

另一方面，`Kafka` 被设计为具备开箱即用的高可扩展性。`Kafka` 使用分区方法来实现可扩展性，其中消息在多个 `Kafka` 代理之间进行分区，每个分区都被复制到多个代理节点中实现容错，这种方法允许 `Kafka` 处理海量数据，同时保证低延迟和高吞吐量性。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/18-Difference-Between-RabbitMQ-Kafka-ActiveMQ/05.png"/>

<hr />

### 3. 性能

性能是选择消息队列时需要考虑的另外一个关键因素，`RabbitMQ`、`Kafka` 和 `ActiveMQ` 都具备不同的性能特征。

`RabbitMQ` 被设计成一个可靠的消息传输系统，这意味着它优先考虑消息传输而不是性能。`RabbitMQ` 处理的消息速率适中，适用于需要严格排序和可靠传输消息的应用程序场景。

另一方面，`Kafka` 专为高性能而设计，可以在低延迟下处理海量数据。`Kafka` 使用分布式架构和优化顺序 `I/O` 来提高性能。

`ActiveMQ` 也是为高性能而设计的，可以以较高速率处理消息。`ActiveMQ` 通过使用异步架构和批处理方案来实现高性能。

这是来自 [confluent 的图表](https://www.confluent.io/blog/kafka-fastest-messaging-system/)，比较了 Apache Kafka、Pulsar 和 Rabbit MQ 的性能

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/18-Difference-Between-RabbitMQ-Kafka-ActiveMQ/06.png"/>

<hr />

### 4. 数据持久化

数据持久性是消息传递系统的一个重要特性，因为它允许在消息传递系统出现故障时存储和检索消息。 RabbitMQ、Kafka 和 ActiveMQ 都有不同的数据持久化方法。

RabbitMQ 默认情况下将消息存储在磁盘上，这允许即使代理关闭也可以持久保存消息。 RabbitMQ 还支持不同的存储后端，包括内存存储，它以数据持久性为代价提供更好的性能。

Kafka 默认将消息存储在磁盘上，并使用基于日志的架构来实现高持久性和可靠性。 Kafka 将消息保留一段可配置的时间，这允许在必要时重播消息。

ActiveMQ 还默认将消息存储在磁盘上，并支持不同的存储后端，包括 JDBC 和基于文件的存储。 ActiveMQ 可以将消息存储在数据库中，以性能为代价提供更好的数据持久性。

这是来自 IBM 的一张很好的图表，它显示了 Kafka 架构：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/18-Difference-Between-RabbitMQ-Kafka-ActiveMQ/07.png"/>

image — https://ibm-cloud-architecture.github.io/refarch-eda/technology/kafka-overview/

<hr />

### 5. 与其他系统集成

与其他系统的集成是选择消息系统时要考虑的重要因素。 RabbitMQ、Kafka 和 ActiveMQ 都具有不同的集成能力。

RabbitMQ 与不同的编程语言很好地集成，包括 Java、Python、Ruby 和 .NET。 RabbitMQ 也有插件，允许它与不同的系统集成，包括数据库、Web 服务器和消息代理。

Kafka 与不同的数据处理系统很好地集成，包括 Apache Spark、Apache Storm 和 Apache Flink。 Kafka 还有一个连接器框架，允许它与不同的数据库和数据源集成。

ActiveMQ 与不同的 JMS 客户端很好地集成，包括 Java、.NET 和 C。 ActiveMQ 还具有允许它与不同系统集成的插件，包括 Apache Camel 和 Apache CXF。

这里还有一个很好的表格来突出显示 Kafka、Rabbit MQ 和 ActiveMQ 之间的区别

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/18-Difference-Between-RabbitMQ-Kafka-ActiveMQ/08.png"/>

<hr />

## 总结

这就是 Apache Kafka、RabbitMQ 和 ActiveMQ 之间的区别。 RabbitMQ、Apache Kafka 和 ActiveMQ 是三种流行的消息传递系统，它们具有不同的特性和功能。

RabbitMQ 和 ActiveMQ 遵循传统的消息传递模型，而 Kafka 使用发布/订阅消息传递模型。 RabbitMQ 和 ActiveMQ 使用集群和代理网络方法来实现可扩展性，而 Kafka 使用分区。 RabbitMQ 优先考虑消息传递而不是性能，而 Kafka 和 ActiveMQ 优先考虑性能。 RabbitMQ、Kafka、ActiveMQ 都具有不同的数据持久化和集成能力。

选择消息系统时，必须考虑应用程序或系统的具体要求。 RabbitMQ 和 ActiveMQ 适用于要求消息严格排序和可靠传递的应用，而 Kafka 适用于流式数据场景。

> RabbitMQ和ActiveMQ适用于对消息速率要求中高的应用，而Kafka适用于对消息速率要求高的应用。

同样，RabbitMQ和ActiveMQ适用于对数据持久性要求高的应用，而Kafka适用于对性能要求高的应用。

这是我认为每个 Java 开发人员都应该准备的一个问题，但如果你想要更多，你还可以准备微服务问题，例如[API Gateway 和 Load Balancer 的区别](https://medium.com/javarevisited/difference-between-api-gateway-and-load-balancer-in-microservices-8c8b552a024)、[SAGA 模式](https://medium.com/javarevisited/what-is-saga-pattern-in-microservice-architecture-which-problem-does-it-solve-de45d7d01d2b)、[如何在微服务中管理事务](https://medium.com/javarevisited/how-to-manage-transactions-in-distributed-systems-and-microservices-d66ff26b405e) 以及 [SAGA 和 CQRS 模式的区别](https://medium.com/javarevisited/difference-between-saga-and-cqrs-design-patterns-in-microservices-acd1729a6b02)，它们在面试中很受欢迎。
