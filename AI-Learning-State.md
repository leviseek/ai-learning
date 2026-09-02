# AI Learning State

> 用途：AI 长期学习存档。
> 规则：新会话开始时先读取本文件；不要重复已经掌握的内容；优先从“当前学习位置”继续。
>
> 最后更新：2026-09-02

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

已完成并形成系统理解：

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
  ↓
KV Cache Transfer
  ↓
Scheduler Policy 基础
```

当前学习位置：

```text
Runnable Queue
    ↓
Priority / Scheduling Score
    ↓
Token Budget
    ↓
Continuous Batching 的实际调度
    ↓
Preemption Policy
    ↓
KV Block Manager 协同
    ↓
TTFT / ITL / Throughput 定量分析
    ↓
LLM Serving 架构
```

---

# 3. 已掌握 / 已讨论

## 3.1 Token 顺序

```text
Token 是模型实际处理的离散输入单位。

Token 顺序不是无关紧要的：
位置不同
→ Attention 关系不同
→ hidden states 不同
→ 后续计算结果不同
```

涉及缓存复用时，prefix 的内容和顺序必须保持一致。

---

## 3.2 KV Cache

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

对于完全相同的 prefix：

```text
Request A
[固定前缀][用户输入]

Request B
[固定前缀][另一段输入]
```

第一次计算 prefix KV，后续请求可以复用，只计算新增部分。

核心收益：

```text
减少重复 prefill
→ 降低 GPU 计算占用
→ 提高并发条件下的有效吞吐
```

Prefix Cache 更准确的理解是“可复用的 KV Block”，而不是另一套独立的 KV 数据。

```text
                 Shared Prefix Blocks
              ┌───────────────┐
              │ B1 B2 B3 B4   │
              └───────┬───────┘
                  ┌───┴───┐
                  ▼       ▼
                 R1       R2
```

多个 Request 可以共享同一组 Prefix KV Blocks，核心机制是引用而不是复制。

```text
Active KV
→ 正在被运行请求引用

Cached KV
→ 当前没有运行请求使用，但保留用于未来 Prefix 命中

Free Block
→ 已无引用、可重新分配
```

重要认知：

> Prefix Cache 是 KV Block 的一种复用 / 生命周期管理策略，而不是完全独立于 KV Cache 的第二套数据。

---

## 3.4 Continuous Batching

传统 Static Batch：

```text
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
Request 可以被 Preempt 后恢复
```

核心不是“Batch 更大”，而是：

> Batch 成员可以持续动态变化，Scheduler 每一轮重新决定当前 GPU 应该处理哪些 Request。

由于 Decode 是逐步 autoregressive 的，不同 Request 最终输出长度不同，因此不能假设所有 Request 会同时结束。

仍可继续细化：

- token-level scheduling
- batch 中不同 sequence length 的实际张量组织
- scheduler policy 的具体实现

---

## 3.5 Scheduler 的三层判断

已经建立：

```text
Admissible
→ 系统资源是否允许接纳这个 Request？

Ready
→ Request 当前依赖是否已经满足，例如 KV 是否可用？

Runnable
→ 即使 Ready，本轮是否应该真正进入 GPU 执行？
```

核心关系：

> Ready 不等于 Runnable。

当前进一步形成：

```text
Runnable
需要同时满足：
KV Ready
&&
Compute Capacity
&&
KV Memory Capacity
```

所以 Scheduler 不是简单的 GPU 空闲 → 找任务，而是：

```text
任务是否满足执行条件
        ↓
满足条件的任务进入 Runnable Queue
        ↓
Scheduler 再从 Runnable Queue 中选任务
        ↓
分配 GPU / Batch / KV / Network 等资源
```

---

## 3.6 Chunked Prefill

```text
长 Prompt
    ↓
一次性完整 Prefill
    ↓
长时间占用 GPU 计算
    ↓
正在 Decode 的 Request 被阻塞
```

Chunked Prefill：

```text
长 Prompt
↓
拆成多个 chunk
↓
与 Decode 交错调度
```

Chunk Size Trade-off：

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

> Chunk Size 是计算调度粒度，不等同于 KV Cache Block Size。

---

## 3.7 Paged KV Cache / PagedAttention

每个 Request 的 KV Cache：

```text
长度不同
→ 动态增长
→ 动态释放
→ 生命周期不同
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

例如：

```text
Logical Block 0 → Physical Block 17
Logical Block 1 → Physical Block 3
Logical Block 2 → Physical Block 42
```

因此逻辑上连续、物理上可以不连续。

核心收益：

```text
动态申请 / 释放固定 Block
→ 不要求大块连续显存
→ 降低外部碎片问题
→ 更适合多 Request 动态 Serving
```

---

## 3.8 KV Block 与 Block Size

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

> 未使用的 slot 仍属于该 Request 的最后一个 KV Block，通常不能简单切给另一个 Request。

系统可以记录 `valid_tokens` 表示最后一个 Block 中真正有效的 token 数。

---

