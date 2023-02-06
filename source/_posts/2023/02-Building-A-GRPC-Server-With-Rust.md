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
subtitle: 本文是一篇翻译文章，在这篇博客文章中，我们将学习如何在不使用连接器或外部库的情况下从头开始编写我们自己的原生 MySQL 客户端。

categories: 
  - 架构设计

tags: 
  - Rust
  - gRPC
---

> 原文链接：https://betterprogramming.pub/building-a-grpc-server-with-rust-be2c52f0860e

<div class="mxgraph" style="max-width:100%;border:1px solid transparent;" data-mxgraph="{&quot;highlight&quot;:&quot;#0000ff&quot;,&quot;nav&quot;:true,&quot;resize&quot;:true,&quot;toolbar&quot;:&quot;zoom layers tags lightbox&quot;,&quot;edit&quot;:&quot;_blank&quot;,&quot;xml&quot;:&quot;&lt;mxfile host=\&quot;Electron\&quot; modified=\&quot;2023-02-06T07:20:28.811Z\&quot; agent=\&quot;5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) draw.io/15.7.3 Chrome/91.0.4472.164 Electron/13.6.1 Safari/537.36\&quot; etag=\&quot;yswMCyzKWQq-mewerCfZ\&quot; version=\&quot;15.7.3\&quot; type=\&quot;device\&quot;&gt;&lt;diagram id=\&quot;C5RBs43oDa-KdzZeNtuy\&quot; name=\&quot;Page-1\&quot;&gt;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&lt;/diagram&gt;&lt;/mxfile&gt;&quot;}"></div>
<script type="text/javascript" src="https://viewer.diagrams.net/js/viewer-static.min.js"></script>

<iframe height=414 width=640 src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/02-Building-A-GRPC-Server-With-Rust/three-body.gif" />
