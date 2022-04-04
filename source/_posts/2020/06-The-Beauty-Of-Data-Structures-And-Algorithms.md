---
title: 极客时间 《数据结构与算法之美》 学习笔记
date: 2020-03-05 09:36:44
cover: https://gitee.com/dongzl/article-images/raw/master/cover/algorithms.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 《极客时间》网站《数据结构与算法之美》课程学习笔记内容。

categories: 
  - 数据结构与算法

tags: 
  - 数据结构
  - 算法
---

本文是 [极客时间](https://time.geekbang.org/) 上 [数据结构与算法之美](https://time.geekbang.org/column/intro/126) 课程的学习笔记内容。

<!-- more -->

## 基础篇

### 10 | 递归：如何用三行代码找到“最终推荐人”？

**递归需要满足的三个条件**：
- 一个问题的解可以分解为几个子问题的解
- 这个问题与分解之后的子问题，除了数据规模不同，求解思路完全一样
- 存在递归终止条件

<font color="red">写递归代码的关键就是找到如何将大问题分解为小问题的规律，并且基于此写出递推公式，然后再推敲终止条件，最后将递推公式和终止条件翻译成代码</font>

**编写递归代码的关键是，只要遇到递归，我们就把它抽象成一个递推公式，不用想一层层的调用关系，不要试图用人脑去分解递归的每个步骤。**

**使用递归可能需要规避的问题**：
- 递归代码要警惕堆栈溢出
- 递归代码要警惕重复计算

[还有程序员天真地以为”尾递归“真的可以避免堆栈溢出！](https://mp.weixin.qq.com/s/Ki3WN2AJ5HhxxmaQ0lVh3Q)

