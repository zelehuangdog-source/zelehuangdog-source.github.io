---
title: "我所知道的所有 Agentic Engineering 技巧（2026 年 6 月）"
date: 2026-06-03
author: repost
categories: [转载, AI工程方法论]
tags: [翻译, AI工程方法论, Claude-Code, Agentic-Engineering, 转载]
---

> **摘要**：作者作为 Compound Engineering、last30days、Printing Press 等热门开源项目的核心贡献者，分享了他在 2026 年 6 月这个时间点所沉淀出的全套 Agentic Engineering 工作流。核心思路是：**不要用 IDE，不要打字写代码，而是通过语音 + plan.md + 多会话并行**的方式来推进所有工作——无论是写代码、做研究、写文章还是处理生活琐事。文章给出了从「想到点子的那一刻就用 /ce-plan 起一个 plan.md」到「让终端默认打开就是 Claude Code」「给 Claude 配一个邮箱地址」「YOLO 模式跳过所有权限确认」「用 Codex + Claude Code 双 200 美元订阅并行干活」等 22 条具体可落地的技巧，并坦诚指出了 AI 上瘾这一隐性风险。

> **转载声明**：本文**翻译整理**自：《Every Agentic Engineering Hack I Know (June 2026)》，作者 Matt Van Horn (@mvanhorn)，原文 https://x.com/mvanhorn。**内容版权归原作者及原出处所有**，本文仅供学习交流，**所有观点均属原作者，不代表本站立场**；如有侵权，请联系删除。


---

三个月前我发了《我所知道的所有 Claude Code 技巧》，阅读量 91.3 万。当时 @kevinrose 问我用什么 IDE，我的回答是：「不用 IDE。只用 plan.md 文件加语音。」

这种方式以前叫 vibe coding。大约去年感恩节前后，模型变得足够好，玩具变成了真家伙——也就是现在大家说的 Agentic Engineering。这是我能持续交付东西的唯一原因。今年我做了 last30days（27K stars）、Printing Press（4K+ stars）、刚发布的 Agent Cookie，并成为了 Python、Go、GStack、Paperclip 等开源大项目的头部贡献者。从高中以来，我就再没做出过别人觉得有价值的软件了。下面是我的所有技巧。

#### HACK

**YOLO TL;DR 技巧**：把整篇文章贴给你的 agent，让它做一个把里面的东西全部装上的计划，然后一条一条干。这就是我的整套技术栈，连读都不用读。

---

#### 1. 想到点子的那一刻，就建一个 CE plan.md

这仍然是第 1 条规则，仍然是我学到的最重要的事。

想到点子的那一刻，立刻 `/ce-plan` 写一个 plan.md。不是「让我想想」，不是「让我先动手写」，每次都是 `/ce-plan`。它也支持图片，所以任何能截下来的东西都可以是起点：

- 疯狂的产品点子：`/ce-plan`
- GitHub 上的 bug：复制 issue URL，粘贴，`/ce-plan`
- 终端报错：`Cmd+Shift+4` 截图，`Ctrl+V` 粘贴，`/ce-plan fix this`
- 截图、报错信息、设计稿、Slack 对话——丢进去就行

如果点子还很模糊，自己都不知道要什么，先用 `/ce-brainstorm` 跟 agent 一起把它想清楚，等想清楚了再 `/ce-plan`。

底层上，`/ce-plan` 会并行 fan out 一堆研究 agent。一个读你的代码库找模式、查你的代码规范；一个搜你之前的解法找经验；如果话题需要，更多 agent 会去查外部文档和最佳实践——全都同时进行。然后它把结果整合，写成一个结构化的 plan.md：问题是什么、方案怎么做、要改哪些文件、带 checkbox 的验收标准、要遵循的你自己代码里的模式。基于你的仓库、你的规范、你的历史，不是泛泛而谈的建议。

`/ce-work` 拿着这个 plan 去执行。上下文炸了？开个新 session，指给它这个 plan，从断点继续。这个 plan 是能扛过一切的检查点。

传统开发是 80% 写代码、20% 规划。这个流程把它倒过来了。思考全部塞进 plan 里，执行只是机械操作。

来自 @kieranklaassen 和 @trevin 的 Compound Engineering 插件让这一切真的能跑起来。我从超级粉丝变成贡献者，现在已经是仅次于核心团队的第三大贡献者。我现在的规则是：除非真的是一行的改动，否则永远先写 plan.md。

