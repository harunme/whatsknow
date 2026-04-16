# 五、数据库设计（MySQL / Redis / ES） · 答案

> 本章共 37 题，覆盖 MySQL（Q1-Q14）、Redis（Q15-Q30）、Elasticsearch（Q31-Q37）

---

## 面试官快速索引

| 题号 | 核心考察点 | 难度 | 追问深度 |
|------|-----------|------|---------|
| Q1 | MySQL 选型、存储引擎 | ⭐⭐ | 可深问 MySQL vs PostgreSQL |
| Q2 | B+ 树 vs 红黑树/B树 | ⭐⭐⭐ | 可深问自适应哈希索引 |
| Q3 | ACID、redo log/undo log/MVCC | ⭐⭐⭐⭐ | 可深问两阶段提交 |
| Q4 | 隔离级别、幻读问题 | ⭐⭐⭐ | 可深问 MVCC 读视图 |
| Q5 | 数据建模ODS/DWD/DWS | ⭐⭐ | 可深问星型/雪花模型 |
| Q6 | 慢SQL排查、explain分析 | ⭐⭐⭐ | 可深问索引覆盖 |
| Q7 | 分库分表策略 | ⭐⭐⭐⭐ | 可深问 ShardingSphere |
| Q8 | SQL基础语法 | ⭐⭐ | 可深问执行顺序 |
| Q9 | JOIN类型、多表优化 | ⭐⭐ | 可深问 JOIN 算法 |
| Q10 | 索引分类、最左前缀 | ⭐⭐⭐ | 可深问索引下推 |
| Q11 | explain type字段分析 | ⭐⭐⭐ | 可深问索引访问类型 |
| Q12 | 视图VIEW | ⭐⭐ | 可深问物化视图 |
| Q13 | 存储过程 vs 函数 | ⭐⭐ | 可深问存储过程缺点 |
| Q14 | 主从复制、半同步 | ⭐⭐⭐ | 可深问 GTID |
| Q15 | Redis vs Memcached | ⭐⭐ | 可深问持久化策略 |
| Q16 | Redis 数据类型场景 | ⭐⭐ | 可深问 sorted set 原理 |
| Q17 | Kafka 选型对比 | ⭐⭐⭐ | 可深问 Kafka 吞吐原理 |
| Q18 | Redis Stream 消费者组 | ⭐⭐⭐ | 可深问 PEL 机制 |
| Q19 | RDB vs AOF、混合持久化 | ⭐⭐⭐ | 可深问 AOF 重写 |
| Q20 | 过期删除 vs 内存淘汰 | ⭐⭐ | 可深问 LRU/LFU 算法 |
| Q21 | 穿透/击穿/雪崩 | ⭐⭐⭐ | 可深问布隆过滤器 |
| Q22 | BigKey发现与处理 | ⭐⭐⭐ | 可深问 lazy free |
| Q23 | MULTI/EXEC/WATCH事务 | ⭐⭐ | 可深问Lua脚本替代 |
| Q24 | Pub/Sub vs Stream | ⭐⭐⭐ | 可深问消费者组语义 |
| Q25 | 缓存读写模式、数据一致性 | ⭐⭐⭐ | 可深问延迟双删 |
| Q26 | 多级缓存一致性 | ⭐⭐⭐ | 可深问 Caffeine |
| Q27 | Redis Lua脚本原子性 | ⭐⭐⭐⭐ | 可深问分布式锁实现 |
| Q28 | 原子操作、计数器Lua | ⭐⭐⭐ | 可深问 DECRBY ATOMIC |
| Q29 | BigKey拆解、UNLINK | ⭐⭐ | 可深问 scan vs keys |
| Q30 | Cluster vs Sentinel | ⭐⭐⭐ | 可深问 hash tags |
| Q31 | ES vs MySQL 选型 | ⭐⭐ | 可深问倒排索引原理 |
| Q32 | 倒排索引原理 | ⭐⭐⭐ | 可深问 FST 结构 |
| Q33 | Shard/Replica 机制 | ⭐⭐⭐ | 可深问 shard 分配策略 |
| Q34 | TF-IDF vs BM25 | ⭐⭐⭐ | 可深问相关性调优 |
| Q35 | ES 查询性能优化 | ⭐⭐⭐ | 可深问 filter cache |
| Q36 | Query DSL、filter vs query | ⭐⭐⭐ | 可深问 terms agg 原理 |
| Q37 | ES聚合查询应用 | ⭐⭐ | 可深问 pipeline agg |

---

## 5.1 MySQL

### Q1：MySQL 是什么？MySQL 和 PostgreSQL、SQLite 的核心区别是什么？你们的 SOC 平台为什么选择 MySQL 作为主数据库？

**考察点**：关系型数据库选型、存储引擎

**答案要点**：
- **MySQL vs PostgreSQL vs SQLite**：
  | 维度 | MySQL | PostgreSQL | SQLite |
  |------|-------|-----------|--------|
  | 类型 | 关系型 + 多种存储引擎 | 纯关系型（单引擎） | 嵌入式（无独立服务） |
  | 事务支持 | InnoDB 完整支持 ACID | 最完整的 ACID 支持 | 有限 ACID |
  | 性能特点 | 读密集型性能好 | 复杂查询（CTE/Window）强 | 小数据极快，无并发 |
  | SQL 标准 | 大部分兼容 | 高度兼容（CTE/Window/JSON） | 基础 SQL |
  | 并发 | MyISAM 表锁差，InnoDB MVCC | MVCC + 行级锁 | 文件锁 |
  | 适用场景 | Web 应用、互联网、读多写少 | 数据仓库、复杂分析 | 移动端、边缘设备、测试 |
- **SOC 平台选择 MySQL 原因**：
  1. 团队对 MySQL 经验积累深厚，运维成熟
  2. InnoDB 的 MVCC 对并发读写友好（告警实时写入 + 查询）
  3. 生态丰富（MySQL Router、MySQL Shell、Percona Toolkit）
  4. 和现有中间件（MySQL Proxy/Canal）集成方便

**可能的追问**：
- MySQL InnoDB 和 MyISAM 存储引擎的区别？
- 什么是 MySQL 的表空间（tablespace）和数据字典？
- 你们 SOC 平台的 MySQL 主从复制是怎么配置的？

---

### Q2：MySQL InnoDB 的索引结构为什么选择 B+ 树而不是 B 树或红黑树？

**考察点**：B+ 树优势、磁盘 IO、索引结构

**答案要点**：
- **为什么不用红黑树**：
  - 红黑树是二叉树，树高 = log₂(n)，百万数据树高约 20
  - 同样数据量，B+ 树每个节点可容纳更多键（16KB 一页），树高仅 3-4
  - B+ 树**磁盘 IO 次数更少**（每次 IO 读取一页，约 16KB）
- **为什么不用 B 树**：
  - B 树所有节点都存数据，相同页大小下，B+ 树的非叶子节点能存更多索引项
  - B+ 树非叶子节点不存数据，同等大小页面可容纳更多索引 → **更矮**
  - **查询路径更稳定**：B 树查到数据可能在任意层级，B+ 树必须查到叶子节点（所有数据在叶子）
  - **范围查询友好**：B+ 树叶子节点用双向链表连接，范围查询无需回溯

**可能的追问**：
- B+ 树的阶（degree）由什么决定？（页大小 / (键大小 + 指针大小)）
- 为什么 MySQL 将页大小设为 16KB？这个值对性能有什么影响？
- 什么是自适应哈希索引（AHI）？InnoDB 何时自动创建 AHI？

---

### Q3：什么是事务的 ACID 特性？InnoDB 如何通过 redo log、undo log、MVCC 来保证这些特性？

**考察点**：ACID 实现机制、WAL、MVCC

**答案要点**：
- **ACID 特性**：
  - **Atomic（原子性）**：事务要么全做，要么全不做
  - **Consistency（一致性）**：事务前后数据库状态一致（约束、触发器、外键）
  - **Isolation（隔离性）**：并发事务相互隔离
  - **Durability（持久性）**：事务提交后，结果永久保存
