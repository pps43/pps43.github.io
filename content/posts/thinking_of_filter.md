---
title: "谈谈滤波"
date: 2014-10-07
draft: false
tags: ["信号处理","数学"]
hideSummary: false
# cover:
#     image: "<image path/url>" # image path/url
#     alt: "<alt text>" # alt text
#     caption: "<text>" # display caption under cover
#     relative: false # when using page bundles set this to true
#     hidden: true # only hide on current single page
---

> 摸清背景，理清思路，探讨意义。

首先谈谈我关于滤波的理解：
- 在**频域**内对信号中某些频率分量进行衰减。关键词：低通、高通。典型的场景是将一段音乐中不同频段的声音分别抽取出来。
- **空间域**内对不同尺度的细节进行处理。关键词：图像处理，滤镜。典型的场景是将一张被关在笼子里的老虎的照片“还原”出被笼子挡住的部分，有点透视的意味。
- 已经超出了两个字的字面意思，甚至也超出了频域、空域的范畴，进入了**统计的领域**。

---

扑面而来的是维纳滤波。先说说维纳其人。

> [维纳](https://en.wikipedia.org/wiki/Norbert_Wiener)（Norbert Wiener, 1894-1964）有个智识超常的父亲，不仅会40多种语言，而且据说数学功夫了得。小维纳在其教导下，以至于维纳初出茅庐时就已经“满身神装”。 后来，维纳研究的领域覆盖哲学、数学、物理学、工程学甚至生物学，是控制论（cybernetics）的开山鼻祖，也是通信界二号祖师爷（一号是香农）。并且功勋卓著，令人发指。其写了一本小册子《人有人的用处》，其中对信息化对人和社会的影响和变革的讨论，今天读起来依然富有前瞻性。是我最佩服的人类之一。
>
> 维纳不仅是个奇才，也是个奇葩。
>
> 摘一段轶事。维纳在教学上的恶劣表现让学生痛不欲生。有一次维纳在讲解一个定理时，想到了一个很直觉的证明方法，于是只在自己的大脑中推演，一下跳过了很多步骤，只写下一个简单的结果。这当然是为他的学生们无法承受的，于是有人很策略地请求他是否能够用另一种方法再证一遍，他说“当然可以”，马上又在脑中推演，又忘了在黑板上书写，经过几分钟的静默之后，只见他在原来的结果处打了一个查对无误的记号，就下课走了。。。
>

---

回归正题。

在信号接收时，由于信道的不确定性和其他种种干扰，接收方需要对接收信号对应的发送信号进行估计。为了评价估计的性能，我们定义了不少指标。这里采用误差平方函数的期望作为指标，指标的值越小越好。这就是`LMS最小均方算法`的出发点。
以下是用极不严谨的符号定义来说明问题。

发送信号\\(S\\)，接收信号为

$$X=S+N$$

则估计值为

$$\hat S = {\bf{T}}\left( X \right)$$

\\({\bf{T}}\left( \bullet \right)\\) 表示一种变换，\\({\bf{E}}\left( \bullet \right)\\) 表示求期望。根据LMS算法的结果，对发送信号的估计值应为：

$$\hat S = {\bf{E}}(S|X)$$

这显然是难以获知的后验信息，所以这个结果仅仅有理论指导意义。

如果多加一层约束，规定：\\({\bf{T}}\left( \bullet \right)\\) 为线性运算，则运用信号与系统的观点，可以将这个线性系统用冲激响应\\(h(t)\\)（离散形式为\\(h[n]\\)）表示。这里以离散形式为例，这个系统的物理实现是一个\\(N\\)阶抽头滤波器，滤波器系数可以写成一个列向量:

$${\bf{h}} = {[h(0),h(1),…,h(N - 1)]^T}$$

使指标逼近最小值的方法有很多种，如`steepest descent`（最佳梯度法）逼近。也就是对\\(C\\)关于\\({\bf{h}}\\)求偏导，再令偏导为0那一套方法。

通过迭代逼近最小值这是个大的话题，在自适应滤波的书中讨论的比较细致（如稳定性、收敛性等），这里做一个粗略的分类：
- 基于梯度的：牛顿法逼近、最佳梯度法;
- 随机梯度法：LMS算法等;

维纳滤波的数学推导[点这里](https://en.wikipedia.org/wiki/Wiener_filter)，或者统计信号处理的教材。

离散形式的结论是：

$${\bf{h}} = R_X^{ - 1}{r_{SX}}$$

这就是维纳滤波器的真身了。看看，结论是如此的简洁！先别兴奋，再对照一下Simon Haykin的《通信系统》一书第三章关于线性预测的讨论：在没有信道影响和噪声的情况下，同样是最小均方准则，结论是：

$${\bf{w}} = R_X^{ - 1}{r_X}$$

绝了！

---
Update:

- 线性预测和维纳滤波器的推导都是采用最小均方准则的，数学上可以说明这类问题的最优解都是上述形式。所以这**不是巧合**。
- 补充解释下为什么大家钟爱于“均方准则”:
    - 理论推导和实现要比诸如绝对值准则简单；
    - 由二次方构成的性能曲面上，偏导为0的点一定是最小值点。

---

Update:

最后摘一段维纳滤波的**限制因素**：
- 要求接收信号是平稳的随机过程。
- 要求知道互相关系数矩阵。
- 计算出的系统可能因不满足因果性而无法实现。

后来的**卡尔曼滤波**在这一点上有所改进。

慢慢发现，与其是滤波，不如说是线性预测，与其说预测，不如说是均衡。这几个概念我一直比较混乱，以后再进行统一和归纳吧。

---

Update:

**滤波和预测的区别在于功能而不是算法或结构。前者是为了得到更满意的当前量，后者是为了得到满意的未来值。**

但有一点，滤波已经不再是狭义的“滤波”了。