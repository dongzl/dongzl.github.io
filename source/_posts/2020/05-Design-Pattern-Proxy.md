---
title: 聊聊代理模式（Proxy）的使用
date: 2020-02-29 14:44:21
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/design_pattern.png
# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl
# post subtitle in your index page
subtitle: 本文旨在学习代理模式中静态代理 & 动态代理的实现技术。
categories: 
  - 设计模式
tags: 
  - 设计模式
---

## 背景介绍

最近在学习 [极客时间](https://time.geekbang.org/) 的 [设计模式之美](https://time.geekbang.org/column/intro/250) 课程，这一篇算是自己的学习总结。代理模式（Proxy）在很多非常优秀框架中都有他的身影，比如，大名鼎鼎的 Spring AOP 就是使用动态代理（Dynamic Proxy）模式；还有就是在一些微服务框架中，如果调用一个远程服务接口，RPC 框架一般都会使用代理模式。

<!-- more -->

## 代理模式介绍

**代理模式定义**

**Proxy Pattern**: Provide a surrogate or placeholder for another object to control access to it.（为其他对象提供一种代理以控制对这个对象的访问。）

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/05-Design-Pattern-Proxy/Design-Pattern-Proxy_01.png" width="600px">

- Subject：定义 RealSubject 对外的接口，且这些接口必须被 Proxy 实现，这样外部调用 proxy 的接口最终都被转化为对 RealSubject 的调用。
- RealSubject：真正的目标对象（被代理对象）。
- Proxy：目标对象的代理，负责控制和管理目标对象，并间接地传递外部对目标对象的访问。

## 静态代理实现

```java

public class StaticProxy {
    
    public static void main(String[] args) throws Exception {
        new ProxyExecute(new RemoteExecute()).execute();
    }
}

interface Executable {
    
    void execute() throws Exception;
}

class RemoteExecute implements Executable {
    
    @Override
    public void execute() throws Exception {
        Thread.sleep(1000L);
        System.out.println("Remote execute ......");
    }
}

class ProxyExecute implements Executable {
    
    private Executable executable;
    
    public ProxyExecute(Executable executable) {
        this.executable = executable;
    }
    
    @Override
    public void execute() throws Exception {
        long start = System.currentTimeMillis();
        executable.execute();
        System.out.println("Time: " + (System.currentTimeMillis() - start));
    }
}
```

上述代码就是静态代理个一个简单实现，由于在静态代理模式中 Proxy 只能代理固定实现某个接口的被代理对象，所以并不十分灵活，所以在这个静态代理的基础上就衍生出了动态代理（Dynamic Proxy）。

## JDK 动态代理实现

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class JDKDynamicProxy {
    
    public static void main(String[] args) throws Exception {
        Executable remote = new RemoteExecute();
        
        System.getProperties().setProperty("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
        
        Executable executable = (Executable)Proxy.newProxyInstance(JDKDynamicProxy.class.getClassLoader(), new Class[]{Executable.class}, new ExecuteInvocationHandler(remote));
        executable.execute();
    }
}

class ExecuteInvocationHandler implements InvocationHandler {
    
    private Executable executable;
    
    public ExecuteInvocationHandler(Executable executable) {
        this.executable = executable;
    }
    
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = method.invoke(executable, args);
        System.out.println("Time: " + (System.currentTimeMillis() - start));
        return result;
    }
}

interface Executable {
    
    void execute() throws Exception;
}

class RemoteExecute implements Executable {
    
    @Override
    public void execute() throws Exception {
        Thread.sleep(1000L);
        System.out.println("Remote execute ......");
    }
}
```

JDK 的动态代理是通过 Proxy 类来实现的

```java
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) 
throws IllegalArgumentException
```
newProxyInstance 方法需要三个参数，
- loader 是生成代理类的类加载器；
- interfaces 指定被代理类实现的接口（所以很多资料都说 JDK 的动态代理需要被代理类必须实现某一个接口，其实证据就在这里）；
- h 调用处理器（实现 InvocationHandler 接口，InvocationHandler 接口中定义了 invoke 方法）。

通过设置 JDK 属性，我们可以将动态代理过程中生成的代理类保存下来，观察 JDK 动态生成的代理类的一些实现细节：

```java
System.getProperties().setProperty("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
```
PS. **这个属性值在不同 JDK 版本名称不同，JDK8 中为 `sun.misc.ProxyGenerator.saveGeneratedFiles`，JDK12 中是 `jdk.proxy.ProxyGenerator.saveGeneratedFiles`，没有深究是从 JDK 哪个版本改的这个属性名。**

通过设置这个参数，在程序运行后会生成一个 `$Proxy0.class` 文件，我们通过工具反编译文件后内容如下：

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package design.pattern.proxy.v2;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

final class $Proxy0 extends Proxy implements Executable {

    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    // 省略 equals 方法
    // 省略 toString 方法

    public final void execute() throws Exception {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (Exception | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    // 省略 hashCode 方法

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("design.pattern.proxy.v2.Executable").getMethod("execute");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

通过类的结构我们可以知道动态生成代理类 `$Proxy0` 继承自 `Proxy` 类，同时实现了我们自定义的 `Executable` 接口，并重写了其中的 `execute` 方法，`execute` 方法实现很简单：

```java
super.h.invoke(this, m3, (Object[])null);
```
调用父类中 `h` 变量的 `invoke` 方法，其中 `h` 变量就是我们实现的 `InvocationHandler` 接口的 `ExecuteInvocationHandler` 类变量。

JDK 动态代理实现流程图：

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/05-Design-Pattern-Proxy/Design-Pattern-Proxy_02.png">

## cglib 动态代理

```java
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

public class CGlibDynamicProxy {
    
    public static void main(String[] args) throws Exception {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(RemoteExecute.class);
        enhancer.setCallback(new ExecuteMethodInterceptor());
        RemoteExecute execute = (RemoteExecute)enhancer.create();
        execute.execute();
    }
}

class ExecuteMethodInterceptor implements MethodInterceptor {
    
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println(o.getClass().getSuperclass().getName());
        long start = System.currentTimeMillis();
        Object result = methodProxy.invokeSuper(o, objects);
        System.out.println("Time: " + (System.currentTimeMillis() - start));
        return result;
    }
}

