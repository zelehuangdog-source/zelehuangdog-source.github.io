---
title: "深入 Harness：我错看 pi 的四个月"
date: 2026-08-13
author: repost
categories: [转载, Claude-Code实战]
tags: [翻译, pi, Claude-Code, AI代理, 开发者工具, 转载]
---

> **摘要**：本文作者 Nick Nisi 分享了他对编码代理 pi 从"看不上"到"彻底拥抱"的四个月心路历程。起初他把 pi 当作更快的 Claude Code 使用，因自己围绕 Claude Code 构建的整套工具链无法迁移而放弃；直到他意识到 pi 的全部产品形态就是扩展（extension），可以在会话内部做任何事情。十天之内他写出了 26 个扩展、删除了 dotfiles 里 2.5 万行代码，并把 pi 用成了更强大的开发环境。文章核心洞察：当 harness 允许你进入内部时，构建在它之外的工具才真正变得可替代。对 Claude Code 用户理解"可扩展 harness"的设计哲学极具参考价值。

> **转载声明**：本文**翻译整理**自：《Inside the Harness: Four Months of Being Wrong About pi》。**内容版权归原作者及原出处所有**，本文仅供学习交流，**所有观点均属原作者，不代表本站立场**；如有侵权，请联系删除。


---

> **TL;DR**：我把 pi 当作更快的 Claude Code 用了四个月，感到厌倦后告诉别人不值得切换。然后我意识到，我围绕 harness 构建的每一个工具——bash hooks、tmux 抓取器、独立 CLI——之所以存在，正是因为 harness 不让我进入内部。pi 会让我进去。十天之后，我在一个 [monorepo](https://github.com/nicknisi/pi-extensions) 里有了 26 个扩展，并从我的 dotfiles 里删除了 25,635 行代码。

7 月 20 日，一位同事在我们的 `#pi` Slack 频道里问了这样一个问题：

> 为什么 @nicknisi 说他认为 pi 不再是个好主意？我想在深入之前搞清楚。

Zack 已经开始在 pi 上构建东西，刚刚听我说这不值得。问得有理，他想知道理由。
我当时的全部回复是："但我爱 pi 呀！:homer-disappear:"
我回避了这个问题。
两周之后，我在十天里跑了 188 次 pi 会话、17 次 Claude Code 会话。Zack，这才是真正的答案。

#### 我已经建好的城市

我生活在终端里。tmux、Neovim、通过管道接进 `fzf` 的 `rg`、一条被我调试到没人能看懂的状态栏。
到今年春天，我已经围绕 Claude Code 建起了一座完整的城市。case 通过流水线分发代理，并用机械的方式强制约定。Fleet 监视着我 tmux 服务器里的每一个代理，把需要我处理的排到最上面。[sessions](https://github.com/nicknisi/sessions) 为机器上的每一次 Claude、Codex 和 pi 对话建立了索引，这样任何会话都能查询之前的任何一次。我的 plugins 仓库承载着 skills，而 tmux 状态栏承载着通知。
它们没有一个运行在 Claude Code 内部。它们全都运行在 Claude Code *旁边*——在 bash、Bun 和 `~/.cache` 下的文件里，从 harness 暴露出来的任何缝隙伸进去。

#### 打动我的那个比喻

今年春天在我们 Slack 的某个角落，人们开始把 pi 比作编码代理界的 Neovim，把 Claude Code 比作 VS Code。一个把你能想到的一切都打包进去；另一个交给你一个光秃秃的主循环，然后退到一边。
如果你看过我的 dotfiles，你就知道为什么这个比喻打动了我。
我花了几周才真正试了它。然后，在 3 月 18 日星期三下午 2:36，我在 `#pi` 里开玩笑说 pi 团队还拥有 `shittycodingagent.ai` 这个域名。
2:48，我第一次打开 pi，输入了 `hello`。
3:07，我问它"pi 能做什么？"，读完了回答，然后想用 `:qa` 退出。
到 3:20，我已经把 `~/.pi` 符号链接进了我的 dotfiles 仓库。3:27，我贴进了我的 Claude Code 状态栏脚本，问："我们能给 pi 复刻一个吗？" 当晚我就提交了一个 pi 状态栏扩展、一个 tmux 状态桥、一个 Night Owl 主题，以及 spinner 动词。
第二天，我问 pi 能不能给自己写扩展。它写了一个。我把结果发到 Slack，只配了一个 🤯。

#### 逐渐漂移

这是我按月份统计的 pi 使用量，以磁盘上的会话数为准：

| 月份 | pi 会话数 | Claude Code 会话数 |
|---|---|---|
| 3 月 | 111 | ~1,200 |
| 4 月 | 17 | ~590 |
| 5 月 | 29 | ~535 |
| 6 月 | 2 | ~400 |
| 7 月 | 0 | ~710 |

我 dotfiles 里的 pi 提交记录讲述了同样的故事：3 月 19 次，然后是 2、5、1、4。7 月零会话。
pi 并不差。它很棒——更快、更安静、权限弹窗更少、模型切换真的能用。`#pi` 里的每个人都这么说，他们是对的。
只是我的城市没有跟着我一起搬家。Fleet 只会读 Claude Code 的 hooks。case 只会调度 Claude。我的 tmux 通知只会为 Claude 窗格亮起。在 pi 里度过的每一个小时，都是在我花了几个月打磨的工具链之外度过的。
我 dotfiles 里最有说服力的一条提交来自 5 月 28 日：`feat: cut tmux + Claude Code over to fleet`。同一天我发布了那篇 Fleet 文章。那是这座城市的高光时刻，也是 pi 对我在结构上变得不可能的那一刻。
所以当 Zack 问我的时候，我告诉他不值得。我没有撒谎。我描述的是真实的成本。我只是把原因搞错了。

#### 我错过了什么

8 月 1 日星期六上午 10:08。我在我的 dotfiles 仓库里打开 pi，输入：

> 到目前为止，我一直在像用 claude code 一样使用 pi。我知道我可以更深地使用 pi、让它更好，但我不确定具体怎么做。除了一个类似 claude code 的简单编码 harness，它还有哪些能力？

四个月来我一直把它当成一个更快的终端聊天窗口，却从没问过它还能做什么。
pi 刻意不内置 subagents（子代理）、plan mode（计划模式）、权限弹窗、todos、后台 bash 和 MCP。这些每一个都是有意留出的扩展点。扩展是一个 TypeScript 模块，可以注册工具、替换内置的 `read`/`edit`/`bash` 工具、钩住每个生命周期事件、拦截或改写工具调用、接管编辑器、footer、header 和覆盖层。这个模块表面*就是*产品。如果它不是一个扩展，pi 就不会内置它。

#### 状态栏的证明

这个对比的两边我都亲手建过，所以它才戳中了我。
为了判断一个 Claude Code 代理是在工作、等待还是完成，Fleet 融合了三个信号，而其中任何一个它都不敢单独相信。写状态文件的 bash hooks（快、结构化、乱序到达）。每个窗格的 JSONL 事件日志（知道意图，但有延迟）。以及 `tmux capture-pane`（地面真相，大约花费 ~50ms，而且 Claude 的 spinner 是*动画*的，所以固定字形匹配会漏掉一半的帧）。这个项目给我的全部教训是：**每一层只有在它真正值得信任的事情上才具有权威性**——因为整个栈里没有任何东西能直接问代理它在做什么。
而 pi 的状态栏扩展只有 519 行 TypeScript。它订阅 `tool_execution_start` 和 `agent_end`，调用 `ctx.getContextUsage()`，把一个渲染函数交给 `ui.setFooter()`，然后从 `footerData` 读取分支。没有抓取、没有融合、没有新鲜度规则。我旧 Fleet 栈里每一个 bash hook 和 `capture-pane` 抓取，都是这一个订阅就能替代的工作。

#### 十天

光 8 月 3 日一天：58 个会话。
我在 `#pi` 里发了条消息："有没有人把自己的 pi 配置开源了？我很想翻一翻！"，然后把能找到的每份都读了一遍（oh-my-pi、pikit、LazyPi，还有某个陌生人的 `my-pi-setup`），就像凌晨 1 点读别人的 dotfiles 那样。
我移植了那些我真正想念的贴心功能：动态工作流、`/goal` 和 `/loop`、`/btw`、会话自动命名、`AskUserQuestion`、草稿暂存、recap 卡片。我替换了我抱怨了好几周的网页搜索工具。我做了 `/mg`——一个《人生切割术》（Severance）风格的"100% 文件完成度"动画，主角是我被像素化的 CEO，还带声音——因为我能做到，而且没人拦得住我。两小时后我把它[发了出来](https://x.com/nicknisi/status/2084377731123355695)，配了它唯一配得上的标题：WorkOS 的一名 Developer and Agent Experiences Engineer 的工作是神秘而重要的。
我还认真地问了 pi：我该不该把这一切打包成一个发行版（distro）给别人用。它拉起了一个 18 代理的研究工作流，回来告诉我：不要。这个生态里的发行版位置同时是满的也是空的，而且能力包（capability package）比精心策划的配置（curated config）大约好上 400 倍。然后它继续往下，告诉了我两件我根本没问的事：我每个会话都在加载大约 2,600 行损坏的代码，还有我的 `trust.json` 被提交到了一个公开仓库里，里面带着我的绝对路径。
那才是最有用的部分。它不同意我，用数字支撑自己的观点，然后继续去寻找还有什么地方不对劲。
8 月 6 日上午 10:36，我把所有东西从 dotfiles 搬进了一个独立的 monorepo。一次提交：53 个文件变更，25 行新增，**25,635 行删除**。

#### 回家了

现在 `pi-extensions` 是 26 个包、32,847 行 TypeScript。状态栏、composer、header、spinner、turn 计时器、固定提示词。子代理分发、工作流、codemode、一个 LLM 议事会（LLM council）。`relay` 用于会话间通信。`cloak` 负责在 `read` 结果到达模型之前擦除其中的秘密。`claude-compat` 让我的 Claude Code 插件继续可用。其中大部分已经发布到了 npm。
还有一个 `bosun`：一个 pi 会话，变成一群无头 pi 船员（headless pi crewmates）之上的联络人——派发任务、转达进度、批准交付。有一个晚上，我让它同时指挥七个代理，然后就去睡觉了。
这又是 Fleet。同样的问题，同样的形状。区别在于：Fleet 是 harness 之外的一个编译好的二进制文件，对着终端输出眯眼看；而 bosun 是会话内部的一个包。
这个故事还有一个工作版本，等允许的时候我会好好讲。简版：就在我拆散 dotfiles 的那一周，我们在 WorkOS 开始基于 pi 构建。因为 pi 的整个表面都是扩展，从"我们自己的 harness 到底长什么样"到"喏，装它"之间的距离，最终是以天来衡量的。
这次切换不是因为速度、成本或模型选择。这些都很重要，但没有一个能让我在四个月里动起来。真正打动我的，是注意到我的栈里有多少东西（case、Fleet、抓取器、`~/.cache` 下的那些文件）仅仅是因为我无法进入 harness 内部才存在的。pi 放你进去。Zack，抱歉当初躲开了——这才是我当时该说的话。
