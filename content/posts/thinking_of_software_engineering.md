---
title: "Facts and wisdom on software engineering"
date: 2023-10-22
draft: false
hideSummary: false
tags: ["DevOps"]
# cover:
#     image: "<image path/url>" # image path/url
#     alt: "<alt text>" # alt text
#     caption: "<text>" # display caption under cover
#     relative: false # when using page bundles set this to true
#     hidden: true # only hide on current single page
---

> 本篇着重于收集编程和软件工程中的洞察和智慧，大部分来自业内有名的前辈，例如`Joel Spolsky`, `John Carmack`等等，也有个人的思考。当我这些年经历过不同行业、国内外不同公司、不同技术栈后，在真正认同这些观点对于产能带来多大提升的同时，也发现它们是多么容易被忽略。总结在这里，常看常新。

---

# About Coding

## I
> **It’s harder to read code than to write it**. 
> 
>  -- Joel, "[Things You Should Never Do, Part I](https://www.joelonsoftware.com/2000/04/06/things-you-should-never-do-part-i/)"

在你抱怨项目代码是一座屎山时，很多时候是因为阅读代码的固有难度带来的负面体验。固有难度是指，你不了解突然插入的这段代码的意图（多数是为了修某个bug），也很难从全局对一长串代码加以认识。不少人因此而畏惧代码重用而复制一部分来形成“自己的特供版本”，殊不知这才是形成真正屎山的源头。

也因此，为了减少屎山的产生，我们要不遗余力的**从各个层面降低阅读代码的门槛**。
- 留下注释说明新增这段代码的意图，从你开始！
- 使用更好的代码浏览工具。这一点Rider的实时搜索要好于VS。SciTools出品的Understand在某些时候也可以尝试用于理解宏观结构。2023年有了GPT，结合AI来理解代码更是值得尝试的方法。后面还会提到，看代码的同时结合调试更能事半功倍。
- 优化编程语言，比如引入更简洁且代码表达力强的语法，即使只是语法糖！在某些情况下可以考虑创造DSL（微信支付团队通过实践证实了这种做法是高效且可靠的）。
- 适当重构业务逻辑。通过多人多版本的迭代，一段代码的表达力可能变得支离破碎。即使只是通过重命名让变量和函数表意更清晰，也是有意义的。你很可能会因此而发现不合预期但尚未发现的bug。庞大如微软的Exchange的一些底层服务，重构和翻新也很积极——这些需要其他基建配合，中小公司需要评估性价比。


## II
> **Use as many tools as you can find to understand the code.** You should do experiments on the system, by adding logs, using asserts as dynamic comments, and using debugger as your companion (even if writing new code).
> 
> --Carmack, interview with Lex Fridman.
>
> **Go through code as it's running.** You can see the entry point, the entire memory, call stack/threads.... the architecture, compiled template code ( C++ template is hard!)... In a word, Never read like a book.
>
> --Cherno, in “[BEST WAY to read and understand code](https://www.youtube.com/watch?v=XTZVbmz7LpY)”.

如何阅读代码？你所能做的最差的方法就是像读一本书那样去阅读。人生苦短。八仙过海，各显神通。总之就是不羞于同时以各种方式来更好的了解代码。有一点是共同的：一边调试，一边阅读！

其他常见的方法列举如下，面对不同类型的项目和不同的目的加以斟选：
- 尝试修改源码后跑UT
- 尝试实现源码的功能，然后对比着看别人的思路
- 查资料，找关键类或接口
- 看提交记录，了解一段功能的演变
- 看设计文档（如有）

## III

> **It's so important to quickly understand everything going on here (in code base).** 	Thus `C` has advantage. If you know `C` you can jump in and not have to learn what paradigms they are using because the simplicity of c. But lisp, you have to restructure your mind before you dare to touch some code... Almost every case I've seem when people mixed languages on their project, that's an mistake. I would stay just in one language so every body can work on it. 
> 
> --Carmack, interview with Lex Fridman.

语言特性的简洁、框架的简洁会带来一个优势：任何人都可以jump in去了解并解决问题。人的脑容量是有限的，解决问题前需要做的准备工作、或者说了解的上下文越少越好。与此同理的一条是：项目采用单一语言开发的优势可能比想象中还要大。

# Beyond Coding

## I

> My biggest point was **everything we are doing really should flow from user value** (by metrics + vision). Don't just be pround of some architecture or specific tech or specific code. It's a fun puzzle game but that really should not be a major motivator for you.
>
> --Carmack, interview with Lex Fridman.

以游戏开发者为例说明。在研究技术的同时，不要忘记产品价值才是目的。不要止步于对某一项技术过于崇拜或炫技，那只是一些智力游戏。这话从卡神的口中说出来，相信没有任何一个资深游戏从业者有底气否认了。