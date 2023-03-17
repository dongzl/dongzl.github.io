---
title: Istio、ePBF 和 RSocket Broker：深入探索 Service Mesh 技术
date: 2023-03-13 20:03:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/service_mesh.png

# author information, multiple authors are set to array
# single author
author:
- nick: Kondah Mouad
  link: https://medium.com/@deepkondah

# post subtitle in your index page
subtitle: 本文是一篇翻译文章，在这篇博客文章中，我们将学习如何借助 `performance schema` 中 `Instrument` 工具来排查 `MySQL` 数据库的一些问题。

categories:
- 架构设计

tags:
- Service Mesh
- Istio
- ePBF
- RSocket
---

> 原文链接：https://medium.com/geekculture/istio-ebpf-and-rsocket-broker-a-deep-dive-into-service-mesh-7ec4871d50bb

## 背景介绍

在微服务时代，单个复杂的应用程序被分解成多个**组件化**、**协作**和**连接**的单元，单个服务往往会绑定越来越多的业务功能，这使得服务治理的难度前所未有。仅仅依靠微服务框架层面的治理是不够的，需要构建高维度的深度治理体系。

本文从多个角度审视服务治理内容，并通过对传统和现代方案进行探索，提供一些对 `Service Mesh`、`Istio`、`eBPF` 和 `RSocket Broker` 技术的理解。

## 1. 服务治理

治理是指建立和实施一套流程，这套流程能够展示出微服务是如何通过协同工作以实现系统设计和构建的业务目标的。服务不要超出其上下文边界非常重要。

服务治理可以通过多种方式实现：

- 服务注册和发现：`Consul`、`ZooKeeper`；
- 服务配置：`Spring Cloud Config`；
- 服务容错：`Hystrix`；
- 网关：`Zuul`、`Spring Cloud Gateway`；
- 负载均衡：`Ribbon`、`Feign`；
- 追踪工具：`Sleuth`、`Sleuth`、`Htrace`；
- 监控工具：`Grafana`、`Prometheus`。

接下来，我们将讨论其中的一些内容。

### 服务注册和发现

在云服务器架构时代，将单个大型单体应用程序拆分成更小的可独立部署的服务单元过程，称为微服务。

当一个服务需要与另一个服务进行通信时，它需要知道远端服务的 IP 和端口。一种直接的解决方案是维护一个配置文件记录目标服务的 IP 地址和端口，但是这种方式存在很多不足，其中之一就是不具备云服务的可扩展性，云服务为我们提供了根据当前负载弹性扩容/缩容服务器实例的能力，使用配置文件的方式根本没有办法使用这种能力。

这就是服务发现发挥作用的地方，它通过提供服务注册的机制来帮助解决上述问题，也就是说，当一个新服务启动并想要参与处理客户端请求时，它会使用 IP 和端口将自己注册到注册中心，并且这些信息可以自动被客户端获取到。此外，通过健康检查机制可以确保请求流量只转发到健康的实例。

### 负载均衡

负载均衡机制能够将网络请求分发到多个服务器以增强处理能力。我们将检查并使用 RSocket 来平衡跨服务器池的客户端请求。

负载均衡是一种请求调度策略，它可以协调多台服务器共同处理网络请求，从而扩展系统的处理能力。在后面的部分中，我们将尝试使用 RSocket 技术实现跨服务器资源池自动均衡客户端请求。

## 2. Sidecar 模式

`Sidecar` 模式可以看做是将应用程序的功能划分为独立的进程运行在同一个调度单元中。`Sidecar` 模式可以用于抽象与应用程序业务逻辑无关的功能，从而降低代码的可重复性和复杂性。可观测性、监控、日志记录、配置、服务熔断等功能可以在 `sidecar` 容器中实现并部署在同一单元上，比如部署在 `Kubernetes` 的一个 `Pod` 中。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/06-Exploring-Service-Mesh-In-Depth-Study/01.png" style="width:100%"/>

<div style="color:DarkGray;font-size:14px;text-align:center;"> https://apisix.apache.org/blog/2021/12/17/exposure-istio-with-apisix-ingress/ </div>

在 `sidecar` 模式下，代理容器可以与 Pod 中的应用程序容器共享相同的网络命名空间。网络命名空间是 `Linux` 内核的结构，它允许容器和 `Pod` 拥有自己独立的网络堆栈，将容器化应用程序相互隔离。这使得应用程序彼此互不影响，这就是为什么我们可以让尽可能多的 Pod 在 80 端口上运行 Web 应用程序。代理将共享相同的网络命名空间，以便它可以拦截和处理应用程序容器进出的流量。

