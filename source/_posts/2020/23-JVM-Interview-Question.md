---
title: JVM 相关面试题汇总（附简要答案）
date: 2020-04-17 22:48:34
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/jvm_study.png
# author information, multiple authors are set to array
# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨在汇总多个大厂在面试中出现的 JVM 问题，并附上简要答案。

categories: 
  - java开发
tags: 
  - JVM
---

> 1、CMS 和 G1 的异同 -- 百度
> 2、G1 什么时候引发 FullGC -- 百度
> 3、说一个最熟悉的垃圾回收算法 -- 百度
> 4、吞吐量优先和响应时间优先的回收器有哪些 -- 百度
> 5、怎么判断内存泄漏 -- 顺丰
> 6、讲一下 CMS 的流程 -- 顺丰
> 7、为什么压缩指针超过 32G 失效 -- 京东
> 8、什么是内存泄漏？GC 调优有经验吗？一般出现 GC 问题你怎么解决？-- 淘宝
> 9、ThreadLocal 有没有内存泄漏问题 -- 阿里 / 蘑菇街
> 10、G1 两个 Region 不是连续的，而且之间还有可达的引用，我现在要回收一个，另一个怎么处理？-- 阿里
> 11、讲一下 JVM 堆内存管理（对象分配过程） -- 阿里
> 12、听说过 CMS 的并发预处理和并发可中断预处理吗？ -- 阿里
> 13、到底多大的对象会被直接扔到老年代？ -- 阿里
> 14、用一句话说明你的 JVM 水平很牛 -- 某个有病的企业

### 1、CMS 和 G1 的异同 -- 百度

[G1 收集器与 CMS 收集器的对比与实战](https://blog.chriscs.com/2017/06/20/g1-vs-cms/)

### 2、G1 什么时候引发 FullGC -- 百度

G1 提供了两种 GC 模式，Young GC 和 Mixed GC，两种都是完全 Stop The World 的。 

- Young GC：选定所有年轻代里的 Region。通过控制年轻代的 Region 个数，即年轻代内存大小，来控制 Young GC 的时间开销；

- Mixed GC：选定所有年轻代里的 Region，外加根据 global concurrent marking 统计得出收集收益高的若干老年代 Region。在用户指定的开销目标范围内尽可能选择收益高的老年代 Region。

由上面的描述可知，Mixed GC 不是 Full GC，它只能回收部分老年代的 Region，如果 Mixed GC 实在无法跟上程序分配内存的速度，导致老年代填满无法继续进行 Mixed GC，就会使用 Serial Old GC（Full GC）来收集整个 GC heap，所以我们可以知道，G1 是不提供 Full GC的。

[Java Hotspot G1 GC 的一些关键技术](https://tech.meituan.com/2016/09/23/g1.html)

### 3、说一个最熟悉的垃圾回收算法 -- 百度

这道题是个开放性问题，可以通过自己熟悉的垃圾回收算法引申一下该算法相应的垃圾收集器实现，比如，**标记--复制**算法，这是 HotSpot 虚拟机新生代垃圾收集器常用的回收算法，对应的实现有 Serial、Parallel Scavenge，在比如**标记--清除**算法，对应的垃圾收集器实现有 CMS 等等。

一般来说，通过上面的问题，肯定继续引申你所熟悉的 HotSpot 垃圾收集器了，就目前的情况下，我理解主要讲讲 CMS 和 G1 吧；因为在往前说，Serial 和 Parallel 种类的收集器比较老了，在往后说 Shenandoah 和 ZGC 收集器又太新了，工作中使用都不多，目前应该就是 CMS 和 G1，使用比较多，而且实现原理上也比较复杂，和面试官比较有的聊。

### 4、吞吐量优先和响应时间优先的回收器有哪些 -- 百度

### 5、怎么判断内存泄漏 -- 顺丰

### 6、讲一下 CMS 的流程 -- 顺丰

- InitialMarking（初始化标记，整个过程STW）
  - 标记GC Roots可达的老年代对象；
  - 遍历新生代对象，标记可达的老年代对象。

- Marking（并发标记）
- Precleaning（预清理）
- AbortablePreclean（可中断的预清理）
- FinalMarking（并发重新标记，STW过程）

[图解CMS垃圾回收机制，你值得拥有](https://www.jianshu.com/p/2a1b2f17d3e4)

### 7、为什么压缩指针超过 32G 失效 -- 京东

[为什么JVM开启指针压缩后支持的最大堆内存是32G?](https://www.zhihu.com/question/365436606)

### 8、什么是内存泄漏？GC 调优有经验吗？一般出现 GC 问题你怎么解决？-- 淘宝

### 9、ThreadLocal 有没有内存泄漏问题 -- 阿里 / 蘑菇街

### 10、G1 两个 Region 不是连续的，而且之间还有可达的引用，我现在要回收一个，另一个怎么处理？-- 阿里

### 11、讲一下 JVM 堆内存管理（对象分配过程） -- 阿里

### 12、听说过 CMS 的并发预处理和并发可中断预处理吗？ -- 阿里

[图解CMS垃圾回收机制，你值得拥有](https://www.jianshu.com/p/2a1b2f17d3e4)

### 13、到底多大的对象会被直接扔到老年代？ -- 阿里

### 14、用一句话说明你的 JVM 水平很牛 -- 某个有病的企业

精通 HotSpot 虚拟机 10 中垃圾收集器实现原理。