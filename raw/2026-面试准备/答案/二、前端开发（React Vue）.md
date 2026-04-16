# 二、前端开发（React / Vue） · 答案

> 本章共 33 题，覆盖前端基础（Q1-Q10）、React 基础（Q11-Q18）、React 工程化（Q19-Q25）、Vue（Q26-Q29）、前端工程进阶（Q30-Q33）

---

## 面试官快速索引

| 题号 | 核心考察点 | 难度 | 追问深度 |
|------|-----------|------|---------|
| Q1 | JS 数据类型、typeof vs instanceof | ⭐⭐ | 可深问 Symbol/BigInt |
| Q2 | 作用域、var/let/const | ⭐⭐ | 可深问暂时性死区 |
| Q3 | 闭包、内存泄漏 | ⭐⭐⭐ | 可深问 v8 垃圾回收 |
| Q4 | this 指向、call/apply/bind | ⭐⭐⭐ | 可深问箭头函数特点 |
| Q5 | 事件循环、微任务 vs 宏任务 | ⭐⭐⭐ | 可深问 Node.js 事件循环 |
| Q6 | 深拷贝 vs 浅拷贝 | ⭐⭐ | 可深问循环引用处理 |
| Q7 | CSS 盒模型 | ⭐⭐ | 可深问 margin collapse |
| Q8 | Flex 布局核心概念 | ⭐⭐ | 可深问 align-content |
| Q9 | CSS Grid vs Flexbox | ⭐⭐ | 可深问 Grid 命名区域 |
| Q10 | BFC 原理与作用 | ⭐⭐⭐ | 可深问 IFC |
| Q11 | 虚拟 DOM + Diff + key | ⭐⭐⭐ | 可深问 Fiber 架构 |
| Q12 | useState vs useReducer | ⭐⭐ | 可深问 dispatch 闭包 |
| Q13 | useEffect 执行时机 | ⭐⭐⭐ | 可深问 cleanup 依赖 |
| Q14 | Fiber 架构、可中断渲染 | ⭐⭐⭐⭐ | 可深问双缓冲 |
| Q15 | React 性能优化三剑客 | ⭐⭐⭐ | 可深问 profiling 工具 |
| Q16 | TS 类型基础 | ⭐⭐ | 可深问结构类型系统 |
| Q17 | 泛型 Generic | ⭐⭐⭐ | 可深问逆变/协变 |
| Q18 | unknown vs any vs never | ⭐⭐⭐ | 可深问类型守卫 |
| Q19 | UmiJS 路由模式 | ⭐⭐ | 可深问运行时路由 |
| Q20 | AntD vs TDesign | ⭐⭐ | 可深问主题定制 |
| Q21 | 前端国际化 | ⭐⭐ | 可深问动态加载 |
| Q22 | 前端迁移重构挑战 | ⭐⭐⭐ | 可深问灰度策略 |
| Q23 | Webpack 构建流程 | ⭐⭐⭐ | 可深问 HMR 原理 |
| Q24 | Vite vs Webpack | ⭐⭐⭐ | 可深问依赖预构建 |
| Q25 | 前端性能优化手段 | ⭐⭐ | 可深问 Lighthouse |
| Q26 | Vue3 Composition API vs React Hooks | ⭐⭐⭐ | 可深问 setup 特点 |
| Q27 | Vue 响应式原理 | ⭐⭐⭐ | 可深问 ref vs reactive |
| Q28 | Web Worker + FFmpeg.wasm | ⭐⭐⭐⭐ | 可深问 OffscreenCanvas |
| Q29 | keep-alive 原理 | ⭐⭐⭐ | 可深问 max 缓存 |
| Q30 | 微前端 qiankun 沙箱 | ⭐⭐⭐⭐ | 可深问样式隔离 CSS Modules |
| Q31 | Monorepo 架构 | ⭐⭐⭐ | 可深问 changeset |
| Q32 | 前端安全 CSP/CSRF | ⭐⭐⭐ | 可深问 JSONP 劫持 |
| Q33 | 前端监控与 CWV | ⭐⭐⭐ | 可深问 PerformanceObserver |

---

## 2.1 前端基础（JS / CSS）

### Q1：JavaScript 的数据类型有哪些？原始类型和引用类型在内存中有什么区别？`typeof` 和 `instanceof` 的区别？

**考察点**：JS 类型系统、内存模型、类型判断

**答案要点**：
- **原始类型（7种）**：`number`、`string`、`boolean`、`undefined`、`null`、`symbol`、`bigint`
- **引用类型**：对象 `object`（含 Array、Function、Date、RegExp 等）
- **内存区别**：
  - 原始类型存在**栈**中，占用固定大小（NaN/null/undefined/boolean 各 1 个机器字，其他按值大小）
  - 引用类型在**堆**中存储实际数据，栈中只存指向堆的指针（引用地址）
  - 原始类型赋值是值拷贝，引用类型赋值是引用拷贝（两者指向同一堆对象）
- **`typeof` vs `instanceof`**：
  - `typeof`：返回字符串，适合判断原始类型（`'string'/'number'/'boolean'/'undefined'/'function'/'object'`），但 `typeof null === 'object'` 是历史 bug
  - `instanceof`：基于原型链，判断对象是否为某构造函数的实例，适合判断引用类型，但无法判断原始包装类型
  - **判断 null 可靠方法**：`Object.prototype.toString.call(x) === '[object Null]'`

**可能的追问**：
- `Symbol` 有哪些应用场景？（对象属性键、防止命名冲突、Symbol.iterator）
- 为什么 ES6 要引入 `BigInt`？和 `number` 的区别是什么？
- `null == undefined` 返回 true，但 `null === undefined` 返回 false，为什么？

---

### Q2：JavaScript 的作用域（全局/函数/块级）和作用域链是什么？`var`、`let`、`const` 的区别？

**考察点**：作用域模型、作用域链查找、变量提升

**答案要点**：
- **作用域种类**：
  - **全局作用域**：最外层，浏览器中 `window` 对象，所有函数可访问
  - **函数作用域**：函数内部，`var` 声明的变量受其约束
  - **块级作用域**：`let`/`const` 在 `{ }` 内有效
- **作用域链**：当查找变量时，从当前作用域开始向上逐层查找，直到全局作用域，形成链式结构。内部函数可以访问外部变量，外部无法访问内部
- **`var` vs `let` vs `const`**：
  | 特性 | var | let | const |
  |------|-----|-----|-------|
  | 作用域 | 函数级 | 块级 | 块级 |
  | 变量提升 | ✅（初始化为 undefined） | ✅（暂时性死区 TB TD） | ✅（暂时性死区） |
  | 重复声明 | ✅ 允许 | ❌ 报错 | ❌ 报错 |
  | 重新赋值 | ✅ | ✅ | ❌ |
  | 暂时性死区 | 无 | 有（TDZ） | 有（TDZ） |
- **暂时性死区（TDZ）**：在 `let`/`const` 声明前访问变量会报错 ReferenceError，此时变量处于 TDZ 中

**可能的追问**：
- 什么是变量提升？`console.log(a)` 和 `var a = 1` 的执行结果是什么？
- 闭包是如何利用作用域链访问外部变量的？
- 为什么 for 循环中用 `var` 和 `setTimeout` 会产生奇怪行为？`let` 如何解决这个问题？

