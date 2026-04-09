# MySQL 数据库实战手册

## Summary

本手册面向具有数据仓库 ETL 实战经验（ODS→DWD→DWS 三层建模、日处理千万级数据、增量更新与 BI 报表开发）的高级 AI 应用工程师，系统梳理 MySQL 从基础使用到高级原理的完整知识点。内容覆盖 MySQL 基本架构与常用操作、存储引擎差异与选型、索引原理与最左前缀、ACID 事务与 MVCC/Next-Key Lock 机制、锁类型与死锁处理、SQL 调优方法论、数仓分层设计、主从复制与读写分离、分库分表策略、备份恢复方案，以及拉链表实现缓慢变化维的完整思路。每章均配有生产级 SQL 示例，最终以 10 道高频面试题收尾，帮助面试者快速定位知识盲区。

---

## 零、基础知识速查

### 0.1 MySQL 基本架构

MySQL 采用 **C/S 架构**，主要由以下组件构成：

```
客户端
  │
  ▼
连接器（Connection Pool）→ 管理连接、权限验证
  │
  ▼
分析器（Parser）        → 词法分析、语法分析 → 生成解析树
  │
  ▼
优化器（Optimizer）     → 选择索引、决定 JOIN 顺序、生成执行计划
  │
  ▼
执行器（Executor）      → 调用存储引擎接口，读写数据
  │
  ▼
存储引擎（InnoDB / MyISAM）→ 真正读写磁盘 / 内存
```

**InnoDB** 是默认存储引擎，支持事务、行级锁、外键、崩溃恢复。**MyISAM** 不支持事务，仅支持表级锁，适合只读场景。

### 0.2 常用 SQL 操作速查

```sql
-- 数据库操作
SHOW DATABASES;
CREATE DATABASE IF NOT EXISTS soc_db DEFAULT CHARSET utf8mb4;
USE soc_db;
DROP DATABASE IF EXISTS soc_db;

-- 表操作
SHOW TABLES;
DESC alert_events;               -- 查看表结构
SHOW CREATE TABLE alert_events;  -- 查看建表语句

CREATE TABLE alert_events (
    id        BIGINT PRIMARY KEY AUTO_INCREMENT,
    alert_id  VARCHAR(64) NOT NULL UNIQUE,
    severity  ENUM('P0','P1','P2','P3') NOT NULL DEFAULT 'P2',
    src_ip    VARCHAR(45),
    dst_ip    VARCHAR(45),
    payload   JSON,
    status    TINYINT DEFAULT 0 COMMENT '0=待处理,1=已确认,2=已处置',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_severity (severity),
    INDEX idx_created (created_at),
    INDEX idx_src_ip (src_ip)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

ALTER TABLE alert_events ADD COLUMN attack_stage VARCHAR(32) AFTER severity;
ALTER TABLE alert_events DROP COLUMN payload;
DROP TABLE IF EXISTS alert_events;

-- 数据操作（CRUD）
INSERT INTO alert_events (alert_id, severity, src_ip) VALUES ('A001', 'P1', '192.168.1.10');
INSERT INTO alert_events (alert_id, severity, src_ip) VALUES
    ('A002', 'P0', '10.0.0.1'),
    ('A003', 'P2', '172.16.0.5');

SELECT * FROM alert_events WHERE severity = 'P0' ORDER BY created_at DESC LIMIT 10;
SELECT severity, COUNT(*) as cnt FROM alert_events GROUP BY severity;
SELECT src_ip, COUNT(*) as total FROM alert_events GROUP BY src_ip HAVING total > 5;

UPDATE alert_events SET status = 2, updated_at = NOW() WHERE alert_id = 'A001';
UPDATE alert_events SET severity = 'P0' WHERE severity = 'P1' AND status = 0;  -- 谨慎用

DELETE FROM alert_events WHERE status = 0 AND created_at < DATE_SUB(NOW(), INTERVAL 30 DAY);

-- 事务
START TRANSACTION;
UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
COMMIT;   -- 或 ROLLBACK;
```

### 0.3 表设计基础

**三大范式（NF）：**
- **1NF**：字段原子性，不可再分（如"地址"应拆为"城市+区县+街道"）
- **2NF**：消除部分依赖，非主键字段完全依赖主键（联合主键场景）
- **3NF**：消除传递依赖，非主键字段不依赖其他非主键字段

