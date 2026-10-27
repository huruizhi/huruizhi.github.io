---
layout: post
title: "Vllm 0.31.0 发布，DeepSeek-V4.1-Flash 性能大幅提升"
date: 2026-10-15 12:00:00 +0800
categories: ["AI","推理"]
tags: ["vLLM","DeepSeek","性能优化"]
description: "从注意力、稀疏索引、KV 缓存到 MoE 执行路径，拆解 vLLM 0.31.0 面向 DeepSeek-V4.1-Flash 的优化，以及性能数据适用的边界。"
source_url: "https://github.com/vllm-project/vllm/releases/tag/v0.31.0"
---

DeepSeek-V4.1-Flash 的推理慢在哪里？可能是长上下文的预填充，也可能是每个 token 生成时的注意力计算、MoE 专家路由，或者多卡之间的数据同步。

vLLM 0.31.0 把这些环节都纳入了优化范围。与其说它是“换了一个更快的 kernel”，不如把它看作一次围绕推理数据流的整理：少读不必读的数据，少搬中间结果，把原本分开执行的步骤合并起来。

这也解释了为什么“性能大幅提升”要放到具体场景里看：不同优化瞄准的瓶颈不同，提升幅度也会随上下文长度、并行配置和硬件而变。

## 注意力和 KV cache：减少每一步要搬运的数据

模型生成新 token 时，需要参考之前 token 的 Key 和 Value。这些数据保存在 KV cache 中，供后续计算重复使用。上下文越长，缓存越大；每一步解码都要读取，显存读写就可能成为主要开销。

vLLM 0.31.0 在 SM100 系列硬件上默认采用面向 DeepSeek-V4.1-Flash 的 FlashMLA mega attention 路径，并配合 NVFP4 压缩 KV cache。可以把它理解为两件配套的工作：压缩表示降低缓存的数据量，专用注意力内核直接按这种表示执行计算，避免为了压缩再增加不必要的转换。

PR #56935 给出了一组单张 GB300 上的 decode kernel 测试：在 CUDA Graphs 和 64-head 模型下，比较 mega attention 与稀疏 FlashMLA 两条路径在不同张量并行度下的执行时间。`s_q` 是该基准中的查询 token 数配置。

| 张量并行度 | s_q=128 耗时（µs） | s_q=512 耗时（µs） | s_q=512 时的差异             |
| ----- | -------------: | -------------: | ------------------------ |
| TP1   |        31 / 40 |       96 / 140 | mega attention 约快 1.45 倍 |
| TP2   |        35 / 35 |      110 / 119 | 差距较小                     |
| TP4   |        34 / 34 |      109 / 109 | 基本持平                     |
| TP8   |        33 / 33 |      107 / 104 | mega attention 略慢        |

表格中每格依次为 mega attention / 稀疏 FlashMLA 的耗时。TP1、`s_q=512` 时，耗时从 140 微秒降到 96 微秒，是这组数据里最明显的差距；TP2 到 TP8 接近持平，TP8 下新路径反而略慢。

这是一项内核级 decode 基准，PR 没有测 prefill，也不是完整服务的端到端结果。它能说明特定配置下内核有明显收益，但不能直接推导出所有部署都会快 1.45 倍。何况这条默认路径有 SM100 的硬件边界，其他 GPU 并不会自动获得同等收益。

## 稀疏索引器：长上下文时，先把“找位置”这步做轻

稀疏注意力不会让每个 token 都关注整个上下文，但需要先挑出值得关注的位置。如果索引器仍然为所有上下文位置计算分数，再丢掉不需要的结果，筛选这一步本身就会吃掉大量计算和内存带宽。

vLLM 的稀疏 MQA logits 优化改为只给候选位置算分。对 V4.1-Flash，候选范围固定为 16K positions。上下文越长，省掉全量打分的价值就越明显。这条路径是 opt-in，需要满足相应的硬件和软件依赖，并非默认对所有服务开启。

PR #56254 在 4 张 GB300、TP4 配置下给出了端到端吞吐数据。下表为启用相对未启用该路径的速度比：

| 上下文长度 / batch | Decode 吞吐 | Prefill 吞吐 |
| ------------- | --------: | ---------: |
| 8K / 64       |     1.06× |      0.95× |
| 100K / 32     |     1.05× |      1.06× |
| 256K / 16     |     1.05× |      1.19× |
| 512K / 8      |     1.05× |      1.43× |
| 1M / 4        |     1.03× |      2.00× |

趋势很清晰：长上下文下 Prefill 收益增长明显，1M 场景达到 2 倍，Decode 则大致提升 3%–6%；而 8K 场景的 Prefill 还慢了约 5%。因此这项优化更适合受长上下文预填充限制的工作负载，不能把“2 倍”当成普遍推理加速。

## KV cache 的另一种省法：只重放滑动窗口所需的部分

