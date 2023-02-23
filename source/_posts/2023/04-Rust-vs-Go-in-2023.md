---
title: （待完成）Rust vs Go in 2023
date: 2023-02-25 20:03:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/Rust-vs-Go.png

# author information, multiple authors are set to array
# single author
author:
  - nick: John Arundel
    link: https://bitfieldconsulting.com/

# post subtitle in your index page
subtitle: 本文是一篇翻译文章，在这篇博客文章中，我们将学习如何在不使用连接器或外部库的情况下从头开始编写我们自己的原生 MySQL 客户端。

categories: 
  - Go语言
  - Rust语言

tags: 
  - Go
  - Rust
---

> 原文链接：https://bitfieldconsulting.com/golang/rust-vs-go

Rust 和 Go，哪一个是更好的选择？在 2023 年，哪一个是你在下一个项目中优先选择的编程语言，为什么选择这个呢？这两者在性能、易用性、安全性、功能特性、使用规模和并发性等方面的比较如何？它们有什么共同点，它们的根本区别在哪里？让我们友好且客观公正地对 `Rust` 和 `Golang` 进行比较并找出这些问题的答案。

### Rust 和 Go 都是让人惊叹的

首先，必须要说明一下 `Go` 和 `Rust` 都是非常优秀的编程语言。它们都很先进、功能强大、并且已经被广泛使用，而且都性能非常出色。

您可能已经阅读过旨在说服您 Go 比 Rust 更好的文章和博客文章，反之亦然。但这真的没有意义；每种编程语言都代表一组权衡。每种语言都针对不同的事物进行了优化，因此您对语言的选择应取决于适合您的语言以及您想用它解决的问题。

在本文中，我将尝试简要概述我认为 Go 是理想选择的地方，以及我认为 Rust 是更好选择的地方。我还将尝试介绍这两种语言的本质（Go 和 Rust 之道，如果你愿意的话）。

虽然它们在语法和风格上有很大不同，但 Rust 和 Go 都是构建软件的一流工具。话虽如此，让我们仔细看看这两种语言。

### Go，Rust：相似之处

Rust 和 Go 有很多共同点，这也是您经常听到它们一起被提及的原因之一。两种语言的一些共同目标是什么？

> Rust 是一种低级静态类型多范式编程语言，专注于安全性和性能。 —Gints Dreimanis

然而：

> Go 是一种开源编程语言，可以轻松构建简单、可靠且高效的软件。—golang.org

#### 内存安全

Go 和 Rust 都属于优先考虑内存安全的现代编程语言。几十年来使用 C 和 C 等较旧的语言已经很清楚，错误和安全漏洞的最大原因之一是不安全或不正确地访问内存。

Rust 和 Go 以不同的方式处理这个问题，但两者都旨在比其他语言更智能、更安全地管理内存，并帮助您编写正确和高性能的程序。