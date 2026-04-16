# 三、Node.js / Koa · 答案

> 本章共 25 题，覆盖 Node.js 核心（Q1-Q8）、Koa 框架（Q9-Q13）、微服务对接（Q14-Q15）、工程进阶（Q16-Q19）、API 安全（Q20-Q25）

---

## 面试官快速索引

| 题号 | 核心考察点 | 难度 | 追问深度 |
|------|-----------|------|---------|
| Q1 | Node.js 架构、V8、非阻塞 I/O | ⭐⭐ | 可深问 libuv 线程池 |
| Q2 | CJS vs ESM、module.exports | ⭐⭐ | 可深问 ESM import 特性 |
| Q3 | Node.js 全局对象、模块作用域 | ⭐⭐ | 可深问 globalThis |
| Q4 | 异常处理、unhandledRejection | ⭐⭐⭐ | 可深问 domain 模块废弃 |
| Q5 | 事件循环阶段（timers/prepare/poll/check） | ⭐⭐⭐⭐ | 可深问 Node.js vs 浏览器事件循环 |
| Q6 | 回调地狱、Promise vs async/await | ⭐⭐ | 可深问错误传播链 |
| Q7 | Stream 四种类型、流式处理大文件 | ⭐⭐⭐ | 可深问背压机制 |
| Q8 | nextTick vs setImmediate | ⭐⭐⭐ | 可深问 queueMicrotask |
| Q9 | Koa 基本用法、ctx、洋葱模型概念 | ⭐⭐ | 可深问中间件注册顺序 |
| Q10 | 洋葱模型中间件机制 | ⭐⭐⭐ | 可深问 compose 实现 |
| Q11 | Koa 网关 vs Nginx 网关 | ⭐⭐ | 可深问 Nginx vs Koa 性能差距 |
| Q12 | Koa 错误处理、try/catch、中间件错误捕获 | ⭐⭐ | 可深问 error 事件 vs 中间件 |
| Q13 | 令牌桶 vs 漏桶、限流实现 | ⭐⭐⭐ | 可深问 Redis 分布式限流 |
| Q14 | BFF 职责、SSR vs API 聚合 | ⭐⭐ | 可深问 BFF vs 直调微服务 |
| Q15 | FastAPI 重构动机、FastAPI vs Koa | ⭐⭐⭐ | 可深问 ASGI vs Koa |
| Q16 | cluster 模式 IPC、父子进程通信 | ⭐⭐⭐ | 可深问负载均衡策略 |
| Q17 | 内存泄漏常见原因、堆快照分析 | ⭐⭐⭐⭐ | 可深问 v8 内存管理 |
| Q18 | nextTick vs setImmediate vs queueMicrotask 优先级 | ⭐⭐⭐ | 可深问微任务调度顺序 |
| Q19 | Graceful Shutdown、SIGTERM 处理 | ⭐⭐⭐ | 可深问 zero-downtime 重启 |
| Q20 | JWT 结构、Access vs Refresh Token | ⭐⭐⭐ | 可深问 JWT 缺陷与改进 |
| Q21 | OAuth2 四种授权模式 | ⭐⭐⭐ | 可深问 PKCE、隐式模式危害 |
| Q22 | RBAC vs ABAC、Koa 网关权限控制 | ⭐⭐⭐ | 可深问策略存储与校验链路 |
| Q23 | CSRF 防御、SameSite vs CSRF Token | ⭐⭐⭐ | 可深问 JWT 的 CSRF 防护 |
| Q24 | 服务间认证、JWT 签名 vs mTLS | ⭐⭐⭐ | 可深问 Istio mTLS |
| Q25 | 令牌桶 vs 滑动窗口、网关 vs 业务层限流 | ⭐⭐⭐ | 可深问 Redis+Lua 限流 |

---

## 3.1 Node.js 核心

### Q1：Node.js 是什么？它和浏览器 JavaScript 的核心区别是什么？为什么 Node.js 能实现非阻塞 I/O？V8 引擎在 Node.js 中扮演什么角色？

**考察点**：Node.js 架构模型、非阻塞 I/O 原理、运行时边界

**答案要点**：
- **Node.js 定义**：基于 Chrome V8 的 JavaScript 运行时，构建在 libuv 之上，提供事件驱动、非阻塞 I/O 模型，用于服务端开发
- **与浏览器 JS 的核心区别**：
  | 维度 | 浏览器 JS | Node.js |
  |------|----------|---------|
  | 执行环境 | 浏览器 | 服务端 |
  | 全局对象 | window | global |
  | 核心 API | DOM/BOM | fs/net/http/buffer |
  | 模块加载 | ESM（CDN import） | CJS + ESM |
  | 访问硬件 | ❌ | ✅（通过原生模块） |
  | 用途 | 前端交互 | 服务端/API/CLI/工具 |
- **非阻塞 I/O 实现**：
  - 依赖 libuv 库（跨平台异步 I/O 抽象层）
  - 线程池处理文件 I/O、DNS 查询等耗时操作，完成后回调放入事件循环队列
  - 主线程不等待，继续处理其他事件，真正实现非阻塞
- **V8 角色**：负责 JavaScript 代码的解析和 JIT 编译执行，不处理 I/O。V8 单线程 + libuv 线程池（默认 4 线程，可配）= Node.js 并发能力

**可能的追问**：
- libuv 线程池大小默认是多少？哪些操作会用到线程池？
- Node.js 的线程池能处理 CPU 密集型任务吗？ Worker Threads 怎么用？
- Node.js 为什么适合 I/O 密集型而不适合 CPU 密集型？

---

### Q2：Node.js 的模块系统：CommonJS（require/module.exports）和 ES Module（import/export）的区别？module.exports 和 exports 的关系是什么？

**考察点**：CJS vs ESM、模块解析差异、exports 引用关系