---

### Q3：什么是闭包？闭包的应用场景（防抖/节流/私有变量）？闭包会造成内存泄漏吗？为什么？

**考察点**：闭包原理、作用域链、内存管理

**答案要点**：
- **闭包定义**：函数 + 其词法环境的组合。即使外部函数已返回，内部函数仍能引用外部函数的变量
- **原理**：内部函数持有外部函数作用域链的引用（closure 对象），即使外部函数栈帧已出栈，堆中的 closure 变量仍被保留
- **应用场景**：
  - **防抖（debounce）**：闭包保存 timer，下次触发清空旧 timer，实现"等待 N 秒无新事件才执行"
  - **节流（throttle）**：闭包保存 lastTime，限制高频事件执行频率
  - **私有变量**：模块模式 IIFE 创建私有作用域，闭包暴露 getter/setter
  - **函数柯里化**：每层返回新函数，闭包保存上层参数
- **内存泄漏风险**：
  - **会**：如果闭包引用了 DOM 元素或大量数据，且没有主动释放，闭包会阻止这些对象被 GC 回收
  - **常见场景**：事件监听器注册后未移除、定时器未清理、Map/Set 持有关键字引用
  - **解决**：主动 `removeEventListener`、清空定时器、置空引用

**可能的追问**：
- V8 引擎如何处理闭包中的变量？是所有外部变量都被捕获还是只有用到的？
- 什么是"闭包导致的内存泄漏"和"闭包引起的引用计数增加"？
- React 中的 useCallback 和 useMemo 依赖闭包解决什么问题？

---

### Q4：`this` 的指向规则是什么？bind/call/apply 的区别？在 React 类组件和箭头函数中 `this` 的表现有什么不同？

**考察点**：this 绑定规则、手动绑定方法、React 组件中的 this

**答案要点**：
- **this 四种绑定规则（优先级从高到低）**：
  1. **new 绑定**：`new Foo()` 中 `this` 指向新创建的对象
  2. **显式绑定**：`call/apply/bind` 绑定，优先级高于隐式绑定
  3. **隐式绑定**：`obj.method()` 中 `this` 指向 `obj`
  4. **默认绑定**：`fn()` 非严格模式下指向 `window`，严格模式下为 `undefined`
- **call vs apply vs bind**：
  - `call(thisArg, arg1, arg2, ...)`：立即调用，参数逐个传入
  - `apply(thisArg, [args])`：立即调用，参数以数组传入
  - `bind(thisArg, ...args)`：返回新函数（不立即调用），永久绑定 this，部分柯里化
- **React 中的表现**：
  - **类组件**：render 方法中的 `this` 默认 undefined（ES6 class 默认严格模式），需要 `.bind(this)` 或箭头函数或 public class field
  - **箭头函数**：箭头函数没有自己的 `this`，它继承定义时所在上下文的 `this`，天然绑定到组件实例

**可能的追问**：
- `new Function('this.x = 1')` 的 this 指向什么？
- React 的合成事件（SyntheticEvent）中，事件处理器中的 this 为什么经常需要 bind？
- 什么是"丢失的 this"？`.bind(this)` 和箭头函数两种解决方案各有什么优劣？

---

### Q5：JavaScript 的事件循环（Event Loop）机制？`setTimeout(fn, 0)` 和 `Promise.resolve().then(fn)` 的执行顺序？

**考察点**：事件循环、微任务 vs 宏任务、执行顺序

**答案要点**：
- **事件循环流程**：
  1. 执行**同步代码栈**（call stack）
  2. **微任务队列**（Microtask Queue）：Promise.then、MutationObserver、queueMicrotask
  3. **宏任务队列**（Macrotask Queue）：setTimeout、setInterval、I/O、UI rendering、setImmediate（Node）
  4. 每次执行完一个宏任务后，清空**所有微任务**，再取下一个宏任务
  5. 重复 2-4
- **setTimeout(fn, 0) vs Promise.then(fn)**：
  - `setTimeout(fn, 0)` → **宏任务**，下一轮事件循环执行
  - `Promise.resolve().then(fn)` → **微任务**，本轮事件循环末尾立即清空
  - 所以 Promise.then 总是先于 setTimeout 执行
- **微任务优于宏任务的原因**：微任务队列在每个宏任务结束后立即清空，保证高优先级任务（如 Promise 回调）不被阻塞

**可能的追问**：
- `async/await` 和 Promise 的关系是什么？`await` 后面的代码相当于微任务还是同步代码？
- Node.js 的事件循环和浏览器有什么区别？`setImmediate` vs `process.nextTick`？
- `queueMicrotask` 和 `Promise.resolve().then` 的区别是什么？

---

### Q6：什么是深拷贝和浅拷贝？`JSON.parse(JSON.stringify())` 的局限性？`structuredClone` 和 `lodash.cloneDeep` 的区别？

**考察点**：拷贝原理、循环引用处理、工具库对比

**答案要点**：
- **浅拷贝**：只复制对象的第一层属性，引用类型属性仍指向原对象（`Object.assign`、`{...obj}`、`array.slice()`）
- **深拷贝**：递归复制所有层级，包括嵌套的对象和数组，新旧对象完全独立
- **`JSON.parse(JSON.stringify())` 局限性**：
  - ❌ 无法拷贝 `undefined`、`symbol`、函数
  - ❌ 无法拷贝 `Date`（变成字符串）、`RegExp`、`Error`
  - ❌ 无法处理循环引用（抛出 `TypeError: cyclic object value`）
  - ❌ `BigInt` 抛出错误
  - ❌ `Map/Set` 转为空对象
  - ✅ 优点：原生支持，无需依赖
- **`structuredClone`**（浏览器原生 API，`globalThis.structuredClone`）：
  - ✅ 支持循环引用
  - ✅ 支持大部分内置类型（Date/Map/Set/ArrayBuffer/BigInt）
  - ❌ 不支持函数、Error、DOM 节点
- **`lodash.cloneDeep`**：
  - ✅ 最完善，支持循环引用、函数、Symbol key
  - ✅ 可自定义拷贝策略
  - ❌ 需要引入库（tree-shaking 可优化）

**可能的追问**：
- 为什么 JSON.stringify 会丢失 undefined 和函数？
- 如何手写一个支持循环引用的深拷贝？（WeakMap 记录已拷贝对象）
- `structuredClone` 的性能比 lodash.cloneDeep 好还是差？为什么？

---

### Q7：CSS 盒模型是什么？`content-box` 和 `border-box` 的区别？移动端适配为什么要用 `border-box`？

**考察点**：盒模型原理、box-sizing、布局计算

**答案要点**：
- **盒模型构成**（从内到外）：`content`（内容）→ `padding`（内边距）→ `border`（边框）→ `margin`（外边距）
- **`content-box`（W3C 标准盒模型）**：
  - 元素宽度/高度 = content（不包含 padding/border/margin）
  - 实际占宽 = content + padding + border + margin
- **`border-box`（IE 盒模型）**：
  - 元素宽度/高度 = content + padding + border（margin 单独计算）
  - 实际占宽 = width（含 padding + border） + margin
