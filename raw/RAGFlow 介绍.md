# RAGFlow

> - **官网**: https://ragflow.io
> - **GitHub**: https://github.com/infiniflow/ragflow
> - **标签**: #RAG #开源 #平台

---

## 是什么

RAGFlow 是 **Inffiniflow（录梦科技）** 开源的 production-ready RAG 平台，定位是端到端的 RAG 解决方案，不只是向量数据库。

---

## 核心架构

```
文档上传
    ↓
DeepDoc 解析（OCR + 布局分析 + 表格理解）
    ↓
智能分块（可人工审查调整）
    ↓
Embedding 向量化
    ↓
向量数据库（内置/可外接）
    ↓
LLM 生成（支持多后端）
    ↓
Web UI / API
```

---

## 核心功能

### 1. DeepDoc 解析引擎

内置文档深度解析能力，解决传统 RAG 的痛点：

- **OCR**：扫描件、图片中的文字识别
- **布局分析**：识别 PDF 多栏、标题、正文区域
- **表格理解**：解析复杂表格，保留行列结构（不是把表格拉成一维文本）
- **多语言**：中英文混排文档

### 2. 可视化分块管理

- Web UI 展示分块结果，可人工审查
- 点击块即可合并/分割/修改
- 解决纯自动分块质量不可控的问题

### 3. 多模态支持

- 图片中的文字 → 向量化检索（基于 CLIP）
- 表格 → 理解结构后再切分，不是暴力拍平

### 4. LLM 后端

支持多种接入方式：
- OpenAI API
- Claude API
- 本地 Ollama
- 其他兼容 OpenAI 格式的 API

---

## 对比 ChromaDB / LangChain

| | ChromaDB | LangChain/LlamaIndex | RAGFlow |
|---|---|---|---|
| 定位 | 向量数据库 | 开发框架 | 完整平台 |
| 文档解析 | ❌ 需自己处理 | ⚠️ 基础支持 | ✅ 内置 DeepDoc |
| 分块管理 | ❌ 代码控制 | ⚠️ 需写代码 | ✅ Web 可视化 |
| 部署难度 | ⭐ 简单 | ⭐⭐ 中等 | ⭐⭐⭐ Docker |
| 适用场景 | 快速原型 | 定制开发 | 生产环境+复杂文档 |

---

## 适用场景

- ✅ 复杂 PDF（多栏、图表、扫描件）
- ✅ 需要人工审核分块质量
- ✅ 快速搭建生产级 RAG 而不写大量代码
- ✅ 团队需要可视化运营界面

---

## 不适用场景

- ❌ 追求极致定制化（用 LangChain 更灵活）
- ❌ 数据量极小（用 ChromaDB 更轻量）
- ❌ 有严格数据主权要求（RAGFlow 有云端组件）

---

## 快速体验

```bash
git clone https://github.com/infiniflow/ragflow.git
cd ragflow
docker compose up -d
# 访问 http://localhost:9380
```

---

## 与 RAG 学习的关联

RAGFlow 集成了 RAG 完整链路：

- **DeepDoc** → 对应 Phase 2 文档分块
- **向量检索** → 对应 Phase 3 检索策略
- **评估面板** → 对应 Phase 4 RAGAS 评估
