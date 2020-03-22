---
title: JDK ThreadPoolExecutor 源码解析
date: 2020-03-18 22:02:59
cover: https://gitee.com/dongzl/article-images/raw/master/cover/java_study.png
# author information, multiple authors are set to array
# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨在总结 JDK 中 ThreadPoolExecutor 线程池相关知识内容。

categories: 
  - java开发
tags: 
  - ThreadPoolExecutor
  - 线程池
---

## 前言

学习 Java 的多线程知识就绕不开线程池，提起线程池那就必然要说到 ThreadPoolExecutor 类，自从 `阿里巴巴编码规范` 问世以来，ThreadPoolExecutor 可能也跟着火了一把，现在随便问个 Java 开发的基本都能背出来`线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险`。

笔者所参与的项目中很多地方都使用了线程池的 ThreadPoolExecutor 类去自定义线程池的一些参数。当然凡事没有绝对，使用 Executors 去创建线程池也是有的，如果只是开启数量可控的很少的线程去执行任务，也没必要大动干戈。

**为什么需要线程池**：
在实际使用中，线程是很占用系统资源的，如果对线程管理不善，很容易导致系统问题。因此，在大多数并发框架中都会使用线程池来管理线程，使用线程池管理线程主要有如下好处:
- 使用线程池可以重复利用已有的线程继续执行任务，避免线程在创建和销毁时造成的消耗；
- 由于没有线程创建和销毁时的消耗，可以提高系统响应速度；
- 通过线程可以对线程进行合理的管理，根据系统的承受能力调整可运行
线程数量的大小等。

## 线程池工作原理

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/15-ThreadPoolExecutor-Source-Code/ThreadPoolExecutor-Source-Code-04.png">

线程池执行所提交的任务过程: 
- 先判断线程池中核心线程池所有的线程是否都在执行任务。如果不是，则新创建一个线程执行刚提交的任务，否则，核心线程池中所有的线程都在执行任务，则进入第2步;
- 判断当前阻塞队列是否已满，如果未满，则将提交的任务放置在阻塞队列中；否则，则进入第3步；
- 判断线程池中所有的线程是否都在执行任务，如果没有，则创建一个新的线程来执行任务，否则，则交给饱和策略进行处理。

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/15-ThreadPoolExecutor-Source-Code/ThreadPoolExecutor-Source-Code-05.png">

## JDK 自带线程池实现

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/15-ThreadPoolExecutor-Source-Code/ThreadPoolExecutor-Source-Code-01.png">

线程池 | corePoolSize | maximumPoolSize | keepAliveTime | unit | workQueue | threadFactory | handler
-|-|-|-|-|-|-|-|
newCachedThreadPool | 0 | Integer.MAX_VALUE | 60 | TimeUnit.SECONDS | SynchronousQueue | DefaultThreadFactory | AbortPolicy
newFixedThreadPool | nThreads | nThreads | 0 | TimeUnit.MILLISECONDS | LinkedBlockingQueue | DefaultThreadFactory | AbortPolicy
newSingleThreadExecutor | 1 | 1 | 0 | TimeUnit.MILLISECONDS | LinkedBlockingQueue | DefaultThreadFactory | AbortPolicy
newSingleThreadScheduledExecutor | 1 | Integer.MAX_VALUE | 0 | TimeUnit.NANOSECONDS | DelayedWorkQueue | DefaultThreadFactory | AbortPolicy
newSingleThreadScheduledExecutor | corePoolSize | Integer.MAX_VALUE | 0 | TimeUnit.NANOSECONDS | DelayedWorkQueue | DefaultThreadFactory | AbortPolicy
newWorkStealingPool | - | - | - | - | - | - | -

补充： 
- 各种线程池实现中 threadFactory 参数可以自定义，也可以使用使用 JDK 默认 `DefaultThreadFactory` 实现类。
  