**实际设计原则：**
```sql
-- 主键设计：优先使用自增 BIGINT 或 UUID
id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY  -- 常见做法

-- 大文本字段单独存
CREATE TABLE alerts (
    id BIGINT PRIMARY KEY,
    summary VARCHAR(200),        -- 告警摘要，直接展示
    detail TEXT,                  -- 告警详情，单独字段
    -- 而不是把 detail 和 summary 混在一起
    INDEX idx_summary (summary)
);

-- 常用字段类型选择
TINYINT   -- 0-255，状态码、开关
INT       -- 42亿范围，计数、ID（分库分表前）
BIGINT    -- 超大范围，主键、日志量
VARCHAR   -- 255以内可变性字段（姓名、IP）
TEXT      -- >255 的文本，描述、JSON
JSON      -- MySQL 5.7+，存储半结构化数据
DATETIME  -- '2026-04-09 14:30:00'，绝对时间
TIMESTAMP -- 带时区的时间戳，更小但有2038问题
```

### 0.4 索引基础概念

```sql
-- 主键索引（唯一且非空，一个表最多一个）
PRIMARY KEY (id)

-- 唯一索引（值唯一，可为空）
UNIQUE INDEX idx_alert_id (alert_id)

-- 普通索引（可重复）
INDEX idx_severity (severity)
INDEX idx_src_ip (src_ip, created_at)  -- 联合索引

-- 查看查询计划（最重要！）
EXPLAIN SELECT * FROM alert_events WHERE severity = 'P0';
-- 关键字段：
-- type: const > eq_ref > ref > range > index > ALL（尽量避免 ALL）
-- key: 实际使用的索引
-- rows: 扫描行数
-- Extra: Using index（覆盖索引）/ Using where / Using filesort（危险！）

-- 索引失效的常见场景
SELECT * FROM alert_events WHERE severity != 'P0';       -- != 不用索引
SELECT * FROM alert_events WHERE severity LIKE '%P0%';    -- 前导 % 不用索引
SELECT * FROM alert_events WHERE YEAR(created_at) = 2026; -- 函数不用索引
SELECT * FROM alert_events WHERE src_ip + 1 = 10;        -- 运算不用索引
```

### 0.5 常见面试基础问题

**Q: CHAR 和 VARCHAR 的区别？**
| 维度 | CHAR | VARCHAR |
|------|------|---------|
| 存储方式 | 固定长度，不足用空格填充 | 可变长度，额外1-2字节存长度 |
| 最大长度 | 255 | 65535（字符集相关）|
| 适用场景 | 固定长度（性别、状态码、手机号） | 可变长度（姓名、地址、描述）|
| 性能 | 更快（固定长度，内存连续）| 略慢（需读取长度）|

**Q: DATETIME 和 TIMESTAMP 的区别？**
- `DATETIME`：8字节，范围 1000-01-01 ~ 9999-12-31，不带时区
- `TIMESTAMP`：4字节，范围 1970-2038，带时区（自动转换）
- 生产中如果需要时区支持，用 `BIGINT` 存时间戳更通用

**Q: COUNT(*) vs COUNT(1) vs COUNT(column)？**
- `COUNT(*)`：MySQL 优化器会直接计数，不取值，InnoDB 最快
- `COUNT(1)`：和 `COUNT(*)` 几乎一样，MySQL 内部优化为等价操作
- `COUNT(column)`：需要判断是否为 NULL，不计数 NULL 值

---

## 一、存储引擎：InnoDB vs MyISAM

### 核心差异

| 特性 | InnoDB | MyISAM |
|---|---|---|
| 事务支持 | ACID 事务（REPEATABLE-READ） | 不支持 |
| 锁粒度 | 行锁 + 间隙锁 | 表锁 |
| 外键约束 | 支持 | 不支持 |
| 聚簇索引 | 是（主键组织数据） | 否（堆表，非聚簇） |
| 崩溃恢复 | 自动通过 redo log 恢复 | 需手动 repair |
| 并发性能 | 高（行级锁） | 低（表锁） |
| 全文索引 | 5.6+ 支持 | 原生支持 |
| 存储空间 | 约 2x（双写缓冲等结构） | 较小 |

### 为什么 InnoDB 是默认引擎

InnoDB 凭借事务安全、行级锁、高并发和崩溃自恢复能力，成为 OLTP 场景的事实标准。MyISAM 仅在纯读/历史归档（无需事务、并发写入极低）场景下有少量保留空间。

### 聚簇索引 vs 非聚簇索引

- **聚簇索引**：数据行直接存储在 B+ 树的叶子节点中，索引即数据。一个表只有一个聚簇索引（通常为主键）。InnoDB 通过主键聚集数据，无主键时选用唯一键，否则生成隐藏 row_id。
- **非聚簇索引**：叶子节点仅存储索引列值和主键值，查询需回表（先查索引得到主键，再查主键索引获取数据行）。MyISAM 所有索引均为非聚簇，InnoDB 的二级索引也遵循此规则。