- **移动端用 border-box 的原因**：
  - 元素宽高固定时，加上 padding/border 不影响布局计算（尤其是百分比宽度 + padding 的场景）
  - 移动端大量使用 `width: 50%` + `padding`，border-box 下直接是 50% 不额外撑大
  - 统一了 `width` 的"视觉大小"语义，不需要用 `calc()` 手动减 padding
  - 常用 CSS reset：`*, *::before, *::after { box-sizing: border-box; }`

**可能的追问**：
- margin 在什么情况下会合并（margin collapse）？合并的规则是什么？
- `box-sizing: padding-box` 为什么没有被广泛使用？
- Flex 项目中的 `flex-basis` 和盒模型的关系是什么？

---

### Q8：Flex 布局的核心概念（主轴/交叉轴）是什么？`justify-content`、`align-items`、`flex-grow/shrink` 各自的作用？

**考察点**：Flex 布局核心概念、属性作用

**答案要点**：
- **核心概念**：
  - **主轴（Main Axis）**：Flex 容器默认水平方向（`flex-direction: row`），项目沿主轴排列
  - **交叉轴（Cross Axis）**：垂直于主轴，主轴为 row 时交叉轴为 column
  - 项目在主轴上的排列由 `justify-content` 控制，在交叉轴上由 `align-items` 控制
- **属性作用**：
  - **`justify-content`**（主轴对齐）：`flex-start` | `flex-end` | `center` | `space-between` | `space-around` | `space-evenly`
  - **`align-items`**（交叉轴对齐）：`stretch`（默认）| `flex-start` | `flex-end` | `center` | `baseline`
  - **`flex-grow`**：项目在**主轴有余量时**如何扩展，数值表示比例（默认为 0 不扩展）
  - **`flex-shrink`**：项目在**主轴空间不足时**如何收缩，数值表示比例（默认为 1，0 则不收缩）
- **flex 简写**：`flex: grow shrink basis`，常用 `flex: 1` = `flex: 1 1 0%`

**可能的追问**：
- `align-content` 和 `align-items` 有什么区别？什么情况下 `align-content` 才生效？
- `flex-wrap: wrap` 时，`flex-basis`/`width`/`flex-grow` 三者的优先级是什么？
- 如何用 Flex 实现垂直居中？两行代码是什么？

---

### Q9：CSS Grid 和 Flexbox 的适用场景？两栏布局（左侧固定 + 右侧自适应）用 Grid 怎么写？

**考察点**：Grid vs Flexbox 场景区别、Grid 语法

**答案要点**：
- **Flexbox 适用场景**：一维布局（单行或单列）、内容驱动的排列、导航栏、卡片列表、水平/垂直居中
- **Grid 适用场景**：二维布局（同时控制行和列）、页面整体布局、需要精确控制对齐的复杂布局
- **两栏布局（左侧固定 300px + 右侧自适应）Grid 写法**：
```css
.container {
  display: grid;
  grid-template-columns: 300px 1fr;  /* 左侧固定 300px，右侧占剩余全部 */
  gap: 0;
}
```
- **Flexbox 等效写法**：
```css
.container {
  display: flex;
}
.sidebar { width: 300px; flex-shrink: 0; }
.main { flex: 1; }
```

**可能的追问**：
- `grid-template-columns: repeat(auto-fill, minmax(200px, 1fr))` 是什么意思？适合什么场景？
- Grid 的命名网格线和隐式网格是什么？
- 如何用 Grid 实现圣杯布局（双栏+header+footer）？

---

### Q10：什么是 BFC（块格式化上下文）？它解决什么问题（margin 塌陷、浮动元素重叠）？

**考察点**：BFC 原理、触发条件、实际应用

**答案要点**：
- **BFC 定义**：Block Formatting Context，块级格式化上下文，Web 页面中一块独立的渲染区域，区域内、外部互不影响
- **触发 BFC 的方式**（任一）：
  - `display: inline-block / table / flex / grid`
  - `position: absolute / fixed`
  - `overflow: hidden / auto / scroll`（不为 visible）
  - `float: left / right`
- **BFC 解决的三个问题**：
  1. **Margin 塌陷**：同属一个 BFC 的两个相邻块级元素，垂直 margin 会合并（塌陷为两者中较大值）。将其中一者放入新的 BFC 可避免塌陷
  2. **浮动元素重叠**：父元素没有 BFC 时，高度塌陷（被浮动子元素撑不起来），子元素会覆盖父元素的兄弟元素。给父元素加 `overflow: hidden` 触发 BFC，则父元素高度正常
  3. **清除浮动**：`clear: both` 的元素在 BFC 中可以正确识别浮动元素边界
- **常见应用**：两栏自适应布局（右侧加 BFC 避免被左侧浮动覆盖）、防止 margin 合并

**可能的追问**：
- IFC（行内格式化上下文）和 BFC 有什么区别？
- `display: flow-root` 触发 BFC 相比 `overflow: hidden` 有什么优势？（更语义化，不会产生滚动条或裁剪内容）
- Flex 容器是 BFC 还是IFC？为什么 Flex 项目可以用 margin: auto 垂直居中？

---

## 2.2 React 基础

### Q11：React 的虚拟 DOM 和 Diff 算法的核心思想是什么？key 的作用是什么？为什么不能用 index 做 key？

**考察点**：虚拟 DOM 原理、Diff 算法、key 的作用与误区

**答案要点**：
- **虚拟 DOM**：用 JS 对象描述真实 DOM 结构（VNode），React 维护两棵虚拟 DOM 树（current/fiber），通过对比找出最小变更批量更新真实 DOM
- **Diff 算法核心思想**（三个假设 + 两层对比）：
  1. **Tree Diff**：只比较同层节点，跨层操作（移动/删除/新建）代价高 → React 建议结构稳定
  2. **Component Diff**：同一位置组件类型不同 → 销毁旧组件 + 创建新组件
  3. **Element Diff**：同一层同类型元素 → 用 `key` 区分，通过 move/insert/delete 操作复用
- **key 的作用**：
  - 帮助 React 识别哪些元素是稳定的（如数据库 ID），哪些需要重新创建/移动
  - Diff 时通过 key 匹配旧树和新树的元素，复用真实 DOM 节点
- **不能用 index 做 key 的场景**：
  - 列表中间插入/删除元素：index 改变，所有 key 都变了，React 误判为删除+重建，性能差且可能丢状态
  - 列表逆序：所有 key 都不稳定
  - ✅ 适合用 index 的场景：列表完全静态（不增删）、顺序不依赖数据

**可能的追问**：
- React 的 Diff 算法时间复杂度是多少？和 vDOM 的性能收益如何权衡？
- 什么是 Fiber？Fiber 如何让 Diff 过程可中断？
- 如果没有 key，React 会怎么处理列表更新？（按位置匹配，可能导致状态错乱）

---

### Q12：React Hooks 中 useState 和 useReducer 的区别？什么场景下应该用 useReducer 而不是 useState？

**考察点**：useState vs useReducer、状态管理复杂度

**答案要点**：
- **区别**：
  - `useState`：适合简单状态（number/string/boolean/单个对象），返回 `[state, setState]`
  - `useReducer`：适合复杂状态逻辑，`reducer(state, action) => newState`，返回 `[state, dispatch]`