##### HACK

- 装 Compound Engineering：`/plugin marketplace add EveryInc/compound-engineering-plugin`
- 粘贴截图、bug URL 或报错，然后 `/ce-plan`，再 `/ce-work`
- 想法模糊？先 `/ce-brainstorm`

---

#### 2. 别读 plan.md

我每次都做 plan.md。但我几乎从来不读它。**Plan 是给 agent 用的，蠢蛋人类。**

强制 plan 存在能让 agent 不偷懒。它逼 agent 去研究、定方案、写下验收标准，然后真的去逐条满足。带 plan 的编码 agent 会做完整工作；不带 plan 的 agent 会偷工减料、提前停下来。Plan 就是它的牵引绳。

所以我让它写 plan，扫一眼标题，然后跑 `/ce-work`。如果有问题就在 session 里直接问：「等等，为什么用这个方案？」或者要个 TLDR；或者看不懂的时候说「eli5 这个 plan」。我得到一段话的版本，点点头，继续。我不会坐在那儿啃 300 行 markdown，那是 agent 的作业，不是我的。

写 plan，信任 plan，别读 plan。

##### HACK

- 不要让自己去读 plan。在 session 里直接问：`TLDR?`、`eli5 this plan`、或者「等等，为什么用这个方案？」

---

#### 3. 用 /ce-plan 处理你最深度的非工程工作——「为 Plan 做 Plan」

大家以为 `/ce-plan` 和 `/ce-work` 是用来写代码的。3 月以来我学到的最大的一件事是：它们不是。我现在最深度的知识工作都跑在同一个循环里，关键技巧是：**让第一个 plan 是「为 plan 做的 plan」**。这也不是我硬把代码工具拗成别的用途——`/ce-plan` 内置了通用规划模式，就是为这种非代码工作设计的。

不只是商业问题。战略文档、产品规格、竞品分析、董事会更新——全都是同一个循环。

举个真实例子。我和前 GV 研究合伙人 Michael Margolis 见了面，聊我手头的一个商业问题。他以「靶心客户法」闻名，让我读他的书——他网站上有免费 PDF。老办法是翻一翻就过去了。我打开 Claude Code 大致说了这么一段：

> 「`/ce-plan` 给我做一个为 plan 做的 plan。我马上要给你两样东西：Margolis 的书 PDF，以及我刚跟他开的两小时会议的完整 Granola 转录稿，里面有我们讨论的全部上下文。我想要一个深思熟虑的计划，告诉我怎么把我的商业问题、那段对话、书里的经验融合成一个我能真用上的东西。**现在不要写那个文档**。写文档本身是工作。我现在只要一个计划——你打算怎么读这本书、怎么挖这份转录稿、怎么产出一个高质量的文档。」

它接下来花了 45 分钟做出了一个**史诗级**的 plan。

这也是我知道的最好用的「让 LLM 不偷懒」的技巧。直接要交付物，它会偷工减料；先让它规划「我将如何产出这个交付物」、再让它去执行——它每次都会做深度版本。

##### HACK

- 深度非代码工作：`/ce-plan` 让它「为 plan 做 plan」，把你所有的上下文和转录稿丢给它，然后 `/ce-work`

---

#### 4. 接受语音输入

语音转 LLM 跟语音转其他任何东西都不一样。转录不需要完美，因为听者理解上下文，它会猜麦克风错听了什么。你可以含糊、可以话说一半、可以重起一句。语音终于能用了，是因为另一头的东西够聪明，能补上空缺。

我的配置：

- **Mac**：Monologue（来自 Every）或 Wispr Flow，挑一个，把语音灌进当前焦点的应用，对着 Claude Code 说话。我给办公室买了一支鹅颈麦
- **手机**：跳过 Monologue 和 Wispr Flow，iOS 上切换太烦。苹果自带的听写就够用了，因为你是在跟 LLM 说话，不是跟人。一半词识别错了 agent 也能听懂。潦草的笔记没问题

一个老实的承认：我一个人的时候用语音很顺。在办公室就不行。有人说你可以小声对着麦说，但我发现自己实际上不会这么做，因为不想没礼貌打扰旁边的人。所以共享办公室里的工位是我整套工作流唯一的弱点。如果你在开放办公室破解了语音输入又没变成那个让人讨厌的人，告诉我怎么做。我是真的想求建议。

##### HACK

