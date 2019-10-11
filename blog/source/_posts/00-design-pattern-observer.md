---
title: 从一个功能设计聊聊观察者模式的使用
date: 2019-10-11 11:06:46
categories:
- design-pattern
tags:
- 设计模式
- 观察者模式
- Observer
- Observable
---

## 背景描述
对接公司电商商城系统，接收订单完成 MQ 消息内容，需要根据订单完成数据实现自己业务部门的一些功能，例如：用户下单满多少金额发放某个礼品；过滤订单中有某个 SKU 的商品需要给用户免费绑定虚拟会员服务等等一些功能。

**PS.一句话需求，根据订单 MQ 消息，执行一些特殊业务逻辑处理，业务场景比较杂，逻辑分支比较多，而且随时会新增、撤销某些业务功能。**

## 目前实现
### 老系统实现分析
这个类似的功能在老的系统是存在，以前系统中直接在一个 class 的方法中处理所有的逻辑，只是每个业务场景单独一个 Service，在主类中依赖所有的 Service，然后开始顺序执行所有的逻辑，
```java
public class TestObserver {
    
    private AService aService;
    
    private BService bService;
    
    private CService cService;
    
    public void handleMessage(String message) {
        // 处理A场景业务
        try {
            aService.dealBusiness(message);
        } catch (Exception e) {
            log.error("Deal A Business exception -> ", e);
        }
        // 处理B场景业务
        try {
            bService.dealBusiness(message);
        } catch (Exception e) {
            log.error("Deal B Business exception -> ", e);
        }
        // 处理C场景业务
        try {
            cService.dealBusiness(message);
        } catch (Exception e) {
            log.error("Deal C Business exception -> ", e);
        }
    }
}
```
从代码实现角度来说，其实是整齐划一的，所以虽然这个代码几千行，但是改起来没难度，不过这种实现方式不能算是优秀的：
- 首先接收到一个消息，顺序执行，前后执行是阻塞的，A 没执行完，B、C 都会等待，万一某个逻辑处理很慢，后面逻辑都会受影响；
- 大段代码不好维护的，从面向对象的五个基本原则之 `开闭原则` 来说，类要对扩展开放，对修改关闭，这个类明显不符合。

### 新系统实现分析
新系统开发之后，对这个功能做了优化，目标是一定要改变这种在一个 class 中执行所有业务逻辑的实现，至少满足 `开闭原则`。
对于这种等待接收 MQ 处理各种业务逻辑的场景，直接想到了使用`观察者模式（Observer）`尝试一下：
- 各种业务逻辑类注册为 Observer（观察者）；
- 接收 MQ 消息的主类为 Observable（被观察者）。

各个业务逻辑类注册监听主类，主类在接收到 MQ 消息后通知所有的业务逻辑类，每一种业务都可以注册成为一个单独的 Observer，新增一种业务时就新增一个 Observer。

如果我们这么分析，好像的确是满足观察者模式的要求的，而且还可以满足 `开闭原则`，不知道这么分析是不是很牵强。

JDK + Spring 实现的观察者模式：

JDK 自带观察者模式实现：
```java

```
使用 Spring 配置文件方式，实现 Observable 和 Observer 依赖关系解耦：
```xml
<!-- 订单完成监听-被观察者 -->
<bean id="orderCompleteMqObservable" class="com.abc.mq.order.observe.OrderCompleteMqObservable"/>
<!-- 订单完成监听-观察者 -->
<bean id="aServiceMqObserver" class="com.abc.mq.order.observe.AServiceMqObserver"/>
<bean id="bServiceMqObserver" class="com.abc.mq.order.observe.BServiceMqObserver"/>
<bean id="cServiceMqObserver" class="com.abc.mq.order.observe.CServiceMqObserver"/>
<!--反射方法调用-->
<bean class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
    <property name="targetObject" ref="orderCompleteMqObservable"/>
    <property name="targetMethod" value="addObservers"/>
    <property name="arguments">
        <list>
            <ref bean="aServiceMqObserver"/>
            <ref bean="bServiceMqObserver"/>
            <ref bean="cServiceMqObserver"/>
        </list>
    </property>
</bean>
```
