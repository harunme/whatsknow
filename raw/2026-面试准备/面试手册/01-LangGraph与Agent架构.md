# LangGraph 与 Agent 架构面试手册

## Summary

LangGraph 是 LangChain 生态中用于构建有状态、多步骤 Agent 工作流的框架。相比 LangChain Expression Language (LEL) 的线性链式调用，LangGraph 通过显式建模节点（Node）和边（Edge）构成的有向图，支持循环、条件分支、状态持久化和人工介入等复杂控制流。本手册围绕一个生产级 SOC（安全运营中心）告警研判 Agent 展开，覆盖 StateGraph 状态机设计、条件边编写、Checkpointer 记忆持久化、loop_count 循环控制、Tool Calling、Human-in-the-loop、Subgraph 模块化拆分、错误处理降级以及生产调优等十个核心主题。每个主题均配以实际代码示例，展示从告警接入到 ATT&CK 战术映射的完整研判流程。面试时，候选人应能清晰解释为何选择 LangGraph 而非 LEL，并能现场画出状态机草图、解释 checkpoint 快照机制，以及描述多轮对话中状态如何跨 API 调用保持一致。

---

## 零、基础知识速查

### 0.1 LLM 是什么

**LLM（Large Language Model，大语言模型）** 是基于海量文本数据预训练、具备自然语言理解和生成能力的深度学习模型。GPT-4、Claude、通义千问都是 LLM。

核心能力：
- **文本生成**：续写、翻译、摘要、代码生成
- **推理能力**：Chain-of-Thought（CoT）推理、思维链
- **工具调用**：Function Calling / Tool Use，让 LLM 调用外部工具
- **上下文学习**：通过 prompt 中的示例学习新任务，无需微调

**Token** 是 LLM 处理文本的基本单位。1个中文 Token ≈ 1-2个汉字，1个英文 Token ≈ 0.75个单词。API 按 Token 计费。

```python
# OpenAI API 调用示例
import openai

client = openai.OpenAI(api_key="sk-xxx")

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "你是一个安全分析师。"},
        {"role": "user", "content": "告警 A001 的 Severity 是什么？"}
    ],
    temperature=0.3,
    max_tokens=256
)
print(response.choices[0].message.content)
```

### 0.2 什么是 AI Agent

**Agent（智能体）** = LLM + 工具 + 记忆 + 规划。Agent 能自主决策调用哪些工具、在多轮对话中保持状态、完成复杂任务。

Agent 的核心组件：
```
Agent = LLM（大脑） + Tools（手） + Memory（记忆） + Planner（规划器）
```

**ReAct 模式**（Reasoning + Acting）是最经典的 Agent 范式：
```
Thought → Action → Observation → Thought → Action → ...
思考要做什么 → 执行工具 → 观察结果 → 继续推理 → ...
```

```python
# ReAct 的伪代码
while step < max_steps:
    thought = llm.think(f"当前状态: {state}, 上一步: {observation}")
    if "finish" in thought:
        return thought["answer"]
    action = parse_action(thought)
    observation = execute_tool(action)
    step += 1
```

### 0.3 LangChain 是什么

**LangChain** 是构建 LLM 应用的 Python 框架，核心组件：

| 组件 | 作用 | 示例 |
|------|------|------|
| **Model I/O** | 统一调用各种 LLM | `ChatOpenAI`, `ChatAnthropic` |
| **Prompt Template** | 动态拼接 prompt | `PromptTemplate.from_template` |
| **Chain** | 串联多个 LLM 调用 | `LLMChain`, ` RetrievalQA` |
| **Agent** | LLM + 工具的编排 | `initialize_agent` |
| **Memory** | 对话历史存储 | `ConversationBufferMemory` |
| **Tool** | 外部能力（搜索/计算/API）| `@tool` 装饰器定义 |
| **Index** | 文档检索 | `VectorStoreRetriever` |

```python
from langchain_openai import ChatOpenAI
from langchain.prompts import PromptTemplate
from langchain.chains import LLMChain

llm = ChatOpenAI(model="gpt-4o", temperature=0.3)
prompt = PromptTemplate.from_template("解释 {concept} 在安全领域的应用")
chain = LLMChain(llm=llm, prompt=prompt)
result = chain.run(concept="RAG")
print(result)
```

### 0.4 LangGraph 核心概念

LangGraph 是 LangChain 的扩展，专为**有状态、多步骤、可中断**的 Agent 工作流设计。

