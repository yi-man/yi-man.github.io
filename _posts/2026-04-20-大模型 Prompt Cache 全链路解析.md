---
title: "大模型 Prompt Cache 全链路解析"
date: 2026-04-20 18:00:00 +0800
categories: [AI]
tags: [LLM, Prompt Cache, Prefix Cache, KV Cache, 性能优化]
pin: false
---

## 一、现象：你可能已经注意到了

在调用大模型 API 时，你可能发现相同的请求有时返回快得出奇，而费用账单里也偶尔出现"cache tokens"字样。这不是偶然——这是 **Prompt Cache（提示词缓存）** 在发挥作用。

让我们先看整体链路，再逐层拆解。
![586](/assets/img/posts/prompt-cache/file-20260420164327132.svg)

整条链路上有三个层次可以发生缓存，越靠左命中收益越大（完全不消耗网络和算力），越靠右命中发生在算力最昂贵的地方。

---

## 二、缓存的影响：有无缓存的区别

### 延迟维度
![685](/assets/img/posts/prompt-cache/file-20260420164327397.svg)

### 成本维度

缓存命中对 token 计费的影响非常直接。以 Anthropic Claude 为例，Cache Read Token 的价格约为普通 Input Token 的 **10%**；OpenAI 的 Prompt Caching 约打 **50% 折扣**。

对于包含大段系统提示、长上下文文档或工具定义的应用场景（Agent、RAG），每次请求可能有 80%+ 的 token 属于可复用前缀，成本节省非常可观。

---

## 三、核心原理：缓存发生在哪个环节

大模型推理分两个阶段：**Prefill（预填充）** 和 **Decode（解码）**。缓存的本质是对 Prefill 阶段的 KV Cache 进行复用。
![473](/assets/img/posts/prompt-cache/file-20260420164327367.svg)

**核心机制**：Transformer 的每一层都需要为所有 token 计算 Key 和 Value 矩阵。Prefix Cache 将"已经计算过的、稳定不变的前缀部分"对应的 KV 矩阵持久化在 GPU 显存中，下次相同前缀的请求来临时直接读取，跳过这部分的 forward pass。

---

## 四、平台侧方案：Prefix Cache 深度解析

平台侧是缓存实现最核心的地方，主要包括两个子层：**LLM 提供商**和**中转站**。

### 4.1 LLM 提供商：Prefix Cache 实现原理
![477](/assets/img/posts/prompt-cache/file-20260420164327314.svg)

**关键设计点**：

**分块哈希（Block-wise Hashing）**：将 prompt 按固定长度（如 128 tokens）切成 block，每个 block 的 hash 依赖父 block 的 hash，形成链式结构。这样即便后续内容变化，已匹配的前缀 block 依然可以命中缓存。

**LRU 淘汰策略**：GPU 显存有限，KV Cache 按 LRU（最近最少使用）淘汰。高复用率的 system prompt block 自然会常驻显存。

**多级存储**：高端部署中 KV Cache 可分层存储——热数据在 GPU HBM（高带宽内存），温数据在 CPU DRAM，冷数据在 NVMe SSD，类似 CPU 三级缓存的设计理念。

### 4.2 中转站（Proxy）：语义缓存

中转站代理（如 LiteLLM、自建 OpenAI-compatible 代理）可以在不触达 LLM 提供商的情况下完成缓存，有两种模式：**语义缓存**是中转站的独特优势——即使用户的措辞略有不同（"帮我总结这段文字" vs "请概括以下内容"），如果语义足够接近（cosine 相似度超过阈值），可直接返回已有答案。常见实现是 Redis + pgvector / Qdrant。

---

## 五、如何提高缓存命中率

### 5.1 平台侧（LLM 提供商）
![477](/assets/img/posts/prompt-cache/file-20260420164327250.svg)

### 5.2 用户侧：如何最大化缓存命中

这是应用开发者最能控制的部分，也是最常被忽视的部分。核心原则是**让 prompt 的前部尽可能稳定、后部集中变化**。
![561](/assets/img/posts/prompt-cache/file-20260420164327159.svg)

## 六、用户侧完整策略清单

除了 prompt 结构调整，用户侧还有以下手段可以系统性提升命中率：

**Prompt 结构设计**

将 system prompt、工具定义（tools/functions）、参考文档等固定内容统一前置。对话轮次中，已完成的历史对话也应稳定保持，不要每次重新摘要或重写——摘要会改变 token 序列，导致缓存失效。

**Session 复用与 TTL 管理**

大多数提供商的 Prefix Cache TTL 在 5 分钟至数小时不等。应用层应维护 session，确保同一会话的多轮请求在 TTL 内连续发出，而不是让会话超时后再重开导致缓存失效。Anthropic 的 `cache_control` 提供了显式的 TTL 延续机制。

**请求去重与本地缓存**

在调用 LLM 前，对完全相同的请求（相同的 messages 序列）在应用层做哈希去重，直接复用上次的响应。这是成本最低的缓存层，甚至不需要消耗任何 cache read token。

**内容稳定性保障**

避免在固定内容中嵌入时间戳、随机 seed、或每次请求变化的 request_id——这些会破坏前缀哈希，令本来可以命中的 block 失效。

**Batch 请求对齐**

在批量处理任务时（如对大量文档做同类型分析），将 system prompt 和任务描述统一，仅更换最后的文档内容，可以让整个 batch 共享同一前缀缓存，显著降低整体延迟和成本。

---

## 七、各层缓存策略对比总览

|层次|缓存类型|命中收益|命中条件|适用场景|实现复杂度|
|---|---|---|---|---|---|
|用户侧  <br>本地缓存|响应级精确缓存|最高 零 API 调用|完全相同请求|重复批量任务|低，应用内 KV|
|中转站  <br>精确缓存|Hash 匹配响应|高 不消耗 API|完全相同请求|多用户重复查询|中，需 Redis|
|中转站  <br>语义缓存|向量相似匹配|中 不消耗 API|语义相近|FAQ / 知识库|高，需向量库|
|LLM 平台  <br>Prefix Cache|KV Block 复用|高 折扣 token|前缀 token 相同|多轮对话 / Agent|平台自动，零配置|
|LLM 平台  <br>显式 Cache|cache_control 标记|高 折扣 token|breakpoint 内内容不变|长系统提示 / RAG|低，API 参数|


## 八、架构师视角：设计决策要点

**分层缓存，各司其职**：不要把所有希望寄托在 LLM 提供商的 Prefix Cache 上。应用层本地缓存是最廉价的防线，中转站语义缓存可以提升整体命中覆盖面，Prefix Cache 则处理"必须发出的请求中尽可能少算"的问题。三层叠加才是完整的成本优化方案。

**缓存边界即架构契约**：一旦决定了 system prompt 和工具定义的结构，就要将其视为稳定接口——随意重构会破坏已建立的缓存热度（cache warmth），尤其是在高并发的生产系统中。

**观测性**：在监控指标中区分 `cache_read_tokens`、`cache_creation_tokens` 和 `input_tokens`，计算 **Cache Hit Rate = cache_read_tokens / (cache_read_tokens + input_tokens)**，这个比率是优化成效的核心 KPI。同时监控 TTFT（Time to First Token）的 P50/P95 分布，缓存命中时 TTFT 下降应当是显著可见的。

**TTL 与预热策略**：对于冷启动场景（如每天早上第一批请求），可以设计"预热请求"——在业务流量到来前，主动发送包含完整 system prompt 的 dummy 请求，将 KV 块加热到 GPU 显存。这在延迟 SLA 严格的服务中尤为重要。
