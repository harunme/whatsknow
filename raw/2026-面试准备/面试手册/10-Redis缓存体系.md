# Redis 缓存体系完全指南

## Summary

Redis 是 AI 工程实践中最重要的基础设施之一，在 RAG 结果缓存、多模型语义去重、分布式锁等场景中扮演关键角色。本手册系统梳理了 Redis 从基础使用到底层实现、从缓存策略到生产排查的完整知识体系，帮助准备 AI 应用工程师面试。

---

## 零、基础知识速查

### 0.1 Redis 是什么

Redis（Remote Dictionary Server）是一个开源的 **内存键值数据库**，支持字符串、哈希、列表、集合、有序集合等多种数据结构。通常用作**缓存层**、**会话存储**、**消息队列**和**分布式锁**。

核心特点：
- 数据存储在**内存**中，读写速度极快（QPS 可达 10万+）
- 支持**持久化**（RDB/AOF），可定期或实时将数据写入磁盘
- 支持**主从复制**，实现读写分离与高可用
- 单线程模型（6.0前），基于 I/O 多路复用（epoll），天然避免并发问题

### 0.2 安装与基本命令

```bash
# 安装（macOS）
brew install redis
redis-server          # 启动服务，默认端口 6379
redis-cli             # 进入命令行

# 安装（Linux）
sudo apt install redis-server
sudo systemctl start redis

# 基本连接与 ping
redis-cli ping        # 返回 PONG 表示正常

# 常用命令速查
redis-cli SET name "Alice"              # 设置字符串
redis-cli GET name                       # 获取 → "Alice"
redis-cli EXISTS name                    # 是否存在 → 1
redis-cli DEL name                       # 删除
redis-cli KEYS "user:*"                 # 模式匹配查找 key（生产慎用）
redis-cli FLUSHALL                       # 清空所有数据（危险！）
redis-cli DBSIZE                         # 当前数据库 key 数量
redis-cli INFO                           # 查看 Redis 状态信息
redis-cli CONFIG GET maxmemory          # 查看最大内存配置
redis-cli AUTH password                 # 带密码连接
redis-cli SELECT 1                       # 切换到 db1
redis-cli FLUSHDB                        # 清空当前数据库
```

### 0.3 常用数据类型与命令对照

| 类型 | 典型命令 | 应用场景 |
|------|----------|----------|
| **String** | `SET/GET/MSET/MGET/INCR/DECR/EXPIRE` | Token 存储、计数器、JSON 缓存 |
| **Hash** | `HSET/HGET/HGETALL/HDEL` | 用户会话、对象缓存、Map 结构 |
| **List** | `LPUSH/RPOP/LRANGE/LEN` | 任务队列、最新消息队列、时间线 |
| **Set** | `SADD/SMEMBERS/SISMEMBER/SCARD` | 标签、去重、关注列表 |
| **ZSet** | `ZADD/ZRANGE/ZSCORE/ZRANK` | 排行榜、延迟队列、权重排序 |
| **Bitmap** | `SETBIT/GETBIT/BITCOUNT` | 签到、活跃用户统计 |
| **HyperLogLog** | `PFADD/PFCOUNT` | UV 统计（去重计数） |
| **Geo** | `GEOADD/GEOPOS/GEODIST` | 附近的人、LBS 应用 |

```python
# Python 客户端：redis-py 基础用法
import redis

r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# String
r.set('user:token:1001', 'sk-xxx', ex=3600)   # 1小时后自动过期
r.setnx('lock:order:123', 'locked')            # 不存在才设（分布式锁基础）
r.incr('counter:page_view')                    # 原子递增

# Hash（适合存对象）
r.hset('user:1001', mapping={'name': 'Alice', 'role': 'admin'})
r.hget('user:1001', 'name')                    # → 'Alice'
r.hgetall('user:1001')                         # → {'name': 'Alice', 'role': 'admin'}

# List（队列）
r.lpush('task:queue', 'task_a', 'task_b')
r.rpop('task:queue')                           # 先进先出

# ZSet（排行榜）
r.zadd('leaderboard', {'Alice': 100, 'Bob': 90})
r.zrevrange('leaderboard', 0, 9, withscores=True)  # 获取前10名

# Key 过期
r.expire('temp:data', 300)                     # 5分钟后自动删除
r.ttl('temp:data')                             # 查看剩余 TTL（秒）
```