**答案要点**：
- **CJS vs ESM 对比**：
  | 特性 | CommonJS | ES Module |
  |------|---------|-----------|
  | 语法 | `require`/`module.exports` | `import`/`export` |
  | 加载时机 | 同步（运行时解析） | 异步（编译时静态分析） |
  | 解析 | 动态（可 `require(var)`） | 静态（只允许顶层 `import`） |
  | 缓存 | ✅（require 缓存） | ✅（ES Module Record 缓存） |
  | 循环引用 | 支持（但可能拿到不完整对象） | 支持（TDZ 导致某些情况报错） |
  | 值拷贝 vs 引用 | 值拷贝（原始类型） | 动态绑定（import 可读不可写） |
  | 顶层层级 | 模块顶层代码即同步执行 | 必须 import 后才能用 |
- **module.exports vs exports**：
  - `exports` 是 `module.exports` 的**引用**（初始指向同一对象）
  - `exports.foo = ...` 等价于 `module.exports.foo = ...`
  - `exports = { ... }` 断开了引用，不再等于 `module.exports`（常见错误），需要改回 `module.exports = { ... }`
  - 结论：不要对 `exports` 重新赋值，只用于添加属性

**可能的追问**：
- Node.js 中如何同时支持 CJS 和 ESM？（`"type": "module"`、`.mjs`/`.cjs` 扩展名）
- `import()` 动态导入和 `require()` 有什么区别？
- ESM 中 `import * as xxx` 和 `import { a, b }` 的区别？

---

### Q3：Node.js 全局对象（process、Buffer、__dirname、__filename、global）各自的作用？什么是模块作用域（Module Scope）？

**考察点**：Node.js 全局 API、模块隔离机制

**答案要点**：
- **核心全局对象**：
  - **`process`**：进程对象，`process.env`（环境变量）、`process.argv`（命令行参数）、`process.cwd()`（当前工作目录）、`process.exit()`、`process.on('uncaughtException')`
  - **`Buffer`**：二进制数据缓冲区，`Buffer.from()`、`Buffer.alloc()`，用于处理 TCP 流、文件 I/O、加密
  - **`__dirname`**：当前模块文件所在的目录路径（不是 cwd）
  - **`__filename`**：当前模块文件的绝对路径
  - **`global`**（或 `globalThis`）：全局对象顶层，浏览器中等价于 `window`，Node.js 中全局变量挂载于此
- **模块作用域**：
  - 每个 `.js` 文件（模块）有独立的顶层作用域
  - 模块内 `var/function` 声明不会泄漏到 `global`
  - 模块间通过 `require` 共享，不通过 global 共享
  - 这就是 Node.js 模块隔离机制，不存在全局变量污染

**可能的追问**：
- `process.env` 在生产环境中如何管理敏感信息？（dotenv、Vault、k8s Secret）
- `Buffer` 的内存是堆内还是堆外？`Buffer.allocUnsafe` 和 `Buffer.alloc` 的区别？
- `globalThis` 为什么重要？（统一浏览器和 Node.js 的全局对象访问方式）

---

### Q4：Node.js 如何处理未捕获异常？process.on('uncaughtException') 和 process.on('unhandledRejection') 的区别？生产环境应该怎么用？

**考察点**：Node.js 异常处理机制、进程稳定性

**答案要点**：
- **uncaughtException**：
  - 捕获所有未被任何 try/catch 包裹的同步错误
  - 事件触发后，Node.js 进入"不稳定状态"，建议立即退出进程重启
  - 使用场景：记录日志、清理资源，然后 `process.exit(1)`
  ```javascript
  process.on('uncaughtException', (err) => {
    logger.error(err, 'Uncaught Exception');
    process.exit(1); // 必须退出，否则处于未定义状态
  });
  ```
- **unhandledRejection**：
  - 捕获 Promise 被 reject 但没有 `.catch()` 处理的情况
  - Node.js 15+ 默认将未处理 rejection 转为 `uncaughtException`（进程退出）
  - Node.js 14 及以下需手动监听，否则静默忽略
- **生产环境最佳实践**：
  - **不要**在 `uncaughtException` 中继续执行业务逻辑
  - 使用进程管理器（PM2）实现自动重启
  - 推荐使用 `domain` 已废弃，使用 `async_hooks` 或 ` zones`（复杂）做更细粒度的错误隔离
  - 所有异步操作加 `.catch()`，或使用 `process.on('unhandledRejection', ...)` 兜底并报警

**可能的追问**：
- 为什么 Node.js 官方建议 `uncaughtException` 后要退出进程？
- PM2 的自动重启在什么情况下会陷入重启循环？如何配置重启策略？
- try/catch 在异步代码中的局限性是什么？（无法捕获 async 函数外的异步错误）

---

### Q5：Node.js 的事件循环机制有几个阶段？每个阶段处理什么类型的任务？

**考察点**：libuv 事件循环六阶段、微任务执行时机

**答案要点**：
- **Node.js 事件循环六阶段**（libuv，基于 poll/epoll）：
  ```
  ┌───────────────────────┐
  │   timers (阶段1)        │  执行 setTimeout / setInterval 回调
  │   pending callbacks    │  执行上一轮延迟的 I/O 回调
  │   idle, prepare        │  内部使用
  │   poll (阶段2)          │  检索新的 I/O 事件；执行 I/O 回调（除 close/cb/timer）
  │   check (阶段3)         │  执行 setImmediate 回调
  │   close callbacks (阶段4) │ 执行 close 事件回调（如 socket.on('close')）
  └───────────────────────┘
  ```
  - **每个阶段结束后、下个阶段开始前**，清空**微任务队列**（Promise.then、queueMicrotask、process.nextTick）
  - `process.nextTick` 在**每个阶段末尾微任务之前**执行，优先级高于其他微任务
- **poll 阶段特殊行为**：
  - poll 队列为空且无 timers → 阻塞等待新 I/O
  - poll 队列不为空 → 同步执行所有 I/O 回调直至清空或达到系统限制
  - 有 `setImmediate` 调度 → 跳过 poll 阻塞，进入 check 阶段

