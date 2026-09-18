---
title: "Karpathy 的 Agentic Engineering 终于有了配套工具"
date: 2026-06-27
author: repost
categories: [转载, AI工程方法论]
tags: [翻译, agentic-engineering, Google-ADK, RAG, 转载]
---

> **摘要**：Karpathy 在 Sequoia Ascent 2026 上将"Agentic Engineering"定义为区分生产级 Agent 开发与随意编码的专业工程学科，其核心技能包括 Spec 设计、Eval 循环和安全监督。然而一直以来缺乏统一的工具链支撑。Google 推出的 Agents CLI 填补了这一空白，通过向编码 Agent 注入 7 个技能（覆盖脚手架、评估、部署），让开发者仅凭自然语言提示就能完成从空文件夹到生产级 Agent 的全生命周期。本文以构建一个 RAG Agent 为例，演示了完整的六步流程。

> **转载声明**：本文**翻译整理**自：《Karpathy's Agentic Engineering Finally Has Proper Tooling》。**内容版权归原作者及原出处所有**，本文仅供学习交流，**所有观点均属原作者，不代表本站立场**；如有侵权，请联系删除。


---

由 Google 构建，以逐步指南的形式讲解。

Karpathy 在 Sequoia Ascent 2026 上将 Agentic Engineering（智能体工程）定义为将生产级 Agent 开发与"氛围编程"（vibe coding）区分开来的专业学科。

他列出的核心技能包括：Spec 设计、Eval 循环和安全监督。

然而，配套工具一直缺失——实践 Agentic Engineering 至今仍需在编辑器、终端（用于脚手架）、浏览器（用于测试）、云控制台（用于部署）和独立的评估框架之间来回切换。

面向生产级 Agentic Engineering 的解决方案如今已在 [Google 的 Agents CLI](https://github.com/google/agents-cli) 中实现。它在一个工具中覆盖了 ADK Agent 的脚手架搭建、评估和部署的完整工作流。

它向你的编码 Agent 注入 7 个技能，教会它 ADK 模式、Eval 结构和部署目标。

此后，编码 Agent 会自动从自然语言驱动整个生命周期，你无需离开编辑器就能完成任何阶段的工作。

让我们通过从零构建一个 RAG Agent 并将其部署为内部知识助手来完整走一遍这个流程。

#### Step 1：安装 Agents CLI

```bash
uvx google-agents-cli setup
```

该命令向编码 Agent 的上下文注入 7 个捆绑技能，涵盖 ADK 代码模式、项目脚手架、基于 LLM-as-judge 评分的评估设置、面向 Agent Runtime 和 Cloud Run 的部署配置，以及 Cloud Trace 可观测性。

每个技能教会编码 Agent 生命周期中特定阶段的工作方式，使其能直接从自然语言提示执行该阶段。

一条 setup 命令即可将这些技能同时安装到所有编码 Agent 中。Antigravity、Claude Code、Cursor、Codex 等，都通过一次安装获得相同的 ADK 专业能力：

#### Step 2：构建 RAG Agent

打开你选择的编码 Agent，描述要构建的 Agent：

```markdown
1. Build a RAG agent that ingests documents, retrieves relevant
2. context, and answers questions with source citations. Use the
3. ADK agentic_rag template with Gemini 3.5 Flash.
```

编码 Agent 激活其 ADK 技能并搭建完整的项目脚手架：

- Claude Code 使用 ADK agentic_rag 模板（以 Vector Search 作为数据存储）搭建了项目。
- 随后它发现模板缺少引文支持，于是重写了 Agent 指令以要求生成带有内联引文的有据可查的回答，并修改了检索器以在每个文档中暴露来源 ID。
- 它配置了数据存储，摄入了一个合成问答语料库（12 条 Python 基础知识条目），并运行了冒烟测试。Agent 返回了带引文的回答，并在检索服务不可用时正确拒绝了幻觉。

注入的技能了解检索增强 Agent 的 ADK 模式，这就是为什么脚手架天然包含了引文支持和 Vector Search 配置。

#### Step 3：本地测试

接下来，我们让编码 Agent 在 localhost 上启动 ADK Web UI：

```plaintext
Spin up a local dev server so I can test this.
```

这将启动一个交互式聊天界面，你可以在此针对真实查询测试 Agent。需要验证两点：

- **第一，检索和引文是否正确？** 我们问"how to merge two dictionaries?"，Agent 从语料库中拉取了正确的上下文，详述了合并运算符和 `update()` 方法，并在行内附上了 [source: 1003]。引文功能正常。
- **第二，缺失上下文时的处理是否正确？** 我们问"who won the FIFA World Cup in 2022?"——这是语料库中没有答案的问题。Agent 回应说它无法根据现有文档回答。

#### Step 4：部署前评估

这是最重要的一步，也是大多数 Agent 教程完全跳过的一步。

```plaintext
1. Generate 20 test scenarios for this RAG agent covering correct
2. retrieval, insufficient context where the agent should say it
3. doesn't know, multi-hop questions, and citation accuracy. Run
4. the full eval suite and show me the results.
```

编码 Agent 生成了 20 个测试场景，分为四个类别：

- 6 个用于正确检索（语料库能回答的问题）
- 5 个用于不充分上下文（应该拒绝回答的问题）
- 5 个用于多跳推理（需要多个文档的问题）
- 4 个用于引文准确性

Karpathy 特别指出了这个缺口：89% 运行 Agent 的团队已经设置了可观测性，但只有 52% 有 Eval。Agents CLI 让你通过一个提示就能生成并运行完整的 Eval 套件。

结果：

- 引文准确率在所有 20 个案例中均为满分 1.00。Agent 从未捏造来源。
- 但幻觉评分标记了一个边缘情况：对于语料库外的问题，Agent 有时会追加通用知识而不是说它没有足够的上下文。Eval 追溯到指令中的一行（"if you already know the answer to a simple question, you may respond directly without using the tools"），删除该行即可解决问题。

#### Step 5：部署到 Agent Runtime

```plaintext
Deploy this to Agent Runtime in us-central1.
```

编码 Agent 首先通过添加部署入口点和基础设施配置来增强项目以适配 Agent Runtime。

然后将 Agent 部署到 Google Cloud，整个过程大约耗时 2-3 分钟。

Cloud Trace 默认启用，因此可观测性从第一个已部署请求起就内置了。

#### Step 6：注册到 Gemini Enterprise

至此，Agent 已部署并可工作，但它只对构建它的开发者可访问。

其他任何想使用它的人都需要端点 URL、正确的 API 凭证，以及足够的上下文来知道这个 Agent 的存在。

在大多数团队中，这是有用的 Agent 悄然消亡的地方。它们能工作，但构建者直接圈子之外没人知道或能访问它们。

让 Agent 执行以下操作即可将应用注册到 Gemini Enterprise 平台，使其在整个组织的 Gemini Enterprise 应用中可被发现：

```plaintext
Register this agent to Gemini Enterprise.
```

任何拥有想要使其可搜索的内部文档的团队，都可以访问相同的知识助手，而无需搭建自己的 RAG 管道。IAM 控制谁能访问它，企业仪表板提供完整的可观测性。

这就是 Karpathy 所描述的、拥有适当工具支撑的 Agentic Engineering 的样子。

通过一个终端会话和六个自然语言提示，Agent 从一个空文件夹变成了组织可以使用的生产级助手。

- [GitHub 上的 Agents CLI →](https://github.com/google/agents-cli)
- [ADK 文档 →](https://adk.dev/)
- [Agent Platform →](https://cloud.google.com/gemini-enterprise/agents)
