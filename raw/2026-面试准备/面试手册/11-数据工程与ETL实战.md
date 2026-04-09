# 数据工程与 ETL 实战面试手册

## Summary

本手册面向具备一定实战经验的高级 AI 应用工程师，系统梳理数据工程核心知识体系，涵盖数据仓库分层建模、ETL 调度设计、数据质量保障与 BI 报表开发四大方向。手册以面试为导向，深度讲解 ODS/DWD/DWS/ADS 四层架构职责与分层价值、DWD 层清洗逻辑与异常检测、缓慢变化维（SCD）三种实现策略、APScheduler 分布式部署下的并发控制、增量抽取与 CDC 选型、以及数据质量监控与血缘追踪体系。通过 Python/SQL 代码示例还原真实生产场景，帮助候选人在面试中展现从数据建模到业务交付的完整工程能力。

---

## 零、基础知识速查

### 0.1 数据相关基本概念

**数据 vs 信息 vs 知识**：
- **数据（Data）**：原始事实，如日志、交易记录、传感器读数
- **信息（Information）**：经过处理、有上下文的数据，如"今日活跃用户 10 万"
- **知识（Knowledge）**：信息的提炼和归纳，如"用户活跃度在周末下降 30%"

**OLTP vs OLAP**：
| 维度 | OLTP（联机事务处理）| OLAP（联机分析处理）|
|------|-----------------|------------------|
| 场景 | 日常业务操作 | 数据分析、报表 |
| 数据量 | 单条/少量 | 海量聚合 |
| 查询 | 简单、高并发 | 复杂、低并发 |
| 工具 | MySQL、PostgreSQL | ClickHouse、StarRocks |
| 例子 | 用户下单、告警插入 | DAU 报表、留存分析 |

### 0.2 SQL 基础速查

```sql
-- 创建表
CREATE TABLE IF NOT EXISTS alert_events (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    alert_id    VARCHAR(64) NOT NULL,
    severity    VARCHAR(8),
    src_ip      VARCHAR(45),
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 查询
SELECT severity, COUNT(*) AS cnt
FROM alert_events
WHERE created_at >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY severity
ORDER BY cnt DESC;

-- 关联查询
SELECT a.alert_id, a.severity, b.action
FROM alert_events a
LEFT JOIN alert_actions b ON a.alert_id = b.alert_id
WHERE a.severity = 'P0';

-- 子查询（查 Top-N）
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (ORDER BY created_at DESC) as rn
    FROM alert_events
) t
WHERE rn <= 10;

-- 窗口函数（比子查询更简洁）
SELECT
    alert_id,
    severity,
    created_at,
    LAG(created_at) OVER (PARTITION BY src_ip ORDER BY created_at) as prev_alert_time,
    DATEDIFF(created_at, LAG(created_at) OVER (PARTITION BY src_ip ORDER BY created_at)) as hours_since_last
FROM alert_events;
```

### 0.3 ETL 基本概念

**ETL（Extract-Transform-Load）** = 抽取 → 转换 → 加载，是数据仓库建设的核心流程：

```
源系统（MySQL/Oracle/日志文件/API）
    ↓ Extract（抽取）
ODS 层（原始数据，不做任何处理）
    ↓ Transform（清洗、转换、聚合）
DWD 层（清洗后的明细数据）
    ↓ Load（汇总）
DWS 层（按主题的汇总宽表）
    ↓ 应用
ADS / 应用层（BI 报表、数据服务）
```

**常见 ETL 工具**：
| 工具 | 特点 | 适用场景 |
|------|------|---------|
| **Apache DolphinScheduler** | 国产，调度能力强 | 国内企业 |
| **Apache Airflow** | Python生态，代码化 | 互联网公司 |
| **DataX** | 阿里开源，高效离线同步 | MySQL→Hive 等 |
| **Flink** | 实时流处理 | 实时 ETL |
| ** kettle / Pentaho** | GUI 操作，上手简单 | 非技术人员 |

### 0.4 数据类型速查

| 数据类型 | 说明 | 例子 |
|---------|------|------|
| **结构化数据** | 有固定 Schema（表）| MySQL 表、CSV |
| **半结构化数据** | 有结构但不固定（JSON/XML）| 日志、API 响应 |
| **非结构化数据** | 无固定结构 | 图片、音视频、文本 |

