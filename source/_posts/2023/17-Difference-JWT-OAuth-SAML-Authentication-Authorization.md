---
title: Web 应用程序中用于身份认证和授权的 JWT、OAuth 和 SAML 之间的差异对比？
date: 2023-06-10 14:34:53
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/jwt_oauth_saml.png

# author information, multiple authors are set to array
# single author
author:
- nick: Soma
  link: https://medium.com/@somasharma_81597
- nick: 董宗磊
  link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文主要对比 Web 应用程序开发中常用于身份认证和授权的 JWT、OAuth 和 SAML 等技术框架之间的差异。

categories:
- web开发

tags:
- JWT
- OAuth
- SAML
---

> 原文链接（请科学上网）：https://medium.com/javarevisited/difference-between-jwt-oauth-and-saml-for-authentication-and-authorization-in-web-apps-75b412754127

朋友们大家好，许多 `Java` 开发人员在面试中会被问到 `JWT`、`OAuth2.0` 和 `SAML` 之间的差异，以及何时使用它们？这是最常见的问题之一，如果大家正在准备 `Java` 开发人员岗位面试并恰巧被问到这个问题，那么正好可以来这篇文章中寻找答案。

在前面的文章中，我分享了一些 `Java` 面试内容，例如 [21 个软件设计模式问题](https://medium.com/javarevisited/21-software-design-pattern-interview-questions-and-answers-b7d1774b5dd2)、[10 个基于微服务场景的问题](https://medium.com/javarevisited/top-10-microservices-problem-solving-questions-for-5-to-10-years-experienced-developers-3391e4f6b591)、[20 个面试 SQL 查询](https://medium.com/javarevisited/20-sql-queries-for-programming-interviews-a7b5a7ea8144)、[50 个微服务问题](https://medium.com/javarevisited/50-microservices-interview-questions-for-java-programmers-70a4a68c4349)、[60 个树数据结构问题](https://medium.com/javarevisited/top-60-tree-data-structure-coding-interview-questions-every-programmer-should-solve-89c4dbda7c5a)、[15 个系统设计问题](https://medium.com/javarevisited/7-system-design-problems-to-crack-software-engineering-interviews-in-2023-13a518467c3e)、[35 个核心 Java 问题](https://medium.com/javarevisited/top-10-java-interview-questions-for-3-to-4-years-experienced-programmers-c4bf6d8b5e7b) 和 [21 个 Lambda 和 Stream 问题](https://medium.com/javarevisited/21-lambda-and-stream-interview-questions-for-java-programmers-38d7e83b5cac)，在这篇文章中我将会系统地回答这个常见问题。

虽然 `JWT`、`OAuth` 和 `SAML` 都是众所周知的用于 `Web` 应用程序中的身份认证和授权的标准，但是它们之间还是存在很多差异。

比如，`JWT` 表示 `JSON Web Token`，它是*使用 `JSON` 对象在各方之间安全传输信息的标准*。它用于对用户身份进行认证和授权，通常用于 `Web` 应用程序，因为 `JWT` 是经过数字签名的，所以它们可以被验证和信任。

另一方面，`OAuth`（开放授权）是一种*开放的授权协议标准，允许第三方应用程序访问用户数据而无需共享用户登录凭证*。它通常用于外部服务需要访问数据的应用程序，例如社交媒体平台或 `API`。

同样，`SAML`（安全断言标记语言）是另一种*用于在各方之间交换身份认证和授权数据的标准，特别是在身份提供者 (`IdP`) 和服务提供者 (`SP`) 之间*。它通常用于企业应用程序中以提供单点登录 (`SSO`) 功能。

`SAML` 最著名的案例之一是被新加坡政府网站使用的 `SingPass` 身份认证服务，用来访问`疫苗证书`、`CPF`、`IRAS` 等政府网站。

现在我们对这些知识有了一些基本了解，是时候深入了解并细致地学习这些内容，以便于我们可以回答面试中关于这些技术的任何问题。

<hr />

## 什么是 JWT（JSON Web Token）？何时需要使用它？

正如前面所说，`JWT` 表示 `JSON Web Token`，这是一种用于在各方之间安全传输信息的令牌。`JWT` 通常用于 `Web` 应用程序中的身份认证和授权。

`JWT` 由三部分组成：**标头**、**有效负载**和**签名**。标头指定令牌的类型和使用的签名算法，而有效负载包含正在传输的实际数据。

> 签名是将标头和有效负载与只有服务器所信任的密钥组合在一起所生成的。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/17-Difference-JWT-OAuth-SAML-Authentication-Authorization/01.png"/>

`JWT` 通常用于 `Web` 应用程序中，作为在客户端和服务器之间传输用户身份认证数据的一种方式。当用户登录时，服务器会生成一个包含用户 `ID` 和相关权限或角色信息的 `JWT`，然后将此令牌返回给客户端，客户端将其存储后并应用到对服务器的后续请求中。

> 服务器会验证 JWT 的真实性并使用其中包含的信息来确认用户是否被授权执行请求的操作。

`JWT` 在多个服务需要共享用户身份认证信息的[分布式系统](https://medium.com/javarevisited/how-to-manage-transactions-in-distributed-systems-and-microservices-d66ff26b405e)中也很有用，每个服务不需要都维护自己的身份认证系统，而是可以使用某个 `JWT` 对所有服务中的用户进行身份认证。

总的来说，当我们需要以安全高效的方式在各方之间传输敏感信息时，`JWT` 非常有用，尤其是在传统的基于会话的身份验证不可行的情况下。

下图很好地**解释了 `JWT`（`JSON Web Token`）如何在 `Web` 应用程序中工作**：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/17-Difference-JWT-OAuth-SAML-Authentication-Authorization/02.png"/>

<hr />

## 什么是 OAuth2.0？何时需要使用它？

[OAuth2.0](https://oauth.net/2/) 是一种用于 `Web` 和移动应用程序中的授权和身份认证的协议，它允许用户授权第三方应用程序访问他们的资源，例如用户的社交媒体账户或其它在线服务，但是不会泄露他们的凭据或密码。

`OAuth2.0` 通过在用户、第三方应用程序和资源服务器之间建立信任关系来工作。该过程涉及几个步骤：

1. 用户通过尝试访问资源服务器上的受保护资源来启动该过程，例如登录社交媒体账户；
2. 资源服务器通过将用户重定向到授权服务器进行响应，它们可以在授权服务器上授予第三方应用程序访问其资源的权限；
3. 然后用户登录到授权服务器并授予第三方应用程序访问其资源的权限；
4. 授权服务器生成访问令牌并将其发送给第三方应用程序；
5. 第三方应用程序使用访问令牌来请求和访问资源服务器上用户的资源。

**`OAuth2.0` 在用户想要授予第三方应用程序访问其资源但又不想泄露其凭据或密码的情况下非常有用**。它通常用于 `Web` 和移动应用程序，尤其是那些依赖外部 `API` 或服务来获取数据和使用某些功能的应用程序。

下图很好地解释了 `OAuth2.0` 的工作流程和工作方式：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/17-Difference-JWT-OAuth-SAML-Authentication-Authorization/03.jpg"/>

`OAuth2.0` 还有助于确保用户保留对其资源的控制权并可以随时撤销访问权限。它为用户提供了一种安全和标准化的方式来与第三方应用程序共享他们的数据和资源，同时也能够维护用户的隐私和安全。

例如现在大多数网站都允许用户使用 `Twitter`、`Facebook` 或 `Google` 账户登录，这种场景下就可以使用 `OAuth` 授权访问的方式，用户无需创建新的用户名或密码，也无需共享个人的 `Google`、`Twitter` 或 `Facebook` 密码即可访问任何第三方网站上的资源。

<hr />

## 什么是 SAML？什么时候使用它？

`SAML` 表示安全断言标记语言，是一种基于 `XML` 的协议，用于在**身份提供者** (`IdP`) 和**服务提供者** (`SP`) 等各方之间交换身份认证和授权数据。

`SAML` 通过在 `IdP` 和 `SP` 之间建立信任关系来工作。`IdP` 负责对用户进行身份认证并生成 `SAML` 断言，其中包含有关用户身份和身份认证状态的信息。`SP` 依靠 `SAML` 断言做出授权决定并授予对受保护资源的访问权限。

`SAML` 可用于多种场景，例如单点登录 (`SSO`) 和联合身份管理。

在 `SSO` 中，`SAML` 用于允许用户使用一组凭据访问多个应用程序或服务。当用户登录到一个应用程序或服务时，`IdP` 会生成一个 `SAML` 断言，该断言可用于对其他应用程序或服务的用户进行身份认证，并且无需用户再次登录。

在联合身份管理中，`SAML` 用于启用组织之间的信任关系，允许用户使用一组凭据访问多个组织的资源。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/17-Difference-JWT-OAuth-SAML-Authentication-Authorization/04.png"/>

`SAML` 通常用于企业环境，用户需要跨不同域或组织访问多个应用程序和服务。它提供了一种标准化的方式来传输身份认证和授权数据，从而更容易做到访问控制管理并确保敏感资源的安全。

`SAML` 应用的一个著名案例是新加坡政府的 `SingPass` 身份验证系统，它允许用户使用 `Singapss` 系统登录访问所有政府网站，如 `CPF` 和 `IRAS`。

总的来说，`SAML` 是一个强大的工具，可以保护跨不同的组织和应用程序的安全、无缝地访问资源，使其成为许多企业和组织的热门选择。

## Web 应用程序中用于身份认证和授权的 JWT、OAuth 和 SAML 之间的差异

现在我们知道了这些技术都是什么、它们是如何工作的以及它们的使用场景，是时候对比一下它们之间的主要差异了。以下是 `JWT`、`OAuth` 和 `SAML` 在几个关键点上的主要区别：

### JWT

- `JWT` 是一种基于令牌的身份验证机制；
- 它用于在各方之间传输信息（用户身份、权限等）；
- 它不依赖于集中式身份认证服务器或会话状态；
- 它通常用于单页应用程序 (`SPA`) 或移动应用程序；
- 它可用于身份认证和授权。

### OAuth

- `OAuth` 是一种用于 `Web` 和移动端应用程序中的授权和身份认证的协议；
- 它通常用于用户授权第三方应用程序对资源的访问权限；
- 它依赖于集中式授权服务器；
- 它通常用于依赖外部 `API` 或服务获取数据和功能的应用程序；
- 它常用于授权，而不是进行身份认证。

### SAML

- `SAML` 是一种用于在身份提供者 (`IdP`) 和服务提供者 (`SP`) 等各方之间交换身份认证和授权数据的协议；
- 它用于在组织之间建立信任关系并启用单点登录 (`SSO`) 和联合身份管理；
- 它依赖于集中式身份提供者 (`IdP`)；
- 它通常用于企业环境；
- 它用于身份认证和授权。

下面的表格内容总结地很好，大家可以打印出来，帮助记忆 `JWT`、`SAML` 和 `OAuth2.0` 之间的差异。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/17-Difference-JWT-OAuth-SAML-Authentication-Authorization/05.png"/>

|       | JWT                   | OAuth                  | SAML                |
|-------|-----------------------|------------------------|---------------------|
| 目标    | 用基于令牌的身份验证机制来传输身份认证信息 | `Web` 应用和移动端应用中认证和授权协议 | 在多个模块之间交换认证和授权协议的协议 |
| 中心化服务 | 无中心化服务                | 中心化认证服务                | 中心化身份提供者            |
| 应用场景  | 认证和授权                 | 只做授权，不做认证              | 认证和授权               |
| 应用程序  | 单页应用程序，移动端应用          | 需要依赖于外部 `API` 或服务的应用程序 | 企业级应用环境             |
| 关联关系  | 各方之间直接交互              | 在第三方应用程序和资源拥有者之间交互     | 在身份提供者和服务提供者之间交互    |
| 认证能力  | 有                     | 无                      | 有                   |
| 授权能力  | 有                     | 有                      | 有                   |

综上所述，`JWT` 是一种基于令牌的身份认证机制，通常用于传递认证信息，`OAuth` 是一种用于用户授予第三方应用程序访问资源的协议，而 `SAML` 是一种用于在各方之间交换身份认证和授权数据的协议建立组织间的信任关系。

<hr />

这就是 `JWT`、`OAuth` 和 `SAML` 在身份认证和授权方面的区别。简而言之，`JWT` 是一种在各方之间安全传输数据的标准，而 `OAuth` 是一种允许第三方应用程序访问用户数据的授权标准。

`SAML` 是用于在 `IdP` 和 `SP` 之间交换身份认证和授权数据的标准，通常用于企业应用程序。这些标准中的每一个都有不同的用途，它们可以一起使用，为 `Web` 应用程序提供安全高效的身份认证和授权过程。

此外，你还可以准备一些微服务问题，比如[API Gateway 和 Load Balancer 的区别](https://medium.com/javarevisited/difference-between-api-gateway-and-load-balancer-in-microservices-8c8b552a024)，[SAGA 模式](https://medium.com/javarevisited/what-is-saga-pattern-in-microservice-architecture-which-problem-does-it-solve-de45d7d01d2b)，[如何在微服务中管理事务](https://medium.com/javarevisited/how-to-manage-transactions-in-distributed-systems-and-microservices-d66ff26b405e)，以及 [SAGA 和 CQRS 模式的区别](https://medium.com/javarevisited/difference-between-saga-and-cqrs-design-patterns-in-microservices-acd1729a6b02)，它们在面试中很受欢迎。