## 3. Service Mesh 探索

微服务是一种架构风格，它可以将具有单一职责和功能的多个单元以模块化的方式组合起来，形成一个非常复杂的、大型的应用程序。

第一代微服务依赖于内置的 `SDK` 实现服务发现、熔断重试等功能（如 `Spring Cloud`）。从开发的角度来看，这种方式存在很大的不足，因为开发人员需要将 `SDK` 包含在软件调用堆栈中并最终完全依赖特定的编程语言，更不用说升级/部署成本、敏捷性原则（当 `SDK` 升级时，应用程序也需要升级）、版本碎片化和较高的学习曲线。即使业务逻辑未发生改变，如果 `SDK` 需要升级，应用程序也需要更新发布新版本，这是非功能代码与业务代码耦合的结果。

### Service Mesh

`Service Mesh` 位于基础设施层，通过 `Sidecar` 模式处理服务之间的通信，以透明代理的形式提供安全、快速、可靠的服务间通信。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/06-Exploring-Service-Mesh-In-Depth-Study/02.png" style="width:100%"/>

<div style="color:DarkGray;font-size:14px;text-align:center;"> https://servicemesh.es </div>

借助 `Service Mesh`，我们可以将 `SDK` 中的大部分能力从应用程序中剥离出来，拆解成独立的进程（容器），以 `sidecar` 的形式进行部署。通过将服务治理能力下沉到基础设施中，微服务只需要做好一件事-专注于业务逻辑，而且会做地越来越好。

这样，基础架构团队就可以更加专注于各种通用能力，真正做到自主演进、透明升级，提升整体效率。

`Service Mesh` 的基础设施层主要分为**控制面**和**数据面**两部分。`Istio` 和 `Linkerd` 是两个流行的开源服务网格框架。稍后我们将重点讨论 `Istio`。

服务网格在整体架构上比较简单，不过就是在各种服务旁边部署很多用户代理，再加上一套任务管理进程。代理在服务网格中称为数据层或数据面，管理进程称为控制层或控制面。数据面拦截不同服务之间的流量并“处理”它们；控制面协调代理的行为，并提供 `API` 或命令行工具来配置版本管理以实现持续集成和部署。

服务之间不会通过网络直接进行调用，而是通过本地的 `sidecar` 代理进行调用，同时 `sidecar` 代理又代表服务管理请求，封装了服务间通信的复杂性。

## 4. Istio 快速入门

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/06-Exploring-Service-Mesh-In-Depth-Study/03.png" style="width:100%"/>

`Istio` 是一个开发的服务治理平台，面向云原生场景以 `Service Mesh` 的形式提供服务，并与 `Kubernetes` 紧密结合。

`Istio` 提供了负载均衡、服务间鉴权、服务监控和其它一些能力。

### Istio 架构

`Istio` 的架构分为控制面和数据面。

- **数据面**：它由遍布整个网格的 Sidecar 代理组成，它与应用服务一起以 sidecar 的形式部署。每个 Sidecar 都会接管服务的进出流量，配合控制平面完成流量控制等功能；
- **控制面**：顾名思义，它在数据面之上，负责控制和管理 Sidecar 代理，完成配置分发、服务发现、授权认证等功能。**在架构中控制面的作用是可以统一管理数据面。**

### 核心组件

下面简单介绍一下 `Istio` 架构中几个核心组件的主要功能。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/06-Exploring-Service-Mesh-In-Depth-Study/04.png" style="width:100%"/>

<div style="color:DarkGray;font-size:14px;text-align:center;"> https://ssup2.github.io/theory_analysis/Istio_Architecture/ </div>

### Envoy

`Envoy` 是使用 `C++` 语言实现的高性能代理。 `Istio` 服务网格将 `Envoy` 代理作为 `Sidecar` 容器注入到应用程序容器旁边，并拦截服务的所有进出流量，执行不同的功能，例如负载平衡、服务熔断、故障注入和暴露 `Prometheus` 和 `Jeager` 收集数据的代理指标。被注入的代理共同构成了服务网格的**数据面**。

### Istiod

`Istiod` 是一个控制面组件，它提供服务发现、配置和证书管理功能。 `Istiod` 采用 `YAML` 格式编写配置文件，并将其转换为 `Envoy` 可使用配置格式。然后它将此配置分发到网格中的所有 `Sidecar`。

- **Pilot**：`Pilot` 组件的主要作用是将路由规则等配置信息转化为 `Sidecar` 可以识别的内容，发送给数据面。可以简单理解为配置分发器，辅助 `Sidecar` 完成流量控制相关功能；

