---
title: 异步 API 设计最佳实践：用于实时通信的服务端事件发送 (SSE)
date: 2023-12-19 18:28:22
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/async_vs_sync.png

# author information, multiple authors are set to array
# single author
author:
- nick: stackademic
  link: https://blog.stackademic.com/
- nick: 董宗磊
  link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 学习系统设计中正向代理和反向代理之间的差异以及何时使用它们。

categories:
- 架构设计

tags:
- SSE
---

> 原文链接：https://blog.stackademic.com/asynchronous-api-design-best-practices-server-sent-event-sse-for-real-time-communication-a3a3e20233d2

在当下应用程序开发领域，实时通信技术不再是一种奢侈品，而是必需品。异步 `API` 设计是实现这一目标的关键，异步 `API` 可以使应用程序能够提供实时的更新和通知能力，并且不受传统请求响应模型的限制。

在本文中，我们将探讨异步 `API` 设计的四种强大技术：**Callbacks**、**WebSocket**、**消息队列**和**服务端事件发送（SSE）**。所有的这些方法都具有独特的优势，对于创建响应式实时通知应用程序非常重要。

## 为什么异步 API 设计很重要

传统的 `API` 设计中的请求-响应模式是有其局限性的。当客户端向服务器发送请求时，通常必须等待响应结果，这可能会导致响应延迟并降低用户体验，尤其在需要实时数据更新等至关重要的场景中。

异步 `API` 设计允许服务器异步处理耗时的任务并立即返回响应结果，从而摆脱了这些限制。这使得客户端无需等待结果即可继续其他操作，并在任务完成后立即接收更新信息。

## 异步 API 工作流程

在异步 `API` 设计中

1. 客户端通过 `Rest Endpoint` 向服务器发送请求；
2. 服务器异步处理耗时任务并立即向客户端返回该任务的响应结果--“我正在处理这个任务”；服务器可以在立即返回的响应结果中发送一个唯一的 `ID`，以便客户端可以使用该 `ID` 定期获取任务状态。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/26-Asynchronous-API-Design-Best-Practices-Server-Sent-Event/01.png"/>

3. **任务完成后**，服务器可以使用多种机制通过响应消息通知客户端；机制的选择通常取决于应用程序的使用场景和所使用的通信协议。

## 如何将数据推送到 API 客户端

理想的情况是当有可用的新数据或新事件时，服务器会通知 `API` 客户端；但是我们无法使用传统的 `HTTP` 请求-响应模型的交互方式来达到这种效果。我们必须找到一种方法让服务器能够将数据推送到客户端--接入异步 `API`。

## 如何实现服务端的实时通知

### 方法一：轮询

客户端反复向服务器发送请求，请求更新的数据；服务器在有新的信息或结果时给出响应。虽然这个方案实施起来很简单，但轮询可能会导致网络流量增加和网络延迟。

### 方法 2：WebSocket

`WebSockets` 提供全双工通信能力，**可以实现一旦服务端有数据更新，可以立即向客户端推送消息**。`WebSocket` 非常适合需要低延迟、实时通信的应用程序。

### 方法 3：服务器事件发送 (SSE)

**服务器需要以单向的方式向客户端推送更新**。它用单个 `HTTP` 连接，与打开多个连接相比减少了开销。

`SSE` 是单向的，这意味着**客户端只能从服务器接收更新数据，不能向服务端发送数据**。`SSE` 适合仅需要单向通信的应用场景。

### 方法 4：消息队列

服务器可以使用消息队列（例如：`RabbitMQ`、`Apache Kafka`）来发布消息；客户端订阅特定主题或队列，并在消息到达时异步接收消息。

### 方法 5：回调 URL

回调 `URL` 对于服务器长时间运行后通知客户端的场景非常有效，它们最大限度地减少了客户端轮询操作或维护持久连接的需要。

**缺点**：客户端必须开放可公开访问的回调 `URL`，这可能会带来安全隐患和隐私问题。此外管理回调 `URL` 和处理回调失败时的重试可能具有很大挑战性。

## 使用服务器事件发送的异步 API

服务器事件发送（`SSE`）提供一种强大的机制，用于在服务器和客户端之间实现异步通信，特别是在 `API` 上下文环境中。`SSE` 是基于浏览器的 `EventSource` 接口实现的，这个接口是由万维网联盟 (`W3C`) 制定的 `HTML5` 标准的一部分；它引入了一种使用 `HTTP` 建立长连接的方法，允许服务器主动将数据推送到客户端，该数据通常与关联的有效负载信息共同构造成一个事件。

最初，`SSE` 的构想是为了方便向 `Web` 应用程序的数据传输，但它在 `API` 领域中的相关性越来越大。`SSE` 提供了引人注目的解决方案来替代传统轮询机制，解决了一些客户端-服务器通信相关的固有挑战问题。

## SSE 是如何工作的

`SSE` 使用标准 `HTTP` 连接，但是会保持较长时间连接而不是立即断开。该连接允许服务器在数据可用时将数据推送到客户端：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/26-Asynchronous-API-Design-Best-Practices-Server-Sent-Event/02.png"/>

