# CLAUDE.md - Personal Knowledge Base Schema

# (Nick Spisak + Karpathy 极简版 · 2026)

## 1. 本知识库的目的（必须写清楚）
这是一个【你的主题，例如：AI 工程、个人成长、电商运营】的个人第二大脑。
目标是把 raw/ 里所有混乱的原始材料（文章、笔记、截图、会议记录、PDF、X 线程等）自动整理成结构化、可搜索、能自我迭代的 Wiki。

## 2. 核心原则（永远遵守）
- 保持 super simple and flat（Karpathy 原话）
- 只有 .md 文件，没有数据库、插件、Notion、Obsidian 复杂结构
- 人类能读懂，AI 也能完美理解
- 所有知识最终落在 wiki/ 文件夹
- outputs/ 只放问答结果和决策报告

## 3. 文件夹结构（AI 必须严格遵守）
- raw/ → 所有原始材料（AI 只读不改）
- wiki/ → AI 整理后的结构化知识库（核心）
- outputs/ → 所有问答、briefing、决策输出

## 4. 编译 Wiki 的规则（当我说“compile the wiki”时执行）

1. 先读取 raw/ 里**所有**文件内容。
2. **必须先创建/更新** wiki/INDEX.md：
	- 漂亮的目录表格
	- 每个主题的简要描述 + 链接
	- 全局关键洞见总结
3. 为每个**主要主题**创建一个独立 .md 文件（文件名用 kebab-case，例如 ai-agent-architecture.md）
4. 每篇文章的标准结构：
```markdown
# 主题名称
## Summary（200-400字高度浓缩）
## Key Points（ bullet points）
## Connections（与其他主题的交叉链接）
## Sources（列出 raw/ 里的原始文件来源）
## Open Questions / Contradictions（如果有就必须标红）
## Last Updated: YYYY-MM-DD