---
layout: post
title: "DeepSeek Sparse Attention（DSA）原理与交互演示"
date: 2026-11-25 00:00:00 +0800
permalink: /ai/architecture/2026/11/25/deepseek-sparse-attention/
categories: [AI, architecture]
tags: [DeepSeek, Transformer, DSA, MLA]
description: "从 Lightning Indexer 和 Top-K 筛选出发，用交互图表理解 DeepSeek-V3.2 的稀疏注意力。"
---

处理长上下文时，模型怎样挑出当前 Token 真正需要读取的位置？这篇文章用 20 个 Token 的例子解释 DeepSeek Sparse Attention，并用矩阵对比全量注意力、滑动窗口和 DSA。你可以调整 Top-K 和上下文长度，观察主注意力实际读取多少个位置。

[打开完整交互讲解]({{ '/ai/architecture/dsa/' | relative_url }})
