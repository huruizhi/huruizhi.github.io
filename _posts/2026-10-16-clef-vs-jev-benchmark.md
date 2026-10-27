---
layout: post
title: "Clef 在 Jev 的主场赢了吗？一次真实 Agent 决策基准测试"
date: 2026-10-16 12:00:00 +0800
categories: ["AI","Agent"]
tags: ["决策模型","Benchmark","Clef"]
description: "117 个样本、1053 次调用：Clef 在为 Jev 编写的决策任务上取得更高准确率，但真正重要的发现是确定性、分数漂移，以及模型阈值不能直接迁移。"
source_url: "https://construct.computer/blog/clef-vs-jev-benchmark/"
---

## Clef 和 Clef-flash 是什么

决策模型不负责写文章，也不负责规划一连串工具调用。它接收当前状态和一组有明确类型的问题，返回每个可选答案的概率，再由代码根据阈值决定下一步。

例如，一封新邮件到达后，Agent 可以询问：

- 这是一封需要回复的客户邮件，还是订阅通知、收据或垃圾邮件？
- 邮件中提到的 Priya Shah，是否就是记忆里已经存在的那个人？
- 接下来准备调用的工具会不会修改重要数据，是否应该先等用户确认？

这些都不是开放式写作任务。系统需要的是一个结构稳定、便于代码读取的判断结果。

Jev 是 TypeSafe AI 推出的闭源决策模型。Clef 和 Clef-flash 来自 Cloudflare：Clef 是基于 Qwen3.8-27B 后训练的 27B 模型，Clef-flash 是较小的 9B 版本。两者都开放权重，采用 Apache 2.0 许可证，可以通过 Workers AI 调用，也可以自行部署。

三者都接受结构相近的 Choice、Score 和 Noul 类型问题。Cloudflare 把 Clef 定位为偏精度的模型，把 Clef-flash 定位为延迟敏感场景下的轻量版本。Clef 系列还能读取图片，而 Jev 在这次比较中只处理文本和 JSON。

## 准确率结果：Clef 领先

这次 benchmark 没有为 Clef 重新设计题目。测试直接取了 Construct Agent 已经实现的五类决策：实体匹配、邮件分类、记忆提取门控、通知优先级和工具调用风险。

117 个样本全部是合成数据，每类 15 到 30 个案例，还刻意多放了一些贴近决策边界的情况。标签在测试前由作者和 Claude 共同整理，但尚未经过第二位人工复核。每个案例交给三个模型各跑三次，调用顺序交错，主测试共计 1053 次调用。

最关键的一点是：所有生产阈值都是按 Jev 校准的，没有为 Clef 做任何调整。

| 评估方式          |   Jev |  Clef | Clef-flash |
| ------------- | ----: | ----: | ---------: |
| 沿用当前生产阈值      | 88.6% | 92.3% |      82.1% |
| 在同一批样本中寻找最佳阈值 | 93.7% | 96.6% |      94.9% |
| 用留出案例选择阈值     | 90.9% | 93.2% |      90.6% |

![Jev、Clef 与 Clef-flash 在当前阈值和各自样本内最佳阈值下的总体准确率](/assets/images/13-clef-vs-jev/01-accuracy-overall.webp)

