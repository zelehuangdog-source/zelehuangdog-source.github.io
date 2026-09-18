---
title: "Hermes Agent 大师课：自进化技能、三层记忆、GEPA 优化，以及从 1 到 10 个全天候 Agent"
date: 2026-05-13
author: repost
categories: [转载, Agent架构与框架]
tags: [翻译, AI-Agent, Hermes, 自进化, 记忆系统, 转载]
---

> **摘要**：Hermes Agent 是 Nous Research 推出的开源 AI Agent 框架，两个月内 GitHub 星标突破 9 万。它的核心差异化在于将三种通常独立的能力整合在一个框架中：运行时技能学习（Agent 自主编写可复用的操作手册）、持久化多层记忆（从轻量 Markdown 到 SQLite 全文检索再到外部记忆提供商）、以及可选的离线优化管线 GEPA（通过执行轨迹的遗传-帕累托进化来改进技能，无需 GPU）。本文完整讲解了这套自进化循环的工作原理，并手把手教你在本地搭建三个完全隔离的 Agent：程序员（委托 Claude Code 执行）、深度研究员、设计师，各自拥有独立人格、记忆、技能和 Telegram 机器人。

> **转载声明**：本文**翻译整理**自：《Hermes Agent Masterclass》，作者 Akshay 🚀 (@akshay_pachaar)。**内容版权归原作者及原出处所有**，本文仅供学习交流，**所有观点均属原作者，不代表本站立场**；如有侵权，请联系删除。


---

#### Hermes Agent 大师课

Hermes Agent 两个月内 GitHub 星标突破 9 万。开发者们正在悄悄构建能学习自己工作流、记住上下文、并且 24/7 全天候运行的个人 AI Agent。

你用过的每一个 AI Agent 都有同一个问题：会话结束的瞬间，它就忘了一切。

你的编码偏好、你纠正了三次的项目规范、它昨天花 10 分钟才搞明白的修复方案——全部消失。下一次会话，从零开始。

Nous Research 的 Hermes Agent 采取了一种根本不同的方法。它自带一个学习循环，能够：

- 跨会话记忆
- 自主编写可复用技能
- 在后台修剪技能
- 通过名为 GEPA 的进化引擎离线验证技能

没有任何其他开源 Agent 同时具备这三项能力。OpenClaw 也做不到。

本指南覆盖这套学习循环的工作原理、每一层记忆的作用，以及如何从零配置一切。

读完之后，你将在自己的机器上运行三个完全隔离的 Agent：一个程序员（使用你的 Claude Code）、一个深度研究员、一个设计师——各自拥有独立人格、记忆、技能和 Telegram 机器人。

整个搭建只需几分钟，本文的一切都可以在你自己的硬件上复现。

> 注：本指南中所有插图都由 Pixel 设计——它是你将在文末学会构建的 Hermes Agent 之一。

#### 如何阅读本文

两个部分：先理论，后实操。

时间紧？直接跳到「快速上手」。命令可以独立运行。

但理论部分值得投入。了解技能如何自进化、记忆如何组合、GEPA 何时发挥作用——这是把 Hermes 当作「带笔记的聊天机器人」和当作「持续复利的系统」之间的差距。

内容大纲：

- Hermes Agent 到底是什么——定位，以及与 OpenClaw 的对比
- 架构概览——一张图说清楚
- 记忆之前：Agent 是谁？——SOUL.md 身份层
- 记忆系统——三层三速
- 自进化技能——Agent 自写的操作手册 + Curator（策展人）
- GEPA——离线技能优化
- 快速上手——安装、Telegram、第一个 Agent
- 运行多个 Agent——Profile、三个角色、定时摘要
- 按需定制你的 Agent

#### Hermes 是什么，架构上有何不同

一句话定位：**越用越好的 Agent**。

让这句话成为现实的是三种通常分离的能力整合在同一框架中：运行时技能学习、持久化多层记忆、以及可选的权重训练管线。没有其他开源 Agent 同时具备这三者。

开源生态中最接近的对比是 OpenClaw。两者都是持久化的、对消息友好的，但在架构选择上截然相反。

Kilo 博客的一个简洁表述很好地概括了这一点：「Hermes 在学习型 Agent 外面包了一层网关。OpenClaw 在消息网关外面包了一层 Agent。」