**可能的追问**：
- 为什么说 `setTimeout(fn, 0)` 不一定比 `setImmediate` 快？
- `setImmediate` 相比 `setTimeout(fn, 0)` 的确定性场景是什么？（I/O 回调内：`setImmediate` 一定先执行）
- poll 阶段执行大量 I/O 回调时，会被 check 阶段的 `setImmediate` 打断吗？

---

### Q6：什么是回调地狱？Promise、async/await 如何解决？它们的错误处理方式有什么区别？

**考察点**：回调地狱、Promise 链式调用、async/await 语法糖

**答案要点**：
- **回调地狱**：多层嵌套回调导致代码横向发展（三角形），难以阅读、调试和维护
  ```javascript
  // 地狱
  fs.readFile('a.json', (err, a) => {
    fs.readFile(a.path, (err, b) => {
      fs.readFile(b.url, (err, c) => { ... })  // 无限嵌套
    })
  })
  ```
- **Promise 解决**：链式调用 `.then().catch()`，扁平化代码，错误沿链条传播
  ```javascript
  fs.promises.readFile('a.json')
    .then(a => fs.promises.readFile(a.path))
    .then(b => fs.promises.readFile(b.url))
    .catch(err => console.error(err))
  ```
- **async/await 解决**：同步写法，异步代码看起来像同步，更符合直觉
  ```javascript
  async function main() {
    try {
      const a = await fs.promises.readFile('a.json')
      const b = await fs.promises.readFile(a.path)
      const c = await fs.promises.readFile(b.url)
    } catch (err) { /* 统一处理 */ }
  }
  ```
- **错误处理区别**：
  - Promise `.catch()` 捕获该链上所有之前的错误，但**之后的 `.then` 仍会执行**（catch 后可继续返回新 Promise）
  - async/await 用 `try/catch`，语义清晰，catch 块后代码不执行
  - async 函数中未被 catch 的 Promise rejection 会向上传播，最终触发 `unhandledRejection`

**可能的追问**：
- `Promise.all`、`Promise.race`、`Promise.allSettled` 的区别和使用场景？
- async/await 中，如果有两个没有依赖的 await，可以并行执行吗？（同时触发两个 await，不需要先等一个）
- `await promise` 和 `await Promise.resolve(promise)` 有什么区别？

---

### Q7：Node.js 中 Stream 的四种类型是什么？在文件导入导出场景中如何使用 Stream 来处理大文件？

**考察点**：Stream 分类、背压机制、大文件处理

**答案要点**：
- **四种 Stream 类型**：
  | 类型 | 方向 | 示例 |
  |------|------|------|
  | **Readable** | 读取数据 | `fs.createReadStream()`、`http.IncomingMessage` |
  | **Writable** | 写入数据 | `fs.createWriteStream()`、`http.ServerResponse` |
  | **Duplex** | 既读又写 | `net.Socket`、`zlib.createGzip` |
  | **Transform** | 读后转换再写 | `zlib.createDeflate`、`crypto.createCipher` |
- **大文件处理示例**（读取大 CSV → 处理 → 写入）：
  ```javascript
  const readStream = fs.createReadStream('big.csv', { highWaterMark: 64 * 1024 }); // 64KB chunk
  const writeStream = fs.createWriteStream('output.csv');
  const transformStream = csvTransform(); // Transform 流

  readStream
    .pipe(transformStream)   // pipe 自动处理背压
    .pipe(writeStream)
    .on('finish', () => console.log('Done'));
  ```
- **背压机制（Backpressure）**：Readable 写入速度 > Writable 消费速度时，`.pipe()` 会自动暂停 Readable，防止内存溢出

**可能的追问**：
- `.pipe()` 和 `.on('data')` + `.on('end')` 手动处理相比，优势是什么？
- `highWaterMark` 对流处理性能的影响是什么？设置过大会怎样？
- Transform 流中 `transform()` 和 `flush()` 回调的区别是什么？

---

### Q8：process.nextTick 和 setImmediate 的区别？哪个先执行？为什么？

**考察点**：nextTick vs setImmediate、执行顺序

**答案要点**：
- **执行顺序**（每个事件循环阶段末尾微任务队列处理时）：
  ```
  每个阶段结束后:
    1. 微任务队列（Promise callbacks, queueMicrotask）
    2. process.nextTick 队列（优先级最高）
  ```
  - `process.nextTick` 在**当前阶段所有微任务之前**执行
  - `setImmediate` 在**check 阶段**执行（poll 阶段之后）
- **结论**：
  - 在 I/O 回调**内部**：`setImmediate` 一定先于 `process.nextTick` 执行（因为 nextTick 在 poll 阶段末尾，但此时已经在 check 阶段）
  - 在同步代码**之后**：`process.nextTick` 先于 `setImmediate`
- **性能影响**：
  - `process.nextTick` 可能导致 I/O starve（不断插入 nextTick 导致 poll 阶段永远不清空）
  - `setImmediate` 更适合在事件循环每个 tick 的末尾执行任务
- **原则**：`process.nextTick` 递归调用可能造成死循环，应优先使用 `setImmediate`

**可能的追问**：
- 为什么 Node.js 要单独实现 `process.nextTick`，而不统一用 `queueMicrotask`？
- `queueMicrotask` 和 `Promise.resolve().then` 有什么关系？
- 如果在 `process.nextTick` 中递归调用 `process.nextTick`，会发生什么？

---

## 3.2 Koa 框架

### Q9：Koa 的基本用法：app.use()、ctx（请求上下文）、next 函数各自的含义？如何快速写一个"Hello World"的 Koa 服务？

**考察点**：Koa 核心概念、middleware 模型

