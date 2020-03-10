---
title: Lombok 框架使用总结
date: 2019-11-30 20:38:18
cover: https://gitee.com/dongzl/article-images/raw/master/cover/java_study.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨在学习和总结 Lombok 框架的使用。

categories: 
  - web开发

tags: 
  - Lombok
---

## 背景描述

最近一直在参与 [ShardingSphere](https://github.com/apache/incubator-shardingsphere) 开源项目的一些工作，在 ShardingSphere 项目中大量的使用了 [Lombok](https://projectlombok.org/) 框架，对于 Lombok 框架只是有个大概了解，并没有实际的使用，也没有做过深入的研究，正好借参与 ShardingSphere 项目的机会对 Lombok 框架做一个学习和总结，先总结一下自己目前使用中遇到的问题，后续持续不断更新。

<!-- more -->

首先，来看一下 Lombok 官方对这个框架的介绍：

> Project Lombok is a java library that automatically plugs into your editor and build tools, spicing up your java.
Never write another getter or equals method again, with one annotation your class has a fully featured builder, Automate your logging variables, and much more.

Lombok 是一个 java 类库，它可以自动插入到编辑器和构建工具中，对 java 代码进行增强。

使用 Lombok 后不需要再实现 getter 或者 equals 方法，使用一个注解，您的类就可以拥有一个功能齐全的构造器，自动生成一个日志变量等等功能。

> 测试 Lombok 版本：1.18.10

## 构造方法相关注解

### @NoArgsConstructor

```java
import lombok.AccessLevel;
import lombok.NoArgsConstructor;

@NoArgsConstructor(access = AccessLevel.PRIVATE, staticName = "of")
public class User {
    
    private Integer id;
    
    private String name;
    
    private String age;
}
```
@NoArgsConstructor 用于生成无参构造器，如果类中存在 final 字段，则会报编译错误。
- `access` 参数用于指定构造器的访问权限，默认为 `AccessLevel.PUBLIC` 表示生成 public 访问权限的构造器；
- `staticName` 参数用于自动生成一个静态的“构造器”工厂，其内部包裹着一个私有的构造器，对外提供创建对象的功能，这是明显的工厂模式。

### @RequiredArgsConstructor

```java
import lombok.AccessLevel;
import lombok.NonNull;
import lombok.RequiredArgsConstructor;

@RequiredArgsConstructor(access = AccessLevel.PUBLIC, staticName = "of")
public class User {
    
    private final Integer id;
    
    @NonNull
    private String name;
    
    private String age;
}

public class Test {
    
    public static void main(String args[]) throws Exception {
        User user = User.of(1, "test");
    }
}
```
@RequiredArgsConstructor 用于按照要求生成部分参数构造器，所谓的要求就是包含 final 和 @NonNull 约束标注的字段，会对 @NonNull 字段进行明确的 null 检查。

### @AllArgsConstructor

```java
import lombok.AccessLevel;
import lombok.AllArgsConstructor;
import lombok.NonNull;
import lombok.RequiredArgsConstructor;

@AllArgsConstructor
@RequiredArgsConstructor(access = AccessLevel.PUBLIC, staticName = "of")
public class User {
    
    private final Integer id;
    
    @NonNull
    private String name;
    
    private Integer age;
}

public class Test {
    
    public static void main(String args[]) throws Exception {
        User user = User.of(1, "test");
        User user2 = new User(1, "test", 10);
    }
}
```
@AllArgsConstructor 用于生成包含所有字段的构造器。

## @Getter & @Setter

@Getter & @Setter 标注在字段上，用于自动生成 get、set 方法，boolean 类型字段 get 方法为 isXXX() 方法。

生成的 get、set 方法默认情况下都是 public 的，但也可以手动指定以下四种范围：
```java
AccessLevel.PUBLIC
AccessLevel.MODULE
AccessLevel.PROTECTED
AccessLevel.PACKAGE
AccessLevel.PRIVATE
```
@Getter & @Setter 也可以标注在类上，表示针对该类中所有的非静态字段进行 get、set 方法自动生成。如果指定某个字段的 `AccessLevel = AccessLevel.NONE`，则可以使该生成动作失效，此时可以手动实现get、set方法，AccessLevel.NONE 可以应用于在某些方法中有一些自定义逻辑的情况下。

```java
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Getter
@Setter
@NoArgsConstructor
public class User {
    
    @Setter(value = AccessLevel.PRIVATE)
    private Integer id;
    
    @Getter(value = AccessLevel.PACKAGE)
    private String name;
    
    private Integer age;
    
    @Getter(value = AccessLevel.NONE)
    private boolean student;
    
    public boolean isStudent() {
        if (age < 18) {
            return true;
        }
        return false;
    }
}
```

## @ToString

@ToString 标注于类之上，用于生成 toString() 方法。

- includeFieldNames：默认为 true，表示在 toString 输出时输出字段名称；
- exclude：字符串数组，表示在 toString 输出时排除掉字段名称，这个属性将要被废弃了，用 @ToString.Exclude 来代替；
- of：字符串数组，明确的列出在 toString 输出时输出字段名称，这个属性将要被废弃了，用 @ToString.Include 来代替；
- callSuper：默认为 false，表示生成 toString 时不输出超类中的字段内容；
- doNotUseGetters：默认为 false，表示获取字段值时通过 get 方法获取，设置为 true 表示直接通过字段获取；
- onlyExplicitlyIncluded：默认为 false，表示在 toString 输出时输出全部非静态字段；设置为 true 时，表示只输出 @ToString.Include 标识的字段；
- Exclude：标识属性，表示在 toString 输出时排除该字段；
- 

```java
import lombok.NoArgsConstructor;
import lombok.ToString;

@NoArgsConstructor
@ToString(includeFieldNames = true, exclude = {"id", "name"}, of = {}, callSuper = true, doNotUseGetters = true)
public class User {
    
    private Integer id;
    
    private String name;
    
    private Integer age;
    
    private boolean student;
}
```

## @EqualsAndHashCode

@EqualsAndHashCode 标注于类之上用于生成hashCode方法和equals方法。

- exclude：字符串数组，表示在 toString 输出时排除掉字段名称；
- of：
- callSuper：默认为 false，表示生成 toString 时不输出超类中的字段内容；
- doNotUseGetters：默认为 false，表示获取字段值时通过 get 方法获取，设置为 true 表示直接通过字段获取。

```java
import lombok.EqualsAndHashCode;
import lombok.NoArgsConstructor;

@NoArgsConstructor
@EqualsAndHashCode
public class User {
    
    private Integer id;
    
    private String name;
}
```

## @Data

@Data 标注于类之上，是 @ToString、@EqualsAndHashCode、@Getter、@Setter、@RequiredArgsConstructor的综合体

- staticConstructor：类似于 @RequiredArgsConstructor 中 staticName 参数。

```java
import lombok.Data;

@Data(staticConstructor = "of")
public class User {
    
    private Integer id;
    
    private String name;
}
```

## @SneakyThrows

@SneakyThrows 标注于方法之上用于隐藏异常抛出语句。

## Log 日志

```java
lombok.extern.apachecommons.CommonsLog;
lombok.extern.flogger.Flogger;
lombok.extern.java.Log;
lombok.extern.jbosslog.JBossLog;
lombok.extern.log4j.Log4j;
lombok.extern.log4j.Log4j2;
lombok.extern.slf4j.Slf4j;
lombok.extern.slf4j.XSlf4j;
```

## 参考资料
- [Project Lombok](https://projectlombok.org/)




