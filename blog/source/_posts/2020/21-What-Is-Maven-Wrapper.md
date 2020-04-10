---
title: Maven Wrapper 工具在开源项目中的使用
date: 2020-04-08 17:11:54
cover: https://gitee.com/dongzl/article-images/raw/master/cover/maven_study.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文是根据参与开源项目的实际经验，总结 Maven Wrapper 工具的使用。

categories: 
  - web开发

tags: 
  - maven
---

> 声明：本文内容摘自 [Mvnw 简介](http://www.javacoder.cn/?p=759) 博客内容，部分内容有调整。

## 背景介绍

最近一段时间在参与 [ShardingSphere](http://shardingsphere.apache.org/) 开源项目的一些工作，在项目有贡献者提交了 PR，将 [Maven Wrapper](https://github.com/takari/maven-wrapper) 工具集成到了该项目中，第一次听说这个工具，就顺带自己学习了一下，同时也将这个工具集成到了 [DolphinScheduler](http://dolphinscheduler.apache.org) 项目中（PR 地址：[Add maven-wrapper support for dolphinscheduler. ](https://github.com/apache/incubator-dolphinscheduler/pull/2381)）。

## Maven Wrapper 原理

`Maven` 是 `java` 开发中广泛使用的项目构建工具，目前 `maven` 工具已经发行了很多个版本，如果在一个公司内部，我们可以通过规定要求大家使用统一的 `maven` 版本，那如果是开源项目，如何保证彼此不熟悉的开源参与者使用的 `maven` 版本保持一致呢，这时候 `Maven Wrapper` 就派上用场了。`Maven Wrapper` 的原理是在 `maven-wrapper.properties` 文件中记录项目中使用的 `maven` 版本，当用户执行 `mvnw clean` 命令时，发现当前用户的 `maven` 版本和期望的版本不一致，那么就下载指定的版本，然后用指定的版本来执行 `mvn` 命令。

## Maven Wrapper 使用

### 在 pom.xml 文件中添加 plugin 声明

```xml
<plugin>
    <groupId>com.rimerosolutions.maven.plugins</groupId>
    <artifactId>wrapper-maven-plugin</artifactId>
    <version>0.0.5</version>
</plugin>
```

当执行 `mvn wrapper:wrapper` 命令时，`Maven Wrapper` 会自动生成 `mvnw.bat、mvnw、maven/maven-wrapper.jar、maven/maven-wrapper.properties` 文件。

```shell
maven
  maven-wrapper.jar
  maven-wrapper.properties
mvnw
mvnw.bat
```

然后我们就可以使用 `mvnw` 代替 `mvn` 执行所有的 `maven` 命令，例如 `mvnw clean package`。

### 直接使用 mvn 命令

```shell
mvn -N io.takari:maven:wrapper

# 3.6.3 表示我们指定使用的 maven 版本
mvn -N io.takari:maven:wrapper -Dmaven=3.6.3
```

这种方式产生的内容和第一种方式是一样的，只是目录结构有所不同，

```shell
.mvn
  wrapper
    maven-wrapper.jar
    maven-wrapper.properties
    MavenWrapperDownloader.java
mvnw
mvnw.cmd
```

`maven-wrapper.jar` 和 `maven-wrapper.properties` 在 **.mvn/wrapper** 目录下。

- maven-wrapper.jar file is used to download and invoke maven from the wrapper shell scripts.

- maven-wrapper.properties is used to specify the URL Maven should be downloaded from if not already present on the system.

- Maven is downloaded by compiling and executing the Java class file MavenWrapperDownloader.java present in the same .mvn folder.

### 使用注意事项

由于使用了指定的 `maven` 版本，如果 `settings.xml` 文件没有放到当前用户下的 `.m2` 目录下，那么执行 `mvnw` 时是无法读取到原有的 `settings.xml` 文件。

## 参考资料

- [mvnw是什么（Maven Wrapper/Maven保持构建工具版本一直的工具）](https://www.cnblogs.com/easonjim/p/7620085.html)
- [A Quick Introduction to Maven Wrapper](https://medium.com/xebia-engineering/a-quick-introduction-to-maven-wrapper-f1d9dbb4ea5e)
- [What is the purpose of mvnw and mvnw.cmd files?](https://stackoverflow.com/questions/38723833/what-is-the-purpose-of-mvnw-and-mvnw-cmd-files)