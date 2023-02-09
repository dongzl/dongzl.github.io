---
title: 使用 Rust 一步一步构建 gRPC 服务器
date: 2023-02-10 20:03:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/rust_grpc.png
# author information, multiple authors are set to array
# single author
author:
  - nick: Yuchen Z.
    link: https://yuchen52.medium.com/

# post subtitle in your index page
subtitle: 本文是一篇翻译文章，在这篇博客文章中，我们将学习如何在不使用连接器或外部库的情况下从头开始编写我们自己的原生 MySQL 客户端。

categories: 
  - 架构设计

tags: 
  - Rust
  - gRPC
---

> 原文链接：https://betterprogramming.pub/building-a-grpc-server-with-rust-be2c52f0860e

## 背景介绍

### RPC、JSON、SOAP 对比

一旦我了解了 [gRPC]() 和 [Thrift]()，就很难回到使用更具过渡性的基于 `JSON` 的 `REST` `API` 或 [SOAP]() `API`。

`gRPC` 和 `Thrift` 这两个著名的 [RPC]() 框架有很多相似之处。前者源自谷歌，后者源自 `Facebook`。它们都易于使用，对各种编程语言都有很好的支持，而且性能都很好。

这两个框架最有价值的功能是支持多语言代码自动生成和服务器端反射，这些特性使得 `API` 本质上是类型安全的；通过服务器端反射，无需阅读和理解接口实现，就可以更轻松地探索 `API` 的模式定义。

### gRPC 和 Thrift 对比

[Apache Thrift]() 在历史上一直是一个受欢迎的选择。但近年来，由于缺乏 Facebook 的持续支持，再加上 [fbthrift]() 的分支项目，逐渐失去了人气。

与此同时，gRPC 已经赶上了越来越多的功能，拥有更健康的生态系统。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/02-Building-A-GRPC-Server-With-Rust/01.webp" style="width:600px"/>

gRPC（蓝）和 Apache Thrift（红）对比。[Google Trends]()

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/02-Building-A-GRPC-Server-With-Rust/02.webp" style="width:600px"/>

gRPC、fbThrift 和 Apache Thrift GitHub star 历史数据。[https://star-history.com]()

截至目前，除非我们的应用程序以某种方式附属于 Facebook，否则没有充分的理由考虑使用 Thrift。

### GraphQL 怎么样？

[GraphQL]() 是另一个由 Facebook 发起的框架。它与上面的两个 RPC 框架有许多相似之处。

