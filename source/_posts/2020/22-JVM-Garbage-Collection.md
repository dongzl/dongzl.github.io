---
title: 如何用一句话证明你的 JVM 水平很牛
date: 2020-04-13 18:37:06
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/jvm_study.png
# author information, multiple authors are set to array
# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 面试官：用一句话证明你的 JVM 水平很牛；面试者：精通 HotSpot 虚拟机的 10 种垃圾回收器的实现原理。
categories: 
  - java开发
tags: 
  - JVM
---

## 背景描述

> 面试官：用一句话证明你的 JVM 水平很牛；面试者：精通 HotSpot 虚拟机的 10 种垃圾回收器的实现原理。

如果真有面试官这么问，会感觉很不专业，但是据听说的确是有人碰到过，现在的 Java 服务端面试，`JVM` 的问题基本上是绕不过去的坎儿了，今天这篇文章由这个奇葩的面试题引入，总结一下 `JVM` 中常用垃圾回收算法的一些知识内容，总结的不一定深入，但是先混个脸儿熟。

## 分代收集理论

分代收集假说：

- 弱分代假说（Weak Generational Hypothesis）：绝大多数对象都是朝生夕灭的。
- 强分代假说（Strong Generational Hypothesis）：熬过越多次垃圾收集过程的对象就越难以消亡。
- 跨代引用假说（Intergenerational Reference Hypothesis）：跨代引用相对于同代引用来说只占极少数。

基于以上几个分代假说，奠定了多款常用的垃圾收集器的设计原则：

- 收集器应该将 Java 堆划分出不同的区域，然后将回收对象依据其年龄（年龄即对象熬过垃圾收集过程的次数）分配到不同的区域之中存储。
- 在Java堆划分出不同的区域之后，垃圾收集器才可以每次只回收其中某一个或者某些部分的区域，同时能够针对不同的区域安排与里面存储对象存亡特征相匹配的垃圾收集算法。
- 如果对象之间存在跨代引用，那么存在互相引用关系的两个对象，是应该倾向于同时生存或者同时消亡的，所以没必要因为存在跨代引用，在新生代垃圾回收时扫描整个老年代，只需在新生代上建立一个全局的数据结构（该结构被称为“记忆集”，Remembered Set），这个结构把老年代划分成若干小块，标识出老年代的哪一块内存会存在跨代引用。

## 垃圾收集算法

### 标记--清除算法（Mark-Sweep）

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/22-JVM-Garbage-Collection/JVM-Garbage-Collection-01.png" width="600px">

优点：

- 系统最初运行时，垃圾回收效率可能会比较高，因为不需要做内存间的拷贝。

缺点：

- 执行效率不稳定，如果 JVM 中包含大量朝生夕灭的对象，就需要进行大量标记和清除的动作，导致标记和清除两个过程的执行效率都随对象数量增长而降低；
- 内存空间的碎片化问题，标记、清除之后会产生大量不连续的内存碎片，如果无法给一个大对象分配足够空间，JVM 会提前触发 Full GC。

### 标记--复制算法（Copying）

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/22-JVM-Garbage-Collection/JVM-Garbage-Collection-02.png" width="600px">

优点：

- 不会有内存碎片化的问题；
- 效率很高，不管是拷贝，还是清除整块空间的内存，速度都是非常快的；
- 顺带的进行了内存的压缩。

缺点：
- 浪费一半内存空间。

### 标记--整理算法（Mark-Compack）

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/22-JVM-Garbage-Collection/JVM-Garbage-Collection-03.png" width="600px">

优点：

- 不会有内存碎片化的问题；
- 不会有内存空间浪费。

缺点：
- 效率并不算高，比较适用于老年代区的垃圾回收。

这个算法是老年代内存区的默认算法，因为老年代中，存活对象比较多，垃圾对象比较少，这个算法压缩的过程实际上是将存活的对象替换掉垃圾对象，所以在垃圾对象比较少的情况下，只需要很少的操作就能完成GC。

### JVM 分代算法

- 年轻代（New / Young）
  - 存活对象少
  - 使用 copy 算法，效率高

- 年老代（Old）
  - 垃圾对象少
  - 使用 Mark-Compack 算法

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/22-JVM-Garbage-Collection/JVM-Garbage-Collection-04.png" width="600px">

## HotSpot 虚拟机垃圾收集器

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/22-JVM-Garbage-Collection/JVM-Garbage-Collection-05.jpg" width="600px">

### Serial

**a stop-the-world, copying collector which uses a single GC thread.**