```python
# Python 读取各种数据格式
import json
import csv
import pandas as pd

# JSON（半结构化日志）
with open("app.log") as f:
    for line in f:
        event = json.loads(line)
        print(event["timestamp"], event["event"])

# CSV
df = pd.read_csv("alerts.csv")
print(df.groupby("severity").size())

# 数据库
import pymysql
conn = pymysql.connect(host="localhost", user="root", password="xxx", database="soc_db")
df = pd.read_sql("SELECT * FROM alert_events WHERE severity='P0'", conn)
```

### 0.5 指标体系基础

数据分析的核心是**指标体系**：

| 指标类型 | 说明 | 例子 |
|---------|------|------|
| **原子指标** | 不可拆分的最小单位 | 订单数、PV、UV |
| **派生指标** | 原子指标 + 时间周期 | 日订单数、周活跃用户 |
| **复合指标** | 多个指标计算 | 转化率 = 下单UV / 访问UV |

```sql
-- 常用聚合函数
SELECT
    COUNT(*)              as 总记录数,
    COUNT(DISTINCT user_id) as 去重用户数,
    SUM(amount)           as 总金额,
    AVG(response_time)    as 平均响应时间,
    MAX(created_at)       as 最新时间,
    MIN(created_at)       as 最早时间
FROM api_logs
WHERE created_at >= DATE_SUB(NOW(), INTERVAL 1 DAY);
```

### 0.6 常见面试基础问题

**Q: 数据湖和数据仓库的区别？**
- **数据湖**：存储原始格式（JSON、Parquet、图片），Schema-on-read（读时建模），成本低，适合探索性分析
- **数据仓库**：存储结构化数据，Schema-on-write（写时建模），成本高，适合固定报表
- 现代趋势：**Lakehouse**（数据湖+数据仓库的融合，如 Delta Lake、Iceberg）

**Q: 为什么要分层？不分层行不行？**
- 分层的目的：数据复用、职责清晰、数据质量分层保障
- 不分层的后果：所有逻辑混在一起，改一处动全身，难以维护
- 小团队可以简化（两层：ODS + 应用层），大团队必须分层

**Q: 实时数据和离线数据的区别？**
- **离线（Batch）**：定期批量处理，如每天凌晨跑 T-1 的报表，延迟小时~天级
- **实时（Streaming）**：事件触发立即处理，如 Flink/Kafka 实时大屏，延迟秒~分钟级
- 成本：实时远高于离线，能用离线解决的不要用实时

---

## 1. 数据仓库分层 — ODS/DWD/DWS/ADS 四层职责

### 1.1 各层核心定义

数据仓库分层是工程实践中最重要的架构决策之一，其本质是通过空间换时间、以结构换清晰度。

**ODS 层（Operational Data Store）**：原始数据层，保留源系统原始面貌。与源表通常保持 1:1 或 N:1 映射关系，不做任何清洗转换，字段保持与上游一致，partition 按数据加载时间划分。ODS 的价值在于留存原始凭证、方便数据回溯与问题排查。

**DWD 层（Data Warehouse Detail）**：明细数据层，对应贴源层的清洗与标准化。这一层完成空值填充、异常值处理、字段类型统一、编码规范化（如性别字段统一为 0/1）、时间格式标准化（如统一为 UTC 时间戳）。DWD 层通常按业务主题建模，例如 `dwd_trade_order_di`（订单明细）对应一个完整的业务实体。

**DWS 层（Data Warehouse Summary）**：汇总数据层，面向业务分析需求构建轻量级汇总宽表。DWS 按业务主题+时间周期组织，如 `dws_user_trade_1d`（用户交易汇总表），支持下游报表直接查询。一个 DWS 表通常对应一个分析指标域。

**ADS 层（Application Data Store）**：应用数据层，也称数据应用层，直接面向业务系统或 BI 报表提供数据服务。这一层通常包含宽表、预聚合结果、或针对特定 BI 看板的定制数据。

### 1.2 为什么要分层