- **InnoDB 核心机制**：
  - **redo log（重做日志）**：
    - 作用：保证 **持久性 + 原子性**
    - 机制：修改数据前先写 redo log（WAL，Write-Ahead Logging），事务提交时必须将 redo log 刷盘
    - 崩溃恢复：重启时根据 redo log 重做已提交事务的修改
    - redo log 是循环写的（ib_logfile0/ib_logfile1）
  - **undo log（回滚日志）**：
    - 作用：保证 **原子性**（回滚）和 **隔离性**（MVCC 快照读）
    - 机制：记录数据修改前的镜像，事务回滚时反向操作恢复旧值
    - undo log 存储在系统表空间（ibdata）中
  - **MVCC（多版本并发控制）**：
    - 作用：保证 **隔离性**，实现可重复读/读已提交
    - 机制：每行数据有两个隐藏列（`DB_TRX_ID` 创建事务ID、`DB_ROLL_PTR` 回滚指针）
    - 读操作读取**事务开始时的快照**（read view），不阻塞写操作
    - 写操作创建新版本数据，旧版本通过 undo log 形成版本链

**可能的追问**：
- binlog 和 redo log 的区别？主从复制依赖哪个？
- 两阶段提交（2PC）是什么？redo log 和 binlog 如何协调？
- 什么是 purge 操作？undo log 何时被清理？

---

### Q4：MySQL 的隔离级别有哪些？默认隔离级别是什么？可重复读能解决幻读吗？

**考察点**：隔离级别、脏读/不可重复读/幻读、MVCC

**答案要点**：
- **四种隔离级别**（标准 SQL 定义，从低到高）：
  | 隔离级别 | 脏读 | 不可重复读 | 幻读 |
  |---------|------|-----------|------|
  | Read Uncommitted | 可能 | 可能 | 可能 |
  | Read Committed（RC）| ❌ | 可能 | 可能 |
  | Repeatable Read（RR）| ❌ | ❌ | **可能**（但 InnoDB 可解决） |
  | Serializable | ❌ | ❌ | ❌ |
- **MySQL 默认：Repeatable Read**
- **RR 解决幻读的方式**：
  - **快照读**（普通 SELECT）：MVCC 快照读，不存在幻读
  - **当前读**（SELECT ... FOR UPDATE / INSERT / UPDATE）：Next-Key Lock（记录锁 + 间隙锁），锁住索引范围内的间隙，防止其他事务插入新记录
  - 结论：**InnoDB RR 级别下，通过 Next-Key Lock 解决了幻读**

**可能的追问**：
- 什么是 Next-Key Lock？它和 Gap Lock、Record Lock 的关系？
- RC 隔离级别下，MVCC 的 read view 是在什么时候生成的？
- 什么是"快照读"和"当前读"？为什么更新操作必须用当前读？

---

### Q5：你在数据报表业务中做了 ODS → DWD → DWS 三层建模，这三层在 MySQL 中的表设计有什么区别？如何处理缓慢变化维度（SCD）？

**考察点**：数据仓库建模、星型模型、SCD

**答案要点**：
- **三层建模区别**：
  - **ODS（Operational Data Store）**：原始层，和源表结构一一对应，尽量保持原貌，加 ETL 时间戳和批次号
  - **DWD（Data Warehouse Detail）**：明细层，做数据清洗（去重、空值填充、格式统一）、统一业务过程（如所有下单事件统一字段）、拉宽表（JOIN 后打宽）
  - **DWS（Data Warehouse Summary）**：汇总层，按主题域聚合（每日活跃用户数、每分钟告警量），供报表直接查询
- **MySQL 表设计区别**：
  - ODS：分区表（按天/按批次），宽表，保留原始数据
  - DWD：分区表 + 主键（业务ID），添加维度代理键，标准化字段类型
  - DWS：高度聚合，大宽表，物化视图（MySQL 8.0+）或定时任务预计算
- **SCD（Slowly Changing Dimension）处理**：
  - **SCD Type 1**：直接覆盖旧值（不保留历史）
  - **SCD Type 2**：保留历史（新增一行，标记有效时间），告警 assignee 变更时保留变更前后两条记录
  - **SCD Type 3**：新增变更前后两个字段（只保留当前和前一个值）

**可能的追问**：
- 星型模型和雪花模型的区别？为什么星型模型在 OLAP 中更常用？
- MySQL 8.0 的窗口函数（Window Functions）如何在 DWS 层应用？
- 如何用 MySQL 实现 T+1 批次更新和实时增量更新？

---

### Q6：慢 SQL 如何排查和优化？explain 的 type 字段中 `ALL`、`index`、`range`、`ref`、`const` 分别代表什么？

**考察点**：慢查询分析、执行计划解读、索引优化

**答案要点**：
- **慢 SQL 排查流程**：
  1. 开启慢查询日志：`slow_query_log=ON`，设置 `long_query_time=1`
  2. 分析慢查询日志：`mysqldumpslow -s t -t 10 slow.log`
  3. `EXPLAIN` 分析执行计划（重点字段：type/key/rows/Extra）
  4. `EXPLAIN ANALYZE`（MySQL 8.0+）查看实际执行成本
- **type 字段含义**（从差到好）：
  - `ALL`：全表扫描，最差
  - `index`：全索引扫描（比 ALL 快，因为索引数据更紧凑）
  - `range`：索引范围扫描（`>`、`<`、`BETWEEN`），常见
  - `ref`：非唯一索引等值查询，返回匹配的多行
  - `eq_ref`：唯一索引等值查询，返回最多一行
  - `const`：最多匹配一行（主键/唯一索引等值查询），最优
- **优化方向**：
  - type=ALL → 添加合适索引
  - Extra 出现 `Using filesort`/`Using temporary` → 优化 ORDER BY / GROUP BY（加索引避免排序）
  - `Using join buffer` → 增加 join 缓冲或加索引

**可能的追问**：
- `Using index`（索引覆盖）和 `Using index condition`（索引下推 ICP）有什么区别？
- `EXPLAIN` 中 rows 字段是否准确？它和实际扫描行数的关系？
- 如何通过 `show profile` 或 `performance_schema` 分析 SQL 各阶段耗时？

---

### Q7：分库分表的场景和策略？数据量大到什么程度需要考虑分库分表？

**考察点**：分库分表策略、ShardingSphere

**答案要点**：
- **何时分库分表**：
  - 单表 > 2000 万行（MySQL B+ 树层级增加，查询变慢）
  - 单库 > 500GB（备份、恢复时间过长）
  - CPU/内存成为瓶颈（连接数、QPS 上限）
  - **先优化索引，再分库分表**（很多性能问题索引能解决）
- **分片策略**：
  - **范围分片**：`user_id % N`（取模），分布均匀，但跨分片查询难
  - **时间分片**：按月/天分片（如告警表按时间分片），适合时序数据
  - **哈希分片**：一致性哈希，减少扩缩容数据迁移
  - **自定义分片**：按业务属性（region/shard_key）
- **跨分片查询解决**：
  - 禁止跨分片 JOIN → 应用层拆分查询
  - 异构索引表（ES 索引告警 ID → 路由到正确分片）
  - 汇总表（定时任务聚合到汇总表）
- **工具**：ShardingSphere（JDBC 代理层）、MyCat（早期方案）、Vitess（YouTube 方案）

**可能的追问**：
- 分库分表后如何做 ID 生成？（雪花算法 / UUID / 分布式 ID 生成器）
- 扩容（增加分片数）时如何迁移数据？一致性哈希如何减少迁移量？
- ShardingSphere 的 `share-spring-boot-starter` 是如何在应用层透明分片的？

---

### Q8：MySQL 的 CRUD（INSERT/SELECT/UPDATE/DELETE）基础语法？`WHERE`/`GROUP BY`/`HAVING`/`ORDER BY`/`LIMIT` 的执行顺序？

**考察点**：SQL 执行顺序、查询子句

**答案要点**：
- **执行顺序**（从先到后）：
  ```
  1. FROM（包括 JOIN）
  2. WHERE
  3. GROUP BY
  4. HAVING
  5. SELECT（投影）
  6. ORDER BY
  7. LIMIT
  ```
