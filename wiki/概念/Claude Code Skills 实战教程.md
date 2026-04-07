---
created: 2026-04-07
tags:
  - concept
  - Claude-Code
  - tutorial
---

# Claude Code Skills 实战教程

从 0 到 1 构建 Claude Code Skill 的 5 步实战方法论，以 AI 日报 Skill 和文章配图 Skill 两个真实案例说明全过程。

## 5 步构建法

```
① 找重复劳动（3次以上可考虑）
        ↓
② 先手动跑通一遍（把每一步记下来）
        ↓
③ 把需求讲给 AI 听（自然语言描述）
        ↓
④ 跑起来，改起来（2-3 轮迭代）
        ↓
⑤ 分享出去（GitHub/ClawHub，可选）
```

### Step 1: 找重复劳动

起点不是"我要做个 XX Skill"，而是：
> "这件事我重复做了 3 次以上，每次流程差不多"

### Step 2: 先手动跑通

**跳过这步会吃亏。** AI 造 Skill 前，自己先手动完整做一遍，把每一步具体在干嘛写下来。
这份手动流程 = 最好的需求文档。

### Step 3: 把需求讲给 AI 听

不用写代码，大白话就行：
> "帮我创建一个 Skill，每天从 20 个 AI 信息源抓新闻，按分类汇总成日报，最后渲染成图片。"

AI 追问细节 → 一问一答 → 需求清晰 → AI 生成 SKILL.md

### Step 4: 跑起来改起来

- 第 1-2 轮：修大问题（流程跑不通、关键步骤缺失）
- 第 3 轮+：调细节（格式微调、措辞优化）

### Step 5: 分享（可选）

GitHub / ClawHub，让别人也能用。

## 真实案例：AI 日报 Skill

**背景**：每天从 20+ 网站（VentureBeat、TechCrunch、OpenAI Blog、HuggingFace Papers、GitHub Trending 等）扒 AI 新闻，筛选+分类+写成日报，40 分钟/天。

**最终效果**：说一句"AI 日报"，整个流程自动跑：
1. 脚本从 20+ 源并发抓取
2. 多 subagent 并行处理不同分类（重大发布/研究论文/行业商业/工具应用）
3. 汇总生成 Markdown 日报
4. 渲染成 Newsletter 风格 HTML
5. 自动截图，按分类切成多张图片

**踩坑**：Hacker News 太杂砍掉；截图太长按分类切；GitHub Trending 加 README 内容验证层。

## 真实案例：文章配图 Skill

**在宝玉的 baoyu-article-illustrator 基础上改进**。原版锁定预设视觉风格，改进：把风格决策权交给 AI，底线约束（干净背景/扁平矢量/信息优先/主色≤3）之上自由发挥。

## 三个元 Skill

| Skill | 功能 |
|---|---|
| **find-skills** | 告诉 AI"我想实现 XX"，它自动搜索 ClawHub 推荐安装 |
| **skill-creator** | 自然语言描述需求，生成标准 SKILL.md |
| **skill-vetter** | 安装第三方 Skill 前安全审查，检测红旗信号 |

组合逻辑：找现成的 → 找不到就造 → 装之前扫描安全。

## 核心建议

1. **Description 要准** — 一句话说清楚干什么，AI 据此匹配
2. **先手动跑通再封装** — 自己不知道完整流程，AI 更不可能知道
3. **一个 Skill 解决一个问题** — 别贪多，各自独立迭代
4. **迭代心态** — 第一版一定不完美，改 2-3 轮就稳了

## 相关概念

- [[Claude Code Skills 工程]] — 互补的工程方法论（反直觉 8 条）
- [[信息聚合工具]] — AI 日报 Skill 的上游数据来源
- [[辅助工具]] — Skill 是 LLM 知识库辅助工具生态的核心