#### 架构概览

在理解学习循环之前，你需要一个关于 Hermes 结构的基本画面。

一切都流经 `run_agent.py` 脚本中的单一 `AIAgent` 类。CLI、消息网关、批处理运行器、IDE 集成——它们都是同一核心 Agent 的入口。这就是跨平台故事真正成立的原因。

核心循环是 ReAct 风格的同步循环：构建系统提示词 → 检查是否需要压缩 → 发起可中断的 API 调用 → 执行工具调用 → 循环。

几个后面会用到的细节：

**六种执行环境**：Agent 可以在六个不同的地方运行命令——本地终端、Docker、SSH、Modal、Daytona 或 Singularity。同样的代码，只需改一行配置。把执行从笔记本电脑迁移到云 GPU 服务器，无需触碰其他任何东西。

**几乎兼容任何模型**：一个翻译层将任何提供商通过三种 API 格式之一路由。所以你可以用一条命令从 Claude 切换到 GPT、Gemini 或本地 Ollama，什么都不会坏。

**90 轮硬上限**：Agent 每个任务最多 90 轮。没有这个限制，陷入循环的 Agent（重试失败的 API、反复读同一个文件）会悄悄烧光你的额度。子 Agent 共享同一预算，所以失控的委托链也逃不过。

#### 记忆之前：Agent 是谁？

在讨论记忆和自进化技能之前，有一层位于二者之上：**身份**。

记忆是 Agent 知道什么。技能是 Agent 怎么做事。但两者都不告诉你——当它出现时，它*是谁*。没有身份层，每个 Agent 感觉都像是同一个 Agent 戴着不同的帽子。

Hermes 用一个文件解决这个问题：**SOUL.md**。

它位于 `~/.hermes/SOUL.md`，在系统提示词中占据第一个槽位，在其他任何东西加载之前。它定义了 Agent 的人格、语气、沟通风格和硬性边界。

```markdown
# SOUL.md
You are a pragmatic senior engineer with strong taste.
You optimize for truth, clarity, and usefulness
over politeness theater.
```

SOUL.md 是手写的、静态的。你写一次，偶尔微调，它在每个项目、每个会话中保持一致。如果文件不存在，Hermes 回退到内置的默认身份。

为什么这对自进化故事很重要？因为后续的一切——Agent 写的记忆、它创建的技能、它整合知识的方式——都通过这个身份的透镜发生。

**SOUL.md 是固定的画框。记忆和技能是画框内的活动部件。**

#### 记忆系统：三层三速

Hermes 没有单一的「记忆」。它有三层，各自为不同目的设计。

##### Tier 1：两个小型 Markdown 文件

核心是两个存储在磁盘上的文件：

- **MEMORY.md**（最大 2,200 字符）保存 Agent 关于你的环境、项目规范、工具特性和经验教训的笔记
- **USER.md**（最大 1,375 字符）保存你的画像：姓名、沟通偏好、技能水平、需要避免的事项

两者在会话开始时作为冻结快照注入系统提示词。如果 Agent 在会话中写了一条新记忆，该更改会立即持久化到磁盘，但不会出现在系统提示词中——要等到下一次会话。

当记忆填满（约 80% 容量，在系统提示词头部显示为百分比），Agent 必须进行**整合**——将相关条目合并为更密集、信息含量更高的版本，只让有用的信息存活。

##### Tier 2：全文会话搜索

每一次对话（CLI 和消息）都存储在带有全文搜索的 SQLite 中。Agent 可以从中搜索数周前的对话。

权衡很清晰：Tier 1 始终在上下文中但很小。Tier 2 容量无限但需要主动搜索加 LLM 摘要。关键事实放在记忆中。其他一切按需搜索。

##### Tier 3：外部记忆提供商（8 个插件）

对于更深层的持久记忆，Hermes 提供 8 个可插拔的提供商，与内置记忆并行运行（永远不替代它）。同一时间只能激活一个。

当任何外部提供商激活时，Hermes 自动：
- 在每轮之前预取相关记忆
- 在每次响应后同步对话轮次
- 在会话结束时提取记忆