- **关键点**：
  - `HAVING` 在 GROUP BY 后执行，可对聚合结果过滤
  - `WHERE` 在 GROUP BY 前执行，只能过滤原始行
  - `SELECT` 执行在 ORDER BY 前，所以 SELECT 的别名无法在 ORDER BY 中使用（MySQL 特殊处理）
  - `LIMIT` 最后执行，限制返回行数
- **示例**：统计 severity>1 的告警数量，按 severity 分组，告警数>10 的按数量降序：
  ```sql
  SELECT severity, COUNT(*) as cnt
  FROM alerts
  WHERE severity > 1
  GROUP BY severity
  HAVING cnt > 10
  ORDER BY cnt DESC
  LIMIT 20;
  ```

**可能的追问**：
- DISTINCT 和 GROUP BY 的区别？性能上哪个更好？
- 子查询和 JOIN 的选择：什么情况下子查询比 JOIN 更好？
- `EXISTS` 和 `IN` 在子查询中的性能差异？

---

### Q9：MySQL 的 JOIN 类型（INNER/LEFT/RIGHT/FULL/CROSS）区别？多表关联查询时如何优化？

**考察点**：JOIN 类型、JOIN 优化

**答案要点**：
- **JOIN 类型**：
  - `INNER JOIN`：两表交集，只保留匹配行
  - `LEFT JOIN`：左表全部保留，右表无匹配则 NULL
  - `RIGHT JOIN`：右表全部保留
  - `FULL OUTER JOIN`：两表全部保留（MySQL 不直接支持，用 `UNION` 实现）
  - `CROSS JOIN`：笛卡尔积（无 ON 条件）