**答案要点**：
- **核心概念**：
  - **`app.use(middleware)`**：注册中间件，函数签名为 `async (ctx, next) => {}`
  - **`ctx`**（请求上下文）：封装了 `Request`（请求）和 `Response`（响应），提供 `ctx.request`/`ctx.response`，以及快捷属性如 `ctx.body`（=`ctx.response.body`）、`ctx.query`、`ctx.params`
  - **`next`**：调用下一个中间件的函数，不调用则后续中间件不执行
- **Hello World 示例**：
  ```javascript
  const Koa = require('koa');
  const app = new Koa();

  app.use(async (ctx, next) => {
    ctx.body = 'Hello World';
    await next(); // 可选
  });

  app.listen(3000, () => console.log('Server running on port 3000'));
  ```

**可能的追问**：
- `ctx.body = null` 和 `ctx.status = 204` 有什么区别？
- Koa 的 `app.context` 是什么？如何在 context 上挂载公共方法？
- Koa 如何获取请求头和查询参数？`ctx.headers` vs `ctx.query` vs `ctx.querystring`？

---

### Q10：Koa 的洋葱模型中间件机制是什么？和 Express 的中间件模型有什么区别？

**考察点**：洋葱模型原理、compose 实现、Express 对比

**答案要点**：
- **洋葱模型**：
  - 每个中间件可以控制**进入**和**离开**的时机
  - `await next()` 之前是"进入"逻辑，`await next()` 之后是"离开"逻辑
  - 所有中间件构成多层嵌套，执行顺序：1→2→3→(最深层)→3→2→1
  - 典型应用：请求日志（next 前记录开始时间，next 后记录结束时间）、事务管理（begin before, commit/rollback after）
  ```javascript
  // 洋葱模型示意
  app.use(async (ctx, next) => {
    console.log('A - before next');     // 1. A 进入
    await next();
    console.log('A - after next');       // 6. A 离开
  });
  app.use(async (ctx, next) => {
    console.log('B - before next');     // 2. B 进入
    await next();
    console.log('B - after next');      // 5. B 离开
  });
  app.use(async (ctx, next) => {
    console.log('C - before next');     // 3. C 进入
    await next();
    console.log('C - after next');      // 4. C 离开（无后续中间件，直接返回）
  });
  // 输出: A→B→C→C→B→A
  ```
- **Koa vs Express**：
  - Express 的 next 是同步的（function-based），错误通过 `next(err)` 传播
  - Koa 的 next 是 async 函数，`await next()` 可控制异步流程，更强大
  - Express 中间件可以调用 `next()` 多次，Koa 中间件通常只调用一次 `next()`

**可能的追问**：
- Koa 洋葱模型的 `compose` 函数（koa-compose）核心实现是怎样的？（reduce + promise 链）
- 如果中间件忘记调用 `await next()` 会怎样？（后续中间件不执行，请求挂起）
- Koa 的错误传播机制是什么？`ctx.throw` 和 `app.on('error')` 关系？

---

### Q11：你在 SOC 平台中用 Koa 开发 API 网关，Koa 做网关层和 Nginx 做网关层有什么区别？为什么选择 Koa？

**考察点**：网关层选型、Node.js 网关优势

**答案要点**：
- **Koa vs Nginx**：
  | 维度 | Nginx | Koa 网关 |
  |------|-------|---------|
  | 层次 | 网络层（TCP/HTTP） | 应用层（业务逻辑） |
  | 擅长 | 反向代理、负载均衡、静态文件、SSL 终结 | 请求聚合、数据转换、鉴权、业务编排 |
  | 性能 | 极高（C 语言，事件驱动 epoll） | 中等（V8 引擎，但足够业务量级） |
  | 灵活性 | 配置文件，不方便业务定制 | JavaScript/TypeScript，可深度定制 |
  | 长连接 | ✅（stream） | ✅（但性能不如 Nginx） |
  | 缓存 | ✅（proxy_cache） | 需自行实现 |
  | 协议转换 | ❌ | ✅（如转 RPC、GraphQL 聚合） |
- **SOC 平台选择 Koa 的原因**：
  1. **业务定制**：需要对接 Nacos 做服务发现、动态路由、请求聚合，这些是 Nginx 无法直接完成的
  2. **前端友好**：前端团队用 TypeScript/Node.js 统一技术栈，开发效率高
  3. **中间件生态**：Koa 洋葱模型非常适合统一注入认证、限流、日志
  4. **与微服务配合**：网关做参数转换、响应适配，后端微服务专注业务逻辑
  5. **生产方案**：K8s Ingress 前面再加一层 Nginx 做负载均衡和 SSL 终结，Koa 在 Ingress 之后

**可能的追问**：
- 为什么生产环境中 Nginx 作为前端代理（SSL 终结、静态资源）和 Koa 网关配合使用？
- Koa 网关如何做服务发现和动态路由？（对接 Nacos SDK）
- 如果 Koa 网关需要处理高并发（10万+ 并发），有哪些优化手段？

---

### Q12：Koa 的错误处理最佳实践是什么？如何在洋葱模型中统一捕获异常？

**考察点**：Koa 错误处理链、error 中间件

**答案要点**：
- **Koa 错误处理最佳实践**：
  1. **统一 error 中间件**（放在最外层，作为第一个中间件）：
     ```javascript
     app.use(async (ctx, next) => {
       try {
         await next();
       } catch (err) {
         const status = err.status || err.statusCode || 500;
         ctx.status = status;
         ctx.body = {
           code: err.code || 'INTERNAL_ERROR',
           message: status === 500 ? 'Internal Server Error' : err.message,
         };
         logger.error({ err, path: ctx.path }, 'Request error');
         ctx.app.emit('error', err, ctx); // 触发 app 层面 error 事件
       }
     });
     ```
  2. **使用 `ctx.throw`**：业务层直接 `ctx.throw(400, 'Invalid parameter')`，自动触发 error 中间件捕获
  3. **`app.on('error')`**：全局日志记录，发送告警（如 Sentry 上报）
