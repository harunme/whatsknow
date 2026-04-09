# SIEM 与 MITRE ATT&CK 框架面试手册

## Summary

本手册面向高级 AI 应用工程师，系统梳理安全运营中心（SOC）平台建设中 SIEM 平台与 MITRE ATT&CK 框架的核心知识体系。现代 SIEM 平台已从传统的日志聚合工具演变为融合大数据存储、自然语言查询、机器学习威胁检测的智能化安全分析中枢。MITRE ATT&CK 框架作为威胁行为的统一"语法表"，为安全团队提供了从战略层（战术选择）到战术层（技术检测）的完整映射语言。在实际工程实现中，SIEM 平台需要解决三大核心挑战：海量异构日志的高效采集与归一化、自然语言查询意图的精准解析、以及基于 ATT&CK 矩阵的自动化攻击阶段识别。本手册将从日志采集架构出发，深入解析归一化管道、NL to SPL/DSL 技术路线、ATT&CK 14 阶段战术映射、告警智能定级、威胁情报关联等关键模块，并结合面试高频考点给出参考答案，帮助候选人构建完整的安全工程知识图谱。

---

## 零、基础知识速查

### 0.1 安全基础概念

**C.I.A. 三元组**（信息安全的基本目标）：
- **Confidentiality（机密性）**：数据不被未授权访问
- **Integrity（完整性）**：数据不被篡改
- **Availability（可用性）**：系统和服务正常运行

**资产、威胁、漏洞的关系**：
```
风险 = 资产价值 × 威胁可能性 × 漏洞严重性
```

**常见攻击类型速查**：
| 攻击类型 | 说明 |
|---------|------|
| 钓鱼邮件 | 社会工程，通过邮件诱导点击恶意链接 |
| 暴力破解 | 反复尝试密码/密钥，直到成功 |
| SQL 注入 | 在用户输入中注入恶意 SQL 语句 |
| XSS | 在网页中注入恶意脚本 |
| 零日漏洞 | 尚未修复的已知漏洞 |
| APT | 高级持续性威胁，长期潜伏的高级攻击 |

### 0.2 日志基础

日志是安全监控的原材料。常见日志来源：

| 来源 | 日志类型 | 关键字段 |
|------|---------|---------|
| 操作系统 | Windows Event / Linux Syslog | 时间、用户、进程、源/目标IP |
| 网络设备 | Firewall / IDS/IPS | 源IP、目标IP、端口、协议 |
| Web 服务器 | Access Log / Error Log | URL、HTTP状态码、User-Agent |
| 数据库 | Audit Log / Slow Query Log | SQL语句、执行时间、用户 |
| 应用程序 | 自定义业务日志 | 业务事件、用户行为 |
| 云服务 | CloudTrail / VSS / CEF | API调用、资源变更 |

**日志格式**：
- **纯文本**：最常见但难解析（Nginx/Apache 日志）
- **JSON**：结构化，易解析（现代应用推荐）
- **Syslog**：Linux 标准网络协议，RFC 5424
- **CEF（Common Event Format）**：ArcSight 格式，厂商兼容

```python
# Python 解析常见日志格式
import json
import re
from datetime import datetime

# JSON 日志（推荐）
log_json = json.loads('{"time":"2026-04-09T10:30:00Z","src_ip":"192.168.1.10","event":"login_failed"}')

# Nginx 日志解析
nginx_pattern = r'(?P<ip>[\d.]+) - - \[(?P<time>[^\]]+)\] "(?P<method>\w+) (?P<path>\S+)" (?P<status>\d+)'
match = re.match(nginx_pattern, '192.168.1.10 - - [09/Apr/2026:10:30:00 +0800] "GET /admin HTTP/1.1" 403')

# Python logging 标准库
import logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)
logger.info("告警已处置", extra={"alert_id": "A001", "severity": "P1"})
```

### 0.3 告警基础

**告警 vs 事件 vs 事故**：
- **事件（Event）**：任何发生的事情，有好有坏
- **告警（Alert）**：事件触发的通知，需要关注
- **事故（Incident）**：确认的安全事件，需要应急响应

