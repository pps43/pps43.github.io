---
title: "二范数下几个问题的统一解释"
date: 2014-11-20
draft: false
tags: ["数学","信号处理"]
# cover:
#     image: "<image path/url>" # image path/url
#     alt: "<alt text>" # alt text
#     caption: "<text>" # display caption under cover
#     relative: false # when using page bundles set this to true
#     hidden: true # only hide on current single page
---

范数、内积空间的概念这里不做赘述，只点几个要点。

粗糙的说，对于V(F)空间，定义内积为一种将两个属于此集合V的元素与一个来自数域F的元素对应起来，并且满足一些抽象但自然的性质（不举）。为了方便将两个元素的内积记为：\\(< \bf{a,b} > ，\bf{a,b}\\) 属于V。

而范数也是一种将V中元素映射成F中的一个数的方法。映射后需要满足的性质如：非负性、齐次性、三角不等式。满足这种性质都可以叫求范数，常见的范数有：1-范数，2-范数，无穷范数。记为：\\({\rm{||}}{\bf{a}}{\rm{||}}\\)

二范数和内积的关系可以写为：\\({\rm{||}}{\bf{a}}{\rm{||}} = \sqrt { < {\bf{a,a}} > }\\)

可以证明一下几个运算在对应的空间中都满足内积的定义。

内积空间满足以下两个重要的不等式：

基础的准备工作就这么多。下面通过发现几个问题的相似性，提出用内积表示的统一形式，其实这些问题的解就是此形式在不同内积空间二范数下的特例。

# P阶线性预测
SimonHaykin《通信原理》P224

求解一系列\\(w(k)\\)，其解是 `wiener-hopf `方程，在以前的文章中有详细解释。

# LMMSE参数估计
很多书都会提及。根据观测数据\\(y\\)求某个待估参数\\(\theta\\)。最一般的表达式如下：

$${\bf{\theta }} = {C_y}^{ - 1}{C_{\theta y}}$$

其实是和P阶线性预测本质是一样的，但仔细看看却有差别。聪明的读者，你们看出来了吗？（此句是向某些恼人的书致敬，每当到关键点，作者就会这样搪塞过去）。

# 最佳平方逼近
常见于数值分析。

首先聊两句插值和逼近的区别。插值要求在插值点上与待插函数完全相同，用一个简单函数近似待插函数。逼近则不要求某些点上函数值一定相同，而是在一段区间上给出准则，然后最小化。其中一个准则便是最佳平方逼近。

再多说一句，函数可以看做无穷维向量，用无穷维向量的角度看函数，有种触类旁通的感觉。

# 统一形式

简写为：

$${\bf{Ax = b}}$$

A矩阵式正定的，而且是`Hermite`阵。应该可以用 `Levinson Dubin` 递归法快速求解。

整理到这里会觉得自己弱爆了，不过是数学上已经有的东西，好比我只是将其重新抄了一遍放在这里。

这确实有点意思。