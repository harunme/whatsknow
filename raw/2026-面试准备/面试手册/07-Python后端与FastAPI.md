# Python 后端开发与 FastAPI 面试手册

## Summary

本手册面向具备一定 FastAPI 开发经验的高级 AI 应用工程师，系统梳理 Python 后端开发从基础到高级的核心技术及面试高频考点。内容覆盖 Python 基础语法与数据结构、asyncio 事件循环的底层机制、FastAPI 的性能优势与依赖注入体系、Pydantic v2 的数据验证与序列化优化、APScheduler 任务调度最佳实践，以及并发编程、装饰器原理、内存管理等进阶主题。

FastAPI 的核心竞争力在于其基于 Starlette 和 Uvicorn 的异步架构，配合 Pydantic v2 的高性能序列化，实现了远高于 Flask/Django 的并发吞吐能力。理解 async/await 的事件循环机制、协程与线程的区别、以及何时应该使用异步或 CPU 密集型处理，是后端工程师的必备技能。本手册以 Python 3.11+ 为基准，结合真实生产场景中的代码示例，帮助读者建立从原理到实践的完整知识链条，为面试中的深度追问做好充分准备。

---

## 零、基础知识速查

### 0.1 Python 基本数据类型

```python
# 基础类型
name: str = "Alice"           # 字符串
age: int = 30                 # 整数
score: float = 99.5           # 浮点数
is_active: bool = True       # 布尔值

# 容器类型
lst: list[int] = [1, 2, 3, 4, 5]
dct: dict[str, int] = {"Alice": 90, "Bob": 85}
s: set[int] = {1, 2, 3}       # 无序不重复
t: tuple[int, str] = (1, "hello")  # 不可变

# 列表操作
lst.append(6)                 # 末尾添加
lst.insert(0, 0)              # 指定位置插入
lst.pop()                     # 弹出末尾
lst.remove(3)                 # 删除第一个3
lst.sort()                    # 排序（原地）
sorted(lst, reverse=True)     # 返回新列表

# 字典操作
dct.get("Alice", 0)           # 安全获取，不存在返回默认值
dct.keys()                    # dict_keys(['Alice', 'Bob'])
dct.values()                  # dict_values([90, 85])
dct.items()                   # dict_items([('Alice', 90), ...])
dct.update({"Charlie": 95})   # 批量更新
```

### 0.2 函数与参数

```python
# 位置参数 vs 关键字参数
def greet(name: str, age: int = 18, *, city: str = "Beijing") -> str:
    """带默认值、仅关键字参数的函数"""
    return f"{name}, {age} years old, from {city}"

greet("Alice")                           # 位置
greet("Bob", 25)                         # 位置 + 默认值
greet("Charlie", city="Shanghai")         # 关键字参数
# greet(city="HK", "Daniel")  ← 错误：关键字参数不能在位置参数前

# *args 和 **kwargs
def sum_all(*args: int, **kwargs: float) -> int:
    total = sum(args)
    for v in kwargs.values():
        total += int(v)
    return total

sum_all(1, 2, 3, a=1.5, b=2.5)  # → 10

# Lambda 表达式
square = lambda x: x ** 2
pairs = [(1, "one"), (3, "three"), (2, "two")]
pairs.sort(key=lambda x: x[0])   # 按第一个元素排序

# 列表推导式（Pythonic 写法）
squares = [x**2 for x in range(10)]       # [0, 1, 4, 9, ...]
evens   = [x for x in range(20) if x % 2 == 0]
dict_   = {name: len(name) for name in ["Alice", "Bob", "Charlie"]}

# 生成器（惰性求值，节省内存）
gen = (x**2 for x in range(10**8))  # 不立即计算
next(gen)  # 0
next(gen)  # 1
# 内存占用远小于列表

def fibonacci(n: int):
    """yield 生成器函数"""
    a, b = 0, 1
    for _ in range(n):
        yield a
        a, b = b, a + b

list(fibonacci(10))  # [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

### 0.3 类与面向对象

```python
from dataclasses import dataclass, field
from typing import Optional

# 数据类（dataclass，自动生成 __init__/__repr__/__eq__）
@dataclass
class AlertEvent:
    alert_id: str
    severity: str = "P2"
    status: int = 0
    tags: list[str] = field(default_factory=list)

    def is_critical(self) -> bool:
        return self.severity == "P0"

    @property
    def display_name(self) -> str:
        return f"[{self.severity}] {self.alert_id}"

