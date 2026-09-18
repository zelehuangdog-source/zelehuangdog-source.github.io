---
title: "LLM推理工作原理：从Prompt到流式响应的全链路解析"
date: 2026-05-07
author: repost
categories: [转载, LLM原理与技术]
tags: [LLM, 推理, Transformer, KV缓存, 量化, 转载]
---

> **转载声明**：本文**转载整理**自：来源 https://x.com/akshay_pachaar/status/2050941458614751327。**内容版权归原作者及原出处所有**，本文仅供学习交流，**所有观点均属原作者，不代表本站立场**；如有侵权，请联系删除。

#### 摘要

本文从第一性原理出发，完整拆解了 LLM 推理的全链路过程：从分词（Tokenization）、嵌入（Embedding）、自注意力（Self-Attention），到预填充/解码（Prefill/Decode）的双阶段拆分、KV 缓存优化，以及量化技术。核心洞察包括：**预填充阶段是计算瓶颈（GPU 算力受限），解码阶段是显存带宽瓶颈（等待数据加载）**；KV 缓存是让自回归生成可行的关键优化，但也是长上下文的主要代价；量化是生产部署中性价比最高的优化手段。理解这些底层机制，能帮助你在模型部署时做出正确的优化决策。

---

#### LLM 推理的工作原理

一次关于从 Prompt 到流式响应之间发生了什么的第一性原理之旅：分词、嵌入、注意力、预填充/解码拆分、KV 缓存和量化。

你输入一个 Prompt。几百毫秒后，文字开始逐个流回给你。看起来很简单，但其实不然。

从你按下回车到第一个 token 出现之间发生的事情，是现代计算中工程最为精密的流水线之一。而最奇妙的是——模型在同一个 GPU、同一次请求中，完成了两项完全不同的工作，且面临完全不同的瓶颈。

一旦你理解了这些，你将再也不会以同样的方式看待一次 `generate()` 调用。

#### 心智模型

LLM 是一个预测下一个 token 的神经网络。只预测一个 token。然后把这个 token 拼到你的 Prompt 末尾，再预测下一个。如此循环。

就这么简单。这就是整个循环。

有意思的问题是：它如何预测下一个 token，以及为什么第二个 token 出来的速度远快于第一个？

#### 第一步：文本变成数字

神经网络不读英文，它读向量。所以你的 Prompt 经历的第一步是**分词（Tokenization）**——把文本切成小块，并为每块分配一个整数 ID。

大多数现代 LLM 使用一种叫做**字节对编码（Byte Pair Encoding, BPE）**的方案。思路是：从原始字符开始，反复合并出现频率最高的相邻字符对，直到得到一个约 50,000 个片段的词汇表。常见词如 `the` 对应一个 token，罕见词如 `unhappiness` 会被拆分成 `un` + `happi` + `ness` 等几个 token。

```python
prompt = "How does inference work?"
ids = tokenizer.encode(prompt)
# ids -> [2437, 1374, 32278, 670, 30]
```

这一步比人们意识到的更重要。在分词器训练数据中代表性不足的语言会被切成更多片段，意味着同一句话需要更多 token，导致更高的成本和更慢的响应。

#### 第二步：每个 token 变成向量

每个整数 ID 在一个巨大的矩阵——**嵌入表（Embedding Table）**中查找对应行。如果你的模型词汇表有 50K 个词，隐藏维度为 4,096，则该表的形状是 `[50000, 4096]`。取出一行，就得到一个向量。

```python
# embedding_table 的形状为 [vocab_size, hidden_dim]
vectors = embedding_table[ids]   # 形状: [num_tokens, 4096]
```

这些向量不是随机的。训练过程中，模型不断微调它们的位置，使得语义相似的 token 在 4,096 维空间中成为邻居。`king` 和 `queen` 是邻居；`python` 和 `snake` 在某个轴上是邻居，`python` 和 `javascript` 在另一个轴上是邻居。

嵌入层也是注入位置信息的地方，因为注意力机制本身不知道哪个 token 在前。现代模型使用如 **RoPE** 这样的方案，根据 token 在序列中的位置对向量进行旋转。

#### 第三步：层层注意力

现在真正的工作开始了。你的向量序列被送入一组 **Transformer 层**，通常 32 层或更多，依次通过。每一层做的事大致相同：

1. 通过**自注意力（Self-Attention）**在 token 之间混合信息
2. 通过**前馈网络（Feed-Forward Network）**在每个 token 内部混合信息

自注意力是值得深入理解的部分。对每个 token，该层通过三个学习到的权重矩阵产生三个新向量：

