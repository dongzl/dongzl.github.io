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
- 架构复杂：需要Control Plane、Data Plane、不同应用的规则推送、Proxy之间的通信安全等。

## 5. eBPF 技术概述

正如我们已经观察到的情况，使用 `Sidecar` 模式，我们需要在每个单元上正确部署配置一个容器；如果仔细观察，每个节点只有一个内核，在同一个节点上运行的所有容器都共享同一个内核，我们能不能利用它来将部署的 Sidecar 代理的数量减少到与节点的数量一致？`eBPF` 正是用于解决这个问题。

eBPF 是一种内核技术，可以运行自定义程序以响应各种事件，包括网络数据包、函数访问等事件。

该技术允许自定义程序直接在主机上运行，无需额外部署 `Sidecar`，从而减少了服务网格中部署的 `Sidecar` 代理的数量。`eBPF` 驱动的解决方案可以提供可观察性、安全性，同时由于不需要在每个单元上额外部署 `Sidecar` 容器所以具备了一定网络优势。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/06-Exploring-Service-Mesh-In-Depth-Study/05.png" style="width:100%"/>

<div style="color:DarkGray;font-size:14px;text-align:center;"> https://isovalent.com/blog/post/2021-12-08-ebpf-servicemesh </div>

2021 年 12 月 2 日，Cilium 项目宣布了 Cilium Service Mesh 的 Beta 测试计划。基于 eBPF 的 Cilium 项目将这种**Sidecarless**模型引入到 `Service Mesh`，以处理 `Service Mesh` 的大部分 `Sidecar` 代理功能，包括 `L7` 路由、负载均衡、`TLS`、访问策略、健康检查、日志记录和追踪。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/06-Exploring-Service-Mesh-In-Depth-Study/06.png" style="width:100%"/>

<div style="color:DarkGray;font-size:14px;text-align:center;"> https://thenewstack.io/how-ebpf-streamlines-the-service-mesh/ </div>

切换到这种新的模式将会给我们带来什么好处？

### YAML 配置减少

在 `Sidecar` 模型中，必须修改 `YAML` 配置为每个 `Pod` 添加一个 `Sidecar` 容器，如果未能正确标记命名空间或 `Pod` 可能会导致 `Sidecar` 注入失败。

支持 `eBPF` 的 `sidecar-free` 代理模型可以在不做任何修改的情况下检测 Pod，并将它们包含在服务网格中，即使攻击者绕过 `Kubernetes` 编排，也可以检测到恶意行为。此外，启用 eBPF 的网络请求可以通过绕过一些内核网络堆栈来提高性能。

### 网络效率

在启用 `eBPF` 的网络中，数据包可以绕过内核的一些网络堆栈，从而提高性能。我们研究一下如何应用于服务网格数据面。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/06-Exploring-Service-Mesh-In-Depth-Study/07.png" style="width:100%"/>

<div style="color:DarkGray;font-size:14px;text-align:center;"> https://thenewstack.io/how-ebpf-streamlines-the-service-mesh/ </div>


