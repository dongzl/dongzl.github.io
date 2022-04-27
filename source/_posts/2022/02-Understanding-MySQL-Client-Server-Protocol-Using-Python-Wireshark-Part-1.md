---
title: 使用 Python 和 Wireshark 理解 MySQL 客户端/服务器协议：第 1 部分
date: 2022-04-24 20:03:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/mysql_python.png
# author information, multiple authors are set to array
# single author
author:
  - nick: Elshad Agayev
    link: https://www.turing.com/blog/author/elshad-agayev/

# post subtitle in your index page
subtitle: 本文是一篇翻译文章，根据 MySQL 官方文档，该博客文章解释了 MySQL 客户端/服务器协议的全部内容。

categories: 
  - 数据库

tags: 
  - MySQL
  - Wireshark
---

> 原文链接：https://www.turing.com/blog/understanding-mysql-client-server-protocol-using-python-and-wireshark-part-1/

## 背景介绍

`MySQL` 的**客户端 / 服务端**协议有很多应用场景。例如：

- `MySQL` 的客户端连接工具，例如：`ConnectorC`，`ConnectorJ` 等等；
- `MySQL` 的 `Proxy` 工具；
- 成为 `Master` 和 `Slave` 之间的桥梁。

那么，`MySQL` 的**客户端 / 服务端**协议到底是什么东西呢？

`MySQL` 的**客户端 / 服务端**协议是一种被接受的约定（规则）。通过这些规则，客户端和服务器可以实现“对话”并能够相互理解。客户端通过带有特殊套接字的 `TCP` 连接连接到 `MySQL` 的服务端，向服务端发送特殊数据包并从服务端接收返回数据。这个连接有会有两个阶段：

- 连接阶段；
- 指令阶段。

下面通过一幅图来描述一下这两个阶段：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/02-Understanding-MySQL-Client-Server-Protocol-Using-Python-Wireshark-Part-1/01.webp" style="width:600px"/>

## 数据包介绍

每个数据包都包含有价值的数据类型。每个数据包的最大长度是 `16M`。如果数据包的长度超过 `16M`，这个数据包就会被切分成多个数据块（每个 `16M` 的块）。首先我们来看一下协议的数据类型。`MySQL` 的客户端和服务端有两种数据类型：

- 整数类型；
- 字符串类型。

请看官方文档：https://dev.mysql.com/doc/internals/en/basic-types.html

## 整数类型

整数类型也被分成两部分：

- 固定长度的整数类型；
- 长度编码的整数类型；

固定长度的整数类型占用：`1`，`2`，`3`，`4`，`6` 或者 `8` 字节。例如，如果我们想用 `int<3>` 数据类型描述数字 `2`，那么我们可以这样写成十六进制格式：`02 00 00`。或者如果我们想用 `int<2>` 数据类型描述数字 `2`，那么我们可以写成这样十六进制格式：`02 00`。

长度编码的整数类型占用：`1`，`3`，`4` 或者 `9` 字节。在长度编码的整数类型之前有 `1` 个字节，这一个字节用来检测整数的长度，我们必须要检查这第一个字节。

- 如果第一个字节小于 `0xfb（< 251）`，那么下一个字节是有价值的（它将存储为 `1` 字节整数）；
- 如果第一个字节等于 `0xfc（== 252）`，那么它将存储为 `2` 字节整数；
- 如果第一个字节等于 `0xfd（== 253）`，那么它将存储为 `3` 字节整数；
- 如果第一个字节等于 `0xfe（== 254）`，那么它将存储为 `8` 字节整数。

如果第一个字节的等于 `0xfb`，那么就不需要读取下一个字节了，它等于 `MySQL` 的 `NULL` 值，如果等于 `0xff` 则表示它未定义。

例如，要将 `fd 03 00 00 ...` 转换为普通整数，我们必须读取第一个字节，它是 `0xfd`。根据上述规则，我们必须读取接下来的 `3` 个字节并将其转换为普通整数，其值在十进制数系统中为 `2`。所以长度编码的整数数据类型的值为 `2`。


## 字符串类型

字符串类型同样被分割成为几部分：

- `String` - 固定长度的字符串类型，它们具有已知的硬编码长度；
- `String` - `Null` 终止的字符串类型，这些字符串以 `0x00` 字节结尾；
- `String` - 可变长度字符串类型，在这样的字符串出现之前是固定长度的整数类型，根据这个整数类型，我们可以计算字符串的实际长度；
- `String` - 长度编码的字符串类型，在这样的字符串之前是长度编码的整数类型，根据该整数类型，我们可以计算字符串的实际长度；
- `String` - 如果字符串类型是数据包的最后一部分，那么字符串的长度可以通过整个数据包长度减去当前位置来计算得出。

## 使用 Wireshark 抓包数据

我们启动 `Wireshark` 工具来嗅探网络数据，通过 `IP` 条件来过滤 `MySQL` 的数据包（在我的环境中，服务端 `IP` 是：`54.235.111.67`）。接下来我们尝试使用我本地的 `MySQL` 本地客户端工具来连接到 `MySQL` 服务器。

