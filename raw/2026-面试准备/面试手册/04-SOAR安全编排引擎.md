# SOAR 安全编排引擎 — 面试手册

## Summary

SOAR（Security Orchestration, Automation and Response）是安全运营领域将人员、流程和技术深度集成的自动化框架。其核心价值在于：将原本需要安全分析师手动操作的安全响应流程（识别→分析→遏制→恢复）编排为可执行、可复用、可审计的自动化剧本。与 SIEM 聚焦日志聚合与告警生成、EDR 聚焦终端威胁检测不同，SOAR 扮演的是"执行引擎"角色——它消费 SIEM/EDR 的输出，驱动实际的安全响应动作（封锁 IP、隔离主机、通知人员）。本手册围绕一个基于 LangGraph 构建的 SOAR 剧本引擎展开，覆盖 DSL 设计、状态机执行模型、人工审批介入、并发隔离、超时机制、标准剧本实现及开发流程等核心维度，帮助工程师在面试中系统性地展示从架构设计到工程落地的完整能力。

---

## 零、基础知识速查

### 0.1 什么是安全运营（SecOps）

安全运营（Security Operations）是企业通过人员、流程、技术持续监控和响应安全威胁的实践活动。核心团队是 **SOC（Security Operations Center，安全运营中心）**。

SOC 团队的关键角色：
- **安全分析师**：监控告警，做初步分析（Tier 1 / Tier 2）
- **安全工程师**：维护安全工具、编写检测规则
- **安全运营经理**：协调响应、管理 SLA

核心指标：
- **MTTD（Mean Time To Detect）**：平均检测时间，越短越好
- **MTTR（Mean Time To Respond）**：平均响应时间，越短越好
- **MTTC（Mean Time To Contain）**：平均遏制时间

### 0.2 安全事件基础

常见安全事件类型：
| 事件类型 | 说明 | 典型特征 |
|---------|------|---------|
| **恶意软件** | 病毒、木马、勒索软件 | 进程异常、网络连接 |
| **网络入侵** | 未授权访问、横向移动 | 异常登录、扫描行为 |
| **数据泄露** | 敏感数据被窃取 | 大量出站流量、异常API调用 |
| **钓鱼邮件** | 社会工程攻击 | 可疑邮件链接、伪装发件人 |
| **DDoS** | 拒绝服务攻击 | 流量突增、服务不可用 |
| **弱口令** | 暴力破解、凭证泄露 | 多次登录失败、异常时间登录 |

### 0.3 常见安全产品

| 产品 | 英文全称 | 作用 |
|------|---------|------|
| **防火墙** | Firewall | 网络层访问控制 |
| **IDS/IPS** | Intrusion Detection/Prevention System | 入侵检测/防御 |
| **EDR** | Endpoint Detection and Response | 终端行为检测与响应 |
| **WAF** | Web Application Firewall | Web 应用防护 |
| **SIEM** | Security Information and Event Management | 日志聚合与关联分析 |
| **SOAR** | Security Orchestration, Automation and Response | 安全编排与自动化响应 |
| **TI** | Threat Intelligence | 威胁情报 |

### 0.4 告警生命周期

```
产生 → 分诊（Triage）→ 分析（Analysis）→ 调查（Investigation）→ 响应（Response）→ 关闭（Closure）
   ↓           ↓              ↓                ↓                ↓
 SIEM 生成   AI/LLM 定级   分析师判断     溯源与影响评估   SOAR/人工处置
```

告警定级（P0-P3）：
- **P0（Critical）**：正在发生的数据泄露、勒索软件，**立即响应**
- **P1（High）**：可疑入侵行为，**1小时内响应**
- **P2（Medium）**：异常行为，需进一步分析，**4小时内响应**
- **P3（Low）**：低风险，常规审计，**24小时内响应**

### 0.5 什么是剧本（Playbook）

安全剧本（Playbook）是安全响应的标准化操作手册，描述"当 X 发生时，做 Y"。