- **关键点**：
  - error 中间件必须放在所有业务中间件**之前**（洋葱模型最外层）
  - 生产环境不应暴露 500 错误的详细堆栈（安全考虑）
  - 所有异步代码用 `try/catch` 包裹，或使用 `await next()` 统一 catch

**可能的追问**：
- `ctx.throw(401)` 和 `ctx.throw(401, 'Unauthorized', { expose: true })` 有什么区别？
- 什么是 Koa 的 `app.context` 和 `ctx.state`？它们各自的用途？
- Koa 如何做请求日志中间件？`ctx.request.id`（请求追踪 ID）如何传递？

---

### Q13：Koa 中如何实现请求限流？令牌桶和漏桶算法的区别？

**考察点**：限流算法、rate-limit 中间件实现

**答案要点**：
- **令牌桶算法**：
  - 以恒定速率向桶中添加令牌，桶有容量上限
  - 请求到达时消耗一个令牌，无令牌则拒绝
  - **允许突发流量**（桶满时一次性通过多个请求）
  - 常用库：`koa2-ratelimit`
- **漏桶算法**：
  - 请求以任意速率进入桶，以恒定速率从桶底漏出
  - 无论突发多大，输出速率恒定
  - **不允许突发**（所有请求必须等待桶底漏出）
- **区别对比**：
  | 特性 | 令牌桶 | 漏桶 |
  |------|-------|------|
  | 突发处理 | ✅ 支持 | ❌ 不支持 |
  | 输出速率 | 可变 | 恒定 |
  | 典型应用 | 限速下载、API 限流 | 流量整形、音视频流控 |
  - Koa 中实现：Redis + Lua 脚本实现原子令牌桶（`INCR` + `EXPIRE` + `GETTTL`）

**可能的追问**：
- 如何用 Redis + Lua 实现原子令牌桶限流？
- 滑动窗口限流相比令牌桶/漏桶有什么优势？
- 分布式限流中，如何保证限流 key 的原子性？（Redis MULTI/EXEC vs Lua 脚本）

---

## 3.3 Node.js 与微服务对接

### Q14：你们通过 Nacos 与后端微服务对接，Node 层作为 BFF（Backend for Frontend）的职责是什么？和直接前端调微服务相比有什么优势？

**考察点**：BFF 架构模式、微服务聚合

**答案要点**：
- **BFF 职责**：
  1. **接口聚合**：前端一次请求，BFF 从多个微服务并行拉取数据后聚合返回（如告警详情页：告警信息 + 关联 Case + 威胁情报 + Enrichment）
  2. **数据转换**：适配前端需要的格式（字段映射、类型转换）
  3. **鉴权透传**：JWT 验证 + 向下游服务透传用户身份
  4. **统一错误处理**：将各微服务错误格式统一为前端友好格式
  5. **限流熔断**：保护下游微服务不过载
- **相比直接前端调微服务**：
  - **减少 HTTP 请求数**：3个并行微服务调用 → 1次前端请求
  - **网络开销小**：内网 BFF 调用延迟低（~1ms），跨域前端直调延迟高
  - **接口版本隔离**：微服务接口变化不影响前端（由 BFF 做兼容）
  - **安全**：隐藏后端微服务地址，前端只能访问 BFF
  - **团队解耦**：前端团队和微服务团队独立迭代，通过 BFF 接口契约对齐

**可能的追问**：
- BFF 层如何做熔断？（当某个下游服务慢或超时时，返回降级数据）
- 如果 BFF 是单点故障，如何高可用部署？（多实例 + 负载均衡 + 健康检查）
- GraphQL 和 BFF 模式有什么相似和不同？

---

### Q15：FastAPI 重构原 Node 层 API 接口时，你调整了接口参数与响应值，这个重构的动机是什么？FastAPI 相比 Koa 在哪些场景更有优势？

**考察点**：FastAPI vs Koa、技术选型理由

**答案要点**：
- **重构动机**：
  1. **类型安全**：Python FastAPI + Pydantic 实现运行时+编译时双重校验，避免 Node 层手动校验参数
  2. **AI 集成**：SOC Analyst Agent 的 AI 研判能力（LangChain/LangGraph）本身就是 Python 技术栈，统一语言减少桥接开销
  3. **性能优化**：Python 异步 I/O（uvicorn + ASGI）性能不弱于 Koa，且支持高并发
  4. **Swagger 文档**：FastAPI 自动生成 OpenAPI 文档，减少接口维护成本
- **FastAPI 优势场景**：
  - **数据密集/计算密集**：AI 推理（RAG/LLM 调用）、数据分析
  - **强类型 API**：Pydantic 自动校验请求/响应，自动生成文档
  - **AI 集成**：LangChain、LangGraph、Pydantic 在 Python 生态更成熟
  - **快速原型**：自动文档 + 类型推导，开发效率高
- **Koa 仍擅长的场景**：
  - 前端团队主导，全栈 TypeScript 单仓库
  - 轻量网关、代理层（Koa 更灵活，中间件更丰富）
  - 快速迭代、轻量 API（Node.js npm 生态更丰富）

**可能的追问**：
- FastAPI 的 ASGI 模型相比 Koa 的 Node.js 单线程有什么性能优势？
- Python 的 GIL 是否影响 FastAPI 的并发性能？（uvicorn worker 数量配置、async I/O）
- FastAPI 如何与 Koa 网关共存？两个层之间如何路由？

---

## 3.4 Node.js 工程进阶

### Q16：Node.js 在 cluster 模式下，父子进程之间如何通信（IPC）？进程间通信的消息格式是什么？master 如何向 worker 传递请求？

**考察点**：cluster 模块原理、IPC 通道、负载均衡

