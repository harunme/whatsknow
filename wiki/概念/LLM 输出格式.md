---
created: 2026-04-03
tags:
  - concept
  - output
---

# LLM 输出格式

[[LLM 知识库]] 可以根据查询需求，以多种格式渲染输出，而非仅仅输出纯文本。

## 支持的格式

| 格式 | 说明 | 浏览方式 |
|---|---|---|
| Markdown `.md` | 文章、摘要、问答答案 | Obsidian |
| Marp 幻灯片 | Markdown 编写的演示文稿 | Obsidian（Marp 插件） |
| Matplotlib 图片 | 图表、数据可视化 | Obsidian（图片嵌入） |
| HTML | 网页输出 | 浏览器 |
| 纯文本 | 终端输出 | CLI |

## Marp 幻灯片

Marp 是一种基于 Markdown 的幻灯片格式。LLM 直接生成 Marp 语法的幻灯片，Obsidian 渲染为可放映的演示文稿。适用场景：

- 将研究综合为汇报材料
- 向他人展示发现
- 自我检视结构化知识

## 结果归档回 wiki

关键模式：查询输出**被归档回 wiki**。这意味着每次探索都为知识库增加内容——wiki 随时间自我增强，而非仅作为静态档案。

## 相关概念

- [[Obsidian 作为 IDE]] — Obsidian 渲染大多数输出格式
- [[Wiki 编译]] — 生成 wiki 内容的编译流水线
- [[LLM 原生问答]] — 触发输出生成的查询