```yaml
# 钓鱼邮件处置剧本（简化版）
name: phishing_email_response
trigger:
  source: siem
  alert_type: phishing_detected

steps:
  - name: 获取邮件详情
    action: get_email_details
    inputs:
      alert_id: "{{ alert.id }}"

  - name: 提取恶意指标
    action: extract_iocs
    inputs:
      email_content: "{{ step1.output.body }}"
    outputs:
      - malicious_urls
      - malicious_attachments

  - name: 封禁恶意URL
    action: block_url
    when: "malicious_urls is not empty"
    inputs:
      url: "{{ step2.output.malicious_urls[0] }}"

  - name: 人工审批
    type: approval
    timeout: 3600  # 1小时超时

  - name: 通知相关人员
    action: send_notification
    inputs:
      recipients: ["{{ alert.src_user }}", "security-team"]
      message: "检测到针对您的钓鱼邮件，已自动处置"
```

### 0.6 常见面试基础问题

**Q: SIEM 和 SOAR 的区别？**
- **SIEM**：收集日志、关联分析、生成告警，**不执行动作**
- **SOAR**：消费告警、驱动响应动作，**执行自动化操作**
- 关系：SIEM 产生告警 → SOAR 自动响应，两者协同

**Q: 什么是 ATT&CK 框架？**
- MITRE ATT&CK 是威胁行为的知识库，将攻击者行为组织成战术（Tactic）和技术（Technique）
- 14 个战术：初始访问→执行→持久化→权限提升→防御规避→凭证访问→发现→收集→命令与控制→数据泄露→影响
- 用于：红蓝对抗、威胁情报分析、安全评估

**Q: 剧本为什么要人工审批节点？**
- 自动封禁 IP、隔离主机等操作有业务影响，不能完全自动化
- 高风险操作（删除用户、关闭服务）必须人工确认
- 合规要求：金融、医疗等行业要求关键操作有人工记录

---

## 1. SOAR 核心概念

### 1.1 与 SIEM/EDR 的关系

- **SIEM（Security Information and Event Management）**：日志采集、关联分析、生成告警。SIEM 是 SOAR 的告警来源，不做自动响应。
- **EDR（Endpoint Detection and Response）**：终端行为检测、取证。EDR 可被 SOAR 调用 API 执行隔离、删除进程等操作。
- **SOAR**：消费 SIEM/EDR 的告警，通过剧本引擎驱动跨工具自动化响应，是安全运营的"中枢神经"。

### 1.2 剧本（Playbook）的本质

剧本是一个**有向状态图**：节点是操作步骤，边是条件转移逻辑。每个剧本实例持有自己的状态（被封禁的 IP 列表、当前阶段、审批记录），引擎负责驱动状态流转。

### 1.3 核心组件

```
告警输入 → 剧本选择器 → 状态机引擎 → 动作执行器 → 反馈输出
                            ↑
                    人工审批节点（挂起/恢复）
```

---

## 2. DSL 设计

### 2.1 为什么需要 DSL

直接用 Python 写剧本虽然灵活，但难以可视化、非安全工程师无法编辑、无法做静态校验。DSL（领域特定语言）将剧本的**意图**与**实现**解耦。

### 2.2 三种 DSL 方案对比

| 方案 | 优势 | 劣势 |
|------|------|------|
| JSON Schema | 易于前端可视化解析，与 REST API 无缝 | 表达力弱，条件分支冗长 |
| YAML | 人类可读，支持注释，适合配置文件 | 不支持复杂类型验证 |
| Python Class (内部 DSL) | 表达力最强，可复用逻辑片段 | 对非工程师不友好 |

**推荐方案：YAML + Python Class 双轨制**
- 安全分析师编辑 YAML 剧本（声明意图）
- 引擎将 YAML 编译为 Python 类（执行逻辑）
- 两层中间表示分离了"说什么"和"怎么做"

### 2.3 YAML DSL 示例

```yaml
# phishing_lockdown.yaml
name: phishing_lockdown
version: "1.0"
trigger:
  type: siem_alert
  severity: HIGH
  rule_name: phishing_detected

variables:
  target_email: "{{ alert.source_email }}"
  target_hosts: "{{ alert.affected_hosts }}"

nodes:
  - id: extract_ioc
    type: action
    action: extract_ioc
    inputs:
      alert_id: "{{ alert.id }}"
    outputs:
      - phishing_url
      - sender_ip

  - id: block_sender
    type: action
    action: block_email_sender
    inputs:
      sender: "{{ target_email }}"
    retry:
      max_attempts: 3
      backoff: exponential

  - id: human_approve
    type: approval
    approver_role: soc_analyst
    timeout: 3600
    on_timeout: escalate
    prompt: "确认封禁以下主机: {{ target_hosts }}"

  - id: isolate_hosts
    type: action
    action: isolate_endpoints
    condition: "{{ human_approve.approved == true }}"
    inputs:
      hosts: "{{ target_hosts }}"

  - id: notify_team
    type: notification
    action: send_notification
    channels: [slack, email]
```

