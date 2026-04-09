---
created: 2026-04-08
tags:
  - concept
  - framework
  - AI-engineering
---

# LangChain

LLM 应用开发框架，定位为 **the agent engineering platform** — 提供组件化、管道化的基础设施，把 LLM 应用开发的主流工程范式显式化。

## 一句话定义

LangChain = **Runnable 统一协议** + **LCEL 组合语法** + **集成生态**。

## 核心架构：三层结构

| 层级 | 包 | 定位 | 版本策略 |
|---|---|---|---|
| **基元层** | `langchain-core` | 接口、协议、Runnable 基元；零第三方依赖 | 无版本号，随时可改 |
| **实现层** | `langchain` (langchain_v1) | agents、chat_models、tools 开箱即用组件 | semver 守护 |
| **编排层** | LangGraph | 状态机、循环、人机交互、持久化 | 独立版本 |

> **关键关系**：langchain (v1) 的 Agent 构建在 LangGraph 之上，不是自己实现执行循环。

## 核心抽象：Runnable

所有组件的**统一调用协议**。

```python
from langchain_core.runnables import Runnable

# 核心方法
chain.invoke(input)       # 单输入 → 单输出
chain.batch([i1, i2])      # 多输入并行 → 多输出
chain.stream(input)        # 流式输出
chain.astream_log(input)   # 流式 + 中间步骤

# 自省接口
chain.input_schema         # pydantic model：接受什么输入
chain.output_schema        # pydantic model：吐出什么输出
chain.config_schema()      # config 规范
```

### 为什么需要 Runnable？

1. **异质组件的统一接口**：LLM 调用、工具执行、提示模板、输出解析都是"输入→输出"，Runnable 让它们用同一套方法被调用
2. **批处理和流式的一等公民**：`batch`/`stream` 与 `invoke` 同等地位，切换执行模式不改业务逻辑
3. **透明性**：每个 Runnable 自我描述输入输出结构，支持动态构建和调试
4. **可观测性**：CallbackManager hook 到每个节点的完整生命周期事件

### Runnable 的隐忧

- 大量可选方法（默认抛 `NotImplementedError`），不同来源的实现质量参差不齐
- 组合多层后调试成本高：`invoke` → `RunnableSequence` → `RunnableParallel` → 各种 `RunnableBinding`

## LCEL：组合语法

LangChain Expression Language，用 **Python 原生语法** 组合 Runnable。

### `|` — 顺序管道（RunnableSequence）

```python
chain = prompt | model | output_parser
chain.invoke("what is 2+2")
# 等价于: output_parser.invoke(model.invoke(prompt.invoke("what is 2+2")))
```

### `{}` dict — 并行管道（RunnableParallel）

```python
chain = prompt | {
    "answer": model,
    "source": retriever,   # 同输入并行发给两个 Runnable
}
result = chain.invoke("query")
# result = {"answer": "...", "source": [...]}
```

### 组合哲学

- **Unix 管道隐喻**：每个 Runnable 单一职责，数据流经过滤器
- **保持 Runnable 身份**：组合后仍是 Runnable，可继续组合、可 batch、可 stream
- **Pythonic**：零学习成本，无新 DSL 关键字
- **可嵌套**：`r1 | { "a": r2 | r3, "b": r4 }` 自然支持树状组合

> 与普通函数组合（`chain = lambda x: f2(f1(x))`) 的本质区别：函数只是值到值的映射，没有身份、没有 schema、没有统一接口。

## Agent 执行模型

### Agent vs Chain

| | Chain | Agent |
|---|---|---|
| 执行路径 | 确定性，预设 | 非确定性，LLM 运行时决定 |
| 工具调用 | 无 | 有（LLM 决定调用哪个） |
| 循环 | 无 | 有（LLM 决定何时停止） |
| 自我修正 | 无 | 观察结果影响下一步 |

### ReAct 循环

```
1. LLM 推理 → 请求一个 action（通常是 tool_call）
2. Agent 执行 tool → 接收观察结果（observation）
3. 观察结果反馈给 LLM
4. 重复，直到 LLM 判断可以结束
5. Agent 返回最终结果
```

LangChain 的 Agent **不实现推理算法**（ReAct 是显式推理框架，LangChain 不是），它只**让 ReAct 更容易实现**：你提供 LLM 和工具，框架负责循环编排。

### AgentState

Agent 被建模为**状态机**：`AgentState` 包含消息历史、工具调用记录、中间输出等，agent 执行循环是状态转换过程。

## 生态概览

```
langchain/
├── langchain-core/        # 基元：Runnable, messages, prompts, tools, ...
├── langchain/             # v1 agents, chat_models, embeddings
├── langchaingraph/        # 低层编排（状态机、循环）
├── partners/              # 第三方集成
│   ├── openai/            # GPT 系列
│   ├── anthropic/         # Claude 系列
│   ├── ollama/            # 本地模型
│   └── qdrant, chroma...  # 向量数据库
└── LangSmith/             # 可观测性平台（evals, tracing, 调试）
```

## 快速开始

```python
from langchain.chat_models import init_chat_model

model = init_chat_model("openai:gpt-5.4")
result = model.invoke("Hello, world!")
```

## 评价

**优势**：
- 快速原型：10 行代码接入 LLM
- 组件可替换：换 model 不改 chain
- 行业词汇统一：整个行业都在借鉴这套抽象

**劣势**：
- 简单任务 overhead 过大
- 复杂任务细粒度控制不足（应该用 LangGraph）
- API 不稳定：core 无版本守护，breaking change 频繁
- 抽象层数多：理解一个 chain 可能需要熟悉十来个类

**适用场景**：
- ✅ 快速原型 + 集成多种 LLM/工具/向量库
- ✅ 需要快速换 model 做对比实验
- ❌ Latency-critical 生产路径（性能税）
- ❌ 需要细粒度状态控制（用 LangGraph）

## 与相关概念的关系

- [[Agent Harness]] — Agent 的运行时工程层；LangChain 是"画蓝图"，Harness 是"搭工厂"
- [[RAG 流程]] — LangChain 提供了完整的 RAG 组件支持（retriever、vectorstore、document loader）
- [[RAGFlow]] — RAGFlow 是 LangChain/LlamaIndex 的生产级替代：RAGFlow 解决端到端体验，LangChain 提供底层组件

## Sources

- [LangChain GitHub](https://github.com/langchain-ai/langchain)
- [LangChain Core 源码](https://github.com/langchain-ai/langchain/tree/master/libs/core/langchain_core)
- [LangChain Docs](https://docs.langchain.com/oss/python/langchain/overview)