alert = AlertEvent(alert_id="A001", severity="P0")
print(alert)              # AlertEvent(alert_id='A001', severity='P0', status=0, tags=[])
print(alert.is_critical()) # True

# 继承
class SecurityAlert(AlertEvent):
    def __init__(self, alert_id: str, src_ip: str, **kwargs):
        super().__init__(alert_id, **kwargs)
        self.src_ip = src_ip

    def get_ip_blocklist(self) -> list[str]:
        """生成封禁 IP 列表的逻辑"""
        if self.severity in ("P0", "P1"):
            return [self.src_ip]
        return []
```

### 0.4 文件与异常处理

```python
import json
from pathlib import Path

# 上下文管理器（with 自动关闭）
with open("config.json", "r", encoding="utf-8") as f:
    config = json.load(f)

# 或者用 Path（更现代）
config = json.loads(Path("config.json").read_text(encoding="utf-8"))

# 异常处理
try:
    result = 10 / 0
except ZeroDivisionError as e:
    print(f"除零错误: {e}")
except (ValueError, TypeError) as e:
    print(f"值类型错误: {e}")
except Exception as e:
    print(f"未知错误: {e}")
    raise  # 重新抛出
else:
    print("没有异常时执行")
finally:
    print("无论如何都执行（如清理资源）")

# 自定义异常
class AlertProcessError(Exception):
    """告警处理异常"""
    def __init__(self, alert_id: str, reason: str):
        self.alert_id = alert_id
        self.reason = reason
        super().__init__(f"告警 {alert_id} 处理失败: {reason}")
```

### 0.5 常用标准库

```python
# 日期时间
from datetime import datetime, timedelta
now = datetime.now()
later = now + timedelta(hours=2)
print(now.strftime("%Y-%m-%d %H:%M:%S"))  # 2026-04-09 14:30:00

# 日志
import logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
logger = logging.getLogger(__name__)
logger.info("告警处理完成")

# 正则表达式
import re
ip_pattern = r"\b(?:[0-9]{1,3}\.){3}[0-9]{1,3}\b"
ips = re.findall(ip_pattern, "源IP: 192.168.1.10 目标: 10.0.0.1")

# 枚举
from enum import Enum
class Severity(str, Enum):
    P0 = "P0"
    P1 = "P1"
    P2 = "P2"
    P3 = "P3"

    def color(self) -> str:
        return {"P0": "red", "P1": "orange", "P2": "yellow", "P3": "gray"}[self.value]

print(Severity.P0.value)      # "P0"
print(Severity.P0.color())    # "red"
```

### 0.6 常见面试基础问题

**Q: Python 中 list 和 tuple 的区别？**
- `list` 是可变的（mutable），可以增删改；`tuple` 不可变（immutable），可作为字典 key 和 set 元素
- `tuple` 性能略高（无需维护额外空间），但实际差异不大
- tuple 常用于固定结构（函数多返回值、数据库行）

**Q: Python 是解释型语言吗？**
- 是，但不完全准确。Python 代码先编译成 `.pyc` 字节码（`.py` 同目录 `__pycache__/`），再由 Python 虚拟机解释执行
- 编译步骤在运行时自动完成，不需要显式编译

**Q: `is` 和 `==` 的区别？**
- `is` 比较的是对象身份（内存地址），是否同一对象
- `==` 比较的是值是否相等
```python
a = [1, 2, 3]
b = [1, 2, 3]
print(a == b)   # True（值相等）
print(a is b)   # False（不同对象，地址不同）

c = b
print(c is b)   # True（同一个对象）
```

---

## 1. asyncio 事件循环

### 1.1 事件循环的工作原理

asyncio 是 Python 3.4 引入的标准库，其核心是事件循环（Event Loop）。事件循环本质上是一个无限循环，负责调度和执行协程：

```python
import asyncio

# 事件循环的简化工作流程
async def main():
    print("start")
    await asyncio.sleep(1)  # 挂起，yield 控制权
    print("done")

