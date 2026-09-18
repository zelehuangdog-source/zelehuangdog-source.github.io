---
title: "Claude Tag 如何充当 Anthropic CI/CD 故障的第一响应者"
date: 2026-08-19
author: repost
categories: [转载, Claude-Code实战]
tags: [翻译, Claude-Tag, OnCall, CI/CD, 转载]
---

> **摘要**：Anthropic 工程师分享了他们如何用 Claude Tag（Claude in Slack）为 CI/CD 故障构建一名 AI On-Call 第一响应者：Claude 全天候监听告警、按规则决定是否拉人，事故发生后中位 14 分钟内发布首份基于证据的分析报告，并能执行回滚 feature flag、提交修复 PR、编写事后复盘等操作。整套体系依赖四个要素——记忆、连接与权限、日程、指令（以 GitHub 仓库中的 markdown skill 形式维护，包含持续积累的 lessons.md 自我改进循环）。文中还附上了一套开源的 oncall-kit，可以把团队自己的事故历史转化为分诊手册，几小时即可复制这套实践。

> **转载声明**：本文**翻译整理**自：《Claude Tag 如何充当 Anthropic CI/CD 故障的第一响应者》。**内容版权归原作者及原出处所有**，本文仅供学习交流，**所有观点均属原作者，不代表本站立场**；如有侵权，请联系删除。


---