- **useReducer 优势**：
  1. 状态逻辑集中：所有状态更新逻辑放在 reducer 一个函数中，测试方便
  2. 可追溯：每个 action 都是显式的，`dispatch({ type: 'INCREMENT' })` 方便调试和日志
  3. 避免回调地狱：多层嵌套更新时，不需要层层传递 setState
  4. 与 Redux DevTools 集成方便
- **选择 useReducer 的场景**：
  - 状态是复合对象，涉及多个子字段的关联更新
  - 状态更新逻辑复杂（有 if/else 分支逻辑）
  - 需要撤销/重做功能（action 历史可追溯）
  - 组件状态提升后，多个子组件需要触发同类更新

**可能的追问**：
- `useReducer` 的 reducer 函数为什么必须是纯函数？
- `useReducer` 的 `dispatch` 闭包有什么陷阱？如何解决陈旧 state 问题？
- `useState` 的函数式更新 `setState(prev => prev+1)` 和 `setState(count+1)` 有什么区别？

---

### Q13：useEffect 的执行时机和依赖数组的工作机制？如何避免 useEffect 导致的无限请求？

**考察点**：useEffect 生命周期语义、依赖数组、常见陷阱

**答案要点**：
- **useEffect 执行时机**：
  - 组件挂载（mount）后执行，**浏览器渲染完成后**才执行（不阻塞渲染）
  - 每次渲染后：根据依赖数组决定是否执行
    - 无依赖数组：每次渲染后都执行
    - 空数组 `[]`：只在挂载时执行一次（类似 componentDidMount）
    - 有依赖 `[a, b]`：a 或 b 变化时执行
  - **卸载时执行 cleanup 函数**（return 的函数）
- **无限请求的原因**：
  - 常见错误：`useEffect(() => { fetchData().then(data => setData(data)) }, [data])` → data 是响应数据，本身在 state 中，setState 后 data 变化触发再请求
  - 或者使用了 `useCallback`/`useMemo` 但依赖了不稳定的引用（对象/函数）
- **解决无限请求**：
  1. 用工厂函数或 ref 隔离响应数据：`setData(data.results)` 不直接用 `data` 作为依赖
  2. 用 `useRef` 保存请求取消标记
  3. 用 `AbortController` 取消请求
  4. 正确理解"依赖数组"——依赖的是**请求参数**（如 id），而不是响应数据

**可能的追问**：
- useEffect 的 cleanup 函数在组件卸载时一定执行吗？React 18 Strict Mode 下行为有什么变化？
- `useEffect` 里的 async 函数如何正确处理？（直接传 async 函数不行，需要在内部定义 async 函数再调用，或用 IIFE）
- 如何用 ESLint `exhaustive-deps` 规则帮助检查依赖？

---

### Q14：React 的 Fiber 架构解决了什么问题？可中断渲染的含义是什么？

**考察点**：Fiber 架构原理、时间切片、优先级调度

**答案要点**：
- **解决的问题**：
  1. **同步渲染阻塞**：React 15 及之前，递归调和（reconcile）过程不可中断，大组件树更新会长时间阻塞主线程，动画/交互卡顿
  2. **无法优先级调度**：所有更新优先级相同，无法优先处理用户交互
- **Fiber 架构核心**：
  - 将渲染工作拆分成**Fiber 节点**（每个 React 元素对应一个 Fiber），形成链表结构
  - **可中断渲染**：Fiber 节点处理完成后，检查是否有更高优先级任务（如用户输入），有则**中断当前工作**，先处理高优先级任务，之后再**恢复**继续
  - **时间切片**：每个 Fiber 节点处理完后，检测剩余时间（deadline），超时则让出主线程（`scheduler.yield`），保证动画帧不被抢占
- **Fiber 双缓冲**：在内存中构建完整的 workInProgress 树，完成后一次性替换 current 树，避免半成品渲染

**可能的追问**：
- Fiber 的优先级队列是如何实现的？（lanes/ expirationTime）
- React 18 的 `useTransition` 是如何利用 Fiber 可中断特性的？
- 什么是"饥饿问题"（starvation）？低优先级更新在高优先级持续插入时如何处理？

---

### Q15：React 中如何做性能优化？memo、useMemo、useCallback 分别在什么场景使用？

**考察点**：React 性能优化手段、memo 化原理、使用边界

**答案要点**：
- **`React.memo`**：包装函数组件，返回**浅比较 props 后的记忆化组件**
  - 适用：组件频繁渲染但 props 变化少（如列表项、纯展示组件）
  - 注意：只比较 props，不比较 context/state；可用第二个参数自定义比较函数
- **`useMemo`**：缓存**计算结果**（expensive calculation）
  - 适用：复杂计算（排序/过滤/正则）、基于 state 派生的数据
  - 注意：过度使用反而增加内存开销，简单的四则运算不需要
- **`useCallback`**：缓存**函数引用**
  - 适用：传给子组件的回调（防止子组件因函数引用变化而重新渲染）、作为 useEffect 的稳定依赖
  - `useCallback(fn, deps)` = `useMemo(() => fn, deps)`
- **何时不用**：
  - 组件本身渲染很快（<1ms），memo 化开销反而更大
  - props 是原始类型，memo 没有意义
  - 盲目 memo 化所有组件，代码可读性差

**可能的追问**：
- `React.memo` 的浅比较有什么坑？深层对象相等性怎么判断？
- 什么是"反模式"——在 useEffect 中定义函数然后依赖它？
- `useTransition` 和 `useDeferredValue` 在性能优化中起什么作用？

---

### Q16：TypeScript 的基础类型（number/string/boolean/array/tuple/enum）和自定义类型（interface/type）的区别？

**考察点**：TS 类型基础、interface vs type 区别

**答案要点**：
- **基础类型**：
  - `number`（含整数和浮点数）、`string`、`boolean`、`null`、`undefined`、`symbol`、`bigint`
  - `array`：`number[]` 或 `Array<number>`
  - `tuple`：`[string, number]`，固定长度、固定位置类型
  - `enum`：`enum Color { Red, Green, Blue }`，编译为反向映射对象（数字 enum），或 `const enum` 编译时内联
- **interface vs type**：
  | 特性 | interface | type |
  |------|-----------|------|
  | 声明合并 | ✅（自动合并同名 interface） | ❌（重复定义报错） |
  | 扩展 | `extends` | `&` 交叉类型 |
  | 复杂类型 | 适合描述对象结构 | 适合别名、联合、映射类型 |
  | 计算属性 | ❌ | ✅（`[K in keyof T]`） |
  | 原始类型别名 | ❌ | ✅ |
- **原则**：描述对象/类结构用 interface，需要灵活性（联合/映射）用 type

**可能的追问**：
- `interface I extends A, B {}` 和 `type T = A & B` 有什么区别？
- `enum` 和 `const enum` 的区别？为什么更推荐 `const enum`？
- 什么是声明文件（.d.ts）？为什么 Node.js 类型定义用 .d.ts 而不是 .ts？

---

### Q17：什么是泛型（Generic）？`Array<T>`、`<T extends keyof U>`、条件类型 `T extends U ? X : Y` 分别用在什么场景？

