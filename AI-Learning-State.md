# AI Learning State

> 用途：AI 长期学习存档。
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

**LLM 推理与 Context Engineering / Serving**

已完成：

```text
Token / Attention
  ↓
KV Cache
  ↓
Prefix Cache
  ↓
Continuous Batching
  ↓
Prefill / Decode Scheduler
  ↓
Chunked Prefill
  ↓
Paged KV Cache / PagedAttention
  ↓
KV Block / Block Size
  ↓
Admission Control
  ↓
KV Cache Eviction / Offloading / Preemption
  ↓
Prefill / Decode Disaggregation
```

当前继续进入：

```text
KV Cache Transfer
→ GPU Interconnect / Networking
→ TTFT / ITL / Throughput 定量分析
→ LLM Serving 架构
```

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

## 3.4 Continuous Batching

已经形成核心理解：

```text
Static / 传统 Batch
=
多个 Request 进入固定 Batch
→ 通常按锁步推进
→ 已完成 Request 不能高效地立刻让出位置
```

Continuous Batching：

```text
Request 动态加入
Request 动态继续
Request 动态完成
Request 动态离开
Request 甚至可以被 Preempt 后恢复
```

核心不是“Batch 更大”，而是：

> **Batch 成员可以持续动态变化，Scheduler 每一轮重新决定当前 GPU 应该处理哪些 Request。**

已理解的原因：

```text
LLM Decode 是逐步 autoregressive 的
→ 每轮每个 Request 推进少量 token
→ 不同 Request 的输出长度不同
→ 长请求可能持续很久
→ Static Batch 容易被长请求拖住
→ 短请求结束后 GPU / batch slot 利用率下降
```

仍需细化：

- token-level scheduling
- batch 中不同 sequence length 的实际张量组织
- scheduler policy 的具体实现

---

## 3.5 Chunked Prefill

已经理解：

```text
长 Prompt
    ↓
一次性完整 Prefill
    ↓
长时间占用 GPU 计算
    ↓
正在 Decode 的 Request 被阻塞
    ↓
ITL / tail latency 变差
```

Chunked Prefill：

```text
长 Prompt
↓
拆成多个 chunk
↓
与 Decode 交错调度
```

已经理解 Chunk Size 的 Trade-off：

```text
Chunk 太大
→ 单次 Prefill 工作量大
→ Decode 被阻塞时间长
→ ITL / 延迟变差

Chunk 太小
→ 调度次数 / kernel launch / 管理开销增加
→ GPU 工作粒度过碎
→ Kernel efficiency 下降
```

关键认知：

> **Chunk Size 是计算调度粒度，不等同于 KV Cache Block Size。**

---

## 3.6 Paged KV Cache / PagedAttention

已经理解核心动机：

```text
每个 Request 的 KV Cache
→ 长度不同
→ 动态增长
→ 动态释放
→ 生命周期不同
```

如果要求连续显存：

```text
容易产生外部碎片
→ 总空闲显存很多
→ 但连续空间不够
→ 扩容可能需要搬迁
```

Paged KV Cache：

```text
Logical KV Cache
      ↓
固定大小 KV Block
      ↓
Block Table
      ↓
Physical KV Blocks
```

已经理解：

```text
Logical Block 0 → Physical Block 17
Logical Block 1 → Physical Block 3
Logical Block 2 → Physical Block 42
```

所以逻辑上连续、物理上可以不连续。

核心收益：

```text
动态申请 / 释放固定 Block
→ 不要求大块连续显存
→ 降低外部碎片问题
→ 更适合多 Request 动态 Serving
```

---

## 3.7 KV Block 与 Block Size

已经明确区分：

```text
Chunk Size
→ Prefill 的计算 / 调度粒度

KV Block Size
→ KV Cache 的显存管理粒度
```

例如：

```text
KV Block = 16 tokens
A = 33 tokens
→ ceil(33 / 16) = 3 Blocks
```

已经理解内部碎片，以及 Block Size 在碎片、元数据和访问效率之间的 Trade-off。

特别注意：

> **未使用的 slot 仍属于该 Request 的最后一个 KV Block，通常不能简单切给另一个 Request。**

系统可以记录 `valid_tokens` 表示最后一个 Block 中真正有效的 token 数。

---

## 3.8 Prefix Cache 与 Block 引用计数

已经理解：

```text
Request 生命周期
≠
KV Cache 生命周期
```

公共 Prefix KV 可以在 Request 完成后继续作为 Cache 存活。

```text
Request A → ref
Prefix Cache → ref
```

只有：

```text
ref_count = 0
```

才真正具备进入 Free Pool 的资格。

---

## 3.9 Admission Control

已经开始建立 Scheduler 的资源视角：

当新 Request D 到来时，不能只看：

```text
D Prefill 当前需要多少显存
```

还要考虑：

```text
Prefix Cache 命中多少
+
新增 KV Blocks 需要多少
+
当前 Running Requests 后续 Decode 还会增长多少
+
GPU KV Pool 当前剩余多少
```

因此：

