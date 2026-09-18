---
title: "AI 原生 SDLC Playbook：Anthropic 官方软件研发全流程变革指南"
date: 2026-08-25
author: repost
categories: [转载, AI工程方法论]
tags: [翻译, AI工程方法论, SDLC, Claude-Code, 转载]
---

> **摘要**：这是 Anthropic 官方发布的 AI 原生软件开发生命周期（SDLC）实战手册。文章的核心论点是：当 AI 让代码编写不再是瓶颈，瓶颈就转移到了构建阶段两侧仍以人类速度运转的环节——规划、评审/测试和部署。手册将传统线性 SDLC 重构为一个闭环，以「提交的产物（committed artifact）」为贯穿主线：intent.md → spec.md → plan.md → diff 与测试 → PR 与评审 → 生产监控，每个阶段的产物进入版本控制并自动触发下一阶段。手册按 Plan / Design / Build / Test / Deploy / Maintain 六个阶段给出了模块化的「打法（play）」，每个打法都包含前提条件、落地步骤、治理考量和先导/滞后衡量指标，并附有 intent.md、plan.md、CLAUDE.md、Skills、Hooks、REVIEW.md、evals 流水线等可直接套用的配置示例。对任何正在规模化落地 Agentic Coding 的工程团队，这是一份体系级参考。

> **转载声明**：本文**翻译整理**自：《The AI-Native SDLC playbook | Claude by Anthropic》。**内容版权归原作者及原出处所有**，本文仅供学习交流，**所有观点均属原作者，不代表本站立场**；如有侵权，请联系删除。


---

#### AI 原生 SDLC Playbook

#### 代码不再是瓶颈

各组织已经开始用 AI 以前一年前难以想象的速度写代码，但围绕代码的流程却没有以同样的速度改变。

