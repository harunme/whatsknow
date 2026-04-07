---
created: 2026-04-07
tags:
  - concept
  - RAG
---

# RAG 流程

Retrieval-Augmented Generation 的完整端到端处理链路，RAGFlow 是该流程的生产级实现代表。

## 完整链路

```
文档上传
    ↓
DeepDoc 解析（OCR + 布局分析 + 表格理解）
    ↓
智能分块（可人工审查/合并/分割）
    ↓
Embedding 向量化
    ↓
向量数据库（检索）
    ↓
LLM 生成
    ↓
输出（Web UI / API）
```

## 各阶段详解

### 1. 文档解析（DeepDoc）

解决传统 RAG 的痛点：
- **OCR**：扫描件、图片中的文字识别
- **布局分析**：识别 PDF 多栏、标题、正文区域
- **表格理解**：解析复杂表格，保留行列结构（不是拍成一维文本）
- **多语言**：中英文混排文档

### 2. 智能分块

- 纯自动分块质量不可控：语义完整性被破坏
- 可视化分块：Web UI 展示 chunks，点击即可合并/分割/修改
- 多模态：图片中的文字 → CLIP 向量化；表格 → 理解结构后再切分

### 3. 向量检索

- 将 chunks 转换为 embedding 向量，存入向量数据库
- 查询时：query → embedding → 相似度检索 → Top-K chunks
- 与 [[注意力机制]] 的关联：语义相似度计算底层依赖注意力机制

### 4. LLM 生成

- 将 Top-K chunks 作为上下文，LLM 生成答案
- 支持多后端：OpenAI / Claude / Ollama
- 需要评估生成质量 → [[RAG 评估]]

## RAGFlow 的优势

| 对比项 | ChromaDB | LangChain/LlamaIndex | RAGFlow |
|---|---|---|---|
| 文档解析 | ❌ 需自己处理 | ⚠️ 基础支持 | ✅ DeepDoc 内置 |
| 分块管理 | ❌ 代码控制 | ⚠️ 需写代码 | ✅ Web 可视化 |
| 部署难度 | ⭐ 简单 | ⭐⭐ 中等 | ⭐⭐⭐ Docker |
| 适用场景 | 快速原型 | 定制开发 | 生产环境+复杂文档 |

## 与 [[注意力机制]] 的关联

RAG 流程中注意力机制在两个地方发挥作用：
1. **Embedding 模型**：现代 embedding 模型（如 BERT-based）使用注意力机制理解文本语义
2. **LLM 生成阶段**：RAG 将检索结果作为上下文喂给 LLM，LLM 内部使用注意力机制整合检索信息和生成内容

## 相关概念

- [[RAG 评估]] — RAG 流程第 5 步：质量评估
- [[注意力机制]] — RAG 的 embedding 和生成阶段都依赖注意力
- [[RAGFlow — 摘要]] — RAG 流程的生产级开源实现
- [[LLM 原生问答]] — 对比：RAG 是检索增强生成；LLM 原生问答不依赖外部检索