**考察点**：TS 泛型、条件类型、映射类型

**答案要点**：
- **`Array<T>` 泛型**：`T` 是类型参数，定义时不固定，使用时指定
  ```typescript
  function first<T>(arr: T[]): T | undefined
  ```
- **`<T extends keyof U>`**：约束泛型参数必须是另一个对象类型的键
  ```typescript
  function getProperty<T, K extends keyof T>(obj: T, key: K): T[K]
  ```
  - `keyof T` 提取 T 的所有键（联合类型）
  - `T[K]` 索引访问获取键对应值的类型
- **条件类型** `T extends U ? X : Y`：
  - 分布律：在泛型中使用时，如果 T 是联合类型，会分别作用于每个成员
  ```typescript
  type IsString<T> = T extends string ? 'yes' : 'no'
  type A = IsString<string | number>  // 'yes' | 'no'
  ```
  - 场景：类型守卫（`Extract`/`Exclude`）、Promise 返回值提取（`Awaited`）

**可能的追问**：
- 什么是 `infer` 关键字？在 `ReturnType<T>` 中如何用 infer 推导类型？
- `Partial<T>`、`Required<T>`、`Readonly<T>`、`Pick<T, K>` 是如何实现的？
- 什么是协变（covariant）和逆变（contravariant）？TypeScript 中的函数类型默认是双变（bivariant）吗？

---

### Q18：TypeScript 的 `unknown` 和 `any` 的区别？`never` 类型用在什么场景？

**考察点**：TS 类型安全层级、never 类型

**答案要点**：
- **`any` vs `unknown`**：
  | 特性 | any | unknown |
  |------|-----|---------|
  | 类型安全 | 无类型检查，可随意访问 | 必须进行类型收窄（narrowing）才能使用 |
  | 赋值 | 任意值可赋给 any | 任意值可赋给 unknown |
  | 赋值给其他类型 | any 可赋给任意类型 | unknown 必须先收窄才能赋给其他类型 |
  | 使用属性/方法 | 直接可用 | 编译器报错，需 typeof/instanceof 收窄 |
  - `unknown` 是 TS 的"类型安全 any"，强制使用者做类型检查
- **`never` 类型**：
  - 表示**永远不可能存在**的值
  - 应用场景：
    1. 函数永远不返回（死循环/抛异常）：`function fail(msg: string): never { throw new Error(msg) }`
    2. 穷尽检查（exhaustive check）：switch 穷尽联合类型所有成员
    3. 收窄到 never 表示类型不可能到达（如类型范围缩小到空集）
  - `never` 是所有类型的子类型，可以赋给任意类型，但反过来不行

**可能的追问**：
- `void` 和 `never` 的区别是什么？
- `unknown[]` 和 `any[]` 有什么区别？`Record<string, unknown>` 的实际用途？
- 如何用 `never` 实现类型安全的 exhaustive switch？

---

## 2.3 React 工程化

### Q19：UmiJS 的约定式路由和配置式路由有什么区别？你在 SOC 平台项目中为什么选择 UmiJS？

**考察点**：UmiJS 路由机制、约定优于配置

**答案要点**：
- **约定式路由**：
  - 目录即路由：`src/pages/` 下的文件结构自动映射为路由
  - `pages/index.tsx` → `/`，`pages/users/List.tsx` → `/users/list`
  - 文件名带 `_` 前缀不生成路由（如 `_layout.tsx`），带 `.` 的为动态路由（如 `$id.tsx`）
  - 优点：零配置，文件结构即路由，开发体验好
- **配置式路由**：
  - 显式在 `config/routes.ts` 中声明路由配置
  - 适合复杂路由（权限控制、多级嵌套、特殊跳转）
  - 完全掌控路由行为
- **SOC 平台选择 UmiJS 的原因**：
  1. **约定式路由**降低心智负担，前端同学按文件组织即可
  2. 内置完善的**权限路由**（wrapper/access）机制，直接对接 RBAC
  3. MFSU（模块联邦）加速本地开发
  4. 生态完善：Ant Design / dva / 国际化开箱即用

**可能的追问**：
- UmiJS 的 MFSU（Module Federation Speed Up）原理是什么？如何减少依赖构建时间？
- UmiJS 如何做代码分割（lazy load）？`dynamic` 和 React.lazy 有什么区别？
- UmiJS 4 和 UmiJS 3 在架构上有什么重大变化？

---

### Q20：Ant Design 和 TDesign 的主要差异是什么？在同一个项目中混用两套组件库会有什么问题？你们是怎么处理的？

**考察点**：UI 组件库选型、设计系统一致性

**答案要点**：
- **主要差异**：
  | 维度 | Ant Design | TDesign |
  |------|-----------|---------|
  | 设计语言 | Material Design 风格，蓝色主色调 | 腾讯自研，蓝色/青色多主题 |  |
  | 生态 | 生态极大（蚂蚁金服背书），社区活跃 | 较新，腾讯内部使用，文档相对少 |
  | React 支持 | 完善 | 完善（桌面端） |
  | Vue 支持 | ✅（ant-design-vue） | ✅（更原生） |
  | 主题定制 | Less 变量覆盖，定制灵活 | Design Token，CSS 变量，数字化设计 |
  | 组件数量 | 非常丰富 | 相对较少，部分组件缺失 |
- **混用问题**：
  1. **样式冲突**：两套库的 CSS 类名可能冲突（reset/button/table 样式覆盖）
  2. **视觉不一致**：按钮、输入框等基础组件风格不同，同页面出现两套设计语言
  3. **打包体积**：两套库都打包，增加产物大小
  4. **维护成本**：API 差异导致学习成本翻倍
- **处理方式**：
  - 统一风格后选一套为主，特殊场景按需引入另一套（如 TDesign 的数字输入框更好看）
  - 用 CSS Modules / BEM 隔离，或用 Shadow DOM 完全隔离

**可能的追问**：
- 如何实现 Ant Design 的主题定制？（ConfigProvider、dynamicTheme）
- 组件库的按需加载（babel-plugin-import）是如何实现的？
- 如果 UI 设计稿和组件库不一致，如何低成本对齐？（Styled Components / CSS-in-JS 改造）

---

### Q21：React 项目如何做国际化（i18n）？你们在迁移 TCE 网关时，国际化改造遇到了哪些坑？

**考察点**：前端国际化方案、react-intl vs react-i18next

**答案要点**：
- **常见方案**：
  - **react-intl**（FormatJS）：基于 ICU MessageFormat，支持复数/性别/日期格式化，雅虎出品
  - **react-i18next**：基于 i18next，功能强大，支持命名空间、分语言加载、插件系统
  - **两者选择**：大项目推荐 `i18next`，社区活跃，工具链完善（i18next-locize-backend）
- **i18n 改造步骤**：
  1. 提取所有硬编码文本为 key（`i18n.t('key.path')`）
  2. 建立语言文件 `locales/zh-CN.json`、`locales/en-US.json`
  3. 切换语言时用 `i18n.changeLanguage('en')`
  4. 动态加载语言包（按需加载减少初始体积）