很多工程团队仍然沿用原来的审批门禁、评审、交接和政策，这些都在拖慢 [Claude Code](https://claude.com/product/claude-code) 这类 Agentic 编码方案带来的生产力提升。

软件开发生命周期（SDLC，Software Development Lifecycle）是把软件从想法带到生产的全过程。大多数组织运行的都差不多是同样的六个阶段：规划、设计、构建、测试、部署和维护。传统上，每个阶段是一个由不同角色拥有的独立环节。产品经理写需求，技术架构师把需求变成设计，工程师照着设计实现，强监管企业的 QA 团队做验证，发布团队上线，运维负责监控线上运行。工作通过文档、工单和签字确认在阶段之间流转。

传统 SDLC 流程繁重，是为了在每一步保证可追责和可控。但传统 SDLC 是为一个时代的高效而设计的——那个时代最耗时、最昂贵的阶段是编写和实现代码，而现在情况已经变了。PRD、估算仪式、产品安全评审，都是为了在长达数周、数月甚至一个季度的开发周期中强制对齐而存在的。

传统 SDLC 的各种控制手段还隐含一个假设：每一步都由人执行。而那些产出价值最高的组织，已经围绕 Agentic AI 当前的能力重建了流程，同时确保人类仍在闭环之中。在本指南中，我们分享 Applied AI 团队在各 SDLC 阶段内部集成 Claude 的若干最佳实践，用于加速开发、让流程跑得更快——这些实践也源自我们与客户的合作。

当代码不再是瓶颈、构建阶段跑得比传统 SDLC 允许的速度更快时，三件事成为现实：

- 瓶颈转移到构建阶段左右两侧的环节，主要是规划、评审/测试和部署——它们仍以人类速度运行。
- 控制手段开始与现实脱节、变得难以执行。当一行行代码是人写的，逐行人审是合理的；但当 diff 大部分由 agent 写出来时，人审就跟不上了。
- 治理成本上升，因为各种例外仍然要流经每周或每月才开一次会的会议和委员会。

构建不再是约束——它周围以人类速度运转的环节才是。人类速度的阶段保持原长度，而构建塌缩为几个小时。

拿安全瓶颈举个例子。安全团队的规模是按人类产出配置的，所以当 agent 把代码产出放大数倍时，要么评审队列堆积，要么代码在评审不足的情况下就上线了。强监管组织无法接受任何一种结果，所以它的安全和政策检查必须跟上 agent 的节奏。

为了真正兑现 Agentic AI 的生产力红利并保证其安全，传统 SDLC 生命周期需要经历与实现阶段同等级别的变革。

##### 什么是 AI 原生 SDLC？

AI 原生 SDLC 是一个重新设计的过程：把旧的控制目标与新的执行手段结合。流程不再是线性流转，而是变成一个环，AI 嵌入在每个节点上。AI 原生 SDLC 推动自动交接、自动触发后续打法，从而解决传统 SDLC 各阶段之间手工、笨重的交接问题。

打法按所属阶段列出；箭头给出的是采纳顺序，两者并不相同。从任意一个「粘土（clay）」打法开始即可——没有任何箭头指向它，说明它不需要前置依赖。对其他任何打法，指向它的箭头就是需要先采纳的打法。

##### 关键转变

下表列出了传统 SDLC 与（由 Claude 支撑的）AI 原生 SDLC 这两个光谱端点之间的对比。大多数组织处于两列之间的某个位置。

| 阶段 | 传统 SDLC | AI 原生 SDLC |
| --- | --- | --- |
| Plan（规划） | 由委员会收集需求，通过工作坊和签字层层提炼，人工撰写成文 | Claude 直接从原始信息源综合痛点，沉淀到 `intent.md` 中——人能读、机器能执行 |
| Design（设计） | 分析师写规格书，设计师再解析规格书 | 需求与设计压缩为与 agent 的一次工作会话，由编码为 Skills 的标准约束，并纳入 git 版本管理 |
| Build（构建） | 测试与代码手写，文档在主体开发完成之后补 | 测试与代码由 AI 生成，组织知识以版本化、机器可读的 `CLAUDE.md` 文件和 Skills 维护 |
| Test（测试） | QA 门禁设在阶段边界 | 持续的 evals（评测）编织进实现过程 |
| Deploy（部署） | 人类逐行评审代码，治理在评审周期中进行且常常不一致 | 多层 agent 评审 + 人类评审仅保留给强监管和关键代码。治理在 AI 行动时就被执行，hooks 充当审批门禁 |
| Maintain（维护） | 人类盯生产环境找 bug | Agent 监控线上部署。任何控制带被突破都会被诊断并回写为新的 `intent.md` 进入闭环 |

贯穿右列的主线是**被提交的产物（committed artifact）**。每个阶段以向版本控制写入一份产物结束（包括 `intent.md`、`spec.md`、`plan.md`、diff 及其测试、带着评审发现的 PR、以及事故记录），下一个阶段以读取它开始。在早期阶段，.md 文件是主要产物，因为产品负责人和 agent 都能读同一个文件并据此行动。从 Build 开始，产物就是代码及其记录。提交链同时也是审计轨迹：谁要求了什么、agent 产出了什么、谁批准了它。

人类仍然对每一个需要判断的决策负责。在 Agentic SDLC 的世界里，人类的注意力随着「必须被评审的产物」一起转移。

每个阶段都提交一份下一阶段可读的产物。意图、规格、计划、diff 与评审发现合在一起，就是审计轨迹。

#### 打法（Plays）

打法是本手册的核心，分为六个非线性的阶段（Plan、Design、Build、Test、Deploy、Maintain），合起来覆盖完整生命周期。

每个打法都包含：

- 变化的是什么；
- 如何起步；
- 具体的实施步骤；
- 治理考量；以及
- 如何衡量它是否生效。

这些步骤是模块化的，组织可以根据自身需要选择在不同时间优先变革不同阶段。每个打法都在「前提条件（Prerequisites）」下列出依赖，依赖图进一步说明了这些依赖关系。

一个阶段以提交一份产物结束，这次提交会启动下一个阶段。一份被接受的 `intent.md` 触发需求与设计过程，一份被批准的 `spec.md` 触发 plan 模式，一个被合并的 PR 触发流水线，生产环境中被突破的控制带写下下一份 `intent.md`——循环如此继续。

一开始，你手动 prompt 每一步；终态是一个闭环，每份被接受的产物自动触发下一道门禁。人类的注意力集中在门禁处，评审 agent 标记出来的内容，而不是每个阶段都从零开始。

打法按所属阶段列出；箭头给出的是采纳顺序，两者并不相同。从任意一个「粘土（clay）」打法开始即可——没有任何箭头指向它，说明它不需要前置依赖。对其他任何打法，指向它的箭头就是需要先采纳的打法。

**01**

#### Plan（规划）

想法不再等待某人来把它们写成文档。意图只被捕捉一次，用发起人自己的语言，作为一份受版本控制、下一阶段可直接执行的产物。

##### 以 intent.md 捕捉

启动软件开发流程的 `intent.md` 可以从不同路径进入：某人有一个想法、一个工单被提交、或者一次告警暴露了一个事故（见阶段 6：维护）。

当一个人有想法时，他与 Claude 头脑风暴并产出一个 markdown 原型规格（proto-spec）。在传统 SDLC 里，同一个人接下来必须说服产品团队的某个成员替他或陪他把这个想法写成正式文档。

Claude 生成的原型规格人类可读、受版本控制、且下一阶段可以立即消费。原型规格被保存为 `intent.md`。

无论意图来自事件触发还是 agent，步骤都一样：产品负责人在 `intent.md` 提交之前评审并修正 agent 写的这份文件。

> **传统**：一个想法要经过 backlog 条目、用户故事、故事点、提炼会议，才有人能对它采取行动。每次交接都发生所有权转移，所以到达工程团队的东西已经和发起人的本意隔了好几层。
>
> **AI 原生**：发起人与 Claude 头脑风暴，把结果用自己的语言写成 `intent.md`，一份原型规格。这份产物包含想要什么、为什么、以及受哪些约束。重复性流程通过 Skills 编码沉淀。

##### 起步

**前提条件**

无。

**基础设施**

非工程师人群的 Claude 访问权限（claude.ai 或 [Cowork](https://claude.com/product/cowork)）；一份约定好的 `intent.md` 模板；一个共享的、受版本控制的意图存放地，由产品负责人盯守。对单一产品来说，最简单的存放地是产品仓库里的 `intent/` 目录。这种做法让产物链紧挨着由它派生的代码。只有当意图横跨多个仓库时，专门的意图仓库才值得这份开销；在 monorepo 里它就是一个目录。阶段 3（Build）的侧栏会讲这个存放地与已经在承载记录的 Jira 或需求工具之间的关系。

搭建是一次性的平台/工程团队任务。需要一名技术团队成员把意图存放地建起来并决定谁可以写入，因为很多贡献者会来自组织各处。

仓库建好之后，没有 git 经验的贡献者不需要直接用 git：通过版本控制系统（如 GitHub）的连接器，Claude 可以替他们从 claude.ai 或 Cowork 提交 markdown 文件。

###### 如何执行

1. 发起人用自己的语言向 Claude 描述问题。他可以说今天做不了什么、谁受这个想法影响、更好的状态长什么样、什么不在范围内。不要求任何正式语言。
2. 头脑风暴直到想法足够具体。Claude 会问分析师会问的问题：范围、用户、约束、以及成功是什么样子。
3. 让 Claude 按组织模板把结果写成 `intent.md`——模板可以由技术团队成员编码为 Skill、并由负责人签字生效。内容可覆盖问题、期望产出、受影响的用户与系统、约束和开放问题。
4. 发起人修正 Claude 理解错的地方。
5. 把 `intent.md` 提交到共享存放地。作者与时间戳加入记录，产品负责人从那里接手。

```markdown
# Intent: claims status self-service
Author: J. Ortiz (claims operations). Status: draft.

## Problem
Customers phone the contact center to ask where their claim is.
Handlers spend roughly a third of call time on status-only queries.

## Proposed outcome
Customers see claim status, next step and expected date in the portal.

## Affected users and systems
Claims handlers, portal team, claims-core API.

## Constraints
No new PII in the portal session. Existing authentication only.

## Open questions
Do third-party loss adjusters need access too?
```

###### 治理考量

证据就是那份被提交的 `intent.md`，上面列有作者、时间戳和完整的修订历史，记录在意图存放地的 git 历史中。产品负责人做批准，「接受或拒绝」这一把意图送入阶段 2（Design）的决策，以 merge 或关闭评审的形式被记录下来。

##### 如何衡量

**先导指标**

从第一次对话到 `intent.md` 被提交的时间，从意图存放地的 git 历史读取（其中记录了作者与时间戳）。预期是从数周的需求引导与提炼周期降到数小时。

**滞后指标**

存活率，即被产品负责人接受进入阶段 2（Design）而非被关闭的 `intent.md` 占比。接受/拒绝决策记录为产物的 merge 或评审关闭。此外，还可以看同一变更的 `spec.md` 首次提交之后，`intent.md` 还被改动过多少次。

**02**

#### Design（设计）

需求与设计合并为一次会话。政策在写规格的同时被应用，而不是几周后在评审中才被发现。

##### 需求与设计

一旦产品负责人批准，Claude 接过被接受的 `intent.md`，产出需求与设计规格书。这个过程由组织在品牌、安全、合规、UX 方面的 [Skills](https://code.claude.com/docs/en/skills) 引导。

产品负责人评审规格书，但不亲自写。这个流程的目标是产出一份工程团队可以据以规划的规格书，并标注出关注点。

前端工作是最清晰的例子。`intent.md` 被接受后，产品负责人在 [Claude Design](https://claude.com/product/design)（beta）中据 `intent.md` 做出设计 mock，在 mock 上迭代，然后导出到 Claude Code 去构建。

> **传统**：需求和设计是由不同团队运行的两个独立阶段。分析师把想法形式化为需求，设计师再把这些需求解析回设计。这种分离是为了可追责，但既慢又有信息损耗。
>
> **AI 原生**：两个阶段发生在一次被 prompt 的会话里。Claude 拿着 `intent.md` 产出需求与设计规格书，受组织 Skills 约束，并标注出关注点。

##### 起步

**前提条件**

写出 `intent.md` 文件；品牌、安全、合规、UX 政策写为 Skills。

**基础设施**

一名有 Claude 访问权限的产品负责人。不要求任何工程技能。

###### 如何执行

1. 产品负责人打开一个加载了组织 Skills 的会话，附上 `intent.md`。
2. 产品负责人的 prompt 指向 `intent.md`，点名约束，并要求标注关注点。先手动跑，之后把它固化成组织级 slash 命令。再往后，把意图存放地里 `intent.md` 的被接受作为触发器：merge 时触发一个非交互任务，加载组织 Skills 跑一遍，并把 `spec.md` 作为 PR 提交（阶段 5 Deploy 的 CI/CD 打法覆盖相关管道）。到那一步，产品负责人的第一次介入就是评审。
3. 同一位产品负责人对照原始想法评审规格：规格是否解决了陈述的问题？`intent.md` 中的开放问题是被回答了还是被合理携带？
4. 优先处理被标注的关注点，因为它们正是分析师本来会上报的点。在工程团队看到规格之前，产品负责人逐个与对应的政策负责人解决。
5. 把 `spec.md` 与 `intent.md` 一起提交。这对文件记录了「要的是什么」和「决定了什么」。
6. 产品负责人决定规格与意图是否进入构建，对组织认定为较高风险的事项咨询技术负责人。这个决定永远由人类队友做出，而接受规格这个动作就启动阶段 3（Build）中的 plan 模式打法。

###### 长什么样（prompt）

```markdown
Read the attached intent.md and produce a requirements and design spec for integrating it into our existing codebase. Apply the skills available to you so the plan conforms to our brand guidelines, security policies and UX standards. Document the spec fully as spec.md, ready to hand to the engineering team. Describe clearly any areas of concern, especially where you cannot satisfy contradicting policies.
```

###### 治理考量

政策不再是几周后在评审中才被发现，而是在写规格的同时被实时读取并应用。组织的 Skills 作为规格的约束条件生效。规格、产出它的 prompt、当时生效的 Skill 版本，全部记录在版本控制中。产品负责人签署规格，并把标注的关注点路由给对应的政策负责人。

##### 如何衡量

**先导指标**

同一变更从 `intent.md` 提交到 `spec.md` 提交的间隔时间（两个 git 时间戳），与旧的「需求 + 设计」周期对比。

**滞后指标**

构建开始后的需求返工。统计同一变更中日期晚于首个 `plan.md` 提交的 `spec.md` 提交次数，git log 可以直接给出。

**03**

#### Build（构建）

没有被接受的计划，就不会有任何实现。组织知识变成 agent 可读的文件，护栏以代码而非习惯的方式运行。

##### Claude Code plan 模式作为默认起点

工程师以 [plan 模式](https://code.claude.com/docs/en/permission-modes)启动 Claude Code 会话，把阶段 2（Design）批准的 `spec.md` 交给 Claude，让它反过来「面试」自己，反复迭代计划直到工程师满意。

> **传统**：工程师读完设计就开始写代码。这个改动怎么做——具体到改哪些文件、写哪些测试——都留在工程师脑子里，顶多记在工单评论里。没有别人能评审它。评审者看到的第一样东西就是完成的 diff，而那时返工已经很昂贵了。
>
> **AI 原生**：工作从 Claude 在 plan 模式下产出的一份书面计划开始——plan 模式下它只读代码库、不做任何修改。工程师在代码写出之前修正计划，批准后的版本作为 `plan.md` 提交，供后续阶段对照检查。

##### 起步

**前提条件**

已有的意图产物（`intent.md` 或 `spec.md`）；有 `CLAUDE.md` 文件会有帮助。

**基础设施**

能访问仓库的 Claude Code。

###### 如何执行

1. 工程师以 plan 模式启动与 Claude 的会话。
2. 工程师把 `intent.md` 和 `spec.md` 给 Claude，要一份实现计划：点名要改的文件、工作的顺序、以及证明它生效的测试。
3. 审问这份计划：这个改动可能弄坏什么？哪一步风险最高？Claude 还有哪些没选的方案？
4. 迭代直到一个从没见过这段对话的工程师也能只凭计划实现这个改动。
5. 把批准的计划提交为 `plan.md`。计划加入审计轨迹，PR 评审打法（阶段 5：Deploy）会拿最终的 diff 对照它。
6. 接受计划，让 Claude 实现。计划足够扎实时，实现往往一遍就过。
7. 当实现偏离计划时，在同一提交中更新 `plan.md`。可以考虑用 hook 强制两者同步。

###### 长什么样（plan.md）

```markdown
# Plan: claims status self-service (from intent.md 2026-06-02)

## Files that change
portal/src/claims/StatusPanel.tsx (new), claims-api/routes/status.py,
claims-api/tests/test_status.py

## Order of work
1. Add the status endpoint behind existing auth.
2. Panel against the endpoint.
3. Wire into the portal nav.

## Risks
The claims-core API rate-limits at 50 rps; the panel must cache.

## Proof
test_status.py covers the four claim states; screenshot matches the
approved mock.
```

###### 治理考量

设计评审发生在任何代码生成之前，此时调整方向还只是改一份文档的事。plan 模式本身就强制了这一点：工程师不接受计划，Claude 就不能编辑文件。计划及其修订、以及谁接受了它，都被记录。例行变更由工程师本人批准，组织认定为较高风险的事项交给技术负责人或架构师。

##### 如何衡量

**先导指标**

从第一次实现就直接合并的变更占比；从计划批准到 PR 合并的时间（数据在 PR 元数据中）。

**滞后指标**

每个变更的返工轮数（同样来自 PR 元数据），以及合并的 diff 与提交的 `plan.md` 的吻合频率。

##### Claude Code 自动模式

Claude Code 也可以运行在 auto 模式下：工程师批准计划、迭代满意之后，Claude 应用每个改动而不再逐次请求确认。随着后续打法中的护栏成熟（调教好的 `CLAUDE.md`、编码政策的 Skills、阻断危险操作的 hooks、以及 Claude 可以运行的测试套件），自动接受成为例行工作的默认选项：紧凑的 `spec.md`、小的爆炸半径、测试已覆盖的代码。

重心正从「用户盯着 agent 做编辑、逐个评审动作」转向「较长自主会话之后对产物的评审」。auto-accept 模式配合 worktree 进一步让个体和团队获得并行能力，它是 SDLC 自主运行、并如阶段 6（维护）所述真正闭环的基础。

##### 侧栏：遗留系统与事实源

*适用于流程产出的每一份产物。*

既有的 SDLC 流程很可能已经在跟踪这些产物，只是不在 markdown 文件里。工作项可能在 Jira，需求在带监管追溯性的工具里，设计在 Figma，变更审批在变更委员会。这些系统很难被取代——审计方和监管方已经接受它们，其他团队也依赖它们——所以 AI 原生 SDLC 必须绕开现状、与之共存。

向 AI 原生 SDLC 过渡时，对流程产出的每一份产物，指定一个系统作为事实源（source of truth），其余系统只保存副本或指向原件的链接。下面几种配置都可以做到单一事实源，选择因产物而异：

**仓库作为事实源。** markdown 产物是权威记录，遗留系统引用某个提交里的文件。对工程主导的组织，这可能是最干净的配置：所有记录都在一个工具里、只有一套时间戳权威。

**遗留系统作为事实源。** Jira、ServiceNow 或需求工具持有权威记录，markdown 产物是工作副本。Claude 在会话开始时读取记录，并在产出规格或计划的同一会话里通过 [MCP](https://code.claude.com/docs/en/mcp) 连接器把结果写回去。

**链接是最低门槛。** 所有产物注明记录 ID，所有遗留记录包含 markdown 文件的 commit SHA。在向 AI 原生 SDLC 过渡时，接受暂时存在两个事实源、先做好链接，是不错的起点。

遗留系统与 markdown 优先的系统可以共存，只要两者之间有链接、或者其一被明确宣布为事实源。

##### CLAUDE.md

[`CLAUDE.md`](https://code.claude.com/docs/en/memory) 给 Claude 提供一个新入职者所需要的上下文：约定、命令、架构，以及团队最常见的错误。过去存在于人们脑中和 wiki 里的知识，变成 agent 在每次会话开始时读取的文件，由整个团队维护，每次犯错都迭代它。

##### 起步

**前提条件**

无。

**基础设施**

一个仓库、装好的 Claude Code、和一名熟悉代码库的工程师。

###### 如何执行

1. 在仓库里跑 `/init`。Claude 从它看到的东西生成一份起步版 `CLAUDE.md`。
2. 把生成的文件裁剪到「新入职者第一天需要的」那么多。保留构建/测试/lint 命令、真正重要的约定、以及 Claude 反复出错的地方。
3. 把 `CLAUDE.md` 提交进 git 放在仓库根目录，让整个团队共享一个版本，改动像代码一样走评审。
4. 一条实用规则：当 Claude 同一个错误犯第二次，就把纠正写进 `CLAUDE.md`。
5. 保持在一页以内——Claude 每次会话开始都要全文读取，任何过期的内容都在白白占用上下文。

###### 长什么样（CLAUDE.md）

```javascript
# Payments service

## Commands
- Build: make build
- Test: make test (unit), make itest (integration, needs docker)
- Lint: make lint (runs in CI; fix before pushing)

## Conventions
- Java 21, Spring Boot 3. No new Lombok.
- Money is always BigDecimal, never double.
- Every endpoint needs an integration test in src/itest.

## Architecture
- api/ holds REST controllers, core/ holds domain logic,
  adapters/ talks to external systems.
- Kafka events are defined in schemas/; never edit generated classes.

## Things Claude gets wrong
- Do not bump dependency versions; the platform team owns them.
- The legacy v1/ package is frozen; changes go in v2/.
```

###### 治理考量

`CLAUDE.md` 受版本控制，所以 agent 所遵循的指令是可评审、可审计的。团队约定通过这个文件被执行，对它的改动记录在 git 历史中，由 code owner 在 PR 评审中批准。

##### 如何衡量

**先导指标**

Claude 重复犯那些 `CLAUDE.md` 本应拦住的错误的频率。对 `CLAUDE.md` 的修正与改动应在 git 历史中跟踪。

**滞后指标**

团队新成员从加入到首个 PR 合并的时间，来自 PR 历史。

##### Skills 即组织知识

Skills 是组织把自身组织知识（institutional knowledge）变得可运营的方式。这些指令是显式的、受版本控制的、被广泛应用的，且在政策变化时集中更新。经验法则：必须被一致应用的组织知识写成 Skill；属于 `CLAUDE.md` 或某个 prompt 的内容不要写成 Skill。

##### 起步

**前提条件**

无硬性要求。有 `CLAUDE.md` 会有帮助，因为它把 agent 的工作知识留在仓库里，但 Skill 并不依赖它。

**基础设施**

一项有明确负责人的政策，以及一份书面的事实源。

###### 如何执行

1. 挑一条今天执行得不一致的知识。可以是安全标准、API 设计约定，或品牌规范。
2. 把它写成一个 Skill：一个包含 `SKILL.md` 的目录，frontmatter 声明何时触发，正文说明要做什么。由一名工程师根据政策负责人的事实源撰写，可用 Claude 辅助。
3. 把 Skill 放进仓库的 `.claude/skills/<name>/`，随代码一起发布；或通过[插件市场](https://code.claude.com/docs/en/plugin-marketplaces)做组织级分发。
4. 测试 Skill 是否触发。用不同的方式让 Claude 做相关任务，确认每次都加载了该 Skill。
5. 政策变化时，修改 Skill 并由政策负责人签字确认。
6. 工程师在下一个会话中自动用上新版本。

###### 长什么样（.claude/skills/secure-api-review/SKILL.md）

```markdown
---
name: secure-api-review
description: Apply the API security standard. Use whenever creating or
  modifying an external-facing endpoint, reviewing API code, or
  generating an OpenAPI spec.
---
# Secure API review

When you create or change an API endpoint:
1. Authentication: every endpoint requires the gateway JWT;
   no anonymous routes outside /health.
2. Input validation: validate request bodies against the OpenAPI
   schema and reject unknown fields.
3. Audit: every state-changing endpoint emits an audit event with
   actor, action, entity and timestamp.
4. Data classification: fields tagged pii in the schema must never
   appear in logs or error messages.

Run scripts/check-endpoints.sh and include its output in your summary.
```

###### 治理考量

Skill 是一种控制手段，不过是建议性的。它让 Claude 更可能在写代码时就应用政策，但没有什么强制某个会话遵守它。必须永远成立的政策需要 Skill 之外确定性的东西兜底，比如阻断操作的 hook，或在 PR 处再次检查政策的评审流程。Skill 让违规变罕见，hook 让违规几乎不可能。Skill 的调用记录在会话轨迹中，政策负责人像评审代码一样评审 Skill 的变更。

##### 如何衡量

**先导指标**

从政策负责人批准政策变更，到更新后的 Skill 被合并的时间，取自 Skill 目录上的 PR。

**滞后指标**

引用该政策的 PR 评审发现数——Skill 在写代码时就应用政策之后，这个数应该趋近于零。如果不趋于零，要么 Skill 没触发，要么它的文本已经和官方政策漂移了。

##### Hooks 作为构建期护栏

Skill 是建议性控制，而 [hook](https://code.claude.com/docs/en/hooks) 是它背后确定性的那一层。Claude 在实现期的动作大多是文件编辑和 shell 命令，所以构建阶段是 hook 触发最密集的地方。

构建期的 hook 可以：

- 阻断对受保护路径的编辑，比如生成的类或冻结的包；
- 在文件编辑后运行格式化和 lint，让漂移永远不累积；
- 把凭据挡在 diff 之外。

凡是政策要求无例外成立的 Skill，都要有 hook 兜底。hook 会在每个匹配它的动作上运行，所以构建期 hook 应当快速、且只作用于被改的文件。更重的检查（如全量测试套件）应放在 commit 或 PR 处。

需要人类批准的 hook 属于阶段 5（Deploy）的门禁——构建过程中弹审批提示，等于把人重新放回所有并行会话的关键路径上。

##### 并行会话与 subagent

一名工程师可以同时驱动多条工作流。

并行会话是另一个完整的 Claude Code 实例，在自己的 [git worktree](https://code.claude.com/docs/en/worktrees) 里处理独立任务。各个独立会话彼此一无所知，唯一共享的是主导它们的工程师。

[subagent](https://code.claude.com/docs/en/sub-agents) 在单个会话内部运行，是一个有自己上下文窗口和工具限制的受限帮手，适合在多个任务里反复出现的工作，比如验证应用是否按预期运行。

并行会话提高工程师同时在途的任务数，subagent 保持每个会话聚焦在自己的任务上。工程师的工作是给它们掌舵和评审。

> **传统**：一名工程师一次做一个任务，一天或一周里有相当一部分时间耗在等构建、等测试、等评审者。等待期间切换任务当然可以，但上下文切换累到很少有人真的这么做。
>
> **AI 原生**：一名工程师同时跑多个 Claude 会话，每个在自己的 worktree 里做自己的任务。重复性工作变成有独立上下文和工具限制的 subagent。工程师的工作转向编排——最终转向构建和监控闭环。

##### 起步

**前提条件**

`CLAUDE.md`（所有会话都会读它）。反馈回路（阶段 4：Test）也有帮助——当会话能自己验证自己的工作时，需要的监督就更少。

**基础设施**

一个 git 仓库（隔离来自 worktree），以及调好的权限设置，让会话不会卡在组织认为安全的命令上等审批。

###### 如何执行

1. 工程师把工作拆成触及不同文件的任务，用 plan 模式打法（阶段 3：Build）产出的计划看哪里是独立的。共享文件的任务放在同一个会话里串行执行。
2. 每个并行任务有自己的 worktree，比如一个终端里 `claude --worktree feature-auth`，另一个里 `claude --worktree fix-rate-limit`。worktree 是一个独立分支上的独立检出，避免会话在文件上撞车。
3. 两三个会话是合理的起点。实际上限是一个人能认真评审多少条流——只在评审跟得上时才加会话。
4. 把重复性工作变成 subagent：定义在 `.claude/agents/` 下的 markdown 文件，各带名字、何时使用的描述、以及可以碰的工具。比如主 agent 完成后剥离多余复杂度的简化器、运行应用并检查行为的验证器、探索代码库并汇报而不淹没主上下文的研究者。把这些定义提交进 git，让全团队共享。

###### 长什么样（.claude/agents/verifier.md）

```javascript
---
name: verifier
description: Runs the app and checks the change works before the session
  reports done
tools: Bash, Read
---
Start the app with make run. Exercise the changed behavior and the two
nearest neighboring flows. Report what you ran, what you saw, and any
behavior that does not match plan.md. Do not fix anything; report only.
```

###### 治理考量

会话更多意味着产出更多，所以控制必须来自仓库里的配置。Hooks 和权限设置在那里对所有会话生效，会话做了什么都有日志、并归属到运行它的工程师名下。

##### 如何衡量

**先导指标**

评审质量保持前提下的、每名工程师的并发会话数（从 OpenTelemetry 导出统计），以及一天中用于掌舵而非等待的时间占比。

**滞后指标**

每人每周合并的变更数，结合按 PR 历史确定的返工率一起看。

**04**

#### Test（测试）

每个会话在人类看到结果之前先自查，而引导 agent 的配置本身也像它写的代码一样接受回归测试。

##### 给 Claude 一个反馈回路

永远给 Claude 一种验证自己工作的方式——测试、构建或截图 diff 都行。会话自查其工作、在人看到之前修掉自己的错误。

反馈回路不要与 verifier subagent（阶段 3：Build）混淆。反馈回路贯穿整个任务、随工作反复运行；verifier subagent 则是封装「最终检查」的一种方式——当会话认为工作完成后，开一个全新上下文窗口跑一遍，这样裁决不会被产出代码时的假设带偏。

> **传统**：「代码能跑」这个信号来得很晚：CI 晚几分钟，测试人员晚几天，生产环境晚几周。当代码由 agent 产出，迟到的信号意味着必须有一个人检查它的全部产出，而这个人就成了瓶颈。
>
> **AI 原生**：会话在人看到之前就有办法自查。跑测试、跑构建、截截图。Claude 迭代直到检查通过，所以到达工程师手里的东西已经通过检查了。搭建这个回路是运行会话的工程师的职责，下面的步骤就是为他们写的。

##### 起步

**前提条件**

无。

**基础设施**

一条命令就能本地跑起来的测试套件和构建。对 UI 工作，让 Claude 能「看到」结果的方式至关重要——浏览器工具或通过 MCP 接入的截图工具。

###### 如何执行

1. 如果今天检查工作需要一串命令加一些环境知识，把它包成单个目标，如 `make test` 或 `npm test`，失败时以非零码退出。
2. 在 `CLAUDE.md` 的 Commands 段落里列出每条命令，并附一个健康输出的示例。
3. 陈述目标时要可量化，让 Claude 不用问你就能检查，比如：「test\_status.py 里所有测试通过」「截图与附带的 mock 一致」「端点返回 200 且带新字段」。
4. 修 bug 时先写失败的测试。让 Claude 把 bug 复现成测试，运行并确认它按你预期的方式失败。提交这个测试。然后才让 Claude 在不修改测试的前提下让它通过——用最后一步的测试文件 hook 强制这个限制。一个先于修复存在、且 agent 改不动的测试，才是 bug 已消失的证明。
5. UI 工作用视觉检查闭环。给 Claude 浏览器或截图工具，给它 mock，让它迭代：实现、截图、对比、调整。两三轮很正常，结果应该每轮都在变好。
6. 把验证变成「完成」的一部分。指令写进 `CLAUDE.md`：报告任务完成之前先跑测试，并贴出输出。
7. 最后，回路本身需要保护——修代码的 agent 不能削弱对那段代码的检查。用一个在修复任务期间阻断测试文件编辑的 hook 实现。替代方案是在评审时检查 diff、拒绝任何触碰测试的改动。

###### 长什么样（CLAUDE.md 验证块）

```javascript
## Verifying your work

- Build: make build (must finish with "Build succeeded")
- Test: make test (all green; never skip or delete a failing test)
- Lint: make lint (zero warnings)

Run all three before reporting any task complete, and paste the output.
If a test fails, fix the code, not the test.
```

###### 治理考量

**强制执行什么**

任务报告完成前的验证，以及修复任务期间禁止 agent 编辑测试文件——两者在组织要求保证时都实现为 hook。

**证据是什么**

Claude 运行并粘贴的 `make test` 原始输出、构建日志或截图 diff——证据直接来自工具链。

**记录在哪里**

会话轨迹（由 OpenTelemetry 导出转发到组织的可观测性平台），以及 PR 的 check run——评审者和后来的审计者都能看到。

**谁批准**

评审 PR 的 code owner。机械性证据已经附上，他可以把注意力集中在意图和风险上。

##### 如何衡量

**先导指标**

agent 产出的变更的 CI 首次通过率，CI 系统本身已支持。

**滞后指标**

每个 PR 的评审时长（来自 PR 元数据）——当测试接住了评审者过去接住的问题，它应该下降——以及来自事故系统的变更失败率。

##### CI 中的持续 evals

Evals（评测）是 AI 原生版本的阶段门禁 QA。实践中，它意味着一套在 agent 配置变化时运行的测试套件。换上新模型或重写 prompt 时，eval 套件会告诉你 agent 是否仍以同样的标准完成工作。

应当把 evals 视为一个活的套件。随着模型进步，曾经有区分度的案例会失去区分度，必须基于持续监控补充新案例。

视使用场景，一些团队可能更愿意按固定节奏离线跑这些 evals，而不是每次变更都跑。下面的步骤针对持续评测。

##### 起步

**前提条件**

`CLAUDE.md` 和反馈回路（阶段 4：Test）。

**基础设施**

能以非交互方式运行 Claude Code 的 CI，以及预算充足的 API key。

###### 如何执行

1. 平台工程师从近期工作中收集 20 到 50 个真实任务，连同各自预期/可接受的结果。
2. 把每个任务写成一个 eval：prompt + 定义「可接受」的检查项（测试通过、lint 干净、行为不变、政策被遵守）。
3. 套件在 CI 中非交互运行：按计划调度，且在任何对 `CLAUDE.md`、Skills 或 hooks 的变更时触发——因为这些配置在引导 agent，理应享受代码级别的回归测试。
4. 用结果给配置变更设门。一个拉低通过率的 Skill 变更，合并前要被评审。
5. 每个生产事故都由处理它的团队写成一个 eval，作为回归测试留在套件里。

###### 长什么样（.github/workflows/agent-evals.yml）

{% raw %}
```yaml
name: Agent evals
on:
  pull_request:
    paths: ['CLAUDE.md', '.claude/**']
  schedule:
    - cron: '0 2 * * *'
jobs:
  evals:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm install -g @anthropic-ai/claude-code
      - name: Run eval suite
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          for eval in evals/*.json; do
            claude -p "$(jq -r '.prompt' $eval)" \
              --allowedTools "Read,Edit,Bash(make test)" \
              --output-format json > result.json
            ./evals/check.sh "$eval" result.json
          done
```
{% endraw %}

###### 治理考量

Evals 给了 QA 一个跟得上 agent 产出的门禁。通过率阈值作为合并检查被强制执行，运行被记录以便跨时间对比，配置变更由拥有它的团队批准。

##### 如何衡量

**先导指标**

eval 通过率随时间的变化（套件每次运行都汇报），以及一个生产事故变成永久 eval 所需的时间。

**滞后指标**

CI 中拦截住的回归与生产中发现的回归之比，数据来自事故系统。

**05**

#### Deploy（部署）

评审双向运行，治理在 agent 行动时就被执行。Agent 做生产门禁之前的一切，一步也不越过后面的门禁。

##### PR 评审闭环中的 AI

Claude 既给出评审也接受评审。它按组织政策评审传入的 PR，也在自己的 PR 上处理评审意见。这让工程师把 PR 评审的注意力集中在行为上——归根结底就是判断意图和风险。

> **传统**：评审容量是按人类产出规划的。PR 等待评审者全文读完，评审质量随评审者负荷波动，作者追着人跑而积压持续增长。
>
> **AI 原生**：所有 PR 得到完全相同的一组评审 pass，发现按严重程度排序。人类注意力上移一层：改动是否符合计划的意图、风险是否可接受。

##### 起步

**前提条件**

阶段 3（Build）产出的更新版 `CLAUDE.md`；若评审 pass 强制执行书面政策则需要 Skills；定义好的 subagent。

**基础设施**

装好 Claude 集成的仓库——管理员开启的托管 [Code Review](https://code.claude.com/docs/en/code-review) 服务（research preview），或在你自己的 CI 里跑 [claude-code-action](https://code.claude.com/docs/en/github-actions)，需要时模型调用走 AWS Bedrock、Google Vertex 或 Microsoft Foundry（CI/CD 打法覆盖部署选项）。值得同时配置要求 code owner 批准的分支保护策略。

###### 如何执行

1. 托管 Code Review 服务是 fastest start。管理员开启并选择仓库。当你需要掌控流水线、或希望 API 调用走自己的云协议时，用 claude-code-action 在自己的 CI 里跑评审（CI/CD 打法覆盖相关管道）。
2. 技术负责人在仓库根目录写 `REVIEW.md` 作为评审政策，按组织关心的 pass 划分：bug 与逻辑错误；安全与漏洞；对照规格（需求打法产出的 `spec.md`）、实现计划（plan 模式打法产出的 `plan.md`）和设计原则的合规性。`REVIEW.md` 还要定义什么算 Important、什么算 Nit，以及跳过什么。
3. 技术负责人设定人类门槛。发现本身不批准也不阻断 PR，分支保护仍然要求 code owner 批准。想按发现结果设合并门禁的平台工程师，可以读取 check run 发布的机器可读的严重度计数。
4. 当评审者或作者在评审评论里 @ claude，Claude 处理该评论并推送修复。PR 线程同时记录请求和改动。这个修复回路通过 claude-code-action 运行；托管服务里则评论 `@claude review` 请求一次全新评审。对 Claude 自己开的 PR，可以更进一步——让 Claude 看护 PR 直到合并。团队把这个回路包成自定义 slash 命令：清扫 PR 上未解决的评审评论和失败的检查、处理并推送修复，直到 PR 变绿、只等 code owner 批准。
5. 评审发现回流进 `CLAUDE.md`。当一次评审第二次标记同一个错误，纠正就作为该评审的一部分写进 `CLAUDE.md`；因为评审也读 `CLAUDE.md`，从下一个 PR 起这个错误就会被拦住。评审也会标记「某个改动让 `CLAUDE.md` 过时了」。
6. 每月一次，技术负责人通过给发现打分来调教评审者，并在 `REVIEW.md` 中限制 Nit 数量。生成的路径和 CI 已经强制的内容被排除在外。

###### 长什么样（REVIEW.md）

```markdown
# Review instructions

## Passes
Run three passes and tag each finding with its pass:
- Bugs: logic errors, broken edge cases, subtle regressions
- Security: injection risks, authentication gaps, PII in logs
- Compliance: the change matches spec.md, plan.md and our design principles

## What Important means here
Reserve Important for findings that would break behavior, leak data
or breach a policy. Style and naming are nits.

## Cap the nits
Report at most five nits per review; summarize the rest as a count.

## Do not report
Generated files under src/gen/ and anything CI already enforces.
```

###### 治理考量

职责分离得以保留：写代码的 agent 没有任何途径批准它。`REVIEW.md` 中的评审政策对所有 PR 生效，发现、修复、打分与批准都记录在 PR 历史中——PR 本身就是审计记录。批准来自分支保护之下的人类，由发现结果为其提供依据。

##### 如何衡量

**先导指标**

首次评审耗时（应降至分钟级），以及无需人类触碰分支就解决掉的评审评论占比，数据直接存在 Git 上。

**滞后指标**

合并前拦截住的缺陷与漏洞，对比逃逸到生产的数量，来自 PR 历史和事故系统。

##### Hooks 作为审批门禁

构建阶段把 hooks 用作护栏，在无人类参与的情况下放行或阻断动作（阶段 3：Build）。hook 还可以「提问」——暂停动作直到指定的人批准——这正是发布门禁需要的。

这个打法放在阶段 5（Deploy）是因为发布门禁是最清晰的场景，但 hooks 并非部署专属：Claude 在哪里行动，它们就在哪里运行。例如，hooks 可以在阶段 3（Build）中阻断没有变更单的迁移和基础设施编辑，也可以在阶段 4（Test）中阻止 agent 在修复任务期间编辑测试文件。

##### 起步

**前提条件**

无。

**基础设施**

一份书面清单：变更流程要求的各项人工审批。

###### 如何执行

1. 工程管理层会同变更管理与合规，列出必须保留的人工审批门禁：变更管理签核、发布授权、受保护路径的编辑等。
2. 平台工程师把每个门禁表达为一个 hook——一段在 Claude 行动前运行的脚本，可以放行、询问或阻断。
3. 团队级 hooks 放进 git 中的 `.claude/settings.json`；不可协商的 hooks 放进由平台或 IT 管理员拥有的 managed settings，个别工程师关不掉。
4. 阻断应当自我解释：hook 拦下一个动作时，原因和获取批准的途径要出现在 Claude 的输出里。

###### 长什么样（.claude/settings.json）

```json
{
    "hooks": {
      "PreToolUse": [
        {
          "matcher": "Bash",
          "hooks": [
            { "type": "command",
              "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/production-gate.sh" }
          ]
        }
      ]
    }
}
```

###### 门禁本身（.claude/hooks/production-gate.sh）

```bash
#!/bin/bash
# Production deploys require a named release authorization
cmd=$(jq -r '.tool_input.command' < /dev/stdin)
if [[ "$cmd" == *"deploy"* && "$cmd" == *"production"* ]]; then
   if [ -z "$RELEASE_APPROVAL" ]; then
     echo "Production deploys need a release authorization." >&2
     exit 2 # exit 2 blocks the action; the message goes to Claude
   fi
fi
exit 0
```

###### 治理考量

Hooks 就是审批门禁。门禁条件对每个人、每一次都强制执行。放行与阻断决策带时间戳记录。门禁还定义了什么算「批准」——一张已批准的变更单，或发布经理的签核。

##### 实战示例：强监管企业的 managed settings

*由平台团队通过 MDM 或管理控制台下发；工程师无法编辑或覆盖其中任何一项。*

```json
{
  "permissions": {
    "deny": [
      "Read(.env*)", "Read(./secrets/**)", "WebFetch",
      "Bash(curl *)", "Bash(wget *)"
    ],
    "allow": [
      "Bash(git *)", "Bash(make build)",
      "Bash(make test)", "Bash(make lint)"
    ],
    "disableBypassPermissionsMode": "disable"
  },
  "allowManagedPermissionRulesOnly": true,
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "allowUnsandboxedCommands": false,
    "network": {
      "allowedDomains": ["git.internal.example.com", "registry.npmjs.org"]
    },
    "credentials": {
      "files": [
        { "path": "~/.ssh", "mode": "deny" },
        { "path": "~/.aws/credentials", "mode": "deny" }
      ],
      "envVars": [
        { "name": "GITHUB_TOKEN", "mode": "deny" }
      ]
    }
  },
  "allowManagedHooksOnly": true,
  "disableSideloadFlags": true,
  "allowManagedMcpServersOnly": true,
  "strictKnownMarketplaces": [
    { "source": "github", "repo": "example-corp/approved-plugins" }
  ],
  "requiredMinimumVersion": "2.1.193"
}
```

**从控制角度看，每一行买到什么**

`permissions.deny` 把秘密挡在 agent 的上下文之外，并阻断通过工具发起的任意网络出口；`permissions.allow` 预先放行安全的内循环，避免 deny 清单变成 prompt 疲劳。

`disableBypassPermissionsMode` 加上 `allowManagedPermissionRulesOnly` 意味着任何工程师、项目文件或命令行参数都无法放宽这些规则。

`sandbox` 关上权限规则关不上的缺口。WebFetch 的工具级 deny 挡不住一条 shell 命令触网；OS 级的域名白名单则直接阻断出口。

`failIfUnavailable` 和 `allowUnsandboxedCommands` 把沙箱变成门禁：沙箱无法初始化时 Claude Code 拒绝启动，沙箱内失败的命令不能拿到沙箱外重试。

`credentials` 堵上 deny 规则留下的另一个缺口。`permissions.deny` 管的是 Claude 的文件工具，但沙箱内的 shell 命令默认仍能读 `~/.ssh` 或 `~/.aws/credentials`；这一块拒绝这些读取，并把点名的秘密从每个沙箱命令的环境中剥离。

`allowManagedHooksOnly` 意味着本打法的审批门禁是唯一运行的 hooks，本地没有任何东西能增补或替换它们。

`disableSideloadFlags` 和 `strictKnownMarketplaces` 意味着工程师机器上的每个 Skill、agent、hook 和 MCP server 都来自组织批准的插件市场，绝不来自某个 home 目录。

`allowManagedMcpServersOnly` 让 agent 的工具面成为平台团队拥有的白名单。

`requiredMinimumVersion` 拒绝在低于批准下限的版本上启动，保证控制手段由组织实际评估过的构建来执行。

把上面当作裁剪的起点，而不是照抄的建议。每个 deny 都在与能力做交易，正确的平衡取决于仓库的数据分级。settings 参考文档记录了每个键，包括 managed-only 的键：[code.claude.com/docs/en/settings](https://code.claude.com/docs/en/settings)

##### 如何衡量（针对 hooks 本身）

**先导指标**

每个审批门禁上的等待时间。每个 hook 决策（带时间戳与放行/阻断裁决）都写入 OpenTelemetry 导出，因此每个门禁的等待都是可见的。

**滞后指标**

hooks 前后越过门禁流入生产的违规数，来自事故系统。

##### CI/CD 集成与部署

在 CI/CD 流水线内非交互地运行 Claude Code，给长时运行的 agent 做沙箱隔离，通过 MCP 集成暴露部署能力，并且在 agent 需要之前就演练回滚路径。

> **传统**：流水线跑确定性脚本，需要判断的事都等人来。比如分诊 flaky 测试、写 changelog、查构建为什么挂了。部署和回滚是人顶着压力照着执行的 runbook。
>
> **AI 原生**：Claude 在流水线内部非交互地处理判断类步骤，运行在带受限凭据的沙箱里。部署工具通过 MCP 暴露给 agent，于是写出并测试了这个变更的工作流也能发布它、回滚它——在组织按环境定义的门禁之内。

##### 起步

**前提条件**

PR 评审闭环中的 Claude，以及作为审批门禁的 hooks——门禁必须先存在，自动化才能加速通过它们。

**基础设施**

装了 claude-code-action 的 CI 平台，或任何能调用 `claude -p` 的 runner；走 API 或 Bedrock、Foundry、Vertex 的模型访问（当流量必须留在组织的云协议内时）；面向部署目标的 MCP servers；一个无长期生产凭据的 agent 作业沙箱配置。

###### 如何执行

1. 平台工程师从只读的判断类步骤起步。在流水线作业中用 `claude -p` 分诊失败的构建、总结 flaky 测试、起草 changelog。
2. 在既有门禁之后加入写步骤：修 lint、更新生成的文档、通过 `@claude` mention 处理评审评论。agent 写的任何东西都通过分支保护以 PR 形式到达，agent 没有直推 main 的通道。
3. 执行沙箱化。agent 作业在按网络策略运行的容器里、用短时效的受限 token 运行，默认不持有任何生产凭据。
4. 通过 MCP 暴露部署。部署、状态、回滚都成为按环境限定作用域的工具，所以 agent 的部署权限是一份白名单，而不是一个带着凭据的 shell 脚本。
5. 按环境分层授予自主权。开发环境，agent 自由部署；生产环境，agent 准备发布、发布经理授权，由一个 hook 强制生产门禁；staging 居中。
6. 回滚应该是流水线里演练最充分的路径——一条 agent 能执行、且在 staging 定期演练的单命令。闭环打法（阶段 6：维护）会在控制带被突破时调用这条回滚，所以它必须提前被验证过。

###### 长什么样（流水线步骤）

```markdown
- name: Triage failed build
  if: failure()
  run: >
    claude -p "Read the build log at out/build.log. Identify the most
    likely cause, say whether the failure looks flaky or real, and write a
    three-line summary for the PR thread." >> triage.md
```

###### 治理考量

统领性原则是：agent 可以行动到生产门禁为止，不能越过它。以下控制手段强制这条原则。

- 分支保护把 agent 写的任何东西变成 PR，没有直达 main 的路径。
- 生产部署 hook 阻断发布，直到指定的发布经理授权。每个非交互运行以 agent 自己的身份行事，所以流水线日志能区分 agent 做了什么、触发它的工程师做了什么。
- 按环境的权限分层决定 agent 在通往门禁的路上能做多少。

##### 如何衡量

**先导指标**

无需 page 人类就完成分诊的流水线故障占比，取自 CI/CD 流水线日志。

**滞后指标**

DORA（DevOps Research and Assessment）指标，CI 系统和部署工具已经在产出。

**06**

#### Maintain（维护）

闭环合拢。一个触发器在没有任何人介入调用路径的情况下唤醒 Claude，它发现的东西以 `intent.md` 重新进入流水线。

##### 维护与闭环

到目前为止，我们讨论的是把 Claude 加入 SDLC 流程的各个阶段，每个阶段都需要人来启动最初几步。而这个阶段的重心，是让 Claude 自主运行、把环闭合。

例如，一个持续运行的监控 agent 可以在一张 bug 工单创建之后，写一份 `intent.md`，并走完需求、计划、构建、测试和评审各阶段。阶段 6（维护）以 headless（无头）方式运行，阶段之间设独立的置信度门禁——一次确定性检查或一个对抗式评审 agent——来决定上一阶段的产出是继续，还是上报人类。

> **传统**：维护是被动阶段。所有工单或事故都在等人来处理、等人重启流程。凌晨三点的告警可能被错过，工单可能躺在 backlog 里直到有人捡起，复盘的行动项如果另一处火情先起，可能根本到不了代码库。
>
> **AI 原生**：一个触发器——控制带突破、工单、频道消息或调度——在调用路径上没有人的情况下唤醒 Claude。Claude 做诊断、只通过带门禁的路径行动，并把发现写成 `intent.md`，随后走上面描述的各个阶段。人对这些工作做分诊和评审，但不再需要人来发起。

##### 闭环

一个确定性脚本盯守生产环境，在控制带被突破时唤醒 Claude。对突破的监控是这个闭环自主运行模式的一个好例子；本阶段末尾的 [Claude Tag](https://claude.com/product/tag)（public beta）小节则覆盖从其他渠道到达的工作。

##### 起步

**前提条件**

`intent.md`——给这个环一个结构化的重启输出。Claude 加速的 PR 评审、作为动作边界的 hooks，以及 CI/CD 的回滚路径（最高自主层级会调用它）。

**基础设施**

检测脚本可查询的指标存储（Prometheus、CI 系统 API 或等价物）、仓库读权限、在 CI 中非交互运行 Claude Code 的方式，或用 [Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview) 搭一个接收 webhook 的服务。

###### 如何执行

1. 服务负责人或平台工程师选一个有稳定滚动基线的指标，比如 CI 测试失败率、部署后 5xx 率，或 PR 周期时间。
2. 写检测脚本：通常是对滚动窗口算均值和标准差，配上（Western Electric 或类似的）判读规则，让控制带既能抓住尖峰也能抓住缓慢漂移。脚本受版本控制、有单元测试，检测完全确定性，不涉及任何模型。
3. 响应层级定义在受版本控制的配置里（下面的 `bands.yaml`）。1σ 只记日志；2σ 以只读方式唤醒 Claude 做诊断；3σ Claude 可以行动——但只能通过向评审门禁开 PR、或触发一个预先批准的 runbook。
4. 触发层可以是 GitHub/GitLab 的定时 workflow、现有监控栈的 webhook，或网络内的 Cron Job。Claude 以无状态方式运行——CI runner 上的非交互步骤，或沙箱容器里的 Agent SDK 服务——CI/CD 打法覆盖了部署与模型访问选项。因为运行无状态且非交互，一个环可以在没有任何人启动的情况下开始和结束。
5. agent 把诊断写成阶段 1（Plan）格式的 `intent.md`：异常及其证据、建议的产出、受影响的系统和开放问题。从这里开始，这个发现像其他任何工作一样走流水线。
6. 服务负责人或值班工程师分诊队列，把面向产品的发现路由给产品负责人。立即修、排期、或驳回。驳回会反过来调校控制带，帮助降噪。
7. 修复上线时，为这次事故补一个 eval（持续 evals 打法），确保此类问题今后受保护。

###### 长什么样（以监控 CI 测试失败率的 bands.yaml 为例）

```yaml
metric: ci_test_failure_rate
baseline: rolling_30d
rules: western_electric
tiers:
  1sigma: { action: log }
  2sigma: { action: diagnose,
            tools: "Read,Grep,Bash(gh run view *)" }
  3sigma: { action: propose,
            routes: [pull_request, runbook:rollback-deploy] }
```

###### 治理考量

层级边界由受版本控制的配置强制执行，权限与 managed settings 拒绝生产访问。调用、发现与分诊决策都带时间戳记录。服务负责人分诊并批准发现，产生的变更走正常 PR 评审门禁，agent 可触发的 runbook 都是提前批准过的。

##### 如何衡量

**先导指标**

从控制带突破到 `intent.md` 出现在分诊队列的时间，对比旧的从事故到复盘行动的时间。检测脚本日志里有突破时间戳和事故层级。

**滞后指标**

最终成为已合并修复的发现占比（分诊队列对比实际 PR 历史），以及同类事故的重复率——修复不断往 eval 套件加案例，这个数应该下降。

###### 示例

- CI 测试失败率突破 3σ 时，agent 隔离 flaky 测试或开一张 revert PR，由评审门禁裁决。
- 部署后 5xx 率在窗口内有部署的情况下突破 3σ 时，agent 触发现有的回滚流水线。
- PR 周期时间触发漂移规则时，agent 给工程管理层写一份报告——这说明这套 harness 对流程指标和生产指标同样有效。

检测保持确定性。控制带被突破后 Claude 才被唤醒，层级决定它可以做什么。

##### Claude 值班：Claude Tag

事故也可以从其他渠道到达，比如职场通讯应用（Slack 或 Teams）。一次事故可能是晚上十点事故频道里一条要求紧急修复的消息，而现在它可以被立即处理。Claude Tag（目前在 Slack 的 public beta）让 Claude 以自己的身份成为这些频道的成员，于是每起新事故都有第一响应人，响应本身成为闭环的一部分、也成为未来事故的记忆。

对话与组织知识留在频道里，频道中的任何人都可以引导和执行响应。任何团队成员都可以实时验证假设、探索新方案、做调查，频道历史则增强了可审计性。通过 MCP 的访问能力，Claude 验证指标回到基线并在话题里确认，把复盘写进一个受版本控制、供未来调查阅读的 lessons 文件。

Claude Tag 接手的不只是事故。通过 MCP 被标到工单上、或在频道里被问到时，Claude 用同样的方式分诊工作：一个小的、边界清晰的修复以 PR 形式经评审门禁到达；更大的事项写成 `intent.md` 进入阶段 1（Plan）——从这一刻起，这个环开始自我供给。

频道就是审计轨迹：请求、诊断、人类授权与修复，全部留在事故被处理的地方。

#### 结语

模型和 harness 变得更强大，让组织能够变革的不只是产码方式，而是整个软件开发生命周期。

这场变革让人类判断保持在流程的中心，同时兼顾大型企业组织的治理与合规要求。

本指南汇集了我们 Applied AI 团队每天为客户执行的许多真实最佳实践，希望它对你是一份实用、可上手的参考。

这个环持续运转。人类的判断始终在它之上。

##### 资源与致谢

以下文档是一个平台团队搭起这些控制手段所需的材料，大致按上线的先后顺序排列。

感谢 Jim Blackhurst、Will Steuk 和 Jamal Arif 对本指南的贡献——它正是受他们此前大量工作的启发、并建立在那些工作之上。

---

> 原文：[The AI-Native SDLC playbook | Claude by Anthropic](https://claude.com/blog/the-ai-native-sdlc-playbook)