# Python 3.7+ 推荐写法
asyncio.run(main())  # 创建事件循环，运行协程，关闭循环
```

事件循环内部维护一个任务队列（Task Queue）。当一个协程执行 `await` 时，当前协程被挂起，事件循环从队列中取出下一个就绪的协程执行。这个过程是单线程的，不需要切换操作系统内核上下文，因此协程切换的开销远低于线程切换。

### 1.2 协程 vs 多线程

| 维度 | 协程（asyncio） | 多线程 |
|------|----------------|--------|
| 切换方式 | 主动 yield（协作式） | 操作系统抢占式调度 |
| 切换开销 | ~微秒级（纯 Python 对象） | ~毫秒级（内核上下文切换） |
| 并发能力 | 轻松支撑数万并发 | 受 GIL 限制，通常数百 |
| CPU 密集型 | 不友好（阻塞事件循环） | 可用多核 |
| 适用场景 | I/O 密集型（HTTP、DB、Redis） | CPU 密集型（计算、加密） |

### 1.3 事件循环的底层调度

事件循环内部使用 `selector` 模块监听 I/O 就绪事件。当调用 `loop.add_reader()` 注册一个 socket 描述符时，内核会在该 socket 可读/可写时通知事件循环，事件循环再调度对应协程继续执行。这就是为什么 asyncio 能够用单线程高效处理大量并发 I/O 连接——整个过程不需要线程阻塞。

---

## 2. FastAPI 性能优势

### 2.1 技术栈分层

FastAPI 的高性能来自其精心选择的技术栈：

```
FastAPI（声明式 API 层）
    ↓ 调用
Starlette（ASGI 微框架，路由/中间件/模板）
    ↓ 挂载
Uvicorn（ASGI 服务器，单进程事件循环）
    ↓ 可选替换
Gunicorn + Uvicorn Worker（多进程 + 异步）
```

FastAPI 本身不运行服务器，它定义了 ASGI（Asynchronous Server Gateway Interface）规范中的接口。Uvicorn 是默认的 ASGI 服务器实现，内部运行一个 asyncio 事件循环。Gunicorn 通过 pre-fork 模式管理多个 Uvicorn Worker 进程，每个 Worker 都是一个独立的事件循环，可以绑定到不同的 CPU 核心。

### 2.2 Pydantic v2 序列化优化

Pydantic v2（2.0+）相比 v1 有重大性能提升，核心原因在于重写了序列化路径：

```python
from pydantic import BaseModel, Field
from datetime import datetime

class User(BaseModel):
    id: int
    name: str = Field(min_length=1, max_length=100)
    created_at: datetime

    class Config:
        populate_by_name = True  # v2 用 model_config 替代

# v2 高性能序列化：使用 model_dump_json 而非 json()
user = User(id=1, name="Alice", created_at=datetime.now())
json_str = user.model_dump_json()        # v2 新 API
data = user.model_dump(mode="python")    # 字典格式
```

v2 引入了 `兰州`（Rust）加速的验证器编译层 `pydantic-core`（基于 pyright），验证速度比 v1 提升数十倍。同时 `model_dump_json` 使用 `orjson` 进行序列化，比标准库 `json` 快 2-3 倍。

---

## 3. Pydantic v2 深度掌握

### 3.1 Field 与数据验证

```python
from pydantic import BaseModel, Field, field_validator
from typing import Optional
import re

class Order(BaseModel):
    order_id: str = Field(pattern=r"^ORD-\d{8}$")  # 正则验证
    amount: float = Field(gt=0, le=1_000_000)     # 数值范围约束
    email: Optional[str] = None
    tags: list[str] = Field(default_factory=list)  # v2 list[str] 而非 List[str]

    # v2 推荐写法：field_validator
    @field_validator("order_id")
    @classmethod
    def validate_order_id(cls, v: str) -> str:
        if not re.match(r"^ORD-\d{8}$", v):
            raise ValueError("订单号格式必须为 ORD-YYYYMMDD")
        return v.upper()

    # 支持同时验证多个字段
    @field_validator("amount")
    @classmethod
    def validate_amount(cls, v: float, info) -> float:
        # info.name 可访问当前字段名
        if v == 0:
            raise ValueError("金额不能为零")
        return round(v, 2)
```

**v1 vs v2 关键区别**：v1 的 `@validator` 在 v2 中被拆分为 `@field_validator`（仅验证单个字段）和 `@model_validator`（验证整个模型）。v2 不再使用 `class Config`，改为 `model_config = ConfigDict(...)` 或直接以类属性方式配置。

### 3.2 序列化与反序列化

```python
from pydantic import SecretStr

class APICredential(BaseModel):
    api_key: SecretStr  # 打印时显示 *****，不暴露真实值
    webhook_url: HttpUrl  # 自动验证 URL 格式

# 反序列化时自动处理
credential = APICredential.model_validate({
    "api_key": "sk-abc123",
    "webhook_url": "https://example.com/callback"
})
```

---

## 4. 依赖注入

### 4.1 FastAPI Depends 机制

FastAPI 的依赖注入系统是其最重要的特性之一，它不依赖任何外部 DI 容器库，而是通过函数和 `Depends()` 实现：

```python
from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from typing import Generator

