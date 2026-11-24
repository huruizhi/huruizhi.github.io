---
layout: post
title: "从第一性原理理解 Loop Transformer"
date: 2026-11-25 01:40:00 +0800
permalink: /ai/architecture/2026/11/25/loop-transformer/
article_url: /ai/architecture/loop-transformer/
evidence_anchor: chapter-3
categories: [AI, architecture]
tags: [GLM, Transformer, Loop Transformer, DSA, KV Cache]
description: "同一组 Transformer 层为什么值得再跑一遍？用图解和三个滑块理解 GLM-5.4 的共享权重、有效计算深度与门控渐进更新。"
evidence_label: "循环层计算"
evidence_steps: ["78 个独立层", "28 层再跑一遍", "106 次层计算"]
evidence_caption: "用滑块观察循环次数怎样改变有效计算深度"
---

GLM-5.4 把中间 28 层再运行一遍，每次前向传播因此执行 106 次层计算。完整讲解从一个四层模型出发，逐步解释共享权重、门控更新、混合 KV Cache 和稀疏索引复用。你可以调整循环次数和两轮 Gate，观察计算深度与状态更新幅度的变化。
{: #evidence-01 }

[打开完整交互讲解]({{ '/ai/architecture/loop-transformer/' | relative_url }})
