---
title: 使用 Python 和 Wireshark 理解 MySQL 客户端/服务器协议：第 2 部分
date: 2022-04-26 20:03:22
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

`next` 方法从二进制响应数据中读取了一部分字节的数据。当我们调用此方法时，它会读取部分字节并将指针指向读取结束的最后位置（更改 self.start 和 self.end 属性的值）。当我们再次调用这个方法时，它会从上次停止的位置开始读取字节数据。

`next` 方法接收五个参数：`length`、`type`、`byteorder`、`signed` 和 `freeze`。如果 `freeze` 参数是 `True`，它将会从二进制响应中读取一部分字节数据，但是并不会改变指针的位置；否则的话，它将读取指定长度的一部分字节数据，并同时修改指针的位置。如果 `length` 参数为指定，则方法读取字节，直到响应字节数组结束。`type` 参数可以是 `int`、`str` 和 `hex` 数据类型。方法 `next` 根据类型参数的值将一部分字节转换为适当的数据类型。

参数 `byteorder` 确定字节到整数类型的转换，这取决于您计算机的体系结构，如果你的机器是 `big-endian`，那么它会在内存中存储从大地址到小地址的字节。如果您的机器是 `little-endian`，那么它会将字节存储在内存中，从小地址到大地址。这就是为什么我们必须知道我们架构的确切类型才能正确地将字节转换为整数。在我的例子中，它是 `little-endian`，这就是为什么我将 `byteorder` 参数的默认值设置为“`little`”。

参数 `signed` 同样用于字节到整数类型的转换，我们告诉函数将每个整数视为无符号整数或有符号整数。

这个类中第二个有意思的方法是：`encrypt_password`。这个方法使用指定的算法来加密密码。

```Python
from hashlib import sha1
    def encrypt_password(self, salt, password):
        bytes1 = sha1(password.encode("utf-8")).digest()
        concat1 = salt.encode('utf-8')
        concat2 = sha1(sha1(password.encode("utf-8")).digest()).digest()
        bytes2 = bytearray()
        bytes2.extend(concat1)
        bytes2.extend(concat2)
        bytes2 = sha1(bytes2).digest()
        hash = bytearray(x ^ y for x, y in zip(bytes1, bytes2))
        return hash
```

这个方法接收两个参数：`salt` 和 `password`。参数 `salt` 是从服务器接收到的握手数据中的两个 `salt1` 和 `salt2` 字符串的串联。参数 `password` 是 `MySQL` 用户的密码字符串。

在[官方文档](https://dev.mysql.com/doc/internals/en/secure-password-authentication.html)中密码的加密算法是：

```sql
SHA1( password ) XOR SHA1( "20-bytes random data from server" <concat> SHA1( SHA1( password ) ) )
```

在这里 **”20-bytes random data from server“** 是从服务器接收到的握手数据中的两个 `salt1` 和 `salt2` 字符串的串联。要想知道握手包是什么内容，请看上一篇文章。

接下来我将会逐行解释一下 `encrypt_password` 方法的内容。

```
bytes1 = sha1(password.encode(“utf-8”)).digest()
```

我们将密码字符串转换为字节，然后使用 `sha1` 函数进行加密，并将结果赋值给 `bytes1` 变量。它等价与算法的这一部分：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/03-Understanding-MySQL-Client-Server-Protocol-Using-Python-Wireshark-Part-2/01.webp" style="width:600px"/>

接下来我们将 `salt` 字符串转换为字节，并赋值给 `concat1` 变量：

```
concat1 = salt.encode(‘utf-8’)
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/03-Understanding-MySQL-Client-Server-Protocol-Using-Python-Wireshark-Part-2/02.webp" style="width:600px"/>

方法的第三行是：

```
concat2 = sha1(sha1(password.encode(“utf-8”)).digest()).digest()
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/03-Understanding-MySQL-Client-Server-Protocol-Using-Python-Wireshark-Part-2/03.webp" style="width:600px"/>

在这里我们使用 `sha1` 函数对密码字符串进行两次加密，并将结果赋值给 `concat2` 字符串变量。

现在我们有了两个变量：`concat1` 和 `concat2`。我们需要使用一个字节数组将他们连接到一起：

```
bytes2 = bytearray()
bytes2.extend(concat1)
bytes2.extend(concat2)
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/03-Understanding-MySQL-Client-Server-Protocol-Using-Python-Wireshark-Part-2/04.webp" style="width:600px"/>


接下来我们需要使用 `sha1` 函数对连接后字节进行加密，并将结果赋值给 `bytes2` 变量：

```
bytes2 = sha1(bytes2).digest()
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/03-Understanding-MySQL-Client-Server-Protocol-Using-Python-Wireshark-Part-2/05.webp" style="width:600px"/>


现在我们已经有了两个加密的字节变量：`bytes1` 和 `bytes2`。现在我们必须在这些变量之间进行按位异或运算并返回获得的哈希值。

```
hash=bytearray(x ^ y for x, y in zip(bytes1, bytes2))
return hash
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2022/03-Understanding-MySQL-Client-Server-Protocol-Using-Python-Wireshark-Part-2/06.webp" style="width:600px"/>


