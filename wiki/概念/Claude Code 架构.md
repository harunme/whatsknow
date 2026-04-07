---
created: 2026-04-07
tags:
  - concept
  - Claude-Code
  - architecture
---

# Claude Code 架构

Claude Code 是 Anthropic 生产的 AI 编程 Agent，其架构是一套完整的 Agent Harness 实现，《御舆》书对此做了 42 万字的深度剖析。

## 整体架构：舆与辕

```
舆（Agent Harness）
  ├── 辕（对话循环）— while(true) 异步生成器主循环
  ├── 辐（工具系统）— 50+ 工具 × 12 类
  ├── 軎辖（权限管线）— 四阶段权限模式
  └── 舆身（核心系统）— 配置/记忆/上下文/钩子
```

## Part 1: 基础篇

| 章节 | 核心内容 |
|---|---|
| 01 范式转移 | Copilot → Claude Code 演进；Harness 五大设计原则；Bun+React/Ink+Zod v4 技术栈 |
| 02 对话循环 | `while(true)` 异步生成器主循环；五种 yield 事件；十种终止原因；QueryDeps 依赖注入 |
| 03 工具系统 | Tool 五要素协议；buildTool 故障安全工厂；45+ 工具 × 12 类；并发分区贪心算法 |
| 04 权限管线 | 四阶段管线；五种权限模式谱系；Bash 规则匹配；推测性分类器 2 秒 Promise.race |

## Part 2: 核心系统

| 章节 | 核心内容 |
|---|---|
| 05 配置与设置 | 六层配置优先级链；合并规则；安全边界与供应链攻击防御；双层功能门控 |
| 06 记忆系统 | 四种封闭式记忆类型；"只保存无法推导的信息"；MEMORY.md 索引；Fork 记忆机制 |
| 07 上下文管理 | 有效窗口公式；四级渐进压缩（Snip→MicroCompact→Collapse→AutoCompact）；断路器模式 |
| 08 钩子系统 | 五种 Hook 类型；26 个生命周期事件；JSON 响应协议；六层优先级；三层安全机制 |

## Part 3: 高级模式

| 章节 | 核心内容 |
|---|---|
| 09 子智能体与 Fork | 三种 Agent 来源；四种内置 Agent；Fork 字节级上下文继承；递归 Fork 防护 |
| 10 协调器模式 | Coordinator-Worker 双重门控；"只编排不执行"约束；四种寻址模式；四阶段工作流 |
| 11 技能系统 | 11 个核心技能；SKILL.md frontmatter；三级参数替换；分层加载；插件缓存 |
| 12 MCP 集成 | 8 种传输协议；五态连接管理；三段式工具命名；Bridge 双向通信系统 |

## Part 4: 工程实践

| 章节 | 核心内容 |
|---|---|
| 13 流式架构 | QueryEngine 生命周期管理；并发控制；启动优化 160ms→65ms（-59%）；惰性加载 |
| 14 Plan 模式 | "先想后做"哲学；计划文件三层恢复策略；本地调度与远程触发 |
| 15 构建 Harness | 六步实现路线图；循环依赖解决方案；四层可观测性体系；安全威胁模型 |

## 关键设计原则

1. **只保存无法推导的信息** — 记忆系统的核心原则，避免冗余存储
2. **"先想后做"** — Plan 模式要求 Agent 先规划再执行
3. **默认不信任** — 权限管线从设计一开始假设 Agent 会犯错
4. **完全可观测** — 思考轨迹、Token 消耗、工具调用全程可回溯

## 与 [[Agent Harness]] 的关系

Claude Code 是 Harness 的最佳产品案例：
- 权限管线 → [[Agent Harness]] 模块 1（沙箱隔离）+ 模块 3（人类介入）
- 记忆系统 → [[Agent Harness]] 模块 5（断点续跑）
- 上下文管理 → [[Agent Harness]] 模块 2（预算控制）
- 钩子系统 → [[Agent Harness]] 模块 4（可观测性）+ 模块 7（扩展点）

## 与 [[Claude Code Skills 工程]] 的关系

Skills（技能插件）是 Claude Code 的扩展机制：
- Ch11 专门分析 11 个核心技能和 SKILL.md frontmatter
- 三级参数替换 + 分层加载 + 插件缓存

## 相关概念

- [[Agent Harness]] — Claude Code 架构是 Harness 的完整产品实现
- [[Claude Code Skills 工程]] — Skills 是 Claude Code 的插件系统
- [[Claude Code Skills 实战教程]] — Skills 的 5 步实战构建法
- [[LLM 知识库]] — Claude Code 是本 wiki 编译所用的核心工具