```sql
-- InnoDB 聚簇索引示例：主键 age 直接组织数据顺序
CREATE TABLE user (
    id BIGINT PRIMARY KEY,  -- 聚簇索引，数据按 id 顺序物理存储
    name VARCHAR(50),
    age INT,
    KEY idx_age (age)       -- 二级索引，叶子存 (age, id)，需回表
) ENGINE=InnoDB;

-- 非聚簇索引查找过程（idx_age）
-- 1. 在 idx_age B+ 树查 age=30，得到主键 id 列表
-- 2. 用主键 id 回主键索引获取完整行
-- 3. 若 SELECT id, age 可直接返回，无需回表（覆盖索引）
```

---

## 2. 索引

### 索引类型与创建

```sql
-- 主键索引：唯一且非空，每个表最多一个
ALTER TABLE dwd_orders ADD PRIMARY KEY (order_id);

-- 唯一索引：允许多个 NULL
CREATE UNIQUE INDEX uk_order_no ON dwd_orders(order_no);

-- 普通索引：加速查询
CREATE INDEX idx_merchant_id ON dwd_orders(merchant_id);

-- 联合索引：多列组合
CREATE INDEX idx_status_create ON dwd_orders(status, create_time);
```

### 最左前缀原则

联合索引 `(status, create_time, merchant_id)` 的索引树按 status→create_time→merchant_id 顺序构建。查询必须包含最左列才能使用索引：

- `WHERE status = 'paid'` — 使用索引（左前缀）
- `WHERE status = 'paid' AND create_time > '2026-01-01'` — 使用索引
- `WHERE create_time > '2026-01-01'` — **不使用索引**（跳过最左列）
- `WHERE status = 'paid' AND merchant_id = 100` — 仅使用 status 列（Extra: Using index condition 表示索引下推生效）

### 覆盖索引

查询的所有列都存在于索引中，无需回表，查询效率最高：

```sql
-- 假设存在联合索引 (status, create_time)
EXPLAIN SELECT status, create_time  -- 直接从索引返回，覆盖索引
FROM dwd_orders
WHERE status = 'paid' AND create_time > '2026-04-01';
```

### 索引下推（Index Condition Pushdown, ICP）

MySQL 5.6+ 引入 ICP，在非聚簇索引遍历时将 WHERE 条件下推到索引层过滤，减少回表次数：

```sql
-- 不使用 ICP：先按最左前缀定位记录，再逐一回表过滤 merchant_id
-- 使用 ICP：直接在二级索引内过滤 merchant_id，减少回表次数
EXPLAIN SELECT * FROM dwd_orders
WHERE status = 'paid'
  AND merchant_id = 1001
  AND create_time BETWEEN '2026-04-01' AND '2026-04-09';
-- Extra 列出现 "Using index condition" 即表示 ICP 生效
```

---

## 3. 事务与 ACID

### 四种隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 默认引擎 |
|---|---|---|---|---|
| READ UNCOMMITTED | 可能 | 可能 | 可能 | — |
| READ COMMITTED | 不可能 | 可能 | 可能 | Oracle |
| REPEATABLE READ（RR） | 不可能 | 不可能 | 可能* | MySQL InnoDB |
| SERIALIZABLE | 不可能 | 不可能 | 不可能 | — |

MySQL InnoDB 在 RR 级别下通过 Next-Key Lock 基本消除了幻读，Oracle 需 SERIALIZABLE 才能解决。

### MVCC 实现（多版本并发控制）

InnoDB 为每行记录维护两个隐藏列：`DATA_TRX_ID`（最近修改的事务ID）和 `DATA_ROLL_PTR`（指向 undo log 的指针）：

- **读已提交（RC）**：每次读取生成新的 ReadView，过滤掉活跃事务列表中未提交的事务数据。
- **可重复读（RR）**：事务开始时生成一次 ReadView，整个事务内复用，快照读始终一致。
- **快照读 vs 当前读**：普通 SELECT 为快照读（无锁），`SELECT ... FOR UPDATE / LOCK IN SHARE MODE` 为当前读（读取最新数据并加锁）。

```sql
-- 快照读：始终读取事务开始时的数据快照
START TRANSACTION;
SELECT balance FROM account WHERE id = 1;  -- 始终返回事务开启时的值

-- 当前读：读取最新数据，适用于需要写入的场景
SELECT balance FROM account WHERE id = 1 FOR UPDATE;
UPDATE account SET balance = balance - 100 WHERE id = 1;
```

### Next-Key Lock（临键锁）

Next-Key Lock = 记录锁（Record Lock）+ 间隙锁（Gap Lock），锁住索引记录及其前后区间，是 RR 级别防止幻读的核心机制：

```sql
-- session A：锁住 id=10 的记录及其 (5,10] 和 (10,20) 间隙
SELECT * FROM orders WHERE id = 10 FOR UPDATE;

-- session B：以下操作均被阻塞（间隙被锁定）
INSERT INTO orders VALUES (15, ...);  -- 插入间隙内，阻塞
INSERT INTO orders VALUES (10, ...);  -- 锁记录本身，阻塞
UPDATE orders SET ... WHERE id = 10; -- 阻塞
```

