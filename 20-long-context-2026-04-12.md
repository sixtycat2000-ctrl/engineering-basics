# 长上下文的救兵：KV Cache 压缩与 Flash Attention

> 128k 上下文的 KV Cache 有 69GB——比模型权重还大 4 倍。三个技术正在解决这个难题：Flash Attention 在 SRAM 里算注意力，TurboQuant 把 KV 压缩 6 倍，PagedAttention 像操作系统分页一样管理显存。

---

## 为什么长上下文这么难

第 1 篇我们算过：8B 模型，128k 上下文的 KV Cache 约 69GB。权重才 4GB。

```
128k 上下文的内存分布（8B 模型，4-bit 权重）

  ┌────────────────────────────────────────────────┐
  │                                                │
  │              KV Cache (~69 GB)                 │  ← 94%
  │                                                │
  ├────┤                                           │
  │权重│                                           │
  │4 GB│                                           │
  └────┘───────────────────────────────────────────┘
```

长上下文推理的瓶颈不在权重，在 KV Cache。三个技术从不同角度解决这个问题。

---

## 入门：三个技术，三个角度

| 技术 | 解决什么 | 怎么做 | 效果 |
|------|---------|--------|------|
| Flash Attention | Attention 计算慢、临时内存大 | 分块在 SRAM 里算，不存 N×N 矩阵 | 速度 2-4x，内存大幅降低 |
| TurboQuant | KV Cache 太大 | 把 KV 从 16-bit 压到 3-bit | 6x 内存减少，零精度损失 |
| PagedAttention | KV Cache 碎片化 | 像操作系统分页一样管理 | 支持更多并发请求 |

它们是互补的，可以叠加使用。

---

## Flash Attention：不在显存里存中间矩阵

### 问题是什么

Attention 的核心计算是 QK^T（Query 和 Key 的点积），产生一个 **N×N 的矩阵**（N = 序列长度）。

```
标准 Attention 计算：

  Q (N×d)  ×  K^T (d×N)  =  Score (N×N)
                                  ↑
                            128k × 128k = 160 亿个数
                            FP16 = 32 GB 临时内存
```

128k 上下文时，这个中间矩阵需要 **32 GB** 临时显存。而且要反复在 HBM（高带宽显存，大但慢）和 SRAM（快速缓存，小但快）之间搬运。

### Flash Attention 怎么解决

核心思路：**分块计算（tiling）**。

不一次性算整个 N×N 矩阵，而是切成小块。每个小块在 GPU 的 **SRAM**（快但小）里完成 softmax、点积等所有操作，只把最终结果写回 HBM。

```
标准 Attention：
  HBM → 算整个 N×N → 存回 HBM → 读回来算 softmax → 存回去...
  反复读写，N×N 矩阵每次都要搬

Flash Attention：
  HBM → 取一小块 Q 和 K → 在 SRAM 里算完 softmax → 直接输出最终结果
  不存中间的 N×N 矩阵，HBM 读写次数大幅减少
```

**效果**：

| 维度 | 标准 Attention | Flash Attention |
|------|--------------|----------------|
| 临时内存 | O(N²) — 128k 时 ~32 GB | O(N) — 线性增长 |
| HBM 读写 | 多次搬运大矩阵 | 最少次数读写 |
| 速度 | 基线 | 2-4x 加速 |

### 三个版本的演进

| 版本 | 年份 | 改进 |
|------|------|------|
| FlashAttention-1 | 2022 | 首次提出分块计算 |
| FlashAttention-2 | 2023 | 优化 warp 调度，更快 |
| FlashAttention-3 | 2024 | 针对 H100 优化，利用异步和 FP8，比 v2 快 1.5-2x |

现在几乎所有推理框架（vLLM、llama.cpp、SGLang）都默认使用 Flash Attention。

---

## TurboQuant：把 KV Cache 压缩 6 倍

Google Research / DeepMind 在 2026 年 3 月发布的 KV Cache 压缩技术。

### 核心思路

标准的 KV Cache 每个值存 16 位（FP16/BF16）。TurboQuant 把它压到 **3 位**，精度几乎零损失。

怎么做到的？关键在于处理 **异常值（outlier）**。

KV 向量里偶尔有非常大的值（和权重一样的问题）。如果直接量化，这些异常值会把 scale 拉大，正常值全被挤成 0。

TurboQuant 用了一个巧妙的变换：

```
标准量化（笛卡尔坐标）：
  直接对 (x, y) 量化 → 异常值拉大 scale → 正常值丢失

TurboQuant（极坐标量化 PolarQuant）：
  (x, y) → 转成 (r, θ) 即 (长度, 角度)
  长度 r 和角度 θ 分别量化 → 异常值被隔离在长度维度 → 角度精度不受影响
```

把向量从笛卡尔坐标转成极坐标，异常值（很大的分量）变成了"方向"和"长度"两个独立的维度。长度可以单独处理，不影响方向信息的精度。

再叠加 **QJL（Quantized Johnson-Lindenstrauss）** 随机投影，进一步压缩。

### 效果