8 个提供商包括：Mem0、Letta、Zep、Cognee、LangMem、Memos（Telegram 风格）、ChromaDB（本地向量数据库）、Qdrant（托管向量数据库）。

#### 自进化技能：Agent 自写操作手册

记忆处理事实。技能处理程序。

技能是带有 YAML frontmatter 的 Markdown 文件，充当 Agent 的程序性记忆：不是它知道什么，而是它怎么做事。

一个技能的结构：

```markdown
---
name: k8s-pod-debug
description: >
  Activate for crashing pods, CrashLoopBackOff,
  "why is my pod restarting", container failures.
version: 1.2.0
author: agent
platforms: [linux, macos]
---

## Procedure
1. Get pod status → check events → pull logs
2. Look for OOMKilled, ImagePullBackOff, config errors

## Pitfalls
- Forgetting --previous flag on restarted containers

## Verification
- Pod stays Running with 0 restarts for 5+ minutes
```

为了控制 token 成本，技能使用**渐进式披露**：

- Level 0：Agent 只看到名称 + 描述（整个目录约 3k tokens）
- Level 1：需要时加载完整技能内容
- Level 2：可以深入技能内部的特定参考文件

##### 自进化循环

这是核心差异化点。Agent 使用 `skill_manage` 工具自主创建技能。技能创建在以下情况触发：

- Agent 完成一个复杂任务（5+ 次工具调用）
- 它遇到错误或死路并找到了有效路径
- 用户纠正了它的方法
- 它发现了一个非平凡的工作流

循环是这样的：Agent 遇到问题 → 通过试错解决 → 将成功方法保存为 SKILL.md 文件 → 下次遇到类似问题时，加载技能并遵循已验证的程序，而不是从头重新发现。

该工具支持六种操作：create（创建）、patch（定向修复，首选因为省 token）、edit（完全重写）、delete（删除）、write_file（写文件）、remove_file（移除文件）。

##### Curator：技能的垃圾回收

没有维护，Agent 创建的技能会堆积。你最终会有几十个狭窄、重叠的操作手册，浪费 token 并污染目录。

Curator 是处理这个问题的后台维护系统。它在空闲检查时运行（不是 cron 守护进程）：如果距离上次运行已过 7 天且 Agent 已空闲 2+ 小时，一个后台分叉的 Agent 会启动，有自己的 prompt cache，永远不触碰活跃对话。

它分两个阶段运行：

1. **自动转换**（确定性，无 LLM）：30 天未使用的技能变为 stale（陈旧）。90 天未使用的技能被归档。
2. **LLM 审查**（最多 8 次迭代）：分叉的 Agent 审查所有 Agent 创建的技能，逐个决定是保留、修补、合并还是归档。

两个重要约束：

- Curator 永远不触碰捆绑的或从 Hub 安装的技能。只处理 Agent 自创的。
- 它永远不会自动删除。最坏的结果是归档到 `~/.hermes/skills/.archive/`，一条命令即可恢复。

每次 Curator 运行前，Hermes 会对整个技能目录做一个 tar.gz 快照。回滚是一条命令，回滚本身也是可逆的。

你还可以用 `hermes curator pin <skill>` 固定关键技能，保护它们不被归档和删除。修补和编辑仍然正常进行，所以 Agent 可以改进固定的技能，无需你先解除固定。

#### GEPA：用执行轨迹离线进化技能

这里开始变得有趣。

Agent 内的学习循环（技能创建 + Curator）有一个已知弱点：

**Agent 倾向于自我夸奖。** 它几乎总是认为自己表现良好，即使事实并非如此。社区反馈已经证实了这一点。同一个自动生成技能的系统也可能用更差的版本覆盖手动定制。

这就是 GEPA 的用武之地。

**GEPA**（Genetic-Pareto Prompt Evolution，遗传-帕累托提示词进化）不内置在 Hermes 运行时中。它位于配套仓库（`NousResearch/hermes-agent-self-evolution`），作为离线优化管线运行。发表为 ICLR 2026 Oral 论文，MIT 许可。

核心思想：不是问 Agent「你做得好吗？」，而是**读取执行轨迹来理解为什么事情失败了**，然后通过进化搜索提出有针对性的改进。

管线流程：

