---
title: Google Guava EventBus 在 ShardingShere 中的应用
date: 2020-02-01 18:02:52
cover: https://gitee.com/dongzl/article-images/raw/master/cover/google_guava.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: Guava 的 EventBus 可以简化生产/消费模型。EventBus 通过非常简单的方式，实现了观察者模式中的监听注册，事件分发。

categories: 
  - web开发

tags: 
  - Google
  - Guava
  - EventBus
---
## Guava EventBus 介绍

Guava 的 EventBus 可以简化生产/消费模型。EventBus 通过非常简单的方式，实现了观察者模式中的监听注册，事件分发。

> Dispatches events to listeners, and provides ways for listeners to register themselves. The EventBus allows publish-subscribe-style communication between components without requiring the components to explicitly register with one another (and thus be aware of each other). It is designed exclusively to replace traditional Java in-processs event distribution using explicit registration. It is not a general-purpose publish-subscribe system, nor is it intended for interprocess communication.

> 将事件分发给监听器，同时提供监听器注册监听的方式。EventBus 支持组件之间的发布-订阅式通信，而不需要组件显式地彼此注册（从而彼此感知对方存在）。它是专门为使用显式注册来替代传统的 Java 进程内事件分发而设计的。它不是一个通用的发布订阅系统，也不支持进程间通信。

<!-- more -->

<img src="https://gitee.com/dongzl/article-images/raw/master/2020/01-Google-Guava-EventBus-ShardingSphere/Google-Guava-EventBus.png" width="600px">

通过阅读 Guava 的 EventBus 源码，EventBus 支持的操作如下：

- Receiving Events（接收事件）
- Posting Events （发布事件）
- Subscriber Methods （订阅事件）
- Dead Events （没有订阅者的事件）

## EventBus 发布/订阅使用

```java
// 订阅者
import com.google.common.eventbus.Subscribe;

public class EventHandler {
    
    @Subscribe
    public void mq(MQEvent mq) {
        System.out.println(mq.getClass().getCanonicalName() + " work");
    }
}

// 订阅事件
public class MQEvent {
    
}

// 测试类
import com.google.common.eventbus.EventBus;

public class EventTest {
    
    public static void main(String[] args) {
        //初始化消息总线
        EventBus eventBus = new EventBus();
        // 注册订阅者
        eventBus.register(new EventHandler());
        //MqEvent推送给订阅者
        MQEvent mqEvent = new MQEvent();
        //发布消息
        eventBus.post(mqEvent);
    }
}
```

## EventBus 在 ShardingSphere 中的实际应用