### 0.4 常见使用场景速查

```
场景                        选用的数据类型
─────────────────────────────────────────────────────
缓存 LLM 回复结果           String（JSON 序列化）
存储用户会话                Hash
实现分布式锁                String（SETNX + EXPIRE）
消息队列                    List（LPUSH + BRPOP）
排行榜                      ZSet
去重（IP 访问记录）          HyperLogLog / Set
限制请求频率                 String（INCR + EXPIRE）
缓存热点数据                String + Hash
```

### 0.5 客户端与连接池

```python
import redis
from redis import ConnectionPool

# 连接池模式（推荐，避免频繁创建连接）
pool = ConnectionPool(host='localhost', port=6379, max_connections=20, decode_responses=True)
r = redis.Redis(connection_pool=pool)

# Cluster 模式
from redis.cluster import RedisCluster
rc = RedisCluster(host='localhost', port=7000)

# Sentinel 模式（高可用）
from redis.sentinel import Sentinel
sentinel = Sentinel([('localhost', 26379)], socket_timeout=0.1)
master = sentinel.master_for('mymaster', socket_timeout=0.1)
```

### 0.6 常见面试基础问题

**Q: Redis 为什么这么快？**
1. 基于内存，磁盘 I/O 不是瓶颈
2. 单线程（6.0前）避免锁开销，基于 epoll/IOCP 的 I/O 多路复用
3. C 语言实现，执行效率高
4. 丰富的数据结构减少序列化/反序列化开销

**Q: Redis 和 Memcached 的区别？**
| 维度 | Redis | Memcached |
|------|-------|-----------|
| 数据结构 | 多种 | 仅 String |
| 持久化 | 支持 RDB/AOF | 不支持 |
| 主从复制 | 支持 | 支持但较弱 |
| 内存管理 | 多种淘汰策略 | LRU |
| 存储容量 | 可存大 value | 不适合大 value（1MB 限制）|
| 线程模型 | 单线程 | 多线程 |

---

## 一、五大数据结构与底层实现

### 1.1 各数据类型核心用法

Redis 提供了 five 官方推荐的核心数据类型，每种都有明确的适用场景：

```python
import redis

r = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)

# String：计数器、Token、JSON 序列化缓存
r.set("llm:token:user_001", "sk-xxxx", ex=3600)
r.set("rag:doc:hash", json.dumps(doc_vector), ex=86400)

# Hash：会话上下文、用户画像（避免大 Key）
r.hset("session:123", mapping={"model": "gpt-4", "temperature": "0.7", "messages": "50"})

# List：任务队列、最新日志、时间线
r.lpush("ai:task:queue", json.dumps({"task_id": "t_001", "type": "embedding"}))
r.lrange("ai:task:queue", 0, -1)

# Set：标签去重、关注列表、共同好友
r.sadd("user:tags:123", "rag", "llm", "agent")
r.sinter("user:tags:123", "user:tags:456")  # 共同兴趣

# ZSet：排行榜、优先级队列、按时间排序的feed
r.zadd("model:latency:ranking", {"gpt-4o": 3200.5, "claude-3-5": 1800.2})
r.zrevrange("model:latency:ranking", 0, 9, withscores=True)
```

### 1.2 SDS 动态字符串优势

Redis 的 String 类型底层使用 SDS（Simple Dynamic String），而非 C 原生 char 数组，这是 Redis 高性能的关键设计：

