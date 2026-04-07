---
created: 2026-04-07
source: "[[raw/想让Agent自主不间断干活？你还需要Harness。]]"
tags:
  - summary
  - source
---

# Agent Harness — 摘要

**来源：** [[@KKaWSB]](https://x.com/KKaWSB/status/2040264824362500572) | 2026-04-04

## Summary

Harness 是 2026 年 AI 圈最热话题，被 Anthropic、OpenAI、Martin Fowler、LangChain 共同讨论。核心理念：AI Agent 是发动机提供动力，Harness 是方向盘+护栏+仪表盘决定智能能否被可靠使用。LangChain 用真实数据证明：同一 Agent，换了 Harness 后从基准测试的前 30 开外冲到前 5（52.8% → 66.5%）。一个完整 Harness 包含 7 大模块（沙箱隔离、预算控制、人类介入、可观测性、断点续跑、错误处理、工具框架），遵循 5 条设计原则（默认不信任、完全可观测、预算铁律、人类在回路、为失败而设计）。与 Agent 框架（解决"智能"问题）的根本区别在于 Harness 解决"工程"问题。

## Key Points

1. **为什么现在火**：行业从"AI Agent 能不能干活"转向"能不能天天稳定干活"，第二个问题靠工程化而非模型能力
2. **LangChain 实证**：同一模型换 Harness，52.8% → 66.5%，前 30 开外冲到前 5
3. **7 大模块**：
   - 沙箱隔离（Docker 容器，执行完销毁）
   - 预算控制（每任务上限 + 每天上限 + 实时监控 + 超时停止）
   - 人类介入通道（关键操作审批，Agent 拿不准时暂停喊人）
   - 完整可观测性（思考轨迹 + 工具调用 + Token 消耗 + 出错回溯）
   - 状态持久化（JSON 格式进度文件，断点续跑）
   - 错误处理（自动识别错误类型 + 重试策略）
   - 工具调用框架（统一接入 + 参数校验 + 超时控制）
4. **5 条设计原则**：默认不信任 / 完全可观测 / 预算铁律 / 人永远在回路 / 为失败而设计
5. **Harness vs 框架**：Agent 框架是蓝图和建材（LangChain/CrewAI）；Harness 是工厂车间（沙箱/监控/预算/人）
6. **Vercel 反直觉发现**：工具从 15 个减到 2 个，编程 Agent 准确率 80% → 100%

## Connections

- 参见 [[Agent Harness]] — 概念文章，深入解读 7 大模块和 5 条设计原则
- 参见 [[Claude Code 架构 — 摘要]] — 御舆书将 Claude Code 作为 Harness 的最佳案例来分析
- 与 [[Transformer 架构]] 对比：Transformer 解决模型架构问题；Harness 解决模型之上的工程可靠性问题
- 相关：[[LLM 知识库]] — 知识库系统本身也是一种 Harness：对 LLM 的上下文、工具、预算进行约束

## Sources

- [[raw/想让Agent自主不间断干活？你还需要Harness。]]

## Last Updated

2026-04-07