正如在 `TCP` 连接到服务器后看到的内容，我们有几个来自服务器的 `MySQL` 数据包，首先是握手数据包。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/02-Understanding-MySQL-Client-Server-Protocol-Using-Python-Wireshark-Part-1/02.webp" style="width:600px"/>

让我们深入研究这个数据包并描述每个字段含义：

前 `3` 个字节是数据包长度：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/02-Understanding-MySQL-Client-Server-Protocol-Using-Python-Wireshark-Part-1/03.webp" style="width:600px"/>

接下来 `1` 字节是数据包序号：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/02-Understanding-MySQL-Client-Server-Protocol-Using-Python-Wireshark-Part-1/04.webp" style="width:600px"/>

剩余字节为 `MySQL` **客户端 / 服务端**协议的握手数据包的有效载荷：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/02-Understanding-MySQL-Client-Server-Protocol-Using-Python-Wireshark-Part-1/05.webp" style="width:600px"/>

我们来详细描述一下握手数据包每一个属性的内容：

- 协议号 - `Int<1>` （Protocol number – Int<1>）
- 服务端版本号 - `String` （Server version – String）
- 线程 id - `Int<4>` （Thread id – Int<4>）
- 盐值1 - `String` （Salt1 – String）
- 服务端能力 - `Int<2>` （Server capabilities – Int<2>）
- 服务端语言 - `Int<1>` （Server language – Int<1>）
- 服务端状态 - `Int<2>` （Server Status – Int<2>）
- 扩展服务端能力 - `Int<2>` （Extended Server Capabilities – Int<2>）
- 身份认证插件长度 - `Int<1>` （Authentication plugin length – Int<1>）
- 保留字节 - `10` 字节 （Reserved bytes – 10 bytes）
- 盐值2 - `String` （Salt2 – String）
- 身份验证插件字符串 - `String` （Authentication plugin string – String）

服务器语言是整数数值，下表将帮助我们通过整数值选择合适的语言：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/02-Understanding-MySQL-Client-Server-Protocol-Using-Python-Wireshark-Part-1/06.webp" style="width:600px"/>

在我的测试环境中服务端语言的整数数值是 `16` 进制的 `0x08`（在十进制数值中也是 `8`）。从上表中我们可以看到 `8` 的等价内容是 `latin1_swedish_ci`，所以我们知道服务器的默认语言是 `latin1_swedish_ci`。

服务器能力和服务器状态同样是整数类型，但是通过读取这些整数的每个 `BIT`，我们可以了解服务器的能力和状态。下图描述了服务器功能和状态位：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/02-Understanding-MySQL-Client-Server-Protocol-Using-Python-Wireshark-Part-1/07.webp" style="width:600px"/>

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/02-Understanding-MySQL-Client-Server-Protocol-Using-Python-Wireshark-Part-1/08.webp" style="width:600px"/>

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/02-Understanding-MySQL-Client-Server-Protocol-Using-Python-Wireshark-Part-1/09.webp" style="width:600px"/>

在握手数据包客户端准备将**登录请求包**发送到服务器进行身份验证。现在让我们研究一下登录请求包：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/02-Understanding-MySQL-Client-Server-Protocol-Using-Python-Wireshark-Part-1/10.webp" style="width:600px"/>

- 前 `3` 字节描述了载荷的长度；
- 接下来的 `1` 字节是数据包序号；
- 客户端能力 - `Int<2>` / 与服务器功能相同
- 客户端扩展能力 - `Int<2>` / 与服务器扩展能力功能相同
- 最大数据包 - `Int<4>` / 描述了数据包的最大长度
- 字符类型 - `Int<1>` / 在我的例子中它是 `16` 进制的 `0x21`（十进制数字系统是 `33`），从表中我们可以看到它是 `utf8_general_ci`，这里将服务器的默认字符集从 `latin1_swedish_ci` 设置为 `utf8_general_ci`。
- 用户名 - `String`
- 密码 - `String`
- 客户端身份验证插件字符串 - `String`

正如你所看到的密码是被加密的。为了加密密码，我们可以使用 `SHA1`、`MD5` 算法，以及从服务器发送的先前握手包中的 `salt1` 和 `salt2` 字符串作为盐值，为密码内容进行加密。

最后如果我们验证成功，我们会从服务器获得 `OK` 数据包；否则我们会得到 `ERR` 数据包。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/02-Understanding-MySQL-Client-Server-Protocol-Using-Python-Wireshark-Part-1/11.webp" style="width:600px"/>

- `3` 字节是数据包的长度；
- `1` 字节时数据包的序号；
- 受影响行数 - `Int<1>`；
- 服务器状态 - `Int<2>`；
- 警告信息 - `Int<2>`。

就这样我们已经完成了理论知识的学习，现在是时候开始开始进行实践了。在本文的第二部分，我们将不使用外部模块或库从头开始编写我们自己的 `MySQL` 本地客户端。