```text
Request D
  ↓
检查 Prefix Cache
  ↓
计算实际新增 KV
  ↓
评估当前 KV Capacity
  ↓
考虑 Running Requests 的持续 Decode 增长
  ↓
决定 Admit / Wait / Preempt 等
```

核心认知：

> **不能把“Prefill 时新增 KV 的容量需求”误认为 Request 的全部未来显存需求。**

---

## 3.10 KV Cache Eviction / Offloading / Preemption

已经理解三者需要区分。

### Finished Request

```text
Request 结束
↓
Request 本身释放
↓
其 KV 可以：
  - 直接释放
  - 转成 Prefix Cache
  - 继续作为 Cached KV 存活
```

### Eviction

```text
GPU Cached KV
↓
从 GPU 删除 / 驱逐
↓
以后需要时重新 Prefill / 重建
```

### Offloading

```text
GPU KV
↓
CPU Memory
↓
暂时释放 GPU 显存
↓
未来恢复时再搬回 GPU
```

### Preemption

```text
Running Request
↓
暂时暂停
↓
释放其 GPU KV Blocks
↓
Request 逻辑状态保留
↓
未来 Resume
```

Preemption 的两种思路：

```text
1. Recompute
   丢掉 KV
   Resume 时重新 Prefill
   → Compute ↑

2. Offload
   KV 保留在 CPU
   Resume 时搬回 GPU
   → Transfer / CPU Memory ↑
```

重要区别：

```text
Finished
≠
Preempted
```

并且：

```text
Request State
≠
KV State
```

---

## 3.11 Prefill / Decode Disaggregation

已形成核心理解：

传统 Aggregated Serving：

```text
同一 GPU
├─ Prefill
└─ Decode
```

Disaggregated Serving：

```text
Request
  ↓
Router
  ├─→ Prefill Pool
  │      ↓
  │   KV Generation
  │      ↓
  │   KV Transfer
  │
  └─→ Decode Pool
         ↓
       Decode
```

### P/D 为什么可以分离

Prefill 与 Decode 的资源特征不同：

```text
Prefill
→ 大量输入 token
→ 更偏 Compute Intensive

Decode
→ 每轮少量新 token
→ 多 Request 并发
→ 对 ITL / tail latency 更敏感
```

因此可以独立配置 P/D Pool 的规模和并行策略。

### P/D 不均衡

已经理解这是一个 Pipeline Balance 问题：

```text
P 很慢
→ KV 尚未 Ready
→ D 即使有空闲资源
→ 也无法对该 Request 开始 Decode
```

所以 P/D 分离把原本 GPU 内部的资源竞争转化为：

```text
Producer（Prefill）
        ↓
Intermediate Result（KV）
        ↓
Consumer（Decode）
```

### KV Transfer 成为新的关键路径

第一次输出的路径可以抽象为：

```text
Queue
→ Prefill
→ KV Transfer
→ Decode Admission
→ First Decode
→ First Token
```

因此：

```text
TTFT
≈ Queue + Prefill + KV Transfer + Decode Admission + First Decode
```

已经理解 Critical Path 思维：如果 KV Transfer 占据主要时间，继续单独优化 Prefill 的边际收益会明显下降。

### Block-level KV Transfer

已经把 Paged KV / KV Block 与 P/D 分离连接起来：

```text
P 产生 KV Blocks
        ↓
Block-level Transfer
        ↓
D 的 KV Block Pool
```

KV Block 可以抽象出：

```text
PRODUCED
  ↓
TRANSFERRING
  ↓
READY
  ↓
ACTIVE
  ↓
RELEASED / EVICTED
```

已经认识到：

> “KV 局部 Transfer”不等于“Decode 可以无条件立即开始”。Decode 所依赖的历史 KV Block 必须在对应计算需要前 Ready。

### Router / Cache-aware Routing

Router 不能只看 GPU utilization，还应考虑：

```text
KV Capacity
KV Transfer Distance
Current Decode Batch
Prefix Cache Locality
Priority / SLA
Expected TTFT / ITL
```

初步 Cost Model：

```text
Cost(P, D, Request)
≈ QueueWait(P)
 + PrefillTime(P)
 + KVTransferTime(P→D)
 + DecodeQueueWait(D)
 + ExpectedITL(D)
```

已经形成：

> **Cache-aware Routing**：Prefix Cache 命中位置本身就是路由决策的一部分。

---

# 4. 当前最重要的系统模型

```text
                         Request
                            │
                            ▼
                         Router
                            │
                  ┌─────────┴─────────┐
                  ▼                   ▼
             Prefill Pool        Decode Pool
                  │                   ▲
                  ▼                   │
             KV Generation            │
                  │                   │
                  └── KV Transfer ────┘
                            │
                            ▼
                          Decode
```

旁路机制：

```text
Prefix Cache
    ↓
Cache-aware Routing

KV Memory Pressure
    ↓
Admission / Eviction / Preemption

Long Prefill
    ↓
Chunked Prefill
```

核心理解已经从“模型怎么跑”升级到：

