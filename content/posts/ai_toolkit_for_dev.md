---
title: "Generative AI Toolkit (3.29)"
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


# For General Task

- Open AI's [ChatGPT](https://chat.openai.com/auth/login)
- New Bing (Powered by GPT4)
- Open AI's ChatGPT Plugins
- [Hugging face](https://huggingface.co/)'s models/dataset/research

# For Document
- Microsoft's Copilot
- Notion AI
- [ChatDoc](https://chatdoc.com/)/[ChatPDF](https://www.chatpdf.com/), upload PDF and analyze.

# For Code

- Github's [Copilot X](https://github.com/features/preview/copilot-x), exsiting Copilot costs $10/mo after 60d trial.
  - Using [Copilot for Docs](https://githubnext.com/projects/copilot-for-docs) to learn a SDK/framework/API.
  > The gist is, to describe a single-responsibility function to let AI generate, rather than a function with long description of chained operations.
- [Cursor](https://www.cursor.so/), or CodeCursor(vscode plugin), analyze opened document/code, currently FREE.
- Open AI's CodeX

# For 3D/2D Art
- [Stable-Diffusion](https://stablediffusionweb.com/#demo)([Github](https://github.com/Stability-AI/stablediffusion)), an open-sourced model can run on PC.
  - Using [ControlNet](https://stablediffusionweb.com/ControlNet) ([Github](https://github.com/lllyasviel/ControlNet) )to add more control to this model.
  - Find/Share self-tuned models on [civitai](https://civitai.com/content/guides/what-is-civitai)
- [Midjourney](https://www.midjourney.com/home/),  tuned from stable diffusion, famous for its artistic style, 25 times FREE try.
- Open AI's [DALL-E-2](https://labs.openai.com/)
- Adobe's [Firefly](https://firefly.adobe.com/)
- Bing's [Image Creator](https://www.bing.com/images/create)

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
- Weakness of current LLM models: math; chain of decision;
- ["Dive into Deep Learning"](https://d2l.ai/) by 李沐。中文版《[动手学深度学习](http://zh-v2.d2l.ai/index.html)》