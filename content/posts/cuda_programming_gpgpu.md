---
title: "GP-GPU编程：CUDA介绍"
date: 2015-02-28
#draft: true
#tags: ["a","b"]
# cover:
#     image: "<image path/url>" # image path/url
#     alt: "<alt text>" # alt text
#     caption: "<text>" # display caption under cover
#     relative: false # when using page bundles set this to true
#     hidden: true # only hide on current single page
---

> 本文简要介绍了利用GPU进行通用计算的历史和方法，以NVDIA家的CUDA架构为例，介绍GPGPU编程的重要概念和简单实践。希望能够在浩如烟海的网络资料之外，给像我这样的初学者提供一些清晰的参考。CUDA C的学习资料在文末给出，NVDIA的官方文档干净友好，听说社区生态也堪称典范，好感度+1。

# 简介
## 历史

近30年来，提升CPU计算性能的主要方法就是提升时钟频率。此方法在目前的硅基半导体工艺下已进入瓶颈，转而用“并行”来提升计算性能（例如多核）。

自1980年代起，一家名为Silicon Graphics的公司在业界推广OpenGL用于渲染三维图像，获得了广泛支持。NVIDIA、ATI等公司也开始对计算机3D图形计算加速进行研究，并将此硬件设备称为“GPU”以区别于传统的CPU。GPU由于其结构特点，具备强大的并行计算能力。（1999年，GPU的始祖GeForce 256横空出世。2002年完整支持DX9的王者Radeon 9700Pro正式开启了GPU的众神时代，Age of Gods. 显卡爱好者可以看看此文：[视觉时代的回响 GPU十年历史追忆](https://news.mydrivers.com/1/276/276298.htm)。）

在很长一段时间里，利用GPU进行渲染是通过OpenGL和DirectX进行的。人们发现传给GPU进行像素点渲染的颜色、纹理坐标等信息其实可以是任何数据，即GPU可以用于通用计算（`General Purpose GPU`）。然而当时的GPU有着难以在显存任意位置进行读写、GPU本身也不具备浮点计算能力、GPU难以debug等严重缺点。更糟的是，必须使用一种叫“shader language”的编程语言，利用OpenGL和DirectX提供的API对GPU进行操作。利用GPU进行通用计算的巨大的学习成本，使得研究人员望而却步。

对于上述硬件缺点，NVDIA推出了基于CUDA架构的GPU，如2006年上市的高端卡[GeForce 8800 GTX](https://en.wikipedia.org/wiki/GeForce_8_series#GeForce_8800_Series)；对于上述软件缺点，NVDIA在C的基础上进行扩展，并在GeForce 8800 GTX上市后几个月后公布CUDA C语言及相应的编译器用于GPGPU的编程。

## 现状

- 性能上，就个人电脑来说，目前的顶级CPU，主频也很难上4GHz，而核心数量很难达到16个（计算能力约64GFLOPS）。而稍微高端点的GPU早就超过3000GFLOPS（以GTX 690为例，核心频率约1GHz，3072个核心）。国外已有建立在Tesla k40集群上的超算中心。下面一图以蔽之吧。
- GPGPU应用：耳熟能详的GPU应用诸如：视景仿真、视频渲染。以及石油勘探、环境科学、航空航天和流体力学计算等领域。举几个典型且较新的例子：医疗成像。智能汽车。深度学习。

# 概念
## 硬件

- Device, Host
    - CPU和内存（host memory）是host，GPU和显存（device memory）是device。

- SM, SP
    - SM：Stream-Multiprocessor，俗称“大核”。SP：Stream-Processor，俗称“小核”。

- Global memory
    - 就是通常意义上的显存。

- Shared memory
    - 为同一个block中若干threads所共享。 存取非常快，可看作user-managed cache。CUDA C中的关键词：`__shared__`

- Constant memory

## 并行层次

并行是一个概念，可以用在不同的层面。CUDA中从高到低有以下几个层次：

> stream -> grid -> block -> (warp) -> thread.

并行层次与存储层次是对应的。
- thread
    - 并行的基本单位。同一个block中的thread通过shared memory共享数据。同一个block中的thread通过__syncthreads()同步，具体见下一节。
- warp
  - In the world of weaving, a warp refers to the group of threads being woven together into fabric. In the CUDA Architecture, a warp refers to a collection of 32 threads that are “woven together” and get executed in lockstep. At every line in your program, each thread in a warp executes the same instruction on different data.
  - When it comes to handling constant memory, NVIDIA hardware can broadcast a single memory read to each half-warp. A half-warp—not nearly as creatively named as a warp—is a group of 16 threads: half of a 32-thread warp. If every thread in a half-warp requests data from the same address in constant memory, your GPU will generate only a single read request and subsequently broadcast the data to every thread. If you are reading a lot of data from constant memory, you will generate only 1/16 (roughly 6 percent) of the memory traffic as you would when using global memory.
  - 最大支持512个threads。只用一个block不能充分利用GPU的资源。
- grid
    - We call the collection of parallel blocks a grid. 同一个grid中的所有thread都执行相同参数的核函数。
- stream
    - A stream is a sequence of commands (possibly issued by different host threads) that execute in order. this behavior is not guaranteed and should therefore not be relied upon for correctness (e.g., inter-kernel communication is undefined). 不同stream可执行不同参数的核函数。

## 通信机制
- Inter-thread level
    - shared memory. 要求是同一个block中的threads。
    - 内置函数 `__syncthreads()`. 作用是当某个线程执行到该函数时，进入等待状态，直到同一线程块（Block）中所有线程都执行到这个函数为止。
    - 原子操作。可以跨block。

- Inter-block level
- Inter-stream level

# 实践

## 环境搭建
采用64位win7 + visual studio 2013 + cuda toolkit 6.5 。早期的cuda环境需要下载cuda toolkit、drivers、cuda SDK，现在全部打包到一起了。Cuda Toolkit 6.5下载地址：[官网](https://developer.nvidia.com/cuda-downloads)
关于软硬件兼容性的问题，2010年的老显卡（指笔者的Quadro NVS 3100M 。怎么样，没听过吧，说多都是泪……此卡介于G210M~G310M之间）仍可以安装 6.5 版，基本可以放心下载。
完整的搭建过程参考官方文档。

## 常见问题
- warning: C4819
    - 只需在项目properties->configuration properties->CUDA C/C++的Additional options那里加一行：-Xcompiler /wd4819

- error：expected and expression (不能识别`<<<`)
    - 首先检查出错的代码文件的属性中，item type是不是CUDA C/C++。如果是，则没有大碍，只是VS的C/C++文本编辑器认为有错，编译器可以通过。这里提到一种自定义宏的方法消除这个小bug。

- error：addKernel launch failed: invalid device function

## 常用工具
- deviceQuery.exe
    - CUDA Samples中自带的检测显卡信息的工具。可以方便的查看显卡的基本信息。如：
        - 型号
        - Compute capability
        - 核心数量（大核SM、小核SP）
        - 时钟频率
        - global memory
        - const memory
        - shared memory/block
        - 最大threads、block、grid限制等

- bandwidthTest.exe
    - CUDA Samples中自带的测试Device和Host之间带宽的工具。还有个重要功能是搭建CUDA环境后验证是否正常运转（看最后Result是否为PASS）

- Visual Profiler