```python
# x 是该层的输入，形状 [num_tokens, hidden_dim]
Q = x @ Wq  # 查询（queries）
K = x @ Wk  # 键（keys）
V = x @ Wv  # 值（values）
```

现在你有了每个 token 的三种视图。关键在于：每个 token 用它的**查询**去看其他所有 token 的**键**，匹配强度决定了要混入多少那个 token 的**值**。

```python
# scores: 每个 token 对其他 token 的注意力分数
raw     = Q @ K.T
scaled  = raw / sqrt(hidden_dim)  # 保持 softmax 数值稳定
weights = softmax(scaled)          # 每行一个 token，加和为 1
attention_output = weights @ V
```

这就是魔法所在。一个 token 通过环顾四周、拉取它认为有用的信息来决定自己需要什么上下文。堆叠 32 层，你就得到了一个能跨越数千个 token 追踪引用关系的模型。

注意力之后，每个 token 的向量经过一个小型的两层前馈网络，完成模型真正"知道"的大部分工作。**注意力负责搬运信息，前馈网络负责处理信息。**

#### 第四步：预测下一个 token

经过最后一层后，模型取出最后位置的向量，投影回词汇表大小，然后应用 softmax 得到所有可能下一个 token 的概率分布。从这个分布中采样，你就得到了第一个生成的 token。

现在我们进入有趣的部分。

#### 没人告诉你的两个阶段

生成一个 200 token 的回复不是一项任务，而是两项在底层完全不同的任务。

##### 阶段一：预填充（Prefill）

当你提交 Prompt 时，模型必须在生成任何内容之前处理所有输入 token。好消息是：这可以**并行**完成。所有 token 的 Q、K、V 同时计算。注意力作为一次大规模的矩阵-矩阵乘法运行。

GPU 喜欢这个。矩阵-矩阵乘法正是它们天生擅长的。这里的瓶颈是**原始算术吞吐量**：GPU 以硅片允许的最大速度全力计算。

衡量这个阶段的指标是 **TTFT（Time to First Token，首 token 延迟）**。它是第一个词出现在你屏幕上之前的空闲时间。

```python
# 预填充：一次性处理整个 Prompt
hidden = embed(prompt_tokens) + positions
for layer in model.layers:
    Q, K, V = project(hidden)             # 对所有 token 同时
    hidden  = attention(Q, K, V) + hidden
    hidden  = feedforward(hidden) + hidden
    cache_kv(layer, K, V)                 # 保存供后续使用
first_token = sample(project_to_vocab(hidden[-1]))
```

##### 阶段二：解码（Decode）

一旦第一个 token 出来，模型切换模式。要生成第 51 个 token，它只需要为**那一个 token** 计算 Q、K、V。前面 50 个 token 呢？它们的 K 和 V 向量没有变。重新计算就是浪费。

所以模型逐个循环，每次一个 token：

```python
# 解码：每次迭代一个 token
token = first_token
steps = 0
while token != STOP and steps < MAX_STEPS:
    x = embed(token) + position(steps)
    for layer in model.layers:
        q, k, v = project(x)
        K_all, V_all = caches[layer].append(k, v)  # 缓存历史 + 新的
        x = layer.forward(q, K_all, V_all, x)      # 注意力 + FFN，残差
    token = sample(project_to_vocab(x))
    steps += 1
    yield token
```

注意变化了什么。你不再是用一个查询矩阵乘以一个键矩阵，而是用**一个查询向量**乘以一个键矩阵。计算量很小。

但 GPU 仍然需要从显存中加载每一个权重矩阵和每一个缓存的 K、V 来完成这微小的计算。瓶颈突然翻转了。芯片有充足的计算余量，却只能坐在那里等待显存传递下一块数据。

这就是为什么**解码是显存带宽瓶颈，预填充是计算瓶颈**。同一个模型，同样的硬件，完全不同的性能特征。

这里的衡量指标是 **ITL（Inter-Token Latency，token 间延迟）**：连续流出的 token 之间的间隔。低 ITL 是让模型感觉快的关键。

#### KV 缓存：让这一切可行的优化

上面那行 `append_to_cache` 才是真正在扛大头。没有它，生成 1,000 个 token 的回复意味着每一步都要对不断增长的整个序列重新计算注意力。二次复杂度，痛苦地慢。

有了它，你只需保存 K 和 V 矩阵一次，然后永远复用。大致结构如下：

