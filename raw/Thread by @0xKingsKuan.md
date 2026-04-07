---
title: "Thread by @0xKingsKuan"
source: "https://x.com/0xKingsKuan/status/2041173514355372356"
author:
  - "[[@0xKingsKuan]]"
published: 2026-04-04
created: 2026-04-07
description: "Karpathy 亲测！7分钟用 AI 搭「第二大脑」，扔掉 Notion 后效率直接起飞！ 3 个文件夹 + 1 个文本文件，AI 自动帮你整理笔记、建立关联、写总结、查矛盾。 零成本、零学习曲线，41 万人收藏的爆火方法来了： 第 1 步：建 3 个文件夹（2 分钟）"
tags:
  - "clippings"
---
**币世王 | TermMax** @0xKingsKuan 2026-04-04

Karpathy 亲测！7分钟用 AI 搭「第二大脑」，扔掉 Notion 后效率直接起飞！

3 个文件夹 + 1 个文本文件，AI 自动帮你整理笔记、建立关联、写总结、查矛盾。

零成本、零学习曲线，41 万人收藏的爆火方法来了：

第 1 步：建 3 个文件夹（2 分钟）

在电脑任意位置新建一个项目文件夹，里面建 3 个子文件夹：

• raw/ → 原始垃圾桶（所有乱七八糟的笔记、文章、截图全扔这里）

• wiki/ → AI 整理后的结构化知识库（人类可读）

• outputs/ → 你问 AI 的所有答案、报告、决策（以后复用）

🔻

第 2 步：把你所有东西全扔进 raw/（10 分钟）

文章、PDF、会议纪要、X 线程、YouTube 字幕、竞品分析、读书笔记……

不要整理！不要改名！ 越乱越好，这就是 AI 的原材料。

🔻

第 3 步：写一个「AI 指令文件」（5 分钟）

在根目录新建 CLAUDE.md（或者 AGENTS.md），复制下面这段模板进去就行：

\# CLAUDE.md - Personal Knowledge Base Schema

\# (Nick Spisak + Karpathy 极简版 · 2026)

\## 1. 本知识库的目的（必须写清楚）

这是一个【你的主题，例如：AI 工程、个人成长、电商运营】的个人第二大脑。

目标是把 raw/ 里所有混乱的原始材料（文章、笔记、截图、会议记录、PDF、X 线程等）自动整理成结构化、可搜索、能自我迭代的 Wiki。

\## 2. 核心原则（永远遵守）

\- 保持 super simple and flat（Karpathy 原话）

\- 只有 .md 文件，没有数据库、插件、Notion、Obsidian 复杂结构

\- 人类能读懂，AI 也能完美理解

\- 所有知识最终落在 wiki/ 文件夹

\- outputs/ 只放问答结果和决策报告

\## 3. 文件夹结构（AI 必须严格遵守）

\- raw/ → 所有原始材料（AI 只读不改）

\- wiki/ → AI 整理后的结构化知识库（核心）

\- outputs/ → 所有问答、briefing、决策输出

\## 4. 编译 Wiki 的规则（当我说“compile the wiki”时执行）

1\. 先读取 raw/ 里\*\*所有\*\*文件内容。

2\. \*\*必须先创建/更新\*\* wiki/INDEX.md：

\- 漂亮的目录表格

\- 每个主题的简要描述 + 链接

\- 全局关键洞见总结

3\. 为每个\*\*主要主题\*\*创建一个独立 .md 文件（文件名用 kebab-case，例如 ai-agent-architecture.md）

4\. 每篇文章的标准结构：

\`\`\`markdown

\# 主题名称

\## Summary（200-400字高度浓缩）

\## Key Points（ bullet points）

\## Connections（与其他主题的交叉链接）

\## Sources（列出 raw/ 里的原始文件来源）

\## Open Questions / Contradictions（如果有就必须标红）

\## Last Updated: YYYY-MM-DD

🔻

第 4 步：让 AI 自动编译 Wiki（15 分钟）

打开 Claude Code / Cursor / Windsurf，随便哪个 AI 编码工具，指向你的文件夹，说一句话：

「读取 raw/ 里所有内容，按照 CLAUDE.md 的规则，在 wiki/ 里生成完整知识库。先建 INDEX.md，再按主题建文章，做好相互链接和总结。」

然后你就去喝咖啡吧……

🔻

第 5 步：开启「越用越聪明」循环

以后所有问题都问你的 wiki：

• 「基于我所有笔记，X 概念的 3 个最大盲区是什么？」

• 「把 A 源和 B 源对这个问题的观点做对比」

• 「用我自己的材料给我写一份 800 字 briefing」

AI 回答完，直接把答案扔回 wiki/ 或 outputs/，知识就完成了自我迭代。

🔻

第 6 步：每月健康检查（防 AI 胡说八道）

让 AI 自己审查整个 wiki，找出矛盾、未解释的概念、没有来源的论断。

这步最容易被忽略，但 Nick 说：错误会像滚雪球一样越滚越大，不检查就是慢性自杀。

🟢

我实操后的真实感受

1\. 以前我有 7 个笔记软件，现在只剩 1 个文件夹。

2\. 搜索效率提升 10 倍以上——AI 读懂了所有上下文。

3\. 最爽的是：我终于敢疯狂收藏东西了，因为我知道 AI 会自动把它们变成我的「超级大脑」。

Karpathy 41 万人收藏的帖子，Nick 直接把它做成了「傻瓜版落地指南」。

现在轮到你了。

行动起来吧！

周末花 30 分钟，建好你的第一版 AI 第二大脑。

等你用上一个月，再回来告诉我：

“我终于理解了什么叫『知识复利』”

> 2026-04-04
> 
> ![文章封面图片](https://pbs.twimg.com/media/HFEgzdzaMAAsz3P?format=jpg&name=large)