---
title: cread 系统畅读卡同步 action 使用 Spring 单例作用域引起的问题
date: 2019-12-10 20:38:18
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/java_study.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨在学习和总结 Lombok 框架的使用。

categories: 
  - web开发

tags: 
  - Singleton
---

## 接口功能描述

1. 在 `cread` 系统中提供了如下 `action` 功能类：

```java
com.jd.mobile.read.web.action.card.SynchroReadCardAction
```

2. 该类的主要应用场景是通过调用该类的 `action` 对应的方法进行畅读卡绑定操作，这个类目中目前只有一个 `execute` 方法，通过如下的url调用，执行该类中的 `execute` 方法，完成对应场景的畅读卡绑定操作：

```
http://cread.e.jd.com/syncReadCard.action?orderId=1234&skuId=1234&pin=test&cardType=5&uuid=xxxxxxx&paramKey=xxxxx
```

在 `SynchroReadCardAction` 类中获取参数的方式是通过定义全局变量，通过 `Struts2` 的 `get` 和 `set` 方法自动完成参数的获取。

3. 目前 `SynchroReadCardAction` 在 `cread` 系统中的配置方式如下：

首先在 `spring-config-struts.xml` 配置文件中对该类进行了配置，将该类的对象托管给 `Spring` 进行管理：

```xml
<!-- http同步畅读卡的action，注入访问key-->
<bean class="com.jd.mobile.read.web.action.card.SynchroReadCardAction">
    <property name="sychroReadCardKey" value="${mobile-cread.sync.httpinterface.key}"/>
</bean>
```

在 `struts-read.xml` 配置文件中映射关系的配置如下：

```xml
<package name="syReadcard" extends="action-json-default">
    <action name="syncReadCard" class="com.jd.mobile.read.web.action.card.SynchroReadCardAction">
        <result type="json" name="success">
            <param name="excludeNullProperties">true</param>
            <param name="enableGZIP">false</param>
            <param name="root">resultMap</param>
            <param name="defaultEncoding">utf-8</param>
        </result>
    </action>
</package>
```

## 应用场景

```
http://cread.e.jd.com/syncReadCard.action?orderId=1234&skuId=1234&pin=test&cardType=5&uuid=xxxxxxx&paramKey=xxxxx
```

上述 `url` 目前调用情况主要是有几个场景：

1. 用户下单直接购买畅读卡，在 `order` 系统生成订单信息之后会调用该 `url` 进行畅读卡绑定：

```java
//直接生成畅读或畅听卡
NameValuePair orderid = new NameValuePair("orderId", String.valueOf(mallOrderDetailDO.getOrderId()));
NameValuePair pin = new NameValuePair("pin", mallOrderDetailDO.getPin());
NameValuePair skuid = new NameValuePair("skuId", mallOrderDetailDO.getItemId() + "");
NameValuePair apptype = new NameValuePair("appType", appTypeEnum.getCode() + "");
NameValuePair uuid = new NameValuePair("uuid", mallOrderDetailDO.getOrderId() + "_" + mallOrderDetailDO.getId());
NameValuePair key = new NameValuePair("paramKey", syncCardParamKey);
NameValuePair[] nameValuePairs = new NameValuePair[]{orderid, pin, skuid, apptype, uuid, key};
String returnJson = HttpClientUtil.sendHttpRequestByParams(syncUrl, nameValuePairs, 3000, "GBK");
```

2. 用户购买会员PLUS之后，`orderman` 系统会收到 `mq` 订单消息，判断用户购买了会员PLUS会直接给用户绑定畅读卡年卡。

3. 用户购买纸书送畅读卡，会调用该 `url` 完成畅读卡绑定。

4. 其他一些活动场景，比如预约华为手机送畅读卡，预约小米手机送畅读卡等场景。

上述 `2`、`3`、`4` 等场景下的调用方式都是通过调用 `order` 系统的 `BookUserSendMsgServiceImpl` 类中的 `createCard` 方法来完成的，当然该方法中的实现原理还是调用上述 `url` 向 `cread` 系统发送 `http` 请求完成畅读卡绑定的。