- **TCE 迁移中的坑**：
  1. **命名冲突**：不同模块用了相同的 i18n key，后加载覆盖前者 → 统一命名空间（如 `module.feature.key`）
  2. **SSR 国际化**：服务端和客户端语言状态同步问题 → 在 Server 端注入 i18next state 到 HTML
  3. **Hook 获取 t 函数**：Class 组件用 HOC `withNamespaces`，Hooks 组件用 `useTranslation`

**可能的追问**：
- 什么是 ICU MessageFormat？如何处理复数和性别？
- 国际化文件中 key 的命名规范是什么？（path 形式 vs 纯英文 key）
- 动态语言切换时，页面刷新不丢失语言状态怎么实现？（URL 参数或 Cookie 同步）

---

### Q22：你在 SOC 平台主导了前端整体重构，从旧架构迁移到 UmiJS + TCE 网关的过程中，最大的技术挑战是什么？如何保证迁移过程中不影响线上业务？

**考察点**：前端重构风险控制、灰度发布、兼容性

**答案要点**：
- **最大技术挑战**：
  1. **路由映射**：旧路由格式与 UmiJS 不兼容（如 `/v1/alerts/:id` vs UmiJS 的 `/alerts/:id`），需要设计兼容层
  2. **状态管理迁移**：dva model 从 0.x 到 4.x API 不兼容，state 结构不同
  3. **API 层差异**：TCE 网关的请求/响应格式与旧 Koa 层有差异（如 code 字段、错误处理）
  4. **多团队并行开发**：迁移期间两套代码同时存在，合并冲突多
- **不影响线上方案**：
  1. **平行开发期**：新旧系统同时运行，通过**路由灰度**按用户切流
  2. **功能开关**：用 `umi-plugin-antd-evil-flag` 或后端配置，对单个功能做开关
  3. **A/B 测试**：用户 ID hash 后路由到新旧系统，逐步扩大新系统比例
  4. **接口兼容层**：TCE 网关代理旧 Koa 接口，保证前端无需大改
  5. **充分测试**：核心用例人工回归 + 自动化测试覆盖

**可能的追问**：
- 如何设计前后端分离的平滑迁移策略？后端接口兼容性如何保障？
- 前端重构过程中如何保证 TypeScript 类型安全？有类型迁移工具吗？
- 灰度发布时，Cookie/Session 如何处理？两套系统 session 格式不同怎么办？

---

### Q23：Webpack 的构建流程（初始化 → 编译 → 输出）是什么？Loader 和 Plugin 的区别？Tree-shaking 的原理？

**考察点**：Webpack 构建机制、Tapable 插件系统

**答案要点**：
- **Webpack 构建流程**（基于 Tapable 插件化架构）：
  1. **初始化**：解析配置（entry/output/loader/plugin），创建 Compiler 对象
  2. **编译（Compilation）**：
     - 从 entry 开始，使用 loader 链处理每个模块（read → loader chain → transform）
     - 解析依赖（`require`/`import`），构建模块依赖图（Module Graph）
     - 遇到 plugin 钩子（emit/optimize/seal），执行插件逻辑
  3. **输出（Emitting）**：
     - 根据 chunk 图生成文件（JS/CSS/HTML）
     - 调用 plugins 处理输出（如 HtmlWebpackPlugin 注入 JS 引用）
     - 输出到 dist 目录
- **Loader vs Plugin**：
  - **Loader**：转换器，**单文件级别**，在模块加载时对源代码做预处理（ts→js、css→js、jsx→js）
  - **Plugin**：扩展器，**构建过程级别**，在 Webpack 整个生命周期（编译/优化/输出）插入自定义逻辑
  - 两者通过 Tapable 事件流协作
- **Tree-shaking 原理**：
  - 基于 ES Module 的**静态分析**（import/export 在编译时确定，不执行代码）
  - 两个阶段：1）标记（Mark）—— 找出所有未使用的导出；2）删除（Drop）—— 移除未使用代码
  - 条件：`sideEffects: false` 告诉 Webpack 可安全删除未使用导出，`usedExports: true` 开启标记
  - 无法 Tree-shaking 的情况：`require()`、`eval()`、动态 import

**可能的追问**：
- Webpack 的 HMR（热模块替换）原理是什么？哪些类型的模块支持 HMR？
- `splitChunks` 的 `chunks: 'all'` 和 `chunks: 'async'` 有什么区别？
- Webpack 5 的 Module Federation（模块联邦）和 Tree-shaking 如何配合工作？

---

### Q24：Vite 相比 Webpack 的优势是什么？ESM 和 CommonJS 在开发阶段的区别？

**考察点**：Vite 架构原理、ESM、HMR 速度

**答案要点**：
- **核心优势**：
  | 维度 | Webpack | Vite |
  |------|---------|------|
  | 开发启动 | 全量构建依赖（慢） | 按需编译（极快） |
  | HMR | 需要重新构建相关模块 | 仅针对变更文件（毫秒级） |
  | 构建 | Rollup（生产） | esbuild/TurboPack |
  | 缓存 | 持久缓存需配置 | 自动按依赖缓存 |
- **ESM vs CommonJS**：
  - **CommonJS**：`require` 同步执行，运行时解析，无法做静态分析（`require(variable)` 合法）
  - **ESM**：`import` 静态解析（只能在顶层），编译时确定依赖关系
  - Vite 利用 ESM 静态性，在开发阶段直接用浏览器 native ESM 按需加载，无需全量打包
- **Vite 冷启动原理**：不做全量打包，只启动 dev server，浏览器请求哪个模块再实时编译哪个（native ESM + HTTP 2 并行）

**可能的追问**：
- Vite 在生产构建时用 Rollup，为什么开发阶段不用 Rollup？
- 什么是 Vite 的"依赖预构建"（esbuild）？哪些类型的 npm 包需要预构建？
- Vite 对 CommonJS 模块的处理方式是什么？（esbuild 转换为 ESM）
- TurboPack（Turbo）相比 Vite 和 Webpack 有什么优势？

---

### Q25：前端项目的性能优化有哪些手段？（代码分割、懒加载、CDN、缓存策略、图片压缩）

**考察点**：前端性能优化手段、Performance API

**答案要点**：
- **代码分割**：
  - `React.lazy(() => import('./HeavyComponent'))` + `<Suspense>`
  - Webpack `splitChunks` 分离第三方库（vendors）和业务代码
  - 路由级分割（路由懒加载）
- **懒加载**：
  - 图片懒加载：`loading="lazy"` 或 IntersectionObserver
  - 非首屏组件：动态 import
- **缓存策略**：
  - HTTP 缓存：`Cache-Control: max-age=31536000` + `immutable`（对 hash 文件名）
  - CDN 缓存：静态资源走 CDN，`Cache-Control: public, max-age`
  - 浏览器缓存：`localStorage`/`IndexedDB` 存数据
  - Service Worker：离线缓存（Workbox）
- **图片优化**：
  - 格式：`WebP` > `AVIF` > `PNG/JPEG`，小图标用 `SVG`
  - 响应式图片：`<img srcset>` 分辨率适配
  - 渐进式 JPEG（PJPEG）
