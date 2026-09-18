---
title: "Claude Code 两大盲区及修复方案"
date: 2026-05-07
author: repost
categories: [转载, Claude-Code实战]
tags: [Claude-Code, skills, AI-Agent, 翻译, 转载]
---

> **转载声明**：本文**翻译整理**自：来源 https://x.com/akshay_pachaar。**内容版权归原作者及原出处所有**，本文仅供学习交流，**所有观点均属原作者，不代表本站立场**；如有侵权，请联系删除。

#### Claude Code 有两大盲区，我把它们都修好了

> 没有 CLAUDE.md、没有子代理、没有规则文件能修复它们。但这两个 skill 可以。

Claude Code 有两个上下文缺口，它们以相同的方式失败：代理看不到它需要的东西，然后烧着 token 不断尝试。

---

#### 第一个盲区：网页抓取

`web_fetch` 不会返回原始页面内容。它会把页面内容通过一个更小的模型处理，返回一份摘要，引用限制为 125 个字符——所以你没法用它来提取完整的教程、产品规格或帖子内容。

`curl` 返回原始 HTML，但会被有反爬保护的网站拦截（Amazon、LinkedIn、大多数电商），无法渲染 JavaScript SPA，而且由于速率限制在规模化使用时会失败。两者都会在约 100KB 时被截断。

#### 第二个盲区：后端集成

当 Claude Code 通过 MCP 与 Supabase 这样的后端对话时，它通过多次独立调用来发现状态（`list_tables`、`execute_sql`、`list_extensions`），每次只返回部分视图。

Auth provider 的配置根本无法查询。当某些东西失败时，错误信息无法区分是平台级别的拒绝还是代码级别的拒绝，于是代理进入重试循环，每次尝试都在烧 token。在我们最近的测试中，一个基于 Supabase 构建的 RAG 应用消耗了 1040 万 token，并且需要 10 次手动修复。

**Bright Data 解决了第一个问题。InsForge 解决了第二个问题。两者都是开源的。**

今天，让我们看看如何将它们设置为 Claude Code 的 skill 并用它们来构建一些有趣的东西。

---

#### Bright Data 配置

Bright Data skill（开源）添加了抓取基础设施，能处理 `web_fetch` 和 `curl` 无法应对的一切。

代理获得了一个四级降级机制，根据目标网站的要求逐步升级：原生 fetch → curl → 浏览器自动化 → 带住宅 IP 和自动验证码解决的代理网络。

对于代理工作流来说，更有用的能力是**结构化数据提取**。

Bright Data 不返回代理需要解析的原始 HTML，而是为 40+ 平台（Amazon、LinkedIn、Instagram、TikTok、YouTube、Reddit）提供预构建的提取器，返回干净的 JSON，包含特定字段如产品价格、评论分数、个人资料数据和帖子内容。

```bash
npx skills add brightdata/skills
```

这会安装多个 skill，涵盖抓取、搜索、结构化数据源、MCP 编排、SDK 最佳实践和 bdata CLI。

---

#### InsForge 配置

同样的 RAG 应用，在 Supabase 上消耗 1040 万 token，在 InsForge 上消耗 370 万 token，零错误。

InsForge（开源，Apache 2.0）充当代理使用 Skills 和 CLI 时的后端上下文工程层。

安装全部四个 Skills（主要的文档和诊断层）：

```bash
npx skills add insforge/insforge-skills
```

这会安装 insforge（SDK 模式）、insforge-cli（基础设施命令）、insforge-debug（故障诊断）和 insforge-integrations（第三方 auth provider）。总元数据开销：会话开始时约 714 token。

将 CLI 链接到你的项目（主要的执行层）：

```bash
npx @insforge/cli link --project-id <project-id>
```

---

#### 构建一个 Google Docs 克隆

有一个 10 小时的视频教如何构建 Google Docs 克隆。

十小时的视频意味着数百个实现细节——实时协作、文档状态同步、编辑器工具栏结构和权限。

这是大量的上下文，Claude Code 很可能难以抓取。即使它设法抓取了，压缩也会删除许多细节。

安装了两个 skill 后，你可以这样做：

```
I want to build what's shown here:
https://www.youtube.com/watch?v=gq2bbDmSokU
Use Bright Data skills to scrape it and then
InsForge as the backend to implement.
Add Google OAuth and build a clean Google-doc
like interface. On every doc, add an "Ask AI"
button that chats with GPT-4o about the content.
Use InsForge's model gateway for the LLM capabilities.
```

Bright Data 抓取了完整的视频内容（字幕、元数据、结构化描述），Claude Code 将其用作构建规格。

InsForge 一次搞定了后端：Google OAuth、带 RLS 的数据库 schema、存储、边缘函数，以及通过 InsForge 内置功能实现的 GPT-4o 聊天模型网关。

最终，它生成了一个可用的 Google Docs 克隆，具备实时编辑、Google OAuth 和 AI 驱动的文档聊天——从单个提示构建，零错误。

---

#### 不限于教程

YouTube 的例子只是抓取方面的简单案例，这不限于教程或从零开始构建。

同样的工作流适用于网上任何技术内容，比如：

- 一个 Reddit 帖子，有人描述了他们如何优化实时同步
- 一个 Hacker News 讨论，走读了一个 auth 架构
- 一个竞品的产品页面，有值得复制的功能

给 Claude Code 链接加上 Bright Data skill，代理抓取内容、理解描述的内容、并在现有应用中实现它。

**所以网上任何技术内容都可以变成构建规格。**

对于基础来源，原生抓取工具就够用了。Bright Data 在以下场景下不可或缺：主动抵抗抓取的来源——比如有激进速率限制的 Reddit 帖子、有反爬检测的 Amazon 产品页面、会对浏览器指纹识别的 LinkedIn 个人资料，以及需要完整浏览器渲染才能加载内容的 JavaScript SPA。

---

#### 相关链接

- Bright Data Skills 仓库
- InsForge GitHub 仓库

---

> 👉 你的 Claude Code 默认配置里装了哪些 skill？
