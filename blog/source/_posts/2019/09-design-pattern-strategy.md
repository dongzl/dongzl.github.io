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