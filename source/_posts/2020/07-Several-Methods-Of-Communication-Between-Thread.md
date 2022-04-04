---
title: 论“茴”字的四种写法：一道面试题总结线程间通信的几种方式
date: 2020-03-07 21:35:58
cover: https://gitee.com/dongzl/article-images/raw/master/cover/java_study.png
# author information, multiple authors are set to array
# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨在通过一道面试题总结线程间通信的几种方式。

categories: 
  - java开发
tags: 
  - Thread
  - Lock
  - synchronized
---

## 背景描述

> 用两个线程，一个输出字母，一个输出数字，交替输出：1A2B3C4D5E...26Z。

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/07-Several-Methods-Of-Communication-Between-Thread/Several-Methods-Of-Communication-Between-Thread.png">

这是一道典型的线程间通信的面试题，两个线程交替**运行-暂停**，并且当一个线程运行后需要暂停，同时要通知另外一个线程运行；另外一个线程得到通知后开始运行，运行后暂停并通知对方线程运行，彼此交替运行，直至打印完整结果。对于这个问题有很多中解法，下面我们就一一分析这些方法。

<!-- more -->

## 解决方案

### LockSupport 类实现

```java
import java.util.concurrent.locks.LockSupport;

public class LockSupportDemo {
    
    private static Thread t1, t2;
    
    public static void main(String[] args) {
        String[] digit = new String[]{"1", "2", "3", "4", "5", "6", "7", "8", "9", "10"};
        String[] alphabet = new String[]{"A", "B", "C", "D", "E", "F", "G", "H", "I", "J"};
        
        t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                for (String d : digit) {
                    System.out.println(d);
                    LockSupport.unpark(t2);
                    LockSupport.park(t1);
                }
            }
        }, "t1");
        
        t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                for (String a : alphabet) {
                    LockSupport.park(t2);
                    System.out.println(a);
                    LockSupport.unpark(t1);
                }
            }
        }, "t2");
        
        t1.start();
        t2.start();
    }
}
```

`LockSupport` 中的 `park` 和 `unpark` 可以实现线程的阻塞与唤醒。

- `park`: Disables the current thread for thread scheduling purposes unless the permit is available.

- `unpark`: Makes available the permit for the given thread, if it was not already available.

### while 循环 + volatile 变量实现

```java
public class WhileCycleDemo {
    
    enum RunThreadEnum {T1, T2}
    
    private static volatile RunThreadEnum run = RunThreadEnum.T1;
    
    public static void main(String[] args) {
        String[] digit = new String[]{"1", "2", "3", "4", "5", "6", "7", "8", "9", "10"};
        String[] alphabet = new String[]{"A", "B", "C", "D", "E", "F", "G", "H", "I", "J"};
    
        new Thread(new Runnable() {
            @Override
            public void run() {
                for (String d : digit) {
                    while (run != RunThreadEnum.T1) {}
                    System.out.println(d);
                    run = RunThreadEnum.T2;
                }
            }
        }, "t1").start();
    
        new Thread(new Runnable() {
            @Override
            public void run() {
                for (String a : alphabet) {
                    while (run != RunThreadEnum.T2) {}
                    System.out.println(a);
                    run = RunThreadEnum.T1;
                }
            }
        }, "t2").start();
    }
}
```

对于 `while 循环 + volatile 变量` 这种实现方案，程序并不难理解，通过交替设置某个变量值的方式实现效果，不过这种方式实现需要注意一点就是 `run` 变量一定要使用 `volatile` 关键字修饰，保证变量的内存可见性。

### AtomicBoolean 类实现

```java
import java.util.concurrent.atomic.AtomicBoolean;

public class AtomicBooleanDemo {
    
    static AtomicBoolean run = new AtomicBoolean(false);
    
    public static void main(String[] args) {
        String[] digit = new String[]{"1", "2", "3", "4", "5", "6", "7", "8", "9", "10"};
        String[] alphabet = new String[]{"A", "B", "C", "D", "E", "F", "G", "H", "I", "J"};
    
        new Thread(new Runnable() {
            @Override
            public void run() {
                for (String d : digit) {
                    while (run.get()) {}
                    System.out.println(d);
                    run.set(true);
                }
            }
        }, "t1").start();
    
        new Thread(new Runnable() {
            @Override
            public void run() {
                for (String a : alphabet) {
                    while (!run.get()) {}
                    System.out.println(a);
                    run.set(false);
                }
            }
        }, "t2").start();
    }
}
```