- **Citadel**：`Citadel` 是 `Istio` 中的一个安全组件，它负责证书生成，允许数据面代理之间进行安全的 `mTLS` 通信；

- **Galley**：`Galley` 是在 `Istio 1.1` 版本中新增加的一个组件，目的是解耦 `Pilot` 和 `Kubernetes` 等底层平台。它共享了原有 `Pilot` 的部分功能，主要负责配置的校验、提取和处理功能。

### Istio 流量转发

流量路由分为流入和流出流程。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/06-Exploring-Service-Mesh-In-Depth-Study/05.png" style="width:100%"/>

<div style="color:DarkGray;font-size:14px;text-align:center;"> https://istio.io/latest/blog/2022/merbridge/ </div>

入站处理程序作用是将下游流量转发到主应用程序容器，另一方面，出站处理程序监听所有出站流量并将其转发到上游。 `Istio` 使用一个 `Init` `Container` 操作 `Pod` 网络命名空间中的 `iptables` 并设置规则，以便将 `Pod` 的入站/出站数据包传输到 `Sidecar`。

`Init` 容器与应用容器相比有如下不同：

- 它在主容器启动之前开始运行，并且运行到所有其它容器结束后才结束运行；
- 如果有很多 `Init` 容器，它们将以顺序方式启动运行。

说到 `Service Mesh`，我们可能首先想到的就是 `Istio` + `Envoy` 构成的 `Sidecar` 的 `Service Mesh` 架构，目前这个架构非常流行。虽然乍一看这个架构没有明显的问题，但仍有几点值得深入考虑：

- 性能下降：`Proxy` 是一个独立的应用程序，需要特定的资源，例如 `CPU` 和`内存`。 `Envoy` 运行通常需要大约 `1G` 的内存；
- 架构复杂：需要控制面、数据面、不同应用的规则推送、`Proxy` 之间的通信安全等；
- 运维成本增加：没有自动化运维工具，就无法部署 `Sidecar`，`Service Mesh` 的典型解决方案是基于 `Kubernetes`，能够减少了很多工作量。

## 5. eBPF 介绍

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/06-Exploring-Service-Mesh-In-Depth-Study/06.png" style="width:100%"/>

正如我们已经观察到的情况，使用 `Sidecar` 模式，我们需要在每个单元上正确部署配置一个容器；如果仔细观察，每个节点只有一个内核，在同一个节点上运行的所有容器都共享同一个内核，我们能不能利用它来将部署的 Sidecar 代理的数量减少到与节点的数量一致？`eBPF` 正是用于解决这个问题。

eBPF 是一种内核技术，可以运行自定义程序以响应各种事件，包括网络数据包、函数访问等事件。

`eBPF` 是一种内核技术，允许自定义程序在内核中运行，这些运行的程序可以响应数以千计的各种事件，`eBPF` 程序可以附加到这些事件上，这些事件包括轨迹点、访问或退出各种功能（在内核或用户空间中）或对服务网格很重要的网络数据包。如果你将一个 `eBPF` 程序添加到一个内核事件中，它就会被触发，无论是哪个进程引起了这个事件，也无论事件是运行在应用程序容器中还是直接运行在主机上。无论你是在可观测性、安全性还是网络，`eBPF` 驱动的解决方案无需部署 `sidecar` 就可以检测到应用程序。

该技术允许自定义程序直接在主机上运行，无需额外部署 `Sidecar`，从而减少了服务网格中部署的 `Sidecar` 代理的数量。`eBPF` 驱动的解决方案可以提供可观察性、安全性，同时由于不需要在每个单元上额外部署 `Sidecar` 容器所以具备了一定网络优势。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/06-Exploring-Service-Mesh-In-Depth-Study/07.png" style="width:100%"/>

<div style="color:DarkGray;font-size:14px;text-align:center;"> https://isovalent.com/blog/post/2021-12-08-ebpf-servicemesh </div>

