---
layout: post
title: "Agent 改了自己的代码，任务怎么接着跑？"
date: 2026-10-27 20:00:00 +0800
categories: [AI, Agent]
tags: [Agent Harness, Pi Durable, Ultimate, 持久化]
description: "从 Ultimate 与 Pi Durable 看代码更新、进程交接和任务恢复如何衔接，以及为什么持久化不能保证外部动作只执行一次。"
---

Agent 如果要修改承载自己的应用，代码写完只是第一步。新代码怎样进入运行中的进程？进程切换时，未完成的任务又从哪里继续？

Ultimate 把这几个问题放在同一套 Coding Agent 架构里处理。它建立在 Pi Durable 之上：Ultimate 管项目扩展、工作区和进程切换，Pi Durable 保存对话与任务状态，供新进程恢复。这里说的“自改造”是修改应用代码和配置，再通过加载机制启用新实现，不涉及模型权重更新。[Ultimate 项目说明](https://github.com/HazAT/ultimate/blob/294c6a5edb26a33027b9ea7d8fed3a6b8debe8c0/README.md)

## 代码改动有不同的生效路径

Ultimate 把项目定制放在 `.ultimate/` 中。因为这些扩展会执行代码，项目需要先被选为可信的本地项目。[项目设置与信任规则](https://github.com/HazAT/ultimate/blob/294c6a5edb26a33027b9ea7d8fed3a6b8debe8c0/docs/settings.md)

不同范围的修改，生效方式也不同：

- 界面扩展和 Web 核心通过 Vite 热更新。
- 服务端扩展保存后会重新装配。已经开始的工具调用继续使用旧实现，之后发起的调用使用新实现。
- 修改 `server/` 或 `shared/` 核心代码，需要调用 `restart`，由 supervisor 切换核心进程。
- `bin/` 中的 supervisor 自身不在这次进程切换范围内，需要人工重新启动 Ultimate。

这些边界来自项目的分层规则。[Ultimate AGENTS.md](https://github.com/HazAT/ultimate/blob/294c6a5edb26a33027b9ea7d8fed3a6b8debe8c0/AGENTS.md#L51-L68)

独立工作区里的代码也不会直接进入运行版本。Ultimate 使用 Git worktree 隔离工作；变更需要经过项目配置的检查与集成流程。重启时，系统还会检查运行中的检出是否干净、是否位于已经落地的提交，不满足条件就拒绝切换。[工作区说明](https://github.com/HazAT/ultimate/blob/294c6a5edb26a33027b9ea7d8fed3a6b8debe8c0/docs/worktrees.md) · [`restart` 实现](https://github.com/HazAT/ultimate/blob/294c6a5edb26a33027b9ea7d8fed3a6b8debe8c0/extensions/self/server.ts#L40-L63)

## 核心进程交接时发生什么

核心代码需要重启时，supervisor 先做 TypeScript 类型检查，再启动候选进程并等待它完成准备。旧核心把重启目标写入持久化状态后，释放存储控制权；候选核心取得控制权并报告就绪，旧核心才退出。

如果候选核心准备失败，旧核心继续运行；如果激活失败，supervisor 会尝试让旧核心重新接管。这一过程减少了切换过程中“旧进程已退出、新进程却没能工作”的风险，但它不能替代对升级兼容性的设计。[supervisor 的 `swap()`](https://github.com/HazAT/ultimate/blob/294c6a5edb26a33027b9ea7d8fed3a6b8debe8c0/bin/ultimate.ts#L349-L455) · [`restart` 的持久记录](https://github.com/HazAT/ultimate/blob/294c6a5edb26a33027b9ea7d8fed3a6b8debe8c0/extensions/self/server.ts#L64-L127)

新核心打开同一个 SQLite 状态目录，重新装配模型和扩展，取回根 conversation 并调用 `resume()`。Pi Durable 记录的 conversation、工具结果和任务检查点，为恢复提供执行位置。[Ultimate harness](https://github.com/HazAT/ultimate/blob/294c6a5edb26a33027b9ea7d8fed3a6b8debe8c0/server/harness.ts#L222-L285) · [Pi Durable：恢复会话](https://github.com/earendil-works/pi/blob/v1.4.0/packages/durable/README.md#persist-and-resume)

这里有两种需要一起恢复的状态：Git worktree 描述代码版本，持久化存储描述 Agent 执行到哪里。只有代码而没有任务状态，Agent 不知道从哪继续；只有任务状态而没有兼容的代码，也未必能正确解释旧检查点。

## 持久化不等于外部动作只执行一次

Pi Durable 会先提交工具调用意图，再执行工具。进程中断后，只有声明 `replay: "safe"` 的工具才会自动重跑；其他中断调用会把 `interrupted` 结果交给模型。`requestId` 能避免同一输入被重复入队，但不能保证输入触发的外部动作只执行一次。[Pi Durable：工具与恢复](https://github.com/earendil-works/pi/blob/v1.4.0/packages/durable/README.md#tools)

原因是框架的存储事务只覆盖框架内部状态。文件写入、进程操作和网络请求发生在事务之外。即使框架知道某个工具调用开始过，也不一定知道外部系统是否已经完成操作。恢复时重试，就可能再次产生副作用。Ultimate 的 durable event 文档也区分了事务内更新与事务外副作用。[Ultimate：durable events](https://github.com/HazAT/ultimate/blob/294c6a5edb26a33027b9ea7d8fed3a6b8debe8c0/docs/extensions.md#durable-events)

因此，可靠地恢复外部写入，还要由具体工具提供幂等键、结果查询，必要时加入人工审批。持久化解决“执行到哪里”的记录问题；外部动作是否已完成，仍要由应用与外部系统共同确认。

## 把它当成架构参考，而不是现成保证

Ultimate 的端到端测试覆盖了重启后接续同一 transcript、类型检查失败时保留旧进程、激活失败回退，以及异常退出后的恢复。这说明仓库作者为这些路径编写了测试；本文没有安装 Ultimate，也没有自行运行测试。[supervisor 测试](https://github.com/HazAT/ultimate/blob/294c6a5edb26a33027b9ea7d8fed3a6b8debe8c0/test/e2e/supervisor.test.ts#L8-L165)

项目 README 说明 Ultimate 是个人项目，不承诺稳定迁移或用户支持；Pi Durable 1.4.0 README 标注为实验性，API 可能变化。[Ultimate README](https://github.com/HazAT/ultimate/blob/294c6a5edb26a33027b9ea7d8fed3a6b8debe8c0/README.md) · [Pi Durable 1.4.0 README](https://github.com/earendil-works/pi/blob/v1.4.0/packages/durable/README.md)

所以，这套实现更适合作为 Agent Harness 的设计参考。值得跟踪的问题包括：旧检查点能否被新代码理解，进程交接失败时如何回退，以及每个外部写入工具在中断后如何判定是否重试。要让 Agent 在重启后继续工作，代码版本和执行状态都得有明确的恢复路径。
