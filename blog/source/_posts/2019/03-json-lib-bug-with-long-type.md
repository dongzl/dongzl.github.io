---
title: json-lib 解析 Long 类型数据 Bug 排查
date: 2019-09-14 08:10:17
cover: https://gitee.com/dongzl/article-images/raw/master/cover/java_study.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文总结了一次在工作使用 json-lib 工具出现的问题以及排查过程。

categories: 
  - web开发

tags: 
  - json-lib
  - Java
  - Long
---

## 背景描述
在开发网文功能过程中，从阅文集团获取的网文数据内容中，定义的数据主键都是 long 数据类型，而且是以 json 格式返回的数据；在通过 json-lib 工具解析 json 的过程中，发现 json-lib 存在的一个 bug。

<!-- more -->

## 现象描述

## 测试代码一

```java
public static void main(String args[]) {
    String json = "{\"chapterId\":13661819409899751}";
    JSONObject jsonObject = JSONObject.fromObject(json);
    System.out.println(jsonObject.getLong("chapterId"));
}
```

输出结果为：

```
13661819409899751
```
上述输出结果没有问题。

## 测试代码二

```java

public static void main(String args[]) {
    String json = "{\"chapterId\":\"13661819409899751\"}";
    JSONObject jsonObject = JSONObject.fromObject(json);
    System.out.println(jsonObject.getLong("chapterId"));
}

```

输出结果为：
```
13661819409899752
```

发现上述结果值是错误的，并没有获取的预期的结果值。

## 问题排查
通过阅读 JSONObject 类中的 getLong 方法实现代码，其内部的处理逻辑如下：

```java
public long getLong(String key) {
    this.verifyIsNull();
    Object o = this.get(key);
    if(o != null) {
        return o instanceof Number?((Number)o).longValue():(long)this.getDouble(key);
    } else {
        throw new JSONException("JSONObject[" + JSONUtils.quote(key) + "] is not a number.");
    }
}
```

- 如果判断 o 是 Number 类型，则直接调用 Number 中的 longValue 方法，获取到 long 类型结果数值，这一步没有问题。
- 如果判 o 断不是 Number 类型，则调用 getDouble 方法。

getDouble 方法实现逻辑如下：

```java
public double getDouble(String key) {
    this.verifyIsNull();
    Object o = this.get(key);
    if(o != null) {
        try {
            return o instanceof Number?((Number)o).doubleValue():Double.parseDouble((String)o);
        } catch (Exception var4) {
            throw new JSONException("JSONObject[" + JSONUtils.quote(key) + "] is not a number.");
        }
    } else {
        throw new JSONException("JSONObject[" + JSONUtils.quote(key) + "] is not a number.");
    }
}
```

- 如果判断 o 是 Number 类型，则直接调用 Number 中的 doubleValue 方法，获取到 double 类型结果数值。
- 如果判断 o 不是 Number 类型，则调用 Double.parseDouble 方法。

在我们处理的 json 字符串中，如果为如下形式：

```
String json = "{\"chapterId\":13661819409899751}";
```

json 字符串中内容不加双引号，自然当做 Number 类型来处理，直接调用 getLong 方法中直接返回 ((Number)o).longValue() 内容，所以返回的数值是没有问题的。

如果我们处理的 json 字符串为如下形式：

```
String json = "{\"chapterId\":\"13661819409899751\"}";
```
json 字符串内容加双引号，并不会当做 Number 类型来处理，最终会调用 Double.parseDouble 方法，返回一个 double 类型数值，最后在 getLong 方法中将 double 类型数值强制转换成 long 类型，其实在 Double.parseDouble 方法中已经出现了精度问题，导致数据不准确。   

例如我们可以做如下测试：

```java
//代码内容
System.out.println(Double.parseDouble("13661819409899751"));

//结果输出
1.3661819409899752E16

```
所以说在上述过程中由于使用 Double.parseDouble 方法引起的精度问题，导致返回数值不准确。

## 处理方式

我们可以通过 jsonObject.getString(“chapterId”) 方式获取到字符串类型内容，然后通过

```java
org.apache.commons.lang.math.NumberUtils.createLong(String str);
```

方法完成对 long 类型转换。