| 特性 | C 字符串 | SDS |
|------|----------|-----|
| 长度获取 | O(n) 遍历 | O(1) 记录在 len 字段 |
| 追加操作 | 缓冲区溢出风险 | 空间预分配 + 惰性释放 |
| 二进制安全 | 否（遇 `\0` 截断） | 是（用 len 而非 `\0` 判断） |
| 内存分配 | 每次可能全量 realloc | 指数增长策略，减少分配次数 |

SDS 在 Redis 3.0 后引入 `embstr` 优化：短字符串（<=44 字节）使用一块连续内存存储 redisObject + SDS 头 + 内容，减少内存碎片。

### 1.3 底层数据结构选择

Redis 6.x 针对不同数据规模和场景，在各类型内部选择最优底层结构：

- **Hash**：`ziplist`（元素 < 512，单个 < 64 字节）→ `hashtable`
- **List**：`ziplist`（< 128 字节）→ `quicklist`（3.2+ 合并了 ziplist + linkedlist，压缩率高）
- **ZSet**：`ziplist`（< 128 字节）→ `skiplist + dict`（同时支持 O(log N) 查分和 O(1) 查 score）
- **Set**：`intset`（全为整数，< 512 个）→ `hashtable`

理解这些内部转换条件，对避免 bigkey 问题至关重要。

---

## 二、持久化机制：RDB / AOF / 混合持久化

### 2.1 三种模式对比

```python
# redis.conf 配置示例
# RDB 模式：定时快照，适合冷备和灾难恢复
save 900 1      # 900s 内有 1 次写操作则触发 bgsave
save 300 10     # 300s 内有 10 次写操作则触发 bgsave
save 60 10000   # 60s 内有 10000 次写操作则触发 bgsave

# AOF 模式：每个写命令追加到文件，数据可靠性更高
appendonly yes
appendfsync everysec   # 推荐：每秒同步，最多丢 1 秒数据
# appendfsync always   # 每次写都同步，最安全但性能差
# appendfsync no       # 交给 OS 决定，可能丢失大量数据

# 混合持久化（Redis 4.0+，5.0 默认）：兼顾两者优点
aof-use-rdb-preamble yes
```

### 2.2 各自的优缺点

**RDB 优点**：文件紧凑（全量快照压缩），恢复速度快（fork 子进程，不阻塞主线程）；适合做冷备，可直接传输到异地。**缺点**：两次快照之间数据丢失，无法精确恢复；fork 操作在数据量大时可能阻塞（Linux 下使用 COW，耗时约几十毫秒到几秒）。

**AOF 优点**：数据安全性高，最多丢 1 秒数据；rewrite 机制（bgrewriteaof）可压缩文件。**缺点**：文件体积通常比 RDB 大；每次写都涉及磁盘 IO，QPS 高时性能有明显开销。

**企业为什么同时开两者**：RDB 作为快速恢复的备份层；AOF 提供实时数据保障。Redis 重启时优先加载 AOF（数据更完整），同时 RDB 可作为 AOF rewrite 失败时的安全垫。

---

## 三、缓存穿透：布隆过滤器 + 空值缓存

**缓存穿透**：查询一个根本不存在的数据（数据不在数据库，也不在缓存），每次请求都穿透到数据库。攻击者可通过构造大量不存在 ID 的请求打垮数据库。

### 3.1 布隆过滤器（RedisBloom）

```python
# pip install redisbloom
from bloom_filter import ScalableBloomFilter

# 初始化布隆过滤器，预先存入所有合法数据 ID
bf = ScalableBloomFilter(mode=ScalableBloomFilter.SMALL_SET_GROWTH)
for doc_id in all_existing_doc_ids:
    bf.add(doc_id)

# 查询时先检查布隆过滤器
def query_doc(doc_id: str):
    if doc_id not in bf:  # 布隆过滤器说不存在，直接返回
        return None
    # 可能存在（假阳性），继续查缓存和数据库
    cached = r.get(f"doc:{doc_id}")
    if cached:
        return json.loads(cached)
    return db.query(doc_id)
```

