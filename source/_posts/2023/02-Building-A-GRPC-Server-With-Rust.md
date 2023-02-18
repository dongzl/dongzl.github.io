---
title: 使用 Rust 一步一步构建 gRPC 服务器
date: 2023-02-10 20:03:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/rust_grpc.png
# author information, multiple authors are set to array
# single author
author:
  - nick: Yuchen Z.
    link: https://yuchen52.medium.com/

# post subtitle in your index page
subtitle: 本文是一篇翻译文章，在这篇博客文章中，我们将学习如何使用 `Rust` 语言一步一步构建一个 `gRPC` 服务端程序。

categories: 
  - 架构设计

tags: 
  - Rust
  - gRPC
---

> 原文链接：https://betterprogramming.pub/building-a-grpc-server-with-rust-be2c52f0860e

## 背景介绍

### RPC、JSON、SOAP 对比

一旦我们了解了 [gRPC](https://grpc.io/) 和 [Thrift](https://github.com/facebook/fbthrift)，就很难回到过去使用基于 `JSON` 的 `REST` `API` 或 [SOAP](https://en.wikipedia.org/wiki/SOAP) `API` 等更具过渡性的框架。

`gRPC` 和 `Thrift` 这两个著名的 [RPC](https://en.wikipedia.org/wiki/Remote_procedure_call) 框架有很多相似之处。前者源自谷歌，后者源自 `Facebook`。它们都易于使用，对各种编程语言都有很好地支持，并且性能都很好。

这两个框架最有价值的功能是支持多语言代码自动生成和服务器端反射，这些特性使得 `API` 本质上是类型安全的；通过服务器端反射，无需阅读和理解接口实现，就可以非常方便地了解 `API` 的接口定义。

### gRPC 和 Thrift 对比

[Apache Thrift](https://thrift.apache.org/) 在过去一直是一个不错的选择。但近年来由于缺乏 `Facebook` 的持续支持，再加上 [fbthrift](https://github.com/facebook/fbthrift) 的分支项目，逐渐失去了人气。

与此同时，`gRPC` 已经实现了越来越多的功能，拥有更健康的生态系统。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/02-Building-A-GRPC-Server-With-Rust/01.png" style="width:600px"/>

<font color=DarkGray size=2>gRPC（蓝）和 Apache Thrift（红）对比。[Google Trends](https://trends.google.com/trends/explore?date=all&q=GRPC,%2Fm%2F02qd4d1)</font>

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/02-Building-A-GRPC-Server-With-Rust/02.png" style="width:600px"/>

<font color=DarkGray size=2>gRPC、fbThrift 和 Apache Thrift GitHub star 历史数据。[https://star-history.com](https://star-history.com/#grpc/grpc&facebook/fbthrift&apache/thrift&Date)</font>

截至目前，除非我们的应用程序以某种方式依赖于 `Facebook`，否则没有充分的理由考虑使用 `Thrift`。

### GraphQL 怎么样？

[GraphQL](https://github.com/graphql/graphql-spec) 是另一个由 `Facebook` 发起的框架，它与上面的两个 `RPC` 框架有许多相似之处。

移动端 `API` 开发过程中最大的痛点之一是有些用户从不升级他们的 `APP`。因为我们需要保持接口向后兼容性，所以我们要么保留 `API` 中已不再使用的旧字段，要么创建 `API` 的多个版本；[GraphQL 出现的一个目的就是为解决这个问题](https://www.youtube.com/watch?v=783ccP__No8) 而被设计成一种“**查询语言**”，它允许客户端指定需要的数据字段，这个特性能够更加方便地处理接口向后兼容性。

`GraphQL` 在移动端 `API` 开发以及面向公众的 `API`（例如：`GitHub`）开发方面具有巨大优势，因为在这两种情况下，我们都无法轻易地控制客户端行为。

但是，如果我们正在为 `Web` 前端构建 `API` 或者为内部后端服务构建 `API`，那么选择 `GraphQL` 而不是 `gRPC` 几乎没有什么优势。

## Rust

以上是目前为止出现过的网络框架的一个小概述。除了网络框架，我们还需要为应用程序确定一种服务端语言。

根据 `Stack Overflow` 上的一项[调查](https://insights.stackoverflow.com/survey/2021#most-loved-dreaded-and-wanted-language-love-dread)显示：“在过去的六年时间里，`Rust` 是最受欢迎的编程语言。”尽管学习曲线相对陡峭，但是它的类型安全、优雅的内存管理、广泛的社区支持和性能，都使 `Rust` 成为一种非常有吸引力和有前途的服务端编程语言。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/02-Building-A-GRPC-Server-With-Rust/03.png" style="width:600px"/>

<font color=DarkGray size=2>Rust 是最受喜爱的编程语言。[Stack Overflow Survey 2021](https://insights.stackoverflow.com/survey/2021#most-loved-dreaded-and-wanted-language-love-dread)</font>

我们也注意到 `Rust` 在行业中得到越来越广泛的应用：[Facebook](https://engineering.fb.com/2021/04/29/developer-tools/rust/)、[Dropbox](https://www.wired.com/2016/03/epic-story-dropboxs-exodus-amazon-cloud-empire/)、[Yelp](https://www.youtube.com/watch?v=u6ZbF4apABk)、[AWS](https://aws.amazon.com/cn/blogs/opensource/sustainability-with-rust/)、[谷歌](https://opensource.googleblog.com/2021/02/google-joins-rust-foundation.html)等。很明显，`Rust` 将会持续发展，并将一直存在。

这就是我们将在今天的教程中看到的内容 -- 在 `Rust` 中使用 `gRPC` 构建一个小型服务器。

### 安装 Rust

使用如下命令来安装 `Rust`：

```shell
$ curl --proto '=https' --tlsv1.2 -sSf <https://sh.rustup.rs> | sh
```

如果之前安装了 `Rust`，我们可以通过以下方式更新它：

```shell
$ rustup update stable
```

我们需要仔细检查 `rustc`（`Rust` 编译器）和 `cargo`（`Rust` 包管理器）的安装版本：

```shell
$ rustc --version
rustc 1.60.0 (7737e0b5c 2022-04-04)
$ cargo --version
cargo 1.60.0 (d1fd9fe2c 2022-03-01)
```

有关安装的更多信息，请查看 https://www.rust-lang.org/tools/install 。

## 创建一个 Rust 项目

运行以下命令创建一个新的“Hello World”项目：

```shell
$ cargo new rust_grpc_demo --bin
```

我们来编译运行这个程序：

```shell
$ cd rust_grpc_demo
$ cargo run
   Compiling rust_grpc_demo v0.1.0 (/Users/yuchen/Documents/rust_grpc_demo)
    Finished dev [unoptimized + debuginfo] target(s) in 1.75s
     Running `target/debug/rust_grpc_demo`
Hello, world!
```

这里显示了我们到目前为止的文件结构：

```shell
$ find . -not -path "./target*" -not -path "./.git*" | sed -e "s/[^-][^\/]*\//  |/g" -e "s/|\([^ ]\)/| - \1/"
  |-Cargo.toml
  |-Cargo.lock
  |-src
  |  |-main.rs
```

### 定义 gRPC 接口

`gRPC` 使用 [Protocol Buffers](https://developers.google.com/protocol-buffers/docs/overview) 工具来序列化和反序列化数据。让我们在 `.proto` 文件中定义服务端 `API`。

```shell
$ mkdir proto
$ touch proto/bookstore.proto
```

我们定义了一个书店服务，只有一个方法：提供一本书的 `ID`，并返回关于这本书的一些详细信息。

```proto
syntax = "proto3";

package bookstore;

// The book store service definition.
service Bookstore {
  // Retrieve a book
  rpc GetBook(GetBookRequest) returns (GetBookResponse) {}
}

// The request with a id of the book
message GetBookRequest {
  string id = 1;
}

// The response details of a book
message GetBookResponse {
  string id = 1;
  string name = 2;
  string author = 3;
  int32 year = 4;
}
```

我们将使用 [tonic](https://docs.rs/tonic/latest/tonic/) 创建我们的 `gRPC` 服务。将以下依赖项添加到 `Cargo.toml` 文件中：

```toml
[package]
name = "rust_grpc_demo"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
tonic = "0.7.1"
tokio = { version = "1.18.0", features = ["macros", "rt-multi-thread"] }
prost = "0.10.1"

[build-dependencies]
tonic-build = "0.7.2"
```

为了从 `bookstore.proto` 生成 `Rust` 代码，我们在 `crate` 的 `build.rs` 构建脚本中使用 `tonic-build`。

```shell
$ touch build.rs
```

将如下内容添加到 `build.rs` 文件中：

```rust
use std::{env, path::PathBuf};

fn main() {
    let proto_file = "./proto/bookstore.proto";

    tonic_build::configure()
        .build_server(true)
        .out_dir("./src")
        .compile(&[proto_file], &["."])
        .unwrap_or_else(|e| panic!("protobuf compile error: {}", e));

    println!("cargo:rerun-if-changed={}", proto_file);
}
```

需要特别指出的是，我们添加了这个 `.out_dir("./src")` 配置，它可以设置将文件默认输出到 `src` 目录，以便我们可以更轻松地查看生成的文件，以达到本文的目的。

在我们进行编译之前还要做另外一件事，`tonic-build` 依赖于 `Protocol Buffers` 编译器，编译器可以将 `.proto` 文件解析为可以转换为 `Rust` 的表示形式。让我们安装 `protobuf`：

```shell
$ brew install protobuf
```

并仔细检查 `protobuf` 编译器是否安装正确：

```shell
$ protoc --version
libprotoc 3.19.4
```

准备编译：

```shell
$ cargo build
    Finished dev [unoptimized + debuginfo] target(s) in 0.31s
```

有了这些操作，我们应该生成一个 `src/bookstore.rs` 文件，此时项目的文件结构应该是这样的：

```shell
  | - Cargo.toml
  | - proto
  |  | - bookstore.proto
  | - Cargo.lock
  | - build.rs
  | - src
  |  | - bookstore.rs
  |  | - main.rs
```

### 服务器端实现

最后，是时候将服务内容组装到一起了，将 `main.rs` 替换为以下内容：

```rust
use tonic::{transport::Server, Request, Response, Status};

use bookstore::bookstore_server::{Bookstore, BookstoreServer};
use bookstore::{GetBookRequest, GetBookResponse};


mod bookstore {
    include!("bookstore.rs");
}


#[derive(Default)]
pub struct BookStoreImpl {}

#[tonic::async_trait]
impl Bookstore for BookStoreImpl {
    async fn get_book(
        &self,
        request: Request<GetBookRequest>,
    ) -> Result<Response<GetBookResponse>, Status> {
        println!("Request from {:?}", request.remote_addr());

        let response = GetBookResponse {
            id: request.into_inner().id,
            author: "Peter".to_owned(),
            name: "Zero to One".to_owned(),
            year: 2014,
        };
        Ok(Response::new(response))
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "[::1]:50051".parse().unwrap();
    let bookstore = BookStoreImpl::default();

    println!("Bookstore server listening on {}", addr);

    Server::builder()
        .add_service(BookstoreServer::new(bookstore))
        .serve(addr)
        .await?;

    Ok(())
}
```

如上述所见，为了简单我们实际上并没有保存书籍的数据库。在这个测试场景我们只是返回一条虚假图书记录。

服务端运行时间：

```shell
$ cargo run
   Compiling rust_grpc_demo v0.1.0 (/Users/yuchen/Documents/rust_grpc_demo)
    Finished dev [unoptimized + debuginfo] target(s) in 2.71s
     Running `target/debug/rust_grpc_demo`
Bookstore server listening on [::1]:50051
```

很高兴我们在 `Rust` 中启动并运行了我们的 `gRPC` 服务！

### 奖励：服务器端反射

如开头所述，我最初对 `gRPC` 印象深刻是因为它具有进行服务端反射的能力。这不仅使服务开发过程中得心应手，也让与前端工程师的沟通变得更加容易。因此如果不解释如何在 `Rust` 服务端代码中使用反射功能，本教程就是不完整的。

将以下内容添加到依赖项中：

```shell
tonic-reflection = "0.4.0"
```

更新 `build.rs` 文件，`// Add this` 注释标记的是需要修改的代码内容。

```rust
use std::{env, path::PathBuf};

fn main() {
    let proto_file = "./proto/book_store.proto";
    let out_dir = PathBuf::from(env::var("OUT_DIR").unwrap()); // Add this

    tonic_build::configure()
        .build_server(true)
        .file_descriptor_set_path(out_dir.join("greeter_descriptor.bin")) // Add this
        .out_dir("./src")
        .compile(&[proto_file], &["."])
        .unwrap_or_else(|e| panic!("protobuf compile error: {}", e));

    println!("cargo:rerun-if-changed={}", proto_file);
}
```

最后，将 `main.rs` 更新为以下内容。

```rust
use tonic::{transport::Server, Request, Response, Status};

use bookstore::bookstore_server::{Bookstore, BookstoreServer};
use bookstore::{GetBookRequest, GetBookResponse};


mod bookstore {
    include!("bookstore.rs");

    // Add this
    pub(crate) const FILE_DESCRIPTOR_SET: &[u8] =
        tonic::include_file_descriptor_set!("greeter_descriptor");
}


#[derive(Default)]
pub struct BookStoreImpl {}

#[tonic::async_trait]
impl Bookstore for BookStoreImpl {
    async fn get_book(
        &self,
        request: Request<GetBookRequest>,
    ) -> Result<Response<GetBookResponse>, Status> {
        println!("Request from {:?}", request.remote_addr());

        let response = GetBookResponse {
            id: request.into_inner().id,
            author: "Peter".to_owned(),
            name: "Zero to One".to_owned(),
            year: 2014,
        };
        Ok(Response::new(response))
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = "[::1]:50051".parse().unwrap();
    let bookstore = BookStoreImpl::default();

    // Add this
    let reflection_service = tonic_reflection::server::Builder::configure()
        .register_encoded_file_descriptor_set(bookstore::FILE_DESCRIPTOR_SET)
        .build()
        .unwrap();

    println!("Bookstore server listening on {}", addr);

    Server::builder()
        .add_service(BookstoreServer::new(bookstore))
        .add_service(reflection_service) // Add this
        .serve(addr)
        .await?;

    Ok(())
}
```

### 测试 gRPC 服务功能

有很多的 `GUI` 客户端工具可以用来和 `gRPC` 服务器进行交互，例如：[Postman](https://www.postman.com/)、[Kreya](https://kreya.app/)、[bloomrpc](https://github.com/bloomrpc/bloomrpc)、[grpcox](https://github.com/gusaul/grpcox) 等。为了简单起见，我们将使用命令行工具 `grpc_cli`。

安装：

```shell
$ brew install grpc
```

并测试我们的第一个 `gRPC` 功能：

```shell
$ grpc_cli call localhost:50051 bookstore.Bookstore.GetBook "id: 'test-book-id'"
connecting to localhost:50051
Received initial metadata from server:
date : Sun, 08 May 2022 20:15:39 GMT
id: "test-book-id"
name: "Zero to One"
author: "Peter"
year: 2014
Rpc succeeded with OK status
```

它看起来运行正常！亲爱的朋友们，这就是我们在 `Rust` 中构建 `gRPC` 服务器的方式。

今天就到此为止。感谢您的阅读，祝您编程愉快！与往常一样，源代码可在 [GitHub](https://github.com/yzhong52/rust_grpc_demo) 上获得。