在唯一索引等值查询时，Next-Key Lock 会退化为记录锁；范围查询（如 `id > 5 AND id < 20`）则锁定多个 Next-Key 区间。

---

## 4. 锁机制

### 共享锁 vs 排他锁

- **共享锁（S锁）**：`SELECT ... LOCK IN SHARE MODE`，允许并发读取，阻塞写操作。
- **排他锁（X锁）**：`SELECT ... FOR UPDATE` 或 DML 操作，阻塞其他读写请求。

### 表锁 vs 行锁

- **表锁**：MyISAM 默认，InnoDB 一般不加表锁（MDL 锁除外）。`LOCK TABLES` 是显式表锁，开销小但粒度粗。
- **行锁**：InnoDB 实现，锁住索引记录而非物理行。行锁在以下情况会升级为表锁：没有索引列WHERE条件、全表扫描、或事务中使用了 `LOCK TABLES`。

```sql
-- 行锁示例：id 是主键（索引），仅锁一行
BEGIN;
UPDATE orders SET status = 'shipped' WHERE id = 10001;
COMMIT;  -- 提交后释放锁

-- 无索引时全表锁（危险！）
UPDATE orders SET status = 'shipped' WHERE merchant_name = '店铺A'; -- 退化为表锁
```

### 间隙锁（Gap Lock）

锁定索引记录之间的空隙，防止幻读插入。间隙锁之间互不冲突，目的是阻塞其他事务在间隙内插入数据。

### 死锁产生与避免

**产生条件**（同时满足）：
1. 互斥：资源只能被一个事务持有
2. 持有并等待：持有资源的同时请求其他资源
3. 不可抢占：已持有的锁不能被强制释放
4. 循环等待：事务间形成锁等待环

```sql
-- 经典死锁场景
-- T1: 锁 A 记录 -> 等待 B 记录
-- T2: 锁 B 记录 -> 等待 A 记录
-- T1: UPDATE orders SET status='paid' WHERE id = 1;  -- 锁住 id=1
-- T2: UPDATE orders SET status='paid' WHERE id = 2;  -- 锁住 id=2
-- T1: UPDATE orders SET amount=100 WHERE id = 2;     -- 等待 T2 释放 id=2
-- T2: UPDATE orders SET amount=100 WHERE id = 1;     -- 等待 T1 释放 id=1 → DEADLOCK
```

**避免策略**：
- 按固定顺序访问多条记录（T1、T2 均先锁 id=1 再锁 id=2）
- 减少事务时长，快进快出
- 尽量在索引列上操作，避免全表扫描引发的行锁膨胀
- 合理使用低隔离级别（读已提交）
- 开启 `innodb_deadlock_detect = ON`（默认开启）自动检测死锁并回滚代价最小的事务

---

## 5. SQL 优化

### EXPLAIN 核心字段解读

```sql
EXPLAIN SELECT o.order_id, u.name, o.amount
FROM dwd_orders o
JOIN dwd_user u ON o.user_id = u.id
WHERE o.status = 'paid'
  AND o.create_time >= '2026-04-01';

-- 关键字段说明：
-- type:       ALL(全表) < index < range < ref < eq_ref < const
--            目标：至少达到 range，避免 ALL
-- key:        实际使用的索引名
-- rows:       预估扫描行数，越少越好
-- filtered:   服务端过滤后剩余行数百分比
-- Extra:
--   Using index           → 覆盖索引，无需回表
--   Using index condition → 索引下推（ICP）
--   Using where           → 服务端过滤，索引列无法覆盖
--   Using filesort       → 无法使用索引排序，需额外排序
--   Using temporary      → 产生临时表（如 DISTINCT/GROUP BY 脱索引）
```

### like %abc 索引失效与优化

```sql
-- 失效：前导通配符无法利用 B+ 树有序性
SELECT * FROM dwd_orders WHERE order_no LIKE '%202604%';

-- 优化方案：
-- 1. 改用后缀匹配（需额外维护 reversed 列并建索引）
-- 2. ES / 全文索引处理模糊搜索
-- 3. 如果 order_no 有固定前缀模式，改为前导匹配
SELECT * FROM dwd_orders WHERE order_no LIKE 'PT202604%';  -- 可用索引
```

### JOIN 优化