### 3.2 空值缓存（短 TTL）

```python
def query_with_null_cache(key: str, db_getter, ttl: int = 300):
    val = r.get(key)
    if val is not None:
        return None if val == "__NULL__" else json.loads(val)

    # 查数据库
    result = db_getter(key)
    if result is None:
        r.setex(f"cache:{key}", ttl, "__NULL__")  # 缓存空值，TTL 短一些
    else:
        r.setex(f"cache:{key}", 3600, json.dumps(result))
    return result
```

### 3.3 接口限流（结合 Redis 实现）

```python
def rate_limit(redis_conn, key: str, max_requests: int, window: int = 60) -> bool:
    """滑动窗口限流"""
    now = time.time()
    pipe = redis_conn.pipeline()
    pipe.zremrangebyscore(key, 0, now - window)
    pipe.zcard(key)
    pipe.zadd(key, {str(now): now})
    pipe.expire(key, window)
    counts, _ = pipe.execute()[1], pipe.execute()[2]
    return counts < max_requests
```

---

## 四、缓存击穿与雪崩

### 4.1 概念区别

- **缓存击穿**：某个热点 key 过期瞬间，大量并发请求全部打到数据库（一个人击穿 vs 万人穿）。
- **缓存雪崩**：大量普通 key 在同一时间集中过期，或者 Redis 本身宕机，导致数据库压力骤增。

两者本质都是缓存失效引发的数据库压力，但击穿是"点"，雪崩是"面"。

### 4.2 分布式锁解决击穿

核心思路：用分布式锁保证只有一个请求去加载数据，其他请求等待或走兜底逻辑：

```python
import uuid, time

def get_with_lock(redis_conn, key: str, db_loader, lock_timeout: int = 10):
    # 1. 先查缓存
    val = redis_conn.get(key)
    if val is not None:
        return json.loads(val)

    # 2. 抢分布式锁
    lock_key = f"lock:{key}"
    lock_val = str(uuid.uuid4())
    acquired = redis_conn.set(lock_key, lock_val, nx=True, ex=lock_timeout)

    if not acquired:
        # 没抢到锁：等一小会儿再查缓存（最多重试3次）
        for _ in range(3):
            time.sleep(0.1)
            val = redis_conn.get(key)
            if val is not None:
                return json.loads(val)
        return None  # 兜底返回

    # 3. 抢到锁：从数据库加载并写回缓存
    try:
        result = db_loader(key)
        if result:
            redis_conn.setex(key, 3600, json.dumps(result))
        return result
    finally:
        # 4. 释放锁（只释放自己的锁）
        if redis_conn.get(lock_key) == lock_val:
            redis_conn.delete(lock_key)
```

### 4.3 防止雪崩

1. 热点数据永不过期：用逻辑过期字段（`logical_expire`）代替物理过期；
2. 过期时间随机化：所有 key 的 TTL 加上 `random.randint(0, 300)`；
3. Redis 高可用部署：主从 + Sentinel 或 Cluster，避免全量不可用；
4. 多级缓存：本地进程缓存（Caffeine/Guava）+ Redis + 数据库，L1 挡住大部分请求。

---

## 五、集群方案：主从复制 / Sentinel / Cluster

### 5.1 三种方案适用场景

| 方案 | 架构 | 适用场景 | 数据分片 | 故障自动切换 |
|------|------|----------|----------|------------|
| **主从复制** | 1 主 N 从，只读 | 读写分离，数据量 < 10GB | 否（全量复制） | 否（需手动切换） |
| **Sentinel** | 主从 + 监控进程 | 需要自动故障切换，数据量 < 10GB | 否 | 是 |
| **Cluster** | 分片 + 主从 | 数据量大（> 10GB），需水平扩容 | 是（16384 槽） | 是（各分片独立切换） |

### 5.2 主从复制原理

Redis 2.8 前使用 SYNC（全量同步），之后支持 PSYNC，支持断点续传：