在 [ShardingSphere](https://github.com/apache/incubator-shardingsphere) 项目中需要将数据库配置信息存储到统一配置中心（例如：ZooKeeper、nacos、Apollo），对于统一的配置中心可以通过监听机制，监听配置中心配置信息的变化，将配置中心变更的信息推送给 ShardingSphere，对于这种情况就是一个典型的发布-订阅模型，在 ShardingSphere 就是通过 EventBus 来完成这个功能的，我们来看一下 ShardingSphere 的代码实现：

- 系统启动时，通过 ShardingOrchestrationFacade.init() 方法注册监听内容，ConfigurationChangedListenerManager.initListeners() 方法用于启动对于系统配置的监听。
```java
public final class ConfigurationChangedListenerManager {
    
    ... ...

    /**
     * Initialize all configuration changed listeners.
     */
    public void initListeners() {
        schemaChangedListener.watch(ChangedType.UPDATED, ChangedType.DELETED);
        propertiesChangedListener.watch(ChangedType.UPDATED);
        authenticationChangedListener.watch(ChangedType.UPDATED);
    }
}
```

- 例如在 PropertiesChangedListener.watch() 方法中，通过调用父类的 PostShardingConfigCenterEventListener.watch(final ChangedType... watchedChangedTypes) 方法完成监听，在 PostShardingConfigCenterEventListener 方法中使用了 EventBus，注册监听配置中心中某个 Key 的变化：
```java
private void watch(final String watchKey, final Collection<ChangedType> watchedChangedTypeList) {
    configCenter.watch(watchKey, new DataChangedEventListener() {
    
        @Override
        public void onChange(final DataChangedEvent dataChangedEvent) {
            if (watchedChangedTypeList.contains(dataChangedEvent.getChangedType())) {
                eventBus.post(createShardingOrchestrationEvent(dataChangedEvent));
            }
        }
    });
}
```

- 对于配置中心监听配置信息的变化都是由不同的框架（ZooKeeper、nacos、Apollo）来完成的，下面以 nacos 为例，看一下监听实现，nacos 中配置信息发生变化后对通过 receiveConfigInfo 方法推送给 ShardingShpere，ShardingShpere 接收到变更后通过 EventBus 的 post 方法发送变更事件，订阅事件的类接收到变更后会进行相应逻辑处理：
```java
public void watch(final String key, final DataChangedEventListener dataChangedEventListener) {
    try {
        String dataId = key.replace("/", ".");
        String group = properties.getProperty("group", "SHARDING_SPHERE_DEFAULT_GROUP");
        configService.addListener(dataId, group, new Listener() {
            
            @Override
            public Executor getExecutor() {
                return null;
            }
            
            @Override
            public void receiveConfigInfo(final String configInfo) {
                dataChangedEventListener.onChange(new DataChangedEvent(key, configInfo, DataChangedEvent.ChangedType.UPDATED));
            }
        });
    } catch (final NacosException ex) {
        log.debug("Nacos watch key exception for: {}", ex.toString());
    }
}
```

- ShardingSphere 中事件订阅，例如，在 ShardingProxyContext 需要监听配置信息变化，在构造方法中将当前实例对象（this）注册到 EventBus，通过 @Subscribe 注解监听配置变更，监听到配置变更的数据后发送给 ShardingShere，处理内部相关逻辑：
```java
public final class ShardingProxyContext {
    
    private static final ShardingProxyContext INSTANCE = new ShardingProxyContext();
    
    private ShardingProperties shardingProperties = new ShardingProperties(new Properties());
    
    private ShardingProxyContext() {
        ShardingOrchestrationEventBus.getInstance().register(this);
    }
    
    public static ShardingProxyContext getInstance() {
        return INSTANCE;
    }
    
    /**
     * Renew properties.
     * 监听事件的变化
     * @param event properties changed event
     */
    @Subscribe
    public synchronized void renew(final PropertiesChangedEvent event) {
        ConfigurationLogger.log(event.getProps());
        shardingProperties = new ShardingProperties(event.getProps());
    }
}
```

## DeadEvent 使用场景

> Wraps an event that was posted, but which had no subscribers and thus could not be delivered.Registering a DeadEvent subscriber is useful for debugging or logging, as it can detect misconfigurations in a system's event distribution.

> 包装了一个被发送的事件，但是这个事件却没有任何订阅者，因此这个事件可能不会被实际发送。注册一个 DeadEvent 事件订阅器对于调试或日志记录很有用，因为它可以检测事件分布系统中的错误配置。

```java
// 定义事件，没有订阅者
public class MQEvent {
    
}

// DeadEvent 处理器
import com.google.common.eventbus.DeadEvent;
import com.google.common.eventbus.Subscribe;

public class DeadEventHandler {
    
    /**
     * 没有订阅者时被触发
     */
    @Subscribe
    public void deadEvent(DeadEvent event){
        System.out.println("Receive a DeadEvent");
    }
}

// 测试类
import com.google.common.eventbus.EventBus;

public class DeadEventTest {
    
    public static void main(String[] args) {
        // 初始化消息总线
        EventBus eventBus = new EventBus();
        // 注册 DeadEvent 订阅者
        eventBus.register(new DeadEventHandler());
        // MqEvent推送给订阅者
        MQEvent mqEvent = new MQEvent();
        // 发布消息，没有订阅者
        eventBus.post(mqEvent);
    }
}
```

## 其他类使用

- AsyncEventBus：异步事件总线，当处理耗时的处理时很有用，我们要依赖Executors来实现异步事件总线。
```java
AsyncEventBus asyncEventBus = new AsyncEventBus(executorService);
```

- AllowConcurrentEvents：在设置观察者时，需要使用注解类@Subscribe来标识一个订阅者，但在注解中还要一个注解@AllowConcurrentEvents，这个注解是用来标识当前订阅者是线程安全的。

```java
/** 
 * Creates a {@code Subscriber} for {@code method} on {@code listener}. 
 */
static Subscriber create(EventBus bus, Object listener, Method method) {
    return isDeclaredThreadSafe(method)
        ? new Subscriber(bus, listener, method)
        : new SynchronizedSubscriber(bus, listener, method);
}
```

## 参考资料

- [Guava - EventBus(事件总线)](http://greengerong.github.io/blog/2014/11/27/guava-eventbus/)
- [Guava eventBus 关于@AllowConcurrentEvents 纪实](https://www.jianshu.com/p/a950d7c294e5)
- [用guava实现简单的事件驱动](https://www.iteye.com/blog/uule-2096279)
- [Guava EventBus](https://www.jianshu.com/p/703fa6cf6e44)
- [走近Guava(六): 事件总线EventBus](https://www.yeetrack.com/?p=1177)