---
layout: post
title: "EmbeddingGemma 2：把文字、代码、图片和视频放进同一个搜索空间"
date: 2026-10-13 12:00:00 +0800
categories: ["AI","Embedding"]
tags: ["EmbeddingGemma","多模态","向量检索"]
description: "EmbeddingGemma 2 是 Google DeepMind 发布的开放权重多模态 embedding 模型。本文从统一向量空间出发，介绍它的工作方式、Mac 本地运行方法、模型细节和适用范围。"
source_url: "https://huggingface.co/google/embeddinggemma-2"
---

Google DeepMind 最近开放了 EmbeddingGemma 2。

它不是一个用来聊天和写答案的模型，而是一个用来生成向量的模型。文字、代码、图片、视频和音频经过它处理后，会变成可以比较距离的数字表示。

这件事听起来有点抽象，但它解决的是一个很具体的问题：

> 手机上的笔记、截图、代码、会议录音和视频，能不能放进同一个搜索系统里？

EmbeddingGemma 2 的主要特点，就是把这些不同类型的内容映射到同一个向量空间。

## 统一向量空间是什么？

传统搜索通常先区分内容类型。

搜文字，用倒排索引；搜图片，单独做图片特征；搜音频，还要先转录成文字。不同类型的内容各有一套处理方式，想做跨媒体搜索，就需要再设计一层转换。

EmbeddingGemma 2 采用了另一种方式：它把文本、代码、图片、视频和音频都转换成 768 维向量，并让这些向量处在同一个空间里。

在这个空间里，语义相近的内容会靠得更近。

比如，我输入一句“会议里提到的备份轮换方案”，系统可以把这句话转换成查询向量，再和笔记、截图、视频片段或录音片段的向量比较。结果不必先知道资料属于哪一种文件类型。

这就是跨模态检索：文字可以搜图片，文字可以搜视频，图片也可以和其他图片或文字比较相似度。

需要注意的是，模型只负责生成向量，不负责把搜索结果整理成一段答案。真正的问答仍然需要检索系统和生成模型配合。EmbeddingGemma 2 更像是资料库前面的“坐标生成器”。

## 在 Mac 上运行一个文本检索示例

最简单的方式是使用 Python 和 Sentence Transformers。

先创建一个虚拟环境。只做文字和代码检索时，可以先安装这两个包：

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -U sentence-transformers transformers
```

如果要处理图片、音频和视频，再安装官方指南列出的媒体依赖：

```bash
python -m pip install -U pillow soundfile torchcodec
```

下面这段代码会在 Apple Silicon 的 MPS 或 CPU 上加载文本模块，比较一个查询和一段文档的相似度。Mac 上先使用 `float32` 更稳妥，也不要改成 `float16`。

```python
import torch
from sentence_transformers import SentenceTransformer

model = SentenceTransformer(
    "google/embeddinggemma-2",
    device="mps" if torch.backends.mps.is_available() else "cpu",
    config_kwargs={
        "vision_config": None,
        "audio_config": None,
    },
    model_kwargs={"torch_dtype": torch.float32},
)

query = "task: search result | query: 如何轮换 GitLab 备份"
document = "title: 备份指南 | text: 定期轮换并验证 GitLab 备份。"

query_embedding = model.encode(
    query,
    truncate_dim=256,
    normalize_embeddings=True,
)
document_embedding = model.encode(
    document,
    truncate_dim=256,
    normalize_embeddings=True,
)

print(model.similarity(query_embedding, document_embedding))
```

这里有三个细节不能省略。

第一，查询和文档使用了不同的任务格式。第二，两个向量使用了相同的 256 维。第三，截断向量后重新做了归一化。少了这些步骤，代码可能仍然运行，但排序质量会悄悄下降。

如果要处理图片、音频或视频，就不要关闭对应的编码器，改用完整的多模态模型，并把媒体文件作为 `{"image": ...}`、`{"audio": ...}` 或 `{"video": ...}` 传入。官方指南还支持在一次输入中交错放入文字和多种媒体。[官方多模态指南](https://ai.google.dev/gemma/docs/embeddinggemma/multimodal-embeddinggemma-with-sentence-transformers)

## 模型细节

EmbeddingGemma 2 由 Google DeepMind 发布，官方 Hugging Face 仓库是 [`google/embeddinggemma-2`](https://huggingface.co/google/embeddinggemma-2)。它是一个 740M 参数的开放权重多模态 embedding 模型，基于 Gemma 4 decoder architecture。

它由几个可以按需加载的部分组成：

- 文本模型和 embedder 合计约 270M 参数；
- 视觉编码器约 170M 参数；
- 音频编码器约 300M 参数；
- 完整多模态模型约 740M 参数；
- 原生输出维度为 768；
- 支持最长 8,192 个 token 的共享上下文。

视觉和音频编码器相互独立。如果只做文本检索，可以不加载它们；如果只处理文字和图片，也可以关闭音频编码器。这种模块化设计降低了端侧部署时的内存压力。

模型还支持 Matryoshka Representation Learning，也就是可以把 768 维向量截短为 512、256 或 128 维。官方结果显示，256 维通常能保留大部分质量，128 维则更适合文本-only 场景，多模态质量下降会更明显。[官方模型卡](https://ai.google.dev/gemma/docs/embeddinggemma/model_card_2)

EmbeddingGemma 2 的代码检索表现也有明显提升。官方 MTEB Code 分数从第一代的 68.76 提升到 78.68。不过这仍然是标准 benchmark 结果，不能直接替代自己的代码库测试。

## 适合什么场景？

### 本地语义搜索

把笔记、文档、代码和附件建立索引，用户用自然语言搜索，而不用记住文件名和目录。

### 多媒体资料检索

在图片、录音和视频资料中，用文字搜索相关内容，或者用一张图片寻找相似图片。

### 端侧 RAG

先在本地完成资料召回，再把少量相关内容交给生成模型处理。敏感资料不必一开始就全部上传云端。

### 代码搜索

让开发者用自然语言查找函数、配置和实现方式，也可以为 Coding Agent 提供本地代码上下文。

### 分类、聚类和去重

向量不仅可以做搜索，也可以用于主题聚类、相似内容识别、重复文件发现和简单分类。

## 它不适合解决什么问题？

EmbeddingGemma 2 不会直接回答问题，也不会自动判断事实真假。它生成的是表示，最终效果取决于向量库、检索策略和后续应用。

100 多种语言支持，也不等于每种语言的效果相同。中文、专业术语、讽刺和隐喻，都需要用自己的数据验证。

官方 benchmark 使用的是全精度 checkpoint。换成本地量化、不同推理库或不同硬件后，速度和质量都可能变化。上线前至少应该用自己的数据测试 Recall@k、MRR、nDCG@k、延迟和内存。

还要注意数值精度。官方模型卡建议使用 `bfloat16` 或 `float32`，不要使用 `float16`，因为激活值范围可能超出 `float16` 的表示范围，导致 NaN 或不容易发现的质量下降。

EmbeddingGemma 2 最值得关注的地方，不只是参数量变大了，而是它把多模态内容放进了同一个可检索空间，又把模块化加载和端侧运行放到了设计中心。

如果你想在 Mac 上给自己的笔记、代码和媒体文件做一个本地搜索入口，它是一个值得先跑起来、再用自己的数据评估的模型。