security = HTTPBearer()

# 同步依赖：每次请求都会调用
def get_db():
    db = DBSession()
    try:
        yield db
    finally:
        db.close()

# 异步依赖：推荐用于 I/O 操作
async def verify_token(
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> str:
    token = credentials.credentials
    payload = await decode_jwt(token)
    if payload is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token 无效或已过期"
        )
    return payload["sub"]

# 全局依赖（所有路由生效）
app = FastAPI(dependencies=[Depends(verify_token)])

# 路由级依赖（仅当前路由生效）
@app.get("/admin/users", dependencies=[Depends(verify_token)])
async def list_users():
    return ["alice", "bob"]

# 带参数的依赖（工厂模式）
def verify_permission(required_scope: str):
    async def checker(token_payload: str = Depends(verify_token)) -> bool:
        scopes = await get_user_scopes(token_payload)
        if required_scope not in scopes:
            raise HTTPException(403, "权限不足")
        return True
    return checker

@app.delete("/articles/{id}", dependencies=[Depends(verify_permission("editor"))])
async def delete_article(id: int):
    ...
```

依赖注入在 FastAPI 内部通过一个缓存机制实现：同一个依赖在同一次请求中只会被调用一次（即使被多个参数引用），这避免了重复计算和数据库查询。

---

## 5. 异步编程

### 5.1 async def vs def

```python
from fastapi import FastAPI
import asyncio

app = FastAPI()

# 异步路由：可并发处理多个请求，await 协程
@app.get("/async")
async def async_handler():
    result = await fetch_from_api()       # 异步 I/O，正确用法
    return result

# 同步路由：会阻塞事件循环，应仅用于纯 CPU 计算
@app.get("/sync")
def sync_handler():
    result = compute_heavy_task()         # CPU 密集，同步执行
    return result

# CPU 密集型任务：必须使用 run_in_executor 放到线程池
@app.get("/heavy")
async def heavy_handler():
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(None, cpu_bound_function, arg)
    return result

# Python 3.9+ 更简洁的写法
@app.get("/heavy-39")
async def heavy_handler_39():
    result = await asyncio.to_thread(cpu_bound_function, arg)
    return result
```

### 5.2 何时用异步

核心原则：所有等待外部资源的操作都应该用 `async`，包括 HTTP 请求、数据库查询、Redis 操作、文件读写。FastAPI 中一个常见的错误是在 `async def` 函数中调用同步阻塞库（如 `requests` 而非 `httpx`、`psycopg2` 而非 `asyncpg`），这会阻塞整个事件循环。

---

## 6. APScheduler 调度

### 6.1 两种调度器对比

```python
from apscheduler.schedulers.blocking import BlockingScheduler
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger
from apscheduler.triggers.interval import IntervalTrigger

# 阻塞式：适合独立脚本或单进程服务
scheduler = BlockingScheduler()
scheduler.add_job(
    run_etl_job,
    CronTrigger(hour=2, minute=0),  # 每天凌晨 2 点
    id="daily_etl",
    max_instances=1,  # 防止上一次未执行完就启动下一次
    coalesce=True,   # 堆积的执行合并为一次
    misfire_grace_time=3600  # 错过后 1 小时内仍可执行
)

# 异步式：与 FastAPI/uvicorn 集成
scheduler = AsyncIOScheduler()
scheduler.add_job(
    async_etl_job,
    IntervalTrigger(hours=6),
    id="periodic_sync"
)
```

### 6.2 多实例部署防重复执行

在 Kubernetes 或多进程部署中，多个实例可能同时触发同一个定时任务。解决方案：

```python
# 方案一：分布式锁（Redis）
from apscheduler.executors.asyncio import AsyncIOExecutor
from apscheduler.events import EVENT_JOB_ERROR, EVENT_JOB_EXECUTED

async def job_with_lock(job_id: str):
    lock = await redis.setnx(f"lock:job:{job_id}", "1", nx=True, ex=300)
    if not lock:
        logger.warning(f"Job {job_id} 被其他实例锁定，跳过执行")
        return
    try:
        await run_task()
    finally:
        await redis.delete(f"lock:job:{job_id}")