```sql
-- 原则：小表驱动大表（让 JOIN 字段有索引的表作为驱动表）
-- MySQL optimizer_switch 决定 JOIN 顺序（小结果集优先）

-- 优化前：未限制状态，全表 JOIN
SELECT o.*, u.name
FROM dwd_orders o
JOIN dwd_user u ON o.user_id = u.id  -- o 无索引，扫描全表

-- 优化后：利用状态过滤缩小驱动表，同时建立 user_id 索引
ALTER TABLE dwd_orders ADD INDEX idx_user_id_status (user_id, status);

SELECT o.*, u.name
FROM dwd_orders o
JOIN dwd_user u ON o.user_id = u.id
WHERE o.status = 'paid';  -- 先过滤，减少 JOIN 数据量
```

### 分页优化

```sql
-- 低效：深度分页，偏移量大时性能急剧下降
SELECT * FROM dwd_orders ORDER BY create_time DESC LIMIT 1000000, 10;
-- 问题：MySQL 先扫描并丢弃前 100 万行

-- 优化方案 1：游标分页（利用主键单调性）
SELECT * FROM dwd_orders
WHERE create_time < '2026-04-09 10:00:00'  -- 上一页最后一条的时间戳
  AND id < 1000000
ORDER BY create_time DESC, id DESC
LIMIT 10;

-- 优化方案 2：延迟关联（覆盖索引减少回表）
SELECT o.* FROM dwd_orders o
INNER JOIN (
    SELECT id FROM dwd_orders ORDER BY create_time DESC LIMIT 1000000, 10
) t USING(id);
```

---

## 6. ODS→DWD→DWS 三层建模

### 各层职责

| 层级 | 全称 | 职责 | 数据特征 |
|---|---|---|---|
| ODS | Operational Data Store | 原始数据层，镜像源系统 | 保持原始结构，一对一同步 |
| DWD | Data Warehouse Detail | 明细数据层，ETL 清洗 | 结构化、去重、异常值处理、统一口径 |
| DWS | Data Warehouse Summary | 汇总数据层，主题宽表 | 按主题聚合，轻度汇总 |

### DWD 层清洗逻辑

```sql
-- DWD 层典型清洗任务（每日增量 ODS → DWD）
-- 1. 去重：同批次内同一业务主键多条记录，取最新
INSERT OVERWRITE TABLE dwd_orders PARTITION(dt = '${bizdate}')
SELECT order_id, user_id, merchant_id, amount, status, create_time, update_time
FROM (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY update_time DESC) AS rn
    FROM ods_orders
    WHERE dt = '${bizdate}'
) t
WHERE rn = 1;

-- 2. 空值填充与类型标准化
-- 3. 枚举值统一（如 status: 'PAID'/'pay'/'1' → 'paid'）
-- 4. 敏感数据脱敏（手机号、身份证中间四位打码）
```

### 缓慢变化维（SCD）

数据仓库中，维度属性随时间变化时如何处理。常见策略：

- **SCD Type 1**：覆盖旧值（如商家名称变更，直接覆盖）
- **SCD Type 2**：新增历史行（拉链表，记录每版数据起止时间）
- **SCD Type 3**：保留新旧两个值（仅记录最近一次变化）

---

## 7. 主从复制

### 异步 vs 半同步

| 模式 | 原理 | 延迟 | 数据安全 |
|---|---|---|---|
| 异步复制 | 主库提交后立即返回，不等从库 | 低（不等从库） | 低（主库崩可能丢数据） |
| 半同步复制 | 主库等至少一个从库 ACK 后返回 | 中 | 中（至少一个从库确认） |
| 全同步复制 | 全部从库 ACK 后才返回 | 高 | 高 |

MySQL 5.7+ 支持 **半同步复制（Lossless）**，保证数据不丢失。

### 主从延迟原因与解决

```sql
-- 常见延迟原因：
-- 1. 大事务：单条 SQL 执行时间过长，从库串行回放耗时更久
-- 2. 从库配置低：CPU/磁盘 IO 不及主库
-- 3. 网络抖动
-- 4. 从库并发回放（slave_parallel_workers）未开启或配置不当

-- 诊断：查看从库 Seconds_Behind_Master
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master: 0  表示同步，无延迟

-- 解决思路：
-- 1. 拆大事务为小事务批量提交
-- 2. 开启从库多线程并发回放（必须使用 GTID 模式）
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';
SET GLOBAL slave_parallel_workers = 8;  -- 建议 CPU 核数的 1/2~2/3

-- 3. 读写分离：读请求打到从库，写请求打到主库
```

### 读写分离

应用层通过数据库中间件（如 ShardingSphere-Proxy、MyCat）或框架内置读写分离路由实现。主库处理写与事务内读，从库处理普通查询。注意：主从同步有延迟，跨主从的事务一致性需使用 GTID + 注解路由。

---

## 8. 分库分表

### 水平 vs 垂直分表

- **水平分表**：将同一表的数据按行拆分到多张表中（分片）。如按 user_id % 4 拆成 user_0/1/2/3。
- **垂直分表**：按列拆分热点列到独立表。如大字段 content 单独成表，减少主表查询 IO。

