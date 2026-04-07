---
created: 2026-04-03
source: "[[raw/Thread by @karpathy]]"
tags:
  - summary
  - source
---

# Thread by @karpathy — 摘要

**来源：** [x.com/karpathy/status/2039805659525644595](https://x.com/karpathy/status/2039805659525644595) | 2026-04-02

## 概述

Andrej Karpathy 描述了他使用 LLM 构建和维护个人知识库（wiki）的工作流程。核心思路是：原始数据 → LLM 编译 wiki → 通过 CLI 工具进行问答与增强 → 在 Obsidian 中浏览。

## 数据摄入（Data Ingest）

将来源文档（文章、论文、代码库、数据集、图片等）索引到 `raw/` 目录，使用 Obsidian Web Clipper 插件将网页文章转为 `.md` 文件，同时下载所有相关图片到本地供 LLM 引用。

## Wiki 编译（Wiki Compilation）

LLM 将 `raw/` 目录增量"编译"成结构化的 wiki：包含所有数据的摘要、双向 backlinks、按概念分类的文章，并相互链接。

## IDE（Obsidian 作为前端）

用 Obsidian 作为"前端"，可以查看原始数据、编译后的 wiki 以及衍生可视化（Marp 幻灯片等）。重要的是，LLM 负责撰写和维护 wiki 的所有内容，人类很少直接编辑。

## 问答（Q&A）

当 wiki 足够大时（如 ~100 篇文章、~40 万字），可以向 LLM Agent 提出各种复杂研究问题，LLM 会自主查阅相关文档给出答案。Karpathy 认为不需要复杂的 RAG，LLM 自动维护索引和摘要已足够应对这一规模。

## 输出（Output）

LLM 输出的形式可以是 Markdown 文件、Marp 幻灯片、matplotlib 图片等，均可在 Obsidian 中浏览。输出结果常常被归档回 wiki，进一步扩充知识库——探索与查询始终在积累。

## 检查与整理（Linting）

LLM 定期对 wiki 进行"健康检查"：发现数据不一致之处、补全缺失信息（借助网络搜索）、找出有趣的联系供撰写新文章参考，从而不断提升数据的完整性和一致性。

## 辅助工具（Extra Tools）

根据需求开发定制工具，例如用 vibe coding 快速搭建一个简单 wiki 搜索引擎，提供 Web UI 直接使用，也通过 CLI 交给 LLM 作为大型查询的工具。

## 进一步探索（Further Explorations）

随着 wiki 规模增长，自然的方向是：合成数据生成 + 微调——将知识从上下文窗口迁移到模型权重中。

## 核心论点

> raw 数据 → LLM 编译 wiki → LLM 操作 CLI 工具 → 在 Obsidian 中浏览。Wiki 归 LLM 所有，人类几乎不需要手写。

## 本 wiki 中的应用示例

Karpathy 的工作流非常适合用来研究机器学习论文。例如：

- `raw/Attention Is All You Need.pdf` — Transformer 原论文，可摄入为 `raw/` 中的原始材料
- 通过本 wiki 的 [[Wiki 编译|编译流程]] 生成 [[Attention Is All You Need — 摘要]] 及 [[Transformer 架构]] 等概念文章
- 所有 [[Transformer 架构]] 相关文章均已互相链接，形成关于注意力机制的知识网络

## 相关概念

- [[LLM 知识库]]
- [[Wiki 编译]]
- [[Obsidian 作为 IDE]]
- [[LLM 原生问答]]
- [[LLM 输出格式]]
- [[Wiki 检查与整理]]
- [[合成数据生成]]