`AtomicBoolean` 类实现也比较好理解，主要是借助内部API实现来保证变量在线程之间的可见性。

### BlockingQueue 阻塞队列实现

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class BlockingQueueDemo {
    
    static BlockingQueue<String> queue1 = new ArrayBlockingQueue<>(1);
    static BlockingQueue<String> queue2 = new ArrayBlockingQueue<>(1);
    
    public static void main(String[] args) {
        String[] digit = new String[]{"1", "2", "3", "4", "5", "6", "7", "8", "9", "10"};
        String[] alphabet = new String[]{"A", "B", "C", "D", "E", "F", "G", "H", "I", "J"};
    
        new Thread(new Runnable() {
            @Override
            public void run() {
                for (String d : digit) {
                    System.out.println(d);
                    try {
                        queue1.put("ok");
                        queue2.take();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }, "t1").start();
    
        new Thread(new Runnable() {
            @Override
            public void run() {
                for (String a : alphabet) {
                    try {
                        queue1.take();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(a);
                    try {
                        queue2.put("ok");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }, "t2").start();
    }
}
```
借助 `BlockingQueue` 类实现，主要是借助阻塞队列的阻塞特性，当队列为空时，调用阻塞队列的 `take` 方法，会阻塞当前线程的执行，直到队列不为空后唤醒当前线程继续执行。

### PipedInputStream & PipedOutputStream 实现

```java
import java.io.IOException;
import java.io.PipedInputStream;
import java.io.PipedOutputStream;

public class PipedStreamDemo {
    
    public static void main(String[] args) throws Exception {
        String[] digit = new String[]{"1", "2", "3", "4", "5", "6", "7", "8", "9", "10"};
        String[] alphabet = new String[]{"A", "B", "C", "D", "E", "F", "G", "H", "I", "J"};
    
        PipedInputStream input1 = new PipedInputStream();
        PipedInputStream input2 = new PipedInputStream();
        PipedOutputStream output1 = new PipedOutputStream();
        PipedOutputStream output2 = new PipedOutputStream();
        
        input1.connect(output2);
        input2.connect(output1);
        
        String msg = "exchange";
    
        new Thread(new Runnable() {
            @Override
            public void run() {
                byte[] buffer = new byte[8];
                try {
                    for (String d : digit) {
                        System.out.println(d);
                        output1.write(msg.getBytes());
                        input1.read(buffer);
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }, "t1").start();
    
        new Thread(new Runnable() {
            @Override
            public void run() {
                byte[] buffer = new byte[8];
                try {
                    for (String a : alphabet) {
                        input2.read(buffer);
                        System.out.println(a);
                        output2.write(msg.getBytes());
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }, "t2").start();
    }
}
```

这种事借助了 `java.io` 包中的 `PipedInputStream & PipedOutputStream` 来实现，这两个类在实际中真是没有使用过，而且从程序运行效果来看，上述代码执行效率非常之低，其实这种实现方案只是一个凑数，开脑洞的方案。

### synchronized + wait() + notifyAll() 实现

```java
public class WaitNotifyDemo {
    
    public static void main(String[] args) {
        
        Object o = new Object();
        
        String[] digit = new String[]{"1", "2", "3", "4", "5", "6", "7", "8", "9", "10"};
        String[] alphabet = new String[]{"A", "B", "C", "D", "E", "F", "G", "H", "I", "J"};
    
        new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (o) {
                    for (String d : digit) {
                        System.out.println(d);
                        try {
                            o.notifyAll();
                            o.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    o.notifyAll(); //一定再调用一次，否则程序无法退出执行
                }
            }
        }, "t1").start();
    
        new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (o) {
                    for (String a : alphabet) {
                        System.out.println(a);
                        try {
                            o.notifyAll();
                            o.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    o.notifyAll();
                }
            }
        }, "t2").start();
    }
}
```
通过 `Object` 类中 `wait()` & `notifyAll()` 方法使用，也可以达到交替打印的效果。其中 `wait()` 方法作用是使当前线程阻塞，释放资源对象 o，`notifyAll()` 方法作用是唤醒正在等待资源对象 o 的线程，使其继续向下执行。这里需要注意一点是在循环打印输出之后，一定要再次调用`notifyAll()` 方法，因为两个线程交替**执行-等待**，最后一定会有一个线程处于等待状态，如果不最后再调用一次`notifyAll()` 方法，那么一定会有一个线程无法退出执行，程序也就无法终止。

PS. 这里还有一点需要注意就是上述程序无法控制**数字和字母输出先后顺序，也行是数字先输出，也许是字母先输出**，因为线程两断代码完全一致，执行先后顺序无法确定，这个问题我们可以借助 `CountDownLatch` 工具类或者通过一个标志变量来处理，程序如下：

```java
import java.util.concurrent.CountDownLatch;

public class WaitNotifyDemo {
    
    private static volatile boolean flag = false;
    
    public static void main(String[] args) {
        
        Object o = new Object();
    
        CountDownLatch latch = new CountDownLatch(1);
        
        String[] digit = new String[]{"1", "2", "3", "4", "5", "6", "7", "8", "9", "10"};
        String[] alphabet = new String[]{"A", "B", "C", "D", "E", "F", "G", "H", "I", "J"};
    
        new Thread(new Runnable() {
            @Override
            public void run() {
                //latch.await(); //借助 CountDownLatch 工具解决先后顺序问题(需要处理异常)
                synchronized (o) {
                    while (!flag) {
                        try {
                            o.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    for (String d : digit) {
                        System.out.println(d);
                        try {
                            o.notifyAll();
                            o.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    o.notifyAll(); //一定再调用一次，否则程序无法退出执行
                }
            }
        }, "t1").start();
    
        new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (o) {
                    for (String a : alphabet) {
                        System.out.println(a);
                        //latch.countDown();
                        flag = true;
                        try {
                            o.notifyAll();
                            o.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                    o.notifyAll();
                }
            }
        }, "t2").start();
    }
}
```

### Lock + Condition 实现

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class LockConditionDemo {
    
    public static void main(String[] args) {
        
        String[] digit = new String[]{"1", "2", "3", "4", "5", "6", "7", "8", "9", "10"};
        String[] alphabet = new String[]{"A", "B", "C", "D", "E", "F", "G", "H", "I", "J"};
    
        Lock lock = new ReentrantLock();
        Condition condition1 = lock.newCondition();
        Condition condition2 = lock.newCondition();
        
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    lock.lock();
                    for (String d : digit) {
                        System.out.println(d);
                        condition2.signalAll();
                        condition1.await();
                    }
                    condition2.signalAll(); // 需要调用一次，否则程序无法终止
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        }, "t1").start();
        
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    lock.lock();
                    for (String a : alphabet) {
                        System.out.println(a);
                        condition1.signalAll();
                        condition2.await();
                    }
                    condition1.signalAll();
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }
        }, "t2").start();
    }
}
```
`Lock + Condition` 实现方案与 `synchronized + wait() + notifyAll() `实现方案非常类似，包括使用的API都存在相似的对应关系：

- synchronized 关键字 VS Lock 类
- Condition.await() VS Object.wait()
- Condition.signalAll() VS Object.notifyAll()

包括需要注意的问题，在使用 `Lock + Condition` 时我们也需要在循环执行最后在调用一次 `signalAll()` 方法，否则程序无法终止运行。

## 最后总结

对于这道面试题，主要还是考察线程之间通信问题，主要的考点应该是在`synchronized + wait() + notifyAll() ` 的使用上，当然对于其他实现方案，比较好的是：
- Lock + Condition 实现
- LockSupport 类实现

对于其它的方案，有的是使用了一些编程技巧，有的是利用 JDK 中现有类的一些实现，并不是重点考察方向，像 `PipedInputStream & PipedOutputStream` 的实现方案，虽然可以达到效果，其实有些充数的嫌疑，如果有面试官硬扣这种实现，就有点像鲁迅笔下的孔乙己先生在和你讨论“茴”字有几种写法的味道了。


