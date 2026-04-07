---
created: 2026-04-07
tags:
  - concept
  - RAG
  - evaluation
---

# RAG 评估

使用 RAGAS 等框架对 RAG（检索增强生成）系统的检索-生成链路进行自动化质量评估。

## RAGAS 三大指标

| 指标 | 评估内容 | 低分意味着 | 如何计算 |
|---|---|---|---|
| **Context Relevance** | 检索到的 chunks 与 query 的相关性 | 检索召回失败，抓了无关内容 | LLM 判断每个 chunk 与 query 的相关性分数 |
| **Faithfulness** | 回答对检索内容的忠实程度 | 模型在瞎编/产生幻觉 | LLM 检查答案中每个事实是否在上下文中可找到 |
| **Answer Relevance** | 回答对问题的实际解决程度 | 答非所问 | LLM 生成追问，验证回答与问题的主题相关性 |

## 三大指标的典型失败模式

```
Context Relevance 低:
  query: "RAG 是什么？"
  retrieved: ["向量数据库使用 HNSW 索引", "RAG 由 Facebook 在 2020 年提出"]
  → 第一条完全不相关 → Context Relevance = 0.5（满分1.0）

Faithfulness 低:
  context: "RAG 由 Facebook 在 2020 年提出"
  answer: "RAG 由 Google 在 2019 年提出"
  → 上下文说的是 Facebook/2020，答案瞎编 → Faithfulness = 0.0

Answer Relevance 低:
  query: "怎么用 RAG 提升问答质量？"
  answer: "RAG 的发展历史..."
  → 回答与问题主题不匹配 → Answer Relevance 低
```

## 为什么不需要人工标注

三个指标全部由 LLM 自动打分：
- **Context Relevance**：LLM 判断 chunk 与 query 的相关程度
- **Faithfulness**：LLM 检查答案中的每个事实陈述是否在上下文中可找到
- **Answer Relevance**：LLM 生成多个追问，验证回答是否覆盖问题主题

这使得 RAG 评估可以在没有 ground truth 答案的情况下持续进行。

## RAGAS 演进

- **2023**：论文 RAGAS 提出三大指标
- **后续**：演变为 [ensembleui/ragas](https://github.com/ensembleui/ragas) 开源项目
- **集成**：LangChain、LlamaIndex 均有官方集成
- **新增指标**：Answer Correctness（与标准答案对比）

## 在 RAG 链路中的位置

```
文档 → 检索 → 生成 → 评估
              ↓
        Context Relevance（检索环节）
        Faithfulness（生成环节）
        Answer Relevance（整体链路）
```

## 相关概念

- [[RAG 流程]] — 评估所在的完整 RAG 链路
- [[RAGAS 评估框架 — 摘要]] — 论文背景和具体计算示例
- [[LLM 原生问答]] — 对比：RAGAS 评估 RAG 系统质量；LLM 原生问答不依赖 RAG
- [[注意力机制]] — RAG 的向量检索（语义相似度）和生成阶段都涉及注意力机制