| 指标 | 结果 |
|------|------|
| 内存减少 | **6 倍**（16-bit → 3-bit） |
| 精度损失 | **零**——LongBench、Needle-in-a-Haystack 等基准几乎完全一致 |
| Attention 速度 | 最高 **8 倍**提升（对比未压缩的 32-bit keys） |
| 需要重新训练？ | 不需要，训练-free |
| 需要看数据？ | 不需要，data-oblivious |

### 实际意义

```
没有 TurboQuant：
  128k context, 8B 模型 → KV Cache ~69 GB → 需要服务器

有 TurboQuant（6x 压缩）：
  128k context, 8B 模型 → KV Cache ~11.5 GB → MacBook 32GB 可能够跑

1M context 呢？原来需要 ~540 GB，压完 ~90 GB → 从不可能变成可能
```

⚠️ TurboQuant 压的是 **KV Cache**，不是模型权重。它和权重量化是互补的——两个一起用，内存压力最小。

---

## PagedAttention：像操作系统一样管理 KV Cache

### 问题是什么

多条对话同时进行时，每条对话的 KV Cache 长度不同，而且持续增长。传统方式给每条对话预分配一大块连续内存，大部分时间是浪费的。

```
传统 KV 内存管理：

  对话1: [████████████    ]  预分配了 128k 的空间，才用了 2k
  对话2: [████████████████████████]  用满了，还要扩展
  对话3: [██                ]  预分配了，刚开始

  浪费严重，而且"外部碎片化"——空闲块不少但不够大
```

### PagedAttention 怎么解决

借鉴操作系统的**虚拟内存分页**：把 KV Cache 切成固定大小的"页"，按需分配，不需要连续的物理内存。

```
PagedAttention 内存管理：

  物理内存被切成固定大小的页：
  [页1][页2][页3][页4][页5][页6][页7][页8]

  对话1 用了页1, 页3, 页5 → 不需要连续
  对话2 用了页2, 页4, 页6 → 按需分配
  对话3 用了页7 → 刚开始，只占 1 页

  用完了的页可以回收 → 内存利用率高
```

**效果**：同一个 GPU 能同时服务更多用户（batch size 更大），KV Cache 的内存浪费从 60-80% 降到 4% 以下。

vLLM 是最早实现 PagedAttention 的推理框架，现在已经是行业标准。

---

## 高级：三个技术叠加

这三个技术不冲突，可以叠加：

```
单条对话的内存优化路径：

1. Flash Attention    → 减少 Attention 的临时内存（SRAM 里算）
2. TurboQuant        → KV Cache 从 16-bit 压到 3-bit（6x）
3. PagedAttention     → 碎片化内存管理（利用率从 20% → 96%）
4. 权重量化 (INT4)   → 权重从 16-bit 压到 4-bit（4x）

叠加效果（128k context, 8B 模型）：
  原始：权重 16GB + KV 69GB = 85 GB
  全部优化：权重 4GB + KV 11.5GB = ~19 GB  ← MacBook 32GB 够了
```

---

## 速查表

| 记住这个 | 为什么 |
|---------|------|
| **128k KV Cache 是权重的 4 倍** | KV 线性增长，权重固定 |
| **Flash Attention 不存中间矩阵** | 分块在 SRAM 里算，O(N²) → O(N) |
| **TurboQuant 把 KV 压 6 倍零损失** | 极坐标量化隔离异常值 |
| **PagedAttention 按页分配** | 像操作系统管内存，浪费从 80% → 4% |

---

## 系列总结

6 篇文章，一个完整的知识链：

```
① 内存账本        → 模型到底要多少内存？
② 浮点数精度      → 为什么能减少位数？
③ 推理两阶段      → 瓶颈在哪？
④ 量化三步曲      → 怎么压缩权重？
⑤ 效果实测        → 压完好不好用？
⑥ 长上下文救兵    → KV Cache 怎么压缩？
```

核心结论：**量化模型是实用的。** 4-bit 量化是甜点——4 倍压缩、2 倍加速、精度只丢 5%。别因为"压缩过"就害怕尝试本地模型。

---

## 参考资料

- [Flash Attention 论文](https://arxiv.org/abs/2205.14135) — FlashAttention-1 原理
- [FlashAttention-2](https://arxiv.org/abs/2307.08691) — 优化 warp 调度
- [FlashAttention-3](https://arxiv.org/abs/2407.08608) — Hopper GPU 优化
- [TurboQuant](https://arxiv.org/abs/2503.10507) — Google 的 KV Cache 压缩（ICLR 2026）
- [PagedAttention / vLLM](https://arxiv.org/abs/2309.06180) — 分页式 KV 管理
- [Efficient Large Language Models: A Survey](https://openreview.net/pdf?id=bsCCJHbO8A) — 模型效率综述

---

## 系列导航

1. [LLM 的内存账本](15-llm-memory-budget-2026-04-12.md) →
2. [浮点数精度](16-floating-point-precision-2026-04-12.md) →
3. [推理的两个阶段](17-inference-two-phases-2026-04-12.md) →
4. [量化三步曲](18-quantization-algorithms-2026-04-12.md) →
5. [量化效果实测](19-quantization-benchmark-2026-04-12.md) →
6. **长上下文的救兵**（本文）