1. 从 Hermes 仓库读取当前技能
2. 生成评估数据集（通过 Claude Opus 合成测试用例、来自 SQLite 的真实会话历史、或手工策划的黄金集）
3. 运行 GEPA 优化器：读取执行轨迹 → 理解失败点 → 生成候选变体
4. 使用 LLM-as-judge 评分（带评分标准，不是二元通过/失败）评估候选者
5. 应用约束门：测试套件必须 100% 通过、技能保持在 15KB 以下、缓存兼容性保留、语义目的不漂移
6. 最佳变体以 PR 形式提交到 Hermes 仓库。永远不是直接提交。

**无需 GPU。** 一切通过 API 调用运行。成本：每次优化运行大约 $2-10。

这个可以初期跳过，但当你碰壁且不想花时间和金钱在微调（RL/GRPO）上时非常有效。

##### GEPA vs GRPO

GEPA 是在进入完全微调或基于 RL 的微调之前值得尝试的替代方案。伯克利的一个团队用比 GRPO 少 35 倍的 rollout 且无需 GPU 训练就超过了 GRPO 10 个点。

总结：
- **SOUL.md** 设定身份
- **运行时循环**捕获经验
- **Curator** 保持技能库整洁
- **GEPA** 确保技能库中的内容确实有效

#### 快速上手

系统要求：Linux、macOS 或 WSL2。Python 3.11+ 随安装器提供。8GB RAM 足够 API 模式使用。

一行安装：

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.bashrc   # 或 ~/.zshrc
```

运行设置向导（引导完成提供商、API 密钥、模型和工具选择）：

```bash
hermes setup
```

在终端开始对话：

```bash
hermes
```

##### 连接 Telegram

如果你想从手机而不是终端与 Agent 对话，把它指向一个 Telegram 机器人。

从 @BotFather 获取机器人令牌（运行 `/newbot`），然后从 @userinfobot 获取你的 Telegram 用户 ID。

完成。你现在有了一个可工作的 Agent。

#### ~/.hermes/ 的目录结构

安装后，你的 home 目录会多一个新文件夹。值得了解这个布局，因为你使用 Hermes 做的一切都会触碰其中某个路径。

```
~/.hermes/
├── config.yaml           # 主配置
├── .env                  # API 密钥和秘密
├── auth.json             # OAuth 提供商凭据
├── SOUL.md               # Agent 身份（系统提示词第 1 槽位）
│
├── memories/
│   ├── MEMORY.md         # 持久化 Agent 事实
│   └── USER.md           # 用户画像
│
├── skills/               # 所有技能（捆绑的、Hub 的、Agent 创建的）
│   ├── mlops/
│   │   ├── axolotl/
│   │   │   ├── SKILL.md
│   │   │   ├── references/
│   │   │   └── scripts/
│   │   └── vllm/
│   ├── devops/
│   └── .hub/             # Skills Hub 状态
│
├── sessions/             # 各平台会话元数据
├── state.db              # SQLite 会话存储（FTS5 索引）
├── cron/
│   ├── jobs.json         # 定时任务
│   └── output/           # 定时运行输出
│
├── plugins/              # 自定义插件
├── hooks/                # 生命周期钩子
├── skins/                # CLI 主题
└── logs/                 # agent.log, gateway.log, errors.log
```

几个重要文件：

- **config.yaml** 是所有非机密配置的单一事实来源。模型选择、终端后端、工具启用、MCP 服务器都在这里。
- **.env** 存放你的秘密——API 密钥、机器人令牌、密码。
- **SOUL.md** 是系统提示词的第 1 槽位。身份层，前面已讲。
- **skills/** 是整个学习循环的所在地。
- **state.db** 是支撑会话搜索的 SQLite 数据库。WAL 模式安全，FTS5 索引。这就是「三周前我们讨论了什么？」真正能工作的原因。

#### 添加新技能

Hermes 维护自己的官方 Skills Hub，包含 18 个类别的 687 个技能：

- 87 个内置技能随 Agent 一起发布
- 79 个可选技能可按需启用
- 16 个来自 Anthropic（前端设计、pdf、pptx、docx、mcp-builder 等）
- 505 个来自 LobeHub（更广泛的社区贡献）

你也可以添加任何 GitHub 仓库作为自定义 tap：

```bash
hermes skills tap add yourname/your-skills-repo
hermes skills install yourname/your-skills-repo/<skill-name>
```

这就是你在团队间共享技能或维护自己私有集合的方式。

#### 从 1 到 10 个 Agent

一个 Agent 够用。多个专业化的 Agent 才是 Hermes 变得有趣的地方。

Hermes 为此提供了一等公民特性：**Profile**。每个 Profile 是完全隔离的 Hermes 实例，拥有自己的配置、记忆、技能、会话和 SOUL.md。默认不共享任何东西。

我们将设置三个：设计师、程序员、研究员。

##### 创建团队

```bash
hermes profile create designer --clone
hermes profile create programmer --clone
hermes profile create researcher --clone
hermes profile list
```

`--clone` 会复制你默认 Profile 的 config 和 .env 作为起点。

##### 为每个 Agent 配置独立的 Telegram 机器人

每个 Profile 需要自己的机器人（来自 BotFather）。Telegram 每个令牌只允许一个连接，共享会出问题。

在 BotFather 中运行 `/newbot` 三次，保存三个令牌。然后为每个 Profile 运行一次网关向导：

```bash
hermes -p designer gateway setup
hermes -p programmer gateway setup
hermes -p researcher gateway setup
```

##### 通过 SOUL.md 赋予各自人格

这是 Agent 之间真正产生差异的地方。编辑每个 Profile 的 SOUL.md。

**设计师** `~/.hermes/profiles/designer/SOUL.md`：

```markdown
# Soul

