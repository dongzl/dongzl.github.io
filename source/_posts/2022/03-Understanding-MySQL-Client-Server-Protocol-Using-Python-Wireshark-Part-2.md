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

## 数据类型 Class

在前面的文章中我们学习了 `MySQL` 客户端 / 服务端协议中的有关 `Int` 和 `String` 数据类型内容。现在我们需要一些 `Class` 来从接收到的数据包中读取字段。

### INT 类

```Python
class Int:
    """see documentation: https://dev.mysql.com/doc/internals/en/integer.html"""

    def __init__(self, package, length=-1, type='fix'):
        self.package = package
        self.length = length
        self.type = type
            

    def next(self):
        # int<n>
        if self.type == 'fix' and self.length > 0:
            return self.package.next(self.length)
        # int<lenenc>
        if self.type == 'lenenc':
            byte = self.package.next(1)
            if byte < 0xfb:
                return self.package.next(1)
            elif byte == 0xfc:
                return self.package.next(2)
            elif byte == 0xfd:
                return self.package.next(3)
            elif byte == 0xfe:
                return self.package.next(8)
```


`Int` 类实现了 `MySQL` 客户端 / 服务端协议中的 `INT` 数据类型。它在初始化时接受数据包参数，参数包应该是继承自 `MYSQL_PACKAGE` 类的任何包类的实例。`next` 方法决定了 `Integer` 的类型（`int<fix>` 类型或者是 `int<lenenc>` 类型. 请参考上一篇文章内容）并且调用包对象中的 `next` 方法来读取接收到的响应的字节部分。

### STR 类

```Python
class Str:
    """see documentation: https://dev.mysql.com/doc/internals/en/string.html"""
    def __init__(self, package, length = -1, type="fix"):
        self.package = package
        self.length = length
        self.type = type

    def next(self):
        # string<fix>
        if self.type == 'fix' and self.length > 0:
            return self.package.next(self.length, str)
        # string<lenenc>
        elif self.type == 'lenenc':
            length = self.package.next(1)
            if length == 0x00:
                return ""
            elif length == 0xfb:
                return "NULL"
            elif length == 0xff:
                return "undefined"
            return self.package.next(length, str)
        # string<var>
        elif self.type == 'var':
            length = Int(self.package, type='lenenc').next()
            return self.package.next(length, str)
        # string<eof>
        elif self.type == 'eof':
            return self.package.next(type=str)
        # string<null> - null terminated strings
        elif self.type == 'null':
            strbytes = bytearray()

            byte = self.package.next(1)
            while True:
                if byte == 0x00:
                    break
                else:
                    strbytes.append(byte)
                    byte = self.package.next(1)
            
            return strbytes.decode('utf-8')
```

`Str` 类实现了 `MySQL` 客户端 / 服务端协议中的 `STRING` 数据类型。它在初始化时接受数据包参数，参数包应该是继承自 `MYSQL_PACKAGE` 类的任何包类的实例。`next` 方法决定了 `String` 的类型（`String<fix>`、`String<Var>`、`String<NULL>`、`String<EOF>` 或者是 `String<lenenc>`。请参考上一篇文章内容）并且调用包对象中的 `next` 方法来读取接收到的响应的字节部分。

### HANDSHAKE_PACKAGE 类

`HANDSHAKE_PACKAGE` 类用来解析从服务端接收到的握手数据包。它继承自 `MYSQL_PACKAGE` 类并且在初始化时接收 `resp` 参数。参数 `resp` 是从服务端接收的以字节为单位的握手包响应数据。

```Python
class HANDSHAKE_PACKAGE(MYSQL_PACKAGE):
    def __init__(self, resp):
        super().__init__(resp)
    
    def parse(self):
        return {
            "package_name": "HANDSHAKE_PACKAGE",
            "package_length": Int(self, 3).next(), #self.next(3),
            "package_number": Int(self, 1).next(), #self.next(1),
            "protocol": Int(self, 1).next(), #self.next(1),
            "server_version": Str(self, type='null').next(),
            "connection_id": Int(self, 4).next(), #self.next(4),
            "salt1": Str(self, type='null').next(),
            "server_capabilities": self.get_server_capabilities(Int(self, 2).next()),
            "server_language": self.get_character_set(Int(self, 1).next()),
            "server_status": self.get_server_status(Int(self, 2).next()),
            "server_extended_capabilities": self.get_server_extended_capabilities(Int(self, 2).next()),
            "authentication_plugin_length": Int(self, 1).next(),
            "unused": Int(self, 10).next(), #self.next(10, hex),
            "salt2": Str(self, type='null').next(),
            "authentication_plugin": Str(self, type='eof').next()
        }
```

方法 `parse` 使用 `Int` 和 `Str` 类从响应中读取属性内容，并将他们存入字典中做为结果返回。

### LOGIN_PACKAGE 类

这个类用于创建登录请求数据包。

