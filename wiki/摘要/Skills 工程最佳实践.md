---
created: 2026-04-07
source: "[[raw/Thread by @chenchengpro]]"
tags:
  - summary
  - source
---

# Skills 工程最佳实践 — 摘要

**来源：** [[@chenchengpro]](https://x.com/chenchengpro/status/2039707154371018930) | 2026-04-02

## Summary

本文总结了构建 Claude Code Skills 的 8 条反直觉工程判断，核心理念是 Skill ≠ 长 Prompt，而是"让 Agent 稳定完成某件事的方法论包"。关键洞见包括：把确定性逻辑（签名计算、渠道映射、错误分类）下沉到脚本而非留在 Agent；SKILL.md 是写给 Agent 的操作手册而非人类文档；多人协作需要四件套交付（入口+业务逻辑+文档+注册）；安全是准入门槛而非加分项；测试时不能让执行体提前知道预期结果；pass@K ≠ pass^k（有能力但不稳定是真实风险）；质量应在中等模型上验证而非仅强模型。

## Key Points

1. **Skill ≠ 长 Prompt** — 核心价值是给 Agent 明确边界：什么顺序做、什么结果停下来
2. **确定性逻辑下沉** — 签名计算、渠道映射、错误分类、重试策略全部进入脚本，Agent 只负责"理解用户想做什么"
3. **SKILL.md 是 Agent 操作手册** — description 写触发场景而非功能简介；主文档做轻索引而非百科全书；细节下沉到 references/ 按需加载
4. **四件套交付** — 新增能力最小交付：可执行入口 + 业务逻辑 + 能力文档 + 总入口注册
5. **安全是门不是分** — 存在硬编码凭证直接归零，不参与加权
6. **Executor/Grader 严格分离** — 执行体不能提前知道 expected outcome，否则 pass rate 虚高
7. **pass@k ≠ pass^k** — 单次成功率 75%，跑 3 次全成功的概率只有 42%
8. **中等模型验证质量** — 强模型会替写得不好的 Skill 兜底，真正的质量看中等模型上是否稳定

## Connections

- 参见 [[Claude Code Skills 工程]] — 概念文章，对 8 条判断做深入解读
- 参见 [[Claude Code Skills 实战教程 — 摘要]] — 对应的实战教程，从 0 到 1 造 Skill
- 相关概念：[[辅助工具]] — Skill 是 LLM 知识库的辅助工具生态核心

## Sources

- [[raw/Thread by @chenchengpro]]

## Last Updated

2026-04-07