```
主节点 → 全量同步（bgsave + 发送 RDB）→ PSYNC 增量同步（主从断开后，offset 续传）
```

**AI 场景实战**：在多模型框架中，将 LLM 请求日志写入主节点，多个 Agent 副本从从节点读取，实现日志采集与业务隔离。

### 5.3 为什么是 16384 个哈希槽

这个问题是高频面试题。核心原因：

1. **网络包大小**：节点间心跳包（ping/pong）携带槽位信息，16384 个 bit = 2KB；65,536 个则需要 8KB，在集群规模大（百节点以上）时，心跳带宽会成为瓶颈；
2. **迁移效率**：槽位越小，rebalance 时单个槽移动的数据量越少，重平衡粒度更精细；
3. **足够用**：16384 个槽，每槽可对应约数十万 key，足够支撑到百节点集群；
4. **Gossip 协议**：Redis Cluster 使用 Gossip 协议进行节点通信，2KB 心跳包在 100 节点下每轮仅 200KB，足够低延迟。

---

## 六、分布式锁

### 6.1 SETNX + EXPIRE 的原子性问题

**不安全的实现**（教科书经典反例）：
```python
# 这两行不是原子的！SETNX 成功后、EXPIRE 执行前进程崩溃，锁永不过期
redis_conn.setnx(lock_key, lock_val)
redis_conn.expire(lock_key, 10)
```

### 6.2 正确的原子锁（SET + EXPIRE 一步到位）

```python
# Redis 2.6.12+ 支持 set(key, value, ex=seconds, nx=True) 原子操作
def acquire_lock(redis_conn, lock_key: str, lock_val: str, ttl: int = 10) -> bool:
    return redis_conn.set(lock_key, lock_val, nx=True, ex=ttl)

def release_lock(redis_conn, lock_key: str, lock_val: str):
    # Lua 脚本保证：只有持有锁的进程才能释放（防止误删他人锁）
    lua = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """
    redis_conn.eval(lua, 1, lock_key, lock_val)
```

### 6.3 RedLock 解决了什么

单节点 Redis 锁在主节点宕机后，从节点晋升期间锁会丢失。RedLock 的思路：向 N 个独立 Redis 节点（奇数个，如 5 个）同时获取锁，超过 N/2+1 节点成功才算加锁成功。**解决了单点故障导致的锁丢失问题**，但引入了时钟漂移、运维复杂度增加等问题，在大多数场景下单节点 Redis 锁 + 合理的 TTL + 重试机制已足够。

---

## 七、缓存一致性：三种模式与延迟双删

### 7.1 三种缓存更新模式

| 模式 | 读流程 | 写流程 | 一致性 | 性能 |
|------|--------|--------|--------|------|
| **Cache Aside** | 读缓存 → 未命中 → 读库 → 写缓存 | 先更新库 → 删除缓存 | 最终一致 | 最常用 |
| **Read Through** | 缓存不存在时，缓存层自动加载 | 同上 | 最终一致 | 读友好 |
| **Write Behind** | — | 先写缓存 → 异步批量写库 | 最终一致（延迟大） | 写友好 |

**Cache Aside**（旁路缓存）是最推荐的模式：业务代码直接操作缓存和数据库，缓存层不感知数据库变化。

### 7.2 延迟双删（解决并发不一致）

在并发场景下，写线程先更新数据库（A），在删除缓存（C）之前，读线程可能已经将旧数据（O）写回了缓存。解决：

```python
def update_with_double_delete(redis_conn, key: str, new_data: dict):
    # 1. 先删除缓存
    redis_conn.delete(key)

    # 2. 更新数据库
    db.update(key, new_data)

    # 3. 延迟 500ms 后再删除一次（给并发的读线程留出时间窗口）
    time.sleep(0.5)
    redis_conn.delete(key)
```

**注意**：延迟时间需要大于一次读 + 写的平均耗时，通常 200-500ms。在 AI 场景下，如果缓存写入本身有 100ms 延迟，延迟双删的间隔应适当放大。

