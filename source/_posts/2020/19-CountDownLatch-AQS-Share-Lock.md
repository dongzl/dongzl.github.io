---
title: CountDownLatch 基于 AQS 共享锁模式实现原理分析
date: 2020-04-01 21:34:19
cover: https://gitee.com/dongzl/article-images/raw/master/cover/java_study.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文通过 JDK 源码分析如何使用 AQS 的共享模式实现 CountDownLatch 类功能。

categories: 
  - java开发

tags: 
  - AQS
  - CountDownLatch
---

## 背景描述

`CountDownLatch` 是 `JDK` 中提供的一个非常有用的工具类，在实际的工作中也有很多应用场景，比如，在一段业务逻辑中可能要进行几个操作，这些操作彼此之间没有关联，这些操作在完成之后继续进行后续操作，如果采用串行的操作方式，业务逻辑执行时间也是累加关系；如果采用 `CountDownLatch` 工具类，可以开启多个线程并行执行这些操作，执行成功后调用 `countDown()` 减一，主线程调用 `await()` 进入等待状态，当子线程全部执行完毕，count 值减到 0 之后，唤醒主线程继续执行，这个时候代码执行时间就不是累加关系了，而是执行最慢操作执行时间，这就是 `CountDownLatch` 工具类的妙处。

> A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.

我们来看一下 JDK 注释中给出的 `CountDownLatch` 的使用示例：

```java
class Driver2 { // ...
    void main() throws InterruptedException {
        CountDownLatch doneSignal = new CountDownLatch(N);
        Executor e = ...
        for (int i = 0; i < N; ++i) {// create and start threads
            e.execute(new WorkerRunnable(doneSignal, i));
        }
        doneSignal.await();           // wait for all to finish
    }
 }

class WorkerRunnable implements Runnable {
    private final CountDownLatch doneSignal;
    private final int i;
    WorkerRunnable(CountDownLatch doneSignal, int i) {
        this.doneSignal = doneSignal;
        this.i = i;
    }
    public void run() {
        try {
            doWork(i);
            doneSignal.countDown();
        } catch (InterruptedException ex) {} // return;
    }
    void doWork() { ... }
}
```

下面我们来分析一下 `CountDownLatch` 的实现原理，本文代码分析都是以 `JDK1.8` 源代码为基础进行分析。

## AQS 原理

`CountDownLatch` 的核心实现原理是基于 `AQS`，`AQS` 全称 `AbstractQueuedSynchronizer`，是 `java.util.concurrent` 中提供的一种高效且可扩展的同步机制；它是一种提供了原子式管理同步状态、阻塞和唤醒线程功能以及队列模型的简单框架。除了 `CountDownLatch` 工具类，JDK 当中的 `Semaphore`、`ReentrantLock` 等工具类都是基于 `AQS` 来实现的。下面我们用 `CountDownLatch` 来分析一下 `AQS` 的实现。

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/19-CountDownLatch-AQS-Share-Lock/CountDownLatch-AQS-Share-Lock-01.png">

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/19-CountDownLatch-AQS-Share-Lock/CountDownLatch-AQS-Share-Lock-02.png">

其实，如果我们阅读 `CountDownLatch` 的源码实现，发现其实它的代码实现非常简单，算上注释也才 300+ 行代码，如果去掉注释的话代码不到 100 行，大部分方法实现都是调用的 `Sync` 这个静态内部类的实现，而 `Sync` 就是继承自 `AbstractQueuedSynchronizer`。

`Sync` 重写了 `AQS` 中的 `tryAcquireShared` 和 `tryReleaseShared` 两个方法。当调用 `CountDownLatch` 的 `awit()` 方法时，会调用内部类 `Sync` 的 `acquireSharedInterruptibly()` 方法，然后在这个方法中会调用 `tryAcquireShared` 方法，这个方法就是 `Sync` 重写的 `AQS` 中的方法；调用 `countDown()` 方法原理基本类似。

通过内部类继承的方式是我们使用 `AbstractQueuedSynchronizer` 的标准方式：

- 内部持有继承自 `AbstractQueuedSynchronizer` 的对象 `Sync`；
- 在 `Sync` 内重写 `AbstractQueuedSynchronizer` 内部 `protected` 的部分或全部方法：

```java
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}

protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}

protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}

protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}

protected boolean isHeldExclusively() {
    throw new UnsupportedOperationException();
}
```

通过需要重写的方法名称我们大致可以得知，`AQS` 中是分成两种模式的：独占模式和共享模式，其中 `CountDownLatch` 使用的是共享模式。

