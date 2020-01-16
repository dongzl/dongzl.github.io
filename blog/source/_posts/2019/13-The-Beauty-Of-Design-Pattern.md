---
title: 极客时间 《设计模式之美》 学习笔记
date: 2019-12-26 21:35:58
categories:
- design-pattern
tags:
- 设计模式
- 设计原则
---

本文是 [极客时间](https://time.geekbang.org/) 上 [设计模式之美](https://time.geekbang.org/column/intro/100039001) 课程的学习笔记内容。

<!-- more -->

## 设计原则与思想：设计原则

### 15 | 理论一：对于单一职责原则，如何判定某个类的职责是否够“单一”？

单一职责原则：Single Responsibility Principle，缩写为 SRP。

我们可以现写一个粗粒度的类，满足业务需求。随着业务的发展，如果粗粒度的类越来越大，代码越来越多，这个时候，我们就可以将这个粗粒度的类，拆分成几个更细度的类。

**如何理解单一职责（SRP）**

一个类只负责完成一个职责或者功能。不要设计大而全的类，要设计粒度小、功能单一的类。单一职责原则是为了实现代码高内聚、低耦合，提高代码的复用性、可读性、可维护性。

**如何判断类的职责是否足够单一**

- 类中的代码行数、函数或者属性过多；
- 类依赖的其他类过多，或者依赖类的其他类过多；
- 私有方法过多；
- 比较难给类起一个合适的名字；
- 类中大量的方法都是集中操作类中的某几个属性。

---

### 16 | 理论二：如何做到“对扩展开放、修改关闭”？扩展和修改各指什么？

开闭原则：Open Closed Principle，简写为 OCP。

> Software entities (modules, classes, functions, etc.) should be open for extension , but closed for modification。
 
我们把它翻译成中文就是：软件实体（模块、类、方法等）应该“对扩展开放、对修改关闭”。

---

### 17 | 理论三：里式替换（LSP）跟多态有何区别？哪些代码违背了LSP？

里式替换原则：Liskov Substitution Principle，缩写为 LSP。

> Functions that use pointers of references to base classes must be able to use objects of derived classes without knowing it。

子类对象（object of subtype/derived class）能够替换程序（program）中父类对象（object of base/parent class）出现的任何地方，并且保证原来程序的逻辑行为（behavior）不变及正确性不被破坏。

**违反里式替换原则的情况：**

- 子类违背父类声明要实现的功能；
- 子类违背父类对输入、输出、异常的约定；
- 子类违背父类注释中所罗列的任何特殊说明。

**多态 & 里式替换原则**：

多态是面向对象编程的一大特性，也是面向对象编程语言的一种语法。它是一种代码实现的思路。而里式替换是一种设计原则，用来指导继承关系中子类该如何设计，子类的设计要保证在替换父类的时候，不改变原有程序的逻辑及不破坏原有程序的正确性。

---

### 18 | 理论四：接口隔离原则有哪三种应用？原则中的“接口”该如何理解？

接口隔离原则：Interface Segregation Principle”，缩写为 ISP。

> Clients should not be forced to depend upon interfaces that they do not use。

如果把“接口”理解为一组接口集合，可以是某个微服务的接口，也可以是某个类库的接口等。如果部分接口只被部分调用者使用，我们就需要将这部分接口隔离出来，单独给这部分调用者使用，而不强迫其他调用者也依赖这部分不会被用到的接口。

如果把“接口”理解为单个 API 接口或函数，部分调用者只需要函数中的部分功能，那我们就需要把函数拆分成粒度更细的多个函数，让调用者只依赖它需要的那个细粒度函数。

如果把“接口”理解为 OOP 中的接口，也可以理解为面向对象编程语言中的接口语法。那接口的设计要尽量单一，不要让接口的实现类和调用者，依赖不需要的接口函数。

---

### 19 | 理论五：控制反转、依赖反转、依赖注入，这三者有何区别和联系？

依赖倒置原则：Dependency Inversion Principle，缩写为 DIP。

> High-level modules shouldn’t depend on low-level modules. Both modules should depend on abstractions. In addition, abstractions shouldn’t depend on details. Details depend on abstractions.

高层模块（high-level modules）不要依赖低层模块（low-level）。高层模块和低层模块应该通过抽象（abstractions）来互相依赖。除此之外，抽象（abstractions）不要依赖具体实现细节（details），具体实现细节（details）依赖抽象（abstractions）。

---

### 20 | 理论六：我为何说KISS、YAGNI原则看似简单，却经常被用错？

> Keep It Simple and Stupid. 尽量保持简单。

**如何写出满足 KISS 原则的代码？**

- 不要使用同事可能不懂的技术来实现代码。
- 不要重复造轮子，要善于使用已经有的工具类库。
- 不要过度优化。不要过度使用一些奇技淫巧来优化代码，牺牲代码的可读性。

**YAGNI 原则**（不要做过度设计）

> YAGNI 原则的英文全称是：You Ain’t Gonna Need It。

---

### 21 | 理论七：重复的代码就一定违背DRY吗？如何提高代码的复用性？

**DRY 原则（Don’t Repeat Yourself）**

- 实现逻辑重复（不违反 DRY 原则）
- 功能语义逻辑（违反 DRY 原则）
- 代码执行重复（违反 DRY 原则）

**提高代码可复用性的方法**

- 减少代码耦合
- 满足单一职责原则
- 模块化
- 业务与非业务逻辑分离
- 通用代码下沉
- 继承、多态、抽象、封装
- 应用模板等设计模式

---

### 22 | 理论八：如何用迪米特法则（LOD）实现“高内聚、松耦合”？

迪米特法则：Law of Demeter，缩写是 LOD。

最小知识原则：The Least Knowledge Principle。

> Each unit should have only limited knowledge about other units: only units “closely” related to the current unit. Or: Each unit should only talk to its friends; Don’t talk to strangers.

---

### 23 | 实战一（上）：针对业务系统的开发，如何做需求分析和设计？

- 技术人也要有一些产品思维。对于产品设计、需求分析，我们要学会“借鉴”，一定不要自己闷头想。一方面这样做很难想全面，另一方面从零开始设计也比较浪费时间。除此之外，我们还可以通过线框图和用户用例来细化业务流程，挖掘一些比较细节的、不容易想到的功能点。

- 面向对象设计聚焦在代码层面（主要是针对类），那系统设计就是聚焦在架构层面（主要是针对模块），两者有很多相似之处。很多设计原则和思想不仅仅可以应用到代码设计中，还能用到架构设计中。实际上，我们可以借鉴面向对象设计的步骤，来做系统设计。

- 面向对象设计的本质就是把合适的代码放到合适的类中。合理地划分代码可以实现代码的高内聚、低耦合，类与类之间的交互简单清晰，代码整体结构一目了然。类比面向对象设计，系统设计实际上就是将合适的功能放到合适的模块中。合理地划分模块也可以做到模块层面的高内聚、低耦合，架构整洁清晰。在面向对象设计中，类设计好之后，我们需要设计类之间的交互关系。类比到系统设计，系统职责划分好之后，接下来就是设计系统之间的交互了。

---

### 24 | 实战一（下）：如何实现一个遵从设计原则的积分兑换系统？

**MVC 三层开发作用：**

- 分层能起到代码复用的作用
- 分层能起到隔离变化的作用
- 分层能起到隔离关注点的作用
- 分层能提高代码的可测试性
- 分层能应对系统的复杂性

<img src="https://static001.geekbang.org/resource/image/33/61/3332071c82182e72518143d05b8eaa61.jpg">

---

### 25 | 实战二（上）：针对非业务的通用框架开发，如何做需求分析和设计？

---

### 26 | 实战二（下）：如何实现一个支持各种统计规则的性能计数器？

---

## 参考资料

- [The Clean Code Blog](https://blog.cleancoder.com/uncle-bob/2014/05/08/SingleReponsibilityPrinciple.html)