**告警字段标准结构**（OSSEM DD）：
```json
{
  "timestamp": "2026-04-09T10:30:00Z",
  "event_type": "network_connection",
  "src_ip": "192.168.1.10",
  "src_port": 54321,
  "dst_ip": "8.8.8.8",
  "dst_port": 443,
  "protocol": "TCP",
  "alert_id": "A001",
  "severity": "high",
  "confidence": 0.85,
  "description": "内网主机连接异常外部IP",
  "mitre_tactics": ["command_and_control"],
  "mitre_techniques": ["T1571"]
}
```

### 0.4 SIEM 是什么

**SIEM（Security Information and Event Management）** 是安全领域的日志聚合与分析平台。

核心功能：
1. **日志采集**：从各种来源收集日志
2. **日志存储**：海量数据存储（通常用 Elasticsearch/Splunk）
3. **日志搜索**：快速查询（类似 SQL 的查询语言）
4. **关联分析**：跨日志源关联发现威胁
5. **告警生成**：规则匹配触发告警

主流 SIEM 产品：
| 产品 | 特点 | 场景 |
|------|------|------|
| **Splunk** | 功能强大，按数据量收费，贵 | 大型企业 |
| **Elastic Stack (ELK)** | 开源，Elasticsearch 为核心 | 中小企业 |
| **Microsoft Sentinel** | 云原生，Azure集成 | Azure用户 |
| **灰鲸/Gravity** | 国产，专有云适配 | 国内企业 |
| **Tencent Cloud SIEM** | 腾讯云集成 | 腾讯云用户 |

### 0.5 常见查询语言

**SPL（Splunk Processing Language）**：
```
index=security sourcetype=firewall
| where dest_port in (22, 3389)
| stats count by src_ip
| where count > 100
| sort -count
```

**KQL（Kibana Query Language）**：
```
event_type: "network" and dest_port: (22 or 3389)
| stats count() by src_ip
| where count > 100
| sort count desc
```

**SQL（在数据湖/数据仓库查询）**：
```sql
SELECT src_ip, COUNT(*) as attempts
FROM firewall_logs
WHERE dest_port IN (22, 3389)
GROUP BY src_ip
HAVING COUNT(*) > 100
ORDER BY attempts DESC;
```

### 0.6 常见面试基础问题

**Q: IDS 和防火墙的区别？**
- **防火墙**：基于规则做访问控制（谁能连谁），工作在网络层/传输层
- **IDS/IPS**：检测网络中的恶意流量/行为，IDS 只告警，IPS 可阻断

**Q: EDR 和杀软的区别？**
- **杀毒软件**：基于签名检测已知恶意软件，离线检测
- **EDR**：基于行为检测，支持实时响应、取证、威胁狩猎，在线和离线都覆盖

**Q: 日志保留多久？**
- 金融行业：监管要求通常 5-7 年（等保三级）
- 普通企业：热数据 30-90 天，温数据 1 年，冷数据归档
- 审计日志（操作日志）：通常单独归档，更长保留

---

## 1. SIEM 核心功能

### 1.1 架构全景

SIEM（Security Information and Event Management，安全管理与事件管理）的核心使命是在海量安全数据中实现**快速检测（Detect）、精准调查（Investigate）、高效响应（Respond）**。典型 SIEM 架构分为四层：

**数据采集层（Collection）：** 通过 Agent（FileBeat、WinBeat）或网络协议（Syslog、CEF）将分散在主机、网络设备、云工作负载的日志汇聚至消息队列（Kafka/RabbitMQ），实现削峰填谷。

**数据处理层（Processing）：** 消息队列消费端执行日志解析、字段提取（正则/Grok 模式）、归一化映射，最终写入存储引擎（Elasticsearch/Splunk Indexer/S3）。

**数据存储层（Storage）：** 采用列式存储（Elasticsearch Lucene segment）或追加写结构（Splunk TSIDX），支持时间范围索引（Time-based index）和基于字段的 Bloom Filter 加速检索。

**分析查询层（Analysis）：** 提供底层查询语言（SPL/ES|QL/DSL）、可视化仪表盘、告警引擎和报表生成能力。现代 SIEM 还集成 ML 模型进行异常检测、UEBA（用户实体行为分析）和自然语言搜索接口。

### 1.2 存储与查询的核心取舍

