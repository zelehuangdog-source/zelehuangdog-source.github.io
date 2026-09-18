---
title: "Agent 工程面的演进：构建 Claude 托管智能体"
date: 2026-06-10
author: repost
categories: [转载, Agent架构与框架]
tags: [翻译, Claude, Agent架构, 生产部署, 转载]
---

> **摘要**：本文来自 Anthropic Applied AI 团队，梳理了 Claude API 从"单次请求-响应"到 Claude Agent SDK、再到 Claude Managed Agents（托管智能体）的三段演进脉络。核心论点是：让 Agent 进入生产环境的瓶颈不在于模型能力，而在于基础设施——包括沙箱、会话持久化、凭证隔离和可观测性。Managed Agents 将"大脑"（Harness）与"双手"（沙箱执行）解耦，把这些基础设施打包为托管服务，使团队可以在数天内而非数月内完成从原型到上线的跨越。文章还具体介绍了 Notion、Rakuten、Sentry 等客户的落地案例，以及 Managed Agents 提供的四大核心优势：凭证隔离、低延迟、持久会话和灵活的部署模式（Anthropic 托管或自托管）。

> **转载声明**：本文**翻译整理**自：《The evolution of agentic surfaces: building with Claude Managed Agents》。**内容版权归原作者及原出处所有**，本文仅供学习交流，**所有观点均属原作者，不代表本站立场**；如有侵权，请联系删除。


---

#### 从一次性原型到生产就绪

让 Agent 进入生产环境，远不止写一个好 prompt 那么简单。Agent 需要一个地方运行它生成的代码、凭证来访问你的数据、可观测的会话，以及能随用量扩展的基础设施。在 Applied AI 团队，我们工作在产品、研究与客户的交叉点上，不断看到同一个规律：**基础设施，是原型与生产 Agent 之间的分水岭**。团队往往把大量开发周期耗费在安全、状态管理、权限控制和 Harness 调优上。

Claude Managed Agents 是我们的一套可组合 API 套件，用于构建和部署生产级 Agent。它将一个针对性能调优的 Agent Harness 与生产基础设施结合，让团队能在数天而非数月内从原型走向上线。本文将介绍 Anthropic Agent 构建模块的演进历程、我们为什么构建 Claude Managed Agents，以及团队今天如何在生产中使用它。

---

#### Agent 架构的演进

**阶段一：纯 API，tokens in，tokens out（2023 年）**

2023 年我们向开发者开放 Claude API 时，接口设计刻意保持简洁：输入 tokens，输出 tokens。你发送一个 prompt，Claude 返回一次补全，然后由你自己构建 Harness 和底层基础设施。

多年来 API 持续丰富，但底层契约从未改变：一次请求，一次模型轮次，由你的应用决定接下来发生什么。很长时间里，这已经足够——总结文档、分类支持工单、改写一段文本，这类工作都能舒适地在单次轮次内完成。

**阶段二：Agent 循环的出现**

然而随着时间推移，人们想要交托的任务开始超出单次轮次能承载的范围。他们希望 Claude 把一项任务从头做到尾——查询信息、采取行动、观察变化、决定下一步。而且希望它在已有的工作系统中运作，比如代码仓库、内部 wiki 或工单系统。

用 API 把 Claude 变成 Agent，意味着自己搭循环：询问模型该做什么，执行工具，把结果喂回去，如此往复。你负责构建和部署 Agent 脚手架，而随着模型演进，脚手架可能还需要不断调整。对于需要完全自定义的 Agent，这种方式合情合理；但对于更可预期、更不复杂的 Agent 工作负载来说，随着模型和产品迭代而持续优化 Harness 就变成了一种负担。

**阶段三：Claude Agent SDK**

Claude Code 是我们于 2025 年发布的 Agentic 编程工具，它让 Claude 能与你的代码库直接交互。其中包含了我们自己版本的 Harness：循环、工具执行、子 Agent、上下文管理，以及让它成为有效 Agent 的丰富能力。开发者自然希望在各自的领域拥有类似的 Harness 机制。

为此，我们发布了 **Claude Agent SDK**。它让开发者能在驱动 Claude Code 的同一套机制上构建自己的 Agent，而不必维护一个自制的循环。对很多团队来说，这是 Agent 真正变得可行的时刻：Harness 已经针对 Claude 调优好，附带基础设施原语，并随着 Claude Code 的演进持续改进。

**阶段四：生产部署的挑战**

即便有了 Harness，在生产环境中部署 Agent 仍然充满挑战：

- **托管与扩展**：Agent 在哪里运行？对于耗时数小时的任务，进程能存活多久？用量增长时如何扩展？
- **会话管理**：Agent 的历史与进度存放在哪里？一次运行能从中断中恢复吗？能回溯检查之前的会话吗？
- **文件系统管理**：真正的工作意味着产出物：编辑代码、写文件、构建输出。Agent 在哪里获得工作空间？轮次之间工作空间如何处理？
- **执行隔离**：Claude 生成的代码必须在某个地方执行。如果代码出错，影响半径有多大？生产环境中你真正信任的边界在哪里？
- **凭证管理**：Agent 需要访问你的系统。如何在不将私有信息暴露给它生成的代码的前提下授权？
- **可观测性**：当一个 Agent 自主工作了一小时后做出了令人意外的事，你能重建它每一步的路径吗？

