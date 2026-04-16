# 四、Python / FastAPI · 答案

> 本章共 30 题，覆盖 Python 基础（Q1-Q9）、Python 进阶（Q10-Q18）、FastAPI（Q19-Q23）、Python 异步编程（Q24-Q30）

---

## 面试官快速索引

| 题号 | 核心考察点 | 难度 | 追问深度 |
|------|-----------|------|---------|
| Q1 | GIL、CPU 密集 vs I/O 密集 | ⭐⭐⭐ | 可深问 GIL 原理 |
| Q2 | 装饰器原理、带参装饰器 | ⭐⭐⭐ | 可深问 functools.wraps |
| Q3 | __init__ vs __new__、单例实现 | ⭐⭐ | 可深问元类实现单例 |
| Q4 | 生成器 vs 迭代器、yield | ⭐⭐⭐ | 可深问生成器表达式 |
| Q5 | 类型注解、Annotated 用途 | ⭐⭐ | 可深问 Pydantic 验证 |
| Q6 | Python 基本数据类型 | ⭐ | 可深问可变 vs 不可变 |
| Q7 | list/tuple/dict/set 对比 | ⭐⭐ | 可深问时间复杂度 |
| Q8 | 字符串格式化 f-string/format/% | ⭐⭐ | 可深问 format spec |
| Q9 | 深拷贝 vs 浅拷贝 | ⭐⭐ | 可深问引用陷阱 |
| Q10 | 异常处理、自定义异常 | ⭐⭐ | 可深问 except 顺序 |
| Q11 | 上下文管理器、@contextmanager | ⭐⭐⭐ | 可深问 __aenter__ |
| Q12 | 装饰器完整写法、functools.wraps | ⭐⭐⭐ | 可深问类装饰器 |
| Q13 | *args/**kwargs 用法 | ⭐⭐ | 可深问解包 |
| Q14 | 魔术方法 | ⭐⭐ | 可深问 dataclasses |
| Q15 | MRO、C3 线性化、super | ⭐⭐⭐⭐ | 可深问钻石继承 |
| Q16 | 垃圾回收（引用计数+标记清除+分代） | ⭐⭐⭐ | 可深问循环引用处理 |
| Q17 | unittest vs pytest、fixture | ⭐⭐⭐ | 可深问 mock/patch |
| Q18 | 标准库核心用法 | ⭐⭐ | 可深问 itertools 进阶 |
| Q19 | FastAPI 基本用法、Swagger | ⭐⭐ | 可深问路由注册顺序 |
| Q20 | ASGI vs WSGI、async/await | ⭐⭐⭐ | 可深问 uvicorn 原理 |
| Q21 | 依赖注入、Depends | ⭐⭐⭐ | 可深问缓存依赖 |
| Q22 | Pydantic BaseModel、验证、Settings | ⭐⭐⭐ | 可深问 validator |
| Q23 | 接口版本管理、向后兼容 | ⭐⭐ | 可深问 schema 演进 |
| Q24 | async def vs def、事件循环调度 | ⭐⭐⭐ | 可深问 uvloop |
| Q25 | gather vs create_task vs wait | ⭐⭐⭐ | 可深问 TaskGroup |
| Q26 | async def vs def 在 FastAPI 中的选择 | ⭐⭐⭐ | 可深问 run_in_executor |
| Q27 | Coroutine vs Future vs Task | ⭐⭐⭐⭐ | 可深问 await 原理 |
| Q28 | BackgroundTasks vs APScheduler | ⭐⭐ | 可深问任务持久化 |
| Q29 | run_in_executor、同步库异步调用 | ⭐⭐⭐ | 可深问 ThreadPoolExecutor |
| Q30 | asyncio.timeout vs wait_for | ⭐⭐⭐ | 可深问 shield |

---

## 4.1 Python 基础

### Q1：Python 的 GIL 是什么？它对多线程有什么影响？在 CPU 密集型和 I/O 密集型场景下分别应该选择多线程还是多进程？

**考察点**：GIL 原理、线程 vs 进程、性能选择

**答案要点**：
- **GIL（Global Interpreter Lock）**：
  - CPython 的一个互斥锁，保证同一时刻只有一个线程执行 Python 字节码
  - 原因：CPython 的内存管理（非线程安全引用计数）需要 GIL 保护，防止同一对象的引用计数同时被两个线程修改导致内存泄漏
- **GIL 对多线程的影响**：
  - **CPU 密集型**：多线程无法真正并行，只能串行执行，GIL 严重限制了 CPU 利用率，CPU 密集型多线程**性能甚至不如单线程**
  - **I/O 密集型**：线程等待 I/O 时（网络/文件），会主动释放 GIL，其他线程可以获得 GIL 并执行。因此 I/O 密集型多线程**仍然有效**
- **场景选择**：
  - **CPU 密集型**：多进程（`multiprocessing` / `concurrent.futures.ProcessPoolExecutor`），每个进程有独立的 Python 解释器和 GIL
  - **I/O 密集型**：多线程（`threading` / `asyncio`），或 asyncio 单线程协程模型
  - **混合型**：多进程 + 每个进程内 asyncio（Uvicorn Worker 模式）

**可能的追问**：
- GIL 是什么时候加入 CPython 的？社区有没有尝试移除 GIL？
- `concurrent.futures` 中的 `ThreadPoolExecutor` 和 `ProcessPoolExecutor` 如何选择？
- PyPy 和 Cython 有 GIL 吗？Jython/IronPython 呢？

---

### Q2：Python 的装饰器原理是什么？写一个带参数的装饰器，用于记录函数执行时间。

**考察点**：装饰器原理、闭包、函数参数化

**答案要点**：
- **装饰器原理**：装饰器是高阶函数，接受被装饰函数作为参数，返回一个新函数替换原函数。本质是闭包：`outer(func) -> inner(*args) -> modified_result`
- **带参装饰器（两层闭包）**：
  ```python
  import functools
  import time

  def log_execution_time(level='INFO'):
      """带参数的装饰器工厂"""
      def decorator(func):
          @functools.wraps(func)  # 保留原函数元信息
          def wrapper(*args, **kwargs):
              start = time.perf_counter()
              result = func(*args, **kwargs)
              elapsed = time.perf_counter() - start
              print(f"[{level}] {func.__name__} took {elapsed:.4f}s")
              return result
          return wrapper
      return decorator

  # 使用
  @log_execution_time(level='DEBUG')
  def fetch_data(url):
      return requests.get(url)
  ```
- **执行流程**：`@log_execution_time('DEBUG')` → 先调用 `log_execution_time('DEBUG')` 返回 `decorator` → 再调用 `decorator(func)` → 返回 `wrapper`
- **`functools.wraps`**：将原函数 `func` 的 `__name__`、`__doc__`、`__annotations__` 等元信息拷贝到 `wrapper`，避免装饰后函数名变成 `wrapper`（对调试和反射非常重要）

**可能的追问**：
- `@functools.lru_cache` 装饰器的原理是什么？如何实现缓存失效？
- 类装饰器和函数装饰器有什么区别？
- 如何用装饰器实现函数重试逻辑（retry）？

---

### Q3：Python 中 `__init__` 和 `__new__` 的区别？你在 RedisStreamAPI 中用 `__new__` 实现单例模式的原理是什么？

**考察点**：对象创建过程、单例模式、new vs init

**答案要点**：
- **`__new__` vs `__init__`**：
  - `__new__`：创建实例，返回实例对象本身。在 `__init__` 之前调用
  - `__init__`：初始化实例，在 `__new__` 之后调用，修改实例属性
  - 默认情况下，`object.__new__` 负责分配内存，`__init__` 负责填充属性
- **单例模式实现**（`__new__`）：
  ```python
  class RedisStreamAPI:
      _instance = None

      def __new__(cls, *args, **kwargs):
          if cls._instance is None:
              cls._instance = super().__new__(cls)
              cls._instance._initialized = False
          return cls._instance

      def __init__(self):
          if self._initialized:
              return
          self._initialized = True
          # 初始化 Redis 连接
          self.redis = redis.Redis(...)
          self.group = 'soc-consumers'
  ```
- **原理**：`__new__` 每次实例化时先被调用，通过类变量 `_instance` 判断是否已存在实例。存在则直接返回已有实例，不走 `__init__`，否则创建新实例

**可能的追问**：
- 为什么 `__new__` 必须是静态方法？（创建对象时实例尚未存在，没有 self）
- 单例模式在多线程环境下的线程安全问题如何解决？（加锁 + 双重检查锁定）
- `__del__` 什么时候会被调用？它的调用顺序是什么？

---

### Q4：Python 的生成器和迭代器的区别？yield 关键字的作用？在处理大量安全日志时，生成器有什么优势？

**考察点**：迭代器协议、生成器函数 vs 生成器对象

**答案要点**：
- **迭代器**：实现了 `__iter__` 和 `__next__` 方法的对象，调用 `next()` 逐个获取元素，遍历完抛出 `StopIteration`
- **生成器**：是迭代器的一种，通过**生成器函数**（含 `yield`）或**生成器表达式**创建
  - 普通函数执行到 `return` 结束，生成器函数执行到 `yield` 暂停，将值产出，保留局部状态
  - 下次调用 `next()` 从 `yield` 断点继续执行
- **yield vs return**：
  - `return`：函数终止，返回值，状态丢失
  - `yield`：函数暂停，产出值，保留局部变量状态
  - 生成器是**惰性计算**，不一次性加载所有数据
- **处理大量日志的优势**：
  - **内存效率**：1 亿条日志不需要全量读入内存，每次只加载一条/一批，内存占用恒定
  - **时间效率**：边读边处理，不需要等全部加载完成
  ```python
  def read_logs(path):
      with open(path, 'r') as f:
          for line in f:
              yield json.loads(line)  # 惰性加载

  # 使用
  for log in read_logs('access.log'):  # 一次只处理一行
      process(log)
  ```

**可能的追问**：
- 什么是 `yield from`？它在 Python 3 中的作用是什么？
- 生成器可以迭代多次吗？（不可以，只能迭代一次）
- `itertools` 中有哪些常用的生成器函数？（islice、takewhile、chain、groupby）

---

### Q5：Python 的类型注解（Type Hints）有什么用？`Annotated[str, "description"]` 在 FastAPI 和 LangChain 中分别起什么作用？

**考察点**：类型注解用途、Annotated 语法

**答案要点**：
- **类型注解用途**：
  1. **静态检查**：配合 `mypy`、`pylint`，在运行前发现类型错误
  2. **IDE 提示**：PyCharm/VSCode 根据类型注解提供自动补全
  3. **文档作用**：代码即文档，函数签名即接口说明
  4. **运行时验证**：配合 Pydantic / FastAPI 做运行时校验
- **`Annotated`**：
  - 语法：`Annotated[T, metadata]`，T 是类型，metadata 是元数据
  - 不改变 T 的类型，只添加额外信息
  - **FastAPI**：用于定义请求参数校验规则，如 `Annotated[str, Field(min_length=1)]`
    ```python
    from fastapi import FastAPI, Query
    app = FastAPI()

    @app.get('/search')
    def search(q: Annotated[str, Query(min_length=3, max_length=50)]):
        return {'query': q}
    ```
  - **LangChain**：用于定义 Tool 的描述，让 LLM 理解工具用途
    ```python
    from langchain_core.tools import tool

    @tool
    def search_threat_intel(query: Annotated[str, "The IP or domain to lookup"]) -> str:
        ...
    ```

**可能的追问**：
- `Optional[str]` 和 `str | None` 有什么区别？
- 什么是 `TypeVar`？什么时候需要自定义泛型？
- Pydantic 中 `model_validate` 和 `parse_obj` 有什么区别？

---

### Q6：Python 的基本数据类型有哪些？`int`、`float`、`str`、`bool`、`None` 的特性和常见操作？

**考察点**：Python 内置类型、可变 vs 不可变

**答案要点**：
- **不可变类型**（值改变时创建新对象，id 变化）：
  - `int`：任意精度整数（大整数无溢出），除法 `/` 返回 float，`//` 整除
  - `float`：双精度浮点，有精度问题（`0.1 + 0.2 != 0.3`），金融计算用 `Decimal`
  - `str`：Unicode 字符串，不可变，支持切片/拼接/格式化，`'abc'[0] = 'x'` 报错
  - `bool`：True/False，是 `int` 的子类（True=1, False=0）
  - `None`：单例对象，代表"空"，类型是 `NoneType`
  - `tuple`、`frozenset`、`bytes`
- **可变类型**：`list`、`dict`、`set`、`bytearray`
- **常见操作**：
  - 字符串：`split`、`strip`、`replace`、`format`、`f"hello {name}"`
  - 列表：`append`、`pop`、`切片`、`列表推导式 [x for x in range(10)]`
  - 字典：`get`、`items`、`keys`、`values`、`d.get('key', 'default')`

**可能的追问**：
- Python 的 int 是任意精度，如何实现的？（动态分配内存，C 层面的变长结构）
- `==` 和 `is` 的区别是什么？Python 小整数池是什么？
- `bool` 为什么是 `int` 的子类？`True + True + True` 等于多少？

---

### Q7：`list`、`tuple`、`dict`、`set` 的区别？增删改查的时间复杂度？什么场景下选择哪种？

**考察点**：数据结构时间复杂度、场景选择

**答案要点**：
| 操作 | list | tuple | dict | set |
|------|------|-------|------|-----|
| 查找 | O(n) | O(n) | O(1) 平均 | O(1) 平均 |
| 插入/删除 | O(1) 尾插，O(n) 头插/中间 | 不支持（不可变） | O(1) 平均 | O(1) 平均 |
| 切片 | ✅ | ✅ | ❌ | ❌ |
| 可变 | ✅ | ❌ | ✅ | ✅ |
| 元素唯一性 | ❌ | ❌ | ✅ key 唯一 | ✅ 元素唯一 |
- **选择场景**：
  - **list**：有序序列、需按索引访问、需要切片、需要头尾操作（`deque` 更优）
  - **tuple**：不可变数据（函数返回值、字典 key）、固定长度记录
  - **dict**：键值映射、快速查找、缓存、计数（`Counter`）
  - **set**：去重、交集/并集计算、成员判断（如 IP 白名单）

**可能的追问**：
- dict 的 O(1) 查找在什么情况下会退化到 O(n)？（哈希冲突过多，Python 用开放寻址解决）
- `collections.deque` 和 `list` 在 append/pop 操作上有什么区别？
- `set` 的交集和并集操作的时间复杂度是什么？

---

### Q8：Python 的字符串格式化有哪些方式？`f-string`、`format()`、`%` 格式化的区别和使用场景？

**考察点**：字符串格式化语法、format spec

**答案要点**：
- **三种方式对比**：
  | 方式 | 语法 | 优点 | 缺点 |
  |------|------|------|------|
  | `%` 格式化 | `'Hello %s, you have %d messages' % (name, count)` | 兼容性好（C 风格） | 啰嗦、可读性差 |
  | `str.format()` | `'Hello {name}, you have {count} messages'.format(name=name, count=count)` | 支持索引/命名占位、格式化规格 | 稍啰嗦 |
  | **f-string** | `f'Hello {name}, you have {count} messages'` | 最简洁、可嵌入表达式 | Python 3.6+，字符串前缀易漏 |
- **f-string 进阶**：
  - `f'{value:.2f}'`：保留两位小数
  - `f'{obj.attr!r}'`：用 repr 格式化
  - `f'{func()}'`：可嵌入函数调用
  - `f'{name.upper()}'`：可调用方法
- **选择**：Python 3.6+ 推荐 f-string，兼顾简洁和功能

**可能的追问**：
- `format()` 的格式规格符（format spec）中 `:>10`、`:,.2f`、`:b` 分别表示什么？
- f-string 中如何处理转义大括号 `{{` 和 `}}`？
- 为什么不能用 f-string 动态格式化日志级别？（字符串拼接即可，不需要 f-string）

---

### Q9：Python 的深拷贝和浅拷贝是什么？`copy.copy()` 和 `copy.deepcopy()` 的区别？在处理嵌套对象时为什么重要？

**考察点**：拷贝机制、引用陷阱

**答案要点**：
- **浅拷贝**：只拷贝对象本身，**不递归拷贝**嵌套对象。嵌套的子对象仍然是引用
- **深拷贝**：递归拷贝所有层级的对象，完全独立
- **拷贝方式**：
  - 浅拷贝：`copy.copy(obj)`、`obj.copy()`（list）、`list(obj)`、`[:]`（列表切片）、`dict(obj)`
  - 深拷贝：`copy.deepcopy(obj)`
- **嵌套对象陷阱示例**：
  ```python
  a = [[1, 2], [3, 4]]
  b = a.copy()        # 浅拷贝
  b[0][0] = 99        # 改了 a！
  # a == [[99, 2], [3, 4]] ← 被意外修改
  ```
- **场景**：深拷贝用于需要完全独立副本的场景（如撤销/重做、函数参数传入后不应影响原对象、JSON 反序列化后独立操作）

**可能的追问**：
- 什么是循环引用的深拷贝？（`copy.deepcopy` 用字典记录已拷贝对象避免无限递归）
- `copy.copy` 对不可变对象（tuple）的行为是什么？（直接返回同一个对象，因为不可变对象无需拷贝）
- Pydantic 的 `.model_copy(deep=True)` 和 `copy.deepcopy` 有什么区别？

---

## 4.2 Python 进阶

### Q10：Python 的异常处理（try/except/finally/raise）机制是什么？自定义异常怎么写？什么场景用异常而不是返回值？

**考察点**：异常机制、自定义异常、异常设计

**答案要点**：
- **try/except/finally 流程**：
  ```python
  try:
      result = risky_operation()
  except ValueError as e:
      print(f'ValueError: {e}')
  except (TypeError, KeyError):
      print('Type or Key error')
  except Exception as e:
      print(f'Unexpected: {e}')
      raise  # 重新抛出
  else:
      print('No exception')  # try 块成功执行时执行
  finally:
      cleanup()  # 无论是否异常，都执行
  ```
- **自定义异常**：
  ```python
  class SOCAlertError(Exception):
      """SOC 告警相关异常基类"""
      def __init__(self, message: str, alert_id: str = None):
          super().__init__(message)
          self.alert_id = alert_id

  class AlertNotFoundError(SOCAlertError):
      """告警不存在"""
      pass
  ```
- **用异常 vs 返回值**：
  - **用异常**：异常代表"不应该发生的错误"、调用者很可能忘记检查返回值、级联错误传播
  - **用返回值**：正常业务流程分支（如查找失败返回 None、文件不存在返回空列表）、性能敏感路径（异常构造开销大）

**可能的追问**：
- `except Exception` 和 `except BaseException` 有什么区别？`except Exception` 会漏掉什么？
- 为什么不要用 `except Exception as e: pass` 静默吞掉异常？
- `try/except/else/finally` 各执行时机是什么？

---

### Q11：Python 的上下文管理器（`with` 语句和 `__enter__`/`__exit__`）有什么用？`@contextmanager` 装饰器如何简化实现？

**考察点**：上下文管理器、上下文协议、生成器实现

**答案要点**：
- **作用**：确保资源获取和释放配对执行，即使中间发生异常，也能正确释放资源（类似 RAII）
- **实现协议**：
  ```python
  class FileManager:
      def __enter__(self):
          self.file = open('data.txt', 'r')
          return self.file
      def __exit__(self, exc_type, exc_val, exc_tb):
          self.file.close()
          return False  # 不吞异常

  with FileManager() as f:
      data = f.read()
  # 自动 close()
  ```
- **`@contextmanager` 简化写法**（基于生成器）：
  ```python
  from contextlib import contextmanager

  @contextmanager
  def managed_resource():
      resource = acquire()
      try:
          yield resource
      finally:
          release(resource)

  with managed_resource() as r:
      r.use()
  # 异常时 finally 仍执行
  ```
- **典型应用**：文件操作、数据库事务（commit/rollback）、锁（acquire/release）、计时器

**可能的追问**：
- `__exit__` 返回 `True` 和 `False` 的区别？（True=吞掉异常，继续执行；False/None=重新抛出）
- 什么是 `contextlib.closing`？什么时候用它？
- async with 需要实现什么协议？（`__aenter__`/`__aexit__`）

---

### Q12：Python 的装饰器（不带参数 / 带参数 / 保留元信息）完整写法？`functools.wraps` 的作用？

**考察点**：装饰器分层实现、元信息保留

**答案要点**：
- **三层装饰器实现**：
  ```python
  import functools

  # 1. 最简装饰器（无参数）
  def log_calls(func):
      @functools.wraps(func)
      def wrapper(*args, **kwargs):
          print(f'Calling {func.__name__}')
          return func(*args, **kwargs)
      return wrapper

  # 2. 带参数装饰器（多一层闭包）
  def log_calls(level='INFO'):
      def decorator(func):
          @functools.wraps(func)
          def wrapper(*args, **kwargs):
              print(f'[{level}] Calling {func.__name__}')
              return func(*args, **kwargs)
          return wrapper
      return decorator

  # 3. 保留元信息的关键
  @functools.wraps(func)
  # 将 func 的 __name__、__doc__、__annotations__、__module__ 拷贝到 wrapper
  ```
- **`functools.wraps` 保留的信息**：使 `wrapper.__name__ == func.__name__`，`inspect.signature(wrapper)` 能正确工作，对 `mypy`、`pylint`、调试工具和文档生成器至关重要

**可能的追问**：
- 装饰器叠加时执行顺序是怎样的？（`@a` → `func` → `wrapper_b` → `wrapper_a`）
- 类装饰器如何实现？
- 如何让装饰器接受可选参数（如 `@log(level)` 或 `@log` 不带参数）？

---

### Q13：Python 的 `*args` 和 `**kwargs` 是什么？它们在函数签名中如何配合使用？

**考察点**：可变参数、解包、签名设计

**答案要点**：
- **`*args`**：将多余的位置参数收集为**元组**（tuple）
- **`**kwargs`**：将多余的关键字参数收集为**字典**
- **签名设计**：
  ```python
  def flex_func(a, b, *args, c=None, **kwargs):
      # a, b = 必需参数
      # args = 额外的位置参数元组
      # c = 关键字必需参数
      # kwargs = 额外的关键字参数字典
      pass
  ```
- **解包调用**：
  ```python
  def process(url, timeout=5, headers=None):
      pass

  # 调用时解包
  options = {'timeout': 10, 'headers': {'Auth': 'Bearer xxx'}}
  process('https://api.example.com', **options)
  ```
- **典型应用**：装饰器转发参数、`functools.partial` 偏函数、`Logger` 的通用日志接口

**可能的追问**：
- 如果函数签名有 `*args`，调用时传了关键字参数会怎样？
- `*args` 和 `**kwargs` 的顺序有什么限制？为什么 kwargs 必须放最后？
- 如何在类型注解中注解 `*args` 和 `**kwargs`？（`*args: tuple[int, ...]`，`**kwargs: dict[str, Any]`）

---

### Q14：Python 的面向对象魔术方法：`__init__`、`__str__`、`__repr__`、`__eq__`、`__hash__` 分别在什么场景使用？

**考察点**：Python 魔术方法、数据类设计

**答案要点**：
- **`__init__`**：实例初始化，设置属性（构造函数）
- **`__str__`**：`str(obj)` / `print(obj)` 时调用，返回人类可读描述
- **`__repr__`**：`repr(obj)` 时调用，返回开发者友好的"代码式"描述（最好 `eval()` 可还原）
  - 若未定义 `__str__`，`print()` 也会回退调用 `__repr__`
- **`__eq__`**：`==` 比较时调用，默认比较对象 id（`is`），自定义后按属性比较
- **`__hash__`**：`hash(obj)` 时调用，用于 set/dict 键，唯一要求：**相等的对象必须有相同的 hash**
  - 定义 `__eq__` 后，`__hash__` 自动变为 `None`（对象不可哈希），需要手动实现
- **使用场景**：
  - `Alert` 类定义 `__eq__`（按 alert_id 比较）、`__hash__`（基于 alert_id），可以用 set 去重
  - `__repr__` 对调试和日志非常重要：`Alert(id='A001')` > `<Alert object at 0x10a>`

**可能的追问**：
- 什么是 dataclass？`@dataclass` 自动生成了哪些方法？
- 如果一个类定义了 `__eq__` 但没有定义 `__hash__`，实例还能放入 set 吗？
- `__init__` 和 `__new__` 的执行顺序是什么？`__new__` 返回什么时不会调用 `__init__`？

---

### Q15：Python 的继承顺序（MRO，C3 线性化算法）是什么？`super()` 的调用原理？

**考察点**：MRO、C3 线性化、多继承菱形问题

**答案要点**：
- **MRO（Method Resolution Order）**：Python 3 的 C3 线性化算法，确定多继承时从哪个父类开始搜索方法
- **C3 算法核心规则**：
  1. 子类优先于父类
  2. 多个父类按声明顺序
  3. 满足局部优先级和单调性
- **查看 MRO**：类名.`__mro__` 或 `类名.mro()`
- **`super()` 原理**：
  - 不是"调用父类方法"，而是**按 MRO 顺序，调用当前类的下一个类的方法**
  - `super()` 等价于 `super(ClassName, self)`，返回 MRO 中 ClassName 的下一个类的代理对象
  - 正确用法：`super().__init__()`（显式传类名和实例）
  - Python 3：`super().__init__()`（可省略参数）= `super(当前类, self).__init__()`

**可能的追问**：
- 菱形继承（A→B,C→D）中，`D` 类的方法调用顺序是什么？
- `super().method()` 和 `BaseClass.method(self)` 有什么区别？
- mixin 模式是如何利用 MRO 的？

---

### Q16：Python 的垃圾回收机制：引用计数 + 标记清除 + 分代回收分别是什么？循环引用如何处理？

**考察点**：GC 三层机制、循环引用、内存管理

**答案要点**：
- **三层机制**：
  1. **引用计数（Reference Counting）**：
     - 每个对象维护一个引用计数（`sys.getrefcount()` 查看）
     - 引用归零立即回收，**实时**（无延迟）
     - 优点：实时；缺点：无法处理循环引用，且维护计数有开销
  2. **标记清除（Mark & Sweep）**：
     - 解决循环引用问题：GC 启动时，从 GC Root（全局变量、调用栈）出发，标记所有可达对象，清除未标记对象
     - 缺点：需要暂停程序（Stop The World）
  3. **分代回收（Generational GC）**：
     - 新创建的对象生命周期短，频繁检查浪费资源
     - Python 将对象分为三代（0/1/2），新对象在 Gen 0，每经过一次 GC 存活则升级到下一代
     - 越老的对象越稳定，检查频率越低（Gen 0 最频繁）
- **循环引用示例**：
  ```python
  a = []; b = [a]
  a.append(b)  # a 和 b 互相引用，但引用计数 > 0，引用计数无法回收
  del a; del b  # 但标记清除能处理
  ```

**可能的追问**：
- `gc.collect()` 什么时候需要手动调用？
- 什么是 `__del__` 方法？它和垃圾回收有什么关系？为什么不推荐使用？
- 什么是 weakref（弱引用）？在什么场景下使用？

---

### Q17：Python 的单元测试：`unittest` 和 `pytest` 的区别？`@pytest.fixture`、`parametrize`、`mock` 怎么用？

**考察点**：测试框架对比、fixture 机制、mock 技巧

**答案要点**：
- **`unittest` vs `pytest`**：
  | 特性 | unittest | pytest |
  |------|---------|--------|
  | 风格 | Java JUnit 风格，类方法 | 函数式，assert 直接用 |
  | fixtures | 需要继承 `unittest.TestCase` | `@pytest.fixture` 装饰器，更灵活 |
  | 参数化 | `@parameterized`（第三方） | `@pytest.mark.parametrize`（内置） |
  | mock | `unittest.mock` | `pytest-mock`（封装 unittest.mock） |
  | 社区 | 标准库 | 插件生态丰富（pytest-django/pytest-asyncio） |
- **`@pytest.fixture`**：
  ```python
  @pytest.fixture
  def db_connection():
      conn = create_test_db()
      yield conn  # 测试结束后执行
      conn.close()
  ```
- **`@pytest.mark.parametrize`**：
  ```python
  @pytest.mark.parametrize('input,expected', [
      ('abc', 3),
      ('hello', 5),
  ])
  def test_len(input, expected):
      assert len(input) == expected
  ```
- **`mock`**：
  ```python
  from unittest.mock import patch, MagicMock

  @patch('module.redis_client')
  def test_alert_cache(mock_redis):
      mock_redis.get.return_value = '{"alert_id": "A001"}'
      # ...
  ```

**可能的追问**：
- 什么是 fixture 的 scope？（session/module/function/class 级别）
- pytest 的断言重写（assert rewriting）是如何工作的？
- 如何用 `pytest-asyncio` 测试 async 函数？

---

### Q18：Python 的常用标准库：`os`/`sys`/`json`/`re`/`datetime`/`collections`/`itertools`/`functools` 各自的核心用法？

**考察点**：Python 标准库广度

**答案要点**：
- **`os`**：文件系统操作，`os.path.join/abspath/exists`、`os.getenv`、`os.chdir`
- **`sys`**：`sys.path`（模块搜索路径）、`sys.argv`（命令行参数）、`sys.exit()`
- **`json`**：`json.loads/dumps`（JSON 序列化）、`JSONDecodeError` 捕获
- **`re`**：`re.match/search/sub/findall`、`re.compile`（预编译）、`re.DOTALL/IGNORECASE`
- **`datetime`**：`datetime.datetime/date/time`、`timedelta`（时间差）、`strftime`/`strptime`（格式化/解析）
- **`collections`**：
  - `defaultdict`：默认值的 dict（如 `defaultdict(int)` 计数）
  - `Counter`：计数器，`most_common(N)` 求 Top N
  - `deque`：双端队列，`popleft`/`appendleft` 高效
  - `namedtuple`：创建带名字的元组子类型
- **`itertools`**：
  - `count/step` 无限迭代器
  - `chain`：合并多个迭代器
  - `islice`：切片迭代器
  - `groupby`：分组
- **`functools`**：
  - `lru_cache`：函数结果缓存（装饰器）
  - `partial`：偏函数
  - `reduce`：函数式聚合
  - `wraps`：保留被装饰函数元信息

**可能的追问**：
- `datetime` 的 `timestamp()` 和 `fromtimestamp()` 与 `pytz` 时区处理的关系是什么？
- `itertools.groupby` 的前提条件是什么？（必须先排序才能正确分组）
- `functools.reduce` 在 Python 3 中变成了什么？

---

## 4.3 FastAPI

### Q19：FastAPI 的基本用法：如何用 @app.get() 定义路由？如何用 Pydantic BaseModel 定义请求体和响应体？为什么 FastAPI 能自动生成 Swagger/OpenAPI 文档？

**考察点**：FastAPI 路由、请求/响应模型、自动文档

**答案要点**：
- **路由定义**：
  ```python
  from fastapi import FastAPI
  from pydantic import BaseModel

  app = FastAPI(title='SOC Alert API', version='1.0')

  class AlertCreate(BaseModel):  # 请求体
      title: str
      severity: int = 1
      tags: list[str] = []

  class AlertResponse(BaseModel):  # 响应体
      id: str
      title: str
      created_at: datetime

  @app.post('/alerts', response_model=AlertResponse)
  def create_alert(alert: AlertCreate) -> AlertResponse:
      # 保存到数据库...
      return AlertResponse(id='A001', title=alert.title, created_at=datetime.now())
  ```
- **Swagger 自动生成原因**：
  - FastAPI 底层基于 Pydantic（类型注解 → JSON Schema）+ OpenAPI 标准
  - Python 类型注解在运行时通过 `inspect` 提取，`Pydantic` 模型自动生成 JSON Schema
  - 自动生成请求/响应示例、参数校验说明、错误码说明
  - 访问 `/docs`（Swagger UI）或 `/redoc` 查看

**可能的追问**：
- FastAPI 如何处理路径参数、查询参数、Header 参数、Cookie 参数？
- `response_model` 和 `return` 对象类型不一致时会发生什么？
- FastAPI 如何做请求参数校验？Pydantic 的验证失败返回什么 HTTP 状态码？

---

### Q20：FastAPI 的异步支持（async/await）是如何实现的？ASGI 和 WSGI 的区别是什么？

**考察点**：ASGI 协议、uvicorn、async 实现原理

**答案要点**：
- **FastAPI 异步实现**：
  - FastAPI 基于 Starlette（ASGI 工具包），运行在 Uvicorn（ASGI 服务器）上
  - `async def` 路由：注册为 ASGI 应用挂载的协程，由 Uvicorn 的事件循环调度
  - 同步函数：Uvicorn 自动用 `run_in_executor` 丢到线程池执行
- **ASGI vs WSGI**：
  | 维度 | WSGI | ASGI |
  |------|------|------|
  | 协议 | PEP 3333，同步 | 异步，支持 WebSocket |
  | 服务器 | Gunicorn（uWSGI） | Uvicorn/Daphne |
  | 并发 | 多进程/线程 | 单进程 + 事件循环（协程） |
  | WebSocket | ❌ | ✅ |
  | 长连接 | 受限 | ✅（StreamingResponse） |
- **ASGI 原理**：定义了 `app(scope)` 返回一个 `receive/send` 协程。Uvicorn 根据请求类型（HTTP/WebSocket）调用对应的 app 方法

**可能的追问**：
- Uvicorn 的 worker 数量如何配置？为什么建议用 `--workers 4` 而不是单 worker？
- 为什么 FastAPI 推荐用 `async def` 但同步 `def` 也可以工作？两者的性能差异有多大？
- 什么是 `lifespan` 事件？（启动和关闭时的生命周期管理）

---

### Q21：FastAPI 的依赖注入系统是如何工作的？和传统手动传参相比有什么优势？

**考察点**：依赖注入、Depends、缓存

**答案要点**：
- **FastAPI 依赖注入**：
  ```python
  from fastapi import Depends, HTTPException

  def get_db():
      db = SessionLocal()
      try:
          return db
      finally:
          db.close()

  def get_current_user(db: Session = Depends(get_db), token: str = Header(...)):
      # db 自动从 get_db() 获取
      user = db.query(User).filter(User.token == token).first()
      if not user:
          raise HTTPException(401, 'Unauthorized')
      return user

  @app.get('/alerts')
  def list_alerts(current_user: User = Depends(get_current_user)):
      # current_user 自动从依赖链获取
      return db.query(Alert).all()
  ```
- **优势**：
  1. **代码复用**：认证、数据库连接等通用逻辑抽取为依赖函数
  2. **易于测试**：测试时传入 mock 对象覆盖真实依赖
  3. **可叠加**：多个 `Depends` 自动按序执行
  4. **懒加载**：只在路由被调用时才执行依赖
  5. **可缓存**：同请求内多次 `Depends` 同一函数，只执行一次（`use_cache=True`）

**可能的追问**：
- 如何实现带参数的依赖注入？（返回一个可调用对象）
- 什么是子依赖？依赖之间可以相互引用吗？
- 如何实现可选依赖？（`Depends(get_optional_db, default=None)`）

---

### Q22：Pydantic 在 FastAPI 中的角色是什么？数据校验、序列化、Settings 管理各自怎么用？

**考察点**：Pydantic v1 vs v2、数据验证、Settings

**答案要点**：
- **Pydantic 三大角色**：
  1. **数据校验**：BaseModel 自动校验类型、范围、必填、正则，通过 `model_validator` 自定义校验
  2. **序列化**：`model_dump()`（dict）、`model_dump_json()`（JSON）、`model_validate()`（从 dict 构建对象）
  3. **Settings 管理**：`BaseSettings` 类，从环境变量/`.env` 文件读取配置
- **Settings 示例**：
  ```python
  from pydantic_settings import BaseSettings

  class Settings(BaseSettings):
      database_url: str = 'sqlite:///./soc.db'
      openai_api_key: str
      jwt_secret: str

      class Config:
          env_file = '.env'
          env_file_encoding = 'utf-8'

  settings = Settings()  # 自动从环境变量读取
  ```
- **数据校验示例**：
  ```python
  from pydantic import field_validator

  class AlertCreate(BaseModel):
      severity: int
      @field_validator('severity')
      @classmethod
      def severity_must_be_valid(cls, v):
          if v < 1 or v > 5:
              raise ValueError('Severity must be 1-5')
          return v
  ```

**可能的追问**：
- Pydantic v2 和 v1 的主要区别是什么？（`model_validator` 替代 `@validator`，`model_validate` 替代 `parse_obj`）
- `Field` 在请求体、查询参数、路径参数中各自的用法是什么？
- 什么是 Pydantic 的 `mode='python'` 和 `mode='json'`？

---

### Q23：FastAPI 如何做接口版本管理？在 SOC 平台 API 重构过程中，如何保证向后兼容？

**考察点**：API 版本管理策略、向后兼容

**答案要点**：
- **常见版本管理策略**：
  1. **URL 路径**（最常用）：`/v1/alerts`、`/v2/alerts`
  2. **Query 参数**：`/alerts?version=2`（不推荐，污染 URL）
  3. **Header**：`Accept: application/vnd.api.v2+json`（HATEOS，不推荐）
- **向后兼容实践**：
  1. **渐进废弃**（sunset/deprecation）：新增 v2 同时保留 v1，过渡期返回 `DeprecationWarning` Header
  2. **扩展优于修改**：新字段加字段，不断开旧客户端
  3. **不删除字段**：废弃字段返回 null 而非删除
  4. **Schema 版本化**：共享的基础模型用 discriminator 区分版本
- **FastAPI 实现**：
  ```python
  # v1
  @app.get('/v1/alerts')
  def list_alerts_v1() -> list[AlertV1]: ...

  # v2
  @app.get('/v2/alerts')
  def list_alerts_v2() -> list[AlertV2]: ...
  # AlertV2 继承 AlertV1 并新增字段
  class AlertV2(AlertV1):
      enrichment: dict | None = None
  ```

**可能的追问**：
- 什么是 API 的 breaking change？哪些变更算 breaking？
- 如何用 OpenAPI 的 `deprecated` 字段标记废弃接口？
- GraphQL 的 schema 演进和 REST API 版本管理有什么相似和不同？

---

## 4.4 Python 异步编程

### Q24：Python 的 `async def` 和普通 `def` 在执行机制上有什么区别？事件循环在底层是如何调度协程的？

**考察点**：协程对象、事件循环调度原理

**答案要点**：
- **`def` vs `async def`**：
  - `def`：返回函数结果，立即执行
  - `async def`：返回一个**协程对象**（Coroutine），不立即执行，需 `await` 或 `asyncio.run()` 启动
- **事件循环调度机制**：
  1. `asyncio.run()` 创建新事件循环，调用 `loop.run_until_complete(coro)`
  2. 事件循环不断从**就绪队列**取协程执行
  3. 协程执行到 `await future` → 暂停协程，将 `Future` 交给 IO 多路复用（epoll）
  4. IO 完成或 Future 就绪 → 将协程重新放入就绪队列
  5. 重复 3-4，直到协程结束
- **协程状态**：Pending（等待中）、Running（运行中）、Cancelled（被取消）、Finished（完成）

**可能的追问**：
- 协程对象和 Task 对象有什么区别？（Task 是 Future 的子类，可被事件循环调度执行）
- 什么是 `asyncio.create_task`？它和直接 `await coro` 有什么区别？
- 事件循环中的 `call_soon` 和 `call_later` 有什么区别？

---

### Q25：`asyncio.gather`、`asyncio.create_task`、`asyncio.wait` 三者的区别是什么？什么场景下用 gather 收集结果，什么场景下用 create_task 飞驰执行？

**考察点**：并发任务管理、Task vs gather

**答案要点**：
- **三者区别**：
  | 方法 | 返回值 | 行为 | 适用场景 |
  |------|--------|------|---------|
  | `gather` | 结果列表（按参数顺序） | 等待**所有**协程完成 | 需要收集所有结果 |
  | `create_task` | Task 对象 | **不等待**，立即返回 | 需要**飞驰执行**，后续单独 await |
  | `wait` | `(done, pending)` 元组 | 等待满足条件的任务集 | 等待部分完成、超时控制 |
- **`gather` 场景**：并行调用多个 API，收集所有响应：
  ```python
  results = await asyncio.gather(
      fetch_alert(id='A001'),
      fetch_enrichment(id='A001'),
      fetch_threat_intel(id='A001'),
  )
  ```
- **`create_task` 场景**：启动后台任务，不阻塞主流程：
  ```python
  task = asyncio.create_task(send_notification(alert))
  # 继续处理主流程，不等待通知发送完成
  await task  # 可在需要时等待
  ```
- **`wait` 场景**：等待任意一个完成，或超时控制：
  ```python
  done, pending = await asyncio.wait([task1, task2], timeout=5.0)
  ```

**可能的追问**：
- `gather` 中有一个协程抛出异常，会发生什么？（默认 `return_exceptions=False` 时整个 gather 抛出）
- `asyncio.as_completed` 和 `gather` 的区别是什么？
- TaskGroup（Python 3.11+）和 `create_task` 有什么区别？

---

### Q26：FastAPI 中路由函数何时用 `async def` 何用 `def`？如果你在 `async def` 路由里调用了同步阻塞代码，会发生什么？

**考察点**：FastAPI 路由函数选择、阻塞调用风险

**答案要点**：
- **选择原则**：
  - **async def**：调用其他 async 库/函数（`aioredis`、`asyncpg`、`httpx.AsyncClient`）
  - **def**：调用同步库（`requests`、`psycopg2`、`time.sleep`）
- **阻塞代码在 async def 中的风险**：
  - FastAPI 在线程池中用 `run_in_executor` 执行 `def` 函数，不会阻塞事件循环
  - 但如果在 `async def` 中直接调用同步阻塞代码（如 `time.sleep`、同步 DB 驱动），**会阻塞整个事件循环**，所有其他协程都被卡住
  ```python
  # ❌ 错误示例
  @app.get('/slow')
  async def slow_route():
      time.sleep(5)  # 阻塞整个事件循环 5 秒
      return {'msg': 'done'}

  # ✅ 正确做法
  @app.get('/slow')
  async def slow_route():
      await asyncio.sleep(5)  # 挂起协程，事件循环可处理其他请求
      return {'msg': 'done'}

  # ✅ 另一个正确做法（第三方库没有 async 版本）
  async def slow_route():
      loop = asyncio.get_event_loop()
      result = await loop.run_in_executor(None, sync_heavy_task)
  ```

**可能的追问**：
- 如何把 `requests` 这样的同步 HTTP 库变成异步？（`run_in_executor` 或直接用 `httpx.AsyncClient`）
- FastAPI 中同一个事件循环处理多个请求，协程如何切换？（`await` 时切换）
- 什么是"协程饥饿"（coroutine starvation）？什么情况下会发生？

---

### Q27：什么是协程对象（Coroutine Object）和 Future/Task？`await future` 和 `future.result()` 在等待机制上有什么区别？

**考察点**：协程层级、Future 模型、await vs blocking

**答案要点**：
- **三层对象关系**：
  - **协程对象（Coroutine）**：由 `async def` 创建，是最底层对象，`await coro` 执行
  - **Future**：表示**异步操作的最终结果**，有 `.result()`（阻塞）/ `await future`（异步）
  - **Task**：Future 的子类，由 `asyncio.create_task(coro)` 创建，**受事件循环管理，可取消、可查询状态**
- **`await future` vs `future.result()`**：
  - `await future`：**异步等待**，让出事件循环，协程在 future 就绪后恢复执行
  - `future.result()`：**阻塞调用**（调用线程），一直等到结果返回，不让出事件循环
  - 在 async 函数中：必须用 `await`，不能用 `.result()`（除非用了 `asyncio.run_in_executor` 包装）
  ```python
  # 正确
  result = await asyncio.create_task(coro())

  # 错误（会阻塞事件循环）
  task = asyncio.create_task(coro())
  result = task.result()
  ```

**可能的追问**：
- Future 的状态（Pending/Running/Cancelled/Finished）是如何管理的？
- `asyncio.ensure_future` 和 `asyncio.create_task` 有什么区别？
- 如何取消一个 Task？取消后 Task 的异常是什么？

---

### Q28：FastAPI 的后台任务（BackgroundTasks）和 APScheduler 的区别是什么？在告警富化场景中，你用后台任务做什么？

**考察点**：后台任务方案对比、任务队列

**答案要点**：
- **FastAPI BackgroundTasks**：
  - 内置，直接在请求生命周期内添加后台任务
  - 请求结束后后台任务仍执行，适合轻量任务
  - 不保证可靠性（进程重启会丢任务）
  - 最多 1 分钟内的任务
- **APScheduler**：
  - 独立调度器，支持 Cron/Interval/Date 触发
  - 任务持久化（配合数据库可重启恢复）
  - 适合定时任务（报表生成、健康检查、缓存预热）
  - 可以是独立进程/线程
- **告警富化场景**：
  - Alert 创建后，BackgroundTasks 触发：
    1. 调用威胁情报 API（IP/域名查询）
    2. 调用 AI 模型推理（LangChain Agent 研判）
    3. 更新 Alert enrichment 字段
  - 前端立即返回（已创建 Alert），后台异步完成富化

**可能的追问**：
- BackgroundTasks 中的任务抛出异常会怎样？（FastAPI 只记录日志，不会重试）
- 如果需要任务队列+重试+死信队列，应该用 Celery 还是 Dramatiq？
- APScheduler 在多实例部署时如何避免重复执行？（分布式锁或调度器选主）

---

### Q29：如果一个 async 函数里需要调用同步第三方库（不支持 await），如何处理而不阻塞事件循环？

**考察点**：run_in_executor、线程池、IO 绑定

**答案要点**：
- **`run_in_executor`**：
  ```python
  import asyncio
  from concurrent.futures import ThreadPoolExecutor

  async def call_sync_library():
      loop = asyncio.get_running_loop()
      # 默认用 ThreadPoolExecutor
      result = await loop.run_in_executor(None, sync_function, arg1, arg2)
      return result
  ```
- **`ProcessPoolExecutor`**（CPU 密集型）：
  ```python
  executor = ProcessPoolExecutor(max_workers=4)
  result = await loop.run_in_executor(executor, cpu_heavy_function, data)
  ```
- **参数说明**：
  - 第一个参数：执行器对象（`ThreadPoolExecutor`/`ProcessPoolExecutor`），`None` = 默认的 ThreadPoolExecutor
  - 第二个参数：要调用的同步函数
  - 后续参数：函数的参数

**可能的追问**：
- `run_in_executor` 会阻塞事件循环吗？（不会，它在线程/进程池中执行，事件循环继续处理其他协程）
- `aiofiles` 是如何实现异步文件 I/O 的？底层原理是什么？
- 什么时候用 `ThreadPoolExecutor` 什么时候用 `ProcessPoolExecutor`？

---

### Q30：asyncio 如何实现超时控制？`asyncio.timeout`（Python 3.11+）和 `asyncio.wait_for` 有什么区别？

**考察点**：超时控制、shield 保护

**答案要点**：
- **`asyncio.wait_for`**（Python 3.7+，通用）：
  ```python
  result = await asyncio.wait_for(coro(), timeout=5.0)
  # 超时后抛出 asyncio.TimeoutError
  ```
- **`asyncio.timeout`**（Python 3.11+，更现代）：
  ```python
  async with asyncio.timeout(5.0) as cm:
      result = await coro()
  # 超时后 cm.expired == True
  ```
- **两者区别**：
  | 特性 | wait_for | timeout |
  |------|---------|---------|
  | 取消行为 | 超时取消整个协程 | 超时后 context manager 退出，可继续执行 |
  | 灵活性 | 只能用于单个协程 | 可包裹多协程 |
  | Python 版本 | 3.7+ | 3.11+ |
- **`asyncio.shield`**：保护协程不被取消（即使外层超时/取消，被 shield 包裹的协程继续执行直到完成）
  ```python
  result = await asyncio.shield(some_coroutine())
  ```

**可能的追问**：
- `asyncio.timeout` 在超时后，被超时的协程会被取消吗？
- `wait_for` 配合 `return_exceptions=True` 有什么效果？
- 如何实现"最多重试 N 次，每次超时"的逻辑？