# 方案二：使用 Scheduler 的 singleton 个实例
scheduler.add_job(job_fn, trigger="cron", id="unique_job", instances=1)
# 注意：这仅在单进程内的多调度器实例间有效
```

---

## 7. 并发 vs 并行

### 7.1 GIL 的影响

Python 的 GIL（Global Interpreter Lock）限制了同一时刻只有一个线程执行 Python 字节码。这对不同类型的任务影响截然不同：

```python
# GIL 影响的演示
import threading, multiprocessing, asyncio, time

# CPU 密集型：多进程才能利用多核
def cpu_task(n):
    return sum(i * i for i in range(n))

# 多进程：绕过 GIL，真正并行
if __name__ == "__main__":
    start = time.time()
    with multiprocessing.Pool(4) as pool:
        results = pool.map(cpu_task, [10_000_000] * 4)
    print(f"多进程耗时: {time.time() - start:.2f}s")  # ~2s（利用 4 核）

    # 多线程：对 CPU 密集型几乎无效
    start = time.time()
    threads = [threading.Thread(target=cpu_task, args=(10_000_000,)) for _ in range(4)]
    [t.start() for t in threads]
    [t.join() for t in threads]
    print(f"多线程耗时: {time.time() - start:.2f}s")  # ~8s（GIL 导致串行）
```

### 7.2 三者对比总结

| 模式 | 切换单位 | 多核利用 | 适用场景 |
|------|---------|---------|---------|
| asyncio | 协程（yield） | 否 | 高并发 I/O |
| threading | OS 线程（抢占式） | 部分（I/O 期间可切换） | 阻塞 I/O、GUI |
| multiprocessing | 进程 | 完全 | CPU 密集计算 |

---

## 8. 装饰器原理

### 8.1 基础装饰器

```python
import functools
import time
from typing import Callable

# 基础装饰器
def timer(func: Callable) -> Callable:
    @functools.wraps(func)  # 保留原函数元数据
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} 耗时 {elapsed:.2f}s")
        return result
    return wrapper

# 带参数的装饰器（需要三层嵌套）
def retry(max_attempts: int = 3, delay: float = 1.0):
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_exception = e
                    if attempt < max_attempts - 1:
                        time.sleep(delay)
            raise last_exception
        return wrapper
    return decorator

@retry(max_attempts=3, delay=0.5)
def call_external_api():
    ...
```

### 8.2 装饰器与 FastAPI 结合

```python
from fastapi import HTTPException
import functools

def require_role(role: str):
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            # FastAPI 的 Depends 机制更适合依赖注入
            # 装饰器更多用于横切关注点（日志、性能、缓存）
            user = kwargs.get("current_user")
            if user.role != role:
                raise HTTPException(403, f"需要 {role} 角色")
            return await func(*args, **kwargs)
        return wrapper
    return decorator
```

---

## 9. Flask vs Django vs FastAPI

### 9.1 核心对比

| 维度 | Flask | Django | FastAPI |
|------|-------|--------|---------|
| 架构哲学 | 微框架，可自由组合 | 全功能框架，约定优于配置 | 现代 API 优先，类型驱动 |
| 性能（QPS） | ~500-1000 | ~300-800 | ~5000-20000+ |
| 并发模型 | 同步 WSGI | 同步 WSGI | ASGI 异步 |
| ORM | SQLAlchemy（手动集成） | Django ORM（内置） | SQLModel / SQLAlchemy（可选） |
| 自动文档 | 无 | Admin 界面 | OpenAPI/Swagger（内置） |
| 类型安全 | 无 | 无 | Pydantic 强类型 |
| 学习曲线 | 低 | 高 | 中低 |

### 9.2 为什么 FastAPI 性能最好

1. **ASGI 异步架构**：事件循环驱动的请求处理，单线程可处理数千并发
2. **Pydantic v2 高效序列化**：Rust 加速的验证和 JSON 序列化
3. **类型推断生成 OpenAPI**：零额外成本生成完整 API 文档
4. **依赖注入轻量化**：无需外部容器，原生 Python 实现

---

## 10. 类型提示

### 10.1 常用类型与运行时有效性

```python
from typing import List, Optional, Union, Callable, Any, TypeVar
from collections.abc import Awaitable
from pydantic import BaseModel

T = TypeVar("T")

# Optional[str] == Union[str, None]
def greet(name: Optional[str]) -> str:
    return f"Hello, {name or 'World'}"

# Callable[[输入类型], 返回类型]
def map_async(
    func: Callable[[int], Awaitable[int]],
    items: list[int]
) -> list[int]:
    return [await func(x) for x in items]

