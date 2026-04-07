---
title: "lintsinghua/claude-code-book: 42万字拆解 AI Agent 的Harness骨架与神经 —— Claude Code 架构深度剖析，15 章从对话循环到构建你自己的 Agent Harness。在线阅读网站："
source: "https://github.com/lintsinghua/claude-code-book"
author:
published:
created: 2026-04-03
description: "42万字拆解 AI Agent 的Harness骨架与神经 —— Claude Code 架构深度剖析，15 章从对话循环到构建你自己的 Agent Harness。在线阅读网站： - lintsinghua/claude-code-book"
tags:
  - "clippings"
---
**[English](https://github.com/lintsinghua/claude-code-book/blob/main/en/README.md)** | **中文**

## 御舆：解码 Agent Harness

### Claude Code 架构深度剖析

  

> *"一器而工聚焉者，车为多。"* ——《考工记》
> 
> 两千年前，造一辆马车是最复杂的系统工程： **舆** 承载乘者，辕定方向，辐传动力，軎辖为约束——每个构件各司其职，合而为一，车方能行。
> 
> 今天，构建一个 AI Agent 亦是如此：对话循环为 **辕** ，工具系统为 **辐** ，权限管线为 **軎辖** ，而将这一切承载于其上、使智能体真正运转的运行时框架—— Agent Harness——正是那个 **舆** 。
> 
> 古人御舆，驾驭的是天地之间最精密的机械；今人御舆，驾驭的是硅基时代最复杂的智能体系统。
> 
> 本书因此得名 **舆书** 。

  

当所有人都在教你怎么 **用** AI Agent—— **这本书带你拆开它。**

  
  
[![御舆：解码 Agent Harness — Claude Code 架构深度剖析](https://github.com/lintsinghua/claude-code-book/raw/main/cover.png)](https://github.com/lintsinghua/claude-code-book/blob/main/cover.png) [![Decoding Agent Harness — A Deep Architectural Analysis of Claude Code](https://private-user-images.githubusercontent.com/129816813/572236996-39efa7d4-4521-444e-a222-fd0acb756e51.png?jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NzUyMDI2MzcsIm5iZiI6MTc3NTIwMjMzNywicGF0aCI6Ii8xMjk4MTY4MTMvNTcyMjM2OTk2LTM5ZWZhN2Q0LTQ1MjEtNDQ0ZS1hMjIyLWZkMGFjYjc1NmU1MS5wbmc_WC1BbXotQWxnb3JpdGhtPUFXUzQtSE1BQy1TSEEyNTYmWC1BbXotQ3JlZGVudGlhbD1BS0lBVkNPRFlMU0E1M1BRSzRaQSUyRjIwMjYwNDAzJTJGdXMtZWFzdC0xJTJGczMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDI2MDQwM1QwNzQ1MzdaJlgtQW16LUV4cGlyZXM9MzAwJlgtQW16LVNpZ25hdHVyZT02YWVjODA5ZjgxZWNlMTdkMjViMTdlYjUwMTQzODFmNjMzNGJlZTEyNjBlNzdkNmZjNmM0N2NhYjU2NGExNjlhJlgtQW16LVNpZ25lZEhlYWRlcnM9aG9zdCJ9.beYOT6rqEFEviQvO_M2CnU9kiqo5Cool-srTyLnDYac)](https://private-user-images.githubusercontent.com/129816813/572236996-39efa7d4-4521-444e-a222-fd0acb756e51.png?jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NzUyMDI2MzcsIm5iZiI6MTc3NTIwMjMzNywicGF0aCI6Ii8xMjk4MTY4MTMvNTcyMjM2OTk2LTM5ZWZhN2Q0LTQ1MjEtNDQ0ZS1hMjIyLWZkMGFjYjc1NmU1MS5wbmc_WC1BbXotQWxnb3JpdGhtPUFXUzQtSE1BQy1TSEEyNTYmWC1BbXotQ3JlZGVudGlhbD1BS0lBVkNPRFlMU0E1M1BRSzRaQSUyRjIwMjYwNDAzJTJGdXMtZWFzdC0xJTJGczMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDI2MDQwM1QwNzQ1MzdaJlgtQW16LUV4cGlyZXM9MzAwJlgtQW16LVNpZ25hdHVyZT02YWVjODA5ZjgxZWNlMTdkMjViMTdlYjUwMTQzODFmNjMzNGJlZTEyNjBlNzdkNmZjNmM0N2NhYjU2NGExNjlhJlgtQW16LVNpZ25lZEhlYWRlcnM9aG9zdCJ9.beYOT6rqEFEviQvO_M2CnU9kiqo5Cool-srTyLnDYac)

---

> **对话循环如何驱动？工具权限为何是四阶段管线？上下文压缩怎样在 token 预算内运转？子智能体如何通过 Fork 继承父级上下文？**
> 
> 读懂 Claude Code 的设计决策，你就拥有了一套 **可迁移到任何 Agent 框架** 的心智模型。

---

## 这本书有什么不同

**不做使用教程，不列 Prompt 技巧。**

市面上充斥着"如何写好 Prompt"和"如何调用 Agent API"的指南。但如果你想知道一个生产级 Agent 系统的 **骨架** 是怎么搭的——几乎没有资料可查。这本书填补了这个空白。

|  | 特色 | 说明 |
| --- | --- | --- |
|  | **架构分析而非 API 文档** | 不讲"怎么调用"，讲"为什么这样设计"——追溯动机、分析权衡、指出反模式 |
|  | **设计哲学而非使用教程** | 从异步生成器到断路器模式，每章提炼可迁移的设计原则 |
|  | **可迁移的认知模型** | 无论你用 LangChain、AutoGen、CrewAI 还是从零构建，书中 139 张架构图直接复用 |

**书中的数据一览**

| 指标 | 数量 |
| --- | --- |
| 全书字数 | 42 万字（中文）/ 75K+ words（English） |
| 正文章节 | 15 章 + 4 篇附录 |
| Mermaid 架构图/流程图/状态机 | 139 张 |
| 覆盖核心子系统 | 工具系统、权限管线、上下文压缩、记忆系统、钩子系统、子智能体调度、MCP 集成、技能插件、流式架构、Plan 模式 |
| 分析的设计决策 | 50+ 个"为什么这样设计" |
| 术语条目（中英对照） | 100 条 |
| 功能标志 | 89 个 |
| 注册工具 | 50+ 个 |

> **声明：** 本书基于对 Claude Code 公开文档和产品行为的架构分析编写，未引用、未使用任何未公开或未授权的源码。Claude Code 为 Anthropic PBC 产品，本书不隶属于、未获授权于、也不代表 Anthropic。

---

## 快速导航

> **时间紧张？** 01 → 02 → 04 → 15，拿到核心认知和动手能力就够用
> 
> **有经验？** 直接读 Part 2 + Part 3，遇到概念缺口回溯 Part 1
> 
> **系统学习？** 从头到尾，每章做练习，最后 Ch15 构建自己的 Harness（约 2–3 周）
> 
> **查资料？** 直接翻 [附录 A](#appendix--%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99%E9%80%9F%E6%9F%A5) （模块定位）/ [B](#appendix--%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99%E9%80%9F%E6%9F%A5) （工具）/ [C](#appendix--%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99%E9%80%9F%E6%9F%A5) （功能标志）/ [D](#appendix--%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99%E9%80%9F%E6%9F%A5) （术语）

---

## 目录

### Part 1. 基础篇 — 建立心智模型

> 理解 Agent 编程的范式转移，建立对 Agent Harness 的整体认知框架。

| # | 章节 | 核心内容 |
| --- | --- | --- |
| 01 | [智能体编程的新范式](https://github.com/lintsinghua/claude-code-book/blob/main/%E7%AC%AC%E4%B8%80%E9%83%A8%E5%88%86-%E5%9F%BA%E7%A1%80%E7%AF%87/01-%E6%99%BA%E8%83%BD%E4%BD%93%E7%BC%96%E7%A8%8B%E7%9A%84%E6%96%B0%E8%8C%83%E5%BC%8F.md) | Copilot → Claude Code 演进；Agent Harness 五大设计原则；Bun + React/Ink + Zod v4 技术栈 |
| 02 | [对话循环 — Agent 的心跳](https://github.com/lintsinghua/claude-code-book/blob/main/%E7%AC%AC%E4%B8%80%E9%83%A8%E5%88%86-%E5%9F%BA%E7%A1%80%E7%AF%87/02-%E5%AF%B9%E8%AF%9D%E5%BE%AA%E7%8E%AF-Agent%E7%9A%84%E5%BF%83%E8%B7%B3.md) | `while(true)` 异步生成器主循环；五种 yield 事件；十种终止原因； `QueryDeps` 依赖注入 |
| 03 | [工具系统 — Agent 的双手](https://github.com/lintsinghua/claude-code-book/blob/main/%E7%AC%AC%E4%B8%80%E9%83%A8%E5%88%86-%E5%9F%BA%E7%A1%80%E7%AF%87/03-%E5%B7%A5%E5%85%B7%E7%B3%BB%E7%BB%9F-Agent%E7%9A%84%E5%8F%8C%E6%89%8B.md) | `Tool<I,O,P>` 五要素协议； `buildTool` 故障安全工厂；45+ 工具 × 12 类；并发分区贪心算法 |
| 04 | [权限管线 — Agent 的护栏](https://github.com/lintsinghua/claude-code-book/blob/main/%E7%AC%AC%E4%B8%80%E9%83%A8%E5%88%86-%E5%9F%BA%E7%A1%80%E7%AF%87/04-%E6%9D%83%E9%99%90%E7%AE%A1%E7%BA%BF-Agent%E7%9A%84%E6%8A%A4%E6%A0%8F.md) | 四阶段管线；五种权限模式谱系；Bash 规则匹配；推测性分类器 2 秒 Promise.race |

### Part 2. 核心系统篇 — 深入子系统

> 拆解 Agent Harness 的四大核心子系统——配置、记忆、上下文、钩子。

| # | 章节 | 核心内容 |
| --- | --- | --- |
| 05 | [设置与配置 — Agent 的基因](https://github.com/lintsinghua/claude-code-book/blob/main/%E7%AC%AC%E4%BA%8C%E9%83%A8%E5%88%86-%E6%A0%B8%E5%BF%83%E7%B3%BB%E7%BB%9F%E7%AF%87/05-%E8%AE%BE%E7%BD%AE%E4%B8%8E%E9%85%8D%E7%BD%AE-Agent%E7%9A%84%E5%9F%BA%E5%9B%A0.md) | 六层配置优先级链；合并规则；安全边界与供应链攻击防御；双层功能门控 |
| 06 | [记忆系统 — Agent 的长期记忆](https://github.com/lintsinghua/claude-code-book/blob/main/%E7%AC%AC%E4%BA%8C%E9%83%A8%E5%88%86-%E6%A0%B8%E5%BF%83%E7%B3%BB%E7%BB%9F%E7%AF%87/06-%E8%AE%B0%E5%BF%86%E7%B3%BB%E7%BB%9F-Agent%E7%9A%84%E9%95%BF%E6%9C%9F%E8%AE%B0%E5%BF%86.md) | 四种封闭式记忆类型；"只保存无法推导的信息"；MEMORY.md 索引；Fork 记忆机制 |
| 07 | [上下文管理 — Agent 的工作记忆](https://github.com/lintsinghua/claude-code-book/blob/main/%E7%AC%AC%E4%BA%8C%E9%83%A8%E5%88%86-%E6%A0%B8%E5%BF%83%E7%B3%BB%E7%BB%9F%E7%AF%87/07-%E4%B8%8A%E4%B8%8B%E6%96%87%E7%AE%A1%E7%90%86-Agent%E7%9A%84%E5%B7%A5%E4%BD%9C%E8%AE%B0%E5%BF%86.md) | 有效窗口公式；四级渐进压缩（Snip→MicroCompact→Collapse→AutoCompact）；断路器模式 |
| 08 | [钩子系统 — Agent 的生命周期扩展点](https://github.com/lintsinghua/claude-code-book/blob/main/%E7%AC%AC%E4%BA%8C%E9%83%A8%E5%88%86-%E6%A0%B8%E5%BF%83%E7%B3%BB%E7%BB%9F%E7%AF%87/08-%E9%92%A9%E5%AD%90%E7%B3%BB%E7%BB%9F-Agent%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E6%89%A9%E5%B1%95%E7%82%B9.md) | 五种 Hook 类型；26 个生命周期事件；JSON 响应协议；六层优先级；三层安全机制 |

### Part 3. 高级模式篇 — Agent 的组合与扩展

> 探索 Agent 如何组合、编排和扩展——从子智能体到 MCP 协议桥接。

| # | 章节 | 核心内容 |
| --- | --- | --- |
| 09 | [子智能体与 Fork 模式](https://github.com/lintsinghua/claude-code-book/blob/main/%E7%AC%AC%E4%B8%89%E9%83%A8%E5%88%86-%E9%AB%98%E7%BA%A7%E6%A8%A1%E5%BC%8F%E7%AF%87/09-%E5%AD%90%E6%99%BA%E8%83%BD%E4%BD%93%E4%B8%8EFork%E6%A8%A1%E5%BC%8F.md) | 三种 Agent 来源；四种内置 Agent；Fork 字节级上下文继承；递归 Fork 防护 |
| 10 | [协调器模式 — 多智能体编排](https://github.com/lintsinghua/claude-code-book/blob/main/%E7%AC%AC%E4%B8%89%E9%83%A8%E5%88%86-%E9%AB%98%E7%BA%A7%E6%A8%A1%E5%BC%8F%E7%AF%87/10-%E5%8D%8F%E8%B0%83%E5%99%A8%E6%A8%A1%E5%BC%8F-%E5%A4%9A%E6%99%BA%E8%83%BD%E4%BD%93%E7%BC%96%E6%8E%92.md) | Coordinator-Worker 双重门控；"只编排不执行"约束；四种寻址模式；四阶段工作流 |
| 11 | [技能系统与插件架构](https://github.com/lintsinghua/claude-code-book/blob/main/%E7%AC%AC%E4%B8%89%E9%83%A8%E5%88%86-%E9%AB%98%E7%BA%A7%E6%A8%A1%E5%BC%8F%E7%AF%87/11-%E6%8A%80%E8%83%BD%E7%B3%BB%E7%BB%9F%E4%B8%8E%E6%8F%92%E4%BB%B6%E6%9E%B6%E6%9E%84.md) | 11 个核心技能；SKILL.md frontmatter；三级参数替换；分层加载；插件缓存 |
| 12 | [MCP 集成与外部协议](https://github.com/lintsinghua/claude-code-book/blob/main/%E7%AC%AC%E4%B8%89%E9%83%A8%E5%88%86-%E9%AB%98%E7%BA%A7%E6%A8%A1%E5%BC%8F%E7%AF%87/12-MCP%E9%9B%86%E6%88%90%E4%B8%8E%E5%A4%96%E9%83%A8%E5%8D%8F%E8%AE%AE.md) | 8 种传输协议；五态连接管理；三段式工具命名；Bridge 双向通信系统 |

### Part 4. 工程实践篇 — 从原理到构建

> 性能优化的工程细节，以及从零构建一个完整 Harness 的实战路线图。

| # | 章节 | 核心内容 |
| --- | --- | --- |
| 13 | [流式架构与性能优化](https://github.com/lintsinghua/claude-code-book/blob/main/%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86-%E5%B7%A5%E7%A8%8B%E5%AE%9E%E8%B7%B5%E7%AF%87/13-%E6%B5%81%E5%BC%8F%E6%9E%B6%E6%9E%84%E4%B8%8E%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96.md) | QueryEngine 生命周期管理；并发控制；启动优化 160ms→65ms（-59%）；惰性加载策略 |
| 14 | [Plan 模式与结构化工作流](https://github.com/lintsinghua/claude-code-book/blob/main/%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86-%E5%B7%A5%E7%A8%8B%E5%AE%9E%E8%B7%B5%E7%AF%87/14-Plan%E6%A8%A1%E5%BC%8F%E4%B8%8E%E7%BB%93%E6%9E%84%E5%8C%96%E5%B7%A5%E4%BD%9C%E6%B5%81.md) | "先想后做"哲学；计划文件三层恢复策略；本地调度与远程触发 |
| 15 | [构建你自己的 Agent Harness](https://github.com/lintsinghua/claude-code-book/blob/main/%E7%AC%AC%E5%9B%9B%E9%83%A8%E5%88%86-%E5%B7%A5%E7%A8%8B%E5%AE%9E%E8%B7%B5%E7%AF%87/15-%E6%9E%84%E5%BB%BA%E4%BD%A0%E8%87%AA%E5%B7%B1%E7%9A%84Agent-Harness.md) | 六步实现路线图；循环依赖解决方案；四层可观测性体系；安全威胁模型 |

### Appendix — 参考资料速查

|  | 内容 |
| --- | --- |
| [A](https://github.com/lintsinghua/claude-code-book/blob/main/%E9%99%84%E5%BD%95/A-%E6%BA%90%E7%A0%81%E5%AF%BC%E8%88%AA%E5%9C%B0%E5%9B%BE.md) | **架构导航地图** — 16 个核心模块、依赖树、6 条数据流路径、四层架构、10 种设计模式 |
| [B](https://github.com/lintsinghua/claude-code-book/blob/main/%E9%99%84%E5%BD%95/B-%E5%B7%A5%E5%85%B7%E5%AE%8C%E6%95%B4%E6%B8%85%E5%8D%95.md) | **工具完整清单** — 50+ 工具 × 12 类，readOnly/destructive/concurrencySafe 属性 |
| [C](https://github.com/lintsinghua/claude-code-book/blob/main/%E9%99%84%E5%BD%95/C-%E5%8A%9F%E8%83%BD%E6%A0%87%E5%BF%97%E9%80%9F%E6%9F%A5%E8%A1%A8.md) | **功能标志速查表** — 89 个 Flag × 13 类，编译时/运行时类型，依赖关系图 |
| [D](https://github.com/lintsinghua/claude-code-book/blob/main/%E9%99%84%E5%BD%95/D-%E6%9C%AF%E8%AF%AD%E8%A1%A8.md) | **术语表** — 100 条中英对照术语，含交叉引用和章节定位 |

---

## 适合谁

|  | 读者 | 收获 |
| --- | --- | --- |
|  | **架构师** | 完整的 Agent 设计空间地图和工程权衡分析 |
|  | **高级工程师** | 工具调用、流式处理、权限管控的底层机制 |
|  | **研究者** | 可发表论文级别的 Agent 系统实现分析 |
|  | **Claude Code 用户** | 理解设计意图，最大化利用其能力 |

---

## 背景

2026 年 3 月 31 日，安全研究员 [Chaofan Shou (@Fried\_rice)](https://x.com/Fried_rice) 发现 npm registry 中的 `@anthropic-ai/claude-code` 包存在构建配置失误，source map 文件引用了未设访问控制的 Cloudflare R2 存储桶。披露推文获得超 1700 万次浏览，引发了技术社区对 Agent 架构的空前讨论。

这本书的诞生正是受到这场讨论的启发——当 Agent 架构成为热门话题，我们意识到需要一本系统性的书来讲解 Agent Harness 的设计原理。

---

## 贡献

欢迎 Issue 和 PR — 修正技术错误、补充实战案例、改进章节结构。

## 致谢

[Linux.Do](https://linux.do/) 社区

---

## Star History

[![Star History Chart](https://camo.githubusercontent.com/4e811b1b790fb00a0b6eebd58517c87718377e0cd7209055d1171be564ef4ad7/68747470733a2f2f6170692e737461722d686973746f72792e636f6d2f7376673f7265706f733d6c696e7473696e676875612f636c617564652d636f64652d626f6f6b26747970653d44617465)](https://star-history.com/#lintsinghua/claude-code-book&Date)

---

  
  
可自由分享和改编，但须署名、非商业使用、并以相同协议共享。