## 3.9 Prefix Cache 与 Block 引用计数

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

更精确的工程模型：

```text
Request Reference
      ↓
KV Block Reference
      ↓
Shared Prefix Cache
```

共享 Prefix 时优先通过 Block 引用关系复用，而不是复制整份 KV。

---

## 3.10 Admission Control

新 Request 不能只看当前 Prefill 的显存需求，还要考虑：

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
Request
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

> 不能把“Prefill 时新增 KV 的容量需求”误认为 Request 的全部未来显存需求。

进一步形成 Reserved Capacity 思维：

```text
Admissible Capacity
≈
Capacity
- Active KV
- Protected / Cached KV
- Reserved KV Growth
```

实际系统不会机械地预留全部未来 token，但 Scheduler 必须考虑未来 Decode 增长，而不是只看瞬时 free memory。

并进一步区分：

```text
Free
→ 当前没有任何引用，可以直接分配

Cached / Evictable
→ 当前保存数据，但在满足策略条件后可淘汰

In-use
→ 被活跃 Request 引用，不应直接淘汰
```

---

## 3.11 KV Cache Eviction / Offloading / Preemption

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

Preemption 两种主要思路：

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

已经认识到 Preemption 是资源交换问题：

```text
GPU Memory ↓
↔
Compute ↑ 或 Transfer ↑
```

---

## 3.12 Prefill / Decode Disaggregation

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

Prefill 更偏大量输入 token / Compute Intensive；Decode 是小步 autoregressive，并发高，对 ITL / tail latency 更敏感，因此可以独立配置 P/D Pool。

---

## 3.13 P/D Pipeline Balance

已经明确认识到：

```text
P 很慢
→ KV 尚未 Ready
→ D 即使有空闲资源
→ 也无法对该 Request 开始 Decode
```

反过来：

```text
P 很快
→ KV 产生很快
→ KV Transfer 堆积
→ D 拿不到可执行的 KV
→ D GPU 空等
```

因此 P/D Serving 不应只看 GPU 算力，还要看：

```text
Pipeline Throughput
≈
min(
    Prefill Throughput,
    KV Transfer Throughput,
    Decode Throughput
)
```

P/D 分离后，本质上形成：

```text
Producer（Prefill）
        ↓
Intermediate Result（KV）
        ↓
Transport（KV Transfer）
        ↓
Consumer（Decode）
```

关键认知：

> 局部 GPU 很强，不代表整体 Serving 吞吐高；流水线中任一段都可能成为瓶颈。

---

## 3.14 KV Transfer

已经建立 KV Transfer 的 Scheduler 视角：

```text
Prefill Done
     ↓
KV Ready
     ↓
Transfering
     ↓
KV Arrived on D
     ↓
Runnable
```

如果 KV Transfer 网络慢：

```text
P GPU 很强
D GPU 很强
Network 很慢
        ↓
KV 在 Transfer Queue 堆积
        ↓
D GPU 缺少可执行 KV
        ↓
D GPU 利用率下降
```

因此 KV Transfer 是 P/D 解耦后的独立资源与瓶颈，Scheduler 需要同时观察：

```text
GPU Compute
+
KV Memory
+
Network / Transfer
+
Request State
```

---

# 4. 当前 Scheduler Policy 理解

已经开始自主设计 Scheduler，而不是只背概念。

## 4.1 Request State Machine

```text
WAITING
  ↓
PREFILLING
  ↓
KV_READY
  ↓
TRANSFERRING
  ↓
RUNNABLE
  ↓
DECODING
  ↓
FINISHED
```

重要区分：

```text
Finished
→ 资源回收事件

Runnable
→ 调度竞争集合
```

Finished Request 不应该继续参与下一轮 Runnable 竞争；应先回收其 Request / GPU / KV 资源，再重新调度。

---

## 4.2 Decode 不能提前知道最终输出长度

用户已经识别：

> 每个 Request 最终 Decode 多少 token 通常事先未知，因此不能依赖一次性预测完整服务时间。

更合理的是增量调度：

```text
每轮 Decode 推进少量 token
→ 重新检查 finished / priority / waiting / resource
→ Continuous Batching 动态加入和退出 Request
```

---

## 4.3 公平性拆分

本轮进一步区分两个层面：

### Request Fairness

```text
更早到达的 Request
→ 基础优先级更高
```

当前倾向：Arrival Order / FIFO。

### Service Fairness

```text
长期占用 Decode 资源的 Request
→ 不能无限霸占执行机会
```

当前形成的策略：

```text
基础规则：Arrival Order / FIFO 倾向
+
等待时间超过阈值 → Starvation Protection / Priority Boost
+
长期占用资源 → 考虑 Service-Time / Token Budget 修正
```

重要：后到的 Request 不会因为“等了一会儿”就天然获得插队权；只有达到明确的 starvation threshold 或其它策略条件后，才可能提升优先级。

---

## 4.4 当前自主推理出的调度流程

面对 100 个 Runnable Request，当前倾向：

