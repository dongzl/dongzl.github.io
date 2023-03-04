---
title: Linux 系统零拷贝--什么是零拷贝以及零拷贝实现原理
date: 2023-03-05 20:03:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/Linux-Zero-Copy.png

# author information, multiple authors are set to array
# single author
author:
  - nick: Tony
    link: https://tonylixu.medium.com/

# post subtitle in your index page
subtitle: 在这篇博客文章中，我们将介绍 Linux 系统中零拷贝概念以及零拷贝实现原理。

categories: 
  - 数据库

tags: 
  - Linux
  - Zero Copy
---

> 原文链接：https://blog.devgenius.io/linux-zero-copy-d61d712813fe

许多 `Web` 应用程序都会提供大量静态文件内容，这相当于从磁盘读取数据后再将完全相同的数据写回 `socket`。每次数据经过用户态内核边界时，都必须进行数据拷贝，这个过程会消耗 `CPU` 时间片，占用内存带宽。

**零拷贝**技术在这种场景下就可以发挥作用，**零拷贝**的目的是消除内核态和用户态之间所有不必要的数据拷贝。无论是 `Kafka` 还是 `Netty`，都用到了**零拷贝**的知识。那么到底什么是**零拷贝**？在本文中我们将探索一下。

## 什么是零拷贝

- `零`：意味着拷贝数据的次数是 `0` 次；
- `拷贝`：意味着数据需要从一种存储介质转移到另外一种存储介质。

所以，如果我们把 `零` 和 `拷贝` 组合在一起，**零拷贝**是指计算机在进行 `IO` 操作时，`CPU` 不需要将数据从一个存储介质拷贝到另一个存储介质，从而减少上下文切换和 `CPU` 拷贝时间，**零拷贝**是一种 `IO` 操作优化技术。

## 传统 IO 的执行流程

例如我们要实现一个下载功能，服务器的任务就是将服务器主机磁盘上的文件从连接的 `socket` 中发送出去。关键代码如下：

```c
while((n = read(diskfd, buf, BUF_SIZE)) > 0)
    write(sockfd, buf , n);
```

传统的 `IO` 流程包括读和写的过程：

- **读**：从磁盘读取数据到内核缓冲区，再拷贝到用户缓冲区；
- **写**：首先将数据写入到 `socket` 缓冲区，最后写到网卡设备。

完整的流程如下所示：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/04-Linux-Zero-Copy/01.png" style="width:800px"/>

- 应用程序调用 `read` 函数，向操作系统发起 `IO` 请求，上下文从用户态切换到内核态；
- `DMA` 控制器从磁盘读取数据到内核缓冲区；
- `CPU` 读取内核缓冲区数据并将数据拷贝到用户应用程序缓冲区，上下文从内核态切换到用户态，`read` 函数调用返回结果；
- 用户应用进程通过 `write` 函数发起 `IO` 请求，上下文从用户态切换到内核态并将数据复制到 `socket` 缓冲区；
- `DMA` 控制器将数据从 `socket` 缓冲区复制到网卡设备，上下文从内核态切换到用户态，此时 `write` 函数返回结果。

从流程图可以看出，传统的 `IO` 流程包括 `4` 次上下文切换，`4` 次数据拷贝（`2` 次 `CPU` 拷贝和 `2` 次 `DMA`拷贝）。

## 内核空间和用户空间

服务器上运行的应用程序需要通过操作系统来执行一些特殊的操作，如磁盘文件的读写、内存的读写等。

因为这些都是比较危险的操作，应用程序不能乱来，只能交给底层操作系统来完成。

因此，操作系统为用户应用分配了两种内存空间：用户空间和内核空间。

- **内核空间**：主要提供进程调度、内存分配、硬件资源连接等功能；
- **用户空间**：提供给每个程序进程的空间，它没有访问内核空间资源的权限。如果应用程序需要使用内核空间的资源，需要经过操作系统进行调用。进程从用户空间切换到内核空间，完成相关操作后，再从内核空间切换回用户空间。

## 用户态和内核态

- 如果进程运行在内核空间，则称为进程的内核态。
- 如果进程运行在用户空间，则称为进程的用户态。

## DMA