- **其他手段**：
  - SSR/SSG：Next.js/Nuxt 减少首屏白屏
  - 预加载关键资源：`<link rel="preload">`
  - 减少重排/重绘：批量 DOM 操作、使用 `transform` 代替 `top/left`
  - 压缩代码：Terser / CSSnano

**可能的追问**：
- `webpack-bundle-analyzer` 如何帮助定位体积问题？
- HTTP 强缓存和协商缓存如何配合使用？Expires 和 Cache-Control 同时存在时谁优先？
- Core Web Vitals（LCP/FID/CLS）如何量化优化效果？

---

## 2.4 Vue

### Q26：Vue 3 的 Composition API 和 React Hooks 的本质区别是什么？

**考察点**：Vue 响应式 vs React 重新执行、Hooks 设计哲学

**答案要点**：
- **核心区别：响应式 vs 重新执行**
  - **React Hooks**：组件 render 后，整个函数组件重新执行，所有 Hooks 重新运行一遍，通过 `useState`/`useRef` 保存状态和副作用。**没有响应式依赖追踪**
  - **Vue 3 Composition API**：setup 只执行一次（组件创建时），响应式状态（`ref`/`reactive`）变化触发的是**依赖它的 DOM/计算属性更新**，而非重新执行整个组件函数
- **具体差异**：
  | 维度 | React Hooks | Vue 3 Composition API |
  |------|------------|----------------------|
  | 执行时机 | 每次渲染都重新执行 | setup 只执行一次 |
  | 状态位置 | useState 返回的状态在组件级别 | ref/reactive 在函数作用域 |
  | 依赖追踪 | 手动依赖数组 | Proxy 响应式自动追踪 |
  | 闭包陷阱 | 常见（依赖数组错误导致 stale closure） | 较少（computed 自动追踪） |
  | 逻辑复用 | 自定义 Hook（逻辑复用函数） | composable（逻辑复用函数） |
- **Vue Composition API 优势**：逻辑更清晰（相关逻辑集中）、更好的 TypeScript 支持、更易 tree-shaking

**可能的追问**：
- Vue 3 中 `setup` 里可以直接用 `async` 吗？有什么限制？
- `watchEffect` 和 `watch` 的区别是什么？
- `toRefs` 的作用是什么？解决了什么问题？

---

### Q27：Vue 的响应式原理？Vue 2 的 Object.defineProperty 和 Vue 3 的 Proxy 有什么区别？

**考察点**：Vue 响应式原理、Proxy vs defineProperty

**答案要点**：
- **Vue 2 响应式（Object.defineProperty）**：
  - 组件初始化时，通过 `Object.defineProperty` 递归劫持 data 的每个属性
  - 每个属性有 `get`（收集依赖，Dependency Tracking）和 `set`（触发更新，Watcher 通知）
  - 缺点：**无法检测对象属性的添加/删除**（需要 `Vue.set`）、**无法检测数组下标变化**（Vue 2 做了hack，但仍有局限）
- **Vue 3 响应式（Proxy）**：
  - 使用 `new Proxy(target, handler)` 代理整个对象，handler 中拦截 `get`/`set`/`deleteProperty` 等操作
  - 支持数组下标监听、对象属性新增/删除、Map/Set 监听
  - 性能更好，不需要对每个属性递归做劫持（懒代理：访问到才代理深层对象）
- **根本区别**：
  - `defineProperty` 是**属性级别**拦截，Proxy 是**对象级别**拦截
  - Proxy 可以拦截 `in` 操作符、`delete obj.prop`、`for...in`
  - Proxy 不污染原对象，`Reflect` 保证默认行为

**可能的追问**：
- Vue 3 的 `ref` 和 `reactive` 的区别是什么？为什么 ref 要用 `.value`？
- Vue 3 中 `watch` 监听 reactive 对象时，deep 选项需要设置吗？
- Vue 2 中数组的响应式有什么特殊处理？（重写 7 个变更方法：push/pop/splice/shift/unshift/sort/reverse）

---

### Q28：你在影音编辑审核平台项目中用了 Vue + FFmpeg.wasm，Web Worker 在这个场景中解决了什么问题？如何将大文件编辑的页面卡顿率降低 90%？

**考察点**：Web Worker 原理、FFmpeg.wasm、UI 主线程保护

**答案要点**：
- **问题**：FFmpeg.wasm 视频转码/剪辑是 CPU 密集型任务，在主线程执行会**阻塞 UI 渲染**，导致页面卡顿、交互无响应
- **Web Worker 解决方案**：
  - 将 FFmpeg 实例化在 Worker 线程中，主线程只负责向 Worker 发命令、接收结果
  - Worker 线程和主线程通过 `postMessage` / `onmessage` 通信
  - Worker 中可使用 `importScripts` 加载 FFmpeg.wasm 的 WASM 文件
- **降低卡顿率的具体做法**：
  1. **主线程**：只负责 UI 渲染和进度展示，不做计算
  2. **Worker**：所有转码/剪辑操作在 Worker 中执行
  3. **通信优化**：进度信息通过 MessageChannel 低频传输（每 N% 报告一次，避免频繁序列化）
  4. **内存管理**：大文件使用 `Transferable` 对象传输所有权，避免拷贝（`worker.postMessage(buffer, [buffer])`）
  5. **结果处理**：Worker 返回处理完毕的 Blob，主线程创建 ObjectURL 预览
- **结果**：主线程帧率从 10fps 恢复到 60fps，卡顿率降低 90%+

**可能的追问**：
- Web Worker 和 Service Worker 有什么区别？SharedWorker 的应用场景？
- `OffscreenCanvas` 在 Worker 中如何与 Canvas 通信？
- FFmpeg.wasm 的 WASM 内存限制是多少？大文件如何分块处理？

---

### Q29：Vue 的 keep-alive 组件作用是什么？ activated 和 deactivated 生命周期在什么场景下使用？

**考察点**：Vue 缓存组件、生命周期钩子

**答案要点**：
- **`keep-alive`**：Vue 内置组件，用于缓存组件实例，避免重复销毁/创建，保留组件状态
  - 被缓存的组件：不会触发 `beforeUnmount`/`unmounted`，而是触发 `deactivated`
  - 再次激活时：不会触发 `onMounted`，而是触发 `activated`
- **常用属性**：
  - `include` / `exclude`：正则或字符串数组，指定缓存/排除哪些组件（按组件名）
  - `max`：最大缓存数量，超过后销毁最久未使用的实例
- **activated / deactivated 使用场景**：
  - **activated**：组件激活时开始定时器/动画/轮询（如仪表盘自动刷新）
  - **deactivated**：组件失活时暂停定时器/动画/停止轮询，防止内存泄漏和不必要的计算
  - 场景：Tab 切换（保持滚动位置）、列表详情页（返回时不重新加载）、表单填写（切走不丢内容）

**可能的追问**：
- `keep-alive` 缓存的组件的 `data` 会保留吗？如果 props 变了会更新吗？
- `max` 属性在 Vue 3 和 Vue 2 中的行为有什么不同？
- 如何手动清除 `keep-alive` 缓存？（`vnode.key` 或 `include` 动态更新）

---

## 2.5 前端工程进阶

### Q30：微前端（Micro-Frontend）是什么？qiankun 的沙箱隔离原理（JS 隔离、CSS 隔离）是如何实现的？你们的 SOC 平台有没有考虑微前端？为什么？