面试中常被问到"不分层行不行"，核心答案围绕三点：**数据血缘清晰**，出问题能快速定位是原始数据异常还是计算逻辑错误；**复用性高**，DWD 层可被多个 DWS 复用，避免重复清洗；**SQL 复杂度可控**，每层只需处理当前层的逻辑，SQL 不会膨胀到难以维护。实践中推荐三层或四层，过于简单的两层（ODS + ADS）会导致 ADS 层 SQL 极其复杂，且上游改字段时影响不可控。

---

## 2. DWD 层清洗 — 核心逻辑与实现

### 2.1 空值处理策略

空值的处理不是简单的 fillna，而需要根据业务语义选择策略：

```python
# DWD 层典型清洗逻辑（Python ETL 脚本片段）
import pandas as pd
from datetime import datetime

def clean_order_data(df: pd.DataFrame) -> pd.DataFrame:
    """
    订单数据 DWD 层清洗
    策略：商品数量空值→0，订单金额空值→0，支付时间空值→保留NULL（代表未支付）
    """
    df['goods_cnt'] = df['goods_cnt'].fillna(0).astype(int)
    df['order_amt'] = df['order_amt'].fillna(0).astype('decimal(12,2)')

    # 手机号脱敏：保留前三位和后四位，中间打码
    df['phone'] = df['phone'].apply(
        lambda x: x[:3] + '****' + x[-4:] if pd.notna(x) and len(x) == 11 else None
    )

    # 时间字段标准化为 UTC 时间戳（秒级）
    df['create_time'] = pd.to_datetime(df['create_time'], errors='coerce')
    df['create_time'] = df['create_time'].dt.floor('s').astype('int64') // 10**9

    return df
```

**处理原则**：业务含义上应为 0 的字段（如未发生的金额、数量）填充 0；业务含义上真实缺失的字段（如未支付的支付时间）保留 NULL；用户隐私字段（如手机号、身份证）在 DWD 层统一脱敏，不在下游处理。

### 2.2 异常值检测

生产环境中常用的异常值检测方法：

```sql
-- DWD 层异常值检测 SQL（MySQL / Spark SQL 通用）
-- 检测逻辑：金额为负、订单数量异常大、单价异常偏离品类均值
WITH order_stats AS (
    SELECT
        category_id,
        AVG(order_amt)   AS avg_amt,
        STDDEV(order_amt) AS std_amt
    FROM dwd_trade_order_di
    WHERE dt = '${biz_date}'
      AND order_amt > 0
    GROUP BY category_id
)
SELECT
    o.order_id,
    o.order_amt,
    s.avg_amt,
    s.std_amt,
    CASE
        WHEN o.order_amt < 0              THEN 'NEGATIVE_AMT'
        WHEN o.goods_cnt > 1000           THEN 'EXCESSIVE_QTY'
        WHEN ABS(o.order_amt - s.avg_amt) > 3 * s.std_amt
                                         THEN 'OUTLIER_PRICE'
        ELSE 'NORMAL'
    END AS anomaly_flag
FROM dwd_trade_order_di o
JOIN order_stats s ON o.category_id = s.category_id
WHERE o.dt = '${biz_date}';
```

检测出的异常记录通常打标写入异常表（`dwd_order_anomaly`），不影响主表清洗流程，供下游业务方复核后决定是否采纳。

### 2.3 数据标准化

跨源数据整合时，编码不统一是最常见的问题。推荐在 DWD 层建立统一编码映射维度表（`dim_code_mapping`），通过 JOIN 实现标准化，而非在每条 ETL 逻辑中硬编码。

---

## 3. 缓慢变化维（SCD）— 三种类型与拉链表实现

### 3.1 SCD Type 1/2/3 对比

SCD（Slowly Changing Dimension）处理的是维度数据随时间变化时如何记录历史的问题，这是数仓面试的高频考点。

**Type 1（覆盖）**：直接更新字段，不保留历史。适用于不需要追踪历史的字段（如用户邮箱、紧急联系方式）。优点是简单，缺点是无法追溯。

**Type 2（增加新行）**：保留完整历史，每条记录有 `start_date` 和 `end_date`，最新记录 `end_date = '9999-12-31'`。适用于客户地址、产品分类等需要历史快照的场景。

