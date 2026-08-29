# AI Learning State

> 用途：AI 长期学习存档。
>
> 规则：新会话开始时先读取本文件；不要重复已经掌握的内容；优先从“当前学习位置”继续。
>
> 最后更新：2026-08-30

---

# 1. 当前总路线

```text
AI 基础
  ↓
Transformer
  ↓
LLM 工作机制
  ↓
LLM 推理
  ↓
Context / KV Cache / Prefix Cache
  ↓
Batching / PagedAttention / Serving
  ↓
Agent
  ↓
AI Coding
  ↓
Skills / MCP / Memory / Routing
  ↓
AI Coding 架构
  ↓
模型服务与推理系统
  ↓
更高级：训练 / 微调 / 分布式推理
```

学习目标不是“背概念”，而是达到：

```text
能解释原理
→ 能画数据流
→ 能分析性能
→ 能看懂实现
→ 能自己实现简化版
→ 能把原理落到游戏客户端 / AI Coding 基础设施
```

---

# 2. 当前学习阶段

## 当前阶段

**LLM 推理与 Context Engineering**

正在从：

```text
Token / Attention
    ↓
KV Cache
    ↓
Prefix Cache
```

继续向：

```text
Continuous Batching
Chunked Prefill
PagedAttention
推理吞吐 / 延迟
Serving
```

推进。

---

# 3. 已掌握 / 已讨论

## 3.1 Token 顺序

已经形成的核心理解：

```text
Token 是模型实际处理的离散输入单位。

Token 顺序不是无关紧要的：
位置不同
→ Attention 关系不同
→ hidden states 不同
→ 后续计算结果不同
```

因此涉及缓存复用时，**prefix 的内容和顺序必须保持一致**。

---

## 3.2 KV Cache

已经理解：

在 autoregressive decode 中：

```text
已有上下文
     ↓
历史 token 的 K / V 已经计算
     ↓
后续生成时复用
     ↓
避免重复计算历史部分
```

核心价值：

```text
减少 decode 阶段对历史 token 的重复计算
→ 降低计算量
→ 提高生成效率
```

---

## 3.3 Prefix Cache

已经形成基本理解：

对于完全相同的 prefix：

```text
Request A
[固定前缀][用户输入]

Request B
[固定前缀][另一段输入]
```

固定前缀可以：

```text
第一次：
prefix
  ↓
prefill
  ↓
得到 prefix KV

后续：
直接复用 prefix KV
  ↓
只计算新增部分
```

核心收益：

```text
减少重复 prefill
→ 降低 GPU 计算占用
→ 提高并发条件下的有效吞吐
```

---

# 4. 当前已经形成的重要认知

## 4.1 KV Cache vs Prefix Cache

当前理解：

```text
KV Cache
=
保存已经计算过的 token 的 K/V

Prefix Cache
=
把“可复用的公共前缀对应 KV”
作为跨请求复用的数据
```

也就是说：

```text
KV Cache 是缓存机制本身
Prefix Cache 是 KV Cache 在跨请求 / 公共前缀复用场景下的一种利用方式
```

### 后续需要进一步严格化

需要继续确认：

- 单请求 KV Cache 生命周期
- Prefix Cache 生命周期
- prefix cache 的 block / page 管理
- cache 命中条件
- tokenizer / position / rope 等因素对复用的影响
- 不同推理框架中的具体实现差异

---

# 5. 目前尚未完全掌握

## 5.1 Continuous Batching

需要理解：

```text
传统 batch
=
等一批 request

Continuous Batching
=
请求动态加入 / 离开 batch
```

重点：

- 为什么对 LLM 特别重要
- decode 阶段为什么天然适合动态 batching
- batch 中不同 request 的 sequence length 如何处理
- scheduler 如何调度 token
- 吞吐与延迟如何权衡

---

## 5.2 Chunked Prefill

需要理解：

```text
长 prompt
    ↓
一次性 prefill
```

可能造成：

```text
GPU 长时间被 prefill 占用
→ decode request 被阻塞
→ 首 token 延迟 / 尾延迟变差
```

Chunked Prefill 的核心方向：

```text
长 prompt
↓
拆成多个 chunk
↓
与 decode 调度交错
```

重点理解：

- 为什么 chunk size 会影响性能
- prefill / decode 的计算特征差异
- scheduler 如何交错调度

---

## 5.3 PagedAttention

下一重点。

需要建立：