---

## 八、Token 成本优化：语义缓存实战

### 8.1 什么是语义缓存

传统的精确 key 缓存（如 MD5(prompt)）对 LLM 请求无效：用户每次表述不同，key 就不同，但语义完全一致，此时需要**语义缓存**。

### 8.2 基于向量相似度的语义缓存

```python
import numpy as np

class SemanticCache:
    def __init__(self, redis_conn, embedding_model):
        self.r = redis_conn
        self.embed = embedding_model  # sentence-transformers 等嵌入模型
        self.threshold = 0.85          # 余弦相似度阈值

    def get_or_query(self, prompt: str, llm_call_fn):
        """语义缓存查询：先找相似 prompt，有则返回缓存结果"""
        query_vec = self.embed(prompt)

        # 从 Redis 扫描所有缓存的向量，计算相似度
        keys = self.r.keys("scache:*:vec")
        best_score, best_result = -1, None

        for key in keys:
            cached_vec_str = self.r.get(key)
            cached_vec = np.array(json.loads(cached_vec_str))
            score = np.dot(query_vec, cached_vec) / (
                np.linalg.norm(query_vec) * np.linalg.norm(cached_vec)
            )

            if score > self.threshold and score > best_score:
                # 命中：取对应结果
                result_key = key.replace(":vec", ":result")
                best_result = self.r.get(result_key)
                best_score = score
                self.r.incr(key.replace(":vec", ":hit"))  # 记录命中次数

        if best_result:
            print(f"语义缓存命中，相似度: {best_score:.3f}")
            return json.loads(best_result)

        # 未命中：调用 LLM，结果写入缓存
        result = llm_call_fn(prompt)
        cache_key = f"scache:{uuid.uuid4().hex[:8]}"
        self.r.setex(cache_key + ":vec", 86400 * 7, json.dumps(query_vec.tolist()))
        self.r.setex(cache_key + ":result", 86400 * 7, json.dumps(result))
        self.r.setex(cache_key + ":hit", 86400 * 7, "0")

        return result

    def cleanup_low_hit_rate(self, min_hits: int = 3):
        """清理命中率低的缓存，节省成本"""
        keys = self.r.keys("scache:*:hit")
        for key in keys:
            if int(self.r.get(key) or 0) < min_hits:
                prefix = key.replace(":hit", "")
                self.r.delete(prefix + ":vec", prefix + ":result", prefix + ":hit")
```

**语义缓存的价值**：在 RAG 系统和多模型框架中，相同的用户意图（不同表述）往往触发相同或相似的 LLM 调用，语义缓存可节省 20-40% 的 Token 成本。

### 8.3 混合缓存策略

```python
def llm_request_cache(redis_conn, prompt: str, model: str, embed_model):
    """精确缓存（快）+ 语义缓存（兜底）+ 布隆过滤器（防护）"""
    # 第一层：精确缓存（exact hash）
    exact_key = f"exact:{hashlib.sha256(prompt.encode()).hexdigest()}"
    if redis_conn.exists(exact_key):
        return json.loads(redis_conn.get(exact_key))

    # 第二层：语义缓存（向量相似度）
    semantic = SemanticCache(redis_conn, embed_model)
    result = semantic.get_or_query(prompt, lambda p: call_llm_api(p, model))

    # 第三层：写入精确缓存
    ttl = 86400 * 7 if semantic.threshold > 0.9 else 86400
    redis_conn.setex(exact_key, ttl, json.dumps(result))
    return result
```

---

## 九、生产问题排查

### 9.1 BigKey 发现与处理