Splunk 的 TSIDX 采用"基于事件的字典编码"——对原始事件做关键词提取和压缩，查询时通过桶索引定位关键词所在的磁盘偏移量，优势是关键字搜索极快，劣势是字段值范围查询（`status >= 500 AND status < 600`）需要解压匹配。Elasticsearch 基于 Lucene 的倒排索引，每个字段单独建立 Posting List，范围查询性能更好，但写入吞吐量受段合并（Segment Merge）影响。实际工程中，高写入量场景（EDR 每秒万级事件）推荐 Kafka + Elasticsearch 组合；高查询复杂度场景（如合规审计跨年日志检索）推荐 Splunk。

---

## 2. SIEM 日志采集

### 2.1 Agent vs Syslog 对比

| 维度 | Agent 方式（FileBeat） | Syslog 方式 |
|------|----------------------|-------------|
| 部署模式 | 轻量 Agent 部署在目标主机 | 目标主机推送或中转服务器拉取 |
| 可靠性 | 支持本地缓存（番茄文件）+ ACK 确认机制 | 仅依赖传输层（UDP/TCP），UDP 易丢包 |
| 采集精度 | 可精确读取文件偏移量，断点续传 | 无法精确控制，文件轮转（logrotate）时可能丢日志 |
| 资源消耗 | ~1-3% CPU，适合生产服务器 | 几乎无资源消耗，但 UDP 无状态无确认 |
| 适用场景 | 应用日志、JSON/CSV 结构化日志 | 网络设备（防火墙、路由器）、遗留系统 |

### 2.2 采集延迟与防丢策略

采集延迟（Log Ingestion Lag）直接影响 MTTD，是 SIEM 工程的核心指标之一。常见延迟来源及应对：

- **文件轮转时隙（Log Rotation Gap）：** 日志文件被 rename 或 truncate 时，Agent 可能错过切换窗口。解法：使用 Inotify（Linux）监听文件 inode 变化，FileBeat 的 `close_inactive` 参数配合 `force_close_files: true`。
- **传输队列积压：** Kafka Consumer Lag 超出阈值时告警，设置 `max.poll.interval.ms` 和消费者数量动态扩容。
- **端到端防丢标准（At-Least-Once）：** 生产端 ACK + 幂等写入（Elasticsearch 的 `_id` 字段用日志内容的 MD5/SHA256 生成）+ 消费者提交 OFFSET 后再删除本地缓存，三步缺一不可。

---

## 3. 日志归一化

### 3.1 归一化的本质

不同安全设备（防火墙、Palo Alto、Fortinet、Windows Event Log、云 CloudTrail）的日志格式差异巨大。归一化是将原始字段映射为统一 CEF（Common Event Format）或 OCSF（Open Cybersecurity Schema Framework）标准字段的过程。

典型映射示例——将 Apache 访问日志和 AWS ALB 日志映射到统一 `network.*` 字段：

```
原始: "192.168.1.100 - frank [10/Oct/2023:13:55:36 -0700] GET /api/users HTTP/1.1" 200 1234
→ dest_ip: 192.168.1.100, http.method: GET, http.url: /api/users, http.status_code: 200

原始: http.client_ip=52.95.4.12 type=ALB tls.cipher=TLS_AES_256_GCM_SHA376
→ src_ip: 52.95.4.12, tls.version: 1.3, tls.cipher: TLS_AES_256_GCM_SHA376
```

### 3.2 映射规则管理与维护

在工程实践中，映射规则由**安全分析师 + SIEM 管理员协同维护**。维护方式分为两类：

- **版本化管理（推荐）：** 映射规则存入 Git，CI/CD 管道在 PR 合并时自动推送至 Logstash pipeline 或 Elasticsearch Ingest Node，避免手动修改导致的配置漂移。
- **动态归一化（高级场景）：** 对于 API Gateway、云原生环境，日志结构随服务迭代频繁变化，使用 JSON Schema 验证 + 自动推断字段类型的管道（Logstash 的 `fingerprint` + ` mutate` 组合）减少人工维护成本。

---

## 4. NL to SPL/DSL 技术路线

### 4.1 技术架构

自然语言转查询语句（NL to Query）是 AI + 安全领域最落地的应用之一。其技术路线分为两大方向：

**Few-Shot Prompt 路线（快速上线）：**
利用 LLM 的上下文学习能力，通过精心设计的系统提示词（System Prompt）引导模型理解查询语法。典型结构：