### ShardingSphere 介绍

Apache ShardingSphere 是主流分库分表中间件，提供 ShardingSphere-JDBC（Java 应用端分片路由，无代理层延迟）和 ShardingSphere-Proxy（数据库代理，支持 MySQL/PostgreSQL 协议，客户端透明）两种部署模式。核心能力包括：分片算法（精确分片、范围分片、hint 强制路由）、分布式主键（雪花算法）、分布式事务（XA/Saga）。

### 分片键选择

分片键应满足：**高基数**（唯一值多）、**查询频繁**（WHERE 条件必备）、**分布均匀**（避免热点分片）。

```sql
-- 按 user_id 分库，按 order_id 分表（双层分片）
-- sharding-databases.yaml 配置
databaseStrategy:
  standard:
    shardingColumn: user_id
    shardingAlgorithmName: mod_strategy
tableRule:
  orders:
    actualDataNodes: ds_${0..1}.orders_${0..3}
    tableStrategy:
      standard:
        shardingColumn: order_id
        shardingAlgorithmName: mod_strategy
```

### 跨分片查询

```sql
-- 跨分片分页：需要合并各分片结果再排序分页（内存压力）
-- 方案：应用层聚合或使用 ShardingSphere 跨节点查询能力

-- 跨分片 JOIN：业务分片键必须一致，否则需广播表或多次查询组装
-- 示例：user_id 和 order_id 均有分片键，无法直接 JOIN
-- 解决：先按 user_id 查用户，再批量查询订单，应用层关联

-- 分片键以外的查询：全路由或 ES 兜底
```

### ID 生成方案

| 方案 | 原理 | 优点 | 缺点 |
|---|---|---|---|
| UUID | 随机 128 位 | 简单，不依赖中心 | 无序，存储大，索引效率低 |
| 雪花算法 | 时间戳 + 机器ID + 序列号 | 有序，有序数字，查询友好 | 依赖时钟，跨机房需特殊处理 |
| 分布式 ID 表 | 数据库自增主键 | 简单有序 | 单点瓶颈，需批量获取 |
| Leaf（美团） | 雪花 + ZK/数据库 | 高可用，有序 | 运维复杂 |

```sql
-- 雪花算法 Java 实现示意（简化）
-- 41bit 时间戳（毫秒，2026 年约用 30 年）+ 10bit 机器 ID + 12bit 序列号
-- 时间回拨时拒绝服务或等待追上
```

---

## 9. 备份与恢复

### 全量 vs 增量备份

```bash
# 全量备份：mysqldump，适合数据量 < 100GB
mysqldump -h127.0.0.1 -uroot -p \
  --single-transaction \    # InnoDB 使用 MVCC 快照备份，不锁表
  --master-data=2 \         # 记录备份时的 binlog position
  --routines --triggers \
  --databases mydb > full_backup_$(date +%Y%m%d).sql

# 增量备份：备份 binlog 文件（自上次备份后的增量变更）
# 通过 mysqlbinlog 从指定 position 增量提取
mysqlbinlog --start-datetime='2026-04-08 00:00:00' \
            --stop-datetime='2026-04-09 00:00:00' \
            mysql-bin.000123 > incremental_backup.sql
```

### xtrabackup 原理

Percona xtrabackup 是生产级热备工具，核心原理：

1. **拷贝阶段**：innobackupex 启动一个后台线程，持续扫描 buffer pool，将脏页（dirty page）复制到备份目录；同时复制所有物理 InnoDB 文件（ibdata1, .ibd 文件）。
2. **日志重做**：备份期间记录 LSN（Log Sequence Number），备份结束后执行 `xtrabackup --prepare`，将 checkpoint LSN 后的 redo log 应用到备份文件，确保数据文件处于一致状态。
3. **恢复**：将备份文件拷贝回数据目录，执行 `innodb_force_recovery`（生产环境慎用）或正常启动即可。

```bash
# 全量热备
innobackupex --user=root --password=xxx --parallel=4 /data/backup/

# 增量备份（基于上次全量）
innobackupex --incremental /data/backup/incr/ \
  --incremental-basedir=/data/backup/full/ \
  --user=root --password=xxx

# 准备（prepare）全量 + 增量合并
innobackupex --apply-log --redo-only /data/backup/full/
innobackupex --apply-log --redo-only /data/backup/full/ \
  --incremental-dir=/data/backup/incr/2026-04-08/
innobackupex --apply-log /data/backup/full/  # 最后一次不做 --redo-only
```

### binlog 数据恢复

