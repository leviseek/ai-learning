# AI Learning Session — 2026-08-30

## 本轮主题

**Prefill / Decode Disaggregation（P/D 分离）**

从上一轮的 `Continuous Batching → Chunked Prefill → Admission Control → Preemption` 继续，进入 LLM Serving 的跨 GPU 资源架构。

## 已掌握

### 1. Aggregated vs Disaggregated

传统 Aggregated Serving：

```text
同一 GPU
├─ Prefill
└─ Decode
```

P/D Disaggregation：

```text
Request
  ↓
Router
  ├─→ Prefill Pool
  │      ↓
  │   KV Cache
  │      ↓
  │   KV Transfer
  │
  └─→ Decode Pool
         ↓
       Decode
```

核心理解：

> P/D 分离不是单纯“把 GPU 拆成两组”，而是把原本 GPU 内部 Prefill/Decode 的资源竞争转化为 Producer（Prefill）→ Intermediate Result（KV）→ Consumer（Decode）的流水线依赖。

### 2. P 与 D 的资源特征不同

```text
Prefill
→ 大量输入 token
→ Compute Intensive

Decode
→ 每轮少量新 token
→ 多 Request 并发
→ 对 ITL / tail latency 更敏感
```

因此 P/D 可以独立扩容与配比，而不是简单追求 GPU 数量相等。

### 3. KV Transfer 成为新的关键路径

完整路径：

```text
Request
→ Prefill
→ KV 生成
→ KV Transfer
→ Decode Admission
→ First Decode
→ First Token
```

因此 TTFT 不再只由 Prefill 决定，还受到 KV Transfer 与 Decode 端排队/调度影响。

已经形成 Critical Path 意识：

```text
TTFT ≈ Queue + Prefill + KV Transfer + Decode Admission + First Decode
```

因此当 KV Transfer 占据主要时间时，继续单独优化 Prefill 的边际收益会明显下降。

### 4. P/D 不均衡本质是 Pipeline Balance 问题

例如：

```text
P 很慢
→ KV 尚未 Ready
→ D 即使还有大量空闲计算资源
→ 也无法对该 Request Decode
```

所以 P/D 之间存在明显的生产者-消费者依赖。

### 5. KV 可以按 Block 进行异步传输

已经把上一阶段的 Paged KV Cache / KV Block 思维连接到 P/D 分离：

```text
P 产生 KV Blocks
        ↓
Block-level Transfer
        ↓
D 的 KV Block Pool
```

可以存在类似状态：

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

> “KV 局部 Transfer”不等于“Decode 可以无条件立即开始”。Decode 所需的历史 KV Block 必须在对应计算需要前 Ready。

因此 Scheduler / KV Block Manager 需要知道：

```text
哪些 Block Ready
哪些 Block 正在 Transfer
哪些 Block 尚未 Transfer
当前 Decode 是否依赖这些 Block
```

### 6. Router 不能只看 GPU Load

已经形成初步 Routing Cost Model：

```text
Cost(P, D, Request)
≈ QueueWait(P)
 + PrefillTime(P)
 + KVTransferTime(P→D)
 + DecodeQueueWait(D)
 + ExpectedITL(D)
```

并且需要进一步考虑：

```text
KV Capacity
KV Transfer Distance
Current Decode Batch
Prefix Cache Locality
Priority / SLA
Expected TTFT / ITL
```

特别形成：

> **Cache-aware Routing**：Prefix Cache 命中位置本身就是路由决策的一部分。

## 本轮纠正

### 1. “显存带宽会决定 KV Cache Size”不准确

需要严格区分：

```text
Compute
→ 决定计算耗时

Memory Capacity
→ 决定能容纳多少 KV Cache

Memory Bandwidth
→ 决定 KV / 权重数据搬运吞吐
```

KV Cache Size 主要由 token 数、层数、KV heads、head dimension、数据类型等决定。

Memory Bandwidth 会影响 Decode 性能和 KV 访问/搬运效率，但不是决定 KV Cache 容量大小的直接因素。

## 当前系统脑图

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
             └─ KV Transfer ─────┘
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

## 下一课

**KV Cache Transfer / Connector / GPU Interconnect**

重点：

```text
GPU → GPU
GPU → CPU → GPU
GPU → NIC → RDMA → NIC → GPU
```

继续学习：

```text
PCIe
NVLink
RDMA
InfiniBand
RoCE
NCCL / KV Connector
```

最终回答：

> 8GB KV 从 Prefill GPU 到 Decode GPU 到底怎么搬？瓶颈在哪里？如何估算 transfer time？为什么 P/D Disaggregation 最终会进入 GPU Interconnect / Networking 的系统工程问题？