## 问题现象及问题原因排查

### 问题现象

目前在做畅读卡结算中发现结算金额出现偏差，通过排查数据发现，有很多用户下单购买的畅读卡没有参与到结算中，原因是因为数据不符合结算条件，参与结算的畅读卡是会根据畅读卡类型进行过滤的，其中只有以下三类畅读卡会进行结算：1、个人购卡；2、企业购卡；4包月卡。也就是 `cardType` 为 `1`、`2`、`4` 的畅读卡会参与结算，目前查询数据库数据发现有很多畅读卡本身是用户购买的畅读卡，`cardType` 类型应该是 `1`，但是数据库中记录的信息是`22`（购买PLUS会员赠送畅读卡），排查 `order` 系统 `http` 请求接口的方式没有发现 `cardType` 类型设置错误的地方。

### 问题定位

后来通过反复查看 `order` 系统和 `cread` 系统中的实现方式，发现了一个问题，在用户下单购买畅读卡时，`order` 系统在 `http` 请求调用 `cread` 系统 `url` 时是没有指定 `cardType` 类型这个参数的，在 `cread` 系统中进行了判断，如果 `cardType` 参数为空，则指定 `cardType` 值为 `1`。而其他的几种调用，比如：购买会员PLUS绑定畅读卡、预约小米手机送畅读卡，在 `http` 请求中显示的指定了 `cardType` 参数的值，这个地方是一个明显的区别。
    
接下来再查看 `SynchroReadCardAction` 类 `spring-config-struts.xml` 的配置方式时发现了问题，这个类在托管给 `Spring` 时 `scope` 属性采用默认值，也就是 `singleton`（单例模式），如果采用这种方式，在系统全局中 `SynchroReadCardAction` 是个单例类，而通过定义全局变量，通过 `Struts2` 的 `get` 和 `set` 方法获取 `http` 请求中的参数时会出现参数不一致的问题，比如：通过会员PLUS绑定畅读卡明确指定了 `cardType` 为 `22`，但是下单购买畅读卡没有明确指定 `cardType` 参数，这时该值就会复用上一次请求中的参数值（因为定义的是全局变量），这种情况下就会导致参数出现错乱。

## 代码验证

### 测试场景一

对于上述推测内容进行了测试，测试代码如下

```java
public class SynchroReadCardAction extends EbookBaseAction {
    //定义全局变量
    private String pin; 
    private Integer cardType;
    public String execute() {
        Map<String, Object> resultMap = new HashMap<String, Object>();
        ValueStack valueStack = ActionContext.getContext().getValueStack();
        logger.error("出参=======pin===" + pin + "===cardType=======" + cardType);
        resultMap.put("cardType", cardType);
        resultMap.put("pin", pin);
        valueStack.set("resultMap", resultMap);
        return SUCCESS;
     }
     ……定义全局变量get和set方法
}
```

上述方法实现很简单，就是将 `http` 请求的参数放入 `valueStack` 返回，查看请求的参数和返回的参数是否一致，测试过程如下：

1. 在 `spring-confi-struts.xml` 中配置中配置 `SynchroReadCardAction` 类，并且不指定 `scope` 参数，采用默认值（`singleton`），测试结果如下：

```
//1、http请求信息
http://cread.e.jd.com/syncReadCard.action?pin=dzllikelsw&cardType=2
返回结果：
{
    "pin": "dzllikelsw",
    "cardType": 2
}

//2、http请求信息
http://cread.e.jd.com/syncReadCard.action?pin=lisi
返回结果：
{
    "pin": "lisi",
    "cardType": 2
}
```

上述测试可以很明显发现问题，在第二个请求中没有指定 `cardType` 参数内容，但是结果中依然有 `cardType` 值为 `2`，错误很明显。

2. 在 `spring-confi-struts.xml` 中配置中配置 `SynchroReadCardAction` 类，并且指定 `scope` 参数，设置为 `prototype`，测试结果如下：

