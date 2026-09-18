---
title: "Pydantic 修复了我的 Agent 记忆"
date: 2026-05-26
author: repost
categories: [转载, Agent记忆与状态]
tags: [翻译, Agent记忆, 知识图谱, Pydantic, 转载]
---

> **摘要**：本文指出 Agent 记忆系统的核心问题不在于存储，而在于结构。向量数据库无法完成多跳推理，知识图谱虽能解决连接问题，但若不约束抽取模型的 schema，图谱退化为无类型的噪音。作者介绍了使用 Pydantic 定义本体（Ontology）的方法——通过 EntityModel 和 EdgeModel 为抽取步骤提供类型约束，使知识图谱具备结构化检索能力。文中以 Zep 开源框架为例，完整演示了从 schema 定义、数据摄入、抽取管线到上下文模板注入的全流程。

> **转载声明**：本文**翻译整理**自：《Pydantic fixed my Agent's Memory》。**内容版权归原作者及原出处所有**，本文仅供学习交流，**所有观点均属原作者，不代表本站立场**；如有侵权，请联系删除。


---

#### Pydantic 修复了我的 Agent 记忆

你的 Agent 记住了所有东西，却什么都没理解。

Agent 记忆最初从向量数据库起步——将事实存为文本块（chunk），按相似度检索。

这在查询需要跨 chunk 连接事实时就会崩溃。问题不在相似度，而在**结构**。

知识图谱（Knowledge Graph）是解药：实体作为节点、关系作为边，用图遍历替代匹配。

但大多数团队撞上了另一堵墙。

当你给 Agent 一个知识图谱用作记忆时，默认行为是由负责抽取的 LLM 自行决定图的结构——它自己挑选实体类型、关系标签和属性。

结果是泛化的、无差别的。

例如，你在构建一个客户支持 Agent。你喂给它 50 段支持对话，覆盖客户、工单、功能和升级历史。

你问："哪些企业客户有未关闭的 sev-1 工单？"

图里有数据。但每张工单都被存为一个 "Topic" 节点，每个客户都是一个 "Object"，每条关系都是 "RELATES_TO"。

根本没法按类型、严重等级或套餐层级过滤。查询返回的全是噪音。

Agent 没忘记任何东西。只是**没人告诉它该关注什么**。

修复方法很直接：**预先定义 schema**。告诉抽取模型你的领域里存在哪些实体类型、哪些关系是合法的、每种类型携带什么属性。

这种组织蓝图叫做**本体（Ontology）**。可以把它理解为你 Agent 大脑的 schema。

接下来我们看看为什么这很重要、缺失它会出什么问题，以及如何用一个 100% 开源方案来实现它。

#### 为什么平面检索在多跳推理上失败

基于向量的记忆将事实存为文本块，按语义相似度检索。当一个查询需要连接不在同一个 chunk 中的事实时，它就失效了。

考虑一个项目中存储的三条事实：

1. Alice 管理 Project Atlas
2. Project Atlas 运行在 PostgreSQL 上
3. PostgreSQL 集群在周二宕机了

一个查询如"Alice 的项目是否受到了周二故障的影响？"需要这三条事实。

向量搜索只会召回事实 1 和 3，因为两者都提到了相关术语。事实 2 是连接 Alice 和 PostgreSQL 的桥梁（通过 Project Atlas），但它既没提到 Alice 也没提到周二。相似度搜索漏掉了它。

知识图谱将实体存为节点、关系存为边。它不是匹配文本，而是遍历连接。

这条链路（Alice → 管理 → Project Atlas → 运行在 → PostgreSQL）正是多跳推理能工作的原因，而它对平面向量检索是不可见的。

#### 记忆管线与抽取步骤的位置

每个基于图的 Agent 记忆系统都遵循一条通用管线：

1. **摄入（Ingest）**：原始数据进入（对话消息、文档、JSON 业务数据）
2. **抽取（Extract）**：LLM 阅读原始数据，决定存在哪些实体、什么关系连接它们、哪些属性重要
3. **存储（Store）**：抽取的实体成为节点，关系成为边，全部持久化到图中
4. **检索（Retrieve）**：查询时搜索图并组装相关事实
5. **交付（Deliver）**：检索到的事实格式化为上下文块，注入 Agent 的 prompt

抽取步骤是一切决策的发生地。它决定了你的图包含什么、结构如何、下游能查询什么。

问题在于：在大多数框架中，这一步是黑盒。你传入文本，LLM 抽出 "实体" 和 "关系"，你得到节点和边。LLM 自己决定类型、标签和属性。

你对它如何分类**零控制权**。

来看看怎么修复。

#### 用 Pydantic 定义 Schema

修复方式与 AI 技术栈各处使用的模式相同：

- FastAPI 端点用 Pydantic 响应模型
- Function Calling 工具用 Pydantic schema
- Agent 记忆在 Zep 中也用同样的方式

使用 `EntityModel`（Pydantic `BaseModel` 的子类）定义自定义实体类型，配合 `EntityText` 字段和描述来引导抽取模型：

```python
from zep_cloud.external_clients.ontology import EntityModel, EntityText
from pydantic import Field

class Project(EntityModel):
    """
    Represents a specific software project, application, 
    or codebase that the user is building or contributing to.
    """

    project_status: EntityText = Field(
        description="Current status: active, completed, paused, or archived.",
    )
    project_type: EntityText = Field(
        description="Type of project: web app, mobile app, API, CLI tool, etc.",
    )
```

这里的 docstring 和字段描述（description）很关键——好的描述配合具体示例能给抽取器足够的信号来准确分类。