- `tryAcquire` 和 `tryRelease` 是对应的，前者是独占模式获取，后者是独占模式释放；

- `tryAcquireShared` 和 `tryReleaseShared` 是对应的，前者是共享模式获取，后者是共享模式释放。

## 源码实现分析

### 构造方法实现

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```
`CountDownLatch` 的构造方法中调用了 `Sync` 的构造方法，`Sync` 的构造方法调用了 `AQS` 类中的 `setState(count);` 方法：

```java
Sync(int count) {
    setState(count);
}
```

`state` 变量是 `AQS` 类中的一个 volatile 变量。在 `CountDownLatch` 中这个 `state` 值就是一个计数器，记录 `countDown` 是否已经减到 0。

```java
/**
 * The synchronization state.
 */
private volatile int state;
```

### await() 方法实现

在调用 `await()` 方法时，会直接调用 `AQS` 类的 `acquireSharedInterruptibly` 方法，在 `acquireSharedInterruptibly` 方法内部会继续调用 `Sync` 实现类中的 `tryAcquireShared` 方法，在 `tryAcquireShared` 方法中判断 `state` 变量值是否为 `0`。

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```
如果 `state` 值不等于 0，说明还有需要等待的线程在运行，则会执行 `doAcquireSharedInterruptibly()` 方法，执行该方法的第一个动作就是尝试加入等待队列，即调用 `addWaiter()` 方法，源码如下：

```java
/**
 * Acquires in shared interruptible mode.
 * @param arg the acquire argument
 */
private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
    //加入等待队列
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        //进入 CAS 循环
        for (;;) {
            //当一个节点(关联一个线程)进入等待队列后， 获取此节点的 prev 节点 
            final Node p = node.predecessor();
            // 如果获取到的 prev 是 head，也就是队列中第一个等待线程
            if (p == head) {
                // 再次尝试申请 反应到 CountDownLatch 就是查看是否还有线程需要等待(state是否为0)
                int r = tryAcquireShared(arg);
                // 如果 r >=0 说明 没有线程需要等待了 state==0
                if (r >= 0) {
                    //尝试将第一个线程关联的节点设置为 head 
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            //经过自旋tryAcquireShared后，state还不为0，就会到这里，第一次的时候，
            //waitStatus是0，那么node的waitStatus就会被置为SIGNAL，第二次再走到这里，
            //就会用LockSupport的park方法把当前线程阻塞住
            if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

这里就是 `AQS` 的核心实现，`AQS` 用内部的一个 `Node` 类维护一个 `CHL Node FIFO` 队列。将当前线程加入等待队列，并通过 `parkAndCheckInterrupt()` 方法实现当前线程的阻塞。

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    // 尝试快速入队操作，因为大多数时候尾节点不为 null
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //如果尾节点为空(也就是队列为空) 或者尝试CAS入队失败(由于并发原因)，进入enq方法
    enq(node);
    return node;
}
```
`addWaiter()` 方法是向等待队列中添加等待者（waiter）。首先构造一个 `Node` 实体，参数为当前线程和一个 `Node` 对象（mode），这个 `mode` 有两种形式，一种是 `SHARED`，另一种是 `EXCLUSIVE`。接下来需要执行入队操作，`addWaiter()` 方法和 `enq()` 方法的 else 分支操作是一样的，这里的操作如果成功了，就不用再进到 `enq()` 方法的循环中去了，可以提高性能；如果没有成功，再调用 `enq()` 方法。

```java
/**
 * Inserts node into queue, initializing if necessary. See picture above.
 * @param node the node to insert
 * @return node's predecessor
 */
private Node enq(final Node node) {
    // 死循环 + CAS 保证所有节点都入队
    for (;;) {
        Node t = tail;
        // 如果队列为空 设置一个空节点作为 head
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            // 加入队尾
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

上面操作就是 `AQS` 等待队列入队方法，操作在无限循环中进行，如果入队成功则返回新的队尾节点（<font color="red">enq 方法中返回的是 t，感觉不是新队尾节点呢，像是队尾的前一个节点呢，不过影响不大，在 addWaiter 方法中返回的 node 是新的队尾节点</font>），否则一直自旋，直到入队成功。假设入队的节点为 node ，上来直接进入循环，在循环中，先拿到尾节点。

- if 分支，如果尾节点为 null，说明现在队列中还没有等待线程，则尝试 CAS 操作将头节点初始化，然后将尾节点也设置为头节点，因为初始化的时候头尾是同一个，这和 AQS 的设计实现有关， AQS 默认要有一个虚拟节点。此时，尾节点不在为空，循环继续，进入 else 分支；

- else 分支，如果尾节点不为 null，node.prev = t ，也就是将当前尾节点设置为待入队节点的前置节点。然后又是利用 CAS 操作，将待入队的节点设置为队列的尾节点，如果 CAS 返回 false，表示未设置成功，继续循环设置，直到设置成功，接着将之前的尾节点（也就是倒数第二个节点）的 next 属性设置为当前尾节点，对应 t.next = node 语句，然后返回当前尾节点，退出循环。

```java
/**
 * Sets head of queue, and checks if successor may be waiting
 * in shared mode, if so propagating if either propagate > 0 or
 * PROPAGATE status was set.
 *
 * @param node the node
 * @param propagate the return value from a tryAcquireShared
 */