三大核心概念：
- **State（状态）**：一个 dict，包含工作流的所有数据，`messages` 列表、`alert_id`、`severity` 等
- **Node（节点）**：Python 函数，接收当前 State，返回更新后的 State
- **Edge（边）**：连接节点的路径，`start → node_a → end`

```python
from langgraph.graph import StateGraph, END

# 定义状态类型
class AlertState(TypedDict):
    messages: list
    alert_id: str
    severity: str
    attack_stage: str

# 定义节点函数
def triage_node(state: AlertState) -> AlertState:
    """告警研判节点"""
    messages = state["messages"]
    last_msg = messages[-1]["content"]
    # LLM 推理...
    return {"severity": "P1", "messages": messages + [{"role": "assistant", "content": "定级为P1"}]}

# 构建图
graph = StateGraph(AlertState)
graph.add_node("triage", triage_node)
graph.add_edge("__root__", "triage")   # 入口
graph.add_edge("triage", END)          # 出口
app = graph.compile()
```

### 0.5 Prompt 的基础结构

```python
# 标准 Prompt 结构
prompt = f"""
系统角色：{system_role}

背景信息：
{context}

用户问题：{user_question}

要求：
1. ...
2. ...
"""

# Few-Shot Prompt（给小费，给例子）
few_shot_prompt = """
示例：
输入：告警 A001，源IP 192.168.1.10，检测到暴力破解
输出：{{"severity": "P1", "attack_stage": "credential_access"}}

现在分析：
输入：{alert_description}
输出："""
```

### 0.6 常见面试基础问题

**Q: LangChain 和 LangGraph 的核心区别？**
- LangChain（LEL）：线性 Chain，`chain.invoke(input)` → output → output，无循环无状态
- LangGraph：有向图，支持循环（`while`）、条件分支（`if`）、人工中断、多轮记忆
- 选 LangGraph 的原因：Agent 需要多轮 Tool Calling、循环推理、人工审批节点

**Q: 什么是 Tool Calling？**
- 让 LLM 根据对话内容决定是否调用外部工具（搜索、API、数据库）
- LLM 输出结构化的 `function_call` 而非普通文本
- LangGraph 中通过 `@tool` 装饰器定义工具，`add_node` 加入图中

**Q: Token 是什么？怎么计算？**
- Token 是模型处理文本的最小单位，中文约 1-2 字/Token，英文约 4 字符/Token
- GPT-4o 上下文窗口 128K tokens，Claude 3.5 支持 200K tokens
- 费用 = (输入 tokens + 输出 tokens) × 单价

---

## 1. LangGraph vs LangChain Expression Language — 为什么选 LangGraph

LangChain Expression Language（LEL，以 `|` 操作符串联 Runnable）适合线性 Chain，链的每一步输出直接作为下一步输入，无分支、无循环。LEL 的局限在于：

- **无内置循环**：需要自行用 while 循环包装，不支持中断恢复。
- **状态不透明**：`input`/`output` 字典在 Chain 内部传递，调试困难。
- **不支持循环内暂停**：无法在 Tool 调用之间插入人工审批。

LangGraph 的核心优势：

```python
# LEL 线性链（无法处理告警需要回退审查的情况）
chain = prompt | model | parser

# LangGraph 有向图（支持循环、条件、人工介入）
# graph = StateGraph(AlertState)
# graph.add_node("triage", triage_node)
# graph.add_node("enrich", enrich_node)
# graph.add_edge("triage", "enrich", conditions...)
```

**选择 LangGraph 的判断标准**：当任务需要「判断-行动-再判断」循环、需要在中间步骤暂停等待外部输入、或者状态需要跨 API 调用持久化时，必须选 LangGraph。

---

## 2. StateGraph 状态机设计 — 告警研判的节点与边

StateGraph 是 LangGraph 的核心数据结构，定义整个 Agent 的状态骨架。

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

class AlertState(TypedDict):
    alert: dict                    # 原始告警：{src_ip, dst_ip, event_type, severity}
    triage_result: str            # 研判结论：true_positive / false_positive / need_review
    attack_stage: str             # 攻击阶段：reconnaissance / initial_access / ...
    mitre_tactics: list[str]      # MITRE ATT&CK 战术列表
    loop_count: int               # 当前循环次数，防止无限递归
    messages: Annotated[list, operator.add]  # 对话历史，可追加