> **LLM Serving 本质上是在 GPU Compute、GPU KV Memory、CPU Memory、KV Transfer、Latency、Throughput 之间持续做资源调度。**

---

# 5. 已发现并纠正的关键误区

## 误区 1：Continuous Batching = 大 Batch

不准确。

```text
Batch 成员动态变化
→ Scheduler 持续调度
```

## 误区 2：Chunk Size = KV Block Size

不准确。

```text
Chunk Size
→ 计算 / 调度粒度

KV Block Size
→ KV 显存管理粒度
```

## 误区 3：Request Finished 就意味着 KV 必须释放

不准确。

```text
Request Finished
→ 可以不再被运行时引用
→ Prefix Cache 可能继续引用其 Block
```

## 误区 4：Prefill KV 容量 = Request 的全部 KV 显存需求

不准确。

```text
Decode 持续生成
→ KV Cache 持续增长
```

## 误区 5：KV Transfer 局部完成就意味着 Decode 可以无条件开始

不准确。

```text
必须保证当前 Decode 所依赖的 KV Blocks 已经 READY
```

## 误区 6：GPU Load 最低的 Worker 就一定是最佳路由目标

不准确。

还需要考虑：

```text
Queue
+ Cache Locality
+ KV Transfer
+ KV Capacity
+ Decode Batch
+ SLA
```

## 误区 7：Memory Bandwidth 决定 KV Cache Size

不准确。

严格区分：

```text
Compute
→ 计算耗时

Memory Capacity
→ 能容纳多少 KV Cache

Memory Bandwidth
→ KV / 权重数据搬运吞吐
```

KV Cache Size 主要由 token 数、层数、KV heads、head dimension、数据类型等决定。

---

# 6. 当前学习顺序

已完成 / 正在深化：

```text
① Continuous Batching                ✅ 已形成核心理解
        ↓
② Prefill / Decode Scheduler          🟡 初步理解，仍需深入实现
        ↓
③ Chunked Prefill                    ✅ 已形成核心理解
        ↓
④ PagedAttention / Paged KV Cache    ✅ 已形成核心理解
        ↓
⑤ KV Cache Block / Block Size        ✅ 已形成核心理解
        ↓
⑥ Prefix Cache 工程实现               ✅ 已形成基本理解
        ↓
⑦ Admission Control                  ✅ 已形成基本理解
        ↓
⑧ KV Cache Eviction / Offloading     ✅ 已形成基本理解
        ↓
⑨ KV Cache Preemption                ✅ 已形成基本理解
        ↓
⑩ Prefill / Decode Disaggregation    ✅ 已形成核心理解
        ↓
⑪ KV Cache Transfer                  ⏳ 下一环
        ↓
⑫ TTFT / ITL / Throughput            ⏳ 后续定量深化
        ↓
⑬ LLM Serving                        ⏳ 后续
```

---

# 7. Session 2026-08-30

## 本次主题

```text
Continuous Batching
→ Chunked Prefill
→ Paged KV Cache / PagedAttention
→ KV Block / Block Size
→ Prefix Cache 引用计数
→ Admission Control
→ Eviction / Offloading / Preemption
→ Prefill / Decode Disaggregation
```

## 本轮新增理解

- P/D Disaggregation 把 Prefill 与 Decode 的资源竞争转化为 Producer → KV → Consumer 的流水线依赖。
- P/D 不均衡本质上是 Pipeline Balance 问题；P 很慢时，D 即使有资源也可能因为 KV 未 Ready 而无法 Decode。
- KV Transfer 进入 TTFT 的关键路径，不能只优化 Prefill 而忽略 Transfer。
- KV 可以以 Block 为粒度进行 Transfer / Ready 状态管理，但 Decode 只能在其当前依赖的 KV Blocks Ready 后执行。
- Router 不应只看 GPU Load，而应综合 Queue、Prefill、KV Transfer、Decode Queue、KV Capacity、Prefix Cache Locality、Expected ITL/TTFT 等因素。
- Cache-aware Routing 是 P/D Serving 中的重要调度维度。

## 本轮纠正

- Memory Bandwidth 不是 KV Cache Size 的直接决定因素；需要区分 Compute、Memory Capacity、Memory Bandwidth。
- “局部 KV Transfer”不等于“Decode 可以无条件提前执行”；必须保证当前 Attention 所需 KV 已 Ready。

## 下一步

```text
KV Cache Transfer
→ GPU → GPU / GPU → CPU → GPU / GPU → NIC → RDMA → NIC → GPU
→ PCIe / NVLink / RDMA / InfiniBand / RoCE
→ KV Connector
→ Transfer Bandwidth / Latency 估算
→ TTFT / ITL / Throughput
```

---

# 8. 学习方法

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

# 10. 给未来 ChatGPT 会话的启动指令

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
> 6. 当前从 KV Cache Transfer 继续，并持续连接 TTFT / ITL / Throughput；
> 7. 每完成一个主题，更新 Learning State，便于下一次会话继续。
>
> 以下是当前学习档案：
>
> [粘贴本文件内容]
