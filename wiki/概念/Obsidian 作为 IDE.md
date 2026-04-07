---
created: 2026-04-03
tags:
  - concept
  - tool
---

# Obsidian 作为 IDE

Obsidian 被用作 [[LLM 知识库]] 的"前端"——阅读和浏览层。LLM 负责所有撰写和维护工作，Obsidian 提供可视化界面。

## 在流水线中的角色

```
raw/ → [LLM 编译] → wiki/ → [在 Obsidian 中浏览]
```

Obsidian 展示：
1. **原始数据** — 摄入的原始文档
2. **编译后的 wiki** — 概念文章、摘要、双向链接
3. **衍生可视化** — Marp 幻灯片、matplotlib 图片、Dataview 动态视图等

## 常用插件

| 插件 | 用途 |
|---|---|
| Web Clipper | [[数据摄入]] 时将网页文章转为 `.md` |
| Marp | 将 Markdown 渲染为幻灯片演示 |
| Dataview | 动态查询和可视化 wiki 结构 |
| 本地图片 | 渲染 raw/ 中下载的图片 |

## 为什么用 Obsidian

LLM 的输出是 Markdown 文件，Obsidian 是 Markdown 的天然最优阅读器，提供：
- 双向链接图谱视图
- 全文搜索
- 图片和图表嵌入
- 幻灯片渲染（Marp）

## 相关概念

- [[LLM 知识库]] — Obsidian 所服务的系统
- [[数据摄入]] — Obsidian Web Clipper 的用武之地
- [[LLM 输出格式]] — Marp 幻灯片是 Obsidian 渲染的输出格式之一