在 Agent SDK 中，上述生产基础设施的许多要素已通过 Claude Code 的机制提供。Agent 拥有真实的文件系统、会话状态可持久化到本地或外部存储，可观测性可通过 OpenTelemetry 导出到你已有的监控体系。

然而，当团队越来越多地将 Agent 从本地开发推向生产时，他们需要一种能以托管基础设施大规模部署的方式。而随着模型及其 Harness 变得越来越高级——运行时间更长、执行代码更多、触达系统更广、采取行动更多——扩展性、安全性和沙箱隔离都变得愈发棘手。

这些障碍有一个共同的架构根源：**Agent Harness 往往与它操作的文件系统运行在同一个容器中**。容器必须先启动（付出冷启动成本）Claude 才能开始思考；Agent 连同代码执行紧挨着你的凭证；容器一旦崩溃，运行也随之中止。

---

#### Managed Agents 的解决方案：大脑与双手解耦

Managed Agents 通过**将大脑与双手解耦**解决了这些问题。调用 Claude 的 Harness 与执行代码的沙箱独立运行，会话——所有模型调用、工具调用和结果的 append-only 日志——将两者串联起来。Claude 可以在容器存在之前就开始推理，沙箱与凭证保持安全距离，整个运行过程可以随时从会话中完整重建。

---

#### 何时使用 Claude Managed Agents

使用 Managed Agents 构建时，用户定义任务、工具和护栏，Anthropic 在我们的基础设施上运行 Agent 并处理底层的 Agentic 循环：如何为 Agent 提供调用工具的执行环境、出错时如何恢复、多 Agent 编排等。

**Harness 不随模型智能演进会出什么问题？** 以下是一个真实案例：在 Claude Sonnet 4.5 上，Agent 在接近上下文末尾时会急于结束，提前终止工作，而不是利用剩余空间——这种模式被称为"上下文焦虑（context anxiety）"。我们的修复方案是向 Harness 添加上下文重置，内置了 Claude 在接近限制时需要帮助保持连贯性的假设。然而这个假设在下一个模型上就失效了。在 Claude Opus 4.5 上，这一行为消失了，我们添加的重置变成了纯粹的开销。

对大多数组织来说，维护 Harness 是不能带来差异化的负担。Harness 必须针对特定模型行为调优；压缩（compaction）、工具执行、缓存等原语在 Claude 上的工作方式与其他模型不同。**有了 Claude Managed Agents，Harness 随模型同步演进，让团队可以聚焦于真正能差异化 Agent 的地方：上下文管理和领域专业知识。**

##### 三个核心资源

Managed Agents 围绕三个核心资源构建：

- **Agent（智能体）**：一项配置——模型、prompt、工具集及其护栏。
- **Environment（环境）**：Agent 运行的执行上下文——沙箱容器、网络规则、预装软件包，托管在我们的云上或你控制的基础设施上。
- **Session（会话）**：每次运行都是一个会话，将 Agent 与 Environment 配对，获得独立的沙箱实例。会话在服务端持久化其完整事件历史、沙箱状态和输出，因此长时间运行的工作可以暂停、干净地恢复，并在事后逐步追溯。

你可以定义一次 Agent 和 Environment，然后随着工作负载增长，在同一配置上运行多个 Session。

---

#### 在 Managed Agents 上构建生产与规模

在 Applied AI 内部，我们看到 Agent 在 Anthropic 内部和客户系统中从原型走向生产，覆盖编程、金融、支持、法律等十几个领域。这给了我们清晰的视角：**Demo 与生产就绪 Agent 之间的差距在哪里，团队最常在哪里卡住。**

以下是构建在 Managed Agents 这类托管服务上最常见的四大理由：

##### 1. 凭证与沙箱隔离

当所有东西运行在同一个容器中，Claude 生成的代码就紧挨着你的凭证，prompt 注入攻击可能通过说服模型读取自身环境来泄露 token。虽然可以在同一容器内设置强大护栏，但解耦架构能实现更安全的方式——**将凭证完全隔离在沙箱之外**。

MCPs、CLIs、GitHub 仓库等工具的 token 存放在独立的保险库（Vault）中，代理按需获取并解密。Managed Agents 提供开箱即用的 Vaults，无需自建密钥存储、每次调用时传输 token，也不会搞不清某个 Agent 代表哪个终端用户行事。Vault 凭证在存储前采用信封加密保护，检索需要经过签名请求 token 验证。

##### 2. 消除沙箱开销带来的低延迟

延迟是许多企业团队的核心指标，用户对等待 Claude 响应有着强烈感知。在没有 Managed Agents 架构的情况下，每个 Session 都必须启动一个容器，即便 Agent 只需要思考而从不调用工具。这部分启动时间被浪费了，用户感受到的是首次响应前的延迟。

