---
title: 计算机中常用数学公式汇总
date: 2020-02-17 22:04:54
cover: https://cdn.jsdelivr.net/gh/dongzl/dongzl.github.io@hexo/source/images/cover/algorithms.png

# single author
author:
  - nick: 董宗磊
    link: https://www.github.com/dongzl

# post subtitle in your index page
subtitle: 本文旨在总结在计算机中常用的数学公式（持续更新中）。

categories: 
  - 其他

tags: 
  - 数学公式
---

- 斐波那契数列通向公式：

$$a_n=\frac{1}{\sqrt{5}} \begin{bmatrix} \begin{pmatrix} \frac{1 + \sqrt{5}}{2} \end{pmatrix}^n - \begin{pmatrix} \frac{1 - \sqrt{5}}{2} \end{pmatrix}^n\end{bmatrix}$$

- 斐波那契数列矩阵方程：

$$\begin{bmatrix} f(n)\\\\ f(n - 1) \end{bmatrix} = \begin{bmatrix} 1&1\\\\ 1&0 \end{bmatrix} \begin{bmatrix} f(n - 1)\\\\ f(n - 2) \end{bmatrix} = \begin{bmatrix} 1&1\\\\ 1&0 \end{bmatrix}^{n + 1} \begin{bmatrix} f(1)\\\\ f(0) \end{bmatrix}$$