```text
连续 KV Cache
        ↓
容易产生内存碎片 / 预分配浪费
        ↓
分页管理
        ↓
KV block / page
        ↓
更高 GPU memory utilization
```

需要重点理解：

- 为什么 LLM KV Cache 特别适合分页
- block table
- logical sequence → physical KV blocks
- page reuse
- prefix cache 与 page/block 的关系

---

# 6. 下一阶段学习顺序

严格按这个顺序推进：

```text
① Continuous Batching
        ↓
② Prefill / Decode Scheduler
        ↓
③ Chunked Prefill
        ↓
④ PagedAttention
        ↓
⑤ KV Cache Block 管理
        ↓
⑥ Prefix Cache 的工程实现
        ↓
⑦ GPU Memory / KV Cache 容量估算
        ↓
⑧ Throughput / Latency 指标
        ↓
⑨ LLM Serving
```

---

# 7. 学习方法

每一个新概念都按照：

```text
1. 它是什么
2. 为什么需要它
3. 它解决什么瓶颈
4. 没有它会怎样
5. 数据如何流动
6. 内存如何变化
7. GPU 在算什么
8. Scheduler 在做什么
9. 一个简化实现
10. 和已有概念的关系
```

禁止只做名词解释。

---

# 8. 学习过程中重点避免的误区

## 误区 1

不要把：

```text
KV Cache
Prefix Cache
PagedAttention
Continuous Batching
```

理解成同一层次的技术。

它们解决的是不同问题：

```text
KV Cache
→ 避免重复计算历史 token

Prefix Cache
→ 跨请求复用公共 prefix

PagedAttention
→ 更高效地管理 KV Cache 显存

Continuous Batching
→ 提高多请求场景下 GPU 利用率 / 吞吐
```

---

## 误区 2

不要只从“算法”理解 LLM 推理。

真正的推理系统是：

```text
Request
  ↓
Tokenizer
  ↓
Scheduler
  ↓
Prefill
  ↓
KV Cache
  ↓
Decode
  ↓
Batching
  ↓
GPU Memory Management
  ↓
Sampling
  ↓
Response
```

后面要逐渐进入“系统工程视角”。

---

# 9. 与个人技术方向的连接

当前 AI 学习不是孤立学习。

需要最终落到：

```text
LLM
 ↓
AI Agent
 ↓
AI Coding
 ↓
OpenCode / Codex
 ↓
Skill
 ↓
MCP
 ↓
Model Router
 ↓
Scheduler
 ↓
Context / Memory
```

并与游戏客户端架构形成连接：

```text
Game Client Architecture
       +
AI Agent Architecture
       +
LLM Inference Architecture
```

最终目标是能设计：

```text
AI Coding Agent Runtime
```

并理解：

```text
Model Router
Context Manager
Memory
Tool Runtime
Scheduler
Agent Runtime
Model Serving
```

---

# 10. 学习状态记录格式

以后每次学习结束，在这里增加：

```markdown
## Session YYYY-MM-DD

### 本次主题
-

### 已掌握
-

### 新增理解
-

### 仍然不清楚
-

### 发现的误区
-

### 下一步
-
```

---

# 11. 下一课

**下一课：Continuous Batching**

目标不是记住定义，而是能够回答：

> 为什么 LLM serving 不能简单使用传统 batch？

并最终画出：

```text
Request A ─┐
Request B ─┼→ Scheduler → GPU
Request C ─┤
Request D ─┘
```

进一步理解：

```text
Prefill
Decode
Token Scheduling
Continuous Batching
```

四者之间的关系。

---

# 12. 给未来 ChatGPT 会话的启动指令

把下面这段直接贴到新会话：

> 我在进行长期 AI 系统学习。
>
> 请把下面的 AI Learning State 当作我的学习进度，不要重复已经掌握的基础内容。
>
> 教学要求：
> 1. 从当前进度继续；
> 2. 不只讲定义，要讲原理、数据流、GPU 计算、显存、调度和工程实现；
> 3. 每个概念都要说明“为什么需要”以及“解决什么瓶颈”；
> 4. 主动指出我的理解中不严谨或错误的地方；
> 5. 适当使用简化实现、伪代码和架构图；
> 6. 当前优先继续 Continuous Batching → Chunked Prefill → PagedAttention；
> 7. 每完成一个主题，更新 Learning State，便于下一次会话继续。
>
> 以下是当前学习档案：
>
> [粘贴本文件内容]