`DMA` 表示直接内存访问。它本质上是主板上一个独立的芯片，它允许外围设备和内存存储之间直接进行 `IO` 数据传输，并且这个过程不需要 `CPU` 的参与。

简单来说就是帮助 `CPU` 转发 `IO` 请求，进行拷贝数据。为什么需要它呢？主要是为了提升效率，它帮助 `CPU` 做一些事情，这段时间 `CPU` 可以空闲下来做其他事情，这就提高了 `CPU` 的利用效率。

我们来看一下这个过程：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/04-Linux-Zero-Copy/02.png" style="width:800px"/>

- 应用程序调用 `read` 函数，向操作系统发起 `IO` 请求，同时进入阻塞状态等待数据返回；
- `CPU` 收到指令后，向 `DMA` 控制器发起指令调度；
- `DMA` 收到请求后，向磁盘发送请求；
- 磁盘将数据读入磁盘控制器缓冲区，并通知 `DMA`；
- `DMA` 将数据从磁盘控制器缓冲区拷贝到内核缓冲区；
- `DMA` 向 `CPU` 发送一个读取数据的信号，`CPU` 负责将数据从内核缓冲区复制到用户缓冲区；
- 用户应用进程从内核态切换到用户态并解除阻塞状态。

## 如何实现零拷贝

我们已经理解了 `DMA` 的工作原理，下面我们来讨论一下如何实现**零拷贝**。首先，**零拷贝**并不意味着不进行数据拷贝，而是减少用户态和内核态上下文切换次数和 `CPU` 拷贝次数；有两种常见的方式实现**零拷贝**：

- **方案一**：`mmap` + `write`
- **方案二**：`sendfile`

### mmap + write

`mmap` 的函数定义如下：

```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

通过 `mmap + write` 实现**零拷贝**处理流程如下图所示：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/04-Linux-Zero-Copy/03.png" style="width:800px"/>

与 `read()` 方法调用相比，这里的主要区别是用户进程通过调用 `mmap` 方法向操作系统内核发起 `IO` 调用，上下文从用户态切换到内核态，然后 `CPU` 使用 `DMA` 控制器将数据从硬盘复制到内核缓冲区。主要步骤是：

- 用户进程通过调用 `mmap` 方法向操作系统内核发起 `IO` 调用，上下文从用户态切换到内核态；
- `CPU` 通过 `DMA` 控制器将数据从磁盘拷贝到内核缓冲区；
- 上下文从内核态切换到用户态，`mmap` 方法调用返回结果；
- 用户进程通过调用 `write` 方法再次对操作系统内核进行 `IO` 调用，上下文从用户态切换到内核状态，最终写入到 `socket` 缓冲区；
- `CPU` 通过 `DMA` 控制器将数据从 `socket` 缓冲区拷贝到网卡设备，上下文从内核态切换到用户态，最后 `write` 方法返回结果。

我们发现通过 `mmap + write` 实现的**零拷贝**技术发生了 `4` 次上下文切换和 `3` 次拷贝（`2` 次 `DMA` 拷贝和 `1` 次 `CPU` 拷贝）。

### sendfile()

`sendfile` 是 `Linux 2.1` 版本后内核引入的系统函数，函数定义如下：

```c
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

`sendfile` 是指在两个文件描述符之间传递数据，它是在操作系统内核中完成的，因此可以避免内核缓冲区和用户缓冲区数据的拷贝操作，可以用来实现**零拷贝**技术。

流程如下图所示：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/04-Linux-Zero-Copy/04.png" style="width:800px"/>

- 用户进程发起 `sendfile` 系统调用，上下文从用户态切换到内核态；
- `DMA` 控制器从磁盘拷贝数据到内核缓冲区；
- `CPU` 将读缓冲区中的数据复制到 `socket` 缓冲区；
- `DMA` 控制器将 `socket` 缓冲区中的数据异步复制到网卡设备；
- 上下文从内核缓冲区切换到用户缓冲区，`sendfile` 函数调用返回；

我们发现通过 `sendfile` 实现的**零拷贝**技术只发生了 `2` 次上下文切换和 `3` 次拷贝（`2` 次 `DMA` 拷贝和 `1` 次 `CPU` 拷贝）。