# Pydantic 中的类型推导
class Config(BaseModel):
    # List[str] 在 Pydantic 中自动推导为 list[str]
    allowed_origins: list[str] = []
    # Union 类型在反序列化时按声明顺序尝试匹配
    cache_ttl: Union[int, float, None] = 3600
    # Callable 在 schema 中显示为 "void"
    validator: Optional[Callable[[Any], Any]] = None
```

**关键点**：Python 类型提示在运行时默认不做检查（Python 3.11+ 可用 ` PEP 695` 新语法）。它们的主要价值在于静态分析（mypy、pyright）和 Pydantic 等库的自动推导。

---

## 11. 内存管理与 GC

### 11.1 引用计数与循环引用

Python 使用引用计数作为主要内存管理机制：

```python
import sys

a = [1, 2, 3]      # 引用计数 = 1
b = a              # 引用计数 = 2
del a              # 引用计数 = 1（未立即释放）
del b              # 引用计数 = 0 → 立即释放

# 循环引用示例
class Node:
    def __init__(self, val):
        self.val = val
        self.next = None

n1 = Node(1)
n2 = Node(2)
n1.next = n2   # n1 -> n2
n2.next = n1   # n2 -> n1，形成循环引用
# 引用计数均为 1，无法被引用计数机制释放
# 需要 GC（垃圾回收器）定期检测并清除循环

import gc
gc.collect()  # 手动触发 GC
```

### 11.2 常见内存泄漏场景

```python
# 场景一：全局列表无限增长
results = []  # 生产环境中持续 append 而不清理
def on_request(data):
    results.append(process(data))  # 泄漏：列表无限增长

# 场景二：缓存未设置上限
from functools import lru_cache

@lru_cache(maxsize=None)  # 无上限缓存会持续占用内存
def expensive_computation(x):
    ...

# 场景三：未关闭数据库连接或文件句柄
def read_files():
    handles = []
    for path in huge_file_list:
        handles.append(open(path))  # 句柄泄漏
    # 正确做法：使用上下文管理器
    # with open(path) as f: ...

# 场景四：全局字典存储状态且不清理
session_store: dict[str, Any] = {}  # 内存泄漏：过期 session 未删除
```

生产环境建议使用 `weakref` 库处理需要引用对象但不希望阻止 GC 的场景，或使用有容量限制的 LRU Cache。

---

## 面试问答

**Q1：FastAPI 中 `async def` 和普通 `def` 在性能上有何区别？**

A：`def` 路由在 Uvicorn 的线程池中执行，不阻塞事件循环，但无法利用 await 异步调用。`async def` 在事件循环中执行，可以直接 `await` 异步库，性能更高（无线程切换开销），但不能在其中直接调用同步阻塞库。

**Q2：Pydantic v2 中 `model_validate` 和 `model_validate_json` 的区别？**

A：`model_validate` 接收字典或对象作为输入，会进行字段验证和类型转换。`model_validate_json` 接收 JSON 字符串，先解析为字典再调用 `model_validate`，适合处理 HTTP 请求体。

**Q3：asyncio.gather vs asyncio.create_task 的区别？**

A：`gather` 并发执行多个协程并等待全部完成，返回结果列表，适合结果需要汇总的场景。`create_task` 将协程调度到事件循环后立即返回 Task 对象，不阻塞调用者，适合需要精细控制任务生命周期的场景（如取消、超时控制）。

**Q4：FastAPI 如何实现请求级别的数据库连接管理？**

A：通过 `Depends` 注入生成器函数，每次请求创建一个新的数据库连接（`yield` 之前），请求结束后自动关闭连接（`yield` 之后）。FastAPI 保证同一个请求中多次 `Depends` 同一个依赖时只调用一次（缓存机制）。

**Q5：描述 APScheduler 在多进程部署中的注意事项。**

A：APScheduler 默认只管理单进程内的调度。多进程部署时每个进程都有独立的调度器实例，可能导致任务被重复执行。解决方案是使用分布式锁（Redis/SQL）确保只有一个实例执行任务，或使用外部调度器（Celery Beat、Django-Q）统一管理。

**Q6：GIL 如何影响 FastAPI 的并发性能？**

A：GIL 对 FastAPI 的影响很小，因为 FastAPI 主要处理 I/O 密集型任务（HTTP 请求、数据库查询），I/O 操作会释放 GIL。真正需要 CPU 并行时，应使用 `asyncio.to_thread`（I/O 密集）或 `multiprocessing`（CPU 密集）。FastAPI 的高并发来自异步 I/O，而非绕过 GIL。

---

*Last Updated: 2026-04-09*
