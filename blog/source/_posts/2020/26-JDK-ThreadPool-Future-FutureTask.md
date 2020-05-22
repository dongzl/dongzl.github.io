---
title: Java 线程池中 Future & FutureTask 使用总结
date: 2020-05-10 20:28:00
cover: https://gitee.com/dongzl/article-images/raw/master/cover/java_study.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨在总结 Java 线程池中 Future & FutureTask 相关知识内容。

categories: 
  - java开发

tags: 
  - Thread
---

## 背景描述

在工作中，对于 `Future` 和 `FutureTask` 类使用频率并不高，正好有个业务场景使用到了 `Future` 类，有些 API 使用并不熟练，这篇文章正好总结记录一下，方便以后学习和使用。

## Java 中创建线程几种方式对比

`Future` 和 `FutureTask` 都是 `Java` 中提供的多线程相关的几个工具类，提到了多线程，首先就要总结一下 `Java` 中创建线程的几种方式：

- 继承 `Thread` 类；
- 实现 `Runnable` 接口；
- 实现 `Callable` 接口；

通过继承 `Thread` 类来实现多线程这种方式就不多说了，由于 `Java` 中是单继承规则，继承 `Thread` 类就无法继承其他类，所以并不是一种特别推荐的方式；通过实现接口倒是提供了两种方式：`Runnable` 和 `Callable`，下面就来对比一下这两种实现方式：

```java
@FunctionalInterface
public interface Runnable {
    /**
     * When an object implementing interface <code>Runnable</code> is used
     * to create a thread, starting the thread causes the object's
     * <code>run</code> method to be called in that separately executing
     * thread.
     * <p>
     * The general contract of the method <code>run</code> is that it may
     * take any action whatsoever.
     *
     * @see     java.lang.Thread#run()
     */
    public abstract void run();
}

@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

多线程 | Runnable | Callable 
-|-|-
返回值 | 无返回值 | 有返回值（泛型）
异常 | 用户捕获异常 | 可以抛出异常
执行 | ExecutorService#submit() | ExecutorService#submit() or ExecutorService#execute()
与 Thread 类关系 | new Thread(Runnable target).start() 启动（Thread 类就实现了 Runnable 接口） | 与 Thread 没关系，只能与 ExecutorService 配合使用

## Future 接口定义

在 `JDK1.5` 之前，`Java` 中还没有提供 `Future` 和 `Callable` 相关接口，创建线程只能通过 `Thread` 类或者是 `Runnable` 接口，这两种方式最大的缺点就是在线程异步执行完毕后无法获取线程的执行结果，如果一定要获取执行结果，只能通过共享变量或者其它线程通信的方式来达到效果，这就是这两种方式备受诟病的地方。`JDK1.5` 之后，`Doug Lea` 大师给我们提供了 `Future` 和 `Callable` 几个相关的工具类，方便我们来获取线程的异步执行结果。

```java
public interface Future<V> {

    // 尝试取消任务执行。如果这个任务已经执行完成，或者已经被取消，或者一些其他原因导致任务无法被取消，该方法操作失败；
    // 如果取消成功，并且这个任务还没有被开始执行，这个任务将永远不会再被执行；
    // 如果任务已经开始执行，将由 mayInterruptIfRunning 参数决定是否尝试中断执行该任务的线程来尝试停止任务；
    // 这个方法执行返回结果后，如果随后调用 isDone 方法将永远返回 true；如果这个方法返回true，那么随后调用 isCancelled 方法将永远返回 true；
    // mayInterruptIfRunning 为 true，中断执行任务线程，false 执行中的任务将会一直执行完毕。
    boolean cancel(boolean mayInterruptIfRunning);

    // 判断任务是否被取消
    boolean isCancelled();

    // 判断任务是否执行完毕
    boolean isDone();
    
    // 用来获取执行结果，这个方法会产生阻塞，会一直等到任务执行完毕才返回
    V get() throws InterruptedException, ExecutionException;