### 2.4 Python Class 编译层

```python
from dataclasses import dataclass, field
from typing import Literal, Any

@dataclass
class PlaybookNode:
    id: str
    node_type: Literal["action", "approval", "condition", "notification"]
    action_name: str | None = None
    condition: str | None = None
    retry_config: dict | None = None
    timeout: int | None = None

@dataclass
class Playbook:
    name: str
    version: str
    trigger: dict
    variables: dict[str, str] = field(default_factory=dict)
    nodes: list[PlaybookNode] = field(default_factory=list)
    edges: list[tuple[str, str]] = field(default_factory=list)

    def compile_from_yaml(self, yaml_data: dict) -> "Playbook":
        """将 YAML AST 编译为 Playbook 执行类"""
        self.name = yaml_data["name"]
        self.version = yaml_data["version"]
        self.trigger = yaml_data["trigger"]
        self.nodes = [
            PlaybookNode(
                id=n["id"],
                node_type=n["type"],
                action_name=n.get("action"),
                condition=n.get("condition"),
                retry_config=n.get("retry"),
                timeout=n.get("timeout"),
            )
            for n in yaml_data.get("nodes", [])
        ]
        # 从节点顺序推断 edges（线性剧本）
        for i in range(len(self.nodes) - 1):
            self.edges.append((self.nodes[i].id, self.nodes[i+1].id))
        return self
```

---

## 3. LangGraph 实现

### 3.1 为什么选 LangGraph

原生状态机实现（如 `asyncio` + `aiofiles`）工作量大，需要手写状态流转、超时调度、并发控制。LangGraph 提供：
- 内置状态图拓扑定义
- 支持条件分支（`conditional_edges`）
- 支持循环（适用于反复检测直到威胁清除）
- 与 LangChain 生态集成（便于接入 LLM 做告警摘要）

### 3.2 PlaybookState 定义

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
import operator

class PlaybookState(TypedDict):
    alert_id: str
    playbook_name: str
    current_node: str
    variables: dict[str, Any]          # 运行时变量（IOC、审批结果等）
    action_results: dict[str, Any]    # 各节点执行结果
    execution_log: list[dict]         # 审计日志
    status: Annotated[
        Literal["running", "paused", "approved", "rejected", "completed", "failed"],
        operator.or_
    ]
    pending_approvals: list[str]      # 等待审批的节点 ID 列表
```

### 3.3 节点函数实现

```python
from langgraph.prebuilt import ToolNode

# 动作节点
def action_node(state: PlaybookState, node: PlaybookNode):
    action_fn = action_registry.get(node.action_name)
    try:
        result = action_fn(**resolve_inputs(node, state))
        state["action_results"][node.id] = result
        state["execution_log"].append({
            "node": node.id, "status": "success", "result": result
        })
    except Exception as e:
        state["action_results"][node.id] = {"error": str(e)}
        state["execution_log"].append({
            "node": node.id, "status": "failed", "error": str(e)
        })
        state["status"] = "failed"
    return state

# 审批节点（挂起状态机）
def approval_node(state: PlaybookState, node: PlaybookNode):
    state["status"] = "paused"
    state["pending_approvals"].append(node.id)
    state["execution_log"].append({
        "node": node.id, "status": "pending_approval",
        "approver_role": node.approver_role
    })
    return state

# 条件路由
def route_next(state: PlaybookState) -> str:
    """基于状态决定下一个节点"""
    if state["status"] == "failed":
        return "rollback_handler"
    if state["pending_approvals"]:
        return "approval_manager"  # 人工介入节点
    if state["status"] == "completed":
        return END
    return "next_action"
```

### 3.4 构建状态图

```python
def build_playbook_graph(playbook: Playbook) -> StateGraph:
    builder = StateGraph(PlaybookState)

    # 注册所有节点
    for node in playbook.nodes:
        if node.node_type == "action":
            builder.add_node(node.id,
                lambda s, n=node: action_node(s, n), name=node.id)
        elif node.node_type == "approval":
            builder.add_node(node.id,
                lambda s, n=node: approval_node(s, n), name=node.id)

    # 注册边
    for src, dst in playbook.edges:
        builder.add_edge(src, dst)

    # 条件入口
    builder.add_conditional_edges(
        "approval_manager",
        route_next,
        {
            "rollback_handler": "rollback_handler",
            "next_action": "next_action",
            END: END,
        }
    )

    builder.set_entry_point(playbook.nodes[0].id)
    return builder.compile()
