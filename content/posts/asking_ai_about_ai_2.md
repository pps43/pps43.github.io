---
title: "用AI学AI.2"
date: 2023-05-14
tags: ["AI"]
hideSummary: false
---

[上一篇]({{< ref "/posts/asking_ai_about_ai.md">}})我们聊了AI模型的分类和大致架构，目前最火的两类模型`LLM`和`Diffusion`都基于`Transformer`架构训练而成，`ChatGPT`则是`LLM`家族中最出色的产品。本文就来聊聊最近出现的基于`ChatGPT`的创新架构和应用。由于`ChatGPT`的训练数据截止到2021年9月，且笔者并没有Plus会员才能使用的`ChatGPT Plugin`，所以这次请到的是[Dr. Phind](https://www.phind.com/)（The AI search engine for developers），大家欢迎。

---
系列文章
1. [用AI学AI.1]({{< ref "/posts/asking_ai_about_ai.md">}})
2. [用AI学AI.2]({{< ref "/posts/asking_ai_about_ai_2.md">}})
3. [用AI学AI.3]({{< ref "/posts/asking_ai_about_ai_3.md">}})
4. [用AI学AI.4]({{< ref "/posts/asking_ai_about_ai_4.md">}})

---


# 笔记

Agent如何工作
- Autonomous agents (e.g., [AutoGPT](https://github.com/Significant-Gravitas/Auto-GPT), [BabyAGI](https://github.com/yoheinakajima/babyagi/tree/main)) can **auto-create** tasks towards the GOAL you give them, and **auto-find** the way to complete task based on GPT models and other data sources (websites, your documents, etc), and even **auto-improve** themselves from previous mistakes. 
> The basic structure is somehow similar to `GOAP` in video game's AI design.

长短期记忆
- They can achieve these because they can have **long-term and short-term memory** powered by Vector Database. ( memory size is much more than single GPT4's context which is 32K tokens, ~24K words.)

向量数据库
- Vector Database is a specialized db to handle vectors (read, write, calculate similarities, etc). they are useful for text because text can be transformed into numberic vectors by **embedding** technique. **Similarity between embeddings (vectors) reflect relevance between text**. Hence above agents can retrieve relevant data from large datasource and sending to GPT as context.

LangChain
- Frameworks like [LangChain](https://github.com/hwchase17/langchain) makes it easier to implement autonomous agents because it provides a standard interface to handle different tools and data sources.

拓展阅读
- [Tutorial: ChatGPT Over Your Data (using LangChain)](https://blog.langchain.dev/tutorial-chatgpt-over-your-data/)
- [Create your own chat bot with GPT-Index & LangChain](https://www.fabrikod.com/create-your-own-chat-bot-with-gpt-index-and-lang-chain)

# 问答

## Auto-GPT
Me:
> what is autogpt and how does it work

Dr. Phind:
> [Auto-GPT](https://github.com/Significant-Gravitas/Auto-GPT) is an experimental open-source project built on the latest ChatGPT model i.e GPT-4. It is a variant of ChatGPT developed by Significant Gravitas and it uses the ChatGPT API to work. Specifically, Auto-GPT utilizes the GPT-4 API, though it seems like it should work with the GPT-3.5 API as well. **Auto-GPT is an AI that does the work for you by setting a goal and then generating and completing tasks**. It does all the follow-up work for you, asking and answering its own prompts. It automates multi-step projects that would have required back-and-forth prompting if you worked directly with GPT-4. In other words, it **"chains together LLM 'thoughts', to autonomously achieve whatever goal you set."**
> 
> There are new apps that have emerged to make Auto-GPT even easier to use, like [AgentGPT](https://agentgpt.reworkd.ai/) and [GodMode](https://godmode.space/), which provide a simple interface where users can input what they want to accomplish directly on a browser page.

Me:
> how does AutoGPT work exactly? Find the techniques behind.

Dr. Phind:
> Auto-GPT is a Python application built on top of GPT-4 that allows the AI to function independently without requiring prompts from the user for each action. 
> 
> Auto-GPT breaks down the AI’s actions into “thoughts,” “reasoning,” and “criticism,” which enables users to understand precisely what the AI is doing and why it’s doing it. 
> 
> Auto-GPT uses **long and short-term memory management**, which is similar to how humans learn from their mistakes. Auto-GPT has the ability to assess its work, improve upon past experiences, and leverage its history to generate more precise results. Auto-GPT's integration with **vector databases**, a memory storage solution, allows it to retain context and make informed decisions.

Me:
> list some autonomous agent besides autogpt

Dr. Phind:
> here are several other autonomous AI agents besides Auto-GPT, some of which are Camel, BabyAGI, and Westworld simulation.
>
> - **Camel** is an autonomous AI agent that uses LangChain to store and retrieve information. It can learn and understand human language and is capable of making decisions based on the context. Camel can generate content for social media and can perform simple tasks like booking a restaurant reservation or scheduling a meeting ([Reference](https://towardsdatascience.com/4-autonomous-ai-agents-you-need-to-know-d612a643fa92)).
>
> - ~~**Westworld simulation** is an autonomous AI agent that simulates the Westworld environment. It uses reinforcement learning to improve its decision-making ability and can learn from its mistakes. Westworld simulation can interact with other agents in the environment and can perform tasks like navigating through the environment, recognizing objects, and even interacting with human~~. 
>   
>   ⚡Me: In fact it's an interactive sandbox environment created by researchers from Stanford and Google, the paper is [here (PDF)](https://arxiv.org/pdf/2304.03442.pdf) and this is the basic architecture. 
>   ![The Generative Agent Architecture](/asking_ai_about_ai_2/GenerativeAgentArchitecture.png)
>
> - **BabyAGI** is another autonomous AI agent that uses GPT-4, Pinecone, LangChain, and Chrome to generate and execute tasks. It can make human-like cognitive decisions and can write and run codes to meet certain objectives. BabyAGI can retain long-term memory of tasks and events with the help of Pinecone and LangChain, making fetching information faster. Its decisiveness makes it an effective tool for cryptocurrency trading, robotics, and autonomous driving.
>
>   ⚡Me: add info from [Github](https://github.com/yoheinakajima/babyagi/tree/main): the script works by running an infinite loop that does the following steps:
>   ![BabyAGI_Architecture](/asking_ai_about_ai_2/BabyAGI_Architecture.png)
> 
> These autonomous AI agents have different capabilities and approaches to accomplishing tasks. Camel is useful for generating content for social media and performing simple tasks, ~~Westworld simulation is useful for simulating environments and interacting with other agents~~ and BabyAGI is useful for decision-making sectors like autonomous driving and robotics.

## BabyAGI

Me:
> what is babyagi and compare with autogpt?

Dr. Phind:
> BabyAGI and Auto-GPT are autonomous AI agents designed to generate and perform tasks based on a given objective. The primary difference between the two is the way they approach the objective, the techniques they use, and their purpose.
> 
> **[BabyAGI](https://github.com/yoheinakajima/babyagi/tree/main) uses a combination of GPT-4, Pinecone, LangChain, and Chrome to create and execute tasks**. It creates a task list based on the objective entered and executes them one by one. The results of one task determine the next one. It can retain a long-term memory of tasks and events with the help of Pinecone and LangChain, which makes fetching information faster. Since it uses a trial-and-error basis to decode the results of previous tasks, BabyAGI can make complex decisions without losing track of the predefined objective.
>
> Auto-GPT, on the other hand, generates and runs multiple tasks at once using GPT-4 and creates an artificial memory space using GPT-3.5 to store results from previous tasks. It has access to internet services like apps, websites, books, documents, and articles and can use them to complete tasks required to reach the objective. Auto-GPT is good at generating text-rich content and images. It can also provide human-like text responses, making it useful for generating content, summarizing texts, and translating text into more than a dozen languages

## Vector Database
Me:
> what is vector database and how does it integrate into above system?

Dr. Phind:
> Vector database is a type of database that is designed to handle high-dimensional data such as **embeddings, which are numerical representations of objects or concepts in a high-dimensional space**. In the context of Auto-GPT, vector databases are used to store the embeddings of documents that are generated by OpenAI's language model. **The vector database is used to enable similarity search over these embeddings, which is used by Auto-GPT to find the most relevant documents to complete a task**. 
>
> (Me: Below is just one example to integrate `pgvector` to auto-gpt, there are other choice, e.g., [weaviate](https://weaviate.io/blog/autogpt-and-weaviate))
> 
> The vector database is integrated into the Postgres database using `pgvector`, an open-source vector similarity search extension for Postgres.  In more detail, open-source vector similarity search extension `pgvector` allows for storing and querying vector embeddings in a Postgres database.  OpenAI's `text-embedding-ada-002 model` outputs 1536 dimensions, so that is the size of the vector used in Auto-GPT. The `pgvector` extension introduces a new data type called `vector`, which is used to create a column named embedding with the vector data type in the Postgres database. The similarity search over these embeddings is performed using a function named `match_documents`, which takes a query embedding, a match threshold, and a match count as inputs and returns a table of document IDs, content, and similarity. The similarity is calculated using vector math operations like cosine distance. ([Reference](https://supabase.com/blog/openai-embeddings-postgres-vector))

Me:
> list some popular vector database product or repo

(Dr.Phind.... Google and Google Bard comes to help):
> Sure, here are some popular vector database products or repos:
> 
> - [FAISS](https://github.com/facebookresearch/faiss) (Facebook AI Similarity Search) is a library for efficient similarity search of dense vectors. It is open source and available on GitHub.
> - [Milvus](https://github.com/milvus-io/milvus): Milvus is a vector database designed for high-performance similarity search and machine learning.Milvus 2.0 is a cloud-native vector database with storage and computation separated by design. All components in this refactored version of Milvus are stateless to enhance elasticity and flexibility.
> - [Annoy](https://github.com/spotify/annoy) (Approximate Nearest Neighbors Oh Yeah)  is a C++ library with Python bindings to search for points in space that are close to a given query point. It also creates large read-only file-based data structures that are mmapped into memory so that many processes may share the same data.
> - [weaviate](https://github.com/weaviate/weaviate)
> 
> These are just a few of the many vector database products and repos that are available. The best product or repo for you will depend on your specific needs and requirements.

## LangChain

Me:
> what is langchain and how does it work?

Dr. Phind:
> [LangChain](https://github.com/hwchase17/langchain) is a Python package that **provides a standard interface** for chains, integrations with other tools, and end-to-end chains for common applications [3]. It is designed to assist in four main areas, which are listed here in increasing order of complexity: prompts, long-term memory (LLMs), tools, chains, agents, and memory [2]. 
> 
> LangChain is a framework **for developing applications powered by language models**. It is created to be data-aware and agentic, meaning that it connects a language model to other sources of data and allows a language model to interact with its environment.
>
> (...)
> LangChain can be used in a variety of ways, including autonomous agents, personal assistants, question answering, chatbots, querying tabular data, code understanding, interacting with APIs, summarization, and evaluation.

Me:
> does AutoGPT use Langchain?

Dr. Phind:
> There is no mention of AutoGPT using LangChain in the provided context. While LangChain is a framework that makes it easier to build scalable AI/LLM apps and chatbots, there is no clear indication that AutoGPT specifically uses LangChain.
