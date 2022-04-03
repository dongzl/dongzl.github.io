---
title: Reactor 模式--Scalable IO in Java
date: 2021-01-09 19:23:58
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl@master/blog/cover/netty_study.png
# author information, multiple authors are set to array
# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文是通过对 Doug Lea 大师在《Scalable IO in Java》文档中内容学习来理解对 Reactor 模式做一个入门学习。

categories: 
  - 设计模式

tags: 
  - Reactor
---

## 背景介绍介绍介绍

周末随便学习一些东西，在[屈定](https://mrdear.cn/)老兄的博客上看到更新了一篇文章[《Netty -- Reactor模型的应用》](https://mrdear.cn/posts/framework-netty-reactor-model.html)，内容分析的很到位，对于 `Reactor` 模式，我了解到的主要还是在 `Netty` 框架中的线程模式使用的是 `Reactor` 模式，有时会想这个东西在在我们的业务系统中会有什么样的应用场景，是不是有机会在某些功能中落地到我们的项目中，而不是一直高高在上，为此还和屈定老兄交流了一下。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl@master/blog/2021/02-Reactor-Pattern-Scalable-IO-In-Java/Reactor-Pattern-Scalable-IO-In-Java-01.jpeg" style="width:300px"/>

看来这个东西也只是“此曲只应天上有，人间能得几回闻”，业务系统落地机会看来不多，也可能还需要慢慢探索，不过还是想在学习一下，不要只是肤浅的理解。

如果在谷歌上搜索 `Reactor` 模式的知识，搜索结果里面一定会有 `Doug Lea` 大师《Scalable IO in Java》文档内容，看这个文档内容应该是一次知识分享的 `PPT` 内容，而且很多分析 `Reactor` 模式的博客文章大部分在引用里面都会提到《Scalable IO in Java》内容，这次的学习也集中在这个文档的内容。

## 目录

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl@master/blog/2021/02-Reactor-Pattern-Scalable-IO-In-Java/Reactor-Pattern-Scalable-IO-In-Java-02.png"/>

## 可扩展的网络服务

### 网络服务

在这一节中，`Doug Lea` 大师总结了在 `web` 应用服务、分布式服务中一般包括如下一些基本的处理流程：

- `Read request`（读请求，比如说 `web` 请求中的 `HttpRequest`）

- `Decode request`（解析 `Request` 数据）

- `Process service` （业务逻辑处理）

- `Encode reply` （包装响应数据）

- `Send reply`（发送响应结果，比如说 `web` 请求中的 `HttpReponse`）

这个处理流程中不同的是每次需要进行的 `XML` 解析，文件传输、`web` 网页生成，服务计算等内容... ...

### 经典服务设计

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl@master/blog/2021/02-Reactor-Pattern-Scalable-IO-In-Java/Reactor-Pattern-Scalable-IO-In-Java-03.png"/>

每一个 `handler` 处理流程可能会启动一个自己独享的线程。

```java
class Server implements Runnable {

    @Override
    public void run() {
        try {
            ServerSocket ss = new ServerSocket(PORT);
            while (!Thead.interrupted()) {
                new Thread(new Handler(ss.accept())).start();
                // or, single-threaded, or a thread pool
            }
        } catch (IOException ex) {
            /* ... */
        }
    }

    static class Handler implements Runnable {
        final Socket socket;
        Handler(Socket s) {
            socket = s;
        }

        @Override
        public void run() {
            try {
                byte[] input = new byte[MAX_INPUT];
                socket.getInputStream().read(input);
                byte[] output = process(input);
                socket.getOutputStream().write(output);
            } catch (IOException ex) {
                /* ... */
            }
        }

        private byte[] process(byte[] cmd) {
            /* ... */
        }
    }
}

//注意：代码示例中的异常处理内容都被忽略掉了
```

### 可扩展性目标

- 负载增加时的平稳降级（支持更多客户端连接）

- 通过增加资源（`CPU`，内存，磁盘，带宽）进行持续改进

- 同时满足可用性和性能目标

  - 低延迟

  - 满足高峰需求

  - 可调节的服务质量

- 分而治之通常是实现任何可扩展性目标的最佳方法

### 分而治之的思想

将整个处理流程分割成为一些小的任务，每个任务执行一个动作而不会产生阻塞。

当某个时刻任务被启用时，开始执行这个任务；在整个过程中，`IO` 事件作为任务启用的触发器。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl@master/blog/2021/02-Reactor-Pattern-Scalable-IO-In-Java/Reactor-Pattern-Scalable-IO-In-Java-06.png"/>

`java.nio` 实现的基本机制：

- 非阻塞的读操作和写操作

- 监测 `IO` 相关事件的调度任务程序

无限变化的可能

- 事件驱动设计系列

### 事件驱动设计

事件驱动的设计通常比替代方案更加有效：

- 更少的资源占用：不需要为每一个客户端请求创建一个线程

- 减少开销：减少上下文切换，减少锁的争用

- 调度可能会变慢：必须手动将动作与事件绑定到一起

事件驱动的设计通常比替代方案程序实现更加复杂：

- 必须分解成简单的非阻塞动作
  
  - 类似于 `GUI` 事件驱动的操作

  - 无法消除所有阻塞：`GC`，页面错误等

- 必须跟踪逻辑服务状态

### 背景介绍：AWT 中的事件

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl@master/blog/2021/02-Reactor-Pattern-Scalable-IO-In-Java/Reactor-Pattern-Scalable-IO-In-Java-07.png"/>

事件驱动的 `IO` 使用相似的想法，但设计不同

## Reactor 模式

`Reactor` 通过调度适当的处理程序来响应 `IO` 事件（和 `AWT` 中的线程作用非常类似）

`Handler` 用于完成非阻塞的动作（和 `AWT` 中 `ActionListeners` 作用类似）

通过将处理程序绑定到事件进行管理（和 `AWT` 中 `addActionListener` 作用类似）

> 参见：Schmidt et al, Pattern-Oriented Software Architecture, Volume 2 (POSA2)
> 或者 Richard Stevens 的网络编程书籍, Matt Welsh 的 SEDA 框架书籍等内容。

### Reactor 模式基本实现

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl@master/blog/2021/02-Reactor-Pattern-Scalable-IO-In-Java/Reactor-Pattern-Scalable-IO-In-Java-04.png"/>

单线程版本实现

### java.nio 支持

- `Channels`：`Channel` 用于实现非阻塞读操作，可以连接到文件、`Socket` 等；

- `Buffers`：`Buffer` 类似于对象数组，能够直接通过 `Channel` 进行读写操作；

- `Selectors`：判断一组 `Channel` 中的哪些发生了 `IO` 事件；

- `SelectionKeys`：维护 `IO` 事件状态和绑定状态

### Reactor 模式第一步：启动

```java
class Reactor implements Runnable {

    final Selector selector;

    final ServerSocketChannel serverSocket;

    public Reactor(int port) throws IOException {
        selector = Selector.open();
        serverSocket = ServerSocketChannel.open();
        serverSocket.socket().bind(new InetSocketAddress(port));
        serverSocket.configureBlocking(false);
        SelectionKey sk = serverSocket.register(selector, SelectionKey.OP_ACCEPT);
        sk.attach(new Acceptor());
    }

    /*
      Alternatively, use explicit SPI provider:
      SelectorProvider p = SelectorProvider.provider();
      selector = p.openSelector();
      serverSocket = p.openServerSocketChannel();
    */
}
```

### Reactor 模式第二步：循环调度

```java
@Override
public void run() { // normally in a new Thread
    try {
        while (!Thread.interrupted()) {
            selector.select();
            Set selected = selector.selectedKeys();
            Iterator it = selected.iterator();
            while (it.hasNext()) {
                dispatch((SelectionKey)(it.next()));
                selected.clear();
            }
        }
    } catch (IOException ex) {
        /* ... */
    }
}

void dispatch(SelectionKey k) {
    Runnable r = (Runnable) (k.attachment());
    if (r != null) {
        r.run();
    }
}
```

### Reactor 模式第三步：Acceptor

```java
class Acceptor implements Runnable { // inner

    @Override
    public void run() {
        try {
            SocketChannel c = serverSocket.accept();
            if (c != null) {
                new Handler(selector, c);
            }
        } catch (IOException ex) {
            /* ... */
        }
    }
}
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl@master/blog/2021/02-Reactor-Pattern-Scalable-IO-In-Java/Reactor-Pattern-Scalable-IO-In-Java-04.png"/>

### Reactor 模式第四步：Handler 启动

```java
final class Handler implements Runnable {
    final SocketChannel socket;
    final SelectionKey sk;
    ByteBuffer input = ByteBuffer.allocate(MAXIN);
    ByteBuffer ouput = ByteBuffer.allocate(MAXOUT);
    static final int READING = 0, SENDING = 1;
    int state = READING;

    public Handler(Selector sel, SocketChannel c) throws IOException {
        socket = c;
        c.configureBlocking(false);
        // Optionally try first read now
        sk = socket.register(sel, 0);
        sk.attach(this);
        sk.interestOps(SelectionKey.OP_READ);
        sel.wakeup();
    }

    boolean inputIsComplete() { /* ... */ }
    boolean outputIsComplete() { /* ... */ }
    void process() { /* ... */ }
```

### Reactor 模式第五步：请求处理

```java
@Override
public void run() {
    try {
        if (state = READING) {
            read();
        } else if (state = SENDING) {
            send();
        }
    } catch (IOException ex) {
        /* ... */
    }
}

void read() throws IOException {
    socket.read(input);
    if (inputIsComplete()) {
        process();
        state = SENDING;
        // Normally also do first write now
        sk.interestOps(SelectionKey.OP_WRITE);
    }
}

void send() throws IOException {
    socket.write(ouput);
    if (inputIsComplete()) {
        sk.cancel();
    }
}
```

### 单个状态处理程序

`GoF` 中状态模式的简单应用：重新绑定适当的处理程序作为附件

```java
class Handler { // ...

    public void run() { // initial state is reader
        socket.read(input);
        if (inputIsComplete()) {
            process();
            sk.attach(new Sender());
            sk.interest(SelectionKey.OP_WRITE);
            sk.selector().wakeup();
        }
    }
    
    class Sender implements Runnable {

        @Override
        public void run(){ // ...
            socket.write(output);
            if (outputIsComplete())
                sk.cancel();
        }
    }
}
```

### 多线程设计实现

- 从策略上添加线程以实现可扩展性：主要适用于多核处理器

- 工作线程：
  
  - `Reactor` 能够快速的触发 Handler 执行：处理程序处理会减慢 `Reactor` 的速度

  - 将非 `IO` 处理程序由其他线程来完成

- 多个 `Reactor` 线程
  
  - `Reactor` 堆线程可以充分利用系统 `IO` 操作

  - 将负载分配给其他 `Reactor` 线程：负载均衡以匹配 `CPU` 和 `IO` 利用率

### 工作线程

- 卸载非 `IO` 处理以加速 `Reactor` 线程：与 POSA2 Proactor 设计类似

PS. 工作线程用于处理 IO 事件，`Reactor` 线程不用关心 `IO` 事件，这样可以提升 `Reactor` 线程 处理速度

- 比将计算绑定处理重新加工成事件驱动的形式更简单：应该仍然是纯非阻塞计算，足够的处理胜过开销

- 但是很难将处理与 `IO` 重叠：最好某个时刻可以先将所有输入读入缓冲区

- 使用线程池，因此可以进行调整和控制：通常需要比客户端少的线程

### 工作线程池

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl@master/blog/2021/02-Reactor-Pattern-Scalable-IO-In-Java/Reactor-Pattern-Scalable-IO-In-Java-05.png"/>

### 线程池处理器

```java
class Handler { // ...

    // uses util.concurrent thread pool
    static PooledExecutor pool = new PooledExecutor(...);
    static final int PROCESSING = 3;

    // ...
    synchronized void read() { // ...
        socket.read(input);
        if (inputIsComplete()) {
            state = PROCESSING;
            pool.execute(new Processer());
        }
    }

    synchronized void processAndHandOff() {
        process();
        state = SENDING; // or rebind attachment
        sk.interest(SelectionKey.OP_WRITE);
    }

    class Processer implements Runnable {
        public void run() {
            processAndHandOff();
        }
    }
}
```

### 协调任务

- `Handoffs`
  
  - 每个任务都会启用，触发或调用下一个任务
  
  - 通常最快，但可能很脆弱

- `Callbacks`：回调每个处理程序的调度程序

  - 设置状态，附件等

  - `GoF` `Mediator` 模式的变体

- `Queues`

  - 例如，跨阶段传递缓冲区

- `Futures`

  - 当每个任务产生结果时
  
  - 协作位于联接或等待 / 通知之上


### 使用线程池执行器

- 可调工作线程池

- 最重要方法 `execute(Runnable r)`

- 控制：

  - 任务队列的种类（任何通道）

  - 最大线程数

  - 最小线程数

  - “温暖”与按需线程

  - 保持活动间隔，直到空闲线程死亡：如有必要，稍后将其替换为新的

  - 饱和策略：阻止，掉落，生产者运行等

### 多个 Reactor 线程模式

使用 `Reactor` 对象池

- 用于匹配 `CPU` 和 `IO` 速率

- 静态或动态构造：每个 `Reactor` 线程都有自己的选择器，线程，分派循环

主 `acceptor` 负责分配到其他的 `Reactor`

```java
Selector[] selectors; // also create threads
int next = 0;
class Acceptor { // ...
    public synchronized void run() { ...
        Socket connection = serverSocket.accept();
        if (connection != null) {
            new Handler(selectors[next], connection);
        }
        if (++next == selectors.length) {
            next = 0;
        }
    }
}
```

### 使用多 Reactor 模式

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl@master/blog/2021/02-Reactor-Pattern-Scalable-IO-In-Java/Reactor-Pattern-Scalable-IO-In-Java-08.png"/>

### 使用其他的 java.nio 特性

- 每个 `Reactor` 支持多个 `Selector`

  - 将不同的处理程序绑定到不同的 `IO` 事件

  - 需要仔细处理同步操作来进行协调

- 文件传输

  - 自动完成文件到网络或网络到文件的复制操作

- 内存映射文件

  - 通过 `buffer` 访问文件

- 直接缓冲

  - 有时可以实现零拷贝传输

  - 但是启动和完成会产生额外的开销

  - 最适合连接时间较长的应用

### 基于连接的扩展

- 非单个服务请求处理

  - 客户端连接

  - 客户端发送一系列的消息 / 请求

  - 断开客户端连接

- 示例场景

  - 数据库和事务监听

  - 多人游戏，聊天等

- 可扩展的基础网络服务模式
  
  - 处理许多存活时间相对较长的客户端请求

  - 跟踪客户端和会话状态（包括丢弃）

  - 跨多个主机分配服务

## API 练习

- `Buffer`

- `ByteBuffer`：`CharBuffer`，`LongBuffer` 等其他一些未列举。

- `Channel`

- `SelectableChannel`

- `SocketChannel`

- `ServerSocketChannel`

- `FileChannel`

- `Selector`

- `SelectionKey`

### Buffer

```java
abstract class Buffer {
    
    int capacity();
    int position();
    Buffer position(int newPosition);
    int limit();
    Buffer limit(int newLimit);
    Buffer mark();
    Buffer reset();
    Buffer clear();
    Buffer flip();
    Buffer rewind();
    int remaining();
    boolean hasRemaining();
    boolean isReadOnly();
}
```

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl@master/blog/2021/02-Reactor-Pattern-Scalable-IO-In-Java/Reactor-Pattern-Scalable-IO-In-Java-09.png"/>

### ByteBuffer

```java
abstract class ByteBuffer extends Buffer {

    static ByteBuffer allocateDirect(int capacity);
    static ByteBuffer allocate(int capacity);
    static ByteBuffer wrap(byte[] src, int offset, int len);
    static ByteBuffer wrap(byte[] src);
    boolean isDirect();
    ByteOrder order();
    ByteBuffer order(ByteOrder bo);
    ByteBuffer slice();
    ByteBuffer duplicate();
    ByteBuffer compact();
    ByteBuffer asReadOnlyBuffer();
    byte get();
    byte get(int index);
    ByteBuffer get(byte[] dst, int offset, int length);
    ByteBuffer get(byte[] dst);
    ByteBuffer put(byte b);
    ByteBuffer put(int index, byte b);
    ByteBuffer put(byte[] src, int offset, int length);
    ByteBuffer put(ByteBuffer src);
    ByteBuffer put(byte[] src);
    char getChar();
    char getChar(int index);
    ByteBuffer putChar(char value);
    ByteBuffer putChar(int index, char value);
    CharBuffer asCharBuffer();
    short getShort();
    short getShort(int index);
    ByteBuffer putShort(short value);
    ByteBuffer putShort(int index, short value);
    ShortBuffer asShortBuffer();
    int getInt();
    int getInt(int index);
    ByteBuffer putInt(int value);
    ByteBuffer putInt(int index, int value);
    IntBuffer asIntBuffer();
    long getLong();
    long getLong(int index);
    ByteBuffer putLong(long value);
    ByteBuffer putLong(int index, long value);
    LongBuffer asLongBuffer();
    float getFloat();
    float getFloat(int index);
    ByteBuffer putFloat(float value);
    ByteBuffer putFloat(int index, float value);
    FloatBuffer asFloatBuffer();
    double getDouble();
    double getDouble(int index);
    ByteBuffer putDouble(double value);
    ByteBuffer putDouble(int index, double value);
    DoubleBuffer asDoubleBuffer();
}
```

### Channel

```java
interface Channel {
    boolean isOpen();
    void close() throws IOException;
}

interface ReadableByteChannel extends Channel {
    int read(ByteBuffer dst) throws IOException;
}

interface WritableByteChannel extends Channel {
    int write(ByteBuffer src) throws IOException;
}

interface ScatteringByteChannel extends ReadableByteChannel {
    int read(ByteBuffer[] dsts, int offset, int length) throws IOException;
    int read(ByteBuffer[] dsts) throws IOException;
}

interface GatheringByteChannel extends WritableByteChannel {
    int write(ByteBuffer[] srcs, int offset, int length) throws IOException;
    int write(ByteBuffer[] srcs) throws IOException;
}
```

### SelectableChannel

```java
abstract class SelectableChannel implements Channel {
    int validOps();
    boolean isRegistered();
    SelectionKey keyFor(Selector sel);
    SelectionKey register(Selector sel, int ops) throws ClosedChannelException;
    void configureBlocking(boolean block) throws IOException;
    boolean isBlocking();
    Object blockingLock();
}
```

### SocketChannel

```java
abstract class SocketChannel implements ByteChannel ... {
    static SocketChannel open() throws IOException;
    Socket socket();
    int validOps();
    boolean isConnected();
    boolean isConnectionPending();
    boolean isInputOpen();
    boolean isOutputOpen();
    boolean connect(SocketAddress remote) throws IOException;
    boolean finishConnect() throws IOException;
    void shutdownInput() throws IOException;
    void shutdownOutput() throws IOException;
    int read(ByteBuffer dst) throws IOException;
    int read(ByteBuffer[] dsts, int offset, int length) throws IOException;
    int read(ByteBuffer[] dsts) throws IOException;
    int write(ByteBuffer src) throws IOException;
    int write(ByteBuffer[] srcs, int offset, int length) throws IOException;int write(ByteBuffer[] srcs) throws IOException;
}
```

### ServerSocketChannel

```java
abstract class ServerSocketChannel extends ... {
    static ServerSocketChannel open() throws IOException;
    int validOps();
    ServerSocket socket();
    SocketChannel accept() throws IOException;
}
```

### FileChannel

```java
abstract class FileChannel implements ... {
    int read(ByteBuffer dst);
    int read(ByteBuffer dst, long position);
    int read(ByteBuffer[] dsts, int offset, int length);
    int read(ByteBuffer[] dsts);
    int write(ByteBuffer src);
    int write(ByteBuffer src, long position);
    int write(ByteBuffer[] srcs, int offset, int length);
    int write(ByteBuffer[] srcs);
    long position();
    void position(long newPosition);
    long size();
    void truncate(long size);
    void force(boolean flushMetaDataToo);
    int transferTo(long position, int count, WritableByteChannel dst);
    int transferFrom(ReadableByteChannel src, long position, int count);
    FileLock lock(long position, long size, boolean shared);
    FileLock lock();
    FileLock tryLock(long pos, long size, boolean shared);
    FileLock tryLock();
    static final int MAP_RO, MAP_RW, MAP_COW;
    MappedByteBuffer map(int mode, long position, int size);
} 
NOTE: ALL methods throw IOException
```

### Selector

```java
abstract class Selector {
    static Selector open() throws IOException;
    Set keys();
    Set selectedKeys();
    int selectNow() throws IOException;
    int select(long timeout) throws IOException;
    int select() throws IOException;
    void wakeup();
    void close() throws IOException;
}
```

### SelectionKey

```java
abstract class SelectionKey {
    static final int OP_READ, OP_WRITE, OP_CONNECT, OP_ACCEPT;
    SelectableChannel channel();
    Selector selector();
    boolean isValid();
    void cancel();
    int interestOps();
    void interestOps(int ops);
    int readyOps();
    boolean isReadable();
    boolean isWritable();
    boolean isConnectable();
    boolean isAcceptable();
    Object attach(Object ob);
    Object attachment();
}
```

## 参考链接

- [Scalable IO in Java](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)