- 客户端模式下的默认新生代收集器；
- 简单而高效，额外的内存消耗最小；
- 可以管理较小内存（几十兆到一两百兆内存）

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/22-JVM-Garbage-Collection/JVM-Garbage-Collection-06.jpg" width="600px">

Serial/Serial Old收集器运行示意图

### Serial Old

**a stop-the-world, mark-sweep-compact collector that uses a single GC thread.**

Serial Old是Serial收集器的老年代版本，它同样是一个单线程收集器，使用标记-整理算法。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/22-JVM-Garbage-Collection/JVM-Garbage-Collection-06.jpg" width="600px">

### Parallel Scavenge

**a stop-the-world, copying collector which uses multiple GC thread.**

- 新生代收集器；
- 基于标记--复制算法的收集器；
- 并行收集的多线程收集器；
- 可控制吞吐量的收集器。

$$吞吐量=\frac{运行用户代码时间}{运行用户代码时间+运行垃圾收集时间}$$

- -XX:MaxGCPauseMillis：控制最大垃圾收集停顿时间；该参数允许的值是一个大于 0 的毫秒数，收集器将尽力保证内存回收花费的时间不超过用户设定值；

- -XX:GCTimeRatio：直接设置吞吐量大小，该参数的值是一个大于 0 小于 100 的整数，也就是垃圾收集时间占总时间的比率，相当于吞吐量的倒数。譬如把此参数设置为 19，那允许的最大垃圾收集时间就占总时间的 5%（即1/(1+19)），默认值为 99，即允许最大 1%（即1/(1+99)）的垃圾收集时间。

- -XX:+UseAdaptiveSizePolicy：开启自适应调节策略，当这个参数被激活之后，就不需要人工一些参数细节了，虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或者最大的吞吐量。

### ParNew

- ParNew 收集器实质上是 Serial 收集器的多线程并行版本。

- 除了 Serial 收集器外，目前只有 ParNew 能与 CMS 收集器配合工作。

- 使用 `-XX:+UseConcMarkSweepGC` 选项激活老年代 CMS 收集器后，ParNew 就是默认开启的新生代收集器，也可以使用 `-XX:+/-UseParNewGC` 来强制使用或禁止它。

- JDK 9 之后 -XX：+UseParNewGC 参数被取消，这意味着ParNew和CMS从此只能互相搭配使用，再也没有其他收集器能够和它们配合了。

- ParNew 收集器默认开启的收集线程数与处理器核心数量相同，在处理器核心非常多的环境中，可以使用 `-XX:ParallelGCThreads` 参数来限制垃圾收集的线程数。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/22-JVM-Garbage-Collection/JVM-Garbage-Collection-08.jpg" width="600px">

ParNew/Serial Old收集器运行示意图

### Parallel Old

Parallel Old是Parallel Scavenge收集器的老年代版本，支持多线程并发收集，基于标记-整理算法实现。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/22-JVM-Garbage-Collection/JVM-Garbage-Collection-09.jpg" width="600px">

Parallel Scavenge/Parallel Old收集器运行示意图

### CMS

CMS（Concurrent Mark Sweep）收集器是一种以获取最短回收停顿时间为目标的收集器，该收集器基于标记--清除算法实现。

CMS 垃圾收集四个步骤：

- 初始标记（CMS initial mark）：Stop The World，初始标记仅仅只是标记一下GC Roots能直接关联到的对象，速度很快；

- 并发标记（CMS concurrent mark）：并发标记阶段就是从GC Roots的直接关联对象开始遍历整个对象图的过程，这个过程耗时较长但是不需要停顿用户线程，可以与垃圾收集线程一起并发运行；

- 重新标记（CMS remark）：Stop The World，重新标记阶段则是为了修正并发标记期间，因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间通常会比初始标记阶段稍长一些，但也远比并发标记阶段的时间短；

- 并发清除（CMS concurrent sweep）：并发清除阶段，清理删除掉标记阶段判断的已经死亡的对象，由于不需要移动存活对象，所以这个阶段也是可以与用户线程同时并发的。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/22-JVM-Garbage-Collection/JVM-Garbage-Collection-10.jpg" width="600px">

Concurrent Mark Sweep收集器运行示意图

CMS 垃圾收集器不足：

- CMS收集器对处理器资源非常敏感；CMS默认启动的回收线程数是（处理器核心数量+3）/4，当处理器核心数比较小时，垃圾回收器会占用比较多的处理器运算资源；