2021 年 12 月 2 日，Cilium 项目宣布了 [Cilium Service Mesh](https://cilium.io/blog/2021/12/01/cilium-service-mesh-beta/) 的 Beta 测试计划。基于 `eBPF` 的 [Cilium](https://cilium.io/) 项目将这种**Sidecarless**模型引入到 `Service Mesh`，以处理 `Service Mesh` 的大部分 `Sidecar` 代理功能，包括 `L7` 路由、负载均衡、`TLS`、访问策略、健康检查、日志记录和链路追踪。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/06-Exploring-Service-Mesh-In-Depth-Study/06.png" style="width:100%"/>

<div style="color:DarkGray;font-size:14px;text-align:center;"> https://thenewstack.io/how-ebpf-streamlines-the-service-mesh/ </div>

切换到这种新的模式将会给我们带来什么好处？

### YAML 配置减少

在 `sidecar` 模式中，需要修改每个指定应用程序 `Pod` 的 `YAML` 文件来添加 `sidecar` 容器，这通常是自动化完成的——例如，使用变异的 `webhook` 在部署每个应用程序 `Pod` 时注入 `sidecar`。

以 `Istio` 为例，它需要标记 `Kubernetes` 命名空间和/或 `Pod` 来定义是否应该注入 `sidecar`。

但是，如果出现问题怎么办？如果命名空间或 `Pod` 没有被正确标记，`sidecar` 将不会被注入，`Pod` 也无法连接到服务网格；更糟糕的是，如果攻击者破坏集群并运行恶意程序，则无法通过服务网格提供的流量观测能力进行监控。

相比之下，在支持 `eBPF` 的 `sidecar-free` 代理模型中，无需任何额外的 `YAML` 配置即可检测到 `Pod`。相反，`CRD` 用于配置集群内的服务网格，甚至现有的 `Pod` 也可以成为服务网格的一部分而无需重新启动。

此外，当攻击者试图通过直接在主机上运行程序来绕过 `Kubernetes` 编排时，`eBPF` 能够监控并控制此类操作，因为所有这些操作都可以从内核中看到。

### 网络效率

在启用 `eBPF` 的网络中，数据包可以绕过内核的一些网络堆栈，从而提高性能。我们研究一下如何将它应用到服务网格数据面。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/06-Exploring-Service-Mesh-In-Depth-Study/07.png" style="width:100%"/>

<div style="color:DarkGray;font-size:14px;text-align:center;"> https://thenewstack.io/how-ebpf-streamlines-the-service-mesh/ </div>

在 `eBPF` 加速和无 `sidecar` 的服务网格模型中，网络数据包传输的路径要短很多。

在服务网格的场景下，代理在传统网络中作为 `sidecar` 运行，数据包到应用程序的路径相当长：入站数据包必须穿过主机 TCP/IP 网络堆栈并通过虚拟网卡连接到 `Pod` 的网络命名空间。从这开始，数据包必须通过 Pod 的网络堆栈才能到达代理，代理通过环回地址将数据包转发给应用程序。考虑到流量必须经过连接两端的代理，与非服务网格场景下的流量相比，延迟将会显著增加。

### 网络加密

服务网格通常用于确保所有应用程序流量都经过身份认证和加密。通过双向 `TLS` (`mTLS`)，服务网格代理组件充当网络连接的端点，并与远程端点协商安全 `TLS` 连接，该连接在不更改应用程序的情况下加密代理之间的通信。

`TLS` 的应用层实现并不是实现组件之间认证和流量加密的唯一方式，也可以使用 `IPSec` 或 `WireGuard` 在网络层进行流量加密，因为它在网络层运行，所以这种加密不仅对应用程序完全透明，而且对代理也完全透明——无论是否有服务网格，都可以启用它。如果我们使用服务网格的唯一原因是提供加密，我们可能需要考虑网络级加密，它不仅更简单，而且还用于验证和加密节点上的任何流量——它不仅限于启用了 sidecar 的工作负载。

## 6. RSocket Broker

RSocket 路由代理是使用 RSocket 协议在泛应用程序之间进行通信的系统。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/06-Exploring-Service-Mesh-In-Depth-Study/10.png" style="width:100%"/>

<div style="color:DarkGray;font-size:14px;text-align:center;"> https://rsocketbyexample.info/java/ </div>

RSocket Broker 的工作原理是：服务调用者（Requester）向 broker 发起服务调用请求，broker 将请求转发给服务提供者（Responder），broker 最终在将 responder 的处理结果返回给服务调用者。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/06-Exploring-Service-Mesh-In-Depth-Study/11.png" style="width:100%"/>

当一个服务提供者应用程序启动时，它会主动与 Broker 创建一个TCP长连接，然后告诉 Broker 它可以提供的服务列表。

当一个服务消费者应用程序启动时，它同样会与 Broker 创建一个 TCP 长连接。当消费者应用程序想要调用一个远程服务时，服务消费者将服务调用请求封装为消息（用唯一的消息ID标识）发送给Broker。broker 收到消息后，根据浮出水面的元信息解析出需要调用的服务，然后在内部服务路由表中查找可以调用的服务。
服务提供者处理请求后，将处理结果封装为消息返回给Broker。Broker 根据消息 ID 将返回的消息转发给服务调用者。请求消费响应消息并执行相应的业务逻辑。

这种基于 Broker 的消息通信方式具有以下优点：

- 不需要第三方健康检查，因为我们知道连接何时启动；
- 无端口监听：服务提供者不再监听端口，与HTTP REST API和gRPC完全不同，更加安全；
- 通信透明：服务调用者和服务提供者无需感知对方的存在；
- 流量控制：如果服务提供者压力过大，broker会自动将消息转发给其他服务提供者（智能负载均衡），可以通过租约来实现；
- 服务注册和发现：无需 `Eureka`、`Consul`、`ZooKeeper` 等第三方注册中心，降低基础设施依赖成本；
- 安全：`Broker` 会验证服务提供者和服务消费者的访问权限，只需要在 `Broker` 上部署 `TLS` 支持即可保证通信通道的安全。

### 没有免费的午餐

Broker 也有一些缺点。由于双方之间没有通信，性能会有所下降。此外，所有通信流量都通过 Broker 转发，因此存在网络瓶颈，但这可以通过集群和 Broker 的高可靠性来缓解。

## 7. 通过 RSocket Broker 进行服务治理

Istio 作为 Service Mesh 的解决方案，其实很难在数据中心之外应用。物联网设备呢？每个手机都安装sidecar？这就是 RSocket 代理发挥作用的地方。

RSocket 路由代理可用于实现服务网格。在下面的方案中，没有运行 sidecar，也没有重复进程。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/06-Exploring-Service-Mesh-In-Depth-Study/12.png" style="width:100%"/>

下面是两种架构方案的典型特征对比：

- 基础设施层：一个是 `sidecar` 代理 + 控制面，另一个是集成了控制面功能的中心化 broker；
- 集中管理：集中化会让管理更全面，比如 `Logging`、`Metrics` 等；
- 通信协议：`RSocket` 方案的一个缺点是应用程序之间必须使用 `RSocket` 通信协议；
- 应用程序或设备访问：并非所有设备都可以安装代理；主要有几个原因：设备和系统本身不支持，比如物联网设备；这是基于 `RSocket` 的方案具有巨大优势的地方；
- 增加运维成本：管理一个由 `10` 台服务器组成的 `RSocket` `Broker` 集群和管理 `10K` 个 `Proxy` 实例是不一样的；
- 效率：`RSocket` 协议性能比 `HTTP` 高 `10` 倍；
- 安全性：`RSocket` 的安全性实现更简单。`Broker` 主要是用 `TLS` + `JWT` 实现，不是 `mTLS`，不需要证书管理。同时，借助 `JWT` 的安全模型，很容易实现更细粒度的权限控制，使用 `RSocket` 代理方案，可以减少攻击面；
- 网络和基础设施依赖：`RSocket` `Broker` 相对于 `Istio` 的一大优势就是不依赖 `Kubernetes`，虽然 `Istio` 也声称不依赖 `Kubernetes`，但是在 `Kubernetes` 之外部署和管理 `sidecar` 代理并不简单，而 `RSocket` `Broker` 可以部署在任何地方；
- ... ...

## 最后总结

服务治理是微服务时代一个非常重要的成长话题。我们研究了通过 `Istio`、`eBPF` 和 `RSocket` `Router` 实现它的不同方法。

## 参考链接

- https://thenewstack.io/how-ebpf-streamlines-the-service-mesh/
- https://www.infoq.com/presentations/rsocket-spring-cloud-gateway/
- https://www.alibabacloud.com/blog/a-brief-on-rsocket-and-reactive-programming_598219
- https://www.slideshare.net/Pivotal/weaving-through-the-mesh-making-sense-of-istio-and-overlapping-technologies
- https://jimmysong.io/blog/sidecar-injection-iptables-and-traffic-routing/
- https://kknews.cc/code/kk5bqn8.html
- https://blog.birost.com/a?ID=00900-415484c9-b8a9-4762-8da9-daf675dcd626
- https://knner.wang/2019/12/27/ServiceMesh-Istio-Series--microservice-kubernetes-istio-relation.html
- https://www.jianshu.com/p/dd818114ab4b
- https://knner.wang/2019/12/27/ServiceMesh-Istio-Series--microservice-kubernetes-istio-relation.html
- https://www.infoq.cn/article/q65ddirtdsbf*e6ki2p4