**Type 3（增加新列）**：在同一行中保留当前值和历史值（如 `current_city` 和 `previous_city`）。适用于最多保留两期历史的场景，不常用。

### 3.2 拉链表实现（Type 2）

```sql
-- 拉链表初始化（首次加载）
INSERT INTO dwd_user_info_zipper
SELECT
    user_id,
    user_name,
    city,
    level,
    '1900-01-01' AS start_date,
    '9999-12-31' AS end_date,
    'I' AS diff_type
FROM ods_user_info WHERE dt = '20240101';

-- 每日增量更新拉链表（标准拉链更新三步走）
-- Step 1: 关闭历史记录（end_date 更新为变更日期）
UPDATE dwd_user_info_zipper
SET end_date = '20240102'
WHERE (user_id, end_date) IN (
    SELECT user_id, '9999-12-31'
    FROM ods_user_info WHERE dt = '20240102'
    -- 有变更的记录：JOIN 后字段值不一致
    AND (user_name, city, level) <>
    (SELECT user_name, city, level FROM dwd_user_info_zipper d
     WHERE d.user_id = ods_user_info.user_id AND d.end_date = '9999-12-31')
);

-- Step 2: 插入新增记录（历史表中不存在的用户）
INSERT INTO dwd_user_info_zipper
SELECT user_id, user_name, city, level,
       '20240102', '9999-12-31', 'I'
FROM ods_user_info
WHERE dt = '20240102'
  AND user_id NOT IN (
      SELECT user_id FROM dwd_user_info_zipper
  );

-- Step 3: 插入变更记录（开启新行，保留历史）
INSERT INTO dwd_user_info_zipper
SELECT o.user_id, o.user_name, o.city, o.level,
       '20240102', '9999-12-31', 'U'
FROM ods_user_info o
WHERE o.dt = '20240102'
  AND EXISTS (
      SELECT 1 FROM dwd_user_info_zipper d
      WHERE d.user_id = o.user_id
        AND d.end_date = '20240102'
  );
```

### 3.3 快照表 vs 拉链表的选择

拉链表存储效率高（每天只存变更数据），但查询历史时需要 `WHERE start_date <= target_date AND end_date > target_date` 条件；每日快照表查询简单（`WHERE dt = target_date`），但存储成本是拉链表的数倍。经验法则：字段数少于 20 个、分析查询频率极高的核心维度表（如用户表）用快照；字段数多、分析侧重历史变化趋势的表用拉链。实践中很多团队选择全量快照 + 按需建拉链，优先保证查询性能。

---

## 4. ETL 调度 — APScheduler 与多实例防重

### 4.1 BlockingScheduler vs AsyncIOScheduler

```python
from apscheduler.schedulers.blocking import BlockingScheduler
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger
import asyncio

# 方式一：BlockingScheduler（适用于独立进程，无异步IO需求）
scheduler_blocking = BlockingScheduler(timezone='Asia/Shanghai')

def daily_etl_job():
    """日级 ODS→DWD 清洗任务，在独立进程中运行"""
    from etl_pipeline import run_dwd_clean
    run_dwd_clean(biz_date='{{ prev_ds }}')

scheduler_blocking.add_job(
    daily_etl_job,
    CronTrigger(day='*', hour=2, minute=0),
    id='dwd_clean_daily',
    replace_existing=True,
    misfire_grace_time=3600  # 容忍1小时内的调度失位
)

# 方式二：AsyncIOScheduler（适用于 FastAPI 异步框架内嵌调度）
scheduler_async = AsyncIOScheduler(timezone='Asia/Shanghai')

async def async_dws_agg_job():
    """DWS 汇总任务，支持异步数据库连接池"""
    from etl_pipeline_async import run_dws_aggregate
    await run_dws_aggregate(biz_date='{{ prev_ds }}')

scheduler_async.add_job(
    async_dws_aggregate_job,
    CronTrigger(day='*', hour=5, minute=0),
    id='dws_agg_daily',
    replace_existing=True
)
```

