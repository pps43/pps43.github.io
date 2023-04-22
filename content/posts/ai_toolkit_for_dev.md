---
title: "Generative AI Toolkit (4.21)"
date: 2023-03-26
hideSummary: false
#draft: true
tags: ["AI"]
# cover:
#     image: "<image path/url>" # image path/url
#     alt: "<alt text>" # alt text
#     caption: "<text>" # display caption under cover
#     relative: false # when using page bundles set this to true
#     hidden: true # only hide on current single page
---


> "There are decades where nothing happens; and there are weeks where decades happen."
> ― Vladimir Lenin

基于大语言模型的AI在这个月带给人们的感受，用列宁的这句话概括再贴切不过了。作为普通人，去拥抱这些AI工具，就像会使用智能手机和搜索引擎；对人类来说，就像学会用电，学会用火。


```mermaid
%%{init: { 'logLevel': 'debug', 'theme': 'dark' } }%%
timeline
    title Era of AI comes in  2023
    2-7: Microsoft New Bing
    3-12: Open AI ChatGPT 90% cheaper
    3-15 : Open AI GPT-4
    3-16 : Microsoft Copilot : Midjourney V5 : Google PaLM API
    3-21 : Adobe FireFly : Nvdia GTC
    3-22 : Github Copilot X : Google Bard
    3-24 : Open AI ChatGPT Plugins
```

------

> 更新：4月以来，AI应用的新概念、新架构、新产品如寒武纪大爆炸一般涌现（[AutoGPT](https://github.com/Significant-Gravitas/Auto-GPT) 首当其冲），非人力所能穷举。
> [这个网站](https://supertools.therundown.ai/)收录了大量AI工具，本文也会持续更新笔者常用、觉得好用的工具。

# For General Task

- 🔥Open AI's [ChatGPT](https://chat.openai.com/auth/login), and Plugins. GPT3.5 is free to use.
- 🔥Microsoft's [NewBing](https://www.bing.com/new). It's said to be powered by GPT4 (internal version).
- Google's [Bard](https://bard.google.com/).

 > After trying so many LLM, ChatGPT is still the best one to be professional and smart. But I still prefer asking different models to get different point of view.

# For Document
- Edge + NewBing. Explain any webpage (including PDF) side by side.
- [ChatDoc](https://chatdoc.com/)/[ChatPDF](https://www.chatpdf.com/), upload PDF and analyze.
- Microsoft's Copilot.

# For Code

- Github's [Copilot X](https://github.com/features/preview/copilot-x), exsiting Copilot costs $10/mo after 60d trial.
  - Using [Copilot for Docs](https://githubnext.com/projects/copilot-for-docs) to learn a SDK/framework/API.
  > The gist is, to describe a single-responsibility function to let AI generate, rather than a function with long description of chained operations.

- [Cursor](https://www.cursor.so/), or CodeCursor(vscode plugin), analyze opened document/code, currently FREE.

# For 3D/2D Art
- 🔥[Stable-Diffusion (SD) web-ui](https://github.com/AUTOMATIC1111/stable-diffusion-webui), totally free and opensource, run model locally on PC.
  - Download/Share models on [civitai](https://civitai.com/content/guides/what-is-civitai)/[Hugging face](https://huggingface.co/)
  - Use [ControlNet](https://stablediffusionweb.com/ControlNet) ([Github](https://github.com/lllyasviel/ControlNet) )to add more controll on specific SD model.
  - Use [LoRA (Low-rank adaption)](https://huggingface.co/docs/diffusers/training/lora) to train faster with less memory.
  - Use [Text Inversion](https://huggingface.co/docs/diffusers/training/text_inversion) to train with amazingly small output.
  - Use [DreamBooth] to train if you need to be really expressive.

- 🔥[Midjourney](https://www.midjourney.com/home/),  famous for its artistic style, ~~25 times FREE try~~.
- Adobe's [Firefly](https://firefly.adobe.com/)
- Open AI's [DALL-E-2](https://labs.openai.com/), generates image with natural language and long prompts, but limited-access and less control.
- Bing's [Image Creator](https://www.bing.com/images/create), generate image with natural language, and free to try.

# For Music
- [Mubert](https://mubert.com/)
- [Soundraw.io](https://soundraw.io/create_music)

------
# Want more power?

If you want to:
- train your own AI based on these models
- know the strength and weakness of current AI models
- know why & how Generative AI works, mathematically

Here are my personal ideas:
- For text, play with [LLaMA](https://github.com/facebookresearch/llama)/[llama.cpp]((https://github.com/ggerganov/llama.cpp) ), or its fined tuned version [Alpaca](https://github.com/tatsu-lab/stanford_alpaca)/[Alpaca-LoRA](https://github.com/tloen/alpaca-lora). For image, play with [Stable-Diffusion](https://github.com/Stability-AI/stablediffusion). They can run on PC/Mac.
- Weakness of current LLM models: math; chain of decision. But they are improving.
- ["Dive into Deep Learning"](https://d2l.ai/) by 李沐。中文版《[动手学深度学习](http://zh-v2.d2l.ai/index.html)》