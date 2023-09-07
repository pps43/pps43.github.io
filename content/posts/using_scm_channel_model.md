---
title: "信道仿真模型 3GPP-SCM"
date: 2014-11-11
# draft: true
hideSummary: false

tags: ["无线通信"]
# cover:
#     image: "<image path/url>" # image path/url
#     alt: "<alt text>" # alt text
#     caption: "<text>" # display caption under cover
#     relative: false # when using page bundles set this to true
#     hidden: true # only hide on current single page
---

在"[衰落信道小议]({{< ref "/posts/thinking_of_fading_channel_models.md">}})"一文中提到了MIMO信道的建模。好的信道模型是以下三个标准的折中：
- 准确性。
- 计算可行。
- 一般性（或者说灵活性）。

3GPP提出的`SCM`（Spatial Channel Model）是符合以上标准的、基于地理的概率模型（Geometry Based Stochastic Model），即使用若干传播路径进行叠加，每条路径的强度和延迟符合一定的随机分布。SCM被WP5小组采用研究室外的MIMO信道，后经其手拓展成SCM-E（E for Extension）模型，再后来WP5还是不满意，便另起炉灶开始了`WINNER`系列信道模型的制定及其实现，这是后话，放在[下一篇]({{< ref "using_winner2_channel_model.md">}})里说。

推荐从SCM开始学习信道仿真。说到底信道仿真就是通过一系列配置，生成某场景下的信道H。接下来的文字认为大家已经看过或正在看以下两个文档：
> [1] 3GPP TR 25.996 V10.0.0, “Spatial channel model for MIMO simulations”, Mar. 2011
> [2] 3GPP TR 25.996, “MATLAB implementation of the 3GPP spatial channel model”, July 2006.

SCM整个模型分为两块分别用于系统级仿真、链路校正。常用的是系统级仿真，即 [1] 第5章的内容。第4章那么长很容易让人找不到要领。

要说的是，SCM信道参数很多，一定要全部掌握而不能依靠默认配置，从而获得似是而非的仿真结果。SCM为了计算方便，对很多参数有假设限定，例如认为用户之间即使靠的很近，也认为Shadow Fading的系数不相关。又例如：AS，DS，SF参数在所有扇区是一致的。天线配置理论上是自由的但实际只可以设定成0.5/4/10倍波长。假设延时抽头数目小于等于6，这就限制了最大带宽。等等。

5.4节给出了计算subpath上的信道增益的原理，这正是前文提到的 Ray-Tracing 方法。

5.5节是一些附加选项，如极化天线、视距信道等等。5.6节讲述了信道系数之间的相关性是怎么选取的。很遗憾，SCM中对inter-cell correlation的处理是给定一个B矩阵，这也表明了SCM是 入门级 的东西。

5.7节，点了一下多小区干扰的建模。

在提供的matlab仿真程序中，对参数 SampleDensity（信道生成使用的采样密度，和多普勒频移参数有关）和Ts（用于将delay从具体时间转化到抽头数，也是通信系统的采样率）说明的不完善，需要自己玩味。另外，最终生成的信道存储在\\(\bf{H}\\)中，是一个五维的矩阵，每一维的具体含义文档里说的很清楚了： \\({\bf{H}}(i,j,k,p,q)\\)表示第\\(q\\)条链路在第\\(p\\)个采样时间点上，从天线\\(i\\)到天线\\(j\\)之间delay为\\(k\\)秒的信道增益。