**答案要点**：
- **IPC 通信机制**：
  - `cluster.isMaster` 判断当前是否为 master 进程
  - `cluster.fork()` 启动 worker 子进程
  - 父子进程通过**IPC通道**（Unix Domain Socket 或 Windows Pipe）通信
  - 通信消息格式：JSON 对象 `{ action: 'online', ... }`，Node.js 内部维护消息队列
  - 常用消息类型：`online`（worker 就绪）、`listening`（监听端口）、`message`（自定义消息）、`disconnect`/`exit`
- **master 向 worker 传递请求**：
  - 在**非 Windows 平台**，Node.js cluster 使用**round-robin**负载均衡
  - 主进程收到请求 → 分发给 worker（master 不处理请求，只做调度）
  - 每个 worker 独立绑定端口（共享 Server），OS 层负责分发连接
  - Windows 下不支持 round-robin，master 用 `child_process.fork()` + 手动消息转发（性能较差）
- **PM2 cluster 模式**：封装了 Node.js 原生 cluster，提供更优雅的进程管理、热重载、负载均衡策略

**可能的追问**：
- cluster 模式下，如果一个 worker 崩溃了，其他 worker 会被影响吗？
- PM2 的 `exec_mode: 'cluster'` 和原生 `cluster` 模块有什么区别？
- cluster 模式下如何共享内存？（`worker.sharedMem` 或 Redis 共享）

---

### Q17：Node.js 的内存泄漏常见原因有哪些？闭包、全局变量、Buffer 使用不当分别会怎样？如何用 `--inspect` 和 Chrome DevTools 做堆快照分析？

**考察点**：内存泄漏原因、v8 堆结构、调试方法

**答案要点**：
- **常见内存泄漏原因**：
  1. **全局变量**：全局 `Map`/`Set`/`Array` 不断添加元素但不清理
  2. **闭包**：回调函数持有外部作用域引用，闭包变量无法被 GC 回收
  3. **未清理的定时器**：`setInterval` 未 `clearInterval`，定时器持有的回调和上下文无法释放
  4. **EventEmitter 监听器未移除**：`emitter.on` 后不 `emitter.off`，重复注册导致重复回调
  5. **Buffer 使用不当**：`Buffer.from()` 大字符串/Buffer 不及时释放
  6. **缓存无上限**：内存缓存不设大小限制，数据无限增长
- **heap snapshot 分析步骤**：
  1. 启动：`node --inspect server.js`，或运行时 `kill -USR2 <pid>` 触发快照
  2. Chrome DevTools → Memory → Load snapshot 打开 `.heapsnapshot` 文件
  3. **Summary 视图**：按构造函数分组，查看占用最大的对象
  4. **Comparison 视图**：对比两个快照，找出增量最大的对象
  5. **Containment 视图**：从根节点（GC Root）追踪对象引用链，找到泄漏路径
- **关键指标**：Shallow Size（对象自身大小）vs Retained Size（含所有引用对象的总大小）

**可能的追问**：
- v8 的分代垃圾回收（New Space / Old Space / Large Object Space）如何影响内存管理？
- 什么是内存泄漏的"渐变型"和"爆发型"？如何分别定位？
- `node --max-old-space-size=4096` 如何选择合适的值？如何监控内存使用率？

---

### Q18：Node.js 的事件循环中，`setImmediate` 和 `process.nextTick` 的优先级是什么？`queueMicrotask` 和它们的区别是什么？

**考察点**：微任务队列优先级、queueMicrotask 规范

**答案要点**：
- **优先级顺序**（在同一事件循环 tick 内）：
  ```
  process.nextTick 队列（优先级最高）
  ↓
  queueMicrotask 队列（Promise callbacks）
  ↓
  其他微任务（MutationObserver 等）
  ↓
  setImmediate 回调（check 阶段）
  ```
- **`queueMicrotask`**：
  - Web 标准 API（与浏览器一致），`globalThis.queueMicrotask(fn)`
  - 回调进入微任务队列，与 Promise.then 同级
- **`process.nextTick` vs `queueMicrotask`**：
  - `process.nextTick` 是 Node.js 专有，高于 `queueMicrotask`
  - `nextTick` 的回调可能在同一阶段多次插入，递归调用会"插队"，可能导致 I/O starve
  - `queueMicrotask` 更安全，遵循 Web 标准，Node.js 建议优先使用
- **选择建议**：新代码优先使用 `queueMicrotask`（标准、跨环境）

**可能的追问**：
- `queueMicrotask` 在 Node.js 和浏览器中的行为是否完全一致？
- 在 V8 层面，微任务队列是如何实现的？（v8::TaskQueue）
- 如果递归调用 `queueMicrotask`，会有什么后果？（同样可能 starve I/O，但比 nextTick 慢插队）

---

### Q19：如何设计一个 Node.js 服务的优雅停机（Graceful Shutdown）？在收到 SIGTERM 信号后，需要做哪些清理工作？

**考察点**：进程信号处理、优雅停机设计

**答案要点**：
- **Graceful Shutdown 设计**：
  ```javascript
  const server = app.listen(3000);
  let isShuttingDown = false;

  // 1. 监听 SIGTERM（K8s/PM2 发送）
  process.on('SIGTERM', async () => {
    if (isShuttingDown) return;
    isShuttingDown = true;
    console.log('Received SIGTERM, shutting down gracefully...');

    // 2. 停止接收新请求（不再 listen）
    server.close(async () => {
      console.log('HTTP server closed');

      // 3. 关闭数据库连接
      await db.close();

      // 4. 关闭 Redis 连接
      await redis.quit();

      // 5. 等待处理中的请求完成（设置超时）
      await waitForActiveRequests(10_000); // 等最多 10s

      console.log('Cleanup done, exiting');
      process.exit(0);
    });

    // 6. 超时强制退出
    setTimeout(() => {
      console.error('Graceful shutdown timeout, forcing exit');
      process.exit(1);
    }, 15_000);
  });
  ```