```

---

## 4. 可视化编排 + 代码扩展

### 4.1 双模式架构

```
┌─────────────────────────────────────────────────────────┐
│                   Playbook 编辑器 UI                      │
│  ┌─────────────────────┐  ┌──────────────────────────┐  │
│  │   拖拽编排模式        │  │   代码编写模式            │  │
│  │  （YAML 生成器）     │  │  （Python Class 编辑器）   │  │
│  └──────────┬──────────┘  └──────────┬───────────────┘  │
│             │                        │                   │
│             ▼                        ▼                   │
│  ┌──────────────────────────────────────────────┐        │
│  │          Playbook AST（统一中间表示）          │        │
│  └──────────────────────┬───────────────────────┘        │
│                         ▼                                │
│  ┌──────────────────────────────────────────────┐        │
│  │          LangGraph 引擎（执行层）              │        │
│  └──────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────┘
```

### 4.2 ReactFlow 可视化集成

前端使用 ReactFlow 渲染节点图，每个节点类型映射到 DSL schema：
- `ActionNode`：绿色圆角矩形，执行动作
- `ApprovalNode`：黄色菱形，人工审批
- `ConditionNode`：灰色六边形，条件判断
- `NotificationNode`：蓝色圆形，通知

拖拽操作实时生成 YAML，代码编辑器修改 YAML 后同步刷新图形。

### 4.3 Python 代码模式（高级扩展）

对于复杂业务逻辑，分析师可切换到代码模式，直接继承 `BasePlaybook`：

```python
class LateralMovementBlocking(BasePlaybook):
    """横向移动阻断剧本 —— 展示代码级扩展"""
    name = "lateral_movement_blocking"

    def detect_lateral(self, state: PlaybookState) -> list[str]:
        """调用 EDR API 获取同网段通信异常"""
        edr = EDRClient()
        source = state["variables"]["source_ip"]
        peers = edr.get_lateral_peers(source)
        return [p["ip"] for p in peers if p["risk_score"] > 80]

    def execute(self, alert_id: str):
        state = self.init_state(alert_id)
        suspicious_ips = self.detect_lateral(state)

        for ip in suspicious_ips:
            state = self.block_ip_with_approval(state, ip)
            if state["status"] == "rejected":
                break  # 分析师拒绝时停止

        return state
```

---

## 5. Human-in-the-Loop

### 5.1 审批节点挂起机制

审批节点不直接执行后续逻辑，而是将状态机挂起（`status: paused`），将审批请求写入数据库表 `pending_approvals`，等待人工处理。

```python
@dataclass
class ApprovalRequest:
    id: str
    playbook_instance_id: str
    node_id: str
    approver_role: str          # soc_analyst / security_manager
    prompt: str
    created_at: datetime
    deadline: datetime          # created_at + timeout
    status: Literal["pending", "approved", "rejected", "expired"]
    decision_by: str | None
    decision_at: datetime | None
```

### 5.2 审批处理 API

```python
# POST /api/v1/approvals/{approval_id}/decision
@app.post("/approvals/{approval_id}/decision")
async def submit_decision(
    approval_id: str,
    decision: Literal["approved", "rejected"],
    approver: str,
    comment: str = ""
):
    req = await db.get_approval(approval_id)
    if req.status != "pending":
        raise HTTPException(400, "Already processed")

    if datetime.utcnow() > req.deadline:
        req.status = "expired"
        await trigger_timeout_handler(approval_id)
        return {"status": "expired"}

    req.status = decision
    req.decision_by = approver
    req.decision_at = datetime.utcnow()
    await db.save(req)

    # 恢复对应剧本实例的执行
    await resume_playbook_instance(req.playbook_instance_id, decision)
