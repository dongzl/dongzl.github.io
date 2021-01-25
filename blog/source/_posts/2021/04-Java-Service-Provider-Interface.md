---
title: Java SPI 使用及原理分析
date: 2021-01-16 10:19:04
cover: https://gitee.com/dongzl/article-images/raw/master/cover/java_study.png
# author information, multiple authors are set to array
# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨在对 Java 中 SPI 机制进行介绍并结合源码分析一下实现原理。

categories: 
  - java开发
tags: 
  - SPI
---

## 背景介绍

对于业务线上的一名研发来说，`Java` 的 `SPI` 机制使用还是不多的，反而倒是在很多的开源框架中，`Java` 的 `SPI` 机制被大量的应用，比如 `Apache Duboo` 框架，其内部很多的模块的扩展机制，比如注册中心、配置中心、负载均衡策略，都是通过 `Java` 的 `SPI` 机制来实现的，还比如笔者一直参与的开源框架 `Apache ShardingSphere` 的，在最新的 `5.X` 版本中实现的微内核、可插拔的机制，同样是通过 `Java` 的 `SPI` 机制来完成的。

[Dubbo 2.7 --> 开发指南 --> SPI 扩展实现](http://dubbo.apache.org/zh/docs/v2.7/dev/impls/)

那 `Java` `SPI` 机制到底是什么东西呢？其实 `Java` `SPI`（`Service Provider Interface`）是 `JDK` 内置的一种动态加载扩展点的实现。在 `ClassPath` 的 `META-INF/services` 目录下放置一个与接口同名的文本文件，文件的内容为接口的实现类，多个实现类用换行符分隔。`JDK` 中使用 `java.util.ServiceLoader` 来加载具体的实现。

## Java SPI 实战

- 定义一个接口 `IRegistry` 用于实现数据储存

```java
package com.dongzl.spi;

public interface IRegistry {

    void register(String url);
}
```

- 提供 `IRegistry` 的实现，`IRegistry` 有两个实现：`ZookeeperRegistry` 和 `EtcdRegistry`

```java
package com.dongzl.spi;

public class ZookeeperRegistry implements IRegistry {

    @Override
    public void register(String url) {
        System.out.println("Register " + url + " service to Zookeeper");
    }
}
```

```java
package com.dongzl.spi;

public class EtcdRegistry implements IRegistry {

    @Override
    public void register(String url) {
        System.out.println("Register " + url + " service to Etcd");
    }
}
```

- 添加配置文件，在 `META-INF/services` 目录添加一个文件，文件名和接口全名称相同，所以文件完整路径是 `META-INF/services/com.dongzl.spi.IRegistry`，文件内容为：

```
com.dongzl.spi.ZookeeperRegistry
com.dongzl.spi.EtcdRegistry
```

- 通过 `ServiceLoader` 加载 `IRepository` 实现

```java
package com.dongzl.spi;

import java.util.Iterator;
import java.util.ServiceLoader;

public class Test {

    public static void main(String[] args) throws Exception {
        ServiceLoader<IRegistry> serviceLoader = ServiceLoader.load(IRegistry.class);
        Iterator<IRegistry> iterator = serviceLoader.iterator();
        while (iterator != null && iterator.hasNext()){
            IRegistry registry = iterator.next();
            System.out.println("class: " + registry.getClass().getName());
            registry.register("SPI");
        }
    }
}
```

在上面的例子中，我们定义了一个扩展点和它的两个实现。在 `ClassPath` 中添加了扩展的配置文件，最后使用 `ServiceLoader` 来加载所有的扩展点。 最终的输出结果为：

```java
class: com.dongzl.spi.ZookeeperRegistry
Register SPI service to Zookeeper
class: com.dongzl.spi.EtcdRegistry
Register SPI service to Etcd
```

## Java SPI 实现原理

```java
private static final String PREFIX = "META-INF/services/";

// 表示正在加载的服务的类或接口
private final Class<S> service;

// 用于定位、加载和实例化提供程序的类加载器
private final ClassLoader loader;

// 创建 ServiceLoader 时采用的访问控制上下文
private final AccessControlContext acc;

// 缓存服务 providers, 按实例化顺序缓存
private LinkedHashMap<String,S> providers = new LinkedHashMap<>();

// 延迟查找迭代器
private LazyIterator lookupIterator;

```

```java
/**
 * Creates a new service loader for the given service type, using the
 * current thread's {@linkplain java.lang.Thread#getContextClassLoader
 * context class loader}.
 *
 * <p> An invocation of this convenience method of the form
 *
 * <blockquote><pre>
 * ServiceLoader.load(<i>service</i>)</pre></blockquote>
 *
 * is equivalent to
 *
 * <blockquote><pre>
 * ServiceLoader.load(<i>service</i>,
 *                    Thread.currentThread().getContextClassLoader())</pre></blockquote>
 *
 * @param  <S> the class of the service type
 *
 * @param  service
 *         The interface or abstract class representing the service
 *
 * @return A new service loader
 */
public static <S> ServiceLoader<S> load(Class<S> service) {
    ClassLoader cl = Thread.currentThread().getContextClassLoader();
    return ServiceLoader.load(service, cl);
}

public static <S> ServiceLoader<S> load(Class<S> service, ClassLoader loader) {
    return new ServiceLoader<>(service, loader);
}
```

我们看到，在 `ServiceLoader.java` 的 `load(Class<S> service)` 方法中，使用当前线程的 `ClassLoader` 作为参数，创建了一个 `ServiceLoader` 对象，通过注释我们也可以了解到，默认不指定类加载参数的的情况下：

```java
ServiceLoader.load(service);
```

与

```java
ServiceLoader.load(service, Thread.currentThread().getContextClassLoader());
```

是等价的。

在 `ServiceLoader` 构造方法有两个参数，分别是 `Class` 对象和指定的类加载器。在构造方法中完成了两件工作：一是变量赋值，二是调用 `reload()` 方法。而 `reload()` 方法的作用是根据接口的 `Class` 对象和类加载器来初始化 `LazyIterator` 对象。

```java
private ServiceLoader(Class<S> svc, ClassLoader cl) {
    service = Objects.requireNonNull(svc, "Service interface cannot be null");
    loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
    acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
    reload();
}

public void reload() {
    providers.clear();
    lookupIterator = new LazyIterator(service, loader);
}
```

在调用 `ServiceLoader` 的 `iterator()` 方法时，在内部创建了 `java.util.Iterator` 接口的匿名实现：

```java
public Iterator<S> iterator() {
    return new Iterator<S>() {

        Iterator<Map.Entry<String,S>> knownProviders
            = providers.entrySet().iterator();

        public boolean hasNext() {
            if (knownProviders.hasNext())
                return true;
            return lookupIterator.hasNext();
        }

        public S next() {
            if (knownProviders.hasNext())
                return knownProviders.next().getValue();
            return lookupIterator.next();
        }

        public void remove() {
            throw new UnsupportedOperationException();
        }

    };
}
```

在 `Iterator` 接口 `hasNext()` 和 `next()` 方法匿名实现中，首先会从全局变量 `providers` 判断是否已经缓存了扩展服务类，如果已缓存，直接返回结果；如果还未缓存，则会继续调用 `LazyIterator` 类中对应的 `hasNext()` 和 `next()` 方法。

在 `LazyIterator` 类内部实现中，`hasNext()` 方法逻辑主要在 `hasNextService()` 方法中完成，`next()` 方法逻辑主要在 `nextService()` 方法中完成。

- `hasNextService()` 方法主要逻辑

```java
// 获取完整路径名称（包名 + 接口名）
String fullName = PREFIX + service.getName();

// 加载配置，得到配置文件的URL集合（可能有多个配置文件）
configs = loader.getResources(fullName);

// 参数是接口的 Class 对象和配置文件的 URL 来解析配置文件
// 返回值是配置文件里面的内容，也就是实现类的全名（包名+类名）
pending = parse(service, configs.nextElement());

// 在 parse 方法中通过字符流的方式读取文件内容
```

- `nextService()` 方法主要逻辑

```java
// 通过 Class.forName java 反射机制加载类
c = Class.forName(cn, false, loader);

// 创建对象
S p = service.cast(c.newInstance());

// 缓存到全局的 LinkedHashMap
providers.put(cn, p);

```

通过这一段分析，我们也就能理解到，在 `hasNextService()` 方法内部只是完成了服务类配置的解析和读取，并没有真正完成具体实现类的初始化和加载，真正的类实例化和加载是在调用 `nextService()` 方法中完成的，这就是为什么 `LazyIterator` 类名中会有 `Lazy`，`Lazy` 主要的意思是在 `ServiceLoader` 初始化中并不会完成服务类的加载，甚至在调用 `Iterator` 对象的 `hasNext()` 方法（对应 `LazyIterator` 的 `hasNextService()` 方法）依旧没有进行类的加载，而真正的加载需要延迟到调用 `Iterator` 对象的 `next()` 方法（对应 `LazyIterator` 的 `nextService()` 方法）中来完成，这就是延迟加载的真正含义。

**loadInstalled() 方法与 load() 方法区别**

在 `ServiceLoader` 类中，除了 `load()` 方法，还有 `loadInstalled()` 方法，这个方法逻辑并不复杂：

```java
/**
 * Creates a new service loader for the given service type, using the
 * extension class loader.
 *
 * <p> This convenience method simply locates the extension class loader,
 * call it <tt><i>extClassLoader</i></tt>, and then returns
 *
 * <blockquote><pre>
 * ServiceLoader.load(<i>service</i>, <i>extClassLoader</i>)</pre></blockquote>
 *
 * <p> If the extension class loader cannot be found then the system class
 * loader is used; if there is no system class loader then the bootstrap
 * class loader is used.
 *
 * <p> This method is intended for use when only installed providers are
 * desired.  The resulting service will only find and load providers that
 * have been installed into the current Java virtual machine; providers on
 * the application's class path will be ignored.
 *
 * @param  <S> the class of the service type
 *
 * @param  service
 *         The interface or abstract class representing the service
 *
 * @return A new service loader
 */
public static <S> ServiceLoader<S> loadInstalled(Class<S> service) {
    ClassLoader cl = ClassLoader.getSystemClassLoader();
    ClassLoader prev = null;
    while (cl != null) {
        prev = cl;
        cl = cl.getParent();
    }
    return ServiceLoader.load(service, prev);
}
```

`load()` 方法和 `loadInstalled()` 方法最大的区别是使用类的加载器不同，`load()` 方法使用 `Thread.currentThread().getContextClassLoader()` 作为类加载器；而 `loadInstalled()` 方法在 `while` 循环内部，通过逐级向上查找最顶级的父 `ClassLoader` 来作为 `ServiceLoader` 的类加载器，最终使用类加载器是按照如下顺序来完成的：

ExtClassLoader --> SysClassLoader --> Bootstrap ClassLoader

那么这个方法的操作存在的意义是什么呢？在注释中也有一段描述：

> 将仅查找并加载已安装到当前的 Java 虚拟机中的 provider 产生的服务；应用程序类路径的 provider 将被忽略。

如果将上面实战案例换成如下测试代码：

```java
ServiceLoader<IRegistry> serviceLoader = ServiceLoader.loadInstalled(IRegistry.class);
```

执行测试程序是不会有任何输出的，也就是我们在应用程序内部定义的 `SPI` 扩展并没有被加载；如果我们将测试程序打成 `jar` 包，放入 `JDK` 安装目录 `jre/lib/ext` 目录下面，再执行我们的测试程序，会正常产生结果，说明我们打包的 `SPI` 扩展已经被正常加载。

## Java SPI 存在不足

`Java` `SPI` 使用虽然简单，也做到了基本的加载扩展点的功能。但还是存在以下的不足：

 - `ServiceLoader` 虽然使用了延迟加载的思想，但是还是会通过遍历一次性加载所有的扩展实现，也就是对服务的实现类需要全部加载并实例化一遍。如果我们并不想使用某些实现类，也同样会被加载并实例化了，这就造成了资源浪费；或者某些服务实例化比较耗时，也会拖慢整个系统性能；

- 获取某个实现类的方式不够灵活，只能通过 `Iterator` 遍历形式获取，无法根据某个参数来获取对应的实现类。

## 总结

在这篇文章中，我们通过对一些著名的 `Java` 开源框架可扩展机制的实现原理分析，引出了 `Java` `SPI` 机制，接下来通过一个小的实战案例演示了 `Java` `SPI` 机制的使用方式，并结合 JDK 源码，对 `Java` `SPI` 实现原理进行了分析，最后我们还总结了使用原生的 `Java` `SPI` 机制可能存在的一些不足，这里我们也留个伏笔，对于存在的不足，我们还有没有更好的实现方案呢？在后续的文章中我们会继续分析。

## 参考链接

- [Java SPI(Service Provider Interface)简介](https://blog.csdn.net/top_code/article/details/51934459)

- [Java SPI机制详解](https://juejin.cn/post/6844903605695152142)

- [Dubbo可扩展机制实战](http://dubbo.apache.org/zh/blog/2019/04/25/dubbo%E5%8F%AF%E6%89%A9%E5%B1%95%E6%9C%BA%E5%88%B6%E5%AE%9E%E6%88%98/)

- [java.util.ServiceLoader源码分析](https://blog.csdn.net/liangyihuai/article/details/50716369)

- [ServiceLoader使用及原理分析](https://blog.csdn.net/a910626/article/details/78811273?utm_source=blogxgwz9)

- [Java SPI机制：ServiceLoader实现原理及应用剖析](https://juejin.cn/post/6844903891746684941)