- `newWorkStealingPool` 属于 `ForkJoinPool` 线程池框架内容，实现上比较特殊。


参数说明
- corePoolSize:核心线程池的大小。
- maximumPoolSize:线程池能创建线程的最大个数
- keepAliveTime:空闲线程存活时间
- unit: 时间单位，为keepAlive Time指定时间单位
- workQueue:阻塞队列，用于保存任务的阻塞队列
- threadFactory: 创建线程的工程类
- handler:饱和策略(拒绝策略)

## JDK 自带阻塞队列实现

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/15-ThreadPoolExecutor-Source-Code/ThreadPoolExecutor-Source-Code-02.png">

## 线程池的生命周期

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/15-ThreadPoolExecutor-Source-Code/ThreadPoolExecutor-Source-Code-03.png">

- RUNNING：能接受新提交的任务，并且也能处理阻塞队列中的任务；

- SHUTDOWN：关闭状态，不再接受新提交的任务，但却可以继续处理阻
塞队列中已保存的任务；

- STOP：不能接受新任务，也不处理队列中的任务，会中断正在处理任务的
线程；

- TIDYING：如果所有的任务都已终止了，workerCount(有效线程数)为0，线程池进入该状态后会调用terminated()方法进入TERMINATED状态；

- TERMINATED：在 terminated (方法执行完后进入该状态，默认
terminated()方法中什么也没有做。

## 拒绝策略

- `ThreadPoolExecutor.AbortPolicy`：丢弃任务并抛出 `RejectedExecutionException` 异常；
- `ThreadPoolExecutor.DiscardPolicy`：也是丢弃任务，但是不抛
出异常；
- `ThreadPoolExecutor.DiscardOldestPolicy`：丢弃队列最前面的
任务，然后重新尝试执行任务(重复此过程)；
- `ThreadPoolExecutor.CallerRunsPolicy`：由调用线程处理该任务。

## ThreadPoolExecutor 源码解析

### 常用变量的解释

```java
// 1. `ctl`，可以看做一个int类型的数字，高3位表示线程池状态，低29位表示worker数量
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
// 2. `COUNT_BITS`，`Integer.SIZE`为32，所以`COUNT_BITS`为29
private static final int COUNT_BITS = Integer.SIZE - 3;
// 3. `CAPACITY`，线程池允许的最大线程数。1左移29位，然后减1，即为 2^29 - 1
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
// 4. 线程池有5种状态，按大小排序如下：RUNNING < SHUTDOWN < STOP < TIDYING < TERMINATED
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
// 5. `runStateOf()`，获取线程池状态，通过按位与操作，低29位将全部变成0
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 6. `workerCountOf()`，获取线程池worker数量，通过按位与操作，高3位将全部变成0
private static int workerCountOf(int c)  { return c & CAPACITY; }
// 7. `ctlOf()`，根据线程池状态和线程池worker数量，生成ctl值
private static int ctlOf(int rs, int wc) { return rs | wc; }

/*
 * Bit field accessors that don't require unpacking ctl.
 * These depend on the bit layout and on workerCount being never negative.
 */
// 8. `runStateLessThan()`，线程池状态小于xx
private static boolean runStateLessThan(int c, int s) {
    return c < s;
}
// 9. `runStateAtLeast()`，线程池状态大于等于xx
private static boolean runStateAtLeast(int c, int s) {
    return c >= s;
}
```

### 构造方法

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    // 基本类型参数校验
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    // 空指针校验
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    // 根据传入参数`unit`和`keepAliveTime`，将存活时间转换为纳秒存到变量`keepAliveTime `中
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

### 提交执行 task 的过程

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
    // worker数量比核心线程数小，直接创建worker执行任务
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // worker数量超过核心线程数，任务直接进入队列
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 线程池状态不是RUNNING状态，说明执行过shutdown命令，需要对新加入的任务执行reject()操作。
        // 这儿为什么需要recheck，是因为任务入队列前后，线程池的状态可能会发生变化。
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 这儿为什么需要判断0值，主要是在线程池构造方法中，核心线程数允许为0
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 如果线程池不是运行状态，或者任务进入队列失败，则尝试创建worker执行任务。
    // 这儿有3点需要注意：
    // 1. 线程池不是运行状态时，addWorker内部会判断线程池状态
    // 2. addWorker第2个参数表示是否创建核心线程
    // 3. addWorker返回false，则说明任务执行失败，需要执行reject操作
    else if (!addWorker(command, false))
        reject(command);
}
```

### addworker 源码解析

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    // 外层自旋
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 这个条件写得比较难懂，我对其进行了调整，和下面的条件等价
        // (rs > SHUTDOWN) || 
        // (rs == SHUTDOWN && firstTask != null) || 
        // (rs == SHUTDOWN && workQueue.isEmpty())
        // 1. 线程池状态大于SHUTDOWN时，直接返回false
        // 2. 线程池状态等于SHUTDOWN，且firstTask不为null，直接返回false
        // 3. 线程池状态等于SHUTDOWN，且队列为空，直接返回false
        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        // 内层自旋
        for (;;) {
            int wc = workerCountOf(c);
            // worker数量超过容量，直接返回false
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // 使用CAS的方式增加worker数量。
            // 若增加成功，则直接跳出外层循环进入到第二部分
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            // 线程池状态发生变化，对外层循环进行自旋
            if (runStateOf(c) != rs)
                continue retry;
            // 其他情况，直接内层循环进行自旋即可
            // else CAS failed due to workerCount change; retry inner loop
        } 
    }
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            // worker的添加必须是串行的，因此需要加锁
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                // 这儿需要重新检查线程池状态
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // worker已经调用过了start()方法，则不再创建worker
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    // worker创建并添加到workers成功
                    workers.add(w);
                    // 更新`largestPoolSize`变量
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            // 启动worker线程
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // worker线程启动失败，说明线程池状态发生了变化（关闭操作被执行），需要进行shutdown相关操作
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

### 线程池 worker 任务单元

```java
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    /**
     * This class will never be serialized, but we provide a
     * serialVersionUID to suppress a javac warning.
     */
    private static final long serialVersionUID = 6138294804551838833L;

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
     * Creates with given first task and thread from ThreadFactory.
     * @param firstTask the first task (null if none)
     */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        // 这儿是Worker的关键所在，使用了线程工厂创建了一个线程。传入的参数为当前worker
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
    }

    // 省略代码...
}
```

### 核心线程执行逻辑-runWorker

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    // 调用unlock()是为了让外部可以中断
    w.unlock(); // allow interrupts
    // 这个变量用于判断是否进入过自旋（while循环）
    boolean completedAbruptly = true;
    try {
        // 这儿是自旋
        // 1. 如果firstTask不为null，则执行firstTask；
        // 2. 如果firstTask为null，则调用getTask()从队列获取任务。
        // 3. 阻塞队列的特性就是：当队列为空时，当前线程会被阻塞等待
        while (task != null || (task = getTask()) != null) {
            // 这儿对worker进行加锁，是为了达到下面的目的
            // 1. 降低锁范围，提升性能
            // 2. 保证每个worker执行的任务是串行的
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            // 如果线程池正在停止，则对当前线程进行中断操作
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            // 执行任务，且在执行前后通过`beforeExecute()`和`afterExecute()`来扩展其功能。
            // 这两个方法在当前类里面为空实现。
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                // 帮助gc
                task = null;
                // 已完成任务数加一 
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // 自旋操作被退出，说明线程池正在结束
        processWorkerExit(w, completedAbruptly);
    }
}
```

## 参考资料
- [Java 线程池 ThreadPoolExecutor 八种拒绝策略浅析](https://mp.weixin.qq.com/s/ahHKn8qs96bTHURQlcvuNA)
