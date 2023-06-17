---
title: （进行中）Difference between RabbitMQ, Apache Kafka, and ActiveMQ
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
subtitle: 本文主要对比 Web 应用程序开发中常用于身份认证和授权的 JWT、OAuth 和 SAML 等技术框架之间的差异。

categories:
- web开发

tags:
- RabbitMQ
- Kafka
- ActiveMQ
---

> 原文链接（请科学上网）：https://medium.com/javarevisited/difference-between-rabbitmq-apache-kafka-and-activemq-65e26b923114

大家好，如果您正在准备 Java 开发人员面试以及 Spring Boot 和微服务，您还应该准备消息代理、kafka、rabbitmq 和 activemq 之类的东西，比如 Kafka、RabbitMQ 和 ActiveMQ 之间有什么区别？也是Java面试的热门问题之一。

在我的上一篇文章中，我分享了[JWT、OAuth、SAML](https://medium.com/javarevisited/difference-between-jwt-oauth-and-saml-for-authentication-and-authorization-in-web-apps-75b412754127)和[REST 与 GraphQL 和 gRPC 之间的区别](https://medium.com/javarevisited/difference-between-rest-graphql-and-grpc-10ac365462b8)，在这篇文章中，我将分享我对 Kafka、RabbitMQ 和 ActiveMQ 这三种流行的[异步消息代理](https://medium.com/javarevisited/how-microservices-communicates-with-each-other-synchronous-vs-asynchronous-communication-pattern-31ca01027c53)的看法沟通。

消息系统在现代分布式架构中扮演着至关重要的角色，应用程序和服务通过网络相互通信。消息传递系统允许发送方和接收方解耦，从而实现[异步通信](https://medium.com/javarevisited/how-microservices-communicates-with-each-other-synchronous-vs-asynchronous-communication-pattern-31ca01027c53)。 RabbitMQ、Apache Kafka 和 ActiveMQ 是业界使用的三种流行消息系统。在本文中，我们将讨论 RabbitMQ、Apache Kafka 和 ActiveMQ 之间的区别。

顺便说一下，如果您正在准备 Java 开发人员面试，您还可以查看我之前发布的关于[21 个软件设计模式问题](https://medium.com/javarevisited/21-software-design-pattern-interview-questions-and-answers-b7d1774b5dd2)、[10 个基于微服务场景的问题](https://medium.com/javarevisited/top-10-microservices-problem-solving-questions-for-5-to-10-years-experienced-developers-3391e4f6b591)、[20 个来自面试的 SQL 查询](https://medium.com/javarevisited/20-sql-queries-for-programming-interviews-a7b5a7ea8144)、[50 个微服务问题](https://medium.com/javarevisited/50-microservices-interview-questions-for-java-programmers-70a4a68c4349)、[60 个树数据结构问题](https://medium.com/javarevisited/top-60-tree-data-structure-coding-interview-questions-every-programmer-should-solve-89c4dbda7c5a)、[15 个系统设计问题](https://medium.com/javarevisited/7-system-design-problems-to-crack-software-engineering-interviews-in-2023-13a518467c3e)，[35 Core Java Questions](https://medium.com/javarevisited/top-10-java-interview-questions-for-3-to-4-years-experienced-programmers-c4bf6d8b5e7b) 和[21 Lambda and Stream questions](https://medium.com/javarevisited/21-lambda-and-stream-interview-questions-for-java-programmers-38d7e83b5cac)，包含大量常见问题，可以帮助你更好地准备面试。

<hr />

## 什么是 RabbitMQ，它用在什么地方？

RabbitMQ 是一个实现高级消息队列协议 (AMQP) 标准的开源消息代理。它是用 Erlang 编写的，具有可插入的体系结构，可以轻松扩展。

RabbitMQ 支持多种消息传递模式，例如发布/订阅、请求/回复和点对点，并且它具有一组强大的功能，例如消息确认、路由和排队。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/18-Difference-Between-RabbitMQ-Kafka-ActiveMQ/01.png"/>

<hr />

## 什么是 Apache Kafka？它用在哪里？

Apache Kafka 是一个开源分布式事件流平台，最初由 LinkedIn 开发。 Kafka 是用 Scala 和 Java 编写的，旨在处理大规模流式数据流。

Kafka 使用发布/订阅消息传递模型，并针对高吞吐量、低延迟和容错进行了优化。

Kafka 具有持久的消息传递模型，这意味着消息存储在磁盘上并且可以多次重放。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/18-Difference-Between-RabbitMQ-Kafka-ActiveMQ/02.png"/>

<hr />

## 什么是ActiveMQ？它在哪里使用？

Apache ActiveMQ 是一个开源消息代理，它实现了 Java 消息服务 (JMS) API。 ActiveMQ 是用 Java 编写的，具有可插入的体系结构，可以轻松扩展。

ActiveMQ 支持多种消息传递模式，例如点对点、发布/订阅和请求/回复，并且它具有一组强大的功能，例如消息确认、路由和排队。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/18-Difference-Between-RabbitMQ-Kafka-ActiveMQ/03.gif"/>

<hr />

## RabbitMQ、Apache Kafka 和 ActiveMQ 之间的区别

现在您已经对 RabbitMQ、ActiveMQ 和 Apache Kafka 有了清晰的认识，是时候找出它们之间从消息传递模型到性能的区别了。以下是 Apache Kafka、RabbitMQ 和 ActiveMQ 之间的主要区别：

### 1. 消息模型

RabbitMQ 和 ActiveMQ 都支持 JMS API，这意味着它们遵循传统的消息传递模型，消息被发送到队列或主题并由一个或多个消费者使用。

另一方面，Kafka 使用发布/订阅消息传递模型，其中消息发布到主题并由一个或多个订阅者使用。

RabbitMQ 和 ActiveMQ 使用的传统消息传递模型非常适合需要严格排序和可靠传递消息的应用程序。

另一方面，Kafka使用的发布/订阅消息模型更适合流数据场景，需要实时处理数据。

这是一个很好的图表，它突出了 Kafka 和 RabbitMQ 之间的架构差异。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/18-Difference-Between-RabbitMQ-Kafka-ActiveMQ/04.png"/>

<hr />

## 2. 可扩展性

可伸缩性是消息传递系统的基本要求，尤其是在处理大量数据时。 RabbitMQ 和 ActiveMQ 都设计为可扩展的，但它们实现可扩展性的方法不同。

方式来实现可伸缩性，其中多个 RabbitMQ broker 连接形成一个集群。消息分布在整个集群中，消费者可以连接到集群中的任何代理来消费消息。 RabbitMQ 还支持联邦，允许多个 RabbitMQ 集群连接在一起。

ActiveMQ 使用代理网络方法来实现可伸缩性，其中多个 ActiveMQ 代理连接形成一个网络。消息分布在整个网络中，消费者可以连接到网络中的任何代理来消费消息。 ActiveMQ 还支持主/从复制，为消息代理提供高可用性。

另一方面，Kafka 被设计为具有开箱即用的高度可扩展性。 Kafka 使用分区方法来实现可伸缩性，其中消息在多个 Kafka 代理之间进行分区。每个分区都被复制到多个代理中以实现容错。这种方法允许 Kafka 处理大量数据，同时保持低延迟和高吞吐量。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/18-Difference-Between-RabbitMQ-Kafka-ActiveMQ/05.png"/>

<hr />

### 3. 性能

性能是选择消息系统时要考虑的另一个关键因素。 RabbitMQ、Kafka 和 ActiveMQ 都具有不同的性能特征。

RabbitMQ 被设计成一个可靠的消息传递系统，这意味着它优先考虑消息传递而不是性能。 RabbitMQ 可以处理适中的消息速率，适用于需要严格排序和可靠传递消息的应用程序。

另一方面，Kafka 专为高性能而设计，可以低延迟处理大量数据。 Kafka 通过使用分布式架构和优化顺序 I/O 来实现这种性能。

ActiveMQ 也是为高性能而设计的，可以处理高消息速率。 ActiveMQ 通过使用异步架构和优化消息批处理来实现此性能。

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