```
System: 你是一个 Splunk SPL 专家。用户输入自然语言查询，返回对应的 SPL 语句。
        只返回 SPL 代码，不做解释。
        示例1: "查找过去1小时登录失败的用户" → "index=auth earliest=-1h | search action=failure | stats count by user"
        示例2: ...
User: {用户输入}
```

优势：零训练成本，快速迭代；劣势：复杂嵌套查询（`join`、`transaction`）的意图还原不稳定，Token 消耗高。

**Fine-tune 路线（生产级精度）：**
使用（自然语言, SPL）对数据集对开源模型（Llama-3/Qwen-2）做 LoRA 微调，训练数据规模通常在 5-10 万对。

```python
# SFT 训练数据示例
{"messages": [
  {"role": "user", "content": "统计过去7天每个源IP的异常登录次数超过5次的记录"},
  {"role": "assistant", "content": "index=login earliest=-7d | stats count(eval(action=\"failure\" OR action=\"MFA_fail\")) as fail_count by src_ip | where fail_count > 5"}
]}
```

**工程落地关键点：**
- **Schema 注入（Schema-on-Demand）：** 在 Prompt 中动态插入用户有权访问的 Index、Field、Tag 信息，防止跨租户查询越权。
- **输出校验层：** 对 LLM 生成的 SPL 做语法解析（ANTLR 生成 AST），校验 `index=` 是否在白名单、`search` 条件是否包含危险命令（如 `|script`）。
- **意图分类 + 路由：** 先用小模型做意图分类（统计查询 / 搜索查询 / 威胁狩猎），再分发给对应的 SPL/ES|QL/管道。

---

## 5. MITRE ATT&CK 框架

### 5.1 框架结构

MITRE ATT&CK（Adversarial Tactics, Techniques & Common Knowledge）矩阵将攻击者行为归纳为 **14 个战术阶段（Tactics）**，每个战术下包含若干技术（Techniques），截至 ATT&CK v15，企业矩阵包含 14 个战术、196 项技术和 401 项子技术。

### 5.2 14 个攻击阶段详解

**初始访问（Initial Access）：** 攻击链起点。常见向量包括钓鱼邮件（Spearphishing Attachment/Link）、利用面向互联网资产漏洞（Exploit Public-Facing Application）、供应链攻击（Supply Chain Compromise）、外部远程服务（External Remote Services）。防御重点：邮件网关、边界扫描、员工安全意识。

**执行（Execution）：** 在目标环境运行恶意代码。技术包括：恶意脚本（PowerShell/VBScript/JScript）、命令行接口（Command and Scripting Interpreter）、寄生代码（Living Off the Land，即使用 `rundll32.exe`、`mshta.exe` 等合法工具）。SOC 平台在此阶段的检测重点是异常进程链和命令行参数熵分析。

**持久化（Persistence）：** 攻击者在系统重启、凭证更换后维持控制权。典型技术：注册表 Run Key（`HKCU\Software\Microsoft\Windows\CurrentVersion\Run`）、计划任务（Scheduled Task）、服务注册、WMI Event Subscription。ATT&CK v14 新增 `Boot or Logon Autostart Execution` 系列技术。

**权限提升（Privilege Escalation）：** 从低权限账户升级至管理员或 SYSTEM 权限。核心路径：内核漏洞利用、Token 窃取（`SeDebugPrivilege` 滥用）、策略滥用（AlwaysInstallElevated）。

**防御规避（Defense Evasion）：** 攻击者刻意绕过安全检测。高级技术包括：禁用安全工具（`taskkill /im MsMpEng.exe`）、伪装成合法进程（DLL 侧加载）、文件混淆（XOR/Base64 编码）、利用可信父进程启动恶意代码（LOLBAS 攻击）。

**凭证访问（Credential Access）：** 窃取账户密码和认证令牌。经典手法：键盘记录（Keylogger）、内存凭证提取（Mimikatz `sekurlsa::logonpasswords`）、NTLM 中继（Pass-the-Hash/Hook）。

**发现（Discovery）：** 攻击者侦察内网环境，定位高价值资产。技术包括：账户发现（`net user /domain`）、网络扫描（ARP/LDAP 枚举）、系统信息收集（`systeminfo`）、文件发现（搜索配置文件中的凭证）。

**横向移动（Lateral Movement）：** 从初始受害主机扩散至其他系统。见第 9 节详述。