规范描述了所返回数据格式的一些内容，包括事件名称、注释、基于文本格式单行或多行的数据以及事件标识符。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2023/26-Asynchronous-API-Design-Best-Practices-Server-Sent-Event/03.png"/>

## 使用案例：电子商务系统中批量产品信息更新 API

在此用例中，电子商务网站允许客户上传包含大量产品列表的 `CSV` 文件。

> 服务器异步处理上传的 CSV 文件并立即发送响应结果以确认上传成功。CSV 文件的解析和验证完成后，服务器使用服务器事件发送（SSE）将处理后的产品数据发送到客户端。

## 客户端到服务器（CSV 上传和异步处理）：

### 1. 客户端初始化 CSV 上传：

客户端与电子商务网站交互并通过用户界面发起 `CSV` 文件上传。

### 2. 客户端发送 CSV 文件：

客户端选择包含产品数据的 `CSV` 文件，并通过向 `/api/upload/csv` 发送 `POST` 请求将其上传到服务器。

### 3. 服务器验证文件并生成唯一 ID：

服务器接收 `CSV` 文件并对其进行验证。如果文件有效，服务器会立即响应 `HTTP 202`（`Accepted`）状态代码来确认上传成功。

- 服务器会为本次上传生成一个**唯一的事务 ID**，用于跟踪进度处理过程。

### 4. 异步处理开始：

服务器开始异步处理 `CSV` 文件。此处理包括 `CSV` 文件解析、数据验证和产品列表的创建。

- 处理可能在后台进行，允许服务器在处理 `CSV` 文件同时能够处理其他请求。

### 5. 通过 SSE 发送数据更新进度：

处理 `CSV` 文件并生成产品数据时，服务器使用服务器事件发送（`SSE`）向客户端发送实时进度更新。`SSE Endpoint`（`/sse`）已经建立，并通过生成的唯一 `ID` 与客户端进行连接。

### 6. 服务器到客户端（更新进度和操作完成）：

1. 客户端侦听 `SSE Endpoint`-实时更新进度：在客户端，`JavaScript` 脚本用来监听与唯一 `ID` 关联的 `SSE Endpoint`（`/sse`）。客户端通过 `SSE` 与服务器建立长连接，实现实时更新。
2. **服务器发送进度更新**： 
  - 在处理 `CSV` 文件时，**服务器会将部分产品数据更新和进度消息发送到 `SSE Endpoint`**，客户端会监听该 `Endpoint` 返回的数据。
  - 服务器会将这些更新数据实时推送到客户端，为用户提供关于 `CSV` 处理进度的反馈。

```java
// Inside your CSV processing logic
String transactionId = "TXN-123"; // Replace with the actual transaction ID
String progressMessage = "Processing 50% complete"; // Replace with your progress message

// Send an SSE update to the client
sseController.sendSseUpdate(transactionId, progressMessage);
```

3. **通过 `SSE` 发送完成消息**：

- 在成功处理整个 `CSV` 文件后，服务器通过 `SSE` 向客户端发送最终完成消息，并通知用户产品数据已可以检索。
- 用户端收到此消息后可以继续从服务器检索已处理的产品数据。

4. **错误处理**：

- 如果上传的 `CSV` 文件存在问题（例如：格式错误或数据无效）或者处理过程中发生错误，服务器会向客户端发送错误响应结果。
- 客户会收到报错通知并可以纠正问题。

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.io.IOException;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@RestController
@RequestMapping("/api/upload")
public class CsvUploadController {

    private final Map<String, SseEmitter> sseEmitters = new ConcurrentHashMap<>();

    @PostMapping("/csv")
    public ResponseEntity<String> uploadCsv(@RequestParam("file") MultipartFile file) {
        if (file.isEmpty()) {
            return ResponseEntity.badRequest().body("Please select a CSV file to upload.");
        }

        if (!isCsvValid(file)) {
            return ResponseEntity.badRequest().body("Invalid CSV file format or data.");
        }

        String transactionId = "TXN-" + System.currentTimeMillis();

        //1. Start asynchronous processing and return a CompletableFuture
        CompletableFuture<Void> processingFuture = CompletableFuture.runAsync(() -> {
            asyncProcessCsv(file, transactionId);
        });

        //2. Return a 202 (Accepted) response with the transaction ID
        return ResponseEntity.status(HttpStatus.ACCEPTED).body(transactionId);
    }



    private boolean isCsvValid(MultipartFile file) {
        // Add your CSV validation logic here
        // Return true if the CSV is valid; otherwise, return false
        return true;
    }