```text
100 个 Request
    ↓
① 检查 Finished
    ↓
Finished → 先释放 GPU / KV / Request 资源
    ↓
② 检查真正 Runnable 的 Request
    ↓
优先考虑已经完成 Prefill 且 KV Transfer 已落地 D GPU 的 Request
    ↓
③ 在 Runnable 集合中按 Request 到达时间选择
    ↓
④ 若某个 Request 等待超过 starvation threshold
   → Priority Boost
    ↓
⑤ 如果长期占用 Decode 资源
   → 考虑 Token Budget / Service Fairness
```

这里的核心思维已经从：

```text
GPU 空闲 → 找任务
```

提升到了：

```text
依赖满足
→ 资源可用
→ Runnable
→ 再进行调度竞争
```

---

# 5. 当前尚未完成 / 下一学习位置

## 5.1 Runnable Queue 的实际排序模型

下一步需要解决：

```text
多个 Runnable Request
↓
Priority 怎么计算？
```

可能需要组合：

```text
Arrival Time
+
Waiting Time
+
Starvation Protection
+
Service Time / Generated Tokens
+
Request Priority
+
Resource Affinity
```

注意：以上是设计维度，不是固定标准公式。

---

## 5.2 Token Budget

核心问题：

```text
R1 已 Decode 200 tokens
R21 刚进入 Runnable
```

不能简单使用：

```text
R21 等得久 → 一定抢占 R1
```

也不能：

```text
R1 先来 → 永远继续
```

需要引入：

```text
每轮 / 每次调度允许 Request 消耗多少 Decode Budget
```

再研究 Budget 与 Batch Size、ITL、Throughput、Fairness 的关系。

---

## 5.3 Preemption Policy

核心问题：

> 什么条件下应该暂停一个正在 Decode 的 Request？

需要综合：

```text
Request Priority
+
Waiting Time
+
Generated Tokens
+
KV Memory Pressure
+
Resume Cost
+
当前系统目标（TTFT / ITL / Throughput）
```

---

## 5.4 KV Block Manager 协同

下一阶段重点理解：

```text
Scheduler
   ↕
KV Block Manager
   ↕
GPU KV Pool
```

尤其研究：

```text
Free Block
Cached / Evictable Block
In-use Block
Reserved Block
```

以及：

```text
Preempt
→ KV Block 怎么处理？

Resume
→ Block 怎么恢复？

Offload
→ CPU / GPU Block 状态如何同步？

Recompute
→ 重新 Prefill 后如何重新建立 Block Table？
```

---

## 5.5 关键待解决场景

当前下一道核心推理题：

```text
D GPU 显存不足
+
存在一个已经严重等待超时的 Request
+
该 Request 的 KV 已经 Transfer 完成
```

需要自己设计：

```text
Eviction
→ 不够时先淘汰 Cached KV？

Preemption
→ 是否暂停已有 Decode Request？

Offloading
→ 是否把某些 KV 搬到 CPU？

Recomputation
→ 是否宁可丢 KV，未来重新 Prefill？
```

最终目标：

> 让一个“逻辑上 Ready 但物理资源不足”的 Request 重新获得 Runnable 条件，并分析为此付出的 Compute / Transfer / Latency 成本。

---

# 6. 当前能力等级

当前已经从：

```text
理解 LLM Serving 概念
```

进入：

> **能够从依赖关系、资源约束、Request 状态机和调度策略角度，自主设计简化版 Inference Scheduler。**

当前最明显的能力提升：

```text
能够主动发现瓶颈传播关系
能够区分 Finished / Ready / Runnable
能够认识到 Scheduler 与 KV Memory Manager 耦合
能够主动考虑 FIFO / Starvation / Service Fairness
能够从 Request 生命周期推导调度策略
```

---

# 7. Session Records

近期学习记录：

```text
2026-08-31
→ Prefill / Decode Disaggregation
→ KV Transfer
→ Scheduler 基础

2026-09-02
→ KV Transfer 作为独立瓶颈
→ Request State Machine
→ Admissible / Ready / Runnable
→ Continuous Batching 增量调度
→ Request Fairness / Service Fairness
→ Starvation Protection
→ Preemption 与 KV Memory
→ Scheduler Policy 自主推理
```

详细 session：

```text
sessions/2026-09-02-inference-scheduler.md
```

---

# 8. 下一次开始学习时

不要重复：

```text
KV Cache 基础
Prefix Cache
Paged KV Cache
Chunked Prefill
Admission Control
Eviction / Offloading 基础
P/D Disaggregation 基础
KV Transfer 基础
```

直接从：

```text
“D GPU 显存不足，但存在一个严重等待超时、KV 已 Transfer 完成的 Request，Scheduler 如何处理？”
```

开始，然后进入：

```text
Runnable Queue
→ Scheduling Score
→ Token Budget
→ Preemption Policy
→ KV Block Manager
→ TTFT / ITL / Throughput 定量分析
→ LLM Serving 架构
```