- **关键步骤**：
  1. `server.close()` 停止接收新连接，但处理中的请求继续完成
  2. 关闭 DB/Redis 连接池（避免连接泄漏）
  3. 将自己从服务注册中心（Nacos/Eureka）注销
  4. 等待活跃请求完成（计数跟踪 + 超时保护）
  5. 写入最后日志

**可能的追问**：
- 为什么 K8s 给容器发 SIGTERM 而不是 SIGKILL？SIGTERM 和 SIGINT 有什么区别？
- PM2 的 `kill` 和 `reload` 命令分别触发什么信号？
- 在容器中，`pid 1` 进程（ENTRYPOINT/CMD）如何正确处理 SIGTERM？（用 tini/init 或 shell 脚本 trap）

---

## 3.5 API 安全

### Q20：JWT（JSON Web Token）的结构是什么？Access Token 和 Refresh Token 为什么通常需要两套？你们的 SOC 平台网关层用的是 JWT 还是 Session？

**考察点**：JWT 结构、Token 安全策略

**答案要点**：
- **JWT 结构**（`.` 分隔的三部分）：
  - **Header**：`{"alg": "HS256", "typ": "JWT"}`，Base64URL 编码
  - **Payload**：`{"sub": "user123", "role": "admin", "exp": 1234567890}`，含标准字段（iss/exp/sub/iat）和业务字段
  - **Signature**：`HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)`
- **Access Token vs Refresh Token**：
  - **Access Token**：短期（15min-1h），包含必要身份信息，频繁传输
  - **Refresh Token**：长期（7d-30d），仅用于换新 Access Token，存储更安全（httpOnly Cookie 或加密存储）
  - 原因：Access Token 泄露风险高（它本身就暴露在各处），但短期有效减少窗口期；Refresh Token 泄露影响可控（可服务端撤销）
- **SOC 平台方案**：Koa 网关层使用 **JWT（Access Token）**，放在 `Authorization: Bearer <token>` 头中，服务端 Redis 维护 Token 黑名单（支持主动失效）

**可能的追问**：
- JWT 的缺点是什么？（无法主动失效、Payload 不加密只能签名）
- JWT 的 `exp` 和 `nbf` 字段分别解决什么问题？
- Access Token 泄露了怎么办？如何实现 Token 主动失效？

---

### Q21：OAuth2 的四种授权模式（授权码/隐式/密码凭证/客户端凭证）各自适合什么场景？为什么隐式模式已被主流废弃？

**考察点**：OAuth2 授权模式、安全风险

**答案要点**：
- **四种模式**：
  | 模式 | 适用场景 | token 传输方式 |
  |------|---------|--------------|
  | **授权码（Authorization Code）** | 有后端服务器的 Web 应用，最安全 | 通过后端 exchange code 获取 token，不暴露在前端 |
  | **隐式（Implicit）** | 纯前端 SPA（无后端），已废弃 | URL fragment 传 token，直接暴露 |
  | **密码凭证（Resource Owner Password）** | 高度可信的自家应用（迁移期） | 直接用户名密码换 token |
  | **客户端凭证（Client Credentials）** | 服务间通信（机器对机器） | 无用户参与，用 client_id + secret 换 token |
- **隐式模式被废弃的原因**：
  1. **Token 暴露在 URL 中**：通过浏览器历史记录、Referer 头泄露
  2. **无 Token 验证**：无法确认 Token 属于哪个 Client
  3. **无法使用 Refresh Token**：隐式模式不支持，而长期 Token 没有后端保护很危险
  4. **PKCE 出现后更好方案**：SPA 现推荐 **授权码 + PKCE** 替代隐式模式
- **SOC 平台场景**：对接第三方系统（企业微信/飞书）用**授权码模式**

**可能的追问**：
- PKCE（Proof Key for Code Exchange）是如何防止授权码拦截攻击的？
- 什么是 `state` 参数？为什么 OAuth2 必须用它防止 CSRF？
- OpenID Connect (OIDC) 是在 OAuth2 基础上增加了什么？

---

### Q22：RBAC（基于角色的访问控制）和 ABAC（基于属性的访问控制）的区别是什么？你们的 SOC 平台用的是哪种？平台里的 Alert/Case/Ticket 等对象的权限是如何控制的？

**考察点**：权限模型设计、RBAC vs ABAC

**答案要点**：
- **RBAC vs ABAC**：
  | 维度 | RBAC | ABAC |
  |------|------|------|
  | 控制维度 | 角色（role） | 用户属性（attr）+ 资源属性 + 环境属性 |
  | 复杂度 | 简单，适合角色边界清晰 | 复杂，适合细粒度动态策略 |
  | 权限粒度 | 粗粒度（按角色分配菜单/功能） | 细粒度（"华东区告警，只有 L3 以上分析师可处理"） |
  | 扩展性 | 新增角色麻烦 | 新增属性即可 |
  | 审计 | 按角色审计 | 按属性策略审计 |
  | 实现 | 简单，数据库 RBAC 表 | 复杂，需要策略引擎（Casbin/OPA） |
- **SOC 平台方案**：以 **RBAC 为主**，Alert/Case/Ticket 对象级别权限通过**数据权限**扩展（如"只能查看自己创建的 Case"）
  - 角色：`admin` / `analyst` / `viewer`
  - Alert 状态流转权限：analyst 可认领、admin 可强制关闭
  - Case 分派：analyst 只能分派给自己的下级 analyst（L3 只能看/处理 L2 分派的 Case）
- **Koa 网关实现**：中间件从 JWT 解析用户角色 → 查询权限表（Redis 缓存）→ 判断是否有对应接口权限

**可能的追问**：
- 如果需要同时支持"角色+数据权限"（如只能看自己创建的告警），如何实现？
- 什么是 ABAC 的策略即代码（Policy as Code）？Casbin 的模型是什么？
- 权限变更（如用户被踢出系统）如何实时生效？（Redis 缓存 TTL + 发布订阅主动失效）

