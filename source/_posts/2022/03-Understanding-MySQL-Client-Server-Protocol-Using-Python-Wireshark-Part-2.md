---
title: 使用 Python 和 Wireshark 理解 MySQL 客户端/服务器协议：第 2 部分
date: 2022-04-24 20:03:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/mysql_python.png
# author information, multiple authors are set to array
# single author
author:
  - nick: Elshad Agayev
    link: https://www.turing.com/blog/author/elshad-agayev/

# post subtitle in your index page
subtitle: 本文是一篇翻译文章，在这篇博客文章中，我们将学习如何在不使用连接器或外部库的情况下从头开始编写我们自己的原生 MySQL 客户端。

categories: 
  - 数据库

tags: 
  - MySQL
  - Wireshark
---

> 原文链接：https://www.turing.com/blog/understanding-mysql-client-server-protocol-using-python-and-wireshark-part-2/

在上一篇文章中我们使用 Wireshark 工具研究了 MySQL 客户端 / 服务端协议内容。现在我们开始使用 Python 语言编码来开发一个模拟 MySQL 原生客户端的工具。最终的代码在这里：[Github repo](https://github.com/elshadaghazade/mysql_native_client_in_python)

首先我们需要创建一个 `MYSQL_PACKAGE` 类。`MYSQL_PACKAGE` 类是其他 `package` 类（例如：`HANDSHAKE_PACKAGE`, `LOGIN_PACKAGE`, `OK_PACKAGE` 等等)的父类。

```Python
class MYSQL_PACKAGE:
    """Data between client and server is exchanged in packages of max 16MByte size."""  

    def __init__(self, resp = b''):
        self.resp = resp
        self.start = 0
        self.end = 0
```

它在初始化时接受 `resp` 参数。 `Resp` 是从服务器接收到的字节数组类型的二进制响应。这个类的一个重要而有趣的方法是 `next` 方法。

```Python
def next(self, length = None, type=int, byteorder='little', signed=False, freeze=False):
    if not freeze:
        if length:
            self.end += length
            portion = self.resp[self.start:self.end]
            self.start = self.end
        else:
            portion = self.resp[self.start:]
            self.start = self.end = 0
    else:
        if length:
            portion = self.resp[self.start:self.start + length]
        else:
            portion = self.resp[self.start:]

    if type is int:
        return int.from_bytes(portion, byteorder=byteorder, signed=signed)
    elif type is str:
        return portion.decode('utf-8')
    elif type is hex:
        return portion.hex()
    else:
        return portion
```