- **JOIN 优化策略**：
  1. **确保被驱动表有索引**（EXPLAIN 中小表为驱动表，被驱动表有索引）
  2. **避免 SELECT ***，只查需要的字段
  3. **小表驱动大表**：`INNER JOIN` 时 MySQL 自动选小表为驱动表
  4. **避免在 JOIN 条件中使用函数**（导致索引失效）
  5. **用 STRAIGHT_JOIN` 强制指定驱动表**（当 MySQL 选错时）
  6. **控制 JOIN 数量**：JOIN 超过 3 张表时考虑拆解或反范式化

**可能的追问**：
- `INNER JOIN` 和 `EXISTS` 哪个更快？什么场景下 EXISTS 更优？
- `STRAIGHT_JOIN` 的使用场景是什么？
- 什么是 NL（Nested Loop）、BNL（Block Nested Loop）、Hash Join？MySQL 8.0+ 的 Hash Join 比 NL 快在哪里？

---

### Q10：MySQL 的索引分类（主键/唯一/普通/全文）和适用场景？联合索引的最左前缀原则是什么？

**考察点**：索引分类、最左前缀、索引设计

**答案要点**：
- **索引分类**：
  - **主键索引（PRIMARY）**：唯一且非空，一张表只能有一个，聚簇索引（数据按主键排序存储）
  - **唯一索引（UNIQUE）**：值唯一，可为空，非聚簇
  - **普通索引（INDEX）**：无唯一性约束，用于加速查询
  - **全文索引（FULLTEXT）**：用于文本全文搜索（`MATCH() AGAINST()`），替代 LIKE 模糊查询
- **最左前缀原则**：
  - 联合索引 `(A, B, C)` 等价于创建了 `(A)`、`(A, B)`、`(A, B, C)` 三个索引
  - 查询必须从最左边开始且**连续**，才能使用索引
  - `WHERE A = 1 AND B > 2 AND C = 3`：A 用索引，B 是范围，索引在 B 处断开，C 不用索引
  - `WHERE B = 2 AND C = 3`：无法使用最左前缀，索引失效
- **索引设计原则**：
  - 选择性高的列放前面（区分度大的字段优先）
  - 等值查询优先于范围查询
  - 避免在索引列上使用函数

**可能的追问**：
- 什么是索引覆盖（覆盖索引）？为什么能减少回表？
- 什么是索引条件下推（Index Condition Pushdown，ICP）？
- 联合索引的选择性如何计算？

---

### Q11：MySQL 的 `EXPLAIN` 分析：type 字段各值的含义？如何根据 explain 结果优化慢查询？

**考察点**：执行计划深度解读、索引优化实战

**答案要点**：
- **type 字段值（从差到好）**：
  - `system` > `const` > `eq_ref` > `ref` > `range` > `index` > `ALL`
  - `system`：表只有一行（系统表），极少出现
  - `const`：主键/唯一索引等值查询，最多一行
  - `eq_ref`：连接时，被驱动表使用唯一索引
  - `ref`：非唯一索引等值扫描
  - `range`：索引范围扫描（BETWEEN、IN、>、<）
  - `index`：全索引扫描（比全表扫描快，因为按索引顺序读）
  - `ALL`：全表扫描，最差
- **其他关键字段**：
  - `key`：实际使用的索引
  - `rows`：预计扫描行数（越大越差）
  - `Extra`：
    - `Using index`：覆盖索引，无需回表
    - `Using index condition`：索引下推
    - `Using where`：存储引擎返回后需服务端过滤
    - `Using filesort`：需要额外排序
    - `Using temporary`：需要临时表

**可能的追问**：
- `Using index condition`（ICP）和 `Using index` 的区别是什么？
- `rows` 是估算值还是精确值？如何让它更准确？
- `Extra` 中的 `Using filesort` 具体是什么排序算法？（快速排序 + 文件排序）

---

### Q12：MySQL 的视图（VIEW）是什么？视图的优点（简化查询/数据安全）和限制（不可更新视图）？

**考察点**：视图类型、可更新条件

**答案要点**：
- **视图本质**：虚拟表，存储 SQL 查询语句而非数据，每次查询视图时动态执行查询语句
- **类型**：
  - **普通视图**：`CREATE VIEW v AS SELECT ...`
  - **物化视图**（MySQL 8.0+）：存储查询结果，类似于"快照"，需要定时刷新
- **优点**：
  1. **简化复杂查询**：将多表 JOIN 封装为视图，前端只需查视图
  2. **数据安全**：不同权限用户看不同视图，隐藏敏感字段
  3. **逻辑独立**：底层表结构变更时，只需修改视图定义，引用视图的应用无需改动
- **不可更新视图**：当视图包含聚合函数、DISTINCT、GROUP BY、UNION、子查询等时，视图不可更新

**可能的追问**：
- 物化视图和普通视图的根本区别是什么？物化视图如何刷新？
- 视图会存储在内存中吗？视图的性能和直接写 SQL 比如何？
- 如何在 MySQL 中实现列级权限（不同用户看到不同字段）？

---

### Q13：MySQL 的存储过程和函数有什么区别？在告警统计报表中有没有用过存储过程？

**考察点**：存储过程 vs 函数、优缺点

**答案要点**：
- **存储过程 vs 函数**：
  | 特性 | 存储过程 | 函数 |
  |------|---------|------|
  | 调用方式 | `CALL proc_name()` | `SELECT func_name()` |
  | 返回值 | 可返回多个结果集或无 | 必须有返回值（RETURN） |
  | SQL 语句 | 可含 DDL/DML/TRUNCATE | 只能含 SELECT（查询语句） |
  | 事务控制 | 可含事务控制语句 | 不允许事务控制 |
- **存储过程优缺点**：
  - ✅ 优点：减少网络传输（业务逻辑在 DB 端）、预编译缓存性能好
  - ❌ 缺点：业务逻辑分散（DB 绑定）、版本管理难、单元测试复杂、移植性差
- **实际场景**：SOC 平台中**尽量不用存储过程**，业务逻辑放在 Python/Java 应用层，数据库只做存储和查询。报表数据在 Python 中处理后写入汇总表

**可能的追问**：
- 存储过程和 prepared statement 有什么区别？
- 为什么不推荐在存储过程中写业务逻辑？
- 存储过程中如何捕获和处理异常？（DECLARE HANDLER）

---

### Q14：MySQL 的主从复制原理？什么是半同步复制？主从延迟的常见原因和解决方案？

**考察点**：主从复制机制、半同步、延迟处理

**答案要点**：
- **主从复制原理**：
  1. **Binlog dump**：主库 dump 线程将 binlog 发送给从库
  2. **I/O Thread**：从库 I/O 线程接收 binlog，写入 relay log（中继日志）
  3. **SQL Thread**：从库 SQL 线程读取 relay log，在从库重放（replay）
  4. 复制方式：`异步复制`（默认） / `半同步复制` / `全同步复制`
- **半同步复制**：
  - 主库事务提交后，至少一个从库 ACK 确认收到并写入 relay log 后，主库才向客户端返回成功
  - 保证至少一个从库有数据，降低数据丢失风险
- **主从延迟原因**：
  1. **从库慢**：从库机器性能差或配置低
  2. **大事务**：主库一个事务执行时间长，binlog 传输到从库执行也长
  3. **从库查询压力大**：从库承担了大量读请求，影响了 relay log 重放
  4. **大事务拆分**：分批提交，避免单事务过大
- **解决方案**：从库开启 `read_only`、读写分离（延迟读）、并行复制（GTID + 并行复制）

**可能的追问**：
- GTID（Global Transaction Identifier）是什么？为什么 GTID 比文件+位点方式更可靠？
- 什么是并行复制（Parallel Replication）？slave_parallel_type 和 logical_clock 是什么？
- 如何监控主从延迟？（`show slave status` 的 `Seconds_Behind_Master`）

---

## 5.2 Redis

### Q15：Redis 是什么？为什么 Redis 能做到微秒级延迟？Redis 和 Memcached 的核心区别是什么？

**考察点**：Redis 内存存储优势、vs Memcached

**答案要点**：
- **为什么快**：
  1. **纯内存操作**：所有数据在内存，O(1) / O(log N) 时间复杂度
  2. **单线程 + IO 多路复用**：单线程避免锁开销，epoll（Linux）/ kqueue（macOS）实现高效事件循环
  3. **C 语言 + 优化数据结构**：SDS（简单动态字符串）、ziplist、quicklist 等高效数据结构
  4. **非阻塞 I/O**：基于 reactor 模型
- **Redis vs Memcached**：
  | 维度 | Redis | Memcached |
  |------|-------|-----------|
  | 数据结构 | 丰富（string/list/set/zset/hash/stream） | 单一（key-value） |
  | 持久化 | 支持（RDB/AOF） | 不支持（纯内存） |
  | 主从复制 | ✅（支持） | ❌（不支持） |
  | 集群 | Cluster（原生支持） | 分布式（客户端分片） |
  | 事务 | 支持（Lua 脚本） | 不支持 |
  | 内存管理 | 自定义 SDS，内存碎片管理 | Slab Allocator |
  | 适用 | 复杂数据结构、需要持久化 | 简单 KV、高性能 |

**可能的追问**：
- Redis 6.0+ 多线程是什么多线程？和单线程模型有什么关系？
- Redis 的 SDS 字符串比 C 字符串好在哪些方面？
- 什么是 Redis 的惰性删除（lazy free）？它和主动删除的区别？

---

### Q16：Redis 支持哪些数据类型？分别适合什么场景？你在项目中用到了哪些？

**考察点**：Redis 数据类型、场景匹配

**答案要点**：
- **五大数据类型 + 两种特殊**：
  | 类型 | 底层实现 | 适用场景 |
  |------|---------|---------|
  | **String** | SDS | 缓存、计数器（INCR）、Session、分布式锁 |
  | **List** | quicklist（ziplist + linkedlist） | 消息队列（BRPOP）、任务队列、最新列表 |
  | **Hash** | ziplist / hashtable | 对象缓存（如用户信息）、聚合统计 |
  | **Set** | intset / hashtable | 去重（访问日志去重）、标签（用户标签交集） |
  | **Sorted Set** | ziplist / skiplist + hashtable | 排行榜（TOP N）、延时队列、有序事件 |
  | **Bitmap** | string 的位操作 | 签到、去重统计、用户在线状态 |
  | **HyperLogLog** | string 编码 | UV 统计（去重计数） |
  | **Geospatial** | ziplist / skiplist | 附近的人、LBS |
  | **Stream** | RadixTree + listpack | 消息队列、事件流 |
- **SOC 平台中的使用**：
  - **String**：`alert:cache:{id}` 缓存告警详情
  - **Hash**：用户会话 `session:{token}` 存储用户信息
  - **Sorted Set**：告警实时排行榜（按 severity 评分）
  - **Stream**：实时告警事件队列（替代 Kafka 的轻量场景）
  - **Set**：`user:{id}:roles` 存储用户角色集合

**可能的追问**：
- ZSet 的 score 相同情况下如何排序？（按 member 字典序）
- 为什么 ZSet 用跳表而不是红黑树？
- Bitmap 在签到场景中如何存储？如何计算连续签到天数？

---

### Q17：Kafka 和 RabbitMQ、Redis Stream 相比有什么优劣？你们为什么在 SOC 平台中选择 Kafka 做告警分发？

**考察点**：消息队列选型、吞吐、持久化

**答案要点**：
- **对比**：
  | 维度 | Kafka | RabbitMQ | Redis Stream |
  |------|-------|---------|------------|
  | 吞吐量 | 百万级/秒 | 万级/秒 | 十万级/秒 |
  | 持久化 | 强（磁盘持久化 + 副本） | 内存+磁盘 | 内存+ RDB/AOF |
  | 消息语义 | At-least-once（可实现 Exactly-once） | At-least-once | At-least-once |
  | 延迟 | 毫秒级 | 微秒级（低负载） | 微秒级 |
  | 消息保留 | 按时间/大小保留，可重消费 | 消费即删除 | 可配置，TTL 后删除 |
  | 功能 | 简单（Topic/Partition） | 丰富（Exchange/路由/死信） | 中等（Consumer Group） |
  | 生态 | 巨大（Spark/Flink/ES） | 中等 | 依赖 Redis 生态 |
- **SOC 平台选 Kafka 原因**：
  1. **告警量大**：高峰期告警每秒数千条，Kafka 的高吞吐是必须的
  2. **消息重消费**：告警分发后可能需要重放（重新分析），Kafka 消息保留时间长
  3. **生态集成**：Kafka Connect 对接 ES/MySQL、Kafka Streams 做实时统计
  4. **顺序消费**：告警按时间顺序处理，Partition 内有序

**可能的追问**：
- Kafka 的吞吐量为什么能这么高？（顺序写 + Zero Copy + 批量压缩）
- RabbitMQ 的 Exchange 路由和 Kafka 的 Topic 消息模型有什么本质不同？
- Redis Stream 在什么场景下比 Kafka 更合适？

---

### Q18：Redis Stream 的消费者组（Consumer Group）机制是什么？`xreadgroup` 的 `>` 和 `0-0` 有什么区别？noack 参数的作用？

**考察点**：Stream 消费者组、PEL、消息确认

**答案要点**：
- **消费者组机制**：
  - `XGROUP CREATE mystream groupname $ MKSTREAM`：创建消费者组
  - 一个 Stream 可以有多个 Consumer Group，每个 Group 独立消费
  - Group 内多个 Consumer 竞争消费同一消息（一条消息只被一个 Consumer 获取）
  - 消息被消费后进入 **PEL（Pending Entries List）**，等待 ACK
- **`>` vs `0-0`**：
  - `XREADGROUP GROUP groupname consumername STREAM mystream ">"`：只读**新消息**（未分配给任何 Consumer 的消息）
  - `XREADGROUP GROUP groupname consumername STREAM mystream "0-0"`：读取 **PEL 中的待确认消息**（之前消费过但未 ACK 的，用于故障恢复时重新消费）
- **`noack`**：`xreadgroup` 的 `NOGACK` 选项，不将消息加入 PEL，消费后自动 ACK（适用于不关心丢失的场景）
- **ACK 示例**：`XACK mystream groupname message-id`

**可能的追问**：
- 如果 Consumer 崩溃了，PEL 中的消息会怎样？（其他 Consumer 可以认领 PEL 中长时间未 ACK 的消息）
- Redis Stream 的 `XREAD` 和 `XREADGROUP` 有什么区别？
- Redis Stream 和 Kafka Consumer Group 语义上有什么不同？

---

### Q19：Redis 的持久化方式 RDB 和 AOF 的区别？混合持久化是什么？

**考察点**：持久化机制、RDB vs AOF、混合模式

**答案要点**：
- **RDB（Redis Database）**：
  - 触发：定时快照（`save` 命令或 `bgsave` 后台进程）
  - 方式：fork 子进程，将内存数据全量写入 RDB 文件
  - 优点：恢复快（直接加载 RDB 文件）、fork 时不影响主进程
  - 缺点：两次快照之间数据丢失、fork 全量拷贝开销大
- **AOF（Append Only File）**：
  - 触发：每个写操作追加到 AOF 文件
  - 模式：`always`（每条同步）/ `everysec`（每秒同步）/ `no`（OS 决定）
  - 优点：数据安全性更高、只追加不修改
  - 缺点：AOF 文件大于 RDB、重写需要时间
- **混合持久化**（Redis 4.0+）：
  - RDB 格式的内存快照 + AOF 重写后增量部分
  - 恢复时先加载 RDB，再重放 AOF 增量
  - 兼顾恢复速度和数据完整性
- **建议**：开启混合持久化（`aof-use-rdb-preamble yes`）

**可能的追问**：
- AOF 重写（rewrite）是什么？为什么需要重写？如何配置自动重写？
- `BGSAVE` fork 子进程时，父进程修改数据如何处理？（copy-on-write 机制）
- Redis 6.0 的 AOF 重写多线程（aof-use-rdb-preamble）是怎样的？

---

### Q20：Redis 的过期删除策略和内存淘汰策略分别是什么？在告警缓存场景下你选择哪种淘汰策略？

**考察点**：过期策略、LRU/LFU、内存管理

**答案要点**：
- **过期删除策略**（何时删除过期的 key）：
  1. **惰性删除**：访问 key 时检查是否过期，过期则删除。节省 CPU，但过期 key 可能长时间占用内存
  2. **定期删除**：每隔一段时间随机检查有过期 key 的 DB，删除过期的（`hz` 参数控制频率）
  3. **定时删除**：为每个 key 创建定时器，过期立即删除。精确但耗 CPU
  - Redis 采用：**惰性删除 + 定期删除** 组合
- **内存淘汰策略**（内存达上限时）：
  - `noeviction`：不淘汰，返回错误（默认）
  - `volatile-lru/allkeys-lru`：LRU 算法淘汰（最近最少使用）
  - `volatile-lfu/allkeys-lfu`：LFU 算法淘汰（最不经常使用）
  - `volatile-random/allkeys-random`：随机淘汰
  - `volatile-ttl`：淘汰 TTL 最短的 key
- **告警缓存场景选择**：
  - `allkeys-lru`：告警缓存数据量大、访问频繁，LRU 最合适
  - `volatile-lru`：热点告警需要保留，TTL 短的告警优先淘汰
  - 实际配置：`maxmemory 2gb`，`maxmemory-policy allkeys-lru`

**可能的追问**：
- Redis 的 LRU 算法是精确 LRU 吗？采样多少个 key？
- LFU 相比 LRU 在哪些场景更优？
- Redis 内存满了但没达到 maxmemory-policy 的淘汰条件，会发生什么？

---

### Q21：缓存穿透、缓存击穿、缓存雪崩分别是什么？如何解决？

**考察点**：缓存三大问题、布隆过滤器、分布式锁

**答案要点**：
- **缓存穿透**（查询不存在的数据）：
  - 原因：恶意查询不存在的主键（如ID=-1），直接打到 DB
  - 解决：
    1. **布隆过滤器**（BloomFilter）：在写入缓存时同步写入 BloomFilter，查询前先检查是否存在
    2. **空值缓存**：查询不存在的结果也缓存（设置短 TTL，如 60s）
    3. **参数校验**：ID 必须是正整数
- **缓存击穿**（热点 key 过期，大量请求击穿到 DB）：
  - 原因：热点 key 过期瞬间，大量请求同时访问
  - 解决：
    1. **分布式锁**（SETNX）：只允许一个请求查 DB，其他等待
    2. **永不过期**：物理不过期，逻辑过期（字段中存过期时间，检查时异步更新）
- **缓存雪崩**（大量 key 同时过期）：
  - 原因：大量 key 集中过期（如故障重启后缓存全空）
  - 解决：
    1. **过期时间加随机值**：`TTL + random(0, 5min)`
    2. **多级缓存**：本地缓存（Caffeine）+ Redis，避免全部失效
    3. **Redis 高可用**：哨兵/Cluster，防止 Redis 全挂了

**可能的追问**：
- 布隆过滤器的误判率如何计算？FP 和 FN 哪个更不能容忍？
- 什么是缓存击穿的"热点 key 发现与永不过期"自动更新机制？
- 本地缓存（如 Guava Cache/Caffeine）和 Redis 的一致性如何保证？

---

### Q22：Redis 的 BigKey 问题是什么？如何发现和解决？`SCAN` 命令和 `KEYS` 命令的区别？

**考察点**：BigKey 危害、扫描命令、lazy free

**答案要点**：
- **BigKey 问题**：
  - 定义：单个 key 的 value 过大（如 String 超 10MB、Collection 含数十万元素）
  - 危害：内存不均（big key 所在节点内存暴涨）、操作超时（SMEMBERS 阻塞）、主从同步慢
- **发现方法**：
  1. `redis-cli --bigkeys`：扫描实例，输出每种类型的最大 key
  2. `MEMORY USAGE key`：查看特定 key 的内存占用
  3. `SCAN` + `TYPE` 遍历，检查每个 key 的大小
  4. 监控 `cmdstat_*` 中高耗时命令（Big Key 操作延迟高）
- **解决方式**：
  1. **拆键**：将大 string 拆为多个小 string（如 `key:part1`, `key:part2`）
  2. **压缩**：value 压缩（GZIP/snappy）
  3. **合理数据结构**：如 List 改用分段缓存
  4. **lazy free**：`UNLINK`（非阻塞删除）代替 `DEL`（阻塞删除）
- **`SCAN` vs `KEYS`**：
  - `KEYS pattern`：一次性返回所有匹配的 key，**阻塞主线程**，大数据量时会造成 Redis 卡顿
  - `SCAN cursor [MATCH pattern] [COUNT count]`：渐进式游标遍历，**非阻塞**，每次返回少量 key，多次调用完成全量扫描

**可能的追问**：
- `UNLINK` 的实现原理是什么？删除不是异步的吗？
- Redis Cluster 中 Big Key 对集群的影响有什么特殊之处？
- 如何设置 Redis 内存警戒线？（`maxmemory-policy` + `maxmemory-samples`）

---

### Q23：Redis 的 `MULTI`/`EXEC`/`WATCH` 事务和关系型数据库的事务有什么区别？

**考察点**：Redis 事务、原子性保障

**答案要点**：
- **Redis 事务**：
  - `MULTI` → 开启事务队列，命令入队而非立即执行
  - `EXEC` → 一次性执行所有队列中的命令
  - `WATCH key` → 乐观锁，key 在 WATCH 之后被其他客户端修改则 EXEC 返回空
- **与 RDBMS 事务的区别**：
  | 特性 | Redis 事务 | RDBMS 事务 |
  |------|-----------|-----------|
  | 原子性 | 批量执行（但命令间无回滚） | ACID 原子性（任意命令失败可回滚） |
  | 回滚 | ❌ 不支持（部分命令失败已执行的不回滚） | ✅ 支持 |
  | 隔离性 | 无隔离（EXEC 前命令不入实际数据） | MVCC 隔离 |
  | 锁 | WATCH 乐观锁 | 悲观锁（SELECT FOR UPDATE） |
- **本质区别**：Redis 事务是**批量执行**（batch），不是 ACID 事务。真正的原子性需要用 **Lua 脚本** 保证（Redis 执行 Lua 脚本是原子性的）
- **实际建议**：Redis 事务主要用于**批量操作优化网络往返**，需要原子性的场景用 Lua 脚本

**可能的追问**：
- Redis Lua 脚本为什么是原子的？（执行 Lua 脚本时不会执行其他命令）
- WATCH 命令如果被多次调用会怎样？UNWATCH 的作用？
- Redis 的 pipeline 和事务有什么区别？

---

### Q24：Redis 的 Pub/Sub 和 Stream 有什么区别？实时告警推送用哪种更合适？

**考察点**：Pub/Sub 语义、Stream 特性

**答案要点**：
- **Pub/Sub**：
  - 发送即推送，**不存储消息**
  - 消费者掉线重连后，**不保留历史消息**
  - 没有 ACK 机制，消息可靠性无保证
  - 适合：低可靠性要求的广播通知
- **Stream**：
  - **持久化**：消息存储在 Stream 中，可反复消费
  - **消费者组**：支持多个 Consumer Group 独立消费
  - **ACK 确认**：消息确认后才从 PEL 中移除
  - **支持范围查询**：可读取历史消息
  - 适合：高可靠性消息传递
- **SOC 实时告警推送选择**：
  - **用 Stream**：告警推送需要**不丢消息**（告警确认后需回溯）、支持**多消费者组**（告警推送 / 告警分析 / 告警通知 各一个 Group）
  - 但实际 SOC 平台告警分发使用 **Kafka**，因为告警量大（百万级/天）、需要与其他系统（ES/MySQL）对接、Stream 无法满足多 Consumer Group 的高吞吐场景

**可能的追问**：
- Pub/Sub 如何保证消息的最终一致性？
- Redis Stream 的 `XINFO GROUPS` 可以查看哪些信息？
- 在 Redis Cluster 模式下，Pub/Sub 有什么限制？

---

### Q25：常见的缓存读写模式（Cache-Aside / Write-Through / Write-Behind）各自的工作流程是什么？你们 SOC 平台的告警缓存用的是什么模式？如何解决缓存和数据库的双写一致性问题？

**考察点**：缓存读写模式、数据一致性策略

**答案要点**：
- **三种模式**：
  1. **Cache-Aside（旁路缓存）**：
     - 读：Cache Miss → 查 DB → 写 Cache
     - 写：更新 DB → 删除 Cache（而非更新 Cache，因为更新 Cache 可能导致脏数据）
  2. **Write-Through（写穿透）**：
     - 写：同时写 Cache 和 DB，Cache 作为 DB 的前置写缓存
  3. **Write-Behind（写回）**：
     - 写：只写 Cache，异步批量写回 DB
     - 风险最高（Cache 丢了数据就丢了）
- **SOC 平台告警缓存模式**：
  - 使用 **Cache-Aside**：`alert:cache:{id}` → 读 Miss 后回源，读服务优先查缓存
  - 告警写入时**不主动写缓存**（告警写入后立即可见，缓存作为读放大优化）
  - 告警变更时**删除缓存**而非更新缓存（避免并发导致数据不一致）
- **双写一致性**：
  - **延时双删**：更新 DB 后删除 Cache，sleep 后再删一次（处理并发读写）
  - **订阅 binlog**：使用 Canal/Debezium 监听 DB 变更，异步更新 Cache
  - **最终一致性 > 强一致性**：缓存的主要目的是加速读取，短时间不一致是可接受的

**可能的追问**：
- 为什么写策略是"删除 Cache"而非"更新 Cache"？
- 如何处理 Cache 和 DB 的双写一致性时的并发问题？
- 什么是"cache stampede"（缓存击穿）？如何用 mutex 或 singleflight 解决？

---

### Q26：多级缓存（本地缓存如 Caffeine/Guava Cache + Redis）的一致性如何保证？缓存预热和缓存失效风暴（Thundering Herd）分别怎么应对？

**考察点**：多级缓存架构、缓存预热、防击穿

**答案要点**：
- **多级缓存架构**：
  ```
  请求 → 本地缓存（Caffeine 1min）→ Redis 缓存（5min）→ MySQL
  ```
- **一致性保证**：
  - **主动失效**：DB 变更时，同时失效 Redis 和本地缓存（广播通知各实例清除本地缓存）
  - **广播机制**：Redis Pub/Sub 或消息队列通知所有实例清除本地缓存
  - **版本号**：本地缓存 Key 加版本号，失效时版本号+1
  - **TTL 兜底**：本地缓存 TTL 短（1min），即使不一致最多持续 1min
- **缓存预热**：
  1. **系统启动时**：主动查询热点数据写入缓存
  2. **定时任务**：每天业务低峰期预热次日热点数据
  3. **懒加载 + 预热**：用户访问时记录热度，热度达到阈值后提前缓存
- **Thundering Herd（缓存失效风暴）**：
  - 原因：大量缓存同时过期，所有请求击穿到 DB
  - 解决：
    1. **TTL 随机化**：各 key 的过期时间加随机偏移（±30%）
    2. **singleflight 模式**：多个并发请求只放一个到 DB，其他等待（Go 的 singleflight，Python 可用 asyncio lock）
    3. **分布式锁**：只允许一个请求回源写缓存

**可能的追问**：
- Caffeine 的 `W-TinyLFU` 淘汰算法原理是什么？
- Redis Cluster 模式下，如何实现缓存失效广播？
- 什么是"热点 key 探测与缓存"？Redis 如何发现热点 key？

---

### Q27：Redis Lua 脚本为什么能保证原子性？在分布式锁的加锁/解锁/续期（看门狗）场景中，Lua 脚本解决了什么问题？给出用 Lua 实现 SETNX + EXPIRE 原子化的脚本。

**考察点**：Lua 脚本原子性、分布式锁实现

**答案要点**：
- **Lua 脚本原子性原理**：
  - Redis 执行 Lua 脚本时，**单线程执行整个脚本**，不会插入其他命令
  - 等价于把多个 Redis 命令打包成一个原子操作
  - Lua 脚本执行期间，Redis 不会执行其他客户端命令
- **分布式锁场景解决的问题**：
  - `SETNX` + `EXPIRE` 两条命令**不是原子**的，SETNX 成功但 EXPIRE 前崩溃会导致死锁
  - 解锁时，检查 value 是否是自己加的锁（防止误删别人的锁），检查和删除必须是原子的
  - 续期（看门狗）需要在锁快过期时自动延长 TTL
- **SETNX + EXPIRE 原子化 Lua 脚本**：
  ```lua
  -- KEYS[1] = lock key, ARGV[1] = expire time (ms), ARGV[2] = unique value (UUID)
  if redis.call('SET', KEYS[1], ARGV[2], 'NX', 'PX', ARGV[1]) then
      return 1
  else
      return 0
  end
  ```
- **分布式锁完整实现**（SET + NX + PX + 校验 value）：
  ```lua
  -- 加锁
  if redis.call('SET', KEYS[1], ARGV[1], 'NX', 'PX', ARGV[2]) == 'OK' then return 1 else return 0 end
  -- 解锁（Lua 保证检查+删除原子）
  if redis.call('GET', KEYS[1]) == ARGV[1] then
      return redis.call('DEL', KEYS[1])
  else
      return 0
  end
  ```

**可能的追问**：
- Redisson 的看门狗（Watchdog）机制是如何实现的？
- Redis 分布式锁的"可重入"特性如何实现？
- 为什么不能用 `redis.call('DEL', KEYS[1])` 直接解锁？（可能删除别人刚加的锁）

---

### Q28：你用 Redis 做过哪些原子操作？`INCR`/`DECR` 在告警计数器场景中的用途是什么？如果需要原子地"先减到0则触发动作"，Lua 脚本如何实现？

**考察点**：Redis 原子操作、计数器、原子递减

**答案要点**：
- **常用原子操作**：
  - `INCR`/`INCRBY`/`DECR`/`DECRBY`：计数器（告警计数、会话计数）
  - `SETNX`：分布式锁、唯一性判断
  - `HSET`/`HINCRBY`：字段增量（如 `alert:stat:{date}` Hash 中按 severity 计数）
  - `SET` + `GET`：组合用 Lua 脚本保证原子
  - `LPUSH`/`RPOP`：消息队列操作
- **告警计数器场景**：
  - `alert:count:today`：`INCR` 每日告警总数
  - `alert:severity:{level}:count`：`HINCRBY alert:severity:20260416 'high' 1` 按严重级别计数
  - 实时告警量监控：`INCR alert:rate:minute:{minute}` + 定时 TTL 清理
- **Lua 实现"减到 0 触发动作"**：
  ```lua
  -- KEYS[1] = key, ARGV[1] = 减少值, ARGV[2] = 阈值（0）, ARGV[3] = 动作
  local current = tonumber(redis.call('GET', KEYS[1]) or 0)
  if current == 0 then
      return 0  -- 已经为0，不操作
  end
  local new_val = current - tonumber(ARGV[1])
  if new_val <= 0 then
      redis.call('SET', KEYS[1], 0)
      -- 触发动作（发通知、发消息）
      redis.call('LPUSH', KEYS[2], ARGV[3])
      return 1  -- 触发成功
  else
      redis.call('DECRBY', KEYS[1], ARGV[1])
      return 0  -- 未触发
  end
  ```

**可能的追问**：
- `INCR` 操作是线程安全的吗？Redis 单线程天然保证了原子性吗？
- Redis 计数器如何实现滑动窗口限流？（Sorted Set 实现）
- `GETSET` 命令在 Redis 新版本中还存在吗？

---

### Q29：Redis 的大 Key 拆解有哪些策略？`UNLINK` 和 `DEL` 的区别是什么？在生产环境中如何安全地删除一个包含百万条数据的 Key？

**考察点**：Big Key 处理、UNLINK、安全删除

**答案要点**：
- **大 Key 拆解策略**：
  1. **按时间/ID 分段**：如 `alert:stat:2024:01`、`alert:stat:2024:02`，按月分表的思想
  2. **Hash 拆为多个 field**：不存大 Hash 存多个小 Hash（`key:{id}` 或 `key:{shard}:{id}`）
  3. **List 改用 Stream**：Stream 支持范围查询，内存效率更高
  4. **压缩 value**：大 string 用 GZIP 压缩
- **`UNLINK` vs `DEL`**：
  - `DEL`：同步删除，**阻塞主线程**，大 Key 删除时 Redis 卡顿
  - `UNLINK`：异步删除，将 key 从 keyspace 中立即移除，内存回收交给后台线程（lazy free 机制）。**非阻塞**，推荐使用
  - UNLINK 适合大 Key，DEL 适合小 Key
- **安全删除百万级数据**：
  ```bash
  # 方案1：SCAN + UNLINK（渐进式，非阻塞）
  redis-cli --scan --pattern "alert:cache:*" | while read key; do
      redis-cli UNLINK "$key"
  done

  # 方案2：Laravel/Redis 客户端渐进删除
  # 分批删除，每次1000条，不阻塞主线程
  ```

**可能的追问**：
- UNLINK 在 Redis 哪个版本引入？之前的版本如何实现非阻塞删除？
- Redis Cluster 中删除大 Key 有哪些额外注意事项？
- 如何监控 Redis 的删除操作是否造成延迟？（`commandstats`）

---

### Q30：Redis 集群（Cluster 模式）和哨兵（Sentinel）模式的区别是什么？你们 SOC 平台的 Redis 部署架构是怎样的？Cluster 模式下 Pipeline 和 MGET/MSET 的行为有什么需要注意的？

**考察点**：Redis 集群架构、Cluster vs Sentinel、跨 slot

**答案要点**：
- **Sentinel vs Cluster**：
  | 维度 | Sentinel | Cluster |
  |------|---------|--------|
  | 数据分片 | 无（主从复制，一主多从） | 16384 个 slot，自动分片 |
  | 扩容 | 不支持自动分片，需手动迁移 | 自动 slot 重分配 |
  | 故障转移 | Sentinel 监控+选主 | 节点间自动选主 |
  | 客户端路由 | 任意节点，支持重定向 | Smart Client，需知道 slot 分布 |
  | 适用 | 读多写少、高可用 | 高并发、大数据量 |
- **SOC 平台 Redis 架构**：
  - **生产**：Redis Cluster 3 主 3 从，Slot 分布在 3 台机器上
  - **Sentinel 监控**：3 个 Sentinel 监控 6 个节点（3 主 + 3 从）
  - 读操作：优先读从节点（`READONLY` + `replicaof`）
  - 写操作：路由到主节点
- **Cluster 下 Pipeline/MGET/MSET 注意事项**：
  - **MGET/MSET 跨 slot 会报错**：`CROSSSLOT` 错误，所有 key 必须在同一 slot
  - 解决：用 `{}` 哈希 tag 强制相同 slot：`{tag}:a` 和 `{tag}:b` 在同一 slot
  - **Pipeline** 在 Cluster 下会将命令拆分，但只在 keys 同属一个 slot 时才能批量

**可能的追问**：
- Redis Cluster 的 16384 个 slot 是如何分配的？节点扩缩容时 slot 如何迁移？
- 什么是 Redis Cluster 的 MOVED 重定向？客户端如何处理？
- Redis Cluster 中，从节点可以接受写操作吗？（`READONLY` 后可读，但不建议写）

---

## 5.3 Elasticsearch

### Q31：Elasticsearch 是什么？它和 MySQL 的对应关系是什么？你们在 SOC 平台中为什么用 ES 而不是直接用 MySQL 做日志检索？

**考察点**：ES 定位、搜索引擎 vs RDBMS

**答案要点**：
- **ES vs MySQL 对应关系**：
  | MySQL | Elasticsearch |
  |-------|--------------|
  | Database | Index（索引） |
  | Table | Type（7.x 后已移除） |
  | Row | Document |
  | Column | Field |
  | Schema（DDL） | Mapping |
  | Index | 倒排索引（全文检索） |
  | JOIN | 嵌套文档 / Parent-Child |
- **为什么用 ES 而非 MySQL**：
  1. **全文检索**：MySQL LIKE 查询无法满足分词、模糊匹配、相关性排序需求；ES 内置多种分词器（ik_max_word/ik_smart）
  2. **海量数据**：ES 分布式架构可水平扩展，单集群支持 PB 级数据；MySQL 分库分表运维成本高
  3. **查询性能**：倒排索引让全字段检索在毫秒级完成；MySQL 全表扫描随数据量线性变慢
  4. **聚合分析**：ES 聚合（Bucket/Metric）支持实时统计分析，MySQL GROUP BY 大数据量性能差

**可能的追问**：
- Elasticsearch 和 Solr 有什么核心区别？
- ES 的 Lucene 索引结构是什么？（段、倒排索引、DocValues）
- 什么是 Elasticsearch 的 Mapping？动态 Mapping 和显式 Mapping 的区别？

---

### Q32：ES 的倒排索引原理是什么？和正排索引的区别？

**考察点**：倒排索引原理、posting list、Term Dictionary

**答案要点**：
- **倒排索引结构**：
  - **正排索引（Forward Index）**：Document → Terms（文档包含哪些词）
  - **倒排索引（Inverted Index）**：Term → Documents（哪些文档包含这个词）
- **ES 倒排索引组件**：
  1. **Term Dictionary（词项字典）**：所有文档中的 term 集合，按字典序排列，支持二分查找
  2. **Term Index（词项索引）**：FST（Finite State Transducer，有限状态传感器）压缩存储，定位 Term Dictionary 位置
  3. **Posting List（倒排列表）**：每个 term 关联的文档 ID 列表 + 词频/位置信息
  4. **Doc Values**：列式存储，用于排序和聚合
- **查询流程**：
  1. 词项经过分词器（Analyzer）处理
  2. FST 定位 Term Dictionary 中的 term
  3. 读取 Posting List
  4. 对多个 term 的 Posting List 做 AND/OR 合并
  5. 根据相关性算法（BM25）排序返回
- **为什么适合搜索**：根据关键词直接找到包含它的文档，无需扫描全表

**可能的追问**：
- FST（Finite State Transducer）的原理是什么？为什么能压缩存储？
- 什么是 Posting List 的跳表（Skip List）合并优化？
- ES 的分词器（Analyzer）由哪三部分组成？（Tokenizer + TokenFilter + CharFilter）

---

### Q33：ES 的分片（Shard）和副本（Replica）机制是什么？如何设计合理的分片数量？

**考察点**：ES 分布式架构、分片设计

**答案要点**：
- **分片（Shard）**：
  - 每个 Index 分为多个 Shard，分布在不同节点
  - 主分片（Primary Shard）数在 Index 创建时确定（默认 5），**不可修改**
  - 每个主分片可配置 0-N 个副本分片（Replica）
  - 数据写入：客户端 → 协调节点 → 主分片（写入）→ 同步到副本 → 返回成功
- **副本机制**：
  - 副本分片提供数据冗余，提高可用性
  - 副本分片可处理读请求（分担查询压力）
  - 主分片故障时，ES 自动将副本晋升为主分片
- **分片数量设计**：
  - **数量 = 节点数 × 1.5 ~ 3**（每个节点分片数不宜过多）
  - 经验公式：`shard_size = index_size / 30GB`（单个分片建议 < 30GB）
  - **副本数**：高可用要求至少 1 个副本；性能瓶颈时增加副本数（读并发）
  - **避免过多分片**：每个分片有开销（内存、文件句柄），过多分片影响集群稳定性
- **SOC 告警索引**：`soc-alerts-{YYYY.MM}`（按月分索引），每索引 3 主分片 + 1 副本

**可能的追问**：
- 分片分配策略（shard allocation）由谁控制？什么是 forced awareness？
- 什么情况下分片会 unassigned？如何排查？
- 什么是"热数据"和"冷数据"分离？ILM（Index Lifecycle Management）如何配置？

---

### Q34：ES 的相关性评分 TF-IDF 和 BM25 的区别？你们在安全日志检索场景中使用什么评分策略？

**考察点**：相关性算法、BM25 优势

**答案要点**：
- **TF-IDF**：
  - `TF = 词项在文档中出现次数 / 文档词项总数`
  - `IDF = log(全部文档数 / 包含该词项的文档数)`
  - `Score = TF × IDF`
  - 问题：词项频率无限增长，TF 越高分越高；文档长度差异大（长文档天然 TF 高）
- **BM25**（ES 5.0+ 默认）：
  - 对 TF 的增长设置**饱和曲线**（TF 增加到一定程度后收益递减）
  - 引入了**文档长度归一化**（短文档匹配更重要）
  - `Score = IDF × (TF × (k1 + 1)) / (TF + k1 × (1 - b + b × dl/avgdl))`
  - 参数：`k1`（词频饱和度，默认 1.2）、`b`（长度归一化，默认 0.75）
- **SOC 安全日志场景评分策略**：
  - 主要用 **Boolean Query**（`filter` 不评分 + `must` 评分），告警检索以过滤为主
  - 时间排序优先：告警检索默认按 `@timestamp` 降序，评分辅助
  - 高严重级别加权：可通过 `boost` 参数提升 severity 高的告警排名
  - `constant_score`：已知是高危告警，忽略 TF-IDF 评分，直接给固定高分

**可能的追问**：
- BM25 的 `k1` 和 `b` 参数如何调优？不同数据集如何选择？
- ES 的 `boost` 参数有哪些用法？（索引级、字段级、查询级）
- 什么是 `function_score`？如何用它实现基于数值的自定义排序？

---

### Q35：你在 SOC 平台中用 ES 做安全事件检索，ES 的查询性能优化有哪些手段？

**点**：ES 性能优化实战

**答案要点**：
- **索引优化**：
  1. **减少字段数量**：`_source` 禁用不需要的字段，`dynamic: false` 禁止自动增加字段
  2. **合适的分片数**：单分片 20-50GB，避免过多小分片
  3. **合理的副本数**：读多写少可增加副本，读写均衡用 1 副本
- **查询优化**：
  1. **filter 替代 query**：`filter` 不计算评分，走缓存，性能更好
  2. **避免通配符前缀**：`*keyword` 无法利用倒排索引，会全表扫描
  3. **分页优化**：避免 `from + size` 深分页（最大 `from + size < 10000`），用 `search_after` 深度分页
  4. **只返回必要字段**：`_source` 限制返回字段，减少网络传输
  5. **预热查询**：对高频聚合查询做 cache warmup
- **架构优化**：
  1. **冷热分离**：热数据（7 天内）放 SSD 节点，冷数据归档到 HDD/ES Frozen
  2. **ILM 策略**：自动 rollup、冷数据 shrink、force merge
  3. **异步写入**：大批量数据用 `_bulk` API + 批量提交，减少请求次数

**可能的追问**：
- `search_after` 分页的原理是什么？它为什么能避免深分页的性能问题？
- 什么是 ES 的 `query cache` 和 `filter cache`？如何监控缓存命中率？
- `force merge` 什么时候做？force merge 过程中有什么注意事项？

---

### Q36：ES 的 Query DSL：`match`、`term`、`range`、`bool`、`filter` 的区别？`filter` 和 `query` 的性能差异？

**考察点**：Query DSL、filter 缓存

**答案要点**：
- **各查询类型**：
  - **`term`**：精确匹配，不分词。用于 keyword 类型字段（`status: "closed"`）
  - **`match`**：全文检索，会分词。用于 text 类型字段（`message: "failed login"`）
  - **`range`**：范围查询（`severity: { gte: 3 }`）
  - **`bool`**：组合查询，`must`（must_not）/`should`/`filter`
- **`query` vs `filter`**：
  | 特性 | query | filter |
  |------|-------|--------|
  | 计算评分 | ✅（计算 TF-IDF/BM25） | ❌（不做评分，0 分数） |
  | 缓存 | ❌ 不缓存 | ✅ 缓存（Bitset 缓存） |
  | 速度 | 慢（需计算相关性） | 快（缓存 + 无评分） |
  | 用途 | 全文搜索（需要相关性排序） | 精确过滤（YES/NO） |
- **最佳实践**：
  - **分离 filter 和 query**：`bool` 中用 `filter` 包裹确定过滤条件，`must` 包裹需要评分的搜索条件
  ```json
  {
    "bool": {
      "filter": [
        { "term": { "status": "open" } },
        { "range": { "timestamp": { "gte": "now-7d" } } }
      ],
      "must": [
        { "match": { "message": "SQL injection" } }
      ]
    }
  }
  ```

**可能的追问**：
- `bool` 查询中 `should` 如何与 `minimum_should_match` 配合？
- `match_phrase` 和 `match` 的区别是什么？什么是 slop？
- `term` 查询 keyword 类型和 text 类型的区别？

---

### Q37：ES 的聚合查询（Aggregations）：`terms`、`avg`、`sum`、`histogram` 在安全日志统计中的应用？

**考察点**：ES 聚合框架、安全分析应用

**答案要点**：
- **Bucket（桶聚合）vs Metric（指标聚合）**：
  - Bucket：按条件分组，每组一个桶（类似 GROUP BY）
  - Metric：计算指标（count/sum/avg/min/max）
- **常见聚合及安全场景应用**：
  - **`terms`（词项聚合）**：
    - 按攻击类型（`attack_type`）分组统计告警数量
    - 按 Top N 源 IP 统计：`{ "terms": { "field": "src_ip", "size": 10 } }`
  - **`avg/sum`**：
    - 平均告警响应时间（MTTR）：`avg` 聚合 `response_time`
    - 每月告警总量：`sum` 聚合每日告警数
  - **`histogram`**：
    - 按固定时间间隔（如 5 分钟）统计告警量，绘制告警趋势图
    - `{ "histogram": { "field": "timestamp", "interval": "5m" } }`
  - **`date_histogram`**：
    - 按天/周/月统计告警，适合趋势分析
  - **`cardinality`**：
    - 独立 IP 数量（去重计数）：`cardinality: { "field": "src_ip" }`
  - **`percentiles`**：
    - 告警处理时长分布（50th/95th/99th 分位）
- **嵌套聚合**：
  - 先按 `attack_type` 分桶，每桶内再按 `avg` 计算平均响应时间：
  ```json
  {
    "aggs": {
      "by_type": {
        "terms": { "field": "attack_type" },
        "aggs": {
          "avg_response_time": { "avg": { "field": "response_time" } }
        }
      }
    }
  }
  ```

**可能的追问**：
- `cardinality` 聚合的精度如何保证？HyperLogLog++ 算法是什么？
- ES Pipeline 聚合（Pipeline Aggregations）是什么？`bucket_selector` 的用法？
- 聚合查询的结果是否有缓存？缓存在哪一层？