private void setHeadAndPropagate(Node node, int propagate) {
    // 备份现在的 head
    Node h = head; // Record old head for check below
    // 抢到锁的线程被唤醒 将这个节点设置为head
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus either before
     *     or after setHead) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     */
    // propagate 一般都会大于 0 或者存在可被唤醒的线程
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        // 只有一个节点 或者是共享模式，释放所有等待线程，各自尝试抢占锁
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

`setHeadAndPropagate` 方法负责将自旋等待或被 `LockSupport` 阻塞的线程唤醒。

`Node` 对象中有一个属性是 `waitStatus`，它有四种状态，分别是：

```java
//线程已被 cancelled ，这种状态的节点将会被忽略，并移出队列
static final int CANCELLED =  1;
// 表示当前线程已被挂起，并且后继节点可以尝试抢占锁
static final int SIGNAL    = -1;
//线程正在等待某些条件
static final int CONDITION = -2;
//共享模式下 无条件所有等待线程尝试抢占锁
static final int PROPAGATE = -3;
```

### countDown() 方法

当执行 `CountDownLatch` 的 `countDown()` 方法，将计数器减一，也就是将 `state` 值减一，当减到 0 的时候，等待队列中的线程被释放。是调用 `AQS` 的 `releaseShared()` 方法来实现的。

```java
// CountDownLatch 类 countDown() 方法
public void countDown() {
    sync.releaseShared(1);
}

// AQS 类
public final boolean releaseShared(int arg) {
    // arg 为固定值 1
    // 如果计数器state 为 0 返回true，前提是调用 countDown() 之前能已经为 0
    if (tryReleaseShared(arg)) {
        // 唤醒等待队列的线程
        doReleaseShared();
        return true;
    }
    return false;
}

// CountDownLatch 类重写 AQS 方法
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    // 依然是循环 + CAS 配合 实现计数器减 1
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}

// AQS 类
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            // 如果节点状态为 SIGNAL，则他的 next 节点也可以尝试被唤醒
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            // 将节点状态设置为 PROPAGATE，表示要向下传播，依次唤醒
            else if (ws == 0 &&
                        !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

因为这是共享型的，当计数器为 0 后，会唤醒等待队列里的所有线程，所有调用了 `await()` 方法的线程都被唤醒，并发执行。这种情况对应到的场景是，有多个线程需要等待一些动作完成，比如一个线程完成初始化动作，其他 5 个线程都需要用到初始化的结果，那么在初始化线程调用 `countDown()` 之前，其他 5 个线程都处在等待状态。一旦执行线程调用了 countDown 方法将计数器减到 0，等待的 5 个线程都被唤醒，开始执行。

## 总结

- `AQS` 分为独占模式和共享模式，`CountDownLatch` 使用了它的共享模式；

- `AQS` 当第一个等待线程（被包装为 Node）要入队的时候，要保证存在一个 `head` 节点，这个 `head` 节点不关联线程，也就是一个虚节点；

- 当队列中的等待节点（关联线程的，非 `head` 节点）抢到锁，将这个节点设置为 `head` 节点；

- 第一次自旋抢锁失败后，`waitStatus` 会被设置为 -1（SIGNAL），第二次再失败，就会被 `LockSupport` 阻塞挂起；

- 如果一个节点的前置节点为 `SIGNAL` 状态，则这个节点可以尝试抢占锁。

## 参考资料

- [Java多线程之---用 CountDownLatch 说明 AQS 的实现原理](https://www.cnblogs.com/fengzheng/p/9153720.html)

- [从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)