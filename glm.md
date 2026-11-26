---
layout: page
title: "GLM 模型研究合集"
permalink: /llm/glm/
description: "按 GLM 模型系列整理架构、循环计算与能力评估。"
---

{% assign glm_posts = site.posts | where_exp: "post", "post.tags contains 'GLM'" %}

这里按 GLM 模型系列整理研究文章，涵盖架构、计算机制与能力评估。目前共 {{ glm_posts | size }} 篇。

## 建议阅读顺序

先读 [GLM‑5.4 架构详解]({{ '/ai/architecture/2026/11/24/glm-5-4-architecture/' | relative_url }})，建立对共享权重、门控和 KV Cache 的整体认识。再看 [Loop Transformer]({{ '/ai/architecture/loop-transformer/' | relative_url }}) 的计算过程。能力与安全评估可独立阅读。

## 全部文章

{% for post in glm_posts %}
{% assign article_url = post.article_url | default: post.url %}
### [{{ post.title }}]({{ article_url | relative_url }})

{{ post.date | date: "%Y-%m-%d" }}

{{ post.description }}

{% endfor %}
