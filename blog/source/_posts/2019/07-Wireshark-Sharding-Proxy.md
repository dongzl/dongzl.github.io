---
title: 记一次 Wireshark 抓包 Sharding-Proxy 过程
date: 2019-10-23 17:39:35
categories:
tags:
- Wireshark
- ShardingSphere
- Sharding-Proxy
- MySQL
---

## 背景描述
最近在在参与 [Apache ShardingSphere](http://shardingsphere.apache.org/) 开源项目的一些工作，在 Github 的 [issue 列表](https://github.com/apache/incubator-shardingsphere/issues)认领了一个问题 [#3005](https://github.com/apache/incubator-shardingsphere/issues/3005)。

产品描述：首先简单介绍一下 ShardingSphere 中的一个产品 Sharding-Proxy，Sharding-Proxy 产品的功能是做一个透明的中间层代理，后面连接很多的 MySQL 数据库（可能是很多分库后 MySQL 数据库）。用户在使用中可以不用直接连到真实 MySQL 数据库上，而是连接到 Sharding-Proxy 上，通过这个工具再连接到真实的数据库上。

<!-- more -->

这个过程对于用户应该是无感知的（所以才叫透明的中间层代理），可以像使用 MySQL 一样使用 Sharding-Proxy，例如通过一些工具，像 MySQL JDBC 驱动或者是客户端工具（Navicat、MySQL Workbench）直接使用 Sharding-Proxy，所以 Sharding-Proxy 需要对 MySQL 协议进行解析和封装。对于客户端发送请求，Sharding-Proxy 解析后重新封装发送给 MySQL 服务器，对于 MySQL 服务器响应数据需要解析后重新封装发送给客户端，这个过程要做到精确，才能实现`透明的中间层代理`的效果。

认领的 issue 中的问题现象是，Navicat 直接连到 MySQL 服务器打开数据表没有问题，而通过 Sharding-Proxy 代理连接后再打开数据表，就会提示 <font color=#FF0000> There is no primary key here. Update will only use exact matching of the old values of the columns here. Thus, it may update more than one record. </font> 这个错误，虽然对于使用是没有影响的，但是依然没有做到完全透明。

## Wireshark 软件使用

对于 Navicat 这种客户端工具，其实是通过网络连接方式连接管理远端数据库，所以一定要走网络协议。所以对于这个问题，首先想到的办法就是通过网络抓包的方式对比一下两种连接方式返回的数据差别。网络抓包好像是有这么个工具 **Wireshark**，对于这个工具只闻名未曾谋面过，果断安装，百度了一下简单实用。

启动软件后，出现一堆网卡，可能是装虚拟机导致出现了好多虚拟网卡吧，不过大部分网卡没有数据流量。

<img src="https://raw.githubusercontent.com/dongzl/dongzl.github.io/hexo/blog/source/images/Wireshark_Sharding-Proxy_01.png" width="600px">

首先是 **Wi-Fi: en0** 网卡，进去之后列表一堆数据，直接刷屏，根本看不出有效信息，毫无头绪。

这个时候发现还有一个 **Loopback: lo0** 网卡，应该是 **环回地址（127.0.0.1）**，这个可能有点意思，因为 MySQL、Sharding-Proxy 和 Navicat 都装在本地开发环境，正常情况不会有真正的网络通信，都是本地搞定。

进入 **Loopback: lo0** 网卡，看到的就是 **Wireshark** 抓取的网络请求的数据了，其实第一次用，现学现卖，使用不是很流畅，不过还是百度大概了解到了一些使用技巧，首先 **Wireshark** 可以使用很多过滤器，过滤出我们比较感兴趣的网络请求数据，比如根据协议抓取数据，根据请求端口抓取数据进行区分。

在这个首先想到的是 MySQL 数据库使用的端口是 3306，Sharing-Proxy 使用的端口是 3307，还有就是 MySQL 有自己的应用层协议，协议名字就是：MySQL。

## Wireshark 抓取 MySQL 协议数据

首先使用 Navicat 直接连到 MySQL 数据库上，在 Wireshark 软件中只过滤 MySQL 协议的网络数据请求：

<img src="https://raw.githubusercontent.com/dongzl/dongzl.github.io/hexo/blog/source/images/Wireshark_Sharding-Proxy_02.png" width="600px">


Wireshark 还是很强大，很直观的。每一次的 Request 请求，紧跟着是是配对的 Response 响应数据。其中在 Request 请求中 **MySQL Protocol** 中显示了 Navicat 客户端发送给 MySQL 数据库的命令：**SHOW VARIABLES LIKE 'lower_case_%'**；在 Response 相应的 **MySQL Protocol** 中显示了 MySQL 数据库发送给 Navicat 客户端的相应结果数据，通过上面的操作和观察，至少能大概明白简单的使用方式了。

## Wireshark 抓包数据对比
Wireshark 大概使用是明白了，现在需要看的就是 Navicat 直连 MySQL和 Navicat 通过 Sharing-Proxy 代理连接 MySQL 到底数据的请求响应过程到底有什么不同了。抓取同一个请求命令在两种不同情况下相应结果数据的差别，理论上来说这就应该是我们排查问题的方向，当然方向是否正确，还需要实操验证。

### Navicat 直连 MySQL 抓包
根据 Navicat 的提示，只有在查询表数据表的时候会提示错误，所以问题大概应该出现在 select 查询语句的请求 & 相应数据上，我们抓取了一张表的 select 语句数据：

<img src="https://raw.githubusercontent.com/dongzl/dongzl.github.io/hexo/blog/source/images/Wireshark_Sharding-Proxy_04.png" width="600px">

对于查询的这张数据表，其实 id 字段就是主键，主键是一定存在的，我们看一下返回结果中有关 id 字段的描述信息：

<img src="https://raw.githubusercontent.com/dongzl/dongzl.github.io/hexo/blog/source/images/Wireshark_Sharding-Proxy_05.png" width="600px">

其中有 **Flags** 属性展开后就是一串标志位，通过实际内容大概就能明白就是标识该字段的一些特殊属性信息：

<img src="https://raw.githubusercontent.com/dongzl/dongzl.github.io/hexo/blog/source/images/Wireshark_Sharding-Proxy_06.png" width="600px">

### Navicat 连接 Sharding-Proxy 抓包

由于 Sharding-Proxy 底层是直接使用 Netty 实现网络连接，使用 TCP 协议，并且监听的端口号是 3307，所以我们在使用 Wireshark 抓包时可以采用端口号过滤的方式过滤网络请求数据：

> tcp.srcport==3307 || tcp.dstport==3307

我们还是抓取了相同一条 select 语句的请求相应数据，由于是直接抓取 TCP 协议包数据，无法像抓取 MySQL 协议数据那样直观了，只能通过二进制数据看出大概样子：

select 请求数据：

<img src="https://raw.githubusercontent.com/dongzl/dongzl.github.io/hexo/blog/source/images/Wireshark_Sharding-Proxy_07.png" width="600px">

Sharding-Proxy 相应结果：

<img src="https://raw.githubusercontent.com/dongzl/dongzl.github.io/hexo/blog/source/images/Wireshark_Sharding-Proxy_08.png" width="600px">

## 结果分析

仔细分析抓取的数据结果，其实还是能看到一些差别的，不过 Sharding-Proxy 抓取到的结果不是很直观，还是给结果分析带来一定困难的，还是直接上结论吧：

<img src="https://raw.githubusercontent.com/dongzl/dongzl.github.io/hexo/blog/source/images/Wireshark_Sharding-Proxy_09.png" width="600px">

<img src="https://raw.githubusercontent.com/dongzl/dongzl.github.io/hexo/blog/source/images/Wireshark_Sharding-Proxy_10.png" width="600px">

分析的结果可以看出，Navicat 工具直连 MySQL 数据库时执行的 select 查询语句中返回的 `Flags` 信息标志位二进制数据为 `03 50`，标识该字段为 `Not null` 和 `Primary key`；但是 Navicat 连接 Sharding-Proxy 代理后返回的数据中该标志位二进制位 `00 00`，其实到这里，结果就很明显了，Sharding-Proxy 返回的数据中 `Flags` 信息缺失，而该数据正式判断该数据是否为主键、是否允许为null等等信息的一种标识。

## 写在最后

其实，现在记录整个内容过程感觉很简单，其实过程并不容易，有些分析其实就是 **经验 + 运气**，而且可能运气的成分还要多一些，不过整个过程下来还是给自己增长很多知识的，不过确实要说一句：**Wireshark 是个很不错的软件，以后要多学习、常使用，应用好了事半功倍**。

## 参考资料
- [Wireshark使用教程](https://www.cnblogs.com/lsdb/p/9254544.html)