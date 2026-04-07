# Wiki 索引

> 本知识库遵循 Karpathy + Nick Spisak 第二大脑方法论，raw/ → wiki/ 编译生成。
> Last Updated: 2026-04-07

---

## 目录

### 一、摘要（来源文章）

| 来源 | 主题 | 摘要 |
|---|---|---|
| [[Attention Is All You Need — 摘要]] | Transformer 原论文（2017） | 完全基于注意力机制的序列转导模型，摒弃 RNN/CNN，奠定现代 LLM 基础 |
| [[Thread by @karpathy — 摘要]] | Karpathy LLM 知识库工作流 | raw/ → LLM 编译 wiki → CLI 问答 → Obsidian 浏览的完整流水线 |
| [[BestBlogs & TrendRadar — 摘要]] | 信息聚合三工具 | BestBlogs（400+信源）、TrendRadar（35平台监控）、CloudFlare Workers 简报 |
| [[Skills 工程最佳实践 — 摘要]] | Skills 不是长 Prompt | 8 条反直觉工程判断：确定性逻辑下沉、SKILL.md 是操作手册、安全是准入门槛 |
| [[Karpathy 第二大脑六步法 — 摘要]] | 六步落地指南 | 建文件夹 → 扔 raw → 写 CLAUDE.md → AI 编译 → 问答循环 → 月度检查 |
| [[Claude Code Skills 实战教程 — 摘要]] | 从 0 到 1 造 Skill | 5步法 + 2个真实案例（AI日报 Skill、文章配图 Skill）+ 三个元 Skill |
| [[时间复杂度 — 摘要]] | Big O 符号 | O(1)/O(log N)/O(N) 速度排序 + 典型算法对照表 |
| [[RAGAS 评估框架 — 摘要]] | RAG 自动化评估 | 三大指标：Context Relevance、Faithfulness、Answer Relevance，无需人工标注 |
| [[RAGFlow — 摘要]] | RAGFlow 开源平台 | DeepDoc 解析 + 可视化分块 + 多模态 + 多 LLM 后端的生产级 RAG 平台 |
| [[Agent Harness — 摘要]] | Agent 的方向盘+护栏 | 7 大模块：沙箱隔离、预算控制、人类介入、可观测性、断点续跑、错误处理、工具框架 |
| [[御舆：Claude Code 架构深度解析 — 摘要]] | 42万字拆解 Claude Code | 15章分4部分：基础→核心系统→高级模式→工程实践，139张 Mermaid 图 |

---

### 二、概念（原子主题文章）

#### 机器学习 / Transformer 架构

| 文章 | 主题 |
|---|---|
| [[Transformer 架构]] | 完全基于注意力的序列建模架构，2017 年 Vaswani 等提出 |
| [[注意力机制]] | 通过 Q/K/V 加权求和动态关注任意位置的计算单元 |
| [[多头注意力]] | 多注意力头并行，关注不同表示子空间的信息 |
| [[自注意力]] | Q/K/V 来自同一序列的注意力，任意位置直接建模依赖 |
| [[掩码自注意力]] | 解码器中遮蔽未来 token 的单向注意力 |
| [[位置编码]] | 为位置无关的注意力注入序列位置信息 |
| [[编码器-解码器]] | Seq2Seq 标准结构：编码器处理输入，解码器生成输出 |
| [[残差连接与层归一化]] | 使极深网络能够有效训练的两大稳定机制 |
| [[序列转导]] | 序列到序列的转换任务，Transformer 的核心应用场景 |
| [[BLEU 评估]] | 衡量生成文本与参考译文相似度的 n-gram 精确度指标 |

#### LLM 知识管理

| 文章 | 主题 |
|---|---|
| [[LLM 知识库]] | 以 LLM 为核心的个人知识库：LLM 撰写和维护 wiki |
| [[Wiki 编译]] | 将 raw/ 异构文档转换为结构化 wiki 的过程 |
| [[Wiki 检查与整理]] | LLM 定期对 wiki 进行健康检查，维持数据完整性 |
| [[LLM 原生问答]] | 不依赖 RAG 的复杂研究问题解答 |
| [[LLM 输出格式]] | Markdown / Marp 幻灯片 / matplotlib 图片等多种输出形态 |
| [[Obsidian 作为 IDE]] | Obsidian 作为 wiki 的前端浏览层 |
| [[数据摄入]] | 将原始材料转换为 LLM 可处理格式的过程 |
| [[合成数据生成]] | 将 wiki 知识从上下文迁移到模型权重的下一阶段 |

#### Claude Code / Agent 系统

| 文章 | 主题 |
|---|---|
| [[Claude Code 架构]] | 御舆书核心：对话循环、工具系统、权限管线、记忆、上下文压缩 |
| [[Agent Harness]] | AI Agent 的方向盘+护栏：7 大模块与 5 条设计原则 |
| [[Claude Code Skills 工程]] | Skills 构建方法论：确定性逻辑下沉、SKILL.md 操作手册、四件套交付 |
| [[Claude Code Skills 实战教程]] | 从 0 到 1 造 Skill 的 5 步法 + 真实案例 + 三个元 Skill |
| [[辅助工具]] | 为 LLM 知识库开发的补充工具（搜索引擎等） |

#### RAG 系统

| 文章 | 主题 |
|---|---|
| [[RAG 流程]] | RAGFlow 端到端链路：DeepDoc 解析 → 智能分块 → 向量检索 → LLM 生成 |
| [[RAG 评估]] | RAGAS 三大指标：Context Relevance、Faithfulness、Answer Relevance |

#### 计算机科学基础

| 文章 | 主题 |
|---|---|
| [[时间复杂度]] | Big O 符号：O(1)/O(log N)/O(N)/O(N²) 等级别及典型算法 |

#### 信息聚合

| 文章 | 主题 |
|---|---|
| [[信息聚合工具]] | BestBlogs / TrendRadar / CloudFlare Workers 三工具的功能与适用场景 |

---

## 全局关键洞见

1. **Wiki 归 LLM 所有，人类几乎不直接编辑。** raw/ 是不可变原材料，wiki/ 是 LLM 编译的产物 ([[LLM 知识库]])
2. **Harness 是 2026 年 Agent 领域的核心议题。** 同一个模型，换不同 Harness，结果天差地别 ([[Agent Harness]])
3. **Skill ≠ 长 Prompt。** Skill 是"让 Agent 稳定完成某件事的方法论包"，核心是把确定性逻辑下沉到脚本 ([[Claude Code Skills 工程]])
4. **RAG 的质量需要独立评估。** RAGAS 三大指标让检索-生成链路无需人工标注即可被测量 ([[RAG 评估]])
5. **Transformer 完全基于注意力，摒弃循环。** 这使并行化训练成为可能，为 GPT/BERT 等一切现代 LLM 奠定基础 ([[Transformer 架构]])

---

## 来源

所有原始材料来自 `[[raw/]]` 目录。编译规则参见 `[[Wiki 编译]]`。