**考察点**：微前端概念、qiankun 沙箱、架构决策

**答案要点**：
- **微前端概念**：将前端应用拆分为多个独立可部署的子应用，每个子应用由独立团队开发、测试、部署，最终在浏览器中组合为一个完整应用（借鉴微服务思想）
- **qiankun 沙箱隔离**：
  - **JS 隔离**（快照沙箱）：
    - 子应用运行时修改全局变量（如 `window.a = 1`），qiankun 记录"快照"（当前 window 属性）
    - 子应用卸载时恢复快照，实现隔离
    - 多子应用场景用**代理沙箱（ProxySandbox）**：为每个子应用创建独立的 `fakeWindow` Proxy，子应用的全局操作都代理到这个 fakeWindow
  - **CSS 隔离**：
    - **Shadow DOM**（严格隔离，样式完全隔离但兼容性差）
    - **动态样式表**：子应用样式自动加前缀（如 `app-xxx-button`）
    - **CSS Modules** 或 **BEM 命名规范**
    - qiankun 使用**约定样式隔离**（运行时注入 CSS 前缀）
- **SOC 平台未采用的原因**：
  1. 团队规模小（前端 3-4 人），微前端带来的复杂度不值当
  2. 模块联邦（Module Federation）能满足大部分场景且侵入性更小
  3. 微前端解决的是**团队协作边界**问题，不是**技术问题**

**可能的追问**：
- qiankun 和 single-spa 有什么关系？qiankun 在 single-spa 基础上做了什么？
- Module Federation（Webpack 5）和 qiankun 各自的适用场景是什么？
- 微前端场景下，多子应用如何共享公共依赖（如 React/antd），避免重复打包？

---

### Q31：Monorepo（Turborepo/Nx）在大型前端项目中解决了什么问题？依赖共享、子包构建缓存、变更影响分析分别怎么实现？

**考察点**：Monorepo 架构、依赖管理、构建优化

**答案要点**：
- **解决的问题**：
  1. **代码复用**：多个项目共享同一 `packages/` 中的工具库/组件库
  2. **统一版本管理**：避免各项目依赖版本不一致（依赖 hoist）
  3. **原子提交**：一个 PR 可修改多个子包（如改 API 同时改前端）
  4. **CI 优化**：只构建/测试变更的子包，而非全量构建
- **Turborepo 实现**：
  - `turbo.json` 定义 pipeline（任务依赖图），`build` 依赖 `^build`（前置子包构建完成）
  - 本地缓存：`out/.turbo`，按 task+hash 缓存构建产物
  - 远程缓存（Turbo Remote）：CI 共享缓存，加速构建
- **Nx 实现**：
  - `project.json` 定义每个子项目的任务和依赖
  - **受影响分析**（affected commands）：基于 git diff 分析哪些子包被改动，自动只构建/测试受影响子包
  - **计算缓存**：同样的输入（代码+依赖+环境）产生同样的输出缓存
- **依赖共享**：`workspace`（pnpm/yarn workspace 或 npm workspaces），包路径别名（如 `@soc/shared-ui`）

**可能的追问**：
- pnpm 的 `pnpm-workspace.yaml` 和 npm/yarn workspace 有什么区别？
- Monorepo 中，子包 A 依赖子包 B，如何保证构建顺序？（拓扑排序 `turbo run build --filter=@soc/ui`）
- Monorepo 的 CI 构建缓存命中率和代码分割如何平衡？

---

### Q32：前端安全：XSS、CSRF、点击劫持的防御手段（CSP、SameSite Cookie、X-Frame-Options）分别在前端代码层面如何落地？你们有没有做过前端安全审计？

**考察点**：前端安全防御、Content Security Policy、实际审计经验

**答案要点**：
- **XSS 防御**：
  - **CSP（Content-Security-Policy）**：在 HTTP 响应头或 `<meta>` 中设置，限制脚本来源、禁止内联脚本
    ```
    Content-Security-Policy: script-src 'self' https://cdn.example.com; object-src 'none'
    ```
  - 输入过滤（React 默认转义 HTML）、输出编码（按上下文：HTML/URL/JS/CSS）
- **CSRF 防御**：
  - **SameSite Cookie**：`SameSite=Strict/Lax`，Lax（默认）在跨域 GET 请求时不带 Cookie
  - **CSRF Token**：`response` 中注入随机 token，表单中携带，服务器校验
  - **双重提交 Cookie**：`Cookie` 和自定义 Header 双重携带 SameSite Cookie
- **点击劫持防御**：
  - **`X-Frame-Options: DENY/SAMEORIGIN`**：禁止页面在 iframe 中渲染
  - **`X-Content-Type-Options: nosniff`**：防止 MIME 类型嗅探
  - **Frame Busting**：`if (top.location !== window.location) top.location = window.location`（可被绕过）
- **SOC 平台的安全审计**：
  - 接入内部安全扫描（SAST）+ 人工代码审查
  - 关键操作（告警处理/Case 创建）加入 CSRF Token 校验
  - 所有外部数据在渲染前做 HTML 转义

**可能的追问**：
- CSP 的 `nonce` 模式是如何工作的？它和 `hash` 模式的区别？
- JSONP 劫持和 CSRF 有什么关系？如何防御？
- 前端安全审计通常检查哪些维度？（OWASP Top 10、敏感信息泄露、依赖漏洞）

---

### Q33：前端监控（性能监控、错误监控、行为监控）如何实现？Core Web Vitals（LCP/FID/CLS）对企业级后台系统有什么意义？

**考察点**：前端监控体系、Performance API、用户体验指标

**答案要点**：
- **三类监控**：
  1. **性能监控**：`PerformanceObserver` 监听 `paint`（FP/FCP）、`largest-contentful-paint`（LCP）、`layout-shift`（CLS）
     - 资源加载：`PerformanceResourceTiming` 记录 JS/CSS/图片加载时间
     - 自定义打点：`performance.mark('custom-event')`
  2. **错误监控**：全局 `window.onerror`（未捕获错误）、`window.unhandledrejection`（Promise rejection）、`Vue.config.errorHandler`
     - 捕获后上报（常用 sentry.io）含堆栈信息、用户行为轨迹（点击热力图）
  3. **行为监控**：点击热力图、页面停留时长、路由切换时间、API 请求成功率
     - 通过 `history.pushState` 劫持监听路由变化
- **Core Web Vitals 意义**：
  - **LCP（Largest Contentful Paint）**：衡量加载性能，企业后台 LCP < 2.5s 优秀，影响用户等待感知
  - **FID（First Input Delay）**：衡量交互性，FID < 100ms 优秀，影响用户操作响应感
  - **CLS（Cumulative Layout Shift）**：衡量视觉稳定性，CLS < 0.1 优秀，影响表单填写、按钮点击误点
  - **对企业的价值**：直接关联 Google SEO 排名（2021 年纳入排名因素），影响 SLA 指标

**可能的追问**：
- Sentry 是如何实现 source map 上传和错误关联的？
- 如何区分 JS 错误和 API 错误的监控策略？
- Performance API 的 `navigation.timing` 模型各字段含义是什么？
