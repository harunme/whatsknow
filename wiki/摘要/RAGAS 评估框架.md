---
created: 2026-04-07
source: "[[raw/RAGAS 论文]]"
tags:
  - summary
  - source
---

# RAGAS 评估框架 — 摘要

**来源：** Arnav Gaikwad et al., University of Munich / IBM Research, 2023 | [arXiv:2309.15217](https://arxiv.org/abs/2309.15217)

## Summary

RAGAS 提出用 LLM 自动化评估 RAG 系统质量的框架，包含三大独立指标：Context Relevance（检索内容与问题的相关性）、Faithfulness（回答对检索内容的忠实程度）、Answer Relevance（回答对问题的实际解决程度）。核心创新是三个指标全部由 LLM 自动打分，无需人工标注 ground truth。RAGAS 后来演变为 ensembleui/ragas 开源项目，集成 LangChain 和 LlamaIndex，新增 Answer Correctness 等指标。这套评估体系让 RAG 系统的检索-生成链路可以在无人工标注的情况下被持续测量和优化。

## Key Points

- **Context Relevance**：LLM 判断检索到的 chunks 与 query 的相关性分数，低分说明检索召回失败（抓了无关内容）
- **Faithfulness**：LLM 检查答案中每个事实是否在上下文中可找到；低分说明模型在瞎编/幻觉
- **Answer Relevance**：LLM 生成追问验证回答与问题的主题相关性；低分说明答非所问
- **无需人工标注**：三个指标全由 LLM 自动打分，降低评估成本
- **针对 RAG 链路设计**：评估的是检索-生成链路的协同质量，而非单纯的生成质量
- **开源实现**：ensembleui/ragas，支持 HuggingFace datasets 格式，与 LangChain/LlamaIndex 集成
- **后续发展**：新增 Answer Correctness（与标准答案对比）

## Connections

- 参见 [[RAG 评估]] — 概念文章，深入讲解三大指标的计算方法和 RAGAS 的后续演进
- 参见 [[RAGFlow — 摘要]] — RAGFlow 内置评估面板，对应 RAGAS Phase 4 评估环节
- 相关：[[注意力机制]] — RAG 的核心也涉及注意力机制在检索和生成中的应用
- 与 [[LLM 原生问答]] 对比：RAGAS 评估的是 RAG 系统质量；LLM 原生问答不依赖 RAG

## Sources

- [[raw/RAGAS 论文]]

## Last Updated

2026-04-07