上面的 Pydantic 描述不仅仅是分类指令，它们**教会抽取器它不认识的词汇**。

Technology 实体遵循相同模式：

```python
class Technology(EntityModel):
    """
    Represents a programming language, framework, library, 
    database, or tool that the user works with.
    """

    tech_category: EntityText = Field(
        description="Category: programming language, framework, database, etc.",
    )
```

边（Edge）类型使用 `EdgeModel` 并携带自己的属性：

```python
from zep_cloud.external_clients.ontology import EdgeModel

class WorksOn(EdgeModel):
    """The user is currently working on, building, or contributing to a project."""
    role: EntityText = Field(
        description="User's role: lead developer, contributor, maintainer, etc.",
    )

class UsesTechnology(EdgeModel):
    """The user actively uses or works with a specific technology."""
    proficiency: EntityText = Field(
        description="Proficiency level: beginner, intermediate, advanced, or expert.",
    )
```

最后，使用 `EntityEdgeSourceTarget` 将这些连接到图中，定义源/目标约束——即哪些实体类型可以通过哪些边类型相连：

```python
from zep_cloud import EntityEdgeSourceTarget

client.graph.set_ontology(
    entities={"Project": Project, "Technology": Technology},
    edges={
        "WORKS_ON": (
            WorksOn,
            [EntityEdgeSourceTarget(source="User", target="Project")],
        ),
        "USES_TECHNOLOGY": (
            UsesTechnology,
            [EntityEdgeSourceTarget(source="User", target="Technology")],
        ),
    },
)
```

这段代码约束了：
- `WORKS_ON` 只能连接 User 到 Project
- `USES_TECHNOLOGY` 只能连接 User 到 Technology

任何不符合这些约束的关系都不会生成类型化的边。

#### 底层发生了什么

当一段对话在 schema 激活状态下被摄入时，Zep 的抽取管线执行五个步骤：

1. **实体抽取**：识别文本中的命名实体
2. **实体消解**：合并重复项（"Nexus" 和 "the Nexus project" 合为一个节点）
3. **事实抽取**：识别关系并输出为类型化的边
4. **事实消解**：检测矛盾并将过时的事实标记为无效（保留历史）
5. **时间抽取**：解析时间引用并映射为每条边的有效时间窗口

你的 Pydantic schema 引导步骤 1 和 3。实体类型告诉抽取器**找什么**，边类型及其约束告诉它**分类什么关系**。消解和时间处理自动完成。

#### 实际效果演示

我们摄入一段对话，其中开发者 Alex 讨论他的工作（一个名为 Nexus 的活跃 Web 应用、他的技术栈、熟练度级别）。

查询 Project 节点返回 Nexus，带有已填充的 `project_status` 和 `project_type` 属性。

节点不再是泛化的 "Topic" 或 "Object"。它是一个 **Project**，带有 schema 中定义的结构化字段。

边也是有类型的：
- `WORKS_ON` 携带 `role: lead developer`
- `USES_TECHNOLOGY` 携带 `proficiency: advanced`（Python 和 Docker）、`proficiency: intermediate`（TypeScript）

现在可以按状态过滤项目、按类别过滤技术，精确回答"哪些活跃项目使用了 PostgreSQL"。

#### 上下文模板（Context Templates）

最后一块拼图是**上下文模板**——将类型化事实组装为可注入 prompt 的上下文块。

你可以定义包含哪些边类型和实体类型，Zep 会将它们格式化为带时间标注的字符串，注入 Agent 的 prompt：

```python
client.context.create_context_template(
    template_id="dev-context",
    template="""# PROJECTS
%{edges types=[WORKS_ON] limit=5}

# TECH STACK
%{edges types=[USES_TECHNOLOGY] limit=10}

# PROJECT DETAILS
%{entities types=[Project] limit=5}

# TECHNOLOGIES
%{entities types=[Technology] limit=10}""",
)
```

结果上下文块中的每条记录都是有类型的、带时间标注的，并携带定义好的属性。模板保存一次，在 Agent 调用中按 ID 引用即可。

#### 10/10/10 约束与 Schema 作为推理边界

Zep 强制执行硬性限制：**10 个自定义实体类型、10 个自定义边类型、每种类型 10 个字段**。

这是有意为之——迫使开发者思考领域中**什么才是重要的**，而不是对所有东西建模。

源/目标约束也充当 Agent 允许记忆什么的**护栏**。如果 schema 不包含连接 Project 到 Competitor 的边类型，抽取模型就不会创建这条关系——即使对话中同时提到了两者。

**Schema 定义了合法记忆的空间。**

这与 Typed Function Calling 背后的原理相同——我们约束 LLM 的输出空间，使其不能产生无效参数。记忆 schema 将同样的约束施加于 Agent 存储的内容。

从 3-4 个实体类型和 3-4 个边类型起步，覆盖领域逻辑的 80%，然后逐步增加复杂度。

**没有 schema 纪律的 Agent 记忆，就是一个表现得像向量存储的图。**

某种意义上，你付出了图构建的代价，却没获得结构化检索的收益。

Schema 是你拿回这个收益的方式。而因为它就是 Pydantic，所以没有新东西要学。

这对**领域特定应用**尤其重要。LLM 抽取在通用知识上表现还行，但一旦你的领域有内部术语、与常见词碰撞的产品名、或训练数据中不存在的行话，无引导的抽取就会产生垃圾。Schema 弥合了这个鸿沟——它将领域词汇直接带入抽取步骤，LLM 不需要事先见过你的术语，只需要你写的定义。