```sql
-- 基于 binlog 恢复误删数据（point-in-time recovery）
-- 1. 找到误操作前最近的 binlog position
SHOW BINLOG EVENTS IN 'mysql-bin.000124' LIMIT 100;

-- 2. 从全量备份恢复
-- mysql -uroot -p mydb < full_backup_20260408.sql

-- 3. 应用 binlog 到误操作前的 position
mysqlbinlog --stop-position=12345678 \
            --database=mydb \
            mysql-bin.000123 mysql-bin.000124 | mysql -uroot -p mydb

-- 4. 恢复误删的特定数据（已知 DELETE 时间范围）
mysqlbinlog --start-datetime='2026-04-09 14:00:00' \
            --stop-datetime='2026-04-09 14:30:00' \
            --database=mydb \
            mysql-bin.000124 | mysql -uroot -p mydb
```

---

## 10. 拉链表

### 拉链表原理

拉链表（SCD Type 2 实现）记录维度数据的历史全貌，适合用户、商家、商品等缓慢变化维度。相比每日快照，拉链表存储空间更优（仅存变化行）；相比全量表，可追溯任意时间点的历史状态。

```sql
-- 建表：额外两列标识时间区间
CREATE TABLE dwd_user_info_his (
    user_id    BIGINT PRIMARY KEY,
    name       VARCHAR(50),
    level      VARCHAR(20),
    phone      VARCHAR(20),
    start_date DATE NOT NULL,   -- 生效日期（含）
    end_date   DATE NOT NULL,   -- 失效日期（含），'9999-12-31' 表示当前有效
    is_current TINYINT DEFAULT 0,
    INDEX idx_user_end (end_date),
    INDEX idx_user_start (start_date)
);

-- 每日增量入链逻辑（以 user_id = 1001 为例）
-- 假设历史：1001 的 level='gold' 从 2026-01-01 生效，end_date='9999-12-31'
-- ODS 今日变化：user_id=1001，level='platinum'

-- Step 1：关闭历史链（老记录 end_date 更新为昨天，is_current=0）
UPDATE dwd_user_info_his
SET end_date = '2026-04-08', is_current = 0
WHERE user_id IN (
    SELECT user_id FROM ods_user WHERE dt = '2026-04-09'
)
  AND is_current = 1;

-- Step 2：插入今日变化记录（新记录 start_date=今天，end_date='9999-12-31'，is_current=1）
INSERT INTO dwd_user_info_his (user_id, name, level, phone, start_date, end_date, is_current)
SELECT user_id, name, level, phone, '2026-04-09', '9999-12-31', 1
FROM ods_user
WHERE dt = '2026-04-09';
```

### 拉链表查询

```sql
-- 查询某用户历史变更记录
SELECT user_id, level, start_date, end_date
FROM dwd_user_info_his
WHERE user_id = 1001
ORDER BY start_date;

-- 查询某时间点的快照（如 2026-03-01 各用户等级）
SELECT user_id, level
FROM dwd_user_info_his
WHERE start_date <= '2026-03-01'
  AND end_date >= '2026-03-01';

-- 查询当前有效的用户维度
SELECT * FROM dwd_user_info_his WHERE is_current = 1;
```

### 拉链表 vs 快照表

| 维度 | 拉链表 | 每日快照 |
|---|---|---|
| 存储 | O(变化次数)，空间最优 | O(n × 天数)，空间浪费大 |
| 历史追溯 | 任意时间点均可查询 | 只能看每日快照日期 |
| 查询性能 | 需范围过滤 end_date | 直接查询当天分区 |
| 实现复杂度 | 高（需入链逻辑） | 低（全量覆盖） |

实践建议：变化频率低（用户等级、商品分类）用拉链表；变化频繁或查询 QPS 极高时考虑每日快照 + 分区表。

---

## 面试高频问答

**Q1：MySQL 为什么用 B+ 树而不是 B 树或哈希索引？**

B+ 树所有叶子节点在同一层，查询路径长度一致（O(log n)），适合范围查询（B 树非叶子节点也存数据，相同高度数据量少，叶子间无链表，范围查询需中序遍历）；哈希索引仅支持等值查询（O(1)），不支持范围和排序，且哈希冲突时退化为链表。InnoDB 自适应哈希索引（Adaptive Hash Index）是热点数据的自动优化，对用户透明。

**Q2：什么是索引失效的常见场景？**

① 索引列参与运算（`WHERE age + 1 = 20`）；② 函数/类型转换（`WHERE LEFT(name,3)='张'`、`WHERE id = '123'` 且 id 为 INT）；③ 前导通配符（`LIKE '%abc'`）；④ 隐式类型转换；⑤ OR 前后列不一致（OR 右边列无索引时全表）；⑥ 使用 NOT IN / NOT EXISTS（在索引列上效果差，建议改用 > / <= 组合）；⑦ 数据量过小（MySQL 估算全表扫描更快时主动放弃索引）。

**Q3：MySQL 事务提交后数据一定持久化到磁盘了吗？**