```python
# 快速发现大 key（Redis 4.0+ 自带）
# redis-cli --bigkeys                          # 扫描全库，按类型报告最大的 key
# redis-cli --scan --pattern '*' | head -1000 | xargs redis-cli --bigkeys  # 指定范围

# Python 扫描脚本（找出超过指定大小的 key）
def find_big_keys(redis_conn, min_size_mb: int = 10):
    big_keys = []
    for key in redis_conn.scan_iter(match="*", count=1000):
        key_type = redis_conn.type(key)
        if key_type == "string":
            size = redis_conn.memory_usage(key)
        elif key_type in ("list", "zset", "set", "hash"):
            size = sum(redis_conn.memory_usage(f"{key}:{i}") or 0
                       for i in range(redis_conn_llen(key) if key_type == "list" else 0))
        else:
            continue
        if size > min_size_mb * 1024 * 1024:
            big_keys.append((key, key_type, size // (1024 * 1024)))
    return big_keys
```

**BigKey 的危害**：单 key 过大导致 RDB 加载慢、AOF rewrite 阻塞、操作超时（HGETALL 可能卡几十秒）。AI 场景中，存储长文本的 embedding 结果或对话历史时尤其容易出现。

### 9.2 连接数爆了

```python
# 查看当前连接数
# redis-cli info clients
# redis-cli client list | wc -l

# 常见原因：连接池配置不当（未设置 max_connections）、未使用连接池、短连接频繁创建

# 正确配置连接池
pool = redis.ConnectionPool(
    host='localhost', port=6379,
    max_connections=50,    # 核心参数：根据服务实例数和 Redis 配置设置
    socket_timeout=3,
    socket_connect_timeout=3,
    retry_on_timeout=True
)
client = redis.Redis(connection_pool=pool)
```

### 9.3 CPU 和内存突增排查

```python
# 监控脚本：每秒采样，输出异常趋势
def monitor_redis(redis_conn, duration: int = 60):
    samples = []
    for _ in range(duration):
        info = redis_conn.info()
        samples.append({
            "ts": time.time(),
            "cpu_sys": info.get("used_cpu_sys", 0),
            "used_memory_mb": info.get("used_memory") / (1024*1024),
            "connected_clients": info.get("connected_clients", 0),
            "instantaneous_ops": info.get("instantaneous_ops_per_sec", 0),
            "keyspace_hits": info.get("keyspace_hits", 0),
            "keyspace_misses": info.get("keyspace_misses", 0),
        })
        time.sleep(1)

    # 异常检测
    cpu_avg = np.mean([s["cpu_sys"] for s in samples])
    mem_trend = np.polyfit(range(len(samples)), [s["used_memory_mb"] for s in samples], 1)[0]
    if mem_trend > 10:  # 每秒内存增长 > 10MB
        print(f"警告：内存泄漏嫌疑，趋势 {mem_trend:.2f} MB/s")
    if cpu_avg > 80:
        print(f"警告：CPU 使用率过高 {cpu_avg:.1f}%")
```

**常见原因**：`KEYS *` 命令阻塞主线程（生产禁用，改用 `SCAN`）；内存碎片率高（`activedefrag yes`）；大 Hash/Set 未设置过期时间导致持续增长。

---

## 十、Redis 在 AI 系统中的应用

### 10.1 RAG 结果缓存

```python
def rag_with_cache(redis_conn, query: str, vector_store, llm_fn, top_k: int = 5):
    """RAG pipeline 带缓存：query hash + doc chunk hash 双 key"""
    # 计算 query 的 embedding
    q_emb = embed(query)

    # 检查 query 级别缓存
    cache_key = f"rag:q:{hashlib.sha256(query.encode()).hexdigest()}"
    if redis_conn.exists(cache_key):
        return json.loads(redis_conn.get(cache_key))

    # 语义检索
    docs = vector_store.similarity_search(query, k=top_k)
    doc_hash = hashlib.md5("|".join([d.page_content for d in docs]).encode()).hexdigest()

    # 检查 docs 级别缓存（相同的 chunk 内容直接复用 LLM 结果）
    doc_cache_key = f"rag:doc:{doc_hash}"
    if redis_conn.exists(doc_cache_key):
        cached_context = json.loads(redis_conn.get(doc_cache_key))
    else:
        cached_context = [d.page_content for d in docs]
        redis_conn.setex(doc_cache_key, 86400 * 30, json.dumps(cached_context))

    # 调用 LLM
    answer = llm_fn(query, cached_context)
    result = {"answer": answer, "sources": [d.metadata for d in docs]}

    redis_conn.setex(cache_key, 3600, json.dumps(result))
    return result
```