    // 用来获取执行结果，如果在指定时间内，还没获取到结果，直接抛出 TimeoutException
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

总结一下，`Java` 中提供的 `Future` 接口主要三个作用：

- 取消任务执行；
- 判断任务是否执行完毕；
- 获取任务执行结果。

## ExecutorService 接口定义

`Future` 只是一个接口定义，需要和 `ExecutorService` 接口配合使用才能达到效果：

```java
public interface Executor {

    // 执行一个 Runnable 类型任务，无返回结果。
    void execute(Runnable command);
}

public interface ExecutorService extends Executor {

    ... ...

    // 执行一个 Callable 类型任务，并通过 Future#get() 获取执行结果。
    <T> Future<T> submit(Callable<T> task);

    // 执行一个 Runnable 类型任务，并通过 Future#get() 获取执行结果，使用比较少；
    // Runnable 类型的任务是不能返回执行结果，但是可以通过指定参数，返回执行结果。
    // 具体的实现逻辑在 RunnableAdapter 中，我理解应该是类似于通过共享变量来获取返回结果的意思。
    <T> Future<T> submit(Runnable task, T result);
    
    // 执行一个 Runnable 类型任务，并通过 Future 返回执行结果，但是返回的结果 Future#get() 为 null。
    Future<?> submit(Runnable task);

    ... ...
}
```

## FutureTask 类的定义

由于 `Future` 只是一个接口定义，是无法直接用来创建对象使用的，所以就有了 `FutureTask` 类。

```java
package java.util.concurrent;

public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}

public class FutureTask<V> implements RunnableFuture<V> {
    
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }

    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }
}
```

通过类的定义我们可以得知，`FutureTask` 类实现了 `RunnableFuture<V>` 接口，`RunnableFuture<V>` 接口继承自 `Runnable` 和 `Future<V>` 接口，所以 `FutureTask` 类是同时实现了 `Runnable` 和 `Future<V>` 接口。

在构造方法上，`FutureTask` 类对 `Callable` 和 `Runnable` 进行了一层封装。

```java
class App {

    ExecutorService executor = ...
    ArchiveSearcher searcher = ...

    void showSearch(final String target) throws InterruptedException {
        
        Future<String> future = executor.submit(new Callable<String>() {
            public String call() {
                return searcher.search(target);
            }});
        displayOtherThings(); // do other things while searching
        try {
            displayText(future.get()); // use future 
        } catch (ExecutionException ex) { 
            cleanup(); 
            return; 
        }
    
        FutureTask<String> future = new FutureTask<String>(new Callable<String>() {
            public String call() {
                return searcher.search(target);
            }
        });
        executor.execute(future);
    }
}
```

## FutureTask 类对 Future 接口的逻辑实现

`FutureTask` 类实现了 `Future` 接口，那么相应的也就实现了 `Future` 接口中的所有方法，下面我们就简单分析一下 `FutureTask` 类对这些方法的实现逻辑：

```java
/**
 * The run state of this task, initially NEW.  The run state
 * transitions to a terminal state only in methods set,
 * setException, and cancel.  During completion, state may take on
 * transient values of COMPLETING (while outcome is being set) or
 * INTERRUPTING (only while interrupting the runner to satisfy a
 * cancel(true)). Transitions from these intermediate to final
 * states use cheaper ordered/lazy writes because values are unique
 * and cannot be further modified.
 *
 * Possible state transitions:
 * NEW -> COMPLETING -> NORMAL
 * NEW -> COMPLETING -> EXCEPTIONAL
 * NEW -> CANCELLED
 * NEW -> INTERRUPTING -> INTERRUPTED
 */
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;
```

`FutureTask` 类中重要的就是 `state` 变量，并且对该变量定义了七种常量类型，一个 `FutureTask` 任务的执行过程就是在这些状态中流转的过程。最初是在 `FutureTask` 的构造方法中，会初始化 `state` 为 `NEW` 类型，所有流转都以该状态为起始。

```java
// CANCELLED、INTERRUPTING、INTERRUPTED 状态都表示任务已被取消
public boolean isCancelled() {
    return state >= CANCELLED;
}

// 状态不等于 NEW 都表示任务已完成
public boolean isDone() {
    return state != NEW;
}

// 1、判断状态是否是 NEW，同时 cas 修改 state 状态值；
// 2、根据 mayInterruptIfRunning 参数决定是中断线程，还是等待线程执行完成。
public boolean cancel(boolean mayInterruptIfRunning) {
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {    // in case call to interrupt throws exception
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    t.interrupt();
            } finally { // final state
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        finishCompletion();
    }
    return true;
}
```