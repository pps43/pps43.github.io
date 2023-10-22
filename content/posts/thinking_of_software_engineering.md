---
title: "Facts and wisdom on software engineering"
date: 2023-10-22
draft: true
hideSummary: false
tags: ["DevOps"]
# cover:
#     image: "<image path/url>" # image path/url
#     alt: "<alt text>" # alt text
#     caption: "<text>" # display caption under cover
#     relative: false # when using page bundles set this to true
#     hidden: true # only hide on current single page
---

> 本篇着重于收集编程和软件工程中的洞察和智慧，以及对应对方法的思考。大部分来自业内有名的前辈，例如`Joel Spolsky`, `John Carmack`等等。当我真正认同这些观点对于工程实施效率带来多大提升的同时，也发现它们是多么容易被忽略。总结在这里，常看常新。

---

# 名句解读

## About Code

> **It’s harder to read code than to write it**. 
> 
>  -- Joe, "[Things You Should Never Do, Part I](https://www.joelonsoftware.com/2000/04/06/things-you-should-never-do-part-i/)"

在你抱怨项目代码是一座屎山时，很多时候是因为阅读代码的固有难度带来的负面体验。固有难度是指，你不了解突然插入的这段代码的意图（多数是为了修某个bug），也很难从全局对一长串代码加以认识。不少人因此而畏惧代码重用而复制一部分来形成“自己的特供版本”，殊不知这才是形成真正屎山的源头。

所以我们要不遗余力的从各个方面降低阅读代码的门槛。
- 优化编程语言，比如引入更简洁且代码表达力强的语法，即使只是语法糖也会有用！在某些情况下可以考虑创造DSL（微信支付团队就是这样提高产出的同时力保安全的）。
- 等效重构，比如让变量和函数表意更清晰。甚至会因此而发现不合预期但尚未发现的bug。
- 留下注释说明新增这段代码的意图，从你开始！
- 使用更好的代码浏览工具。这一点Rider的实时搜索要好于VS。SciTools出品的Understand在某些时候也可以尝试用于理解宏观结构。2023年有了GPT，结合AI来理解代码更是值得尝试的方法。当然看代码的同时结合调试更能事半功倍，后面还会谈到。

---
> **It's so important to quickly understand everything going on here (in code base).** 	Thus c has advantage. If you know c you can jump in and not have to learn what paradigms they are using because the simplicity of c. But lisp, you have to restructure your mind before you dare to touch some code... Almost every case I've seem when people mixed languages on their project, that's an mistake. I would stay just in one language so every body can work on it. 
> 
> --Carmack, interview with Lex Fridman.

语言特性的简洁、框架的简洁会带来一个优势：任何人都可以jump in去了解并解决问题。人的脑容量是有限的，弄懂问题时需要做的准备工作、或者说了解的上下文越少越好。这也导出一个观点：项目采用单一语言开发的优势可能比想象中还要大。

---
> **Use as many tools as you can find to understand the code.** You should do experiments on the system, by adding logs, using asserts as dynamic comments, and using debugger as your companion (even if writing new code).
> 
> --Carmack, interview with Lex Fridman.
>
> **Go through code as it's running.** You can see the entry point, the entire memory, call stack/threads.... the architecture, compiled template code ( C++ template is hard!)... In a word, Never read like a book.
>
> --Cherno, in “[BEST WAY to read and understand code](https://www.youtube.com/watch?v=XTZVbmz7LpY)”.

如何阅读代码？你所能做的最差的方法就是像读一本书那样去阅读。人生苦短，代码如山。八仙过海，各显神通。总之就是不羞于以各种道具来更好的了解代码。以及有一点是共同的：一边调试，一边阅读！

其他常见的方法列举如下，面对不同类型的项目和不同的目的加以斟选：
- 尝试修改源码后跑UT
- 尝试实现源码的功能，然后对比着看别人的思路
- 查资料，找关键类或接口
- 看提交记录，了解一段功能的演变
- 看设计文档（如有）

---

## Beyond Coding

> My biggest point was **everything we are doing really should flow from user value** (by metrics + vision). Don't just be pround of some architecture or specific tech or specific code. It's a fun puzzle game but that really should not be a major motivator for you.
>
> --Carmack, interview with Lex Fridman.

# 推荐阅读

- https://www.joelonsoftware.com/