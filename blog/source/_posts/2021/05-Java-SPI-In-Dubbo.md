---
title: Apache Dubbo 可扩展机制分析
date: 2021-01-23 18:00:56
cover: https://gitee.com/dongzl/article-images/raw/master/cover/dubbo_study.png
# author information, multiple authors are set to array
# single author
author:
  - nick: Apache Dubbo
    link: http://dubbo.apache.org/

# post subtitle in your index page
subtitle: 本文转载自 Apache Dubbo 官网文章，旨在通过源码对 Dubbo 可扩展机制进行深入分析。

categories: 
  - java开发
tags: 
  - SPI
---

## 背景介绍

在上一篇文章 [Java SPI 使用及原理分析](https://dongzl.github.io/2021/01/16/04-Java-Service-Provider-Interface/) 中我们通过对一些著名的 `Java` 开源框架可扩展机制的实现原理分析，引出了 `Java` `SPI` 机制，同时通过一个小的实战案例演示了 `Java` `SPI` 机制的使用方式，并结合 JDK 源码，对 `Java` `SPI` 实现原理进行了分析，最后还总结了使用原生的 `Java` `SPI` 机制可能存在的一些不足；这一篇文章中再结合 `Apache Dubbo` 开源框架，来分析一下 `Java` `SPI` 在实践场景中的应用，我们知道 Java SPI 机制还是存在一些不足的，那这些不足在 `Apache Dubbo` 框架又是如何解决的呢？

PS. 这一篇文章的内容并非原创，而是来源于 [Apache Dubbo](http://dubbo.apache.org/) 官网的博客文章，文章的版权属于 `Apache Dubbo`，本人只对原文部分内容进行排版和美化。

- [Dubbo可扩展机制实战](http://dubbo.apache.org/zh/blog/2019/04/25/dubbo%E5%8F%AF%E6%89%A9%E5%B1%95%E6%9C%BA%E5%88%B6%E5%AE%9E%E6%88%98/)

- [Dubbo可扩展机制源码解析](http://dubbo.apache.org/zh/blog/2019/05/02/dubbo%E5%8F%AF%E6%89%A9%E5%B1%95%E6%9C%BA%E5%88%B6%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/)

# Dubbo 可扩展机制实战

## Dubbo的扩展机制

在 `Dubbo` 的官网上，`Dubbo` 描述自己是一个高性能的 `RPC` 框架。今天我想聊聊 `Dubbo` 的另一个很棒的特性，就是它的可扩展性。 如同罗马不是一天建成的，任何系统都一定是从小系统不断发展成为大系统的，想要从一开始就把系统设计的足够完善是不可能的；相反的，我们应该关注当下的需求，然后再不断地对系统进行迭代。在代码层面，要求我们适当的对关注点进行抽象和隔离，在软件不断添加功能和特性时，依然能保持良好的结构和可维护性，同时允许第三方开发者对其功能进行扩展。在某些时候，软件设计者对扩展性的追求甚至超过了性能。

在谈到软件设计时，可扩展性一直被谈起，那到底什么才是可扩展性，什么样的框架才算有良好的可扩展性呢？它必须要做到以下两点：

- 作为框架的维护者，在添加一个新功能时，只需要添加一些新代码，而不用大量的修改现有的代码，即符合开闭原则；

- 作为框架的使用者，在添加一个新功能时，不需要去修改框架的源码，在自己的工程中添加代码即可。

`Dubbo` 很好的做到了上面两点。这要得益于 `Dubbo` 的 **微内核** + **插件** 的机制。接下来的章节中我们会慢慢揭开 `Dubbo` 扩展机制的神秘面纱。

## 可扩展的几种解决方案

通常可扩展的实现有下面几种：

- Factory 模式

- IOC 容器

- OSGI 容器

`Dubbo` 作为一个框架，不希望强依赖其他的 `IOC` 容器，比如 `Spring`、`Guice`。`OSGI` 也是一个很重的实现，不适合 `Dubbo`。最终 `Dubbo` 的实现参考了 `Java` 原生的 `SPI` 机制，但对其进行了一些扩展，以满足 `Dubbo` 的需求。

## Dubbo 的 SPI 机制

`Java` `SPI` 的使用很简单。也做到了基本的加载扩展点的功能。但 `Java` `SPI` 有以下的不足：

- 需要遍历所有的实现，并实例化，然后我们在循环中才能找到我们需要的实现；

- 配置文件中只是简单的列出了所有的扩展实现，而没有给他们命名，导致在程序中很难去准确的引用它们；

- 扩展如果依赖其他的扩展，做不到自动注入和装配；

- 不提供类似于 `Spring` 的 `IOC` 和 `AOP` 功能；

- 扩展很难和其他的框架集成，比如扩展里面依赖了一个 `Spring` `bean`，原生的 `Java` `SPI` 不支持。

所以 `Java` `SPI` 应付一些简单的场景是可以的，但对于 `Dubbo`，它的功能还是比较弱的。`Dubbo` 对原生 `SPI` 机制进行了一些扩展，接下来，我们就更深入地了解下 `Dubbo` 的 `SPI` 机制。

## Dubbo 扩展点机制基本概念

在深入学习 `Dubbo` 的扩展机制之前，我们先明确 `Dubbo` `SPI` 中的一些基本概念，在接下来的内容中，我们会多次用到这些术语。

### 扩展点 (Extension Point)

是一个 `Java` 的接口。

### 扩展 (Extension)

扩展点的实现类。

### 扩展实例 (Extension Instance)

扩展点实现类的实例。

### 扩展自适应实例 (Extension Adaptive Instance)

第一次接触这个概念时，可能不太好理解（我第一次也是这样的…）。如果称它为扩展代理类，可能更好理解些。扩展的自适应实例其实就是一个 `Extension` 的代理，它实现了扩展点接口。在调用扩展点的接口方法时，会根据实际的参数来决定要使用哪个扩展。比如一个 `IRepository` 的扩展点，有一个 `save` 方法。有两个实现 `MysqlRepository` 和 `MongoRepository`。`IRepository` 的自适应实例在调用接口方法的时候，会根据 `save` 方法中的参数，来决定要调用哪个 `IRepository` 的实现。如果方法参数中有 `repository=mysql`，那么就调用 `MysqlRepository` 的 `save` 方法。如果 `repository=mongo`，就调用 `MongoRepository` 的 `save` 方法。和面向对象的延迟绑定很类似。为什么 `Dubbo` 会引入扩展自适应实例的概念呢？

- `Dubbo` 中的配置有两种，一种是固定的系统级别的配置，在 `Dubbo` 启动之后就不会再改了。还有一种是运行时的配置，可能对于每一次的 `RPC`，这些配置都不同。比如在 `XML` 文件中配置了超时时间是 `10` 秒钟，这个配置在 `Dubbo` 启动之后，就不会改变了。但针对某一次的 `RPC` 调用，可以设置它的超时时间是 `30` 秒钟，以覆盖系统级别的配置。对于 `Dubbo` 而言，每一次的 `RPC` 调用的参数都是未知的。只有在运行时，根据这些参数才能做出正确的决定。

- 很多时候，我们的类都是一个单例的，比如 `Spring` 的 `bean`，在 `Spring` `bean` 都实例化时，如果它依赖某个扩展点，但是在 `bean` 实例化时，是不知道究竟该使用哪个具体的扩展实现的。这时候就需要一个代理模式了，它实现了扩展点接口，方法内部可以根据运行时参数，动态的选择合适的扩展实现。而这个代理就是自适应实例。自适应扩展实例在 `Dubbo` 中的使用非常广泛，`Dubbo` 中，每一个扩展都会有一个自适应类，如果我们没有提供，`Dubbo` 会使用字节码工具为我们自动生成一个。所以我们基本感觉不到自适应类的存在。后面会有例子说明自适应类是怎么工作的。

### @SPI

`@SPI` 注解作用于扩展点的接口上，表明该接口是一个扩展点。可以被 `Dubbo` 的 `ExtentionLoader` 加载。如果没有此注解 `ExtensionLoader` 调用会异常。

### @Adaptive

`@Adaptive` 注解用在扩展接口的方法上。表示该方法是一个自适应方法。`Dubbo` 在为扩展点生成自适应实例时，如果方法有 `@Adaptive` 注解，会为该方法生成对应的代码。方法内部会根据方法的参数，来决定使用哪个扩展。 `@Adaptive` 注解用在类上代表实现一个装饰类，类似于设计模式中的装饰模式，它主要作用是返回指定类，目前在整个系统中 `AdaptiveCompiler`、 `AdaptiveExtensionFactory` 这两个类拥有该注解。

### ExtentionLoader

类似于 `Java` `SPI` 的 `ServiceLoader`，负责扩展的加载和生命周期维护。

### 扩展别名

和 `Java` `SPI` 不同，`Dubbo` 中的扩展都有一个别名，用于在应用中引用它们。比如

```
random=com.alibaba.dubbo.rpc.cluster.loadbalance.RandomLoadBalance
roundrobin=com.alibaba.dubbo.rpc.cluster.loadbalance.RoundRobinLoadBalance
```

其中的 `random`，`roundrobin` 就是对应扩展的别名。这样我们在配置文件中使用 `random` 或 `roundrobin` 就可以了。

### 一些路径 

和 `Java` `SPI` 从 `/META-INF/services` 目录加载扩展配置类似，`Dubbo` 也会从以下路径去加载扩展配置文件：

- META-INF/dubbo/internal

- META-INF/dubbo

- META-INF/services

## Dubbo 的 LoadBalance 扩展点解读

在了解了 `Dubbo` 的一些基本概念后，让我们一起来看一个 `Dubbo` 中实际的扩展点，对这些概念有一个更直观的认识。

我们选择的是 `Dubbo` 中的 `LoadBalance` 扩展点。`Dubbo` 中的一个服务，通常有多个 `Provider`，`consumer` 调用服务时，需要在多个 `Provider` 中选择一个，这就是一个 `LoadBalance`。我们一起来看看在 `Dubbo` 中，`LoadBalance` 是如何成为一个扩展点的。

### LoadBalance 接口

```java
@SPI(RandomLoadBalance.NAME)
public interface LoadBalance {

    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;
}
```

`LoadBalance` 接口只有一个 `select` 方法。`select` 方法从多个 `invoker` 中选择其中一个。上面代码中和 `Dubbo` `SPI` 相关的元素有：

- `@SPI(RandomLoadBalance.NAME)`：`@SPI` 作用于 `LoadBalance` 接口，表示接口 `LoadBalance` 是一个扩展点。如果没有 `@SPI` 注解，试图去加载扩展时，会抛出异常。`@SPI` 注解有一个参数，该参数表示该扩展点的默认实现的别名。如果没有显示的指定扩展，就使用默认实现。`RandomLoadBalance.NAME` 是一个常量，值是 `random`，是一个随机负载均衡的实现。 `random` 的定义在配置文件 `META-INF/dubbo/internal/com.alibaba.dubbo.rpc.cluster.LoadBalance` 中：

```java
random=com.alibaba.dubbo.rpc.cluster.loadbalance.RandomLoadBalance
roundrobin=com.alibaba.dubbo.rpc.cluster.loadbalance.RoundRobinLoadBalance
leastactive=com.alibaba.dubbo.rpc.cluster.loadbalance.LeastActiveLoadBalance
consistenthash=com.alibaba.dubbo.rpc.cluster.loadbalance.ConsistentHashLoadBalance
```

可以看到文件中定义了 `4` 个 `LoadBalance` 的扩展实现。由于负载均衡的实现不是本次的内容，这里就不过多说明。只用知道 `Dubbo` 提供了 `4` 种负载均衡的实现，我们可以通过 `XML` 文件，`properties` 文件，`JVM` 参数显式的指定一个实现。如果没有，默认使用随机。

<img src="https://gitee.com/dongzl/article-images/raw/master/2021/05-Java-SPI-In-Dubbo/dubbo-loadbalance.png" style="width:800px"/>

- `@Adaptive(“loadbalance”)`：`@Adaptive` 注解修饰 `select` 方法，表明方法 `select` 方法是一个可自适应的方法。`Dubbo` 会自动生成该方法对应的代码，当调用 `select` 方法时，会根据具体的方法参数来决定调用哪个扩展实现的 `select` 方法。`@Adaptive` 注解的参数 `loadbalance` 表示方法参数中的 `loadbalance` 的值作为实际要调用的扩展实例。 但奇怪的是，我们发现 `select` 的方法中并没有 `loadbalance` 参数，那怎么获取 `loadbalance` 的值呢？`select` 方法中还有一个 `URL` 类型的参数，`Dubbo` 就是从 `URL` 中获取 `loadbalance` 的值的。这里涉及到 `Dubbo` 的 `URL` 总线模式，简单说，`URL` 中包含了 `RPC` 调用中的所有参数。`URL` 类中有一个 `Map<String, String> parameters` 字段，`parameters` 中就包含了 `loadbalance`。

### 获取 LoadBalance 扩展

`Dubbo` 中获取 `LoadBalance` 的代码如下：

```
LoadBalance lb = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(loadbalanceName);
```

使用 `ExtensionLoader.getExtensionLoader(LoadBalance.class)` 方法获取一个 `ExtensionLoader` 的实例，然后调用 `getExtension`，传入一个扩展的别名来获取对应的扩展实例。

## 自定义一个 LoadBalance 扩展

本节中，我们通过一个简单的例子，来自己实现一个 `LoadBalance`，并把它集成到 `Dubbo` 中。我会列出一些关键的步骤和代码，也可以从这个[地址](https://github.com/vangoleo/dubbo-spi-demo) 下载完整的 `demo`。

### 实现 LoadBalance 接口 

首先，编写一个自己实现的 `LoadBalance`，因为是为了演示 `Dubbo` 的扩展机制，而不是 `LoadBalance` 的实现，所以这里 `LoadBalance` 的实现非常简单，选择第一个 `invoker`，并在控制台输出一条日志。

```java
package com.dubbo.spi.demo.consumer;

public class DemoLoadBalance implements LoadBalance {
    @Override
    public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException {
        System.out.println("DemoLoadBalance: Select the first invoker...");
        return invokers.get(0);
    }
}
```

### 添加扩展配置文件

添加文件：`META-INF/dubbo/com.alibaba.dubbo.rpc.cluster.LoadBalance`。文件内容如下：

```java
demo=com.dubbo.spi.demo.consumer.DemoLoadBalance
```

### 配置使用自定义 LoadBalance

通过上面的两步，已经添加了一个名字为 `demo` 的 `LoadBalance` 实现，并在配置文件中进行了相应的配置。接下来，需要显式的告诉 `Dubbo` 使用 `demo` 的负载均衡实现。如果是通过 `Spring` 的方式使用 `Dubbo`，可以在xml文件中进行设置。

```xml
<dubbo:reference id="helloService" interface="com.dubbo.spi.demo.api.IHelloService" loadbalance="demo" />
```

在 `consumer` 端的 `dubbo:reference` 中配置 `<loadbalance=“demo”>`

### 启动 Dubbo

启动 `Dubbo`，调用一次 `IHelloService`，可以看到控制台会输出一条 

```
DemoLoadBalance: Select the first invoker...
```

日志，说明 `Dubbo` 的确是使用了我们自定义的 `LoadBalance`。

## 总结

到此，我们从 `Java` `SPI` 开始，了解了 `Dubbo` `SPI` 的基本概念，并结合了 `Dubbo` 中的 `LoadBalance` 加深了理解。最后，我们还实践了一下，创建了一个自定义 `LoadBalance`，并集成到 `Dubbo` 中。相信通过这里理论和实践的结合，大家对 `Dubbo` 的可扩展有更深入的理解。 总结一下，`Dubbo` `SPI` 有以下的特点：

- 对 `Dubbo` 进行扩展，不需要改动 `Dubbo` 的源码；

- 自定义的 `Dubbo` 的扩展点实现，是一个普通的 `Java` 类，`Dubbo` 没有引入任何 `Dubbo` 特有的元素，对代码侵入性几乎为零；

- 将扩展注册到 `Dubbo` 中，只需要在 `ClassPath` 中添加配置文件，使用简单，而且不会对现有代码造成影响，符合开闭原则；

- `Dubbo` 的扩展机制设计默认值：`@SPI(“dubbo”)` 代表默认的 `SPI` 对象；

- `Dubbo` 的扩展机制支持 `IOC`、`AOP` 等高级功能；

- `Dubbo`的扩展机制能很好的支持第三方 `IOC` 容器，默认支持 `Spring` `Bean`，可自己扩展来支持其他容器，比如 `Google` 的 `Guice`；

- 切换扩展点的实现，只需要在配置文件中修改具体的实现，不需要改代码，使用方便。

# Dubbo 可扩展机制源码解析

## ExtensionLoader

`ExtensionLoader` 是最核心的类，负责扩展点的加载和生命周期管理，我们就以这个类开始吧。 `ExtensionLoader` 的方法比较多，比较常用的方法有：

- `public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type)`

- `public T getExtension(String name)`

- `public T getAdaptiveExtension()`

比较常见的用法有：

- `LoadBalance lb = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(loadbalanceName)`

- `RouterFactory routerFactory = ExtensionLoader.getExtensionLoader(RouterFactory.class).getAdaptiveExtension()`

说明：在接下来展示的源码中，我会将无关的代码（比如日志，异常捕获等）去掉，方便大家阅读和理解。

1. `getExtensionLoader` 方法 这是一个静态工厂方法，入参是一个可扩展的接口，返回一个该接口的 `ExtensionLoader` 实体类。通过这个实体类，可以根据 `name` 获得具体的扩展，也可以获得一个自适应扩展。

```java
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    // 扩展点必须是接口
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
    }
    // 必须要有@SPI注解
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type without @SPI Annotation!");
    }
    // 从缓存中根据接口获取对应的ExtensionLoader
    // 每个扩展只会被加载一次
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        // 初始化扩展
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
    
private ExtensionLoader(Class<?> type) {
    this.type = type;
    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```

2. `getExtension` 方法

```java
public T getExtension(String name) {
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    Object instance = holder.get();
    // 从缓存中获取，如果不存在就创建
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

`getExtension` 方法中做了一些判断和缓存，主要的逻辑在 `createExtension` 方法中，我们继续看 `createExtension` 方法。

```java
private T createExtension(String name) {
    // 根据扩展点名称得到扩展类，比如对于LoadBalance，根据random得到RandomLoadBalance类
    Class<?> clazz = getExtensionClasses().get(name);
    
    T instance = (T) EXTENSION_INSTANCES.get(clazz);
    if (instance == null) {
        // 使用反射调用nesInstance来创建扩展类的一个示例
        EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
        instance = (T) EXTENSION_INSTANCES.get(clazz);
    }
    // 对扩展类示例进行依赖注入
    injectExtension(instance);
    // 如果有wrapper，添加wrapper
    Set<Class<?>> wrapperClasses = cachedWrapperClasses;
    if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
        for (Class<?> wrapperClass : wrapperClasses) {
            instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
        }
    }
    return instance;
}
```

`createExtension` 方法做了以下事情：

- 先根据 `name` 来得到对应的扩展类，从 `ClassPath` 下 `META-INF` 文件夹下读取扩展点配置文件；

- 使用反射创建一个扩展类的实例；

- 对扩展类实例的属性进行依赖注入，即 `IOC`；

- 如果有 `wrapper`，添加 `wrapper`，即 `AOP`。

下面我们来重点看下这 `4` 个过程

1. 根据 `name` 获取对应的扩展类 先看代码：

```java
private Map<String, Class<?>> getExtensionClasses() {
    Map<String, Class<?>> classes = cachedClasses.get();
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}

// synchronized in getExtensionClasses
private Map<String, Class<?>> loadExtensionClasses() {
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if (defaultAnnotation != null) {
        String value = defaultAnnotation.value();
        if (value != null && (value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            if (names.length > 1) {
                throw new IllegalStateException("more than 1 default extension name on extension " + type.getName());
            }
            if (names.length == 1) cachedDefaultName = names[0];
        }
    }

    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    loadFile(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
    loadFile(extensionClasses, DUBBO_DIRECTORY);
    loadFile(extensionClasses, SERVICES_DIRECTORY);
    return extensionClasses;
}
```

过程很简单，先从缓存中获取，如果没有，就从配置文件中加载。配置文件的路径就是之前提到的：

- `META-INF/dubbo/internal`

- `META-INF/dubbo`

- `META-INF/services`

2. 使用反射创建扩展实例：这个过程很简单，使用 `clazz.newInstance()` 来完成。创建的扩展实例的属性都是空值。

3. 扩展实例自动装配：在实际的场景中，类之间都是有依赖的。扩展实例中也会引用一些依赖，比如简单的 `Java` 类，另一个 `Dubbo` 的扩展或一个 `Spring` `Bean` 等。依赖的情况很复杂，`Dubbo` 的处理也相对复杂些。我们稍后会有专门的章节对其进行说明，现在，我们只需要知道，`Dubbo` 可以正确的注入扩展点中的普通依赖，`Dubbo` 扩展依赖或 `Spring` 依赖等。

4. 扩展实例自动包装：自动包装就是要实现类似于 `Spring` 的 `AOP` 功能。`Dubbo` 利用它在内部实现一些通用的功能，比如日志，监控等。关于扩展实例自动包装的内容，也会在后面单独讲解。

经过上面的 `4` 步，`Dubbo` 就创建并初始化了一个扩展实例。这个实例的依赖被注入了，也根据需要被包装了。到此为止，这个扩展实例就可以被使用了。

## Dubbo SPI 高级用法之自动装配

自动装配的相关代码在 `injectExtension` 方法中：

```java
private T injectExtension(T instance) {
    for (Method method : instance.getClass().getMethods()) {
        if (method.getName().startsWith("set")
                && method.getParameterTypes().length == 1
                && Modifier.isPublic(method.getModifiers())) {
            Class<?> pt = method.getParameterTypes()[0];
          
            String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
            Object object = objectFactory.getExtension(pt, property);
            if (object != null) {
                method.invoke(instance, object);
            }
        }
    }
    return instance;
}
```

要实现对扩展实例的依赖的自动装配，首先需要知道有哪些依赖，这些依赖的类型是什么。`Dubbo` 的方案是查找 `Java` 标准的 `setter` 方法。即方法名以 `set` 开始，只有一个参数。如果扩展类中有这样的 `set` 方法，`Dubbo` 会对其进行依赖注入，类似于 `Spring` 的 `set` 方法注入。 但是 `Dubbo` 中的依赖注入比 `Spring` 要复杂，因为 `Spring` 注入的都是 `Spring` `bean`，都是由 `Spring` 容器来管理的。而 `Dubbo` 的依赖注入中，需要注入的可能是另一个 `Dubbo` 的扩展，也可能是一个 `Spring` `Bean`，或是 `Google` `guice` 的组件，或其他任何一个框架中的组件。`Dubbo` 需要能够从任何一个场景中加载扩展。在 `injectExtension` 方法中，是用 `Object object = objectFactory.getExtension(pt, property)` 来实现的。`objectFactory` 是 `ExtensionFactory` 类型的，在创建 `ExtensionLoader` 时被初始化：

```java
private ExtensionLoader(Class<?> type) {
    this.type = type;
    objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```

`objectFacory` 本身也是一个扩展，通过 `ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension())` 来获取。

<img src="https://gitee.com/dongzl/article-images/raw/master/2021/05-Java-SPI-In-Dubbo/dubbo-extensionfactory.png" style="width:800px"/>

`ExtensionFactory` 有三个实现：

1. `SpiExtensionFactory`：`Dubbo` 自己的 `SPI` 去加载 `Extension`；

2. `SpringExtensionFactory`：从 `Spring` 容器中去加载 `Extension`；

3. `AdaptiveExtensionFactory`: 自适应的 `AdaptiveExtensionLoader`。

这里要注意 `AdaptiveExtensionFactory`，源码如下：

```java
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {

    private final List<ExtensionFactory> factories;

    public AdaptiveExtensionFactory() {
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    public <T> T getExtension(Class<T> type, String name) {
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }
}
```

`AdaptiveExtensionLoader` 类有 `@Adaptive` 注解。前面提到了，`Dubbo` 会为每一个扩展创建一个自适应实例。如果扩展类上有 `@Adaptive`，会使用该类作为自适应类。如果没有，`Dubbo` 会为我们创建一个。所以 `ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension())` 会返回一个 `AdaptiveExtensionLoader` 实例，作为自适应扩展实例。 `AdaptiveExtensionLoader` 会遍历所有的 `ExtensionFactory` 实现，尝试着去加载扩展。如果找到了，返回。如果没有，在下一个 `ExtensionFactory` 中继续找。`Dubbo` 内置了两个 `ExtensionFactory`，分别从 `Dubbo` 自身的扩展机制和 `Spring` 容器中去寻找。由于 `ExtensionFactory` 本身也是一个扩展点，我们可以实现自己的 `ExtensionFactory`，让 `Dubbo` 的自动装配支持我们自定义的组件。比如，我们在项目中使用了 `Google` 的 `guice` 这个 `IOC` 容器。我们可以实现自己的 `GuiceExtensionFactory`，让 `Dubbo` 支持从 `guice` 容器中加载扩展。

## Dubbo SPI 高级用法之 AOP

在用 `Spring` 的时候，我们经常会用到 `AOP` 功能。在目标类的方法前后插入其他逻辑。比如通常使用 `Spring` `AOP` 来实现日志，监控和鉴权等功能。 `Dubbo` 的扩展机制，是否也支持类似的功能呢？答案是 `yes`。在 `Dubbo` 中，有一种特殊的类，被称为 `Wrapper` 类。通过装饰者模式，使用包装类包装原始的扩展点实例。在原始扩展点实现前后插入其他逻辑，实现 `AOP` 功能。

### 什么是 Wrapper 类

那什么样类的才是 `Dubbo` 扩展机制中的 `Wrapper` 类呢？`Wrapper` 类是一个有复制构造函数的类，也是典型的装饰者模式。下面就是一个 `Wrapper` 类:

```java
class A{
    private A a;
    public A(A a){
        this.a = a;
    }
}
```

类 `A` 有一个构造函数 `public A(A a)`，构造函数的参数是 `A` 本身。这样的类就可以成为 `Dubbo` 扩展机制中的一个 `Wrapper` 类。`Dubbo` 中这样的 `Wrapper` 类有 `ProtocolFilterWrapper`, `ProtocolListenerWrapper` 等, 大家可以查看源码加深理解。

### 怎么配置 Wrapper 类

在 `Dubbo` 中 `Wrapper` 类也是一个扩展点，和其他的扩展点一样，也是在 `META-INF` 文件夹中配置的。比如前面举例的 `ProtocolFilterWrapper` 和 `ProtocolListenerWrapper` 就是在路径 `dubbo-rpc/dubbo-rpc-api/src/main/resources/META-INF/dubbo/internal/org.apache.dubbo.rpc.Protocol` 中配置的：

```
filter=org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper
listener=org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper
mock=org.apache.dubbo.rpc.support.MockProtocol
```

在 `Dubbo` 加载扩展配置文件时，有一段如下的代码：

```java
try {  
    clazz.getConstructor(type);    
    Set<Class<?>> wrappers = cachedWrapperClasses;
    if (wrappers == null) {
        cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
        wrappers = cachedWrapperClasses;
    }
    wrappers.add(clazz);
} catch (NoSuchMethodException e) {
    
}
```

这段代码的意思是，如果扩展类有复制构造函数，就把该类存起来，供以后使用。有复制构造函数的类就是 `Wrapper` 类。通过 `clazz.getConstructor(type)` 来获取参数是扩展点接口的构造函数。注意构造函数的参数类型是扩展点接口，而不是扩展类。 以 `Protocol` 为例。配置文件 `dubbo-rpc/dubbo-rpc-api/src/main/resources/META-INF/dubbo/internal/org.apache.dubbo.rpc.Protocol` 中定义了 `filter=org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper`。 `ProtocolFilterWrapper`代码如下：

```java
public class ProtocolFilterWrapper implements Protocol {

    private final Protocol protocol;

    // 有一个参数是Protocol的复制构造函数
    public ProtocolFilterWrapper(Protocol protocol) {
        if (protocol == null) {
            throw new IllegalArgumentException("protocol == null");
        }
        this.protocol = protocol;
    }
}
```

`ProtocolFilterWrapper` 有一个构造函数 `public ProtocolFilterWrapper(Protocol protocol)`，参数是扩展点 `Protocol`，所以它是一个 `Dubbo` 扩展机制中的 `Wrapper` 类。`ExtensionLoader` 会把它缓存起来，供以后创建 `Extension` 实例的时候，使用这些包装类依次包装原始扩展点。

## 扩展点自适应

前面讲到过，`Dubbo` 需要在运行时根据方法参数来决定该使用哪个扩展，所以有了扩展点自适应实例。其实是一个扩展点的代理，将扩展的选择从 `Dubbo` 启动时，延迟到 `RPC` 调用时。`Dubbo` 中每一个扩展点都有一个自适应类，如果没有显式提供，`Dubbo` 会自动为我们创建一个，默认使用 `Javaassist`。 先来看下创建自适应扩展类的代码：

```java
public T getAdaptiveExtension() {
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                      instance = createAdaptiveExtension();
                      cachedAdaptiveInstance.set(instance); 
                }
            }        
    }

    return (T) instance;
}
```

继续看 `createAdaptiveExtension` 方法

```java
private T createAdaptiveExtension() {        
    return injectExtension((T) getAdaptiveExtensionClass().newInstance());
}
```

继续看 `getAdaptiveExtensionClass` 方法

```java
private Class<?> getAdaptiveExtensionClass() {
    getExtensionClasses();
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

继续看 `createAdaptiveExtensionClass` 方法，绕了一大圈，终于来到了具体的实现了。看这个 `createAdaptiveExtensionClass` 方法，它首先会生成自适应类的 `Java` 源码，然后再将源码编译成 `Java` 的字节码，加载到 `JVM` 中。


```java
private Class<?> createAdaptiveExtensionClass() {
    String code = createAdaptiveExtensionClassCode();
    ClassLoader classLoader = findClassLoader();
    org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    return compiler.compile(code, classLoader);
}
```

`Compiler` 的代码，默认实现是 `javassist`。

```java
@SPI("javassist")
public interface Compiler {
    Class<?> compile(String code, ClassLoader classLoader);
}
```

`createAdaptiveExtensionClassCode()` 方法中使用一个 `StringBuilder` 来构建自适应类的 `Java` 源码。方法实现比较长，这里就不贴代码了。这种生成字节码的方式也挺有意思的，先生成 `Java` 源代码，然后编译，加载到 `JVM` 中。通过这种方式，可以更好的控制生成的 `Java` 类。而且这样也不用 `care` 各个字节码生成框架的 `api` 等。因为 `xxx.java` 文件是 `Java` 通用的，也是我们最熟悉的。只是代码的可读性不强，需要一点一点构建 `xx.java` 的内容。 下面是使用 `createAdaptiveExtensionClassCode` 方法为 `Protocol` 创建的自适应类的Java代码范例：

```java
package org.apache.dubbo.rpc;

import org.apache.dubbo.common.extension.ExtensionLoader;

public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {
    public void destroy() {
        throw new UnsupportedOperationException("method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException("method public abstract int org.apache.dubbo.rpc.Protocol.getDefaultPort() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }

    public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0, org.apache.dubbo.common.URL arg1) throws org.apache.dubbo.rpc.RpcException {
        if (arg1 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }
}
```

大致的逻辑和开始说的一样，通过 `url` 解析出参数，解析的逻辑由 `@Adaptive` 的 `value` 参数控制，然后再根据得到的扩展点名获取扩展点实现，然后进行调用。如果大家想知道具体的构建 `.java` 代码的逻辑，可以看 `createAdaptiveExtensionClassCode` 的完整实现。 在生成的 `Protocol$Adaptive` 中，发现 `getDefaultPort` 和 `destroy` 方法都是直接抛出异常的，这是为什么呢？来看看 `Protocol` 的源码：

```java
@SPI("dubbo")
public interface Protocol {

    int getDefaultPort();

    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    void destroy();
}
```

可以看到 `Protocol` 接口中有 `4` 个方法，但只有 `export` 和 `refer` 两个方法使用了 `@Adaptive` 注解。`Dubbo` 自动生成的自适应实例，只有 `@Adaptive` 修饰的方法才有具体的实现。所以，`Protocol$Adaptive` 类中，也只有 `export` 和 `refer` 这两个方法有具体的实现，其余方法都是抛出异常。