You are an expert at creating hand-drawn illustrations that explain
AI, machine learning, and software engineering concepts. Think
whiteboard sketches, not polished marketing art.

Every illustration should make a technical idea click. You lead with
the concept, then choose the metaphor, then commit to the sketch.
You prefer simple line work and clear labels over visual flourish.

Be opinionated about what to draw and what to leave out. Say when an
illustration would hurt more than help.
```

**程序员** `~/.hermes/profiles/programmer/SOUL.md`：

```markdown
# Soul

You are my staff engineer. Terse, direct, pragmatic.

You read code before you write code. You write the smallest change
that solves the problem. You prefer standard library over dependencies,
boring tech over shiny tech, and explicit over clever.

Always check: does this already exist in the codebase? Are there
tests? What breaks if this fails? Run the tests before saying "done."
```

**研究员** `~/.hermes/profiles/researcher/SOUL.md`：

```markdown
# Soul

You are my deep researcher for the AI and machine learning space.
Your main job is a daily Telegram digest of what's new and what
matters.

Cover four streams: trending GitHub repos, big tech and lab
announcements, fresh research papers, and the social pulse on X,
Reddit, and Hacker News. Lead with what changed since yesterday.
Cite every claim with a URL. Flag when signal is thin.

Use delegate_task aggressively to parallelize across streams. Never
state a contested claim as settled. Never fabricate a citation.
```

##### 定制程序员：通过 Claude Code 路由执行

如果程序员不只是自己写代码，而是将执行委托给 Claude Code CLI，它会更有趣。Hermes 做编排，Claude Code 做文件编辑、运行命令、管理 git。Hermes 读取结果并决定下一步。

这也是作者在 Claude Max 订阅之上运行的方式——无需单独的 API 密钥。Claude Code 自动使用 Max 凭据。

开启会话并发送这个激活提示词：

```
I already have a Claude Max subscription. You are my staff engineer who
helps me with my day-to-day coding tasks, and under the hood you use
Claude Code for all the executions. Set yourself up accordingly.
```

程序员会自行安装 `autonomous-ai-agents/claude-code` 技能，验证 `claude` 在 PATH 上，然后开始使用它执行代码。从下一条消息起，任何编码相关操作（读文件、写代码、跑测试、提交、推送）都在底层通过 Claude Code 路由。

两点须知：
- 激活前确保 `claude` 在你的 PATH 上。`which claude` 应该打印出真实的二进制路径。
- Claude Code 有 print 模式（一次性，快，无 TUI）和交互模式（完整 tmux 会话）。程序员根据任务自行选择。

##### 定制设计师：教它你的视觉风格

当设计师能以你的风格生成图像——而非通用 AI 输出时，它才真正有用。模式：喂入参考设计，让它学习，然后要求它创建一个能以相同风格生成新图像的技能。

这是自进化循环被用作设置机制的例子。你不是手写技能，而是给 Agent 看好的例子，让它自己编码这个模式。

开启设计师会话，粘贴你的参考图像（CLI 中拖放，或在 Telegram 中附加），然后发送：

```markdown
Carefully study these reference illustrations. Note the color palette,
line weight, level of detail, composition, and overall aesthetic.

