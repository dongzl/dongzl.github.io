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

<!-- more -->

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
// 订单完成 MQ 被观察者 Observable
public class OrderCompleteMqObservable extends Observable {
    /**
     * 注册监听器
     * @param observers
     */
    public void addObservers(List<Observer> observers) {
        if (CollectionUtils.isEmpty(observers)) {
            throw new RuntimeException("observers is not null");
        }
        for (Observer observer : observers) {
            this.addObserver(observer);
        }
    }
    /**
     * 处理业务逻辑
     * @param message MQ消息
     */
    public void dealBusiness(String message) {
        setChanged();
        notifyObservers(message);
    }
}

// 订单完成观察者 Observer
public class AServiceMqObserver implements Observer {

    @Override
    public void update(Observable o, Object arg) {
        String message = (String) arg;
        // Deal A Service Business
    }
}

public class BServiceMqObserver implements Observer {

    @Override
    public void update(Observable o, Object arg) {
        String message = (String) arg;
        // Deal B Service Business
    }
}
```
使用 Spring 配置文件方式，实现 Observable 和 Observer 依赖关系解耦：
```xml
<!-- 订单完成监听-被观察者 -->
<bean id="orderCompleteMqObservable" class="com.abc.mq.order.observe.OrderCompleteMqObservable"/>
<!-- 订单完成监听-观察者 -->
<bean id="aServiceMqObserver" class="com.abc.mq.order.observe.AServiceMqObserver"/>
<bean id="bServiceMqObserver" class="com.abc.mq.order.observe.BServiceMqObserver"/>
<!--反射方法调用-->
<bean class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
    <property name="targetObject" ref="orderCompleteMqObservable"/>
    <property name="targetMethod" value="addObservers"/>
    <property name="arguments">
        <list>
            <ref bean="aServiceMqObserver"/>
            <ref bean="bServiceMqObserver"/>
        </list>
    </property>
</bean>
```
## 观察者模式的四要素
### 定义与动机
**定义：** 

观察者模式（Observer Pattern）：定义对象间的一种`一对多依赖关系`，使得每当`一个对象状态发生改变`时，其相关`依赖对象皆得到通知并被自动更新`。(Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.)

观察者模式又叫做`发布-订阅（Publish/Subscribe）`模式、`模型-视图（Model/View）`模式、`源-监听器（Source/Listener）`模式或`从属者（Dependents）`模式。观察者模式是一种`对象行为型`模式。

**动机：**

建立一种<font color=#FF0000>对象与对象之间的依赖关系</font>，一个对象发生改变时将自动通知其他对象，其他对象将相应做出反应。在此，发生改变的对象称为<font color=#FF0000>观察目标</font>，而被通知的对象称为<font color=#FF0000>观察者</font>，<font color=#FF0000>一个观察目标可以对应多个观察者</font>，而且这些观察者之间没有相互联系，可以根据需要<font color=#FF0000>增加和删除</font>观察者，使得系统更易于扩展，这就是观察者模式的模式动机。

### 结构与分析
**模式结构：**

类图：

<img src="https://raw.githubusercontent.com/dongzl/dongzl.github.io/hexo/blog/source/images/Design_Pattern_Observer_01.png" width="600px">

角色：
- Subject：抽象主题（被观察者）角色把所有对观察者对象的引用保存在一个集合（比如 ArrayList 集合）里，每个主题都可以有任何数量的观察者，抽象主题提供一个接口，可以增加和删除观察者对象，抽象主题角色又叫做抽象被观察者（Observable）角色；
- ConcreteSubject：具体主题（具体被观察者）将有关状态存入具体观察者对象，在具体主题的内部状态改变时，给所有登记过的观察者发出通知，具体主题角色又叫做具体被观察者（Concrete Observable）角色；
- Observer：抽象观察者，为所有的具体观察者定义一个接口，在得到主题的通知时更新自己，这个接口叫做更新接口；
- ConcreteObserver：具体观察者，存储与主题的状态匹配的状态，具体观察者角色实现抽象观察者角色所要求的更新接口，以便使本身的状态与主题的状态协调，如果需要，具体观察者角色可以保持一个指向具体主题对象的引用。

JDK 对观察者模式的扩展：

<img src="https://raw.githubusercontent.com/dongzl/dongzl.github.io/hexo/blog/source/images/Design_Pattern_Observer_03.png" width="600px">

**时序图：**

<img src="https://raw.githubusercontent.com/dongzl/dongzl.github.io/hexo/blog/source/images/Design_Pattern_Observer_02.png" width="600px">

### 优点 & 缺点
**优点：**
- 表示层和数据逻辑层的分离；
- 观察者和被观察者之间建立了一个抽象的耦合；
- 支持广播通信；
- 符合“开闭原则”。

**缺点：**
- 如果一个观察目标对象有很多直接和间接的观察者的话，将所有的观察者都通知到会花费很多时间；
- 如果在观察者和观察目标之间有循环依赖的话，观察目标会触发它们之间进行循环调用，可能导致系统崩溃；
- 观察者模式没有相应的机制让观察者知道所观察的目标对象是怎么发生变化的，而仅仅只是知道观察目标发生了变化（<font color=#FF0000>对于 JDK 中自带的观察者模式的实现，应该没有这个问题，观察者可以知道被观察的目标发生的变化</font>）。

### 应用场景

- 一个抽象模型有两个方面，其中**一个方面依赖于另一个方面**，将这些方面**封装在独立的对象中使它们可以各自独立地改变和复用**；
- **一个对象的改变将导致其他一个或多个对象也发生改变**，而不知道**具体有多少对象将发生改变**，可以降低对象之间的耦合度；
- **一个对象必须通知其他对象，而并不知道这些对象是谁**；
- 需要在系统中创建一个触发链，A对象的行为将影响B对象，B对象的行为将影响C对象……，可以使用观察者模式创建一种**链式触发机制**。

## 观察者模式 & 推拉模型

## 参考资料
- [观察者模式 vs 发布订阅模式](https://zhuanlan.zhihu.com/p/51357583)
- [Java设计模式の观察者模式（推拉模型）](https://www.cnblogs.com/KongkOngL/p/6849859.html)