def triage_node(state: AlertState) -> AlertState:
    """告警分级节点：基于规则 + LLM 判断"""
    alert = state["alert"]
    severity = alert.get("severity", "medium")

    if severity == "critical":
        triage_result = "need_review"  # 关键告警需人工复核
        mitre_tactics = ["lateral_movement"]
        attack_stage = "lateral_movement"
    elif severity == "high":
        triage_result = "need_review"
        mitre_tactics = ["initial_access"]
        attack_stage = "initial_access"
    else:
        triage_result = "false_positive"
        mitre_tactics = []
        attack_stage = "unknown"

    return {
        "triage_result": triage_result,
        "mitre_tactics": mitre_tactics,
        "attack_stage": attack_stage,
        "messages": [f"[triage] 告警等级={severity}, 研判={triage_result}"]
    }

# 构建图
graph = StateGraph(AlertState)
graph.add_node("triage", triage_node)
# ... 更多节点 ...
```

状态字段设计原则：将研判中间结果（如 `attack_stage`）放入 state 而非返回临时变量，确保 Checkpointer 能序列化完整上下文。

---

## 3. Conditional Edges — 条件判断的写法

条件边是 LangGraph 实现分支逻辑的核心机制。定义一个路由函数，返回目标节点名称或 `END`。

```python
def route_after_triage(state: AlertState) -> str:
    """根据研判结果路由到下一节点"""
    result = state["triage_result"]

    if result == "true_positive":
        return "escalate"          # 升级处理
    elif result == "false_positive":
        return END                 # 直接结束
    elif result == "need_review":
        if state["loop_count"] >= 3:
            return "auto_close"    # 循环超限，强制关闭
        return "enrich"            # 进入情报丰富
    return END

# 注册条件边：triage 节点的出边根据路由函数动态决定
graph.add_conditional_edges(
    "triage",
    route_after_triage,
    {
        "escalate": "escalate",
        "enrich": "enrich",
        "auto_close": "auto_close",
    }
)
```

常见陷阱：`add_conditional_edges` 的第三个参数字典必须覆盖路由函数所有可能的返回值，否则运行时会抛出 `ValueError: Missing key`。路由函数应当是纯函数（无副作用），依赖 `state` 而非外部变量。

---

## 4. Checkpointer 与 Memory — 多轮对话状态持久化

Checkpointer 将 StateGraph 的快照持久化到存储后端（默认内存，支持 PostgreSQL、SQLite 等），使得 Agent 能在中断后从断点恢复，实现真正的多轮对话 Memory。

```python
from langgraph.checkpoint.memory import MemorySaver

# 内存版 Checkpointer（开发/测试用）
checkpointer = MemorySaver()

# 生产环境推荐：SQLite 本地持久化
# from langgraph.checkpoint.sqlite import SqliteSaver
# checkpointer = SqliteSaver.from_conn_string(":memory:")  # 或指定文件路径

graph = StateGraph(AlertState)
graph.add_node("triage", triage_node)
graph.add_node("enrich", enrich_node)
# ... 构建图 ...
graph.add_edge("triage", "enrich", conditions=route_after_triage)

compiled = graph.compile(checkpointer=checkpointer)

# 调用时传入 thread_id，同一 thread_id 共享状态
config = {"configurable": {"thread_id": "alert-session-001"}}

# 第一轮：处理告警 A
result = compiled.invoke({"alert": alert_a, "loop_count": 0}, config)
# 第二轮：用户追加输入，无需重建状态，LangGraph 自动从 checkpoint 恢复
result = compiled.invoke(
    {"messages": ["确认该 IP 是否在白名单中"]},
    config
)
```

Checkpointer 的内部机制：每次节点执行完成后，LangGraph 将当前 `state` 的完整快照写入存储（包含 `messages` 追加历史）。`thread_id` 是分区的键，同一会话的所有调用共享同一个 checkpoint 链。

---

## 5. loop_count 限制 — 防止无限循环的机制

LangGraph 的 `loop_count` 是内置的保护机制，但需要开发者显式维护和检查。

```python
def enrich_node(state: AlertState) -> AlertState:
    """情报丰富节点：调用威胁情报平台查询 IP 信誉"""
    ip = state["alert"]["src_ip"]
    reputation = threat_intel_lookup(ip)  # 假设的工具调用

    # 如果情报不充分，循环回到 triage 重新判断
    new_loop_count = state["loop_count"] + 1

    if not reputation and new_loop_count < 3:
        # 情报不足，需要降级或再尝试
        return {
            "loop_count": new_loop_count,
            "messages": ["[enrich] 情报不足，尝试其他数据源"]
        }
    else:
        return {
            "loop_count": new_loop_count,
            "triage_result": "escalate",
            "messages": [f"[enrich] IP {ip} 情报得分={reputation.get('score', 0)}"]
        }