- Mac：装 Monologue 或 Wispr Flow。手机：用苹果听写。买一支鹅颈麦

---

#### 5. 在 cmux 里开一堆又一堆 tab

我一天的实际状态是这样：4 到 6 个 cmux tab，有时更多，每个一个独立 session：

- 一个在写 plan
- 一个在按另一个 plan 构建
- 一个在跑 last30days
- 一个在修我刚才测出来的 bug

`/ce-plan` 在一个窗口跑研究的同时，我切到另一个窗口 `/ce-work` 一个已经写好的 plan；那个在构建的同时，第三个窗口我粘了个新 bug 进去。等我转回第一个时，它已经做完在等我了。

我听说 Orca 在移动端做得很好。我以前是 Ghostty 纯粹主义者，但在 Ghostty 里丢了太多通知。

##### HACK

- 用 cmux
- 保持 4–6 个 tab 开着，每个 tab 一个不同任务

---

#### 6. 让终端默认进 Claude 或 Codex，而不是 shell

新 tab 应该直接打开就是 Claude Code，不是 shell。开一个 tab，你已经在跟 agent 对话了。不用 `cd`，不用打 `claude`。当一个新 session 只要一个按键时，你就会开很多很多个。我也不用文件夹，agent 自己能找到你的项目。

##### HACK

把这段贴给 agent：

> 「让每个新终端 tab 直接打开 Claude Code。在 `~/.config/ghostty/config` 里加一行 `command = ~/.local/bin/claude-launcher.sh`，不要动文件里其他设置。然后创建 `~/.local/bin/claude-launcher.sh`，运行 `claude --dangerously-skip-permissions`，Claude 退出时打印一段简短提示并把我落到一个交互式 login zsh。给脚本 `chmod +x`。Ghostty 和 cmux 都吃这个，因为 cmux 读的是同一份 Ghostty 配置。」

---

#### 7. 给每个窗口开远程控制，并给 Claude Code 或 Codex 一个邮箱地址

两个让每个 session 都能在任何地方触达的技巧。

##### 每开一个新窗口都自动打开远程控制

把远程控制设成每个 session 自动开。

现在每个窗口都可以在 Claude 移动端 app 里访问。在桌前开一个 session，走开，在手机上接着任务跑到一半的同一个 live run。在排队等位的时候，你正遥控着家里那台 Mac 上还在跑的东西。

##### 给你的 Claude 一个邮箱地址