**收集（Collection）：** 从受害主机收集目标数据。技术：`Screen Capture`、`Audio Capture`、邮件收集（Search Email）、本地数据压缩后暂存。

**命令与控制（C2, Command and Control）：** 建立攻击者与受害网络的通信通道。常见协议：DNS-over-HTTPS（隐蔽隧道）、HTTP/S 伪装、C2 框架（Cobalt Strike、Metasploit）、域前置（Domain Fronting）。

**泄露（Exfiltration）：** 将数据从受害者网络传出。典型特征：非工作时间大量出站流量、压缩包加密（HTTPS POST）、分段小批量泄露（低慢大攻击模式）。

**影响（Impact）：** 对业务造成实质破坏。技术：数据加密勒索（Ransomware）、数据销毁（Deletion）、系统覆盖（OS Bootloader Manipulation）、DoS 攻击。

此外还有**资源开发（Resource Development）** 和 **获取支持（Reconnaissance）** 两个前置战术，以及 **损害（Impact）** 中包含的账户操作和业务操作中断。

---

## 6. ATT&CK Navigator 使用与战术矩阵

### 6.1 核心功能

ATT&CK Navigator（[https://mitre-attack.github.io/attack-navigator/](https://mitre-attack.github.io/attack-navigator/)）是一个开源 Web 工具，以矩阵热力图形式展示 ATT&CK 技术覆盖情况。

核心操作：
- **加载 layer 文件（JSON 格式）：** 每个 layer 代表一个评分维度，如红蓝对抗中红队使用的技术子集。
- **色块渲染（Color Coding）：** 数值型评分（0-100）映射到渐变色，最常见用法是标注检测覆盖率（绿=已覆盖，红=未覆盖）。
- **战术过滤（Technique Filtering）：** 仅显示特定战术下的技术，如选择 "Persistence" 战术后矩阵聚焦在该列。
- **版本切换：** 对比 v13 和 v15 的技术增减，追踪 APT 组织战术演变。

### 6.2 SOC 平台集成实践

在 SOC 平台中，ATT&CK Navigator 主要用于：
1. **检测覆盖率评估：** 安全团队对每个检测规则打标签（关联 ATT&CK Technique ID），导出 layer 文件后在 Navigator 中可视化，快速识别盲区。
2. **对手模拟（Adversary Emulation）：** 以特定 APT 组织（如 APT29、LockBit）的战术曲线为基准，设计红蓝对抗场景。
3. **告警归因辅助：** 收到告警时，根据告警关联的 Technique ID 在 Navigator 中快速定位该技术在攻击链中的位置，辅助研判。

---

## 7. 告警分类 P0/P1/P2/P3

### 7.1 定级标准

告警定级是 SOC 运营效率的关键调节阀。业界通常采用**四层金字塔模型**：

| 级别 | 名称 | 判定标准 | SLA 响应时间 |
|------|------|----------|-------------|
| **P0** | 紧急 | 活跃内网攻击、勒索软件行为、高管钓鱼 | 15 分钟 |
| **P1** | 高危 | 已确认的外部攻击、横向移动行为 | 1 小时 |
| **P2** | 中危 | 可疑但未确认的异常行为 | 4 小时 |
| **P3** | 低危 | 政策违规、扫描行为、误报高发规则 | 24 小时 |

### 7.2 LLM 判断 vs 规则引擎

**规则引擎（确定性路径）：**
```
IF event_type = "process_create"
   AND parent_process IN ["mshta.exe", "wscript.exe", "rundll32.exe"]
   AND command_line CONTAINS "vbscript"
   AND src_ip NOT IN whitelist
THEN alert(severity=P1)
```
优势：可解释性强（每条告警有明确触发规则）、性能高；劣势：规则爆炸、无法检测未知威胁。

**LLM 判断（概率推断路径）：**
将告警上下文（进程树、关联告警、资产信息、用户行为基线）打包为结构化 Prompt，输入微调后的安全 LLM，输出 `(severity, confidence, recommendation)` 三元组。

```json
{
  "alert_id": "SEC-2024-00891",
  "context": {
    "primary_event": "powershell.exe spawned by winword.exe with encoded command",
    "process_tree": ["winword.exe → powershell.exe → net.exe"],
    "user": "marketing_analyst@corp.com",
    "asset_criticality": "standard",
    "recent_alerts": ["EDR-4032: Suspicious macro execution"]
  },
  "llm_output": {
    "severity": "P1",
    "confidence": 0.87,
    "reasoning": "父进程 winword.exe 启动 PowerShell 执行 Base64 编码命令，符合 T1059.001 攻击模式，结合用户为非技术岗位，行为异常置信度高。",
    "recommendation": "立即隔离主机并通知用户确认"
  }
}
```

**工程最佳实践（混合架构）：**
- **规则引擎优先（P3/P2）：** 规则命中直接定级，毫秒级响应，避免 LLM 幻觉风险。
- **LLM 二级提升（P0/P1 兜底）：** 规则未命中但上下文存在异常关联时，触发 LLM 分析；或者将所有 P0/P1 告警经过 LLM 做二次研判，降低误报率。
- **反馈闭环：** 分析师每次处理告警的选择（降级/升级/确认为真攻击）回流为 Prompt 微调数据集，持续提升 LLM 定级准确率。

### 7.3 降级与升级场景

- **降级（P1→P2）：** 告警触发但经分析师确认属于白名单行为（如安全团队渗透测试、运维自动化脚本）；关联资产经确认为测试环境。
- **升级（P2→P1）：** 单一 P2 告警（如"异常时间登录"）在短时间内（15 分钟）与多个 P2 告警聚合（如"同一用户新建管理员账户"），攻击链置信度显著提升时应自动升级。
- **跨级别合并：** 同源 IP 或同用户的多级别告警取最高级别处理，避免 P3 告警被低优处理后掩盖真实攻击。

---

## 8. 告警聚合

### 8.1 告警分组策略

告警聚合（Alert Grouping）的目标是减少告警疲劳（Alert Fatigue）。常见的分组维度：

**时间窗口聚合（Temporal Grouping）：**
```
同一源 IP 在 5 分钟窗口内触发的所有告警 → 合并为 1 个 Incident
```
适用场景：NIDS 扫描告警（攻击者短时间内对多个端口发起探测）。

**攻击链关联聚合（Kill Chain Grouping）：**
基于 ATT&CK 战术阶段的有向图建模，同一受害主机的告警若构成攻击链（初始访问→执行→持久化），则合并为高优先级 Incident。

**实体维度聚合（Entity-based Grouping）：**
以受害者资产（hostname、account）为 Key，同一资产关联的所有告警聚合。关键设计：区分"同资产告警聚合"与"跨资产告警聚合"，前者合并，后者作为集群分析。

**语义相似性聚合（ML-based）：**
使用告警文本嵌入向量（Sentence-BERT）计算相似度，余弦相似度 > 0.85 的告警自动归并。优势是能发现规则无法覆盖的新型攻击模式，劣势是推理开销大，适合 P2/P3 告警的后台异步聚合。

### 8.2 同攻击链告警合并实战

在 Kibana/Elasticsearch 中，可用 `transform` 持续聚合实现：

```json
{
  "group_by": ["source.ip", "user.name"],
  "aggs": {
    "alert_types": { "terms": { "field": "rule.name" } },
    "att&ck_tactics": { "terms": { "field": "threat.tactic.id" } },
    "first_seen": { "min": { "field": "@timestamp" } },
    "last_seen": { "max": { "field": "@timestamp" } }
  }
}
```

---

## 9. MTTD / MTTR

### 9.1 定义与行业基准

**MTTD（Mean Time to Detect，平均检测时间）：** 从攻击发生到 SIEM 平台产生告警的平均时间间隔。影响因素：日志采集延迟、日志处理管道延迟、检测规则触发时间窗。

**MTTR（Mean Time to Respond，平均响应时间）：** 从告警产生到安全团队完成响应（隔离、止血、溯源）的平均时间。

**行业基准对比（2024 Verizon DBIR + IBM Cost of Data Breach Report）：**

| 指标 | 业界均值 | 领先水平（<25th percentile） | 落后水平（>75th percentile） |
|------|----------|------------------------------|------------------------------|
| MTTD | ~200 天（APT 检测） | < 24 小时 | > 6 个月 |
| MTTR | ~70 天（完整修复） | < 30 天 | > 90 天 |

**AI/ML 对 MTTD 的影响：** 引入 ML 模型进行异常检测（UEBA）后，部分领先企业的 MTTD 从 200+ 天缩短至 24-48 小时，但误报率同步上升 30-40%，需配套建立告警 triage 质量评估体系（Precision/Recall/F1 评分）。

---

## 10. 威胁情报整合

### 10.1 IOC 类型与生命周期

威胁情报（Threat Intelligence）的核心载体是 IOC（Indicators of Compromise，可陷指标）。主要类型：

- **IP 类：** 恶意 C2 服务器 IP、僵尸网络节点、Tor 出口节点
- **域名类：** 钓鱼域名（DGA 生成域名）、恶意软件托管域名
- **Hash 类：** PE Hash（MD5/SHA1/SHA256）、文件大小+编译时间组合（TP-Link）
- **URL 类：** 社工钓鱼 URL、C2 通信路径

IOC 存在生命周期：生成（由 TI 供应商或内部沙箱分析产出）→ 分发（STIX/TAXII 协议推送到 SIEM）→ 匹配（规则引擎或实时流计算）→ 过时（恶意基础设施被关停）。SIEM 平台需建立 IOC 过期清理机制，防止历史低置信度 IOC 占用查询资源。

### 10.2 外部 TI 对接架构

标准对接协议：STIX 2.1（结构化描述）+ TAXII 2.1（传输协议）。

```
TI 供应商（AlienVault OTX / Recorded Future / VirusTotal）
  → TAXII Server（PKI 认证）
    → SIEM Connector（STIX Pattern → ES/Splunk 查询规则自动转换）
      → 实时匹配引擎
        → IOC Match Alert
```

在 Elasticsearch 中，可用 `Threat Match` 预计算面板实现 IOC 命中告警。

**AI 工程侧优化：** 将 IOC 与告警上下文的语义相似度作为告警优先级加权因子——当告警中出现的 IP/域名/Hash 与高置信度 TI 命中时，自动将告警从 P2 提升至 P1。

---

## 11. 横向移动检测

### 11.1 检测方法对比

横向移动（Lateral Movement）检测是 SOC 平台的核心难点之一，因为攻击者使用合法协议和工具，行为噪声大。

**基于网络流量检测（NDR 路线）：**
- 检测 SMB/PsExec/WMI 远程执行特征：`SMB Session Setup` 中的异常账户使用、同一源 IP 对大量内部 IP 的 445/3389 端口连接（端口扫描→横向移动前兆）。
- 检测 RDP 异常：非工作时间大文件传输（.clipboard 交互）、同一用户从多个地理源 IP 登录（ Impossible Travel 分析）。
- 工具：Zeek（网络流量分析）+ Suricata（规则匹配）+ SIEM 关联分析。

**基于主机日志检测（EDR + HIDS 路线）：**
- Windows Event Log 4688（Process Create）：检测 `Parent_Process_Name` 异常（如 `excel.exe → cmd.exe → powershell.exe`）。
- Windows Event Log 4624/4625：同一账户短时间内多台主机登录（Pass-the-Hash 检测需结合 LSA 缓存分析）。
- Linux `auditd` 日志：SSH 密钥复用（`auth.log` 中同一公钥登录不同主机）。

**工程最佳实践：** NDR + EDR 联合分析——网络流量发现"源IP X 对内部多个IP的 SMB 连接"，同时 EDR 主机日志确认"源主机 Y 上生成了可疑的 PsExec 服务"，两路信号关联置信度远高于单一数据源。

---

## 12. APT 检测策略

### 12.1 "低慢大"攻击特征

APT（高级持续性威胁）的核心特征是"低（低频低熵）、慢（长时间跨度）、大（大规模目标）"——攻击者刻意放慢行动节奏、降低单次行为显著性，以规避阈值型检测规则。

**检测策略设计：**

**策略一：长周期行为基线（Baseline）：**
建立用户/资产 90 天行为基线，包含：每日平均登录时间、常用访问资产集合、常用命令频率、API 调用量。当单日行为偏离基线超过 3σ（标准差）时触发异常告警。

**策略二：跨时间窗口关联（Temporal Correlation）：**
APT 攻击的各阶段行为可能分散在数周甚至数月内，传统 SIEM 的小时/天级时间窗口无法关联。解决方案：使用 Elasticsearch 的 `Rollup Search` 或 Spark Structured Streaming 创建多时间粒度的滑动窗口聚合（如"过去 30 天内，同一用户访问了 10+ 个未曾访问过的资产"）。

**策略三：威胁狩猎（Threat Hunting）：**
由安全分析师主动发起的假设驱动（Hypothesis-driven）调查，典型假设："如果我们被 APT 组织 X 入侵，他们会在凭证访问阶段窃取哪些凭据？"→ 在 SIEM 中执行威胁狩猎查询，主动发现被动告警系统未覆盖的异常。

**策略四：情报驱动狩猎（Intelligence-led Hunting）：**
基于 APT 组织的战术画像（TTP Profile），在 Navigator 中导入 APT29 战术曲线，针对性地在 SIEM 中搜寻该组织惯用的技术特征（如 `T1003.001` LSASS Memory Dump 在 EDR 日志中的特征码）。

---

## 面试 Q&A

**Q1: SIEM 的日志归一化和查询性能如何平衡？**

归一化越细，存储膨胀越大（每个归一化字段都是额外的倒排索引 entry）；归一化越粗，查询语义越模糊。工程上通常采用"双通道架构"——原始日志以 Raw 格式压缩存储用于合规取证，归一化字段写入高性能索引用于实时查询，两者在同一平台透明关联，无需重复存储。

**Q2: NL to SPL 最大的工程难点是什么？**

跨租户数据隔离是 NL to SPL 在多租户 SaaS SIEM 中最大的安全约束。LLM 生成的 SPL 语句必须通过强制性的 Schema 注入——在每次推理前将当前用户有权访问的 index 白名单和字段列表注入 Prompt，防止通过自然语言诱导生成跨租户查询。同时，生成结果的 AST 校验层（拒绝包含 `|script`、`|delete`、`|index=` 的语句）是防止 Prompt 注入攻击的最后一道防线。

**Q3: 如何用 MITRE ATT&CK 框架评估 SOC 平台的检测能力？**

建立"检测规则 ↔ ATT&CK Technique"的映射表（Detection Coverage Matrix），对每个 Technique 评估三个维度：是否有对应规则（覆盖性）、规则触发置信度多高（准确性）、告警到响应的 MTTR 多短（时效性）。将三个维度的评分综合为 Technique 覆盖评分，导入 ATT&CK Navigator 生成热力图，红色密集区域即为检测盲区，需要优先补充规则。

**Q4: P0/P1 告警误报率居高不下，如何系统性解决？**

误报率高通常有两个根本原因：一是规则过于宽泛（"任何 PowerShell 执行"都告警，忽略了运维脚本场景），二是缺乏上下文 enrichment（告警缺少资产重要性、用户角色、历史行为基线等辅助信息）。系统性解法：引入 ML 模型对每个告警做多维度 enrichment，计算告警置信度分数，仅置信度低于阈值（可调整，如 0.6）的告警才推送给分析师处理；同时建立反馈回路，将分析师的处置结果（月度汇总）用于重训模型，形成闭环优化。黄金指标：分析师有效处理率（真正威胁告警 / 总推送告警）应持续提升，误报率（P3→P0 升级被拒绝率）应持续下降。

**Q5: APT 检测中，为什么传统阈值告警失效？**

APT 的"慢"特性导致单次行为永远不触发短窗口阈值。例如：攻击者每 48 小时执行一次凭证访问操作，连续执行 30 天，累计窃取了 15 个账户的凭据，但任何单日告警规则（"1小时内超过5次登录失败"）都不会触发。解法是建立**滑动窗口可配置的多粒度检测规则**——同一行为模式在短窗口（1小时）内低阈值告警，在长窗口（30天）内低阈值持续告警，两路并行，兼顾误报率和 APT 检测覆盖率。

**Q6: 在 SOC 平台中如何设计告警聚合规则，既不过度合并丢失细节，又不过度细分造成告警风暴？**

采用"分层聚合"架构：Level 1（实时层，秒级）——单一主机单一类型告警立即生成 P0/P1；Level 2（5分钟窗口层）——同一主机关联告警按 ATT&CK 战术阶段聚合，生成 Incident；Level 3（异步层，后台任务）——跨主机跨战术的 APT 相关告警使用 ML 聚类，每周生成 Threat Hunting Report。关键原则：聚合发生在分析之后而非之前——原始告警始终保留，聚合视图是降噪展示层而非数据丢失层。