- CMS 垃圾收集器无法处理“浮动垃圾（Floating Garbage）”，浮动垃圾是 CMS 在并发标记和并发清理阶段用户线程运行产生的垃圾；所以需要预留一段内存空间供并发收集时的程序运作使用，`-XX:CMSInitiatingOccu-pancyFraction` 参数用于设置该比例，表示老年代使用了该比率的空间后就会被触发垃圾回收；

- 基于标记--清除算法实现，会产生内存碎片。
  - -XX:+UseCMS-CompactAtFullCollection：在CMS收集器不得不进行Full GC时开启内存碎片的合并整理过程；
  - -XX:CMSFullGCsBefore-Compaction：这个参数的作用是要求 CMS 收集器在执行过若干次（数量由参数值决定）不整理空间的 Full GC 之后，下一次进入 Full GC 前会先进行碎片整理（默认值为 0，表示每次进入 Full GC 时都进行碎片整理）。

### Garbage First

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/22-JVM-Garbage-Collection/JVM-Garbage-Collection-11.png" width="600px">

- G1是一款主要面向服务端应用的垃圾收集器。

- G1不再坚持固定大小以及固定数量的分代区域划分，而是把连续的Java堆划分为多个大小相等的独立区域（Region），每一个Region都可以根据需要，扮演新生代的Eden空间、Survivor空间，或者老年代空间。

- Region中还有一类特殊的Humongous区域，专门用来存储大对象。G1认为只要大小超过了一个Region容量一半的对象即可判定为大对象。

- -XX:G1HeapRegionSize：指定 Region 大小，取值范围 1MB ~ 32MB， 为 2 的 N 次幂。

- G1 收集器负责跟踪各个 Region 里面的垃圾堆积的“价值”大小，价值即回收所获得的空间大小以及回收所需时间的经验值，然后在后台维护一个优先级列表，每次根据用户设定允许的收集停顿时间（使用参数-XX:MaxGCPauseMillis指定，默认值是200毫秒），优先处理回收价值收益最大的那些 Region，这也就是“Garbage First” 名字的由来。

G1 收集器工作步骤：
- 初始标记 （Initial Marking）
- 并发标记（Concurrent Marking）
- 最终标记（Final Marking）
- 筛选回收（Live Data Counting and Evacuation）

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/22-JVM-Garbage-Collection/JVM-Garbage-Collection-12.jpg" width="600px">

### Shenandoah

Shenandoah 收集器工作过程：

- 初始标记（Initial Marking）

- 并发标记（Concurrent Marking）

- 最终标记（Final Marking）

- 并发清理（Concurrent Cleanup）

- 并发回收（Concurrent Evacuation）

- 初始引用更新（Initial Update Reference）

- 并发引用更新（Concurrent Update Reference）

- 最终引用更新（Final Update Reference）

- 并发清理（Concurrent Cleanup）

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/22-JVM-Garbage-Collection/JVM-Garbage-Collection-13.png" width="600px">

Shenandoah 收集器的工作过程

### ZGC

ZGC 是一款在 JDK 11 中新加入的具有实验性质的低延迟垃圾收集器。

ZGC 和 Shenandoah 的目标是高度相似的，都希望在尽可能对吞吐量影响不太大的前提下，实现在任意堆内存大小下都可以把垃圾收集的停顿时间限制在十毫秒以内的低延迟。

ZGC 也是采用基于Region的堆内存布局，ZGC 的 Region 具有动态性——动态创建和销毁，以及动态的区域容量大小。

- 小型Region（Small Region）：容量固定为2MB，用于放置小于256KB的小对象。

- 中型Region（Medium Region）：容量固定为32MB，用于放置大于等于256KB但小于4MB的对象。

- 大型 Region（Large Region）：容量不固定，可以动态变化，但必须为2MB的整数倍，用于放置4MB或以上的大对象。

<img src="https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/2020/22-JVM-Garbage-Collection/JVM-Garbage-Collection-14.jpg" width="600px">

ZGC 执行过程：

- 并发标记 （Concurrent Mark）

- 并发预备重分配 （Concurrent Prepare for Relocate）

- 并发重分配 （Concurrent Relocate）

- 并发重映射 （Concurrent Remap）

### Epsilon

Epsilon 被形容是一个无操作的收集器，如果某个应用只要运行数分钟甚至数秒，只要 Java 虚拟机能正确分配内存，在堆耗尽之前就会退出，那显然运行负载极小、没有任何回收行为的 Epsilon 便是很恰当的选择。

## 参考资料

- 深入理解Java虚拟机：JVM高级特性与最佳实践（第3版）作者：周志明 机械工业出版社 2029-12-01 出版，ISBN：9787111641247