class RemoteExecute {
    
    public void execute() throws Exception {
        Thread.sleep(1000L);
        System.out.println("Remote execute ......");
    }
}
```

cglib 实现的动态代理功能使用的是 `Enhancer` 类，其中需要定义 `MethodInterceptor` 接口实现类，`MethodInterceptor` 接口与 JDK 动态代理中的 InvocationHandler 接口作用非常类型，内部通过反射调用被代理类的方法。

`cglib` 动态代理一个比较的的优势是被代理类（RemoteExecute）不需要实现任何接口，这个就是很多资料上说的 `cglib` 动态代理与 `JDK` 动态代理的一个很大区别就是：**cglib 动态代理不需要被代理类实现任何接口，而 JDK 动态代理需要被代理类必须要实现一个接口。** 由于 `cglib` 动态代理的这个优势，所以 `Spring AOP` 的动态代理就是通过 `cglib` 来实现的。

## 代理模式使用总结

代理模式的目的：
- 为外部调用者提供一个访问服务提供者的代理对象。

代理模式的动机：
- 限制对目标对象的直接访问，降低耦合度。

代理模式优点：
- 低耦合（协调调用者和被调用者）
- 易扩展
- 灵活度高

代理模式缺点：
- 间接访问可能会延迟请求相应
- 增加工作量

代理模式分类：
- 静态代理
- 动态代理

## 参考资料

- [设计模式之代理模式（proxy pattern）](https://www.cnblogs.com/yssjun/archive/2019/05/31/10889022.html)

- [JDK动态代理](https://www.cnblogs.com/liuyun1995/p/8144628.html)

- [面试不再怕-说透动静态代理！](https://my.oschina.net/jiangxinJava/blog/4286243)