**选择原则**：如果主进程是 FastAPI 异步服务，用 AsyncIOScheduler 避免阻塞事件循环；如果 ETL 作为独立批处理进程，用 BlockingScheduler 更稳定可靠。需要特别注意 APScheduler 默认内存存储，进程重启后 job 状态不持久，建议搭配数据库（PostgreSQL/MySQL）作为 JobStore。

### 4.2 多实例防重复执行

多实例部署时（容器多副本 + 负载均衡），同一任务可能被多个实例同时触发，导致数据重复或覆盖。核心解法是分布式锁。

```python
import redis
from contextlib import contextmanager

redis_client = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)

@contextmanager
def distributed_lock(lock_name: str, ttl: int = 3600):
    """
    基于 Redis 的分布式锁
    lock_name: 任务唯一标识，建议格式 'etl_lock:{job_name}:{biz_date}'
    ttl: 锁自动过期时间，防止任务挂掉后锁永不释放
    """
    lock_key = f'distributed_lock:{lock_name}'
    acquired = redis_client.set(lock_key, '1', nx=True, ex=ttl)
    if not acquired:
        raise RuntimeError(f'任务 {lock_name} 已被其他实例持有，跳过执行')
    try:
        yield
    finally:
        # 任务完成后主动释放锁
        redis_client.delete(lock_key)

def etl_job_with_lock(biz_date: str):
    job_name = f'dwd_clean:{biz_date}'
    try:
        with distributed_lock(job_name, ttl=7200):
            # 业务逻辑
            run_dwd_clean(biz_date)
    except RuntimeError as e:
        print(f'跳过重复执行: {e}')
```

**补充方案**：如果不用 Redis，也可以在数据库建一张 `etl_job_lock` 表，用 `SELECT ... FOR UPDATE` 实现悲观锁：

```sql
-- 数据库锁方案（MySQL）
BEGIN;
SELECT * FROM etl_job_lock
WHERE job_name = 'dwd_clean_20240102' FOR UPDATE;
-- 获取到锁后执行业务逻辑，结束时 DELETE 或 UPDATE 释放锁
COMMIT;
```

---

## 5. 增量抽取 vs 全量抽取 — CDC 实战

### 5.1 增量判断条件选择

增量抽取的核心是找到"上次处理之后的新增和变更数据"。判断条件的优先级：

1. **时间戳字段**（推荐）：源表有 `updated_at` 或 `create_time` 字段，按 `WHERE updated_at > '{last_run_time}'` 增量拉取。适用场景：源表结构可控且有时间戳字段。
2. **自增 ID**：适用于插入为主、很少更新的表，`WHERE id > '{last_max_id}'`。
3. **全量比对**（不得已）：源表无时间戳、无自增 ID 时，用全量拉取后在 ETL 侧做 left join 比对差异。数据量大时性能差，通常是最后的兜底方案。

```python
def incremental_extract(table: str, last_run_time: str, current_run_time: str) -> pd.DataFrame:
    """
    基于时间戳的增量抽取
    """
    query = f"""
        SELECT * FROM {table}
        WHERE updated_at > '{last_run_time}'
          AND updated_at <= '{current_run_time}'
    """
    return pd.read_sql(query, con=source_db_conn)
```

### 5.2 CDC 方案对比

| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|----------|
| 时间戳 CDC | 轮询 updated_at | 实现简单，无需改上游 | 无法捕获删除操作，需要上游维护时间戳 | MySQL 等关系型数据库 |
| 逻辑日志 CDC | 读取 binlog/Redo Log | 实时性强，能捕获删除和变更 | 需要开启日志、对上游有侵入、需要 Canal/Debezium | Kafka + Debezium 实时同步 |
| 触发器 CDC | 数据库触发器写增量表 | 对应用透明 | 增加数据库负载、侵入性强 | 历史遗留系统的过渡方案 |
| 快照比对 CDC | 定期全量快照，两次对比差异 | 无需改上游 | 存储和计算成本高 | 无法接入上述方案时 |

生产实践中，日级批处理推荐时间戳 CDC；实时数仓（分钟级延迟）推荐基于 Debezium + Kafka 的 binlog CDC。

---

## 6. 数据质量 — 监控体系与血缘追踪

### 6.1 核心监控指标

