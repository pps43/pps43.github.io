---
title: "Agentic Engineering (2)"
date: 2026-05-23T18:18:13+08:00
hideSummary: false
tags: ["AI", "软件工程"]
---

上一篇 [Agentic Engineering (1)]({{< ref "posts/notes_on_agentic_engineering_patterns_part_i.md" >}}) 主要写了一些原则性的理解，这篇继续记录Simon Willison一些启发性的实践和用法。个人最感兴趣的放在最前。

# 让 agent 把原生工具编译到 WASM

还有一个很 inspiring 的用法：让 agent 帮你把 native tools 编译到 WebAssembly，然后做成网页工具。

Simon 有一篇很好的案例：[Using coding agents to make command-line tools available in a web browser](https://simonwillison.net/guides/agentic-engineering-patterns/gif-optimization/)。大意是把 `gifsicle` 这样的命令行工具搬到浏览器里，做成一个在线 GIF 优化工具。

这个模式的价值是：

- 很多成熟 native tools 已经解决了核心算法问题；
- 网页 UI 让普通用户也能使用；
- WASM 让它们可以在浏览器本地运行，避免服务器算力开销；
- agent 擅长这个工作流；

这里的 heavy-lifting 往往不是算法本身，而是构建链路：找编译参数、处理 Emscripten、接文件输入输出、设计 UI、写 demo、测浏览器兼容性。coding agent 很适合在这些工程细节里反复试错，因为每一步都可以通过 build、页面操作和样例文件验证。

# 让 agent 用动画解释算法

另一个很实用的用法，是让 agent 构建一个 animated HTML 来解释算法。

很多算法用文字解释会显得很抽象，但一旦变成可以播放、暂停、单步执行的动画，理解成本会下降很多。比如：

- 排序算法中元素如何交换；
- BFS/DFS 如何展开 frontier；
- Dijkstra 如何更新距离；
- 动态规划表格如何被填充；
- 几何算法中向量、交点、包围盒如何变化。

这类任务非常适合 agent，因为它通常不需要进入复杂工程体系，只要生成一个单文件 HTML，带一点 CSS、Canvas 或 SVG，再加上基础交互就够了。人提供算法和想看的重点，agent 负责把解释变成可视化。

这也是一个很好的学习工具：让 agent 先写动画，再让它根据动画讲解算法，甚至让它故意加入几个边界 case。相比静态文章，交互页面更容易暴露自己到底有没有理解。

# 让 agent 更好的讲解代码

作者写了一个工具增强 agent 的分析产物：[Showboat](https://github.com/simonw/showboat)。其用处在于：它可以帮助 LLM 把真实代码片段嵌入到文档里，而不是靠模型凭记忆复述。这样能显著降低幻觉和细节错误的风险。

一个可以直接复用的阅读并分析源码的 prompt：

```text
Read the source and then plan a linear walkthrough of the code that explains how it all works in detail. Then run "uvx showboat --help" to learn showboat - use showboat to create a walkthrough.md file in the repo and build the walkthrough in there, using showboat note for commentary and showboat exec plus sed or grep or cat or whatever you need to include snippets of code you are talking about.
```

`linear walkthrough`让agent安排一条阅读路径：入口在哪里，数据怎么流动，核心抽象是什么，哪些文件只是 glue code，哪些地方是风险点。


# 让 agent 参考另一个代码库

> Telling agents to use another codebase as reference is a powerful shortcut for communicating complex concepts with minimal additional information needed in the prompt.

有些东西很难用自然语言说清楚，比如“做一个类似 X 的插件系统，但更轻量”“按照 Y 项目的风格组织 CLI 命令”“参考 Z 的错误处理方式”。这时与其写很长的 prompt，不如给 agent 一个真实 repo 作为 reference。

但要加一个要求：

> Clone the reference repo to /tmp.

原因很简单：不要让参考代码混进当前项目，也不要让 agent 在提交时把无关 repo 一起带进去。`/tmp` 是一个很好的隔离区。可以让 agent 读取、比较、摘录设计模式，但最终修改只发生在当前工作区。

这个技巧尤其适合这些情况：

- 你想复刻某个项目的交互结构，但实现完全不同；
- 你想沿用某个工具的命令行体验；
- 你想学习一个成熟项目如何组织测试、文档、配置；
- 你需要让 agent 快速理解一种“审美”，而不是解释一堆抽象形容词。

当然，参考不是复制。更好的 prompt 是要求 agent 先总结 reference repo 的相关设计，再说明哪些会借鉴、哪些不会借鉴、为什么，然后才开始实现。


# 让 agent 关注测试方法

简单一句话就能让 agent 在实现功能时激发被深度训练出的TDD方法论：

> Use red/green TDD.

也就是先写一个失败测试，确认它真的失败，再实现代码让测试通过。这个约束会迫使 agent 把“完成”的定义落到可验证的行为上，而不是只写出看起来合理的实现。

另一个简单但有效的 prompt 是：

> First run the tests.

这有助于 agent 先通过测试代码了解当前系统的状态、注意点等等，在实现功能时更加审慎。

除了通过命令行测试输入输出，如果项目包含交互式 Web UI，`Playwright` 是目前最强工具没有之一。它不仅能点按钮、填表单、验证 DOM，还能截图、模拟移动端、等待网络空闲、检查视觉布局。对于 agent 来说几乎完成了闭环迭代。



# 尝试自己从零写一个 agent

可以帮助你快速深入理解 coding agent 怎么工作。

> LLM + system prompt + tools in a loop. 

最小版本可以非常朴素：

1. 准备一个 system prompt，说明 agent 的角色、约束和输出格式；
2. 把用户目标、历史消息、工具定义一起发给 LLM；
3. 如果模型要求调用工具，就由宿主程序执行工具；
4. 把工具结果追加回上下文；
5. 循环，直到模型给出最终答案，或者触发最大轮数、超时、权限限制。

真正做过一遍之后，会发现很多听起来抽象的概念（reasoning, tool-calling, token-caching, subagent, etc）都是为了解决对应的实际问题。从而意识到 agent 不是一个黑箱，而是可以可设计、调试、改进的系统，在优化Harness层的时候也更游刃有余。




# 小结

这些 use cases 背后其实是同一个思想：不要只把 coding agent 当成“自动写代码”的机器，而用于全面提升全流程和认知。