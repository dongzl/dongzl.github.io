---
title: （进行中）Web 应用程序中用于身份认证和授权的 JWT、OAuth 和 SAML 之间的差异对比？
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

朋友们大家好，现在 Java 开发人员面试中最常见的问题之一是 JWT、OAuth2.0 和 SAML 之间的区别？以及何时使用它们。如果您正在准备 Java 开发人员面试并提出这个问题并寻找答案，那么您来对地方了。

过去，我分享了一些 Java 面试资源，例如[21 个软件设计模式问题](https://medium.com/javarevisited/21-software-design-pattern-interview-questions-and-answers-b7d1774b5dd2)、[10 个基于微服务场景的问题](https://medium.com/javarevisited/top-10-microservices-problem-solving-questions-for-5-to-10-years-experienced-developers-3391e4f6b591)、[20 个面试 SQL 查询](https://medium.com/javarevisited/20-sql-queries-for-programming-interviews-a7b5a7ea8144)、[50 个微服务问题](https://medium.com/javarevisited/50-microservices-interview-questions-for-java-programmers-70a4a68c4349)、[60 个树数据结构问题](https://medium.com/javarevisited/top-60-tree-data-structure-coding-interview-questions-every-programmer-should-solve-89c4dbda7c5a)、[15 个系统设计问题](https://medium.com/javarevisited/7-system-design-problems-to-crack-software-engineering-interviews-in-2023-13a518467c3e)和 [35 个核心 Java问题](https://medium.com/javarevisited/top-10-java-interview-questions-for-3-to-4-years-experienced-programmers-c4bf6d8b5e7b)和 [21 个 Lambda 和 Stream 问题](https://medium.com/javarevisited/21-lambda-and-stream-interview-questions-for-java-programmers-38d7e83b5cac)，在本文中，我将一劳永逸地回答这个常见问题。

虽然 JWT、OAuth 和 SAML 都是众所周知的标准，用于 Web 应用程序中的身份验证和授权目的，但它们之间存在许多差异。

例如，JWT 代表 JSON Web Token，它是*作为 JSON 对象在各方之间安全传输信息的标准*。它用于对用户进行身份验证和授权，常用于现代 Web 应用程序。 JWT 是经过数字签名的，所以它们可以被验证和信任。

另一方面，OAuth（开放授权）是一种*开放的授权标准，允许第三方应用程序访问用户数据而无需用户共享其登录凭据*。它通常用于需要从外部服务访问数据的应用程序，例如社交媒体平台或 API。

同样，SAML（安全断言标记语言）是另*一种用于在各方之间交换身份验证和授权数据的标准，特别是在身份提供者 (IdP) 和服务提供者 (SP) 之间*。它通常用于企业应用程序中以提供单点登录 (SSO) 功能。

SAML 最流行的示例之一是 SingPass 身份验证，它使用新加坡政府访问政府网站，如疫苗证书、CPF、IRAS 等。

现在，我们了解了基础知识，是时候深入了解并更详细地学习它们，以便您可以回答任何后续问题。

<hr />

## 什么是 JWT（JSON 网络令牌）？什么时候使用它？

正如我所说，JWT 代表 JSON Web Token，这是一种用于在各方之间安全传输信息的令牌。 JWT 通常用于 Web 应用程序中的身份验证和授权目的。

JWT 由三部分组成：标头、有效负载和签名。标头指定令牌的类型和使用的签名算法，而有效负载包含正在传输的实际数据。

> 签名是通过将标头和有效负载与只有服务器知道的密钥组合在一起创建的。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/17-Difference-JWT-OAuth-SAML-Authentication-Authorization/01.png"/>

JWT 通常用于 Web 应用程序中，作为在客户端和服务器之间传输用户身份验证数据的一种方式。当用户登录时，服务器会生成一个包含用户 ID 和任何相关权限或角色的 JWT。然后将此令牌发送回客户端，将其存储并包含在对服务器的后续请求中。

> 然后服务器可以验证 JWT 的真实性并使用其中包含的信息来确定用户是否被授权执行请求的操作。

JWT 在多个服务需要共享用户身份验证信息的[分布式系统](https://medium.com/javarevisited/how-to-manage-transactions-in-distributed-systems-and-microservices-d66ff26b405e)中也很有用。不是每个服务都维护自己的身份验证系统，而是可以使用单个 JWT 对所有服务中的用户进行身份验证。

总的来说，当您需要以安全高效的方式在各方之间传输敏感信息时，JWT 非常有用，尤其是在传统的基于会话的身份验证不可行的情况下。

这是一个很好的图表，它解释了 JWT（JSON Web 令牌）如何在 Web 应用程序中工作：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/17-Difference-JWT-OAuth-SAML-Authentication-Authorization/02.png"/>

<hr />

## 什么是OAuth2.0？什么时候使用它？

[OAuth2.0](https://oauth.net/2/) 是一种用于 Web 和移动应用程序中的授权和身份验证的协议。它允许用户授权第三方应用程序访问他们的资源，例如他们的社交媒体帐户或其他在线服务，而无需泄露他们的凭据或密码。

OAuth2.0通过在用户、第三方应用程序和资源服务器之间建立信任关系来工作。该过程涉及几个步骤：

1. 用户通过尝试访问资源服务器上的受保护资源来启动该过程，例如登录社交媒体帐户。
2. 资源服务器通过将用户重定向到授权服务器进行响应，他们可以在授权服务器上授予第三方应用程序访问其资源的权限。
3. 然后用户登录到授权服务器并授予第三方应用程序访问其资源的权限。
4. 授权服务器生成访问令牌并将其发送给第三方应用程序。
5. 第三方应用程序使用访问令牌来请求和访问资源服务器上用户的资源。

OAuth2.0 在用户想要授予第三方应用程序访问其资源但又不想泄露其凭据或密码的情况下非常有用。它通常用于 Web 和移动应用程序，尤其是那些依赖外部 API 或服务来获取数据和功能的应用程序。

这是一个很好的图表，它解释了 OAuth2.0 的工作流程和工作方式：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/17-Difference-JWT-OAuth-SAML-Authentication-Authorization/03.jpg"/>

OAuth2.0 还有助于确保用户保留对其资源的控制权并可以随时撤销访问权限。它为用户提供了一种安全和标准化的方式来与第三方应用程序共享他们的数据和资源，同时维护他们的隐私和安全。

例如，现在大多数网站都允许您使用 Twitter、Facebook 或 Google 帐户登录，您无需创建新的用户名或密码，也无需共享您的 Google、Twitter 或 Facebook 密码即可访问任何第三方网站上的资源，但您仍然可以使用 OAuth 使用它。

<hr />

## 什么是 SAML？什么时候使用它？

SAML 代表安全断言标记语言，是一种基于 XML 的协议，用于在身份提供者 (IdP) 和服务提供者 (SP) 等各方之间交换身份验证和授权数据。

SAML 通过在 IdP 和 SP 之间建立信任关系来工作。 IdP 负责对用户进行身份验证并生成 SAML 断言，其中包含有关用户身份和身份验证状态的信息。 SP 依靠 SAML 断言做出授权决定并授予对受保护资源的访问权限。

SAML 可用于多种场景，例如单点登录 (SSO) 和联合。

在 SSO 中，SAML 用于允许用户使用一组凭据访问多个应用程序或服务。当用户登录到一个应用程序或服务时，IdP 会生成一个 SAML 断言，该断言可用于对其他应用程序或服务的用户进行身份验证，而无需用户再次登录。

在联合中，SAML 用于启用组织之间的信任关系，允许用户使用一组凭据访问多个组织的资源。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/17-Difference-JWT-OAuth-SAML-Authentication-Authorization/04.png"/>

SAML 通常用于企业环境，用户需要跨不同域或组织访问多个应用程序和服务。它提供了一种标准化的方式来交换身份验证和授权数据，从而更容易管理访问控制并确保敏感资源的安全。

SAML 使用的一个流行示例是新加坡政府的 SingPass 身份验证系统，它允许您使用 Singapss 登录访问所有政府网站，如 CPF 和 IRAS。

总的来说，SAML 是一个强大的工具，可以跨不同的组织和应用程序安全、无缝地访问资源，使其成为许多企业和组织的热门选择

## Web 应用程序中用于身份验证和授权的 JWT、OAuth 和 SAML 之间的区别

现在我们知道了它们是什么、它们如何工作以及它们的使用位置，是时候修改它们之间的主要区别了。以下是 JWT、OAuth 和 SAML 在点格式方面的一些主要区别：

### JWT

- JWT 是一种基于令牌的身份验证机制。
- 它用于在各方之间传输声明（用户身份、权限等）。
- 它不依赖于集中式身份验证服务器或会话状态。
- 它通常用于单页应用程序 (SPA) 和移动应用程序。
- 它可用于身份验证和授权。

### OAuth

- OAuth 是一种用于 Web 和移动应用程序中的授权和身份验证的协议。
- 它用于代表用户授予第三方应用程序对资源的访问权限。
- 它依赖于集中式授权服务器。
- 它通常用于依赖外部 API 或服务获取数据和功能的应用程序。
- 它用于授权，而不是身份验证。

### SAML

- SAML 是一种用于在身份提供者 (IdP) 和服务提供者 (SP) 等各方之间交换身份验证和授权数据的协议。
- 它用于在组织之间建立信任关系并启用单点登录 (SSO) 和联合。
- 它依赖于集中式身份提供者 (IdP)。
- 它通常用于企业环境。
- 它用于身份验证和授权。

这是一个很好的表格，您可以打印它来记住 JWT、SAML 和 OAuth2.0 之间的这些差异。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/17-Difference-JWT-OAuth-SAML-Authentication-Authorization/05.png"/>

综上所述，JWT 是一种基于令牌的身份验证机制，用于传递声明，OAuth 是一种用于代表用户授予第三方应用程序访问资源的协议，而 SAML 是一种用于在各方之间交换身份验证和授权数据的协议建立组织间的信任关系。

<hr />

这就是 JWT、OAuth 和 SAML 在身份验证和授权方面的区别。简而言之，JWT 是一种在各方之间安全传输数据的标准，而 OAuth 是一种允许第三方应用程序访问用户数据的授权标准。

SAML 是用于在 IdP 和 SP 之间交换身份验证和授权数据的标准，通常用于企业应用程序。这些标准中的每一个都有不同的用途，它们可以一起使用，为 Web 应用程序提供安全高效的身份验证和授权过程。

此外，你还可以准备一些微服务问题，比如[API Gateway 和 Load Balancer 的区别](https://medium.com/javarevisited/difference-between-api-gateway-and-load-balancer-in-microservices-8c8b552a024)，[SAGA 模式](https://medium.com/javarevisited/what-is-saga-pattern-in-microservice-architecture-which-problem-does-it-solve-de45d7d01d2b)，[如何在微服务中管理事务](https://medium.com/javarevisited/how-to-manage-transactions-in-distributed-systems-and-microservices-d66ff26b405e)，以及 [SAGA 和 CQRS 模式的区别](https://medium.com/javarevisited/difference-between-saga-and-cqrs-design-patterns-in-microservices-acd1729a6b02)，它们在面试中很受欢迎。