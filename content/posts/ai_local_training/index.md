---
title: 自己训练AI模型
date: 2026-07-11
tags:
  - AI
hideSummary: false
draft: true
---

> 绝知此事要躬行。我的想法比较简单，通过自己上手摸一遍模型的预训练、微调、蒸馏，实际感受一些关键技术是怎么做的，如多模态、工具调用、SFT、LoRA、DPO。


## 选择训练框架

本地硬件规格为 M2Max + 32G同一内存 + 1T SSD。

> https://github.com/hiyouga/LlamaFactory
> 
> http://karpathy.github.io/2026/02/12/microgpt/
## Task 1. Pre-train Simple LLM
- Architecture: A tiny Transformer or a simple Bigram/RNN model.
- Dataset: A simple text file (e.g., a book from Project Gutenberg).
- Vocabulary: Letters and punctuation instead of complex tokens.
- Goal: Watch the "loss" drop and see random gibberish turn into readable words.

## Task 2. Pre-train Multi-modal VLM

## Task 3. Fine-Tune (SFT + QLoRA)

## Task 4. Fine-Tune (DPO)
## Task 5. Distill

## Task 6. Tool-Use & Function-Calling

