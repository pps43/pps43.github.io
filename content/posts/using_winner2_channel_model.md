---
title: "信道仿真模型 WINNER II"
date: 2014-11-12
# draft: true
tags: ["无线通信"]
# cover:
#     image: "<image path/url>" # image path/url
#     alt: "<alt text>" # alt text
#     caption: "<text>" # display caption under cover
#     relative: false # when using page bundles set this to true
#     hidden: true # only hide on current single page
---

在"[信道仿真模型 3GPP-SCM]({{< ref "/posts/using_scm_channel_model.md">}})"一文中提到了WINNER模型是诺西牵头搞的原3GPP SCM的增强版，关于WINNER模型的官方介绍参见：[http://www.ist-winner.org/](http://www.ist-winner.org/)

考虑到篇幅，官方将winner模型的文档分成上下两册。上册为主，着重介绍信道模型结构和参数，下册为辅，详细记录了信道测量和分析。另有一份关于matlab实现的说明文档。

winner模型是一种geometry-based、stochastic模型，自scm发展而来。其**物理参数采集自欧美的城市、乡镇或农村**。

在仿真时：信道参数根据一定的统计分布随机生成；天线图样从模型中剥离，由用户决定；信道实现由详细的小尺度参数（delay、power、AoA、AoD）通过模拟多径叠加完成。这种多径叠加得到信道的方式体现了geometry-based，在MIMO规模较大时，仿真复杂度反而要比correlation-based方法低（参见文献[^1]的3.6.3节）。这种叠加造成了：1、天线元间的信道相关性；2、依赖于地理环境的多普勒频谱的瞬时衰落（不太理解）。

winner模型框架又可以细分为：generic channel model（GCM）、 clustered delay line（CDL）等。主要特性通过GCM介绍，CDL作为链路级仿真时减小计算量所使用的替代模型，和Tapped Delay Line模型类似。

GCM有三个层次的随机性。从上到下依次为：
- 大尺度参数（LSP）根据分布函数表随机生成（LSP的空间相关性参见文献[^1]的3.3.1节）；
- 小尺度参数根据大尺度参数和分布函数表随机生成；
- 各散射体的初相位随机生成。

从多径角度看信道构成，是`rays–>paths(clusters)–>links`。从时间上看信道构成，是`drops–>segments`。每个segment中的LSP是固定的。而每个drop中除了rays的相位，其他参数也都是固定的。drops之间一般彼此不相关，除非选择使用time evolution。time evolution的实现目前只提供一种Basic Method：在时间上划分出若干子区间，逐次用新cluster替换掉旧cluster。

至此，模型完全非随机了。

如果只看文献[^1]，其所介绍的特性是明晰的、周全的。但目前的matlab实现代码提供的和文献[^1]中规定的尚有出入，使用者必须亲自实践才能略知一二。而且文献[^2]中涉及的数据结构、参数设置等话题过于庞杂，故以后有时间再续。被频繁关注的Antenna Field Pattern、扇区化、布局等话题在文献[^2]中都有详细的介绍。下面列举一些实践中让我读了文档后仍感到困惑的地方：

- 将100MHz带宽通过DownScaling转化成20MHz的具体步骤？matlab版本的代码实现支持此功能吗？
  > 文献[^2]中没有讲到DownScaling

- DelaySamplingInterval的设定有哪些讲究？与系统带宽有关吗？
  > 从文献[^1]中看，此参数只在DownScaling中被提及。文献[^2]中对此参数的解释是：DelaySamplingInterval determines the sampling grid in delay domain. All path delays are rounded to the nearest grid point. It can also be set to zero.

- TimeSamplingInterval的设定受SampleDensity的影响，设定上有什么讲究？
  > 不详。

- Drop和Segment等效于多少个sample？（有时为了研究100个sample能不能将信道遍历，需要知道这个细节）
  > 可以在文献[^1]的表4.5中查到Segment的折算后的长度。对于drop，其说法是：
  > 
  > When obtaining channel parameters quasi-stationarity has been assumed within intervals of 10-50 wavelengths. Therefore we propose to set the drop duration corresponding to the movement of up to 50 wavelengths. In a simulation, the duration of a drop can be selected as desired. It is a common practice to use drops in the simulations…
  > 
  > 但文献[^2]中并未对这一要求是怎么实现的加以说明，也没有明确给出drop和segment和sample的对应关系。所以samples之间的相关性和遍历性一概无从得知了。

- 天线设置中有个rotation的参数，[Rotx，Roty，Rotz]的含义是？
  > 根据单位是rad，通过网上搜集资料，最为接近的理解来自[这里](https://www.siggraph.org/education/materials/HyperGraph/modeling/mod_tran/3drota.htm)。后来又翻出来rotate_vector.m这个函数，总算看明白了，同网页上讲的一致。



[^1]: IST-WINNER D1.1.2 P. Kyösti, et al., “WINNER II Channel Models”, ver1.1, Sept. 2007. Available: [https://www.ist-winner.org/WINNER2-Deliverables/D1.1.2v1.1.pdf](https://www.ist-winner.org/WINNER2-Deliverables/D1.1.2v1.1.pdf)

[^2]: L. Hentilä, P. Kyösti, M. Käske, M. Narandzic, and M. Alatossava. (2007, December.) MATLAB implementation of the WINNER Phase II Channel Model ver1.1 [Online]. Available: [https://www.ist-winner.org/phase_2_model.html](https://www.ist-winner.org/phase_2_model.html)