# 强制中断机制：通过 RecursionLimit
compiled = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["enrich"],  # 可选：在 enrich 节点前中断，供外部介入
)
```

LangGraph 默认不自动限制循环次数，开发者需要在状态中维护 `loop_count` 并在条件边或节点内主动检查。另一个内置保护是 `max_iterations` 参数（在某些版本中），但主流方案仍是显式 `loop_count`。

---

## 6. Tool Calling — 怎么定义 tools 和 tool_choice

LangGraph 中 Tool 通过 `bind_tools` 绑定到模型，支持强制指定调用某个 Tool（`tool_choice`）。

```python
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from pydantic import BaseModel

# 定义工具：输出 schema 通过 Pydantic 约束
class IPReputationInput(BaseModel):
    ip: str
    include_geolocation: bool = False

@tool(args_schema=IPReputationInput)
def lookup_ip_reputation(ip: str, include_geolocation: bool = False) -> dict:
    """查询 IP 威胁情报信誉分数"""
    # 实际项目中调用外部 API（如 VirusTotal、AlienVault OTX）
    return {
        "ip": ip,
        "score": 85,
        "tags": ["malware-c2", "botnet"],
        "country": "RU" if include_geolocation else None
    }

@tool
def block_ip(ip: str, reason: str) -> str:
    """将恶意 IP 加入防火墙黑名单"""
    return f"IP {ip} 已封禁，原因：{reason}"

tools = [lookup_ip_reputation, block_ip]
model = ChatOpenAI(model="gpt-4o").bind_tools(tools)

# 定义 tool_choice：强制模型必须调用 lookup_ip_reputation
# None = 模型自主选择（默认）; "required" = 强制至少调用一个; {"type": "function", "function": {"name": "lookup_ip_reputation"}} = 指定工具
def get_tool_choice(state: AlertState) -> str:
    return "required"

# 节点中直接调用模型
def enrich_node(state: AlertState) -> AlertState:
    events = state.get("messages", [])
    last_msg = events[-1] if events else ""
    response = model.invoke(last_msg)
    tool_calls = response.tool_calls

    # 执行工具
    tool_results = []
    for call in tool_calls:
        selected_tool = {"lookup_ip_reputation": lookup_ip_reputation,
                         "block_ip": block_ip}.get(call["name"])
        if selected_tool:
            result = selected_tool.invoke(call["args"])
            tool_results.append(result)

    return {"messages": [response] + tool_results}
```

生产中建议：为每个 Tool 定义严格的 Pydantic `args_schema`，这样 LangChain 能自动做参数校验和 Schema 生成，减少 Tool 调用时的参数错误。

---

## 7. Human-in-the-loop — 人工审批节点怎么挂起和恢复

`interrupt_before` / `interrupt_after` 是 LangGraph 实现 Human-in-the-loop 的核心 API。

```python
# 编译时指定在特定节点前挂起
compiled = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["escalate"],  # 在升级节点前暂停，等待人工确认
)

# Agent 运行流程
config = {"configurable": {"thread_id": "alert-session-042"}}
alert = {"alert": {"src_ip": "1.2.3.4", "event_type": "ssh_bruteforce", "severity": "high"}, "loop_count": 0}

# 第一次调用：执行到 escalate 前的所有节点，然后暂停
stream = compiled.invoke(alert, config, stream_mode="values")
# 此时 state 已包含 triage + enrich 的结果，escalate 节点尚未执行

# 前端展示给安全分析师：当前告警状态、人工操作建议
# analyst_response = "确认封禁该 IP，原因：SSH 暴力破解确认"

