---
title: 系统设计中正向代理和反向代理的差异
date: 2023-11-18 20:03:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/forward_reverse_proxy_v2.png

# author information, multiple authors are set to array
# single author
author:
- nick: Soma
  link: https://medium.com/@somasharma_81597
- nick: 董宗磊
  link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 了解系统设计中正向代理和反向代理之间的区别以及何时使用它们。

categories:
- 架构设计

tags:
- 正向代理
- 反向代理
---

> 原文链接：https://medium.com/javarevisited/difference-between-forward-proxy-and-reverse-proxy-in-system-design-da05c1f5f6ad

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/25-Difference-Between-Forward-Proxy-Reverse-Proxy-System-Design/01.webp"/>

朋友们大家好，如果大家正在准备系统设计的面试，那么了解正向代理和反向代理之间的差异是非常重要，这是关于系统设计最常见的面试问题之一，[API网关和负载均衡器之间的差异](https://dongzl.github.io/2023/07/30/20-Difference-Between-API-Gateway-Load-Balancer-Microservices/)，我们之前的文章中已经了解过。

我们在设计复杂系统时，通常会使用代理服务器来提升系统性能、安全性和可靠性。代理服务器位于客户端和服务器之间，用于管理流经客户端和服务器之间的流量。

我们经常使用到的两种类型的代理服务是正向代理和反向代理。虽然两者的设计目的都是**提高系统的性能和安全性**，但它们的工作方式却有很大的不同，而且是在不同的场景中使用。在本文中，我们将探讨正向代理和反向代理在[系统设计](https://medium.com/javarevisited/top-10-system-design-concepts-every-programmer-should-learn-54375d8557a6)方面的差异。

顺便说一句，如果大家正在准备高级开发人员的岗位面试，那么除了系统设计之外，我们还应该熟悉像微服务等不同架构，以及各种微服务设计模式，例如[事件溯源](https://medium.com/javarevisited/what-is-event-sourcing-design-pattern-in-microservices-architecture-how-does-it-work-b38c996d445a)、[CQRS](https://medium.com/javarevisited/what-is-cqrs-command-and-query-responsibility-segregation-pattern-7b1b38514edd)、[SAGA](https://dongzl.github.io/2023/08/20/22-What-SAGA-Pattern-Microservice-Architecture/)、[微服务独立数据库](https://medium.com/javarevisited/what-is-database-per-microservices-pattern-what-problem-does-it-solve-60b8c5478825)、[API 网关](https://dongzl.github.io/2023/07/30/20-Difference-Between-API-Gateway-Load-Balancer-Microservices/)、[熔断器](https://medium.com/javarevisited/what-is-circuit-breaker-design-pattern-in-microservices-java-spring-cloud-netflix-hystrix-example-f285929d7f68)，这些文章内容将会在面试过程中为大家提供很大帮助，因为这些内容通常能够用于衡量一名开发人员的资历水平。

现在让我们回到本文的主题，详细了解什么是正向代理和反向代理、它们的优缺点、如何去使用它们以及最重要的正向代理和反向代理之间的差异。

## 什么是正向代理？在什么场景中使用它？

正向代理是位于客户端和互联网之间的代理服务器。客户端通过正向代理从互联网请求资源或服务，正向代理充当请求代理中介，将请求转发到互联网，然后将响应结果返回给客户端。

正向代理通常用于**互联网访问控制**、**内容过滤**或为客户端提供匿名访问。正向代理还可以通过缓存频繁被请求的内容来加速对资源的访问。

这是一个正向代理的示例：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/25-Difference-Between-Forward-Proxy-Reverse-Proxy-System-Design/02.webp"/>

我们可以看到客户端连接到正向代理，然后正向代理将请求路由到互联网服务。

### 正向代理架构的优缺点

现在大家已经熟悉了代理服务在正向代理中的作用，那么我们就可以很容易地总结出它的优缺点。以下是使用正向代理的一些优缺点总结：

**优点：**
1. **增强安全性**：正向代理可以通过隐藏访问互联网服务的客户端的原始 `IP` 地址来提供额外的安全性保障；
2. **提升速度和性能**：缓存被频繁请求的资源可以缩短客户端访问互联网服务的响应时间；
3. **访问控制**：通过限制对某些资源的访问，服务提供方可以使用正向代理来阻止对敏感数据进行未经授权访问；
4. **匿名**：用户在浏览互联网服务时可以保持匿名状态，因为他们的 `IP` 地址是被隐藏的。

**缺点：**
1. **复杂配置**：正向代理需要更复杂的配置，因为它们必须在每个设备上配置才能生效；
2. **单点故障**：如果正向代理出现故障，所有依赖它的设备也将无法访问互联网服务；
3. **延迟增加**：正向代理会增加响应延迟并降低对互联网服务访问的整体性能；
4. **有限控制**：正向代理可能会限制用户访问某些资源的能力，从而给用户带来困扰并降低工作效率。

除了这些限制之外，它提供的访问控制能力和安全能力是在各种系统架构上使用正向代理的主要原因。下面让我们继续学习一下什么是反向代理以及它是如何工作的。

<hr />

## 什么是反向代理？什么时候使用它？

**反向代理是位于客户端和源服务器之间的服务器**，接收来自客户端的请求并将其转发到适当的服务器。

然后来自服务器的响应返回给代理服务并转发到客户端。从本质上讲，它有助于保护源服务器不被客户端直接访问。

反向代理通常**用作负载平衡器，用来平衡多个服务器之间的请求负载**，通过隐藏服务器基础架构的详细信息来提高安全性，并提供缓存、SSL终端等其它增值服务。

反向代理常用于以下场景：

- **负载平衡**：在多个服务器之间分配传入流量，以提高性能和可用性；
- **安全性**：保护后端服务器免于直接暴露在互联网环境并防止未经授权的访问；
- **可扩展性**：实现在客户端无感的情况下水平扩展服务器基础架构。

反向代理**为客户端提供单一入口点**，使管理和监控后端服务器的流量变得更加容易。

反向代理还在客户端和服务器之间提供一定程度的抽象，使得在客户端无感的情况下修改或升级服务器基础结构。

以下是 `NGINX` 反向代理设置的示例：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/25-Difference-Between-Forward-Proxy-Reverse-Proxy-System-Design/03.webp"/>

我们可以看到客户端直接连接到互联网，但服务器位于代理后面，因此客户端永远不会知道其发出的请求被处理的服务器，从而实现从外部保护我们的基础设施服务。

### 反向代理架构的优缺点

以下是使用反向代理架构的一些优点和缺点：

**优点：**
1. **提高安全性**：反向代理可以通过屏蔽后端服务器的身份和位置来提供额外的安全层保障，防止外部客户端直接访问内部服务器；
2. **更好的可扩展性**：反向代理可以在多个后端服务器之间均匀分配请求流量，确保不会有某一台服务器过载从而导致应用程序崩溃；
3. **提高性能**：通过缓存和压缩数据，反向代理可以减少客户端和服务器之间传输的数据量，从而缩短响应时间；
4. **简化架构**：反向代理可用于将多个后端服务器整合到单个端点，从而简化应用程序的整体架构。

**缺点：**
1. **单点故障**：如果反向代理发生故障，整个服务可能会变得不可用，这是它最大的缺点；
2. **复杂性增加**：实施和维护反向代理可能比维护简单的 `client-server` 架构更复杂。
3. **自定义限制**：反向代理可能无法提供与客户端和服务器之间直接访问所能提供相同级别服务能力，这可能会限制应用程序的某些功能；
4. **额外成本**：实施反向代理可能需要额外的硬件和软件，这可能会增加整个系统架构的成本。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/25-Difference-Between-Forward-Proxy-Reverse-Proxy-System-Design/04.png"/>

<hr />

## 正向代理和反向代理之间的差异

现在我们已经基本了解了什么是正向代理和反向代理、它们的所处的位置以及它们的功能，现在需要研究一下这二者之间的差异以便我们更好地理解它们：

**1. 方向**
正向代理和反向代理之间的主要差异在于流量的方向。正向代理用于将流量从客户端转发到 `Internet`，而反向代理用于将流量从 `Internet` 转发到应用服务器。

**2. 客户端访问**
使用正向代理时，必须显式配置客户端才能使用代理服务器。相比之下，反向代理对客户端来说是透明的，客户端可以直接访问应用服务器，而不需要知道代理服务器的地址。

**3.负载均衡**
反向代理可以将客户端的请求分发到多个服务器以平衡负载，而正向代理则不具备该能力。

**4. 缓存**
正向代理可以缓存被频繁访问的资源，从而降低应用服务器的负载并可以缩短请求的响应时间。反向代理还可以缓存资源，但缓存通常在更靠近客户端的地方进行，以提高性能。

**5. 安全**
正向代理可以通过隐藏因特网上的 `IP` 地址来保护客户端的身份；反向代理可以通过隐藏服务器身份并向互联网公开指定 `IP` 地址来保护应用服务器。

**6. SSL/TLS 终端**
反向代理可以代理应用服务器终止 `SSL/TLS` 连接，从而减少服务器的负载并简化证书管理；正向代理通常不会终止 `SSL/TLS` 的连接。

**7. 内容过滤**
正向代理可用于内容过滤、限制对特定资源的访问以及强制执行特定访问策略；反向代理也可以进行内容过滤，但这通常是在靠近客户端的地方完成的，从而降低应用服务器上的负载。

**8. 路由**
正向代理可用于根据预定义的规则将流量路由到不同的服务器；反向代理也可以执行请求路由，但这通常是根据请求的 `URL` 和其他标识来完成的。

**9. 可扩展性**
反向代理可以通过在多个服务器之间分配流量来水平扩展应用程序；正向代理不提供这种可扩展性能力。

**10. 网络复杂性**
正向代理的设置和管理相对简单；而反向代理由于其负载平衡、`SSL/TLS` 终端和缓存功能可能导致架构更加复杂。

这是一张来自 `ByteByteGo` 网站的精美图表，这个网站是学习系统设计的更好地方之一，它还强调了正向代理和反向代理之间的区别：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/25-Difference-Between-Forward-Proxy-Reverse-Proxy-System-Design/05.jpeg"/>