数据质量监控应在 ETL 流程中自动执行，而非人工检查。关键指标及阈值设计：

```sql
-- 数据质量检查 SQL（在 DWD 层 ETL 后执行）
-- 检查结果写入 dwd_data_quality_log 表

SELECT
    'dwd_trade_order_di'       AS table_name,
    '${biz_date}'              AS check_date,
    COUNT(*)                   AS total_rows,
    COUNTIF(order_amt IS NULL) AS null_amt_cnt,
    ROUND(COUNTIF(order_amt IS NULL) / COUNT(*), 4) AS null_amt_rate,
    COUNTIF(order_id IS NULL OR order_id = '') AS null_id_cnt,
    COUNT(DISTINCT order_id)   AS distinct_order_cnt,
    COUNT(*) - COUNT(DISTINCT order_id) AS duplicate_order_cnt,
    COUNTIF(create_time < '2020-01-01') AS abnormal_time_cnt,
    -- 波动检测：当日行数与近7日均值偏差超过30%则告警
    CASE
        WHEN COUNT(*) < AVG_CNT * 0.7 OR COUNT(*) > AVG_CNT * 1.3
        THEN 'WARN'
        ELSE 'OK'
    END AS volume_alert
FROM dwd_trade_order_di
WHERE dt = '${biz_date}', (
    SELECT AVG(cnt) AS avg_cnt
    FROM (
        SELECT COUNT(*) AS cnt FROM dwd_trade_order_di
        WHERE dt BETWEEN DATE_SUB('${biz_date}', 7) AND DATE_SUB('${biz_date}', 1)
        GROUP BY dt
    ) t
) stats;
```

**告警阈值设计原则**：空值率常规表 < 1%，核心交易表 < 0.01%；重复率 < 0.1%；行数波动告警阈值通常设为与近 7 日均值偏差 ±30%（可按业务调整）。

### 6.2 数据血缘追踪

血缘追踪是数据治理的基础能力。推荐在 ODS/DWD/DWD 层维护血缘元数据表：

```sql
CREATE TABLE meta_data_lineage (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    source_table VARCHAR(128) COMMENT '上游表名',
    source_column VARCHAR(128) COMMENT '上游字段名',
    target_table VARCHAR(128) COMMENT '下游表名',
    target_column VARCHAR(128) COMMENT '下游字段名',
    transform_rule VARCHAR(512) COMMENT '转换规则说明',
    etl_job_name VARCHAR(128) COMMENT '负责的ETL任务名',
    update_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_target (target_table, target_column),
    INDEX idx_source (source_table, source_column)
);

-- 同时推荐在代码层面通过装饰器自动注册血缘
```

面试中展示这一能力的关键在于：不仅知道血缘表怎么建，还要能说清楚血缘在生产中的实际用途——影响分析（如某上游字段变更，快速评估下游影响范围）、数据溯源（报表数据异常时从 ADS 逆向查找到 ODS 原始记录）、合规审计（GDPR 等法规要求的数据处理透明度）。

---

## 7. BI 报表开发 — 指标体系与数仓对接

### 7.1 核心指标体系设计

BI 报表的核心是建立清晰的指标体系，通常分为三层：

**原子指标**：不可拆分的最小度量单位，如 `订单数`、`支付金额`、`注册用户数`。

**派生指标**：在原子指标基础上加上时间周期和修饰词，如 `近7日支付金额`、`今日新增注册用户数`。

**复合指标**：由多个原子指标计算得出，如 `客单价 = 支付金额 / 支付订单数`、`转化率 = 下单用户数 / 访客数`。

数仓层通常在 DWS 层完成派生指标的预计算，在 ADS 层提供复合指标的计算接口。

### 7.2 数仓表对接 BI 工具