不一定。事务提交（COMMIT）只保证数据写入 redo log 文件（持久化到磁盘或文件系统缓存，取决于 `innodb_flush_log_at_trx_commit` 参数）。数据页（`.ibd` 文件）由后台 page cleaner 线程异步刷盘。若设置为 0（每秒刷盘）或 2（仅刷到 OS 缓存），极端断电场景可能丢失少量已提交事务。设为 1（每次提交强制刷盘）可保证最强持久性。

**Q4：什么是幻读？Next-Key Lock 如何解决？**

幻读指同一事务内两次相同查询返回不一致的行（因其他事务插入了新行）。在 RC 隔离级别下，普通 SELECT 每次都读最新快照，不加锁，允许幻读。RR 级别下，MySQL 使用 Next-Key Lock 锁定查询范围内的所有间隙，新插入被阻塞，从而消除幻读。但 Next-Key Lock 仅锁定已存在的记录，间隙内新插入仍需应用层或 SERIALIZABLE 级别兜底。

**Q5：一条 UPDATE 语句的执行流程是什么？**

① 解析器（Parser）生成语法树；② 预处理器（Preprocessor）验证表名、列名；③ 优化器（Optimizer）选择索引和执行计划；④ 执行器（Executor）调用存储引擎 API：先获取行锁（如 `FOR UPDATE`），读取数据到 buffer pool，修改数据（写 undo log 用于回滚，写 redo log 用于持久化），更新 buffer pool 中的数据页，标记脏页；⑤ 事务提交时 redo log 刷盘，释放行锁。读未提交以下一级锁即可。

**Q6：Explain 中 type 字段有哪些值，从好到坏排序？**

`const > eq_ref > ref > range > index > ALL`。const：主键或唯一索引等值查找，最多匹配一行；eq_ref：关联查询中，被驱动表使用唯一索引等值访问；ref：普通索引等值查找，可能返回多行；range：索引范围扫描（`BETWEEN`、`IN`、`>`、`<`）；index：全索引扫描（仅扫描索引树，比 ALL 快因为索引小）；ALL：全表扫描，最差，需优化。

**Q7：DWD 层和 DWS 层的本质区别是什么？**

DWD 是**原子明细层**，一个业务过程对应一张或多张事实表，保持最细粒度（如每笔订单、每次支付），不做聚合、不做汇总，数据可回溯。DWS 是**主题汇总层**，按业务主题（如用户主题、交易主题）进行轻度聚合（如日活跃用户数、店铺日订单量），服务于上层 BI 和 ADS 直接查询。DWS 的汇总粒度由业务需求决定，一般不跨业务过程。

**Q8：主从复制延迟很大怎么排查和优化？**

排查步骤：① `SHOW SLAVE STATUS\G` 查看 `Seconds_Behind_Master` 和 `Last_SQL_Error`；② `SHOW PROCESSLIST` 确认从库 SQL 线程状态；③ 分析主库慢查询日志，确认大事务。优化方向：大事务拆小（分批提交，减少 binlog 日志量）；从库开启多线程并发回放（GTID + `slave_parallel_workers`）；升级从库硬件（SSD、CPU）；读写分离在从库延迟稳定后再启用；若延迟不可接受，考虑 Canal/Debezium + 消息队列 + 消费者方案替代传统主从。

**Q9：分库分表后如何保证 ID 全局唯一？**

常用方案：① 雪花算法（Snowflake）——推荐，本地生成无中心，性能高，需处理时钟回拨（等待追上或拒绝）；② 分布式 ID 发号器（数据库批量取段）——如 Leaf 段模式，每次从数据库批量获取 ID 段，本地累加，数据库无单点压力；③ UUID v7（有序 UUID，时间戳放高位，数据库支持越来越好）；④ 分片键内嵌机器标识（分片号 + 机器号 + 序列号）。实际选型取决于对有序性（主键索引 B+ 树平衡）、趋势递增（写入热点分布）和可用性的综合要求。

**Q10：生产环境误删了一张表，如何快速恢复？**

步骤：① 立即 STOP SLAVE（若是从库）或禁止写入防止覆盖 binlog；② 若有延迟从库（数据比主库新），可在从库 `FLASHH TABLE FOR RECOVERY` 临时恢复；③ 无从库时，用全量备份 + binlog 增量恢复：解压备份文件，找到误删前的 binlog position，执行 `mysqlbinlog --stop-position=X mysql-bin.N | mysql` 恢复数据到误删前一刻；④ 若备份较旧，恢复后继续应用后续增量的 binlog 到误删前一刻；⑤ 确认数据完整后，将数据写回主库或切换业务连接。建议：重要表开启 `DELAYED_INSERT_DELAY` 或使用 binlog 实时解析工具（Canal）同步到备份库。
