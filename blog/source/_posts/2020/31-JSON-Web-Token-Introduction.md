---
title: （译）JSON Web Token 介绍
date: 2020-06-20 11:08:31
cover: https://gitee.com/dongzl/article-images/raw/master/cover/jwt_study.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文是 [JWT 官网](https://jwt.io/) 中对 JSON Web Token 工具英文介绍的翻译内容，旨在对 JSON Web Token 工具有一个出版认识。

categories: 
  - web开发

tags: 
  - JWT
---

> 原文链接：[Introduction to JSON Web Tokens](https://jwt.io/introduction/)

> 最新消息：免费获取[JWT用户手册](https://auth0.com/resources/ebooks/jwt-handbook?_ga=2.202584857.907178229.1592620695-1260208969.1592620695)，深入学习JWT。

### JSON Web Token 是什么

JSON Web Token (JWT) 是一个开放标准（[RFC 7519](https://tools.ietf.org/html/rfc7519)），它定义了一种紧凑且自包含的方式，用于作为 JSON 对象在各部分之间来进行安全传输信息。由于这些信息进行了数字签名，所以这些信息是可以被验证和被信任的。JWT 使用加密方式（使用 HMAC 算法）或者是使用 RSA 和 ECDSA 等公钥/私钥对来进行签名。

尽管可以对 JWT 进行加密以提供双方之间的保密性，但我们将重点关注已签名的令牌。签名的令牌可以验证其中包含的声明的完整性，而加密的令牌则将这些声明隐藏在其他方的面前。当使用公钥/私钥对对令牌进行签名时，签名还证明只有持有私钥的一方才是对其进行签名的一方。

### 什么时候需要使用 JSON Web Tokens

这里列举了一些 JSON Web Tokens 的使用场景：

- **授权**：这个是 JWT 做常见的使用场景。一旦用户登录以后，随后的所有请求都将包含 JWT，允许用户访问这个 token 所允许访问的路径、服务和资源。单点登录（SSO）是目前 JWT 被广泛使用的一项功能，因为它的开销很小并且可以在不同的域中轻松使用。

- **信息交换**：JSON Web Tokens 是一种在多个部分之间进行安全信息传输非常好的一种交互方式。因为可以对 JWT 进行签名（例如：使用公钥/私钥对进行签名），我们就可以确定发送方就是他们所说的他们自己。另外，由于签名是可以使用header 和 payload 信息进行计算的，因为我们还可以验证内容是否被篡改过。

### JSON Web Token 的结构

JSON Web Token 以紧凑的形式由三部分组成，这些部分由`.`分隔，分别是：

- Header
- Payload
- Signature

因此，JWT 结构通常如下所示：

```
xxxxx.yyyyy.zzzzz
```

下面我们来拆解一下不同部分的内容。

#### Header

header 通常包含两部分内容：token 的类型，一般就是 JWT；另一部分是签名使用的算法，例如 `HMAC`、`SHA256` 或者是 `RSA`。

例如：

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

然后，将 JSON 的内容进行 Base64 编码后成为 JWT 的第一部分内容。

#### Payload

JSON Web Token 的第二部分内容是 Payload，这里面包含了声明的信息。声明是有关实体（通常是用户信息）和其他数据的一些声明内容。共有三种类型的声明：注册的声明，公共的声明和私有的声明。

- **Registered claims**：这里有一系列预先定义好的声明，这些声明不是强制的，但是是被推荐使用的的，这其中包括了一系列有用的，可互操作的声明。包括：iss (发行人), exp (到期时间), sub (主题), aud (受众), 和 [其他一些类型声明](https://tools.ietf.org/html/rfc7519#section-4.1).

**PS. 请注意，声明名称仅可以是三个字符，因为 JWT 是紧凑的。**

- **Public claims**：这些可以由使用 JWT 的人随意定义。但是为了避免冲突，应在 [IANA JSON Web](https://www.iana.org/assignments/jwt/jwt.xhtml) 令牌注册表中定义它们，或将其定义为包含抗冲突名称空间的 URI。

- **Private claims**：这些是自定义声明，旨在用于同意使用这些声明的各方之间共享信息，它既不是注册声明也不是公共声明。

payload 示例如下：

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

在对 payload 内容进行 Base64 编码之后，它成为 JWT 的第二部分内容。

**PS. 请注意，对于已签名的令牌，此信息尽管可以防止被篡改，但任何人都可以读取这些信息。除非将其加密，否则请勿将机密信息放入 JWT 的 header 或 payload 中。**

#### Signature

要创建签名部分，我们必须获取编码后的 header，编码后的 payload，密钥，header 中指定的算法，并对其进行签名。

例如，如果我们想使用 `HMAC SHA256` 算法进行签名，创建签名的方式如下：

```java
HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)
```

签名用来验证消息在传输的过程中是没有被修改过的，并且在 token 使用私钥进行签名的场景下，它可以验证 JWT 信息的发送者自身的合法性。

> The signature is used to verify the message wasn't changed along the way, and, in the case of tokens signed with a private key, it can also verify that the sender of the JWT is who it says it is.

#### Putting all together

输出的结果是由三个以 `.` 分割的 Base64 编码后字符串组成的内容，可以在HTML 和 HTTP 环境中轻松传递这些字符串，与基于 XML 的标准（例如SAML）相比，它更紧凑。

下图显示了一个 JWT 内容，它已对先前的 header 和 payload 进行了编码，并使用用一个密钥进行了签名。

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/31-JSON-Web-Token-Introduction/JSON-Web-Token-Introduction-01.png" width="800px">

如果我们想使用 JWT 并将这些概念付诸实践，我们可以使用 [jwt.io Debugger](https://jwt.io/#debugger-io) 工具解码，验证和生成 JWT 内容。

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/31-JSON-Web-Token-Introduction/JSON-Web-Token-Introduction-02.png" width="800px">

### 如何使用 JSON Web Tokens

在身份验证中，当用户使用其凭据成功登录时，将给他返回一个 JSON Web Token。由于令牌就是凭据内容，因此必须格外小心以防止安全问题。通常，令牌的保留时间不应超过要求的时间。

由于[缺乏安全性，您也不应该将敏感的会话数据存储在浏览器存储中](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html#local-storage)。

当用户想要访问受保护的路径和资源时，在 User-Agent 通常应该在 Bearer 模式中使用授权的 header 发送 JWT，header 的内容如下：

```
Authorization: Bearer <token>
```

在某些场景下，这就是一种无状态授权机制。在访问服务端中受保护的路径资源时，服务端将会检查 `Authorization header` 中有效的 JWT 信息内容，如果存在，则将允许用户访问受保护的资源。如果 JWT 包含必要的数据，则可以减少查询数据库中某些操作的需求，尽管这种情况并非总是存在。

服务器的受保护路由将在 `Authorization header` 中检查有效的 JWT，如果存在，则将允许用户访问受保护的资源。如果JWT包含必要的数据，则可以减少查询数据库中某些操作的需求，尽管这种情况并非总是如此。

如果令牌是在 `Authorization header` 中发送的，则跨域资源共享（CORS）将不在是一个问题，因为它不使用 cookie。

下面的流程图显示如何获取一个 JWT 并使用它来访问 API 或者其他资源内容：

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/31-JSON-Web-Token-Introduction/JSON-Web-Token-Introduction-03.png" width="800px">

- 应用系统或者是其他客户端向授权服务器请求授权信息。这是通过其中的一种授权流程来完成的。例如：典型的符合 [OpenID Connect](https://openid.net/connect/) 的 Web 应用程序将使用授权认证代码访问 `/oauth/authorize` 端点来完成。

- 当授权认证被授予以后，授权认证服务器会将一个可用的 token 内容返回给应用系统。

- 应用系统会使用这个 token 去访问受保护的资源（例如：某个 API）。

**PS. 请注意，使用签名令牌，令牌或令牌中包含的所有信息都会暴露给用户或其他使用方，即使他们无法更改它。这意味着我们不应将机密信息放入令牌中。**

### 为什么要使用 JSON Web Tokens

我们来讨论一下与 `Simple Web Tokens (SWT)` 和 `Security Assertion Markup Language Tokens (SAML)` 相比使用 `JSON Web Tokens (JWT) ` 的优势。

与 XML 相比，JSON 是小巧的（非冗长的），当 JSON 被编码之后它依然是非常小巧的，从而使 JWT 比 SAML 更加紧凑。这使得在 HTML 和 HTTP 环境中使用 JWT 传递数据是一个不错的选择。

在安全方面，只能使用 HMAC 算法通过共享密钥对 SWT 进行对称签名。但是，JWT 和 SAML 令牌可以使用 X.509 证书形式的共有/私有密钥对进行签名。与签名 JSON 的简单性相比，使用 XML数字签名工具 签名 XML 而不引入模糊的安全漏洞是非常困难的。

JSON 解析器在大多数编程语言中都很常见，因为它们直接映射到对象。相反，XML 没有一种天然的文档到对象映射关系。这使得使用 JWT 比使用 SAML 断言机制更加容易。

关于用法，JWT 是在 Internet 上规模使用的。这强调了在多个平台（尤其是移动平台）上对 JSON Web Token 进行客户端处理的简便性。

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/31-JSON-Web-Token-Introduction/JSON-Web-Token-Introduction-04.png" width="800px">

编码的 JWT 和编码的 SAML 的长度比较

如果您想了解有关 JSON Web Token 的更多信息，或者想开始使用它在自己的应用程序中进行身份验证，请浏览在 Auth0 上的 [JSON Web Token 页面](https://auth0.com/learn/json-web-tokens/?_ga=2.4585275.907178229.1592620695-1260208969.1592620695)。

### 原文内容

{% pdf https://gitee.com/dongzl/article-images/raw/master/2020/31-JSON-Web-Token-Introduction/JSON-Web-Token-Introduction-05.pdf %}