    private void asyncProcessCsv(MultipartFile file, String transactionId) {
        CompletableFuture<Void> processingFuture = CompletableFuture.runAsync(() -> {
            // Your CSV processing logic here
            try (CSVReader csvReader = new CSVReader(
                    new InputStreamReader(file.getInputStream()))) {

                // Process CSV rows here
                // ...

                // Send progress updates via SSE
                for (int i = 1; i <= totalRows; i++) {
                    String progressMessage = "Processing row " + i;
                    sendProgressUpdate(transactionId, progressMessage);
                }

                // Send completion message via SSE
                sendCompletionMessage(transactionId, "CSV processing completed.");
            } catch (Exception e) {
                // Handle exceptions during processing
                sendErrorMessage(transactionId, "Error during processing: " + e.getMessage());
            } finally {
                sseEmitters.remove(transactionId);
            }
        });

        // Handle any exceptions that occur during processing
        processingFuture.exceptionally(ex -> {
            sendErrorMessage(transactionId, "Error during processing: " + ex.getMessage());
            return null;
        });
    }


    @GetMapping("/sse/{transactionId}")
    public SseEmitter getSseEmitter(@PathVariable String transactionId) {
        SseEmitter sseEmitter = new SseEmitter();
        sseEmitters.put(transactionId, sseEmitter);
        return sseEmitter;
    }


    private void sendProgressUpdate(String transactionId, String message) {
        SseEmitter sseEmitter = sseEmitters.get(transactionId);
        if (sseEmitter != null) {
            try {
                sseEmitter.send(SseEmitter.event().name("progress").data(message));
            } catch (IOException e) {
                // Handle exceptions when sending SSE updates
                e.printStackTrace();
            }
        }
    }


    private void sendCompletionMessage(String transactionId, String message) {
        SseEmitter sseEmitter = sseEmitters.get(transactionId);
        if (sseEmitter != null) {
            try {
                sseEmitter.send(SseEmitter.event().name("complete").data(message));
                sseEmitter.complete(); // Close the SSE connection
            } catch (IOException e) {
                // Handle exceptions when sending SSE updates
                e.printStackTrace();
            }
        }
    }


    private void sendErrorMessage(String transactionId, String message) {
        SseEmitter sseEmitter = sseEmitters.get(transactionId);
        if (sseEmitter != null) {
            try {
                sseEmitter.send(SseEmitter.event().name("error").data(message));
                sseEmitter.completeWithError(new RuntimeException(message)); // Complete with an error
            } catch (IOException e) {
                // Handle exceptions when sending SSE updates
                e.printStackTrace();
            }
        }
    }
}
```

**关键点**：
- 接收到请求后，服务器立即返回响应结果，并携带 `TXN ID` 信息；
- 客户端接收到 `TXN ID` 后，会注册到 `SSE` 来获取实时进度更新、错误处理消息和接收任务完成通知；
- 服务器异步处理任务，在处理完成后会通过 `SSE` 向客户端发送通知。

**客户端实现**：

在客户端（通常是网页端），我们需要使用 `JavaScript` 侦听 `SSE Endpoint`（`/sse/stream`）并处理返回的更新数据。下面这段代码是如何在 `JavaScript` 中执行操作的简化示例：

```html
<!DOCTYPE html>
<html>
<head>
    <title>Asynchronous Order Processing</title>
</head>
<body>
    <h1>Asynchronous Order Processing</h1>
    <button onclick="processOrder()">Process Order</button>
    <div id="result"></div>

    <script>
        let eventSource = null;

        async function processOrder() {
            const orderRequest = {
                csvFilePath: "Path"
            };

            try {
                const response = await fetch('/api/upload/csv', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json'
                    },
                    body: JSON.stringify(orderRequest)
                });

                if (response.status === 202) {
                    document.getElementById('result').textContent = 'Order processing initiated. Waiting for completion...';
                     const transactionId =response.result.transactionId;
                    // Connect to the SSE endpoint for this order
        const eventSource = new EventSource(`/sse/stream?transactionId=${transactionId}`);                    
                    eventSource.onmessage = (event) => {
                        document.getElementById('result').textContent = event.data;
                    };
                    
                    eventSource.onerror = (error) => {
                        console.error('SSE Error:', error);
                    };
                }
            } catch (error) {
                console.error('Error:', error);
            }
        }

        function closeEventSource() {
            if (eventSource) {
                eventSource.close();
                eventSource = null;
            }
        }

        // Close the SSE connection when leaving the page
        window.addEventListener('beforeunload', closeEventSource);
    </script>
</body>
</html>
```

## 结论

在当下应用程序开发领域，快速响应能力和实时通信是至关重要的。异步 `API` 设计及其一系列技术实现（例如：Callbacks、WebSocket、消息队列和服务端事件发送（`SSE`））使开发人员能够快速构建面向用户提供实时数据更新和通知的应用程序。在我们现实生活中的电子商务案例中，`SSE` 被证明是一个颠覆性技术实现，能够为客户提供实时进度更新和数据通知，同时能够优化性能和用户体验。

当我们再思考异步 `API` 领域设计时，我们需要考虑应用程序的需求场景，并选择最符合我们需要的目标方法。无论是让用户了解产品更新还是在协作平台中实时协作，掌握这些异步技术将使我们的应用程序在当今瞬息万变的数字世界中脱颖而出。