```python
# 每个 Transformer 层一个 KVCache
class KVCache:
    def __init__(self):
        self.K = None  # 目前为止所有的 key，形状 [tokens, dim]
        self.V = None  # 目前为止所有的 value，形状 [tokens, dim]

    def append(self, k_new, v_new):
        if self.K is None:
            self.K, self.V = k_new, v_new  # 第一个 token
        else:
            self.K = concat([self.K, k_new], axis=token_axis)
            self.V = concat([self.V, v_new], axis=token_axis)
        return self.K, self.V  # 完整历史
```

加速效果巨大，对长生成可达 5 倍以上。但代价是：缓存占用 GPU 显存，且随每个 token 增长。每一层都保存自己的 K 和 V 张量。对于 13B 模型，大约**每个 token 占 1 MB**。4K token 上下文光缓存就烧掉 4 GB 显存。

这就是为什么长上下文让人感觉慢且贵。不是模型"脑力"不够，而是缓存空间不够。

应对方法很有创意：将缓存量化为 INT8 或 INT4；丢弃滑动窗口之外的 token；跨注意力头共享 K 和 V（分组查询注意力，GQA）；或者像操作系统分页内存一样分页管理缓存（PagedAttention，vLLM 背后的核心技巧）。

#### 前沿研究：从根本上缩小缓存

量化和分页把缓存当作固定成本来处理。DeepSeek 的 V4 系列（2025 年底预览）采取了更激进的路线：**重新设计注意力机制，使缓存从一开始就很小**。

他们的混合方案结合了两种压缩注意力变体——一种稀疏，一种稠密——都在高度压缩的 KV 流上操作。在百万 token 上下文下，V4-Pro 报告缓存大小约为前代的 10%，每 token 计算量约为前代的 27%。

重点不在具体架构，而在于 **KV 缓存已经成为整个领域围绕其优化模型设计的瓶颈**。当注意力机制本身都在被重新设计以最小化缓存时，你就知道约束条件已经转移了。

#### 量化：用精度换速度

训练需要精度。推理不需要。

大多数生产部署使用 FP16 或 BF16 而非 FP32，这将显存减半，并在 Tensor Core 上大致实现吞吐量翻倍。更激进的方案会走得更远，将权重量化到 INT8 甚至 INT4。

数学很直观。一个 7B 参数的模型需要：

| 精度 | 显存占用 |
|------|---------|
| FP32 | 28 GB |
| FP16 | 14 GB |
| INT8 | 7 GB |
| INT4 | 3.5 GB |

最后这个数字就是为什么你能在笔记本 GPU 上运行 7B 模型。GPTQ 和 AWQ 等方法为每个通道选择缩放因子，使有损压缩对质量的损害尽可能小。做得好的话，INT4 在大多数基准测试上能在原始模型的一个百分点以内。

#### 全流程总览

这是一个 Prompt 的完整旅程，端到端：

1. **分词**：文本变成整数 ID
2. **嵌入**：ID 变成向量，位置信息被注入
3. **预填充**：每一层对所有输入 token 并行处理。计算瓶颈。KV 缓存被填充。第一个输出 token 产生
4. **解码循环**：对每个新 token——为新 token 投影 Q，对缓存的 K 和 V 做注意力，运行前馈网络，采样。将新的 K 和 V 追加到缓存。显存带宽瓶颈
5. **反分词**：Token ID 被映射回字符，流式传输到你的屏幕

现代推理框架如 vLLM、TensorRT-LLM 和 Text Generation Inference 在这个循环外层包裹了**连续批处理**（多个用户的 token 在同一 GPU 步骤中交错执行）、**推测解码**（小模型起草 token，大模型验证）和精巧的内存管理。这就是单块 GPU 如何同时服务数十个并发用户的原理。

#### 这应该改变你的思考方式

一旦全貌清晰，几个实用的要点：

- **长 Prompt 在 TTFT 上昂贵，长输出在 ITL 上昂贵。** 它们压力在不同的地方。针对你的用户实际感受到的那个去优化。
- **上下文长度不是免费的。** 翻倍不仅仅是翻倍计算量——它膨胀 KV 缓存并挤压你的批处理大小。
- **量化是你手中性价比最高的旋钮。** 从 FP16 到 INT8 通常能将延迟减半，质量损失可忽略不计。
- **GPU 利用率可能具有误导性。** 一个在预填充时把 GPU 跑满的模型，在解码时可能只有 30% 利用率。解决方案不是更多算力，而是更快的显存或更小的缓存。

Transformer 架构获得了所有关注，但推理性能的生死取决于那些"无聊的东西"——内存布局、缓存管理、位宽。真正的艺术是从你手头的硬件中榨取最大性能。

现在，当有人告诉你他们的模型很慢时，你就知道该先问什么问题了：**是启动慢，还是流式输出慢？**