```
//1、http请求信息
http://cread.e.jd.com/syncReadCard.action?pin=dzllikelsw&cardType=2
返回结果：
{
    "pin": "dzllikelsw",
    "cardType": 2
}

//2、http请求信息
http://cread.e.jd.com/syncReadCard.action?pin=lisi
返回结果：
{
     "pin": "lisi"
}
```

通过上述测试，我们可以发现，这次的请求返回结果和我们预期的内容是一致的。

### 测试场景二

1. 修改测试代码内容如下，在 `spring-config-struts.xml` 中将 `SynchroReadCardAction` 配置还原为默认配置（`singleton`）：

```java
public class SynchroReadCardAction extends EbookBaseAction {
    //定义全局变量
    private String pin; 
    private Integer cardType;

    public String execute() {
        Map<String, Object> resultMap = new HashMap<String, Object>();
        ValueStack valueStack = ActionContext.getContext().getValueStack();
        try {
            Thread.sleep(15000);
        } catch (Exception e) {
        
        }
        logger.error("出参=======pin===" + pin + "===cardType=======" + cardType);
        resultMap.put("cardType", cardType);
        resultMap.put("pin", pin);
        valueStack.set("resultMap", resultMap);
        return SUCCESS;
    }
    ……定义全局变量get和set方法
}
```

```
//1、http请求信息
http://cread.e.jd.com/syncReadCard.action?pin=lisi&cardType=7
返回结果：
{
    "pin": "lisi",
    "cardType": 6
}

//2、http请求信息
http://cread.e.jd.com/syncReadCard.action?pin=lisi&cardType=6
返回结果：
{
    "pin": "lisi",
    "cardType": 6
}
```

上述代码中我们通过让线程睡眠 `15s` 的方式来模拟一下出现并发时的场景，出现并发时可能会出现多个请求同时进入代码块执行方法体，通过上述测试，我们首先发起一个请求，让后又发起另外一个请求，请求参数不同，由于存在先后顺序，**导致后面的请求会将前面的请求参数覆盖掉**，通过结果展示，的确出现了参数返回值错误的情况。

如果将 `SynchroReadCardAction` 类配置为多例（`prototype`）模式，我们发现上述情况是不会出现的，返回值是正常。

### 测试场景三

修改测试代码内容如下，在 `Spring` 中将 `SynchroReadCardAction` 还还原为默认配置（`singleton`）：

```java
public class SynchroReadCardAction extends EbookBaseAction {
    //定义全局变量
    private String pin; 
    private Integer cardType;

    public String execute() {
        Map<String, Object> resultMap = new HashMap<String, Object>();
        ValueStack valueStack = ActionContext.getContext().getValueStack();
        String pin = request.getParameter("pin");
        String cardType = request.getParameter("cardType");
        logger.error("出参=======pin===" + pin + "===cardType=======" + cardType);
        resultMap.put("cardType", cardType);
        resultMap.put("pin", pin);
        valueStack.set("resultMap", resultMap);
        return SUCCESS;
    }
    ……定义全局变量get和set方法
}
```

上述测试代码中我们将获取参数的方式修改为通过 `request.getParameter` 参数的方式，测试结果如下：

```
//1、http请求信息
http://cread.e.jd.com/syncReadCard.action?pin=dzllikelsw&cardType=2
返回结果：
{
    "pin": "dzllikelsw",
    "cardType": 2
}

//2、http请求信息
http://cread.e.jd.com/syncReadCard.action?pin=lisi
返回结果：
{
    "pin": "lisi"
}
```

通过上述测试，我们可以确定，通过使用 `request.getParameter` 的方式来获取参数，由于不再使用全局变量，即使设置为单例作用域也是不会出现参数错误问题的。

## 最终解决方式

1. 将 `spring-confi-struts.xml` 配置文件中 `SynchroReadCardAction` 类的作用域修改为 `prototype` 类型。（**目前线上采用的修改方式，改动很小即可解决问题**）

2. 修改程序中获取参数的方式，修改为使用 `request.getParameter` 方式获取参数，不使用全局变量并通过 `get`、`set` 方法获取参数，这种情况下 `SynchroReadCardAction` 的配置还可以继续采用默认配置（`singleton`）。