### 10.2 多模型结果去重与优先级队列

```python
# 去重：相同 task_id 的任务，只执行一次，重复请求复用结果
def deduplicated_model_call(redis_conn, task_id: str, model: str, prompt: str):
    dedup_key = f"dedup:{hashlib.sha256((task_id + model).encode()).hexdigest()}"
    if redis_conn.exists(dedup_key):
        result = redis_conn.get(dedup_key)
        print(f"去重命中：task={task_id}, model={model}")
        return json.loads(result)

    result = call_model_api(model, prompt)
    redis_conn.setex(dedup_key, 7200, json.dumps(result))
    return result

# APScheduler 任务队列（结合 Redis 实现分布式调度）
from apscheduler.schedulers.background import BackgroundScheduler
from apscheduler.jobstores.redis import RedisJobStore

jobstores = {"default": RedisJobStore(hosts=[{"host": "localhost", "port": 6379}])}
scheduler = BackgroundScheduler(jobstores=jobstores)
scheduler.add_job(flush_embedding_results, "interval", seconds=30, id="flush_embeddings")
scheduler.start()
```

---

## 面试 Q&A

**Q1：Redis 的线程模型是怎样的？**
Redis 4.x 之前是单线程（纯内存操作，无锁竞争），6.0 引入了 IO 多线程（`io-threads 4`），将网络 IO 和命令解析并行化，但命令执行仍是单线程。这保证了 Redis 的线程安全特性。

**Q2：Redis 和 Memcached 的核心区别是什么？**
Redis 支持五大数据结构（String/Hash/List/Set/ZSet），Memcached 只支持 KV String；Redis 支持持久化（RDB/AOF），Memcached 不支持；Redis 支持主从复制和集群，Memcached 无集群方案；Redis 支持 Lua 脚本原子操作，Memcached 不支持。

**Q3：Redis 的过期 key 删除策略是什么？**
两种策略并存：**惰性删除**——访问 key 时检查是否过期，过期则删除（节省 CPU，但过期 key 占用内存）；**定期删除**——每隔一段时间，随机检查部分 key，删除过期的（平衡 CPU 和内存）。两种结合是 Redis 默认行为，RDB 和 AOF 持久化时对过期 key 也有特殊处理（从库不会主动删除过期 key，等主库发送 DEL 命令）。

**Q4：为什么 Redis 用跳表而不是红黑树做 ZSet？**
跳表实现简单，比红黑树更容易做范围查询（ZRANGE/ZRANGEBYSCORE）；插入/删除时只需局部更新 O(log N)，无需旋转； Redis 作者 antirez 个人偏好，以及在并发场景下跳表更容易做无锁并发扩展。

**Q5：Redis Cluster 为什么不支持跨槽事务（MULTI/EXEC）？**
Redis Cluster 每个 key 路由到固定槽位，不同 key 可能在不同节点，无法在单节点完成事务。解决方案：使用 `{tag}.key` 强制同槽（用 `{}` 包裹相同 tag），或使用 Lua 脚本跨节点批量操作（需 Redis 6.0+ 支持）。

**Q6：生产环境如何规划 Redis 内存？**
建议预留 30% buffer（`maxmemory` 设置为机器内存 * 70%）；启用 `maxmemory-policy allkeys-lru` 防止 OOM；开启内存碎片整理（`activedefrag yes`）；定期 `INFO memory` 监控 `mem_fragmentation_ratio`（超过 1.5 需关注）。

---

*Last Updated: 2026-04-09*