---

### Q23：API 层如何防御 CSRF 攻击？SameSite Cookie 和 CSRF Token 各自解决了什么问题？

**考察点**：CSRF 机制、双重防御策略

**答案要点**：
- **CSRF 攻击原理**：用户登录目标网站后，访问恶意页面，页面自动携带目标站点的 Cookie 发送跨域请求（如 POST 转账），浏览器自动带上同源 Cookie
- **SameSite Cookie**：
  - `SameSite=Strict`：所有跨域请求都不带 Cookie，完全防止 CSRF
  - `SameSite=Lax`（新浏览器默认）：GET 请求允许跨域带 Cookie，POST 不带
  - **不完美之处**：Lax 下 GET 请求仍可能遭受 CSRF（如 `<img src="http://bank.com/transfer?to=hacker&amount=1000">`）
- **CSRF Token**：
  - 服务端生成随机 Token，嵌入表单（`<input type="hidden" value="csrf-token">`）或响应 Header
  - 请求时携带 Token，服务端验证 Token 存在且正确
  - Token 不在 Cookie 中，通过 URL 参数/Header 传输，攻击者无法获取
- **双重防御**：SameSite Cookie（第一道防线）+ CSRF Token（第二道防线兜底），最安全
- **Koa 实现**：统一 Koa 中间件：POST/PUT/DELETE 请求校验 `X-CSRF-Token` Header 或 Body Token

**可能的追问**：
- JWT 放在 Cookie 中时，如何防止 CSRF？（SameSite + CSRF Token 或双重 Cookie）
- 什么是 JSONP 劫持？它和 CSRF 有什么关系？
- 纯前端 SPA 中，如果 Token 放在 localStorage，CSRF 攻击还成立吗？

---

### Q24：内部微服务之间的 API 调用如何做身份认证？JWT + 签名验签 vs mTLS 哪个更适合你们 Nacos 集群内的服务间通信？

**考察点**：服务间认证、安全通信

**答案要点**：
- **JWT + 签名验签**：
  - 服务 A 调用服务 B，携带 `Authorization: Bearer <jwt>`
  - JWT 包含调用方服务身份（service_id）、调用方角色、有效期
  - 服务 B 用共享密钥或公钥验签 JWT
  - 额外签名：请求体 + timestamp + nonce，防重放攻击
  - 优点：简单，与 HTTP 协议解耦，可透传多层
- **mTLS（双向 TLS）**：
  - 服务双方都有证书（服务证书 + CA 根证书）
  - 双方互相验证证书，实现双向身份认证
  - 通信全程加密，防止中间人攻击
  - 优点：安全等级更高，Kubernetes/Istio 原生支持（Sidecar 自动处理）
- **SOC 平台方案选择**：
  - **K8s 集群内**：推荐 **mTLS**（Istio/Linkerd 自动 mTLS，无需应用层代码改动，零信任网络）
  - **跨 K8s 集群或非 K8s 环境**：使用 **JWT + HMAC 签名**（应用层控制，灵活可审计）
  - Nacos 集群内的服务注册/心跳：**Nacos 自带 SDK 鉴权**（accessToken）

**可能的追问**：
- mTLS 的证书如何管理？SPIFFE/SPIRE 在 K8s 中如何实现自动证书轮换？
- 服务 A → 服务 B → 服务 C 的链式调用，如何透传调用方身份？（JWT 字段透传 + 签名链）
- 如何防止重放攻击？（timestamp + nonce + Redis 记录已用 nonce）

---

### Q25：接口限流在网关层实现和业务层实现有什么区别？令牌桶和滑动窗口在 Koa 中分别怎么实现？

**考察点**：限流架构、算法实现

**答案要点**：
- **网关层 vs 业务层限流**：
  | 维度 | 网关层限流 | 业务层限流 |
  |------|----------|-----------|
  | 粒度 | 粗粒度（IP/用户/token） | 细粒度（功能/接口级别） |
  | 性能影响 | 最早拦截，性能好 | 资源已消耗后限流，开销大 |
  | 实时性 | 全局统一 | 可自定义 |
  | 典型场景 | 防爬虫、DDoS 防护 | 热点接口保护、费用控制 |
- **令牌桶 Koa 实现**（Redis + Lua 原子化）：
  ```lua
  -- Redis Lua 脚本（令牌桶）
  local key = KEYS[1]
  local rate = tonumber(ARGV[1])      -- 每秒产生令牌数
  local capacity = tonumber(ARGV[2]) -- 桶容量
  local now = tonumber(ARGV[3])
  local requested = tonumber(ARGV[4]) -- 请求消耗令牌

  local data = redis.call('HMGET', key, 'tokens', 'last_time')
  local tokens = tonumber(data[1]) or capacity
  local last_time = tonumber(data[2]) or now

  local elapsed = math.max(0, now - last_time)
  local to_add = elapsed * rate
  tokens = math.min(capacity, tokens + to_add)

  if tokens >= requested then
    tokens = tokens - requested
    redis.call('HMSET', key, 'tokens', tokens, 'last_time', now)
    redis.call('EXPIRE', key, math.ceil(capacity / rate) + 1)
    return 1  -- 允许
  else
    return 0  -- 拒绝
  end
  ```
- **滑动窗口 Koa 实现**：Redis Sorted Set，用时间戳做 score，请求 ID 做 member，窗口内 count > limit 则拒绝
- **告警推送接口限流策略**：`/api/alert/push` 限流为 1000 QPS/实例，触发限流返回 `429 Too Many Requests` 并在响应头返回 `Retry-After`

**可能的追问**：
- 为什么 Lua 脚本能保证限流的原子性？（Redis 单线程执行 Lua 脚本）
- 滑动窗口比令牌桶精度高在哪里？
- 如果 Redis 挂了，限流系统如何降级？（本地限流兜底或"fail open"）
