---
created: 2026-04-07
source: "[[raw/实战教学从0到1写出一个你自己的Skill]]"
tags:
  - summary
  - source
---

# Claude Code Skills 实战教程 — 摘要

**来源：** [[@bozhou_ai]](https://x.com/bozhou_ai/status/2039596877650551056) | 2026-04-02

## Summary

本文是以两个真实案例（AI 日报 Skill + 文章配图 Skill）讲解如何从 0 到 1 构建 Claude Code Skill 的实战教程。Skill 本质是一个文件夹，核心是一个 SKILL.md 文件。5 步构建法：①找到重复劳动（3 次以上可考虑）；②先手动跑通一遍；③把需求用大白话讲给 AI；④跑起来改起来（2-3 轮）；⑤分享到 GitHub/ClawHub。三个推荐安装的元 Skill：find-skills（发现现成 Skill）、skill-creator（自己造）、skill-vetter（安全审查）。核心体会：40 分钟的日报工作变成一句话，AI 日报 Skill 自动抓 20+ 信息源、并行处理分类、渲染成图片。

## Key Points

- **Skill 结构**：一个文件夹 + SKILL.md（核心）+ scripts/（可选）+ references/（可选）
- **Description 字段**：AI 匹配 Skill 的第一道关，一句话说清楚干什么
- **5 步法**：找重复劳动 → 手动跑通 → 讲给 AI → 迭代改错 → 分享（可选）
- **AI 日报 Skill 案例**：20+ 信息源并发抓取 → 多 subagent 并行分类 → Markdown 日报 → HTML 渲染 → 按分类切图发社交媒体
- **文章配图 Skill 案例**：在宝玉的 baoyu-article-illustrator 基础上改进风格策略（把风格决策权交给 AI，而非锁定预设）
- **踩坑记录**：Hacker News 太杂砍掉；截图太长切成多张；GitHub Trending 加 README 验证层提升质量
- **三个元 Skill**：find-skills（技能发现）+ skill-creator（技能创建）+ skill-vetter（安全审查）
- **描述要准、先手动跑通、一个 Skill 解决一个问题** — 三条核心建议

## Connections

- 参见 [[Claude Code Skills 工程]] — 工程方法论，与本文互补（方法论 + 实战结合）
- 参见 [[Claude Code Skills 实战教程]] — 概念文章，深入分析 5 步法细节
- 相关：[[辅助工具]] — Skill 是 LLM 知识库的扩展工具生态

## Sources

- [[raw/实战教学从0到1写出一个你自己的Skill]]

## Last Updated

2026-04-07