```

### 5.3 超时处理

```python
async def trigger_timeout_handler(approval_id: str):
    req = await db.get_approval(approval_id)
    strategy = req.timeout_strategy  # escalate | auto_approve | auto_reject

    if strategy == "escalate":
        # 通知上一级审批人（通常是 SOC Lead）
        await notify(role="soc_lead", message=f"审批超时: {req.id}")
        req.approver_role = "soc_lead"
        req.deadline = datetime.utcnow() + timedelta(hours=1)
        await db.save(req)

    elif strategy == "auto_reject":
        await resume_playbook_instance(req.playbook_instance_id, "rejected")
        await notify(role="soc_analyst",
            message=f"审批超时已自动拒绝，实例: {req.playbook_instance_id}")

    elif strategy == "auto_approve":
        await resume_playbook_instance(req.playbook_instance_id, "approved")
```

### 5.4 审批人失联处理

失联定义：审批人在 deadline 内既未审批也未响应。通过心跳检测（或 Slack/邮件的已读回执）判断。若超过 2 次升级通知仍无响应，剧本进入人工介入模式（暂停并告警 SOC Lead）。

---

## 6. 执行上下文隔离

### 6.1 为什么要隔离

当同一 IP 被多个告警触发多个剧本时，可能出现竞态：剧本 A 想封禁 IP 1.2.3.4，剧本 B 想解封 1.2.3.4。若无协调，可能互相覆盖。

### 6.2 资源锁机制

```python
import asyncio
from contextlib import asynccontextmanager

class ResourceLockManager:
    """分布式资源锁，防止多剧本对同一 IOC 的冲突操作"""
    def __init__(self):
        self._locks: dict[str, asyncio.Lock] = {}
        self._owners: dict[str, str] = {}  # resource → playbook_instance_id

    @asynccontextmanager
    async def acquire(self, resource_type: str, resource_id: str,
                      playbook_instance_id: str, timeout: int = 30):
        key = f"{resource_type}:{resource_id}"
        if key not in self._locks:
            self._locks[key] = asyncio.Lock()

        lock = self._locks[key]
        try:
            await asyncio.wait_for(lock.acquire(), timeout=timeout)
        except asyncio.TimeoutError:
            raise ResourceLockTimeout(
                f"资源 {key} 被占用，超时 {timeout}s，请重试"
            )

        self._owners[key] = playbook_instance_id
        try:
            yield key
        finally:
            self._owners.pop(key, None)
            lock.release()

    async def is_locked(self, resource_type: str, resource_id: str) -> bool:
        key = f"{resource_type}:{resource_id}"
        return key in self._locks and self._locks[key].locked()
```

### 6.3 锁在剧本执行中的应用

```python
async def block_ip_action(state: PlaybookState, ip: str):
    lock_mgr = state["lock_manager"]

    async with lock_mgr.acquire("ip", ip, state["instance_id"]):
        if await is_already_blocked(ip):
            state["action_results"]["block_ip"] = {
                "status": "skipped",
                "reason": "already_blocked"
            }
            return state

        await firewall_api.block(ip)
        await db.log_action(state["instance_id"], "block_ip", ip)
        state["action_results"]["block_ip"] = {"status": "blocked", "ip": ip}

    return state
```

---

## 7. 执行超时机制

### 7.1 超时分层

| 层级 | 范围 | 默认值 | 策略 |
|------|------|--------|------|
| 节点超时 | 单个 action 节点 | 60s | 重试 → 失败 |
| 审批超时 | Approval 节点 | 3600s | 升级/自动审批 |
| 剧本超时 | 整条剧本 | 7200s | 中止 + 告警 |

### 7.2 节点级超时实现

```python
import asyncio

async def execute_with_timeout(
    coro,
    timeout_seconds: int,
    node_id: str,
    retry_config: dict | None = None
) -> dict:
    max_attempts = (retry_config or {}).get("max_attempts", 1)
    base_delay = (retry_config or {}).get("backoff", 1)

    for attempt in range(max_attempts):
        try:
            result = await asyncio.wait_for(coro, timeout=timeout_seconds)
            return {"status": "success", "result": result, "attempt": attempt + 1}
        except asyncio.TimeoutError:
            delay = base_delay * (2 ** attempt)  # 指数退避
            await asyncio.sleep(delay)
            if attempt == max_attempts - 1:
                return {
                    "status": "failed",
                    "error": f"timeout after {max_attempts} attempts",
                    "last_timeout": timeout_seconds
                }

    return {"status": "failed", "error": "max_attempts_exceeded"}