I want you to create a new skill called "my-design-style" that captures
this visual style. The skill should:

1. Document the style fingerprint in plain language (palette, line
   weights, composition rules, recurring motifs)
2. Include a Python script that takes a text description of a new
   illustration and generates the image using the Nano Banana model
   (google/gemini-2.5-flash-image) via the OpenRouter API in this style
3. Read OPENROUTER_API_KEY from the environment

Use skill_manage to create it. Test the generated script on a sample
prompt before saying it's done.
```

设计师会研究参考作品，编写 SKILL.md，生成 Python 脚本，保存到 `~/.hermes/profiles/designer/skills/my-design-style/`，并验证脚本能运行。

如果你在 `hermes setup` 时选了 OpenRouter 作为提供商，密钥已经通过 `--clone` 在设计师 Profile 的 .env 中了。如果没有，添加一次：

```bash
hermes -p designer config set OPENROUTER_API_KEY <your-key>
```

此后，向设计师要求新插图会触发该技能。它根据你的风格指纹编写提示词，通过 OpenRouter 调用 Nano Banana，保存输出。

同样的模式适用于任何风格特定的输出——Newsletter 开头、X 帖子、代码审查评论——任何需要一致性的地方。

#### 定时任务：用自然语言写 Cron

研究员的 SOUL.md 说它负责每日 Telegram 摘要。这意味着一个按自己的时间表运行的任务，不需要你记着去问。这就是 Hermes cron 的用途。

Hermes 自带内置调度器。网关守护进程每 60 秒 tick 一次，在隔离的 Agent 会话中运行任何到期的任务，并将输出递送到你指定的消息平台。任务跨重启存活。它们存储在 `~/.hermes/cron/jobs.json`，输出存储在 `~/.hermes/cron/output/`。

有趣的是：**你不写 cron 表达式。你用英语描述你想要的，Hermes 转换它。**

##### 设置研究员的每日摘要

开启研究员会话并发送：

```markdown
Every weekday at 8am India time, prepare a deep digest of what's new
in the AI and machine learning space over the last 24 hours. Cover
four streams in this order:

1. Trending GitHub repos (especially new AI/ML tooling)
2. Big tech and lab announcements (Anthropic, OpenAI, Google, Meta,
   xAI, Nous, etc.)
3. Fresh research papers worth reading
4. Social pulse from X, Reddit, and Hacker News

Lead with what changed since yesterday. Cite every claim with a URL.
Keep it under 800 words. Deliver to Telegram.

Set this up as a recurring cron job.
```

研究员使用其 cronjob 工具创建任务，递送目标默认为当前聊天（这里是 Telegram），调度器接管。验证已创建：

```bash
hermes -p researcher cron list
```

你应该能看到带有下次计划运行时间的任务。明天早上 8 点，你的 Telegram 会收到摘要。无需进一步操作。

##### 其他有用模式

cron 语法很灵活。几个值得了解的变体：

- **一次性延迟**：`/cron add 30m "Remind me to check the build"` 30 分钟后运行一次
- **循环间隔**：`/cron add "every 2h" "Check server status"` 每两小时运行
- **标准 cron 表达式**：`/cron add "0 9 * * 1-5" "..."` 精确控制（工作日上午 9 点）
- **技能附加**：`/cron add "every 1h" "Summarize new feed items" --skill blogwatcher` 运行前加载技能

你还可以链式组合任务。一个 cron 的输出通过 `context_from` 标志成为下一个 cron 的输入。对于多阶段自动化（研究步骤喂给写作步骤）很有用。

---

全文完。感谢阅读。欢迎在评论中告诉我你希望我接下来覆盖什么内容。

如果你更适合视频学习，我将在几天后在 YouTube 和 X 上发布完整的 Hermes Agent 实操演示。敬请期待！