vLLM 官方技术博客还介绍了 SWA bounded replay。DeepSeek-V4.1-Flash 同时使用全局 KV 和滑动窗口注意力（SWA），而 SWA 每层只需最近的 128 个位置。对于较深的 decoder 层，系统可以只重放这段有限窗口，不必在这些层上重复处理完整 prompt。

按博客描述的实现，第 20 层仍为所有 prompt token 生成全局 KV，第 21 到 39 层只处理末尾 128 个 token。裁剪后的 decoder 层再配合 CUDA Graphs 执行，以降低 kernel 启动等开销。它的取舍同样直接：减少长 prompt 的计算量和 prefix cache 存储，但 bounded replay 并非逐位完全相同的计算路径，质量需要单独评估。

博客报告，在 GB200 单请求、关闭 prefix cache 的设置下，decoder replay 加 CUDA Graphs 将 prefill 计算时间降低了 30%–40%（每组取 3 次运行的中位数）。这里的条件很关键：短 prompt 若不用 CUDA Graphs，额外的启动成本可能抵消节省，部分短输入甚至出现回退。

## MoE 与多卡通信：少几次启动，少一些中间周转

MoE 模型先通过 gate 决定 token 交给哪些专家，再执行专家计算，最后汇总结果；多卡运行时还要经过 all-reduce 等通信步骤。如果这些细碎步骤逐个启动 kernel、逐次读写中间结果，累积开销同样会拖高延迟。

0.31.0 中的 Mega-Gate 把 gate 矩阵计算与专家选择融合在一起。PR #56266 的 GB300 路由 kernel 测试显示，中等规模及更大的 token rows 快约 1.18–1.31 倍；较小规模（M≤16）没有收益，部分情况更慢。这里测的只是路由子步骤，不包含专家计算、通信或整个服务流程。

另一些改动把 mHC 边界计算、TP all-reduce 和 MoE 结果汇总等相邻步骤融合，减少独立 kernel 启动和数据交接。这类优化瞄准的是特定并行配置和小批量执行路径。相关服务端数据里，有的端到端 decode 吞吐提升在 2% 以内，较大 batch 则基本持平。它们属于补齐执行链路的局部改进，不宜和索引器的长上下文收益混成同一个倍数。

## 怎么理解“大幅提升”

vLLM 官方博客还汇总了 DeepSeek-V4.1-Flash 在 AgentX 上的结果：低延迟配置相较 day-0 约提升 1.9 倍，高吞吐配置约提升 5 倍；接近 100K 吞吐时，TTFT（首 token 延迟）降低约 70%。这些结果来自 bounded replay、CUDA Graphs 和多个 kernel 优化的组合。

![vLLM 在 SemiAnalysis AgentX 上的吞吐与交互性对比：2026 年 9 月 11 日至 10 月 2 日](/assets/images/vllm-deepseek-v41-agentx-results.png)

*图：vLLM 官方博客引用的 AgentX 对比图，展示 GB300 NVL72 上 day-0 与 10 月 2 日版本的吞吐和交互性变化。来源：[vLLM 官方技术博客](https://github.com/vllm-project/vllm-project.github.io/blob/main/_posts/2026-10-07-deepseek-v41-flash.md)，数据源为 SemiAnalysis InferenceX。*

还要把时间和归因说清楚：这篇博客发布于 10 月 7 日，结果观察截至 10 月 2 日，记录的是从 day-0 到当时的累计开发进展，并非以 vLLM 0.31.0 标签为起点和终点的严格对照实验。所以它能说明的是，多项优化组合之后特定 AgentX 工作负载取得了怎样的结果；不能把全部提升都归给 0.31.0 的某个单一改动。

如果只记住一个结论：0.31.0 的优化覆盖了推理链路里的不同瓶颈，但不存在适用于所有请求的统一加速倍数。长上下文 Prefill 可以重点看稀疏索引和 bounded replay；Decode 要关注注意力内核、KV cache、批次和 TP 配置；MoE 与多卡场景则要看路由、同步和通信成本。

评估自己的部署时，应当用相同硬件、模型、请求分布和并发设置做前后对照，分别观察 Prefill 吞吐、TTFT、Decode 速度、整体吞吐和显存占用。这样才知道优化有没有命中真实瓶颈。

**参考资料**： [vLLM 0.31.0 发布说明](https://github.com/vllm-project/vllm/releases/tag/v0.31.0) · [vLLM DeepSeek-V4.1-Flash 技术博客](https://github.com/vllm-project/vllm-project.github.io/blob/main/_posts/2026-10-07-deepseek-v41-flash.md) · [FlashMLA + NVFP4 KV cache PR #56935](https://github.com/vllm-project/vllm/pull/56935) · [稀疏 MQA logits PR #56254](https://github.com/vllm-project/vllm/pull/56254) · [Mega-Gate PR #56266](https://github.com/vllm-project/vllm/pull/56266) · [DeepSeek-V4.1-Flash 技术报告](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash/blob/main/DeepSeek_V41_Tech_Report.pdf)
