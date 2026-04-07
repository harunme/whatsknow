---
created: 2026-04-07
tags:
  - concept
  - Claude-Code
  - engineering
---

# Claude Code Skills 工程

构建 Claude Code Skills 的工程方法论，核心理念是 Skill ≠ 长 Prompt，而是"让 Agent 稳定完成某件事的方法论包"。

## 核心原则

### 1. Skill ≠ 长 Prompt

Skill 的价值不是让模型"知道更多"，而是让模型在明确边界里知道：
- 该按什么**顺序**做
- 什么结果必须**停下来**

传统做法把所有逻辑都塞进 prompt → 模型容易发散，结果不稳定。
正确做法：把确定性逻辑下沉到脚本，Agent 只负责"理解用户想做什么"。

### 2. 确定性逻辑下沉

以下内容**不应**留在 Agent 里：
- 签名计算
- 渠道映射
- 错误分类
- 重试策略
- 跨能力数据格式

这些全部下沉到 `scripts/` 目录的脚本中。

### 3. SKILL.md 是 Agent 的操作手册

写给 Agent 看 ≠ 写给人看：

| 字段 | 写给人类 | 写给 Agent |
|---|---|---|
| description | 功能简介 | **触发场景**（AI 据此匹配 Skill） |
| 主文档 | 百科全书 | 轻索引 + references/ 按需加载 |
| 细节 | — | 下沉到 references/ |

> "文档写出来了，不等于 Agent 学会了。"

### 4. 四件套交付

多人协作时，新增能力最小交付必须四件齐套：
1. **可执行入口**（Agent 的触发点）
2. **业务逻辑**（scripts/ 中的确定性代码）
3. **能力文档**（references/ 中的详细说明）
4. **总入口注册**（让 Agent 能找到这个 Skill）

### 5. 安全是准入门槛

存在硬编码凭证的 Skill，其他维度全满分也直接归零。
安全不参与加权，是**准入门槛**。

### 6. Executor/Grader 严格分离

执行体（executor）不能提前知道 expected outcome。
如果把 expected outcome 传给子 Agent，pass rate 会虚高——模型在"迎合评测"而非执行 Skill。

### 7. pass@k ≠ pass^k

单次成功率 75%，跑 3 次全部成功 = 0.75³ = 42%。
"有能力但不稳定"是需要被识别的状态——这种 Skill 不敢在生产环境托付。

### 8. 中等模型验证质量

强模型会替写得不好的 Skill 兜底。
真正的质量应在**中等模型**上验证。如果只有最强模型才能跑通，这个高分是借来的。

## 架构图

```
SKILL.md
  ├── description（触发场景）← Agent 据此匹配
  ├── 操作指南（轻索引）
  └── references/
        ├── detail-A.md
        └── detail-B.md

scripts/
  ├── deterministic-logic.sh  ← 确定性逻辑下沉
  └── retry-strategy.py
```

## 相关概念

- [[Claude Code Skills 实战教程]] — 5 步构建法 + 真实案例
- [[辅助工具]] — Skill 是辅助工具生态的核心
- [[Agent Harness]] — Skill 的执行也需要 Harness 的保护（预算控制、错误处理）
- [[Wiki 编译]] — Skill 本身也是一种 wiki 的构建工具
