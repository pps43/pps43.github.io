---
title: "HTML在AI时代的复兴"
date: 2026-05-14
draft: true
hideSummary: false
tags: ["AI", "杂谈"]
---

> 软件不只是最终产品，也可以是过程产物。AI 让我们重新拥有了为一次性问题制作一次性软件工具的自由。

[Simon Willison](https://simonwillison.net/) 最近转发并评论了 Thariq Shihipar 的一篇文章：[Using Claude Code: The Unreasonable Effectiveness of HTML](https://simonwillison.net/2026/May/8/unreasonable-effectiveness-of-html/)。Thariq 的核心判断可以概括成几句话：HTML 的信息密度更高，可读性更好，还能把文档变成交互界面，并且支持多媒体数据嵌入到一个文件中，因此更容易用一个链接分享。

而 Simon 大佬是 Django, Datasette 的作者，他提到，自己过去习惯让模型输出 Markdown：在 GPT-4 时代，只有8k的上下文窗口让 Markdown 省token省的非常有价值。但随着 LLM 动辄 1M 的上下文窗口成为标配，他最近开始重新让 LLM 直接生成 HTML 页面，**尤其是当输出主要是给人读、给他人分享的时候**。

> 提示词举例：让 Claude Code 为一个 PR 生成 HTML artifact，重点解释 streaming/backpressure 逻辑，把真实 diff 渲染出来，加上行边注释，并按严重程度给发现的问题着色。


Simon 在 2025 年 12 月写过一篇更偏实践的文章：[Useful patterns for building HTML tools](https://simonwillison.net/2025/Dec/10/html-tools/)。他把这类东西称为 `HTML tools`：单个 HTML 文件，内联 CSS 和 JavaScript，必要时从 CDN 加载少量依赖，不用 React，不用构建步骤，保持几百行规模。过去两年他做了 150 多个这样的工具，大多由 LLM 编写。

关于 `HTML tools` 的一些启发：

- 单文件优先。一个文件最容易复制、粘贴、托管、分享，也最容易让 Agent 整体读懂。
- 尽量不要 React。不是 React 不好，而是 JSX 和构建步骤会让一次性工具变重。
- 依赖走 CDN。PDF.js、Tesseract.js、Pyodide、图表库、diff 库，都可以按需引入。
- 让复制粘贴成为主交互。输入从剪贴板来，输出一键复制回编辑器、PR、聊天窗口或下一轮 prompt。
- 状态能放 URL 就放 URL，太大或涉及 API key 时用 localStorage。
- 多利用浏览器本地能力：`<input type="file">` 可以直接读取本地文件，Canvas 可以处理图像，Blob 可以生成下载文件，CORS 友好的 API 可以直接取数据。
- 用 WebAssembly 和 Pyodide 扩展边界。Python、OCR、SQLite、压缩、图像处理，都可能在浏览器里完成。
- 保留 prompt 和 transcript。它们不仅是溯源材料，也是下一次 remix 的原料。



# 代价是什么

HTML 的代价也很明显：生成更慢，token 更多，Git diff 的结果有更多噪声，解决冲突也更痛苦了。所以我更愿意混合使用：

- Markdown 适合“我要和这份文本共同写作”。
- HTML 适合“我要理解、审查、演示、比较、调参、分享，并把结果带回下一轮协作”。

小结一下。单页 HTML 的复兴，某种程度上就是一种人文关怀的复兴。有人说，将来的软件系统大部分是为 agent 而不是为人类而设计，所以 UI 将不再那么重要。也许从短期看是的，但我还是相信 UI 始终有一席之地，科技应当为人服务，而人是视觉动物。



# 一些提示词的例子

这篇文章最后，我想把它落到几个可以直接试的 prompt 上：

```text
请为这个 PR 生成一个单页 HTML 报告。渲染关键 diff，给重要行加旁注，用颜色区分风险等级，并画出数据流/调用路径图。
```

```text
我还没想好这个功能的交互方向。请生成一个 HTML 文件，给出 5 种明显不同的设计方案，并排展示，每种方案旁边写清楚它牺牲了什么、强化了什么。
```

```text
请阅读这个模块，生成一个单页 HTML 教程：包含整体架构图、核心代码片段的逐行解释、常见误区，以及一个可以切换输入参数观察输出变化的小交互。
```

```text
请为这组任务做一个临时 HTML 编辑器：支持拖拽排序、分组、添加备注，最后提供“复制为 Markdown”和“复制为 JSON”两个按钮，方便我把结果带回下一步工作。
```

# 一些AI输出的html的例子

1. 动画效果演示
{{< html-revival >}}