```sql
-- BI 报表宽表设计（DWS/ADS 层）
-- 核心原则：宽表字段要"BI友好"，即一张表覆盖一个主题域内所有常用维度

CREATE TABLE ads_business_dashboard (
    stat_date         DATE COMMENT '统计日期',
    platform          VARCHAR(32) COMMENT '平台：APP/小程序/H5',
    channel           VARCHAR(64) COMMENT '渠道来源',
    new_users         INT COMMENT '新增用户数',
    active_users      INT COMMENT '活跃用户数',
    orders            INT COMMENT '下单笔数',
    gmv               DECIMAL(16,2) COMMENT '成交总额',
    conversion_rate   DECIMAL(6,4) COMMENT '下单转化率',
    avg_order_amt     DECIMAL(12,2) COMMENT '客单价',
    PRIMARY KEY (stat_date, platform, channel)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
PARTITION BY RANGE (TO_DAYS(stat_date)) (
    PARTITION p_202401 VALUES LESS THAN (TO_DAYS('2024-02-01')),
    PARTITION p_202402 VALUES LESS THAN (TO_DAYS('2024-03-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

BI 工具（Tableau/帆软/PowerBI）通过 JDBC/ODBC 连接数仓，查询 ADS 层宽表渲染图表。面试中应强调：报表 SQL 性能优化的责任在数仓侧——BI 同学写复杂交叉表时，数仓工程师应提供预聚合宽表支持，而非让 BI 直接 join 多张明细表。

---

## 8. 多源异构数据处理

### 8.1 不同数据源的统一读取框架

```python
import json
import xml.etree.ElementTree as ET
import pandas as pd
from abc import ABC, abstractmethod

class DataSourceReader(ABC):
    @abstractmethod
    def read(self, path: str, **kwargs) -> pd.DataFrame:
        pass

class JSONReader(DataSourceReader):
    def read(self, path: str, **kwargs) -> pd.DataFrame:
        with open(path, 'r', encoding='utf-8') as f:
            data = json.load(f)
        # 支持单条记录或记录列表
        records = data if isinstance(data, list) else [data]
        return pd.DataFrame(records)

class XMLReader(DataSourceReader):
    def read(self, path: str, **kwargs) -> pd.DataFrame:
        records = []
        for event, elem in ET.iterparse(path, events=['end']):
            record = {child.tag: child.text for child in elem}
            records.append(record)
            elem.clear()  # 释放内存，防止 XML 文档过大时 OOM
        return pd.DataFrame(records)

class CSVReader(DataSourceReader):
    def read(self, path: str, **kwargs) -> pd.DataFrame:
        return pd.read_csv(path, **kwargs)

class DBReader(DataSourceReader):
    def __init__(self, conn):
        self.conn = conn

    def read(self, query: str, **kwargs) -> pd.DataFrame:
        return pd.read_sql(query, self.conn, **kwargs)

# 工厂模式统一调用
def get_reader(source_type: str, **conn_params) -> DataSourceReader:
    readers = {
        'json': JSONReader(),
        'xml': XMLReader(),
        'csv': CSVReader(),
        'db': DBReader(conn_params['conn']),
    }
    return readers[source_type]
```

---

## 9. 数据治理 — 元数据管理与隐私保护

### 9.1 元数据管理

元数据分为三类：**技术元数据**（表名、字段类型、血缘关系、ETL 作业信息）、**业务元数据**（指标定义、业务口径、数据Owner）、**操作元数据**（数据访问日志、运行日志）。实践中推荐 Apache Atlas 或 DataHub 作为元数据管理平台，但面试中更重要的是展示对元数据价值的理解——元数据平台不是目的，数据可发现性（能找到想要的数据）、数据可解释性（能理解数据的业务含义）和数据合规性（能满足审计要求）才是目的。

### 9.2 隐私保护与数据脱敏

```sql
-- 敏感字段脱敏策略（根据字段类型选择脱敏方式）
-- 手机号：保留前三位和后四位，中间打码
SELECT CONCAT(SUBSTRING(phone, 1, 3), '****', SUBSTRING(phone, -4)) AS phone_masked
FROM dwd_user_info;

-- 身份证号：只显示前六位和后四位（出生地和出生日信息保留行政区划）
SELECT CONCAT(SUBSTRING(id_card, 1, 6), '**********', SUBSTRING(id_card, -4)) AS id_card_masked;

-- 金额字段：加噪处理（用于数据分析场景，不影响统计趋势）
SELECT AVG(order_amt + (RAND() - 0.5) * 10) AS order_amt_noised
FROM dwd_trade_order_di WHERE dt = '20240101';