```

### 7.3 剧本超时监控

```python
async def monitor_playbook_timeout(instance_id: str, deadline: datetime):
    """后台协程，监控剧本是否超时"""
    while datetime.utcnow() < deadline:
        await asyncio.sleep(10)  # 每 10s 检查一次
        instance = await db.get_instance(instance_id)
        if instance.status in ("completed", "failed"):
            return  # 已正常结束

    # 超时中止
    await db.update_status(instance_id, "aborted")
    await notify(role="soc_lead",
        message=f"剧本 {instance.playbook_name} 执行超时，已自动中止。实例ID: {instance_id}")
    await trigger_incident_report(instance_id)
```

---

## 8. 50+ 标准剧本实现逻辑

### 8.1 钓鱼封禁（Phishing Lockdown）

```yaml
# 触发条件：SIEM 钓鱼规则命中（发件人新域名 + 含可疑链接）
# 执行流程：
#  1. 提取 IOC（钓鱼 URL、发件人 IP、目标邮箱列表）
#  2. 邮件网关封锁发件人域名
#  3. SOC 分析师审批是否隔离受感染主机
#  4. EDR 隔离终端（若审批通过）
#  5. 通知 IT 运维和受影响部门
#  6. 写入威胁情报库（供后续关联检测）
```

核心逻辑片段：

```python
async def phishing_lockdown_flow(alert: dict) -> PlaybookState:
    state = init_state(alert)

    # 提取 IOC
    iocs = await extract_ioc(alert["alert_id"])
    state["variables"].update(iocs)

    # 封锁发件人（幂等操作）
    await mail_gateway.block_domain(iocs["sender_domain"])
    state["action_results"]["block_domain"] = "done"

    # 审批隔离主机
    approved = await request_approval(
        role="soc_analyst",
        prompt=f"封禁 {iocs['affected_hosts']} 的终端访问？",
        timeout=3600
    )

    if approved:
        for host in iocs["affected_hosts"]:
            await edr.isolate(host)
        state["execution_log"].append({"action": "hosts_isolated"})
    else:
        state["execution_log"].append({"action": "isolation_skipped"})

    # 通知
    await send_notification(
        channels=["slack", "email"],
        message=f"钓鱼告警已处理，封锁域名: {iocs['sender_domain']}"
    )

    return state
```

### 8.2 横向移动阻断（Lateral Movement Blocking）

```yaml
# 触发条件：EDR 检测到同一来源 IP 向多个内部主机发起 SMB/RDP 连接
# 执行流程：
#  1. EDR API 获取攻击者横向路径（同网段通信图）
#  2. 计算风险评分（访问主机数 × 访问频率）
#  3. 评分 > 80：自动封锁攻击者 IP
#  4. 评分 50-80：人工审批后封锁
#  5. 评分 < 50：仅记录，不封锁（避免误报）
```

```python
async def lateral_movement_flow(state: PlaybookState) -> PlaybookState:
    source_ip = state["variables"]["source_ip"]
    edr = EDRClient()

    # 获取横向路径
    lateral_peers = await edr.get_lateral_peers(source_ip)

    risk_score = sum(
        p["access_frequency"] * p["privilege_level"]
        for p in lateral_peers
    ) / (len(lateral_peers) + 1)

    state["variables"]["risk_score"] = risk_score

    if risk_score > 80:
        # 高风险：自动阻断
        await firewall.block(source_ip)
        await edr.kill_suspicious_processes(source_ip)
        await create_security_incident(source_ip, risk_score, lateral_peers)
    elif risk_score > 50:
        # 中风险：等待审批
        await request_approval(role="soc_analyst",
            prompt=f"中风险横向移动，IP={source_ip}，风险评分={risk_score}")
        if state["pending_decision"] == "approved":
            await firewall.block(source_ip)
    else:
        # 低风险：仅记录
        await db.log_detection(source_ip, risk_score, "monitoring_only")

    return state
