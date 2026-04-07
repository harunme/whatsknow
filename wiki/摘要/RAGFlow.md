---
created: 2026-04-07
source: "[[raw/RAGFlow 介绍]]"
tags:
  - summary
  - source
---

# RAGFlow — 摘要

**来源：** Infiniflow（录梦科技）| [ragflow.io](https://ragflow.io) | [GitHub](https://github.com/infiniflow/ragflow)

## Summary

RAGFlow 是 Infiniflow 开源的 production-ready RAG 平台，提供端到端完整链路而非仅仅向量数据库。核心能力：DeepDoc 解析引擎（OCR + 布局分析 + 表格理解）、可视化分块管理（Web UI 审查/合并/分割 chunks）、多模态支持（CLIP 处理图片文字）、多 LLM 后端（OpenAI/Claude/Ollama）。对比 ChromaDB（轻量向量库）和 LangChain（开发框架），RAGFlow 定位是"完整平台"，适合复杂 PDF（多栏/图表/扫描件）、需要人工审核分块质量、快速搭建生产级 RAG 的场景。

## Key Points

- **DeepDoc 解析引擎**：OCR 扫描件识别、多栏布局分析、表格结构保留（不是拍成一维文本）、中英文混排
- **可视化分块管理**：Web UI 展示 chunks，点击即可合并/分割/修改，解决纯自动分块质量不可控问题
- **多模态支持**：图片文字向量化检索（CLIP）、表格结构理解后切分
- **多 LLM 后端**：OpenAI API、Claude API、本地 Ollama、其他 OpenAI 兼容格式
- **对比定位**：ChromaDB（向量库/快速原型）< LangChain（开发框架/定制）< RAGFlow（完整平台/生产环境）
- **适用场景**：复杂 PDF、需要人工审核分块、团队可视化运营界面
- **不适用**：追求极致定制化、数据量极小、有严格数据主权要求
- **快速启动**：`docker compose up -d` → 访问 http://localhost:9380

## Connections

- 参见 [[RAG 流程]] — 概念文章，详细拆解 RAGFlow 的端到端链路各阶段
- 与 [[RAGAS 评估框架 — 摘要]] 结合，构成完整的 RAG 构建-评估闭环
- 相关：[[注意力机制]] — RAGFlow 的向量检索和生成阶段都涉及注意力机制的应用

## Sources

- [[raw/RAGFlow 介绍]]

## Last Updated

2026-04-07