Claude Code 可以通过 AgentMail 拥有邮箱地址。AgentMail 创始人 Adi @adisingh 教我的。给那个收件箱发邮件，会开一个新 session，按主题和正文里的内容开始干活，附件也能按路径访问。晚饭时碰到 bug？从手机发个邮件，回到屏幕前 session 已经在跑了。我把整套东西开源了：[github.com/mvanhorn/agentmail-to-claude-code](https://github.com/mvanhorn/agentmail-to-claude-code)。

三个组件：

1. 一个守护进程通过 WebSocket 监视 AgentMail 收件箱。每收到一封白名单内的邮件，就开一个新 Claude session，把邮件写进一个 prompt 文件，然后让 Claude 读它并去做
2. 两个终端后端：cmux 或独立 Ghostty，对接你已经在用的那个
3. 一个发送器。我把它接到了 Hermes 里的 `cc` 命令，所以从手机执行 `cc <task>`，它就以一个工作中的 session 形式落地到我的 Mac，不用 VPN 不用 SSH

白名单是闸门。只有你自己控制的地址能进来，DKIM 或 SPF 失败的邮件在 session 打开之前就会被丢弃。

##### HACK

- 始终开远程控制：在 `~/.claude/settings.json` 里加 `"remoteControlAtStartup": true`
- 给 Claude 一个邮箱。把这段贴给 agent：

> 「用 `github.com/mvanhorn/agentmail-to-claude-code` 给 Claude Code 配一个邮箱地址。clone 它，建一个 AgentMail 收件箱，把 `cc.env` 填上我的 API key、收件箱、只允许我自己地址的白名单、以及我的终端（cmux 或 Ghostty），然后跑守护进程并装成 launchd 任务。当我给那个收件箱发邮件时，这台 Mac 上应该开一个新 Claude Code session 并按主题和正文开始干活。」

---

#### 8. 危险地跳过权限确认——是的，我说的就是字面意思

Claude Code 每个编辑和命令都问一遍权限。开 6 个 session 时你不可能一直盯着。两个设置让它能用。有人说 auto 是「更安全」的方式，但对我太慢了。

`skipDangerousModePermissionPrompt: true` 是关键。没有它，Claude 每个 session 都要你确认一次。也可以 `Shift+Tab` 切。有人告诉我新的 "auto" 模式能在更安全的前提下达到差不多效果。也许吧。我说 YOLO。这是我自己的电脑，搞砸了还有 GitHub。我帮一个朋友配 Claude Code 时，AI 主动劝他别开这个，你得对它强硬。

另一个设置是声音 hook，开 6 个 session 时不可妥协。

走开，听到声音回来。6 个 session 在跑时，声音是你判断哪个刚做完的方式。

##### HACK

把这段贴进 `~/.claude/settings.json`：

```json
{
  "permissions": {
    "allow": [
      "WebSearch", "WebFetch", "Bash", "Read", "Write",
      "Edit", "Glob", "Grep", "Task", "TodoWrite"
    ],
    "deny": [],
    "defaultMode": "bypassPermissions"
  },
  "skipDangerousModePermissionPrompt": true
}
```

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "afplay /System/Library/Sounds/Blow.aiff"
          }
        ]
      }
    ]
  }
}
```

Codex 有同款 YOLO 模式。在 `~/.codex/config.toml`：

```toml
approval_policy = "never"
sandbox_mode = "danger-full-access"
```

或者一次性 `codex --yolo` 启动。

---

#### 9. 我如何让大部分代码跑在 Codex 上，却从不打开 Codex CLI

我整天往 Codex 派活，但几乎从不打开 Codex CLI 来做这件事。**Claude 做规划，Codex 做构建**，我从不离开 Claude session。

三种从 Claude 内派活给 Codex 的方式：

1. **Codex IDE 插件**：发任务，应用结果，从不掉到 Codex 终端
2. **`/ce-work --codex`**：从 Compound Engineering 循环里直接把构建工作派给 Codex
3. **Printing Press Codex 模式**：在 Printing 一个新 CLI 时，prompt 末尾加 `codex`，构建工作就交给 Codex

我的设置——两边引擎都拉到 extra-high reasoning：

- **Codex**：reasoning xhigh，fast mode 一直开
- **Claude Code**：reasoning xhigh，fast mode 关。它的 fast mode 是在 200 美元 Max 套餐之外按 token 收费，我跳过

两个 200 美元套餐并排，相当于第二台引擎。我把大型并行构建推给 Codex，Claude 负责规划和品味。也有朋友反过来用——Codex 构建，Claude 审查。

##### HACK

- Codex：reasoning xhigh，fast mode 开。Claude Code：xhigh，fast mode 关
- 派活给 Codex：Codex IDE 插件、`/ce-work --codex`、或 Printing Press prompt 末尾加 `codex`

---

#### 10. 规划之前先研究：last30days

我 `/ce-plan` 之前通常先 `/last30days` 跑一下这个主题。

之前我在 Vercel 的 agent-browser 和 Playwright 之间选。我没看文档，跑了 `/last30days Vercel agent browser vs Playwright`。几分钟内：几十条 Reddit 帖子、X 推文、YouTube 视频、HN 故事。结论是 agent-browser 每次调用上下文用得少得多，Playwright 光工具定义就甩出几千 token。我把整段输出喂给 `/ce-plan integrate agent-browser`。最后的 plan 扎根于社区当下真实知道的东西，而不是 6 个月前的训练数据。

last30days 是开源的，现在 26K+ stars。它并行搜索 Reddit、X、YouTube、TikTok、Instagram、HN、Polymarket、GitHub、以及网页。我选库前跑、做功能前跑、见生意伙伴前跑、写文章前跑。本文里有几样东西就是我跑过的。研究、规划、构建——这才是真正的循环。

##### HACK

- 装 last30days。`/ce-plan` 之前先跑 `/last30days <topic>`
- 别忘了配一个 ScrapeCreators key

---

#### 11. Granola 一切，并把**原始**转录稿丢进 LLM

我跟一个候选人吃午饭。我们聊产品、聊吃的、聊孩子，90 分钟普通对话里穿插着一个产品想法。Granola 在录。结束后我把完整原始转录稿粘进 Claude Code：`/ce-plan` 把这个变成一个产品提案。

**关键是「原始」**。我不先总结。我把整份乱糟糟的转录稿丢进去，包括聊寿司的题外话，让 Claude 拿着我真实的代码库和我之前写过的所有战略 plan 去抽取。Granola 上下文 + 代码库 + 历史 plan = 黄金。它一发就出了一个提案，自动忽略了餐厅闲聊，我当晚就发出去了。那个人现在全职在我们这工作。

3 月以来的升级：Printing Press Granola CLI。简直魔法。我可以把任何会议作为干净的结构化数据直接拉进 session、跨我所有的会议搜索、找到三周前某人说过的某句话，然后管道进一个 plan。再没有复制粘贴了。每场会议的上下文都是一条命令的距离。

##### HACK

- 把 Granola 原始转录稿丢进 `/ce-plan`，**不要先总结**。装 Printing Press Granola CLI

---

#### 12. 你是「人类信号」

这是我花了最久才领悟的心态转变。当你跑 6 个 agent 时，**你的工作不是干活，你的工作是当信号**。

Agent 提供量。你提供品味、方向、以及「反应—重定向」循环。你看回来的东西，说「方案二更接近，但用方案一的措辞」「先把最大的风险解决掉」「这段太长了」，它们就动了。这个循环里稀缺且有价值的东西是你的判断力，不是你的打字。我越是接受自己作为人类信号、越停止试图同时也当一只干活的手，我交付得就越多。

**你做品味，让它们做手。**

##### HACK

- 用你的脑子去指挥 agent 来给世界创造价值。你的脑子还有用

---

#### 13. HyperFrames：用它做视频，做一切

视频以前是我会外包或干脆放弃的东西。现在我做视频跟做其他东西完全一样：我说话，agent 构建，我给反馈。

HyperFrames 让我把视频当 HTML 来构建，所以 agent 可以写它。循环跟代码完全一样，只是产出是 MP4 而不是 PR。每个视频是一个文件夹，里面有个 `script.md`，逐场景写、动态字幕、每个节奏点都有 caption。Agent 把脚本变成合成画面并渲染。没有编辑器，没有时间轴。

我用这个方式做的发布预告片：

- Granola CLI demo
- Agent Cookie 发布

视频的成本掉到了一段对话，所以任何值得做视频的东西现在都会有视频：发布预告、产品 demo、动画解说、带字幕的短片。也不只发 X，我会把渲染好的 demo 直接丢进 PR——比如这条丢进 atlas-lean（Facebook 的 AI 研究项目）的。

##### HACK

- 用 HyperFrames 构建视频：写一个 `script.md`，让 agent 渲染成 MP4
- GIF 上传到 catbox，在 GitHub、PR、README、issue 里都能漂亮地显示

---

#### 14. 你的笔记就是你 agent 的知识库

3 月那个「战略文件夹」技巧泛化了。每次 plan 越来越好的原因，是 Claude 能访问我之前写过的每一个 plan。**复利上下文**。所以我把它指向我整个大脑。

我让它接的工具：

- **Bear**，配 Bear CLI。十年的笔记、会议、半成品想法、决策——agent 都能读能写。私人 RAG，只是不叫这个名。我放进去的越多，每个 session 就越聪明
- **Obsidian**。我不用，但很多人用，插件生态很深
- **gbrain**。我跨机器跨 agent 同步的大脑
- **supermemory**。一个给 agent 用的记忆层，很多人推荐。我在试，结论待定

技巧的形状才是重点：挑一个有 CLI 或 API 的笔记工具，把 agent 指过去，让你自己的知识做复利。

##### HACK

- 把 agent 指向两个东西：你写的笔记应用（Bear、Obsidian）和帮你记忆的 agent 大脑（gbrain、supermemory）。挑有 CLI 或 API 的，agent 才能读

---

#### 15. 在任何地方工作——我的 Mac mini

##### HACK

- **Mosh**，必须 SSH 时用它。在烂 wifi 和漫游下保持 session 像本地一样响应。普通 SSH 下 Claude Code 慢得像爬，每个按键都在等往返。在远程机器上，这就是「能用」和「痛苦」的区别
- **Tmux**，飞机上用。SSH 进远程机器在 tmux session 里干活，工作跑在那台机器上而不是你的笔记本。跨大西洋 wifi 断 20 分钟，重连、attach，回到你刚才的位置。我整趟从欧洲飞回家的航程都在交付功能
- **Hermes 和 OpenClaw**，都开着，做远程自治工作。Hermes 给你一个会重复任务越做越好的自学习生态；OpenClaw 给你 agent 写出来的技能广度。我两个换着用。如果你早期试 OpenClaw 后弃了，整个清掉重来
- **Agent Cookie** 让你的 Mac mini 和主 Mac 之间的 cookie 和 `.env` 保持同步

---

#### 16. Proof：把 plan 发给同事

plan.md 对我完美，但递给一个不住在终端里的人就废了。这是最后一个真正的缺口，Proof（也来自 Every）把它补上了。

在 Proof 里打开一个 plan、像看文档一样读它——挺好。但它真正变得不可或缺的地方是「把 plan 发给同事」。我把一个 plan.md 或规格丢进 Proof，发链接，一个不在终端里的人能干净地读、行内评论，那些评论会回流到我跟 agent 的循环里。再不用把 markdown 粘进 Slack 然后看它渲染成垃圾。这是为整个 plan 文件工作流提供的「人在回路审查」，也是我第一次把 agentic 工作分享给一个普通同事时不再尴尬。

我写本文时就把它装进了 Proof。它就是这么被 review 的。

我整篇文章是在 cmux 里、旁边开着 Proof review 写完的。

##### HACK

- 分享 plan：把 .md 丢进 Proof，发链接，把评论拉回到循环里

---

#### 17. 写你自己的 Skill

最大的提升不是用 agent，是教会它一些黏住的招。**任何我做超过两次的事情，都变成 skill**：一个我的 agent 永远能跑的可复用命令。先用「写自己的 skill」的方式自动化你的工作流。

你不用从零写。让我打通这件事的关键技巧是：把 agent 指向一个已经能跑的 skill，让它照葫芦画瓢。字面意思就是：「看一下 Compound Engineering 这个 skill，帮我做一个像它一样的来自动化 [我想自动化的 X]」。它读一个好例子，学习结构，给我搭脚手架。我用这个方法堆出了一摞 skill。

这也是我现在大部分开源生活。看我 GitHub，作品几乎都是 skill 和围绕它们的工具。last30days 起初就是我自己想要的一个 skill，现在开源 26K+ stars。Printing Press 是一整个生成 agent 原生 CLI 的工厂，是我用得最多的个人工具，我自己已经合并了 320+ PR 进去。我也是 Compound Engineering 本身的头部贡献者。这一切都没有宏大计划，每一块都是我跑得够频繁、值得让 agent 永远擅长它的某个工作流。

写一次 skill。之后每个 session 都更快。这就是 Compound Engineering 里「复利」的部分。

##### HACK

- 任何做超过两次的事情，做成 skill：「看一下 Compound Engineering 的 skill，帮我做一个像它一样的来做 [X]」

---

#### 18. 开源：给你爱的项目做贡献

让我交付自己项目的同一个循环，也能交付别人的项目。我合并进开源的 PR 已经有几百个，包括 Python、Go、OpenCV、Vercel 的 Agent Browser、OpenClaw。不是顺手改个 typo，是我每天用的工具上的真实功能。

不知不觉间，我经常出现在贡献者列表的前列：

- Compound Engineering、Superpowers、Emdash 第 3
- GStack、Paperclip 第 4
- Vercel Agent Browser 第 6
- Camoufox 第 2

@pejmanjohn 开玩笑说他打开一个仓库时，在贡献者头像格子里找我的脸已经成了他个人版的「找瓦尔多」。

但合并的 PR 不是真正的奖品，**是人**。我跳进 Discord，认识维护者，交真朋友。这对招聘也极好——我刚招了一个用这个方式认识的工程师进我新公司。你给你爱的东西做贡献，认识爱它的人，然后复利就来了。

##### HACK

- 挑一个你每天用的工具，找它真正缺的某样东西，用同样的 `/ce-plan` + `/ce-work` 循环把它做掉
- 出现在项目的 Discord 里。PR 让你进门，但留下来是因为人
- 在 X 上提供价值。每月付 1–3 美元订阅你尊敬的人。我每月付 1 美元给 @garrytan，提交 PR 时我可以发条 X 推文给他，他会收到一条「我是付费客户」的特别提醒。我也付费给 @jason、@teknium

---

#### 19. 我目前的笔记本配置

我那台两年的笔记本在我现在跑的负载下基本不能用——6 个 Claude session 加 Codex，一整天。所以我换成了 64GB 内存的 M5 Max。是头猛兽，我爱它。但即使这样它也被工作量碾压：我这台全新的机器电池有时只能撑 1 小时。

所以我恐慌性买电了。现在我到哪都背一块 Anker 电池砖，Tesla 里也常备一个 Anker 充电器，让车在路上给我续。

##### HACK

- 永不睡眠：`sudo pmset -a disablesleep 1`。背一块 Anker 电池砖；车里放一个充电器

---

#### 20. Printing Press：跑真实生活的 CLI 们

前面这些技巧大多活在终端里。这条是离开终端的。Printing Press 是一队 CLI，包装真实世界的服务，让 agent 可以直接「去办那件事」。它现在已经是独立项目 @ppressdev，3.7K+ stars，我和 @trevin 一起做。

让它们真的能跑的关键件是认证，昨晚刚发布：**Agent Cookie**。它把你真实的浏览器 session 交给 CLI，让它「以你的身份」操作——不用粘密码，不用重新登录。这就是「一个知道某服务的 agent」变成「一个登录进了那个服务的 agent」的开关。

一个真实的下午，从头到尾：

- **Tesla 预热**：孩子 10 分钟内上车——「把车预热到 72°F」。Tesla CLI 触发，我们走出来的时候车已经暖了
- **Instacart**：「在 Instacart 上的 Costco 加一打 Corona」
- **ESPN 轮询**：一个 session 替我看比赛，只在比分接近时 ping 我。我没刷新过任何东西，只收到一条真正重要的提醒
- **Alaska Airlines 给孩子订旅程**：拉机票和肩膀日期、查我们的 Atmos 余额，喂给 `/ce-plan`，得到一个订票策略——最便宜的日子和何时下手。**我是在足球场边操作的**

不是「AI 帮我写代码」。Agentic Engineering 在帮我办差、看比赛、热车、订行程，与此同时我在做别的事。

##### HACK

- 从 [printingpress.dev](https://printingpress.dev) 装一个现成的 CLI，把差事直接交给 agent
- 不用密码痛苦：Agent Cookie 把你真实的浏览器 session 给 CLI，让它以你的身份操作
- **真正的 hack 是 print 你自己的**。挑你一整天都在用的某个 API 或服务，让 Printing Press 给它生成一个 agent 原生 CLI。你为自己工作流定制的那一个，才是真的会改变你工作方式的那一个

---

#### 21. 老实话：AI 上瘾

Agent 本来是要替我们干活的。结果我所有的朋友都比一辈子任何时候都干得更狠。

简单的回应是「歇一歇，去碰碰草」。但这不是这一节的主题。**这一节的主题是上瘾**。用 agent 构建是有史以来最棒的电子游戏，那个循环就是这么爽。

我有一些朋友我是真担心。他们因为「能造出任何东西」这件事兴奋到不做别的事。然后他们发布，没用户。这没关系，我也发布过很多没用户的东西。陷阱不是空的发布，**陷阱是消失在「构建」里、丢掉你身边的人**。

所以小心。和你爱的人聊聊。问问自己：到底有没有人真的想要你在做的这个东西？如果诚实的答案是「这只是给我自己用的工具」，那也 OK。我做过的最好的一些东西就只是给我自己。

如果你确实想要受众，那就走 Gary Vaynerchuk 一直布道的内容路径。从某个地方开始，往虚空里发，希望有一个人注意到。然后三个、十个、一百个，慢慢做到几千个。**没有人是从几千开始的**。任何你做的东西也一样。

##### HACK

- 休息。去碰碰草
- 跟你爱的人聊天
- 做人想要的东西——哪怕「人」就只是你自己

---

#### 22. 这篇文章就是这么写出来的

这是一份 markdown 文件。Claude Code 在 cmux 里，我对着 Monologue 说话：「把『不用 IDE』那个开头进化一下」「把『别读 plan』那节写得更辣一点」「加上 Tesla 和 Instacart 那段故事」。它重写，我反应，然后丢进 Proof 评审。last30days 喂了新鲜素材。顺便说，这次没用 Zed 了，我不用了。**不用 IDE。不打字写代码。说话、规划、构建**。从书桌、沙发、车里、足球场边。

这就是 6 月这个时间点我所知道的全部。一个语音 app、一个 plan-file 插件、几处配置改动、一堆 tab、一台 Mac mini、两台远程机器、和一队跑真实生活的 CLI。

##### HACK

- 把整篇文章复制粘贴给你的 agent，让它把能装的东西都装上。你的 agentic engineering 工作流会因此变好
