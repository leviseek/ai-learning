# AI Learning Session — 2026-09-02

## 当前学习阶段

**LLM 推理与 Context Engineering / Serving → Prefill / Decode Disaggregation → Scheduler Policy**

## 本轮已掌握

### 1. KV Transfer 是 P/D 解耦中的独立瓶颈

```text
Prefill → KV Cache → KV Transfer → Decode
```

即使 P GPU 和 D GPU 单独算力都很强，只要 KV Transfer 网络吞吐/延迟成为瓶颈：

```text
P 很快
→ KV 产生很快
→ Transfer 堆积
→ D 拿不到可执行的 KV
→ D GPU 空等
```

因此 P/D Serving 的系统吞吐不能只看 GPU 算力，还要看：

```text
min(Prefill Throughput,
    KV Transfer Throughput,
    Decode Throughput)
```

这进一步建立了 Pipeline Balance 思维。

### 2. Scheduler 不只是 GPU 分配器

Scheduler 需要协调：

```text
Request State
+ GPU Compute
+ KV Cache
+ KV Transfer
+ GPU KV Memory
+ Execution Order
```

Request 是否能执行，需要区分：

```text
Admissible
→ 资源是否允许接纳？

Ready
→ 所有执行前置条件是否满足？

Runnable
→ Ready 之后，本轮是否真正获得 GPU 执行机会？
```

核心认知：

> Ready 不等于 Runnable。

### 3. Request State Machine 视角

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

其中：

```text
Finished → 资源回收事件
Runnable → 调度竞争集合
```

Finished Request 不应该继续参与下一轮 Runnable 竞争；应先回收其 Request / GPU / KV 资源，再重新调度。

### 4. Decode 调度不能预知最终输出长度

Request 最终 Decode 多少 token 通常事先未知，因此不应该依赖一次性预测完整服务时间。

更合理的思路是增量调度：

```text
每轮 Decode 推进少量 token
→ 重新检查 finished / priority / waiting / resource
→ Continuous Batching 动态加入和退出 Request
```

### 5. 公平性的正确拆分

本轮进一步区分：

**Request Fairness**

```text
更早到达的 Request
→ 基础优先级更高
```

**Service Fairness**

```text
长期占用 Decode 资源的 Request
→ 不能无限霸占执行机会
```

用户当前形成的策略：

```text
基础规则：Arrival Order / FIFO 倾向
+
等待时间超过阈值 → Starvation Protection / Priority Boost
+
长期占用资源 → 考虑 Service-Time / Token Budget 修正
```

重要：后到的 Request 并不会因为“等待了一会儿”就天然拥有插队权；只有达到定义好的 starvation threshold 等策略条件后，才可能提升优先级。

### 6. KV Cache 与 Preemption 的关系

暂停一个正在 Decode 的 Request 并不只是：

```text
Request → PAUSED
```

还必须处理它占用的 GPU KV Cache：

```text
Preemption
├─ Recompute
│  └─ 丢掉 KV，恢复时重新 Prefill
└─ Offload
   └─ KV 搬到 CPU，恢复时再 Transfer 回 GPU
```

因此 Preemption 本质是资源交换：

```text
GPU Memory ↓
↔ Compute ↑ 或 Transfer ↑
```

### 7. Scheduler 与 KV Memory Manager 强耦合

即使：

```text
KV Transfer 已完成
D GPU Compute 有空
```

如果 D GPU 上没有足够的 KV Memory：

```text
Request 仍然不可 Runnable
```

因此：

```text
Runnable ≠ Compute Ready only

Runnable 还需要：
KV Ready
&&
Compute Capacity
&&
KV Memory Capacity
```

---

## 本轮用户自主推理结果

用户能够独立识别：

```text
100 个 Runnable Request
→ 先清理 Finished
→ 优先检查已经完成 Prefill + KV Transfer 的 Request
→ 在可运行集合中继续按 Request 到达时间选择
→ 对等待超时的 Request 再考虑优先级提升
```

并进一步认识到：

```text
公平不是简单的“谁等得久谁先”
而是 Arrival Order + Starvation Protection + Service Fairness 的组合
```

---

## 当前能力等级判断

已从“理解 LLM Serving 概念”进入：

> **能够从依赖关系、资源约束、状态机和调度策略角度自主设计简化版 Inference Scheduler。**

下一步不再重复 KV Cache 基础。

## 下一学习位置

进入：

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
```

重点问题：

> **当 D GPU 显存不足，但存在一个已经严重等待超时的 Request 时，Scheduler 应该如何组合 Eviction / Preemption / Offloading / Recomputation，最终把它变成 Runnable？**

之后继续进入 TTFT / ITL / Throughput 的定量分析与 LLM Serving 架构。