*图 1：左侧沿用为 Jev 设置的当前阈值，右侧在同一批样本中为各模型寻找最佳阈值。图源：[ConstructComputer 原文](https://construct.computer/blog/clef-vs-jev-benchmark/)。*

只看第一行，Clef 比 Jev 高 3.7 个百分点。但这 3.7 个百分点的 95% 置信区间是 -1.1 到 8.8，跨过了 0——不能据此断言 Clef 普遍优于 Jev。

测试集中还有三个实体匹配案例，在真实产品里会先被代码拦截，根本不会送到模型：名字完全相同，或者只有大小写差异。Jev 恰好在这三个案例上表现较差。去掉它们之后，Clef 与 Jev 的差距从 3.7 个百分点缩小到 2.0 个百分点——Clef 92.1%，Jev 90.1%。

分项结果也不像排行榜那么整齐。Clef 在实体匹配和通知优先级上领先；Jev 赢下了邮件分类，并且在记忆提取门控中保持满分；工具调用风险上两者持平。

![三个模型在五类 Agent 决策上的准确率](/assets/images/13-clef-vs-jev/02-accuracy-by-decision.webp)

*图 2：五类决策沿用当前生产阈值时的准确率。不同模型擅长的任务并不完全相同。图源：[ConstructComputer 原文](https://construct.computer/blog/clef-vs-jev-benchmark/)。*

所以，这组数据能支撑的表述其实很窄：在一批 Clef 从未见过、而且是专门为 Jev 编写的私有提示词上，Clef 没有表现出可测量的劣势，还在未调阈值的情况下拿到了更高的分数。

## Clef 是否真的具有确定性

准确率只告诉我们一次调用答得对不对。Agent 还关心另一个问题：完全相同的输入再次出现时，模型会不会改变主意？

研究者把 117 个请求分别调用三次。Clef 和 Clef-flash 在全部 351 次重复调用中，都返回了完全相同的概率，精确到小数点后四位。

为了排除缓存的干扰，他们又改动了不会进入模型输入的部分：给 URL 加唯一查询参数、重新格式化 JSON、打乱 JSON key 的顺序。每次响应都显示缓存未命中，返回的也是新的 inference ID，但结果依然纹丝不动。

Jev 就不一样了。真正参与决策的那个概率，在相同输入下最多漂了 0.10，两个案例因此跨过阈值，最终判断跟着翻转。如果把 Jev 返回的全部概率都算进来，最大变化达到 0.16，117 个案例里有 100 个至少出现过一次不同的数字。

![三个模型在重复调用中的最大概率变化](/assets/images/13-clef-vs-jev/05-repeatability.webp)

*图 3：每类决策连续三次调用时，参与判断的概率最大变化。Clef 和 Clef-flash 均为 0。图源：[ConstructComputer 原文](https://construct.computer/blog/clef-vs-jev-benchmark/)。*

Cloudflare 给出的机制解释是：Clef 先用 Qwen 做一次 prefill，再并行计算 schema 中所有合法选项的分数。这个决策步骤是非自回归的，不需要逐 token 采样中间文本，因此可以稳定地从内部表示得到每个选项的概率。

这次测试能确认的，只是可观察到的结果：在这批请求上，Clef 的重复输出完全一致。它无法证明 Clef 在所有输入、所有托管版本、所有运行环境下都永远不变。

不过，对需要跨过阈值才执行动作的 Agent 来说，这个结果非常实际。准确率差两三个点，影响的只是总体统计；而一个贴着阈值的案例在两次调用之间翻转，直接改变的是某一次工具调用会不会真的发生。

## Jev 模型，分数为什么变了

9 月 20 日，ConstructComputer 为实体匹配做过三次校准：

- `Dr. Priya Shah` 与 `Priya Shah` 的匹配概率是 0.71；
- `Priya Shah` 与同名记录的匹配概率是 0.76；
- `Atlas migration` 与已有项目的匹配概率是 0.82。

三个分数都高于 0.70，团队便把实体合并阈值放在这里。

两周后的 10 月 4 日，他们重新发送相同请求。通过 Workers AI 得到的平均概率分别降到 0.47、0.42 和 0.60；直接调用 TypeSafe API，并把版本固定为 `jev-1.13.0`，结果仍然接近，分别是 0.46、0.43 和 0.61。

![同一 Jev 版本在两次校准中的概率变化](/assets/images/13-clef-vs-jev/04-jev-drift.webp)

*图 4：三条校准请求在 9 月 20 日和 10 月 4 日的 Jev 分数。横线是 0.70 合并阈值。图源：[ConstructComputer 原文](https://construct.computer/blog/clef-vs-jev-benchmark/)。*

研究者又为每条请求追加了 20 次调用。60 次调用没有一次够到 0.70，最高只有 0.66。换句话说，原本全部站在合并阈值之上的三个正例，两周后全部跌到了阈值之下。

研究员没有找到分数变化的原因，也没有据此断言 Jev 的整体能力下降。在这次 benchmark 里，Jev 依然没有错误合并两个不同的人；出问题的是召回率——应该合并的记录只有 44% 被合并，剩下的全变成了重复项。

## 实体匹配暴露出的真实差异

五类任务中，实体匹配最能说明“错误的代价并不对称”。

漏掉一次正确匹配，只是在 Agent 记忆里多留一条重复记录，之后还有机会清理；而一旦把两个人、两个项目或两家公司错误合并，它们的历史信息就被永久搅在一起，无法还原。所以这项任务的原则是：宁愿少合并，也绝不乱合并。

测试集包含 15 组应该合并的实体，以及 15 组不应该合并的实体。在 0.70 阈值下：

| 模型         | 应合并案例中成功合并 | 所有合并中判断正确 |
| ---------- | ---------: | --------: |
| Jev        |        44% |      100% |
| Clef       |        80% |       92% |
| Clef-flash |        80% |       80% |

![三个模型在实体匹配任务上的召回率与精确率](/assets/images/13-clef-vs-jev/03-entity-match.webp)

*图 5：左侧是应该合并的记录中实际完成合并的比例，右侧是所有合并结果中正确合并的比例。图源：[ConstructComputer 原文](https://construct.computer/blog/clef-vs-jev-benchmark/)。*

Clef 识别出了这项功能真正想解决的别名：没有重音符号的 `Jose Garcia Marquez` 与带重音符号的姓名得到 0.98；`Stripe, Inc.` 与 `Stripe` 得到 0.98；`I.B.M.` 与 `IBM` 得到 0.99。

它也栽过一次：把 `Acme` 与单独存在的 `Acme Corporation` 判为同一实体，概率 0.78。这个案例本身就有争议，也提醒我们实体匹配的标签并不总是客观标准答案。

Clef-flash 则明显更倾向于说“是”。它把 `Acme Labs Europe` 与 `Acme Labs` 判为同一实体，也把 `pricing` 与 `pricing model` 视为匹配。对那种只决定“要不要跳过一次后台任务”的门控来说，这种积极或许可以接受；但对会永久改写记忆的合并操作，风险就高得多了。

把 Clef 阈值提高到 0.80 后，它合并了 15 个正例中的 10 个，并且没有错误合并。但这个 0.80 是从同一组 30 个案例里选出来的，只适合作为 shadow mode 的起点，不能直接复制到其他系统。

## 成本比较

在 10 月 4 日的测试时点，三种模型的每百万输入 Token 标价分别是：Jev 0.042 美元、Clef-flash 0.09 美元、Clef 0.24 美元，输出均不收费。

只看单价，Clef 是 Jev 的 5.7 倍。但同一批请求经过不同 tokenizer 之后，实际计费的 Token 数并不一样：Jev 平均每次调用算 541 个输入 Token，Clef 系列平均只有 387 个，少了约 28%。

按这批请求的平均长度换算，每百万次决策的成本大约是：

| 模型         | 平均输入 Token/次 | 每百万次决策成本 |
| ---------- | -----------: | -------: |
| Jev        |          541 |    23 美元 |
| Clef-flash |          387 |    35 美元 |
| Clef       |          387 |    93 美元 |

![Jev、Clef 与 Clef-flash 每百万次决策的估算成本](/assets/images/13-clef-vs-jev/07-cost.webp)

*图 6：按 10 月 4 日标价和这批请求的平均 Token 数换算。图源：[ConstructComputer 原文](https://construct.computer/blog/clef-vs-jev-benchmark/)。*

主测试的 1053 次调用，总共只花了大约 5 美分。一个每天做 1000 次类似决策的 Agent，用 Clef 一天约 9 美分，用 Jev 一天约 2 美分。对每天处理数十亿次决策的系统来说，这个差距必须认真算账；但对普通 Agent 而言，错误动作和不可重复判断带来的代价，往往远高于这点差价。

这里还要补充一个发布后的变化。10 月 9 日，Cloudflare 把托管 Clef-flash 的价格从每百万输入 Token 0.09 美元降到 0.038 美元，同时把托管上下文窗口从 64K 调整为 24K。原图和上表保留的是 10 月 4 日 benchmark 时的历史价格，不能代表今天的采购成本。正式选型前，务必重新核对各家的当前价格和限制。

## Clef、Clef-flash、Jev 应该怎么选

这次测试只覆盖了三个模型、五类决策和 117 个合成案例。所以下面这张表，更适合当作搭建自己评测集的起点，而不是一份通用排行榜。

| 需求                 | 更适合先评估            | 原因与边界                                                 |
| ------------------ | ----------------- | ----------------------------------------------------- |
| 合并、删除、发送等高风险动作     | Clef，高阈值运行        | 这组测试中精度和重复性更好，但仍应先进入 shadow mode，关键动作保留人工确认           |
| 只决定是否跳过或降低优先级的后台门控 | Clef-flash        | 调整自己的阈值后接近 Jev 的准确率，模型更小；不要把这项结论直接扩展到不可逆操作            |
| 需要相同输入得到相同概率       | Clef 或 Clef-flash | 这次重复测试中完全一致；仍要用自己的请求持续复测                              |
| 需要读取图片             | Clef 或 Clef-flash | 两者带视觉编码器，Jev 在比较时只支持文本输入                              |
| 需要固定权重、自托管或继续微调    | Clef 或 Clef-flash | 开放权重并采用 Apache 2.0 许可证                                |
| 追求最低调用成本           | 重新按当前价格核算         | Jev 在 10 月 4 日测试时单价最低；Clef-flash 在 10 月 9 日降价后改变了比较结果 |
| 已经稳定运行 Jev         | 不急于直接切换           | Jev 赢下邮件分类，实体匹配也没有发生错误合并；先检查分数漂移和阈值是否仍然有效             |

模型的平均准确率只是选型的一部分。真正应该和业务一起评估的，还有错误动作的代价、概率是否稳定、能否固定版本、输入是否包含敏感数据，以及团队是否有能力持续维护校准集。

## 如何试用 Clef

在 Workers AI 中，可以通过 `@cf/cloudflare/clef` 或 `@cf/cloudflare/clef-flash` 调用模型。下面用一条实体匹配请求展示最小结构：

```bash
curl "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/ai/run/@cf/cloudflare/clef" \
  -X POST \
  -H "Authorization: Bearer $CLOUDFLARE_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "clef",
    "state": {
      "entity": { "type": "person", "name": "Dr. Priya Shah" }
    },
    "questions": {
      "match": {
        "type": "choice",
        "instructions": "Which candidate is the same entity?",
        "criteria": {
          "node-priya": "Priya Shah (person)",
          "none": "No candidate is clearly the same entity"
        }
      }
    }
  }'
```

如果从 Jev 迁移，接口层的最小改动确实可能只有一个模型 ID。但读取结果时要用各选项对应的 `probabilities`，不要假设不同模型的 `confidence` 字段含义相同。

更不要把原来的 0.70、0.80 或 0.85 阈值原样搬过去。先准备一小组同时包含正常案例和边界案例的标注数据，重复调用几次，再让新模型在 shadow mode 里观察真实流量。等概率分布、误报代价和漏报代价都摸清楚了，再定生产阈值。

这次 benchmark 只有 117 个合成案例，标签也还没经过第二位人工复核。它没有证明 Clef 在所有任务上都优于 Jev，也没有测出能代表生产 Worker 的端到端延迟。

它真正留下的，是一套可复用的方法：不要只看公开排行榜，也不要因为两个模型共用一套 API，就把它们当成同一把尺子。拿自己的提示词、自己的边界案例、自己的错误成本去测，才知道一个决策模型配不配进入 Agent 的真实执行路径。

**参考资料**：[ConstructComputer benchmark 原文](https://construct.computer/blog/clef-vs-jev-benchmark/) · [完整研究论文](https://construct.computer/blog/clef-vs-jev-benchmark/compatible-apis-incompatible-thresholds.pdf) · [Cloudflare Clef 发布说明](https://blog.cloudflare.com/clef-decision-models/) · [Cloudflare Clef 模型文档](https://developers.cloudflare.com/workers-ai/models/clef/) · [Clef 模型卡](https://huggingface.co/Cloudflare/clef) · [10 月 9 日价格与托管配置更新](https://developers.cloudflare.com/changelog/post/2026-10-09-clef-omni-workers-ai/)