# 第二次调用：传入分析师的决策，LangGraph 从 checkpoint 恢复并继续执行
result = compiled.invoke(
    {"human_decision": "approve_escalation", "messages": ["确认升级"]},
    config
)
```

`interrupt_before` 会将图执行流暂停在指定节点，所有状态已写入 Checkpointer。外部系统（如 SOC 平台的 Web UI）可读取当前状态并向用户展示，待用户决策后再次调用 `invoke`，LangGraph 自动从断点恢复。关键点：恢复时需在输入中包含用户的决策（如 `human_decision`），否则图无法判断后续走向。

---

## 8. Subgraph — 复杂剧本的模块化拆分

Subgraph（嵌套图）允许将大型 Agent 拆分为独立可测试的子图，适合 SOC 场景中的「情报丰富子图」和「响应处置子图」独立开发。

```python
# 子图：情报丰富（可独立测试）
enrich_graph = StateGraph(AlertState)
enrich_graph.add_node("ip_reputation", ip_reputation_node)
enrich_graph.add_node("domain_check", domain_check_node)
enrich_graph.add_node("whois_lookup", whois_node)
enrich_graph.add_edge("ip_reputation", "domain_check")
enrich_graph.add_edge("domain_check", "whois_lookup")
enrich_subgraph = enrich_graph.compile()

# 父图：将子图作为单一节点嵌入
parent_graph = StateGraph(AlertState)
parent_graph.add_node("triage", triage_node)
parent_graph.add_node("enrich", enrich_subgraph)  # 子图作为节点
parent_graph.add_node("escalate", escalate_node)
parent_graph.add_edge("triage", "enrich")
parent_graph.add_edge("enrich", "escalate")

parent_compiled = parent_graph.compile(checkpointer=checkpointer)
```

子图的输入输出格式应与父图状态兼容。在父图中向 `enrich` 节点传入 `state` 时，LangGraph 会将 `state` 的相关字段透传给子图，子图的返回值会合并回父图状态。注意：`interrupt_before` 不支持子图内部的节点，只能在子图入口或出口处中断。

---

## 9. Error Handling — 节点执行失败怎么降级

LangGraph 本身不提供内置的错误处理装饰器，通常通过 `try/except` 包装节点实现降级。

```python
from langgraph.errors import NodeInterrupt

def safe_enrich_node(state: AlertState) -> AlertState:
    """带降级的情报丰富节点"""
    try:
        ip = state["alert"]["src_ip"]
        result = lookup_ip_reputation.invoke({"ip": ip})
        return {
            "messages": [f"[enrich] IP {ip} 情报得分={result['score']}"],
            "enrich_success": True
        }
    except ToolExecutionError as e:
        # 工具调用失败，降级到规则引擎兜底
        logger.warning(f"情报 API 调用失败，降级到规则引擎: {e}")
        return {
            "messages": ["[enrich] 情报 API 不可用，使用规则引擎兜底"],
            "enrich_success": False,
            "triage_result": "need_review",  # 降级后标记需人工处理
            "attack_stage": "degraded_mode"
        }
    except Exception as e:
        # 未知异常，强制进入人工复核流程
        raise NodeInterrupt(f"enrich 节点异常，已暂停等待人工介入: {e}")

def safe_triage_node(state: AlertState) -> AlertState:
    """带超时控制的 triage 节点"""
    try:
        # 业务逻辑
        return triage_node(state)
    except TimeoutError:
        return {
            "messages": ["[triage] LLM 调用超时，标记为 need_review"],
            "triage_result": "need_review"
        }
```

生产中推荐：在图级别使用 `update_state` 注入降级标记，而非让节点抛出异常中断整张图。LangGraph 的 `NodeInterrupt` 适合需要人工介入的异常，但对于可自动降级的错误（API 超时、Rate Limit），在节点内捕获并返回降级状态更平滑。

---

## 10. 生产调优 — prompt 版本管理、输出 Schema 校验

生产环境中，Prompt 和输出解析的版本管理直接影响 Agent 稳定性。

```python
from pydantic import BaseModel, Field, field_validator

# 严格输出 Schema（Pydantic 约束 LLM 输出）
class TriageOutput(BaseModel):
    triage_result: str = Field(description="研判结论")
    attack_stage: str = Field(description="攻击阶段")
    mitre_tactics: list[str] = Field(description="MITRE ATT&CK 战术列表")
    confidence: float = Field(description="研判置信度 0-1")

    @field_validator("triage_result")
    @classmethod
    def validate_result(cls, v: str) -> str:
        allowed = {"true_positive", "false_positive", "need_review"}
        if v not in allowed:
            raise ValueError(f"triage_result 必须在 {allowed} 中，当前值: {v}")
        return v

