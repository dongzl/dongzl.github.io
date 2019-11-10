---
title: 从一个功能设计聊聊策略模式 + 工厂模式的使用
date: 2019-11-06 16:25:58
categories:
- design-pattern
tags:
- 设计模式
- 策略模式
---

## 背景描述
线上业务销售一种 VIP 虚拟卡，对于已售出的 VIP 虚拟卡在给供应商结算时需要根据销售渠道做不同区分，例如，Android 端销售的虚拟卡按实际销售金额结算；iOS端销售的虚拟卡按销售价 70% 结算（做苹果开发的都知道，苹果公司雁过拔毛，要拔走 30%）；还有就是可能与其他商品捆绑销售，捆绑销售的金额根据合作不同部门、不同商品金额都不同。目前情况即是如此，虚拟卡销售渠道很多，不同渠道结算金额不同，而且还会扩展（最近就在经历扩展，华为属于 Android 渠道，但是也开始玩苹果那套规则，拔 30% 的毛）。

**PS. 一句话需求，销售商品结算，不同销售渠道销售金额不同，而且随时会扩展新的渠道。**

## 代码实现
如果设计模式玩溜了，对于这种业务场景，很容易想到使用策略（Strategy）模式来完成功能，所以这里也直接上代码实现：

### 策略接口定义
```java
public interface CalculateStrategy {
    /**
     * 计算金额
     * @param param 参数
     * @return 计算结果
     * @throws Exception
     */
    public Integer calculate(StrategyParam param) throws Exception;
}
```
### 策略实现类
```java
// Android 渠道结算策略
public class AndroidVipOrderStrategy implements CalculateStrategy {

    @Override
    public Integer calculate(StrategyParam param) throws Exception {
        Order order = queryOrder(param);
        return order.getOnlineAmount;
    }
}

// iOS 渠道结算策略
public class IOSVipOrderStrategy implements CalculateStrategy {

    @Override
    public Integer calculate(StrategyParam param) throws Exception {
        Order order = queryOrder(param);
        return order.getOnlineAmount * 0.7;
    }
}

// 捆绑组合渠道结算策略
public class BindVipOrderStrategy implements CalculateStrategy {

    @Override
    public Integer calculate(StrategyParam param) throws Exception {
        return 199;
    }
}
```

### 策略Context类
```java

public class StrategyContext {

    private StrategyParam param;

    public StrategyFactory(StrategyParam param) {
        this.param = param;
    }

    public Integer calculate() {
        CalculateStrategy result = getStrategy();
        if (result == null) {
            return 0;
        }
        return result.calculate(param);
    }

    public CalculateStrategy getStrategy() {
        CalculateStrategy result = null;
        SourceTypeEnum type = SourceTypeEnum.valueOf(param.getVipSourceType());
        switch (type) {
            case ANDROID:
                result = new AndroidVipOrderStrategy();
                break;
            case IOS:
                result = new IOSVipOrderStrategy();
                break;
            case BIND:
                result = new BindVipOrderStrategy();
                break;
            default:
                throw new UnsupportedOperationException();
        }
        return result;
    }
}
```
### 客户端调用
```java
public class Client {
    public static void main(String args[]) throws Exception {
        StrategyParam param = new StrategyParam();
        System.out.println(new StrategyContext(param).calculate());
    }
}
```

### 类图描述
<img src="https://raw.githubusercontent.com/dongzl/dongzl.github.io/hexo/blog/source/images/Design_Pattern_Strategy_01.png" width="600px">

## 观察者模式的四要素
### 定义与动机
**定义：** 

策略模式（Strategy Pattern）：定义**一系列算法**，将**每一个算法封装起来**，并让他们可以**相互替换**。策略模式**让算法独立于使用它的客户而变化**，也称为**政策模式**（Policy）。策略模式是一种**对象行为型**模式。

Strategy Pattern: Define **a family of algorithms**, **encapsulate each one**, and **make them interchangeable**. Strategy lets the algorithm vary independently from clients that use it.

**动机：**

完成一项任务，往往可以有很多种不同的方式，<font color="red">每一种方式称为一个策略</font>，我们<font color="red">可以根据环境或者条件的不同选择不同的策略来完成任务</font>。

为了解决这些问题，<font color="red">可以定义一些独立的类来封装不同的算法，每一个类封装一个具体的算法</font>，在这里，<font color="red">每一个封装算法的类我们都可以称之为策略(Strategy)</font>，为了保证这些策略的一致性，一般会用一个<font color="red">抽象的策略类来做算法的定义，而具体每种算法则对应于一个具体策略类</font>。

### 结构与分析
**模式结构：**

类图：



### 优点 & 缺点
**优点：**

**缺点：**

### 应用场景

## 写在最后

## 参考资料