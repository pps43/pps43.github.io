---
title: "Agentic Engineering (1)"
date: 2026-05-18T22:26:33+08:00
hideSummary: false
tags: ["AI", "软件工程"]
---

最近读 Simon Willison 的 [Agentic Engineering Patterns](https://simonwillison.net/guides/agentic-engineering-patterns/)，颇有收获。这里先写第一部分，主要有：agent 到底是什么，代码变便宜之后人的工作还剩什么，以及我们应该怎样调整自己的开发习惯。

一言以蔽之：agentic engineering 不是让 AI 替代工程判断，而是把工程判断从“亲手写每一行代码”迁移到“设计目标、工具、约束、验证和反馈循环”。

# 什么是 agent

Simon 有一个非常简洁的定义，我很喜欢：

> Agents run tools in a loop to achieve a goal.

也就是说，一个 agent 本质上是一段软件，它调用 LLM，把你的 prompt 和一组 tool definitions 交给模型；模型决定要调用哪些工具；宿主程序执行工具，把结果再塞回模型；如此循环，直到目标完成或失败。

这个定义把注意力从“模型聪不聪明”转移到“loop 设计得好不好”。目标是否清晰？工具是否足够？权限是否过大？反馈是否可靠？是否有测试、lint、截图、benchmark 之类可验证的信号？这些才是 agent 能不能稳定工作的关键。


# 人还剩下什么工作

> 太多了。

**写代码从来不是软件工程师工作的全部，真正困难的部分一直是推敲写什么代码**。一个软件问题通常有很多种解法，每种解法背后都有取舍：简单性、性能、可维护性、迁移成本、团队熟悉度、可观测性、如何应对未来需求变化。工程师的工作，是在具体上下文中判断哪条路最合适。

agent 没有消灭这部分工作，反而把它放大了。因为当生成代码的成本下降，可行的方案数量更多了。以前我们常常因为“写起来太费时间”而提前排除一些方案；现在这些方案可以并行交给AI打样，于是人的工作变成：

- 定义什么才算“完成”；
- 提供恰如其分的上下文和工具；
- 约束 agent 不要破坏不该碰的东西；
- 判断是否真的解决了问题；
- 把这次工作中的经验沉淀为下次 agent 能复用的规则、文档、测试。

大概是现在说的`Harness`。我们仍然要懂代码，因为我们要判断代码；我们仍然要懂系统，因为我们要判断局部修改会怎样影响整体；我们甚至要更懂验证，因为 agent 特别容易给你呈现“看起来差不多”的成果。而人需要保证结果是有效的，以及持续优化过程。因此AI时代的工程能力肯定不是下降到“会写 prompt”这么简单，而是上升到“会设计可执行的反馈循环”。复杂度不会消失，只会转移。

# 写代码变得容易意味着什么

> 采用 agentic engineering 最大的挑战之一，是接受“写代码变容易”之后带来的后果。

过去开发功能很耗时。宏观上，团队花大量时间做设计、估算、排期，是为了把昂贵的工程师人力用在刀刃上。微观上，工程师每天也会做很多基于时间成本的判断：这个函数要不要顺手重构？这个边界条件要不要加测试？这个调试面板值不值得做？文档要不要更新？很多时候答案并不是“它不重要”，而是“现在不值得花这个时间”。

coding agents 改变了成本结构。把代码敲进电脑这件事，正在接近免费。更麻烦的是，并行 agent 让一个工程师可以同时在多个分支里实现、重构、测试、写文档。**这确实破坏了专业软件开发者长期形成的直觉：以前不值得做的事，现在还不值得吗**？

> New code is almost free, good code is not.

所谓 `good code`，至少包括：

- 它真的工作，并且没有明显 bug；
- 我们知道它为什么工作，有测试或其他证据；
- 它解决的是正确的问题；
- 它足够简单，避免不必要的复杂度；
- 它覆盖错误路径，而不只是 happy path；
- 它有合适的文档；
- 它的设计不会阻碍未来合理的变化；
- 它满足场景需要的各种 `-ilities`，例如 reliability、security、accessibility、observability、scalability。

agent 可以帮助我们接近这些目标，但不能替我们承担最终责任。坏消息是，好代码依然贵；好消息是，很多过去因为“太花时间”而不做的质量工作，现在变得便宜了。

> 一个很实用的习惯：当本能告诉你“这个不值得做”时，不妨开一个异步 agent session 试一下。最坏的情况，无非是十分钟后发现结果不值得合并；但它可能会给你一个可运行的 prototype、一组测试、一个重构分支，或者至少帮你排除一条路。

# 把你实践过的知识积累和再利用

特别喜欢 Simon 的另一条建议：[Hoard things you know how to do](https://simonwillison.net/guides/agentic-engineering-patterns/hoard-things-you-know-how-to-do/)。

软件工程的一大能力，是知道什么事情可能做到，什么事情不太可能做到，并且大概知道应该从哪里下手。**光“听说可以”是不够的。真正有价值的是你见过跑起来具体是怎样的**。一个能跑的 demo、一个最小 repo、一段 benchmark、一份踩坑记录，都会变成未来判断问题时的线索。

过去积累这些 proof-of-concepts 很费时间。现在可以让 LLM 和 coding agent 帮你：让它研究一个小问题，给你一个可运行 demo 和一份报告；让它把某个 API 调通；让它尝试根据某个idea做一个工具……

**然后这些资产会反过来成为 agent 的高质量输入**。例如让 agent 组合两个已经能工作的例子，做出新的东西。这大大增强了模型输出的内容的可靠程度，因为它是建立在你亲自验证过的`building block`之上。

# AI应该帮助我们产出更好的代码

有些悲观的人认为全面使用 agent 会导致团队交付更多低质量代码。这个风险当然存在，但我同意 Simon 在 [AI should help us produce better code](https://simonwillison.net/guides/agentic-engineering-patterns/better-code/) 里的判断：

> shipping worse code with agents is a choice。

意思是，你其实可以选择优化流程避免 agent 让代码质量下降。是 review 不够？测试不够？任务边界太大？prompt 没有说明约束？agent 没有可用的验证工具？还是团队把“能运行”误认为“能交付”？

更何况，agent 真正体现价值的方向之一，就是提高代码质量。有三个场景：

**第一类场景是技术债。**很多技术债并不难，只是耗时间：统一命名、拆分大文件、合并重复逻辑、补错误处理、迁移旧 API、给边界条件补测试。过去这些工作很难排上优先级，因为它们不像新功能那样显眼。现在可以开一个 branch 或 worktree，让 agent 在后台做。如果结果好，就 review 合并；如果差，直接丢掉。质量改进的边际成本下降之后，我们可以对小的 code smell 更不宽容。

**第二类场景是规划阶段**。LLM 不一定能提出天才方案，但它很擅长补齐常见方案。它可以帮助我们避免漏掉显然可行但当时没想到的选项。更重要的是，coding agents 可以直接 prototype，用结果说话。

**第三类场景是复利式工程**。[Compound Engineering](https://every.to/chain-of-thought/compound-engineering-how-every-codes-with-agents) 一文提到一个 `plan-work-review-compound` 循环：每次提交，不只是合并代码，还要复盘这次 agent 协作中哪些指令、约束、测试、上下文有效，并把它们沉淀回代码库或团队流程。这样下一次 agent 不是从零开始，而是在更好的环境里工作。

> 每次 commit 都是一个创造复利的机会。


# 小结

agentic engineering 让软件工程的成本结构彻底的变了。

> Stop playing with cheap code.

要把精力放在打磨以下事项或能力上：
- 定义更清晰的问题；
- 构建可运行的反馈循环；
- 对 good code 的判断；
- 积累验证过的解法；

agent 可以帮我们试更多可能性，但最终需要我们为软件增加真正的价值。