# Prompt 版本化管理：使用版本号和变更日志
PROMPTS = {
    "v1.0": """你是一个 SOC 安全分析师。分析以下告警并输出 JSON。
告警内容：{alert}
输出格式：{format_instructions}""",
    "v1.1": """你是一个 SOC 安全分析师，具备 MITRE ATT&CK 框架知识。
分析以下告警，考虑 IOC 关联和上下文：
{alert}
【重要】如不确定，输出 need_review，不要强行分类。
输出格式：{format_instructions}""",
}

def get_triage_prompt(version: str = "v1.1") -> str:
    return PROMPTS.get(version, PROMPTS["v1.1"])

# 链式校验：Prompt 版本选择 + Schema 校验
structured_model = model.with_structured_output(TriageOutput)

def versioned_triage_node(state: AlertState) -> AlertState:
    prompt_version = state.get("prompt_version", "v1.1")
    prompt = get_triage_prompt(prompt_version)
    formatted = prompt.format(
        alert=state["alert"],
        format_instructions=structured_model.get_format_instructions()
    )
    response = structured_model.invoke(formatted)
    return {
        "triage_result": response.triage_result,
        "attack_stage": response.attack_stage,
        "mitre_tactics": response.mitre_tactics,
        "messages": [f"[triage-{prompt_version}] {response.triage_result} 置信度={response.confidence}"]
    }
```

生产建议：
- **Prompt 版本化**：用字典管理 Prompt 模板，通过 `state["prompt_version"]` 动态选择，便于灰度回滚。
- **Schema 校验**：用 Pydantic 定义输出格式，配合 `model.with_structured_output()` 自动校验 LLM 输出，拒绝不合规结果。
- **Token 预算控制**：LangGraph 的 `messages` 字段会不断追加，需定期做 `messages[-N:]` 截断，防止上下文膨胀超过模型 token 限制。

---

## 面试问答

**Q1：LangGraph 和 LangChain LEL 的核心区别是什么？什么场景下你会选 LEL？**

LangGraph 构建有向有环图，支持条件分支、循环、人工中断和状态持久化；LEL 是线性链，无内置循环和状态记忆。选择 LEL 的场景：Prompt 固定、无分支、调用步骤不超过 3-4 步、不需要跨请求记忆的简单 RAG 或分类任务。

**Q2：解释 Checkpointer 的工作原理。为什么 thread_id 很重要？**

Checkpointer 在每个节点执行完成后将完整状态快照写入存储介质（内存/SQLite/PG）。`thread_id` 是状态分区键：同一 `thread_id` 的所有调用共享同一个 checkpoint 链，LangGraph 通过 `thread_id` 恢复中断前的状态，实现多轮对话记忆。不同 `thread_id` 完全隔离。

**Q3：如何防止 Agent 陷入无限循环？**

两种机制配合使用：显式在 `AlertState` 中维护 `loop_count`，在条件边路由函数和节点内部检查阈值并强制路由到终结节点；同时利用 `interrupt_before` 在关键节点前设断点，外部通过判断 `loop_count` 决定是否继续执行。

**Q4：Human-in-the-loop 在 LangGraph 中怎么实现？**

通过 `compile(interrupt_before=["节点名"])` 在指定节点前挂起图执行。LangGraph 将状态快照写入 Checkpointer 后暂停，外部系统读取状态供人工决策，决策结果随下一次 `invoke` 传入，LangGraph 自动从断点恢复执行。

**Q5：Subgraph 的适用场景是什么？子图和父图之间如何传递状态？**

当某个子流程需要独立开发、测试或复用时使用 Subgraph（如 SOC 中的情报丰富流程）。父图中将子图编译结果作为普通节点嵌入，LangGraph 自动将父图的 `state` 透传给子图，子图的返回值合并回父图状态。

**Q6：你的 SOC Agent 如何做错误降级？**

三层降级策略：工具级异常（API 超时、Rate Limit）在节点内 `try/except` 捕获，返回降级状态并标记 `enrich_success=False`，后续节点感知降级标记走简化流程；节点级未知异常抛出 `NodeInterrupt` 暂停图等待人工；图级设置 `interrupt_after` 所有节点作为兜底断点。

**Q7：生产环境中如何管理 Prompt 版本？**

用版本化字典管理 Prompt 模板（如 `PROMPTS["v1.1"]`），通过状态字段 `prompt_version` 动态选择，支持灰度发布和快速回滚。配合 Pydantic Schema 做输出校验，Prompt 变更时同步更新 Schema 版本，确保 LLM 输出始终可解析。