```

### 8.3 挖矿检测响应（Mining Detection Response）

```yaml
# 触发条件：CPU 持续 > 85% + 连接到已知矿池 IP 段 + 进程名匹配
# 执行流程：
#  1. 确认挖矿进程存在（读取 /proc 或 WMI）
#  2. 自动杀掉挖矿进程（高置信度时，置信度 >= 0.9）
#  3. 中置信度时人工审批
#  4. 封锁矿池域名（DNS Sinkhole）
#  5. 添加主机到监控观察列表（72h）
```

### 8.4 其他典型剧本

| 剧本名称 | 触发条件 | 核心动作 | 审批必要性 |
|----------|----------|----------|-----------|
| 勒索软件检测 | 文件加密速率异常 | 快照备份、隔离主机 | 需审批（影响业务） |
| 账户被盗响应 | 异常登录 + 新设备 | 重置密码、撤销会话 | 需审批 |
| 供应链攻击 | 内部包仓库异常 | 阻断构建、触发审计 | 需 SOC Lead 审批 |
| DDoS 自动缓解 | 流量突增 5x | BGP 黑路由、CDN 切换 | 自动（时间敏感） |
| 敏感数据外泄 | DLP 告警 | 阻断上传、加密文件 | 需审批 |

---

## 9. 剧本失败回滚

### 9.1 回滚设计原则

不是所有动作都能回滚（杀掉的进程无法复活），因此回滚策略分为三级：
- **硬回滚（Reversible）**：能撤销的操作（解封 IP、恢复文件权限）
- **补偿操作（Compensation）**：无法撤销时执行补偿（通知人工复查、创建备份快照）
- **告警升级（Escalation）**：无法处理时通知人工介入

### 9.2 回滚机制实现

```python
from enum import Enum

class RollbackStrategy(Enum):
    REVERSE = "reverse"       # 精确反向操作
    COMPENSATE = "compensate" # 补偿操作
    ESCALATE = "escalate"     # 升级人工

class RollbackRegistry:
    """回滚操作注册表"""
    def __init__(self):
        self._handlers: dict[str, callable] = {}

    def register(self, action_name: str, rollback_fn: callable,
                 strategy: RollbackStrategy):
        self._handlers[action_name] = (rollback_fn, strategy)

    async def rollback_node(self, action_name: str,
                            action_result: dict, state: PlaybookState):
        if action_name not in self._handlers:
            await self._escalate(state, f"No rollback handler: {action_name}")
            return

        rollback_fn, strategy = self._handlers[action_name]

        try:
            await rollback_fn(action_result)
            state["execution_log"].append({
                "node": "rollback",
                "action": action_name,
                "status": "rolled_back",
                "strategy": strategy.value
            })
        except Exception as e:
            await self._escalate(state, f"Rollback failed: {e}")

    async def _escalate(self, state: PlaybookState, reason: str):
        state["status"] = "failed"
        await notify(role="soc_lead",
            message=f"剧本回滚失败需人工介入: {reason}，实例: {state['instance_id']}")
        await create_incident(state)

# 注册回滚处理
rollback_registry = RollbackRegistry()
rollback_registry.register(
    "block_ip",
    rollback_fn=lambda result: firewall.unblock(result["ip"]),
    strategy=RollbackStrategy.REVERSE
)
rollback_registry.register(
    "isolate_endpoints",
    rollback_fn=lambda result: edr.reconnect(result["hosts"]),
    strategy=RollbackStrategy.COMPENSATE
)
```

### 9.3 事务性执行（多动作原子组）

```python
async def execute_atomic_group(actions: list[Action], state: PlaybookState):
    """一组动作要么全部成功，要么全部回滚"""
    executed = []
    try:
        for action in actions:
            result = await action.execute()
            executed.append((action, result))
            state["action_results"][action.id] = result
    except Exception as e:
        # 回滚已执行的动作
        for action, result in reversed(executed):
            await rollback_registry.rollback_node(action.name, result, state)
        state["status"] = "failed"
        raise AtomicExecutionError(f"Group failed: {e}")
```

---

## 10. 剧本开发周期

### 10.1 从需求到上线的完整流程

```
需求提出（安全团队）
    ↓
剧本设计（YAML 草稿 + 流程图评审）
    ↓
静态校验（JSON Schema 验证 + 循环检测 + 死节点检测）
    ↓
单元测试（pytest + mock 所有外部 API）
    ↓
集成测试（连接真实 SIEM/EDR 模拟环境）
    ↓
安全评审（审批节点配置审查、超危操作权限审计）
    ↓
灰度发布（先在 non-production 环境运行 48h）
    ↓
上线（版本化存储，保留历史版本）
    ↓
监控 + 告警（执行成功率、超时率、人工审批通过率）
```

### 10.2 自动化剧本测试框架

```python
import pytest
from unittest.mock import AsyncMock, MagicMock

