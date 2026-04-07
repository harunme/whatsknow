# RAGAS: Retrieval-Augmented Generation Assessment

> RAGAS: Automated Evaluation of Retrieval-Augmented Generation
> - **论文**: https://arxiv.org/abs/2309.15217
> - **作者**: Arnav Gaikwad et al. (University of Munich / IBM Research)
> - **年份**: 2023
> - **标签**: #RAG #评估 #论文

---

## 核心贡献

提出 **RAGAS**（Retrieval-Augmented Generation Assessment），一个用 LLM 自动化评估 RAG 系统质量的框架。

核心观点：RAG 系统有三个需要独立评估的环节——**检索**、**生成**、**整体**，每个环节对应不同的指标。

---

## 三大评估指标

| 指标 | 评估什么 | 用什么信号 |
|------|---------|-----------|
| **Context Relevance** | 检索回来的内容和问题是否相关 | LLM 判断检索块与 query 的相关性 |
| **Faithfulness** | 生成的回答是否忠实于检索到的内容 | LLM 检查答案中每个事实是否在上下文中可找到 |
| **Answer Relevance** | 回答是否真正解决了用户的问题 | LLM 生成追问，验证回答与问题的主题相关性 |

---

## 方法亮点

1. **无需人工标注 ground truth**：三个指标全部由 LLM 自动打分
2. **针对 RAG 链路设计**：不是评估生成质量，而是评估检索-生成链路的协同质量
3. **可复现**：开源实现，支持 HuggingFace datasets 格式输入

---

## Faithfulness 计算示例

```
输入：
  question: "RAG 是什么？"
  context: "RAG 由 Facebook AI 在 2020 年提出，结合检索和生成模块。"
  answer: "RAG 由 Google 在 2019 年提出。"

LLM 评估：
  - "RAG 由 Google 在 2019 年提出" → ❌ NOT SUPPORTED（上下文说是 Facebook/2020）
  - Faithfulness = 0/1 = 0

→ Faithfulness 低 = 答案瞎编，幻觉严重
```

---

## Context Relevance 计算示例

```
输入：
  question: "RAG 是什么？"
  retrieved_docs: ["向量数据库使用 HNSW 索引", "RAG 由 Facebook 在 2020 年提出"]

LLM 评估：
  - "向量数据库使用 HNSW 索引" → 0（和问题无关）
  - "RAG 由 Facebook 在 2020 年提出" → 1（直接相关）
  - Context Relevance = 1/2 = 0.5

→ Context Relevance 低 = 检索不准，召回失败
```

---

## 后续发展

- RAGAS 演变为 [EnsembleUI/ragas](https://github.com/ensembleui/ragas)，持续维护
- 与 LangChain、LlamaIndex 集成
- 新增指标：Answer Correctness（与标准答案对比）

---

## 与 RAG 学习的关系

- **Context Relevance** → 评估 Phase 3 检索环节（RRF、Re-ranking 是否有效）
- **Faithfulness** → 评估 Phase 4 生成环节（有无幻觉）
- **Answer Relevance** → 评估整体 Pipeline 是否真正解决用户问题