```Python
class LOGIN_PACKAGE(MYSQL_PACKAGE):
    def __init__(self, handshake):
        super().__init__()
        self.handshake_info = handshake.parse()
    
    def create_package(self, user, password, package_number):
        package = bytearray()
        # client capabilities
        package.extend(self.capabilities_2_bytes(self.client_capabilities))
        # extended client capabilities
        package.extend(self.capabilities_2_bytes(self.extended_client_capabilities))
        # max package -> 16777216
        max_package = (16777216).to_bytes(4, byteorder='little')
        package.extend(max_package)
        # charset -> 33 (utf8_general_ci)
        package.append(33)
        # 23 bytes are reserved
        reserved = (0).to_bytes(23, byteorder='little')
        package.extend(reserved)
        # username (null byte end)
        package.extend(user.encode('utf-8'))
        package.append(0)
        # password
        salt = self.handshake_info['salt1'] + self.handshake_info['salt2']
        encrypted_password = self.encrypt_password(salt.strip(), password)
        length = len(encrypted_password)
        package.append(length)
        package.extend(encrypted_password)
        # authentication plugin
        plugin = self.handshake_info['authentication_plugin'].encode('utf-8')
        package.extend(plugin)

        finpack = bytearray()
        package_length = len(package)
        
        finpack.append(package_length)
        finpack.extend((0).to_bytes(2, byteorder='little'))
        finpack.append(package_number)
        finpack.extend(package)

        return finpack
```

`OK` 数据包和 `ERR` 数据包是在服务端认证之后或者向服务端发送查询命令阶段后服务端反馈的响应数据包。

```Python
class OK_PACKAGE(MYSQL_PACKAGE):
    def __init__(self, resp):
        super().__init__(resp)

    def parse(self):
        return {
            "package_name": "OK_PACKAGE",
            "package_length": Int(self, 3).next(), #self.next(3),
            "package_number": Int(self, 1).next(), #self.next(1),
            "header": hex(Int(self, 1).next()),
            "affected_rows": Int(self, 1).next(), #self.next(1),
            "last_insert_id": Int(self, 1).next(), #self.next(1),
            "server_status": self.get_server_status(Int(self, 2).next()),
            "warnings": Int(self, 2).next()
        }
```

```Python
class ERR_PACKAGE(MYSQL_PACKAGE):
    def __init__(self, resp):
        super().__init__(resp)

    def parse(self):
        return {
            "package_name": "ERR_PACKAGE",
            "package_length": Int(self, 3).next(), #self.next(3),
            "package_number": Int(self, 1).next(), #self.next(1),
            "header": hex(Int(self, 1).next()), #self.next(1, hex),
            "error_code": Int(self, 2).next(), #self.next(2),
            "sql_state": Str(self, 6).next(),
            "error_message": Str(self, type='eof').next()
        }
```

### MYSQL 类

`MYSQL` 类是是创建服务端 `TCP` 连接的包装类，使用上述类从服务端发送和接收包。

```Python
def __enter__(self):
    self.client = socket(AF_INET, SOCK_STREAM)
    ip = gethostbyname(self.host)
    address=(ip,int(self.port))
    self.client.connect(address)
    return self

def __exit__(self, exc_type, exc_value, traceback):
    print("Good Bye!")
    self.close()

def close(self):
    self.client.close()
```

```Python
class MySQL: 
    def __init__(self, host="", port="", user="", password=""):
        self.host = host
        self.port = port
        self.user = user
        self.password = password
    
    def connect(self):
        resp = self.client.recv(65536)
        return HANDSHAKE_PACKAGE(resp)

    def login(self, handshake_package, package_number):
        """Sending Authentication package"""
        login_package = LOGIN_PACKAGE(handshake_package)
        package = login_package.create_package(user=self.user, password=self.password, package_number=package_number)
        self.client.sendall(package)
        resp = self.client.recv(65536)
        package = self.detect_package(resp)
        if isinstance(package, ERR_PACKAGE):
            info = package.parse()
            raise Exception(f"MySQL Server Error: {info['error_code']} - {info['error_message']}")
        elif isinstance(package, OK_PACKAGE):
            return package.parse()['package_number']
        elif isinstance(package, EOF_PACKAGE):
            return False
```

我认为这个类一切都很清晰了。我已经定义了 `__enter__` 和 `__exit__` 以便能够使用带有“with”语句的这个类来自动关闭 `TCP` 连接。在 `__enter__` 方法中我正在通过套接字创建 `TCP` 连接，在 `__exit__` 方法中去关闭创建的连接。这个类在初始化中接收 `host`、`port`、`user` 和 `password` 参数。

在连接方法中我们从服务端接收了握手数据包：

```
resp = self.client.recv(65536)
return HANDSHAKE_PACKAGE(resp)
```

在登录方法中我们使用 `LOGIN_PACKAGE` 和 `HANDSHAKE_PACKAGE` 类创建登录请求数据包，并发送到服务端，同时从服务端获取 `OK` 或者 `ERR` 数据包。

就这样。我们已经实现了连接阶段，为了避免这篇文章太长，我不会解释命令阶段，因为命令阶段比连接阶段更容易，您可以使用从本文和以前的文章中积累的知识自行研究。

示例视频：

<iframe width="560" height="315" src="https://www.youtube.com/embed/lO81kjtdTYc" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