*[用我们的部署工具包搭建你自己的 Claude On-Call](https://github.com/anthropics/oncall-kit)*。*

几周前我正在 On-Call，晚上 10 点同事在 Slack 上给我发消息：一个新服务上大约有 44 个测试没有跑起来。

放在过去，我就得停下手头的事，坐到笔记本前，疲惫地叹口气，然后开始长达一小时排查加修复。但现在我的工作流完全变了：我把 @Claude 拉进来，问它看到了什么。

这一次，Claude 发现这些测试是当天早上某个 feature flag（功能开关）打开后消失的，并且判断回滚是安全的。我让同事回滚了开关。3 分钟后 Claude 在 Slack 上提醒我，确认跳过规则确实已被移除、错误率回到了基线。

过去几个月，Claude Tag 一直是 Anthropic CI/CD 故障的 On-Call 第一响应者。这不仅拯救了我们的个人生活，还让每次 CI 事故都有了一名即时响应者：在近期所有产生了事故报告的事件中，第一份态势报告都出自 Claude 之手，**通常在 15 分钟内发布首份分析**。

本文将带你看懂我们构建了什么、它是如何运转的，这样你也能自己搭一套，从此不再害怕轮值 On-Call。

#### 我们的 Claude On-Call 架构

在展开事故响应流程的每个阶段之前，我先给出整体架构的概览，让你在细节展开前先建立全局图景。

一个 On-Call agent 需要**记忆（memory）**，才能记住已经做过什么；需要**连接与权限（connections and access）**，才能调查、理解和行动；需要**日程（schedules）**，才能知道何时该回到工作岗位；还需要**指令（instructions）**，才能知道该做什么。

[Claude Tag](https://claude.com/product/tag) 是我们 On-Call agent 的骨干。Claude Tag 在 On-Call Slack 频道中保持跨会话记忆，也是在事故期间逐轮下发指令的界面。Claude 还能对 On-Call 频道和其他频道中的事件实时响应。例行事务的排程——也就是 Claude 定期执行的动作——同样发生在这个频道里，用自然语言提示即可，比如"每周一 EST 早上 9 点跑一次 CI 交接"。

[Claude Tag 拥有自己的服务账号](https://claude.com/blog/agent-identity-access-model)，并且能访问 Anthropic CI 工程师所需的工具，比如 Datadog 和 Grafana。这是由频道管理员一次性配置好的（[配置方法在这里](https://claude.com/docs/claude-tag/admins/setup-overview#choose-which-tools-to-connect)）。

除了 On-Call 频道，我们还让 Claude 监听其他几个同样有 Claude Tag 成员身份的相关频道，这样它能获取更多上下文，比如服务告警、配置变更、PR 的进展等。

固定指令以 skill 形式放在 markdown 文件里，提交在 GitHub 仓库中。这样多个队友可以一起迭代它们，我们也能像管理代码一样管理这些变更。仓库里还包含路由指令、策略，以及一条自我改进循环的一部分——经验教训日志（lessons learned log）。

这套架构的搭建只花了几个小时，而不是几天。我们在 GitHub 上开源了一套通用的 [on-call 部署工具包](https://github.com/anthropics/oncall-kit)，可以帮你快速起步搭建类似的 agent。它能把你们团队自己的事故历史转化为分诊手册，最终在你的事故频道里留下一个只读的 Claude，负责诊断、上报和持续学习。[你可以看着它对一个虚构团队的历史记录跑完全流程](https://github.com/anthropics/oncall-kit/blob/main/test-fixtures/RUNBOOK.md)，大约十分钟。

用 TL;DR 的方式总结步骤：

- 你需要 [Claude Team 或 Claude Enterprise](https://support.claude.com/en/collections/9387370-team-and-enterprise-plans) 订阅
- 组织所有者需要通过 Claude Tag 把 Claude 加入 On-Call Slack 频道
- 组织所有者还需要把 On-Call Slack 频道中的 Claude 连接到相应的 connector、GitHub 仓库，并配置好 [Claude Code Remote](https://code.claude.com/docs/en/remote-control)
- 把 Claude 加入你的事故频道，指示它监控事故并立即分诊

下面我们深入看看事故的每个阶段上，这场变革分别是什么样子。

#### 检测

Claude 不只改变你响应事故的方式，它首先改变的是你发现事故的方式。以前，事故检测有两大失效模式。

第一种：人类很难始终有先见之明，设定出阈值恰到好处的完美规则。当没有足够数据来分析流量模式时，这尤其困难。

为此，我们让 Claude 在新服务上线的头几天分析数据和涌入的告警，建议补充新规则，并微调那些过宽或过窄的规则。

事故检测的第二大失效模式是告警疲劳：逐条检查和甄别每个触发的告警非常枯燥。但 Claude 不会像人类那样疲劳。

Claude 监听每个告警频道里的所有相关告警，并对照[根 oncall.md 文件](https://github.com/anthropics/oncall-kit/blob/main/templates/ONCALL.md)中的标准来判断：这条告警是可以等到早上再处理，还是必须立即拉起 On-Call。举例来说，经过数据调优后，文件里的一条规则可以是："如果错误率超过 2% 且持续 5 分钟以上，并且当前不是已知的发布窗口，则拉起 On-Call；否则写入 lessons.md。"

Claude On-Call 告警流程还有另外两种触发方式：

- CI 团队成员可以在 On-Call 频道里报告问题——就像开头那个 44 个测试消失的例子；或者
- 公司任何人都可通过内部页面发起事故。如果被标记为 CI 基础设施事故，系统会为该事故创建一个 Slack 频道，我们的 On-Call Claude 会自动接手。

这里的关键结论是：告警流程是确定性（deterministic）的，而 On-Call 上报则兼有确定性和 agentic 两条路径。

#### 分诊

让 Claude 过滤告警噪音是一回事，真正的节省来自调查环节。Claude 在事故发起后中位 14 分钟就发布第一份有证据支撑的分析，最快的案例中，它在首份报告里 4 分钟内就点明了根因。

当告警被上报为事故时，Claude 往往已经带着一个有证据支撑的假设守在 Slack 频道里，供我们审阅。Claude Tag 会启动一个[动态工作流（dynamic workflow）](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)：一个编排 agent（orchestration agent）拉起多个执行者子 agent（executor subagent），分头调查每个依赖项和数据源。

对我们的场景来说，这些数据源是 Grafana、日志存储、PagerDuty、GitHub、Kubernetes 和 Slack 事故频道——全部通过 [MCP Connector](https://code.claude.com/docs/en/mcp) 接入。Claude 可以并行追踪多条线索，帮助缩短 MTTR（mean time to resolution，平均解决时长）。

执行者把发现回报给编排 agent，由它综合提炼，以一份条理清晰的 SITREP（态势报告）呈现出来。

编排者和执行者 agent 并不是盲目搜索。它们由一个调查 skill 引导，其中包含[针对每类 bug 的更详细参考 markdown 文件](https://github.com/anthropics/oncall-kit/tree/main/skills/triage)。

举个例子，有一个 617 行、针对 shadow divergence（影子环境不一致）类 bug 的调查 skill，把我一次典型排查中的每一步都沉淀了进去。我是这样构建它的：在某次事故中与 Claude 逐轮协作排查，然后让它把这段经验整理成文件。

lessons.md 同样指导着 Claude 的排查。这个 markdown 文件是我们已解决的每一起事故的滚动日志：发生了什么、根因是什么、怎么修的、有什么值得记住的坑。Claude 会自动向其中追加条目。每次新调查都从读它开始，所以 Claude 的第一个假设总是从"最近发生过什么"出发。

如果同一模式出现足够多次，我们就把它晋升进调查 skill 本身。我最喜欢的一条是 Claude 写关于我的：我在看指标之前就凭一个配置文件下了假设，于是 lessons.md 里现在写着："先查数据，再做理论。配置告诉你什么可能出错；指标告诉你什么确实出错了。"

即便有这些工具和上下文，Claude 也不总是第一次就答对。人类的直觉和经验依然重要。Claude Tag 让团队可以以多人模式协同排查事故——我们任何人都可以实时介入调整调查方向，或补充一个假设，并肩作战。

#### 解决

如果 Claude 能上报和排查告警，它也能修好问题吗？这个问题的答案因团队而异，下面说说我们的做法。

我们团队的大多数部署都在 feature flag 后面进行。我在 Claude Code 里创建了另一个 agent，使用我的权限，能够在各个 feature flag 后面执行渐进式发布。

我们发布流程的第一阶段通常是 Claude 管理金丝雀流量、监控问题，并自动上调或下调某个 feature flag。这一块完全可以单独写一篇文章，这里不再展开。

Claude Tag 帮我团队走通的其他解决路径还有：

- 告诉我们是否需要排空（drain）或隔离（cordon）Kubernetes 集群的某些部分；
- 在需求激增时给出扩容部分基础设施的操作指引（这种情况很少见，但 Claude 拿着确切的缓解方案回来时，帮助非常大）；以及最频繁的——
- 以 PR 形式给出修复，由 On-Call 的人审查、合并、部署，快速收尾。

#### 验证、沟通与交接

Claude 用调查阶段同一批 MCP Connector 和工具来验证修复是否生效。按照 oncall.md 中的固定指令，它会向 lessons.md 写一份事后复盘，并生成交接用的 SITREP。

为了跨多个事故沟通全局态势，我们创建了一个叫 ci-weather 的 agent。它汇总每个事故 Slack 频道的信息、构建指标、合并队列统计和发布延迟，然后以新闻播报的风格把报告发到一个公司内任何人都能看的公开频道。现在，工程师们想知道该不该暂缓合并、想知道"CI 到底怎么了"时，直接看那个频道就行，不用再来打扰我们。

说句实话：这个报告的格式我们迭代了好几版。Claude 可以一次性写出生成状态报告的 skill，但真正让它可读的，是团队特有的品味。这是人与人沟通的功课，不是管道工程。

最后，虽然 Claude 在 lessons.md 里为自己记着流水账，我们也希望为人类产出交接报告——每周一。Claude 会生成日报和周报，这样团队里的一位成员可以从另一位停下的地方无缝接手。

#### 从监控事故，到监控一套事故响应系统

我们的软件工程师现在平均每季度[发布的代码量是 2021 至 2025 年间的 8 倍](https://www.anthropic.com/institute/recursive-self-improvement)。虽然我们始终保持着很高的质量水位（每个 PR 都有具名的人类负责人、每次变更都必须审批后才能合并、每次变更都经过同一套 CI 门禁），但要跟上 agentic coding 的步伐，唯一的办法就是 agentic CI。

Claude 吸收了我工作中枯燥的部分——深夜被打断、事故通报——让我得以专注于那些真正能提升系统可靠性的中长期架构改进。

我们构建的这套东西最棒的地方在于：它不散乱。我们的 On-Call 流程本来就活在 Slack 里，而现在，Claude 加入了这个频道。

如何起步：

- 你需要 [Claude Team 或 Claude Enterprise](https://support.claude.com/en/collections/9387370-team-and-enterprise-plans) 订阅
- 组织所有者需要通过 Claude Tag 把 Claude 加入 On-Call Slack 频道
- 组织所有者还需要把 On-Call Slack 频道中的 Claude 连接到相应的 connector、GitHub 仓库，并配置好 [Claude Code Remote](https://code.claude.com/docs/en/remote-control)
- 把 Claude 加入你的事故频道，指示它监控事故并立即分诊

*[用我们的部署工具包搭建你自己的 Claude On-Call](https://github.com/anthropics/oncall-kit)*。*

*本文作者为 Sachin Malhotra（Anthropic 技术团队成员），Michael Segner（Anthropic 员工）亦有贡献。*
