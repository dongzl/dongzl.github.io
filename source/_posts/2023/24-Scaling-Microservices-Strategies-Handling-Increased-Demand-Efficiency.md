---
title: 微服务扩展：以 99% 的处理效率来应对不断增长的需求
date: 2023-09-23 20:03:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/scaling_microservices.png

# author information, multiple authors are set to array
# single author
author:
- nick: Soma
  link: https://medium.com/@somasharma_81597
- nick: 董宗磊
  link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 打造弹性微服务基础设施：以 99% 的处理效率来应对不断增长的需求的行之有效的策略。

categories:
- 架构设计

tags:
- 微服务
- 弹性
---

> 原文链接：https://blog.stackademic.com/scaling-microservices-strategies-for-handling-increased-demand-with-99-efficiency-1ce47dd02490

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/24-Scaling-Microservices-Strategies-Handling-Increased-Demand-Efficiency/04.webp"/>

<font color=DarkGray size=2>image_credit — uber.com</font>

朋友们大家好，很多朋友都曾建议我写一篇如何**在实际工作中实现微服务可扩展性**的文章？这就是我写这篇文章的初衷，在这篇文章中我将会分享我对实现微服务可扩展性的一些理解，并为大家提供一个将微服务扩展至百万服务的案例，同时能够保障 99% 的时间正常提供服务，而这正是 `Uber.com`（我最喜欢的出租车叫车服务）的应用程序。

