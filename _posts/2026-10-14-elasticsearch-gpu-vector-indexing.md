---
layout: post
title: "1.38 亿条向量不到 10 分钟：Elasticsearch 如何用 GPU 加速索引"
date: 2026-10-14 12:00:00 +0800
categories: ["AI","搜索"]
tags: ["Elasticsearch","GPU","向量索引"]
description: "Elastic 报告用 8 张 NVIDIA GPU 在不到 10 分钟内为 1.38 亿条向量建好索引。拆解 cuVS 加速的环节、基准结果和适用边界。"
source_url: "https://www.elastic.co/search-labs/blog/vector-indexing-gpu-elasticsearch-cuvs"
---

Elasticsearch 用不到 10 分钟完成了1.38 亿条向量索引构建。

这是 Elastic 最近公布的一组 GPU 基准结果。Elastic 把 NVIDIA cuVS 集成到 Elasticsearch，用 GPU 来建 HNSW 向量索引。厂商测试报告称，索引吞吐量最高约提升 7 倍。

## 向量索引为什么会用到 GPU？

向量检索会先把文本、图片或其他内容转换成一串数字。搜索时，系统要从大量向量中找出与查询最相近的结果。

一种常见办法是建立 HNSW 图。可以把每个向量想成图上的一个点，再为它连接若干相近的点。查询时，系统沿着这些连接逐步靠近目标，避免把每条数据都扫描一遍。

问题在于，建图本身要处理大量向量之间的关系。数据规模越大，构建索引的计算开销越明显，也会和搜索、写入等工作争用 CPU。

Elastic 的做法，是通过 NVIDIA cuVS 把 HNSW 建图卸载到 GPU。GPU 并没有因此接管所有搜索和分析工作；搜索仍然可以在 CPU 上执行。

## 1.38 亿条向量的测试，具体测了什么？

Elastic 使用 Elasticsearch Rally 的 `msmarco` track 做宏观基准测试，数据包含 1.38 亿条、每条 1024 维的向量。博客称，这些向量代表约 4 TB 的多模态文档。

测试服务器配有双路 AMD EPYC 9555 处理器。GPU 组搭载 8 张 NVIDIA RTX PRO 6000；CPU 对照组是同样的 CPU 配置，只是关闭 GPU。图表给出的目标召回率是 95%。

结果是：CPU 组的索引吞吐约为每秒 3.8 万条，GPU 组约为每秒 28.1 万条。两者相比约 7.4 倍，Elastic 概括为约 7 倍。按总耗时看，1.38 亿条向量的索引构建从 CPU 上约 1 小时缩短到 GPU 上不到 10 分钟。

![Elasticsearch 在 CPU 与 GPU 上构建向量索引的吞吐量对比](/assets/images/11-elasticsearch-gpu/vector-indexing-throughput.png)

*图 1：CPU 约 38K docs/s，GPU 约 281K docs/s。图源：[Elastic 原文](https://www.elastic.co/search-labs/blog/vector-indexing-gpu-elasticsearch-cuvs)。*

这组数据的价值在于展示了大规模建图的潜在收益。它不能单独说明普通规模的数据也会快 7 倍：硬件数量、数据规模、索引配置和供数速度都会影响最终结果。

## 速度变快，搜索质量有变化吗？

索引建得快，如果搜索结果质量明显变差，提速就未必有意义。

Elastic 对比了 GPU 建图和 CPU 建图之后的召回率与查询吞吐曲线。博客称两条曲线大致重叠：在这组测试里，GPU 建图没有明显改变 HNSW 搜索的召回率与吞吐权衡。

![GPU 建图与 CPU 建图的向量索引在不同召回率下的查询吞吐曲线](/assets/images/11-elasticsearch-gpu/qps-vs-recall.png)

*图 2：两种索引的 QPS 与召回率曲线大致接近。图源：[Elastic 原文](https://www.elastic.co/search-labs/blog/vector-indexing-gpu-elasticsearch-cuvs)。*

这里的“没有明显变化”仍然只适用于这组基准。换成自己的向量模型、数据分布、索引参数和查询集合，召回表现可能不同。迁移时还是要用实际查询做相关性和召回评估。

GPU 建图还影响了索引与搜索同时运行时的延迟。Elastic 报告，p90 延迟从 CPU 建图时的约 45 毫秒降到 GPU 建图时的约 7 毫秒。

这不是 GPU 把每次搜索都加速了约 6 倍。搜索仍由 CPU 执行；延迟下降的主要原因，是建图转移到 GPU 后，CPU 资源不再被大量索引计算挤占。

![并发索引期间 CPU 搜索的 p90 延迟对比](/assets/images/11-elasticsearch-gpu/p90-search-latency.png)

*图 3：CPU 建图时约 45 ms，GPU 建图时约 7 ms。图源：[Elastic 原文](https://www.elastic.co/search-labs/blog/vector-indexing-gpu-elasticsearch-cuvs)。*

## Force merge 为什么也快了这么多？

基准还比较了 HNSW 索引的 Force merge。测试把每个 shard 合并到 4 个 segment：CPU 花了约 249 分钟，GPU 只用约 5 分钟，差距约 48 倍。

![CPU 与 GPU 构建的向量索引执行 Force merge 所需时间](/assets/images/11-elasticsearch-gpu/force-merge-time.png)

*图 4：该基准中 CPU 约 249 分钟，GPU 约 5 分钟。图源：[Elastic 原文](https://www.elastic.co/search-labs/blog/vector-indexing-gpu-elasticsearch-cuvs)。*

这个数字只针对这组规模和合并目标下的 Force merge，不能推导出常规写入、refresh 或后台合并都会快 48 倍。更准确的理解是：GPU 加速同样显著缩短了这项特定的索引合并工作。

## 什么样的工作负载值得评估？

如果集群经常批量导入大量向量、重建索引，或者索引任务会让在线搜索的 CPU 延迟明显升高，GPU 建图值得做一次对照基准。它的吸引力不只在于更快完成导入，也在于索引期间可能给 CPU 搜索留出更多资源。

如果主要瓶颈是全文检索、聚合、磁盘、网络或客户端供数慢，这项能力未必能解决问题。官方排障文档也提醒，GPU 只加速特定的建图环节；JSON 解析、存储吞吐和输入速度仍可能限制整体吞吐。

部署条件也需要提前核对。当前官方文档列出的前提包括 Enterprise 订阅、Linux x86_64、Java 22 或更新版本，以及至少 8 GB 显存、Compute Capability 不低于 8.0 的 NVIDIA Ampere 或更新架构 GPU。支持的向量索引类型为 `hnsw` 和 `int8_hnsw`，元素类型为 `float`；还需要匹配 Elasticsearch 版本的驱动、CUDA 和 cuVS 运行库。

GPU 加速向量索引的关键变化，是把 HNSW 建图这项计算密集型工作从 CPU 移到 GPU。是否值得采用，最终取决于建图是不是你当前的瓶颈，以及释放出来的 CPU 和节省的时间能否抵消 GPU 成本。

**来源**：[Elastic 基准测试原文](https://www.elastic.co/search-labs/blog/vector-indexing-gpu-elasticsearch-cuvs)；[Elastic GPU 向量索引文档](https://www.elastic.co/docs/solutions/search/vector/gpu-vector-indexing)。