class TestPhishingLockdown:
    @pytest.fixture
    def mock_dependencies(self):
        return {
            "mail_gateway": AsyncMock(),
            "edr": AsyncMock(),
            "notification": AsyncMock(),
            "db": AsyncMock(),
        }

    @pytest.mark.asyncio
    async def test_block_domain_called(self, mock_dependencies):
        playbook = PhishingLockdownPlaybook(
            dependencies=mock_dependencies
        )
        alert = create_mock_alert(
            rule="phishing_detected",
            sender_domain="evil.com",
            affected_hosts=["host-42"]
        )

        state = await playbook.execute(alert)

        mock_dependencies["mail_gateway"].block_domain.assert_called_once_with(
            "evil.com"
        )
        assert state["action_results"]["block_domain"]["status"] == "done"

    @pytest.mark.asyncio
    async def test_isolation_requires_approval(self, mock_dependencies):
        # 模拟审批拒绝
        mock_dependencies["approval"] = AsyncMock(return_value=False)
        playbook = PhishingLockdownPlaybook(dependencies=mock_dependencies)
        state = await playbook.execute(create_mock_alert())

        mock_dependencies["edr"].isolate.assert_not_called()  # 不应隔离

    def test_yaml_schema_validation(self):
        with open("playbooks/phishing_lockdown.yaml") as f:
            data = yaml.safe_load(f)
        validate_playbook_schema(data)  # JSON Schema 校验
        assert data["nodes"][-1]["type"] in ("notification", "END")
```

### 10.3 关键质量指标

- 剧本执行成功率：目标 > 98%
- 平均执行时间：从告警到完成 < 10 分钟（高优先级）
- 人工审批平均响应时间：< 30 分钟
- 回滚触发频率：< 2%（反映前置条件判断不足）

---

## 面试问答

**Q1: SOAR 和传统安全自动化的核心区别是什么？**

A: 传统自动化是单点工具的脚本调用（如 Python 脚本调用 API）。SOAR 的本质是**有状态的状态机**，支持条件分支、人工审批介入、失败回滚、超时管理和执行上下文隔离。关键区别在于"编排"——不是线性执行一条命令，而是按照预定义图结构协调多系统、多角色的复杂流程。

**Q2: 为什么选择 LangGraph 而不是自研状态机？**

A: LangGraph 提供开箱即用的条件路由、循环支持和检查点（checkpoint）机制，自研需要处理状态序列化、节点拓扑验证、超时调度等大量基础设施工作。LangGraph 的 `conditional_edges` 和 `StateGraph` 原语足够表达所有剧本拓扑，同时与 LangChain 生态集成（LLM 告警摘要）也有战略价值。但 LangGraph 不是银弹——对于超高性能场景（毫秒级响应），原生 `asyncio` + 自研状态机更可控。

**Q3: 如何防止两个剧本同时封锁同一个 IP 导致竞态？**

A: 通过 `ResourceLockManager` 实现资源级互斥锁。以 `resource_type:resource_id` 为键获取 `asyncio.Lock`，锁持有者记录在 `_owners` 中便于调试和告警。超时机制（默认 30s）防止持锁崩溃导致的死锁。注意这只解决了单机并发问题；生产环境还需 Redis 分布式锁配合。

**Q4: 审批超时后自动审批会不会有安全风险？**

A: 风险存在，因此默认策略是 **escalate（升级）** 而非 auto_approve。只有低风险操作（如非破坏性通知）配置 auto_approve；涉及隔离主机、封锁用户等高危操作，auto_approve 必须经过 CISO 审批才可启用。所有超时决策均记录审计日志。

**Q5: 剧本失败后怎么保证系统不会处于不一致状态？**

A: 三层保障：
1. 幂等设计：封锁 IP、杀进程等操作设计为幂等，重复执行不产生额外影响。
2. 事务性原子组：高风险操作组（封锁+隔离）使用原子组，要么全成功要么全回滚。
3. 补偿机制：无法精确回滚时执行补偿操作（如创建快照供人工恢复），最终升级 SOC Lead 人工兜底。

**Q6: 50+ 剧本如何管理版本和灰度？**

A: 每个剧本文件包含 `version` 字段，数据库存储历史版本快照。上线策略：先在非生产环境全量测试，再在生产环境以影子模式（shadow mode，只记录不执行）运行 48h，对比影子执行结果与实际告警匹配率，达标后全量切换。所有版本保留，可随时回滚。

---

*Last Updated: 2026-04-09*
