---
created: 2026-04-07
source: "[[raw/lintsinghuaclaude-code-book 御舆]]"
tags:
  - summary
  - source
---

# 御舆：Claude Code 架构深度解析 — 摘要

**来源：** [lintsinghua/claude-code-book](https://github.com/lintsinghua/claude-code-book) | 2026-03 | 42万字

## Summary

《御舆》是一本 42 万字的 Claude Code 架构深度剖析书，从《考工记》"舆载辕行"出发，将 Claude Code 的 Agent Harness 架构比作马车之舆：对话循环为辕定方向，工具系统为辐传动力，权限管线为軎辖为约束。书分 4 部分共 15 章：①基础篇（范式转移、对话循环、工具系统、权限管线）；②核心系统篇（配置、记忆、上下文、钩子）；③高级模式篇（子智能体/Fork、协调器、技能插件、MCP 集成）；④工程实践篇（流式架构、Plan 模式、构建自己的 Harness）。含 139 张 Mermaid 架构图、100+ 中英术语、89 个功能标志、50+ 注册工具分析。是第一本系统讲解 Agent Harness 工程原理的书。

## Key Points

- **Part 1 基础篇**：范式转移（Copilot→Claude Code）+ `while(true)` 异步生成器对话循环 + 5 要素 Tool 协议 + 四阶段权限管线
- **Part 2 核心系统**：六层配置优先级链 + 四种封闭式记忆类型（只保存无法推导的信息）+ 四级渐进上下文压缩（Snip→MicroCompact→Collapse→AutoCompact）+ 26 个生命周期事件钩子
- **Part 3 高级模式**：Fork 字节级上下文继承 + Coordinator-Worker 双重门控编排 + 11 个核心技能 + 8 种 MCP 传输协议
- **Part 4 工程实践**：并发控制 + 启动优化 160ms→65ms（-59%）+ "先想后做" Plan 模式 + 六步构建自己的 Harness 路线图
- **Appendix**：架构地图（16 模块/依赖树/6条数据流） + 工具完整清单（50+工具×12类） + 功能标志速查（89个×13类） + 术语表（100条中英）
- **背景**：2026 年 3 月 npm `@anthropic-ai/claude-code` source map 泄露事件引发 Agent 架构讨论热潮
- **核心价值**：读懂 Claude Code 的设计决策，就拥有了一套可迁移到任何 Agent 框架的心智模型

## Connections

- 参见 [[Claude Code 架构]] — 概念文章，对 4 部分 15 章内容做结构化概述
- 与 [[Agent Harness — 摘要]] 深度关联：Claude Code 是 Harness 的最佳案例，御舆书系统性拆解其架构
- 与 [[Claude Code Skills 工程]] 和 [[Claude Code Skills 实战教程]] 相关：Skills 是 Claude Code 的插件系统，御舆书 Ch11 专门分析
- 相关：[[LLM 知识库]] — Claude Code 正是本 wiki 编译所用的工具

## Sources

- [[raw/lintsinghuaclaude-code-book 御舆]]

## Last Updated

2026-04-07