有了 Managed Agents，**Claude 在环境并行启动时立即开始推理**，从不运行工具的 Session 完全跳过容器。用户无需等待容器启动就能看到第一个 token，而 Agent 真正需要执行时，环境已经准备好了。在我们的测试中，这将首 token 时间（TTFT）在中位数情况（p50）缩短了约 60%，在最慢的情况（p95）缩短了 90% 以上。

##### 3. 支持会话管理、可观测性和记忆的可靠持久会话

Managed Agents 不是以请求/响应为单位思考，而是以**事件**为单位。一个 Session 是持续流入的事件流：每次模型调用、工具调用和结果都被追加到一个存活于运行 Agent 进程之外的日志中。

这种架构带来的好处：
- Agent 工作时可以实时获取事件流更新
- 随时恢复任意 Session，无需管理数据库或存档点
- 历史在交互之间保留，除非你删除 Session
- Session 空闲时其容器会被存档，方便从暂停处干净恢复
- 整个运行已经是一份事件记录，可观测性和记忆随之而来

Claude Developer Console 提供 Agent Session 的原生可视化时间线视图，以及深度检查任意 Transcript 的调试体验。Managed Agents 还带有 **Memory（记忆）** 和 **Dreaming（梦境）** 功能，同样利用这种 Session 持久性。Dreaming 是一个计划任务，它审阅 Agent Session 和记忆存储，提取规律并筛选记忆，让 Agent 随时间持续改进。

##### 4. Anthropic 托管或自托管容器的灵活性

默认情况下，使用 Managed Agents 可以将编排和工具执行都委托给 Anthropic 托管的云容器，实现简单易用的托管与扩展，提供更快的上线路径。

由于 Managed Agents 中大脑与双手是解耦的，双手可以存在于任何地方，包括你的 VPC（虚拟私有云）内部。因此，我们还为希望控制工具执行的团队提供**自托管沙箱**，让 Agent 的代码、文件系统和网络出口永远不离开他们的环境。我们还提供 **MCP 隧道（MCP tunnels）**，让你能将 Claude 连接到私有网络内部运行的 MCP（模型上下文协议）服务器。自托管沙箱控制 Agent 代码的执行位置，MCP 隧道控制 Anthropic 如何访问网络内部的 MCP 服务器，让你能精确控制什么留在你的边界内。

> Managed Agents 内置的可观测性控制台记录每一个事件，你可以拖动时间线、展开任意步骤并读取其原始数据。

除上述功能外，附加能力还包括：让 Agent 按评分标准给自身工作打分的 Outcomes（结果评估）、多 Agent 编排、权限策略和 Webhook。

---

#### 客户今天如何构建 Managed Agents

在各行业，客户已经在用 Claude Managed Agents 将 Agent 落地生产。以下是几个典型案例：

- **Notion**：在 Managed Agents 上运行其自定义 Agent——团队直接从任务看板把工作分配给 Claude，Claude 抓取任务周边的文档、会议记录和关联数据，完成的代码、演示文稿和站点回流到工作空间供审阅。数十个任务并行运行，团队描述早期原型将大约 12 小时的工作压缩到了 20 分钟。
- **Rakuten**：用 Managed Agents 在产品、销售、市场和金融部门各交付了专业 Agent，每个 Agent 大约一周内上线。
- **Sentry**：将 Seer 调试 Agent 与一个负责编写补丁、提交 PR 的 Claude Agent 配对，由一名工程师在数周而非数月内完成构建。
- **Asana**：构建了在项目内接手任务的 AI 队友（AI Teammates）。
- **Atlassian**：将开发者 Agent 接入 Jira 工作流。

---

#### 开始使用 Claude Managed Agents

我们在构建 Managed Agents 时，希望通过 Claude Code 和 platform.claude.com 的 Claude Developer Console 尽可能简化 Agent 的启动。控制台的快速入门（quickstart）可以让你从 Agent 模板出发，或用自然语言描述你想构建的 Agent，然后在几分钟内将其变成可以保护和部署的生产就绪 Agent。

在 Claude Code 中，`/claude-api` skill 默认提供，并为构建 Claude Managed Agents 应用提供详尽的最新参考资料。强烈建议使用它来获取最佳实践。运行 `/claude-api managed-agents-onboard` 可以开启一个采访式引导流程，从零开始搭建新的 Managed Agent。

---

#### 托管智能体的未来

随着团队分享他们在 Managed Agents 上的构建成果，我们看到他们以前花在生产基础设施上的时间，现在转移到了真正能差异化其 Agent 的地方：管理上下文，为用户量身定制体验。现在，当新模型发布时，你更新 Agent 使用新模型、重跑评估、上线改进——整个过程无需触动底层架构。

---

*本文由 Anthropic Applied AI 团队的 Gagan Bhat 和 Isabella He 撰写，感谢 Hema Thanki、Jess Yan 和 Molly Vorwerck 的贡献。*