[创建](https://www.java67.com/2023/06/how-to-create-microservice-application.html)一个能够正常运行的微服务是一回事，而创建一个每天与数百万用户完成数十亿次事件交互的微服务则完全不同。

近年来，[微服务架构](https://javarevisited.blogspot.com/2021/09/microservices-design-patterns-principles.html#axzz8EaNxWt4z)因其模块化、独立且高度可扩展的软件架构能力而受到广泛欢迎。然而，随着应用系统用户数量和访问量的增长，如何实现微服务的高效扩展也成为一项极具挑战的任务。

在前面的几篇文章中，我一直在分享我在微服务和 `Spring Cloud` 方面的经验，例如[微服务最佳实践](https://medium.com/javarevisited/10-microservices-design-principles-every-developer-should-know-44f2f69e960f)、[微服务设计模式](https://medium.com/javarevisited/top-10-microservice-design-patterns-for-experienced-developers-f4f5f782810e)、[50 个微服务常见面试题](https://medium.com/javarevisited/50-microservices-interview-questions-for-java-programmers-70a4a68c4349)和 [Spring Cloud 所具备的 10 个微服务特性](https://medium.com/javarevisited/10-spring-cloud-features-which-makes-microservice-development-easier-in-java-e061885422fe)，我之前还分享过关于[SAGA 设计模式](https://medium.com/javarevisited/what-is-saga-pattern-in-microservice-architecture-which-problem-does-it-solve-de45d7d01d2b)和[单体与微服务架构](https://medium.com/javarevisited/difference-between-microservices-and-monolithic-architecture-for-java-interviews-af525908c2d5)的文章。

在本文中，我们将深入探讨如何应对微服务架构中不断增长的流量和不断变化的需求，会重点关注负载平衡、弹性扩展和性能优化，同时我们还将通过一个真实的案例来理解如何在实际工作中运用这些概念。

<hr />

### 1. 微服务扩展方面的挑战

微服务架构将单一应用程序拆分成多个较小的、互连的服务，这些服务通过网络进行通信。

这种设计非常地灵活，可以更充分的实现资源利用以及服务的独立开发和部署。然而随着用户流量的增长，个别的服务可能会出现性能瓶颈，遇到性能问题。

如何实现[微服务](https://www.java67.com/2023/01/what-is-cqrs-command-query.html)扩展，软件开发团队会面临几个挑战，来应对微服务架构中不断增长的流量和不断变化的需求。主要挑战包括：

1. **服务依赖**：微服务通常依赖其他服务来实现特定功能；随着流量的增加，服务之间的互相依赖可能会导致瓶颈和性能问题，从而影响整个系统的可扩展性；
2. **数据管理**：跨分布式微服务场景中协调和管理数据可能会变得复杂，尤其是在处理海量数据时，确保跨多个服务的数据一致性、可用性和完整性可能是一个需要面对的挑战；
3. **网络通信开销**：微服务通过网络进行通信，随着服务数量的增长，网络通信可能会产生巨大的开销，这可能会导致系统延迟和响应时间变长，从而影响用户体验；
4. **维护复杂性**：维护和监控大量微服务需要强大的基础设施、部署和监控工具；随着系统的扩展，管理工作会变得越来越复杂；
5. **动态可扩展性**：虽然自动扩展是一项关键策略，但根据实际需要动态添加或删除实例可能会导致资源争用，尤其是在操作不当的情况下；
6. **一致性和事务**：确保分布式微服务之间的事务一致性是一项具有挑战性的任务，确保多个服务的变更操作全部完成或在发生故障时全部回滚是很复杂的工作；
7. **状态管理**：管理微服务的状态，尤其是在需要状态的应用场景中，可能会很困难；处理有状态服务的故障转移和主从复制增加了整个扩展过程的复杂性；
8. **安全和授权**：在处理增长的流量的同时确保微服务的安全性需要强大的身份验证和授权机制；随着请求数量的增长，安全性管理变得非常重要。

应对这些挑战需要仔细进行系统规划、架构设计以及采用一些在微服务开发、部署和运维方面的**最佳实践**。微服务的自动扩展需要专业技术知识、监控工具和持续优化工作相结合。

但是不用担心，我们将会学习一些可以遵循的最佳实践，将微服务扩展到数百万级用户量，并保持 99% 的效率。

<hr />

### 2. 理解负载均衡

负载均衡是在微服务的多个实例之间分配传入请求流量的关键策略，来确保资源被充分利用并防止某个实例流量过载。

有多种负载均衡算法，每种算法都有其优缺点和使用场景：

1. **轮询算法**：请求按顺序循环均匀分布在多个实例之间；
2. **最少连接算法**：流量会被路由到活动连接最少的服务器；
3. **加权轮询算法**：根据容量为服务器分配不同的权重。

我们还可以使用 [API 网关](https://medium.com/javarevisited/difference-between-api-gateway-and-load-balancer-in-microservices-8c8b552a024) 等模式作为微服务架构中的负载均衡器，以在微服务的多个实例之间分配请求流量。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/24-Scaling-Microservices-Strategies-Handling-Increased-Demand-Efficiency/01.webp"/>

[API 网关](https://medium.com/javarevisited/what-is-api-gateway-pattern-in-microservices-architecture-what-problem-does-it-solve-ebf75ae84698)充当客户端访问各种微服务的单一入口点，提供负载均衡、流量路由、安全性等能力，我在之前的文章中也深入探讨了 [API 网关的工作原理](https://medium.com/javarevisited/what-is-api-gateway-pattern-in-microservices-architecture-what-problem-does-it-solve-ebf75ae84698)。

<hr />

### 3. 自动扩展：适应需求

这是您可以用来将微服务扩展到数百万个的另一种策略。自动缩放允许微服务根据流量波动动态调整其资源。

自动扩展系统无需手动配置和取消配置实例，而是持续监控负载并根据需要添加或删除实例。

这确保了最佳性能和成本效率。

触发自动缩放的关键指标包括CPU利用率、内存消耗和响应时间。AWS、Google Cloud和Azure等所有主要云平台都提供可与微服务部署集成的自动扩展服务。

以下是在 AWS 云中设置 Auto Scaling 的示例：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/24-Scaling-Microservices-Strategies-Handling-Increased-Demand-Efficiency/02.webp"/>

<hr />

### 4. 性能优化：保证高效执行

这可能是扩展应用程序并保持其响应能力的最古老的策略。性能优化涉及微调微服务以最大限度地提高效率并最大限度地缩短响应时间。技术包括：

1. 缓存：将经常访问的数据存储在内存中，以减少数据库查询。
2. 异步处理：将资源密集型任务移至后台工作人员或队列。
3. 数据库分片：将数据分布到多个数据库实例以提高读写性能。

这是AWS无服务器架构中缓存的另一个很好的例子

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/24-Scaling-Microservices-Strategies-Handling-Increased-Demand-Efficiency/03.webp"/>

<hr />

### 5. 现实世界的例子：Uber 的可扩展性之旅

没有比看例子更好的方法来理解任何事情了。让我们看一个现实世界的例子来说明这些扩展策略的实际应用：叫车巨头 Uber。

Uber 的微服务架构使其能够在全球众多城市无缝运营，每天为数百万用户提供服务。随着 Uber 的受欢迎程度飙升，有效扩展其服务变得至关重要。

顺便说一句，这是我做的另一件事，建议其他开发人员阅读 Uber、NetFlix 等科技巨头的工程博客，了解他们如何解决挑战。这将提高您的知识和理解力。

Uber 的微服务架构如下所示：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/24-Scaling-Microservices-Strategies-Handling-Increased-Demand-Efficiency/04.webp"/>

您可以看到他们正在使用API 网关、缓存代理以及更多您可以学习的东西，以将您的微服务扩展到数百万个。

### 6. Uber 中的负载均衡

为了管理高流量，Uber 采用了强大的负载平衡机制。当用户打开 Uber 应用程序时，他们的请求将被定向到附近的数据中心。

在数据中心内，负载均衡器使用最少连接算法将请求分发到最不繁忙的微服务实例。

这确保了用户请求的均匀分布，防止任何单个微服务实例变得不堪重负。您可以在此处阅读有关 Uber 实时负载平衡的更多信息。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/24-Scaling-Microservices-Strategies-Handling-Increased-Demand-Efficiency/05.webp"/>

<hr />

### 7. 自动扩展以满足需求

在高峰时段或特殊活动期间，优步的乘车请求会激增。为了应对这些高峰，Uber 广泛利用了自动扩展功能。

例如，负责匹配乘客和司机的“乘车请求”微服务会根据传入请求数量和平均响应时间等指标动态扩展其实例。

Uber 的自动扩展系统会监控这些指标并自动调整实例数量，以确保较短的响应时间和快速的乘车匹配。当流量减少时，不必要的实例就会减少，从而优化资源使用和成本。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/24-Scaling-Microservices-Strategies-Handling-Increased-Demand-Efficiency/06.webp"/>

<hr />

### 8. 性能优化，打造无缝体验
   
Uber 的应用程序提供实时位置跟踪，这需要高性能的微服务。为了优化性能，Uber 广泛使用缓存。

驾驶员和乘客数据以及乘车状态更新都缓存在内存中，从而减少了频繁的数据库查询的需要。

此外，Uber 采用异步处理来执行计算乘车费用和发送通知等任务。

通过将这些任务转移到后台工作人员和队列，微服务可以快速处理乘车数据，而不会影响用户体验。

### 结论

这就是将微服务扩展到数百万用户所需要做的所有重要事情。虽然这看起来很简单，但当你真正去做时，你会意识到它需要仔细的计划和执行。

随着用户需求不断增加，有效扩展微服务是现代应用程序开发的一个关键方面。负载均衡、自动扩展和性能优化是确保基于微服务的应用程序实现最佳性能、可靠性和成本效益的基本策略。

Uber 等现实世界的例子展示了这些策略的实际实施，展示了它们在处理高流量和需求方面的有效性。

通过利用负载平衡算法、自动扩展机制和性能优化技术，您可以创建弹性且响应迅速的微服务生态系统，即使在高峰使用期间也能提供无缝的用户体验。

总之，面对不断增长的用户期望和流量，掌握扩展微服务的艺术对于旨在提供高质量数字服务的企业至关重要。

顺便说一句，如果你正在准备微服务面试，那么你还应该准备一些问题，比如API网关和负载均衡器之间的区别、SAGA模式、如何管理微服务中的事务、SAGA和CQRS模式之间的区别，这些问题在面试中很受欢迎。