-- 动态脱敏：按用户权限级别决定脱敏程度
CASE data_access_level
    WHEN 'FULL'     THEN phone
    WHEN 'PARTIAL'  THEN CONCAT(SUBSTRING(phone,1,3),'****',SUBSTRING(phone,-4))
    WHEN 'NONE'     THEN '***'
END AS phone_displayed
```

数据脱敏的核心原则：**生产环境严格脱敏、测试环境使用合成数据、分析场景加噪而非完全遮盖**。GDPR 合规要求下，用户同意书范围外的数据不得用于分析，脱敏是技术兜底手段而非合规替代。

---

## 面试高频问答

**Q1: 数仓分层的目的和代价是什么？**

分层核心目的是解耦数据加工链路、实现 SQL 复用、控制单条 SQL 复杂度。代价是存储成本增加（每层都有一份数据）、数据链路变长（延迟累加）、维护成本上升（多了 ETL 任务和依赖管理）。实践中要权衡：对于简单业务（10 张以内源表），两层足够；对于复杂业务（50+ 表），三层起步。

**Q2: 拉链表查询效率低怎么优化？**

常用方案：一是按时间分区 + 拉链表组合（`dt` 分区字段存储每天的变更快照，拉链表存储历史变更链）；二是建立拉链表的物化视图或预聚合表；三是将拉链表同步到 ClickHouse / Druid 等列存引擎，利用列式存储的向量化查询加速。核心思路是牺牲部分存储换查询效率。

**Q3: 每日百万级数据 ETL 耗时 4 小时，怎么优化？**

优化路径按性价比排序：优先检查 SQL 层面（是否全表扫描、是否有不必要的 JOIN、是否缺少分区裁剪）；其次优化任务并行度（将串行 ETL 改为依赖关系下的最大并行）；最后才是硬件扩容（加 CPU / 内存 / 换 SSD）。常见瓶颈：ODS→DWD 层通常是 IO 密集型（大量数据读取），DWS 汇总层是 CPU 密集型（大量聚合计算），针对性优化。

**Q4: APScheduler 任务漏调了怎么办？**

APScheduler 配置 `misfire_grace_time` 容忍一定时间内的失位。漏调后的补救策略：建立 ETL 补数机制——手动触发或定时扫描任务状态表，对比"应该执行时间"和"实际执行时间"，自动补跑缺失的分区数据。生产中推荐记录每次任务执行状态（成功/失败/跳过）到数据库，便于监控和补数。

**Q5: 数据延迟（上游数据迟到）怎么处理？**

常见方案：一是设置等待窗口（配置"最晚数据到达时间"，超过窗口则跳过或使用前一日数据）；二是多层级联等待（下游任务轮询上游分区是否就绪）；三是 Lambda 架构（实时层 + 批处理层合并，实时层延迟低但可能不准确，批处理层定期修正）。生产中通常设置两个告警：数据迟到 30 分钟提示、数据迟到 2 小时升级。

**Q6: 数据血源在实际工作中最大的价值是什么？**

影响分析是血缘最重要的应用场景。当某个上游字段需要下线或修改时，通过血缘快速列出所有受影响的下游表和报表，避免改一处坏一片。在面试中可以举具体例子：公司要下线一个废弃字段，通过血缘分析发现该字段实际被 3 个 DWD 表、2 个 DWS 表、1 个 ADS 表和 2 张 BI 报表依赖，字段下线需要协调多方确认，这就是血缘的工程价值。

---

## 附录：关键配置参考

| 场景 | 推荐配置 |
|------|----------|
| APScheduler 日级任务 | `CronTrigger(hour=2, minute=0)`, `misfire_grace_time=3600` |
| Redis 分布式锁 TTL | `2x 任务预估执行时间`，最少 30 分钟 |
| 数据质量空值率告警 | 常规表 < 1%，核心表 < 0.01% |
| 增量 CDC 时间窗口 | 上游时区对齐 + 边界值用 `>` 而非 `>=` 防止重复 |
| MySQL 分区策略 | 按日期分区 `PARTITION BY RANGE (TO_DAYS(dt))`，每月底自动建下月分区 |
