---
title: （进行中）Difference between API Gateway and Load Balancer in Microservices?
date: 2023-07-30 16:52:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/Load_Balancer-vs-API_Gateway.png

# author information, multiple authors are set to array
# single author
author:
- nick: Soma
  link: https://medium.com/@somasharma_81597
- nick: 董宗磊
  link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: xxxxxxxxxx。

categories:
- 架构设计

tags:
- Service Mesh
---

> 原文链接（请科学上网）：https://medium.com/javarevisited/difference-between-api-gateway-and-load-balancer-in-microservices-8c8b552a024

朋友们大家好，`API` 网关和负载均衡技术之间的差异对比是非常[流行的微服务面试问题](https://medium.com/javarevisited/50-microservices-interview-questions-for-java-programmers-70a4a68c4349)之一，面试官经常会在电话或现场面试中用这个问题来考察经验丰富的 `Java` 开发人员。过去，我分享了[基本的微服务设计原则和最佳实践](https://medium.com/javarevisited/10-microservices-design-principles-every-developer-should-know-44f2f69e960f)以及流行的[微服务设计模式](https://medium.com/javarevisited/top-10-microservice-design-patterns-for-experienced-developers-f4f5f782810e) 等文章；在本文中，我将回答这个问题，并告诉大家它们之间的关键区别。

虽然 `API` 网关和负载均衡都是微服务架构中的重要组件，但它们的用途却不相同。

**API 网关充当所有 API 请求的单一入口点**，提供*请求路由*、*请求限流*、*身份认证*和 *API 版本控制*等功能，并且还向客户端应用程序隐藏底层微服务的复杂性。

另一方面，**负载均衡负责在微服务的多个实例之间分发传入请求**，以提升服务可用性、性能和可扩展性。它有助于*在多个实例之间均匀分配工作负载*，并确保每个实例都得到充分利用。

换句话说，`API` 网关提供与 `API` 管理相关的**更高层级的能力**，而负载均衡提供与跨微服务的多个实例的流量分发相关的**较低层级的能力**。

## 微服务中的 API 网关模式是什么？

[API 网关](https://medium.com/javarevisited/what-is-api-gateway-pattern-in-microservices-architecture-what-problem-does-it-solve-ebf75ae84698)是微服务架构中使用的基本模式之一，它充当**反向代理**，将客户端的请求路由到多个内部服务。它还为所有客户端提供与系统交互的单一入口点，从而实现更好的[可扩展性](https://medium.com/javarevisited/difference-between-horizontal-scalability-vs-vertical-scalability-67455efc91c)、安全性和对 API 的控制。

API Gateway 处理身份验证、速率限制和缓存等常见任务，同时还抽象了底层服务的复杂性。当我们经历可以使用 API Gateway 的现实场景时，事情会更加清楚，所以让我们这样做。

假设您有一个基于微服务的电子商务应用程序，允许用户浏览和购买产品。该应用程序由**多个微服务**组成，包括产品目录服务、购物车服务、订单服务和用户服务。

**为了简化客户端与这些微服务之间的交互，您可以使用 API 网关。** API 网关充当所有客户端请求的单一入口点，并将它们路由到适当的微服务。

例如，**当用户想要查看产品目录时，他们向 API 网关发出请求，API 网关将请求转发到产品目录服务。**同样，当用户想要下订单时，他们会向 API 网关发出请求，API 网关会将请求转发到订单服务。

*通过使用API网关，您可以简化客户端代码，减少需要发出的请求数量*，并为客户端与微服务交互提供统一的接口。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/20-Difference-Between-API-Gateway-Load-Balancer-Microservices/02.png"/>

## 什么时候使用API网关？

现在我们知道什么是 API Gateway 以及它提供哪些服务，就很容易理解何时使用它了。 API网关可用于多种场景，例如

1. **为多个微服务提供统一的入口点**：API网关可以作为多个微服务的单一入口点，为客户端提供统一的接口；
2. **实施安全性**：API网关可以为所有微服务实施安全策略，例如身份验证和授权；
3. **提高性能**：API网关可以缓存微服务的响应并减少到达后端的请求数量；
4. **简化客户端**：API Gateway 可以抽象底层微服务的复杂性，为客户端交互提供更简单的接口；
5. **版本控制和路由**：API网关可以根据请求参数将请求路由到同一微服务的不同版本或不同的微服务。

因此，您可以看到它所做的不仅仅是将您的请求转发到微服务，这可能是 API 网关和负载均衡器之间的主要区别。

## 什么是负载均衡器？它解决什么问题？

负载均衡器是一个在服务器集群中的多个服务器或节点之间分配传入网络流量的组件，不仅在微服务架构中，而且在任何架构中都是如此。

它通过在服务器之间均匀分配工作负载，有助于提高应用程序和服务的性能、可扩展性和可用性。

通过这种方式，**负载均衡器可以确保没有任何一台服务器流量过载**，而其他服务器则保持空闲，从而提高资源利用率并提高整个系统的可靠性。

这是一个很好的图表，显示了负载均衡器的工作原理以及如何在硬件和软件级别使用它：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/20-Difference-Between-API-Gateway-Load-Balancer-Microservices/03.png"/>

## 何时在微服务中使用负载均衡器？

正如我所说，负载均衡器主要用于微服务架构中，以便在同一服务的多个实例之间分配传入的网络流量。

负载均衡器可以帮助提高应用程序的可用性和可扩展性，这就是为什么它们甚至在微服务架构出现之前就已经存在了很长一段时间。

当应用程序的流量过高并且单个服务实例无法处理负载时，也会使用负载均衡器。当应用程序跨多个服务器或数据中心部署时，也会使用负载平衡器。

<hr />

## 微服务中负载均衡器的优缺点

现在，让我们看看在微服务架构中使用负载均衡器的优点和缺点：

优点：

- **提高性能和可扩展性**：负载均衡器在多个服务器之间均匀分配流量，减少每个服务器上的负载并确保没有服务器不堪重负；
- **高可用性和容错能力**：如果一台服务器出现故障，负载均衡器可以自动将流量重定向到其他健康的服务器，确保即使某些服务器出现故障，服务仍然可用；
- **灵活性和定制**：负载均衡器可以使用不同的算法和规则进行定制，以最适合系统特定需求的方式处理流量。

缺点：

- **复杂性和成本**：负载均衡器会增加系统的复杂性和成本，因为它们需要额外的硬件或软件来管理和维护；
- **单点故障**：如果负载均衡器本身发生故障，则整个系统可能变得不可用；
- **延迟增加**：根据设置，负载均衡器可以通过在客户端和服务器之间添加额外的跃点来增加系统延迟。

<hr />

## 微服务架构中负载均衡器和API网关的区别

以下是 API 网关和负载均衡器之间的 10 个逐点差异：

### 1. 目标

API网关的主要目的是为微服务提供统一的API，而负载均衡器的主要目的是在多个服务器之间均匀分配流量。

### 2. 功能性

API网关可以执行多种功能，例如路由、安全、负载均衡和API管理，而负载均衡器仅处理流量分配。

### 3. 路由

API 网关根据一组预定义的规则路由请求，而负载均衡器根据预定义的算法（例如循环法或最少连接）路由请求。

### 4. 协议支持

API网关通常支持多种协议，如HTTP、WebSocket、MQTT等，而负载均衡器仅支持传输层协议，如TCP、UDP等。

### 5. 安全性

API网关提供身份验证、授权和SSL终止等功能，而负载均衡器仅提供SSL卸载等基本安全功能。

### 6. 缓存

API网关可以缓存微服务的响应以提高性能，而负载均衡器不提供缓存功能。

### 7. 可移植性

API网关可以在不同格式之间转换数据，例如JSON到XML，而负载均衡器不提供数据转换功能。

### 8. 服务发现

API网关可以与服务发现机制集成以动态发现微服务，而负载均衡器则依赖于静态配置。

### 9. 粒度

API网关可以提供对API端点的细粒度控制，而负载均衡器仅控制服务器级别的流量。

### 10. 可扩展性

API网关可以处理大量API请求并管理微服务的扩展，而负载均衡器仅提供水平扩展功能。

<hr />

这就是**微服务中API网关和负载均衡器之间的区别**。正如我所说，API 网关提供与 API 管理相关的**高级功能**，例如安全性、版本控制、单一入口点，而负载均衡器提供与跨微服务的多个实例的流量分配相关的**低级功能**。

因此，如果您需要的不仅仅是将请求转发到多个实例，您应该使用 API Gateway 而不是负载均衡器，但如果您的要求只是将传入流量分发到多个应用程序实例，则只需使用负载均衡器。
