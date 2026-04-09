# 前端工程与 React/Vue 实战手册

## Summary

本手册面向具备 React + TypeScript 实战经验、期望在面试中展现中高级前端深度的 AI 应用工程师。核心覆盖两大框架的底层原理对比（虚拟 DOM / 响应式机制）、React Hooks 体系与状态管理、性能优化工程实践、TypeScript 泛型集成、大列表场景优化、i18n 架构设计，以及 Docker/K8s 灰度部署链路。重点不是 API 记忆，而是理解 Why——为什么 React 不推荐随机 key、为什么 useEffect 的依赖数组决定了一切、Vue 3 的 Proxy 相比 defineProperty 有何本质优势。文档最后附高频面试 Q&A，适合临阵查漏补缺。

---

## 零、基础知识速查

### 0.1 HTML / CSS / JavaScript 速查

```html
<!-- HTML 基本结构 -->
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>告警监控</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <div id="root"></div>
  <script src="app.js"></script>
</body>
</html>
```

```css
/* CSS 基础 */
.container {
  display: flex;
  gap: 16px;
  padding: 20px;
  background: #1a1a2e;
  color: #eee;
}

.alert-card {
  border: 1px solid #e74c3c;
  border-radius: 8px;
  padding: 16px;
}

.alert-card.critical {
  background: #2d1f1f;
}

/* Flexbox 居中 */
.center {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* Grid 布局 */
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 16px;
}
```

```javascript
// JavaScript 基础
const alerts = [
  { id: 'A001', severity: 'P0', message: '检测到勒索软件' },
  { id: 'A002', severity: 'P2', message: '异常登录' }
];

// 箭头函数
const criticalAlerts = alerts.filter(a => a.severity === 'P0');

// 解构赋值
const { id, severity, message } = alerts[0];

// 模板字符串
const html = `<div class="alert-card">${id}: ${message}</div>`;

// Promise（异步）
fetch('/api/alerts')
  .then(res => res.json())
  .then(data => console.log(data))
  .catch(err => console.error(err));

// async/await
async function loadAlerts() {
  try {
    const res = await fetch('/api/alerts');
    const data = await res.json();
    return data;
  } catch (err) {
    console.error(err);
  }
}
```

### 0.2 React 基础

```jsx
// JSX 基本语法
function AlertCard({ alert }) {
  return (
    <div className={`alert-card ${alert.severity.toLowerCase()}`}>
      <h3>{alert.id}</h3>
      <p>{alert.message}</p>
      <span>{alert.severity}</span>
    </div>
  );
}

// 列表渲染
function AlertList({ alerts }) {
  return (
    <div className="alert-list">
      {alerts.map(alert => (
        <AlertCard key={alert.id} alert={alert} />
      ))}
    </div>
  );
}

// 状态管理（useState）
import { useState } from 'react';

function AlertPage() {
  const [alerts, setAlerts] = useState([]);
  const [filter, setFilter] = useState('ALL');

  const filtered = alerts.filter(a =>
    filter === 'ALL' || a.severity === filter
  );

  return (
    <div>
      <select onChange={e => setFilter(e.target.value)}>
        <option value="ALL">全部</option>
        <option value="P0">P0</option>
        <option value="P1">P1</option>
      </select>
      <AlertList alerts={filtered} />
    </div>
  );
}
```

### 0.3 Vue 基础

```vue
<!-- Vue 3 单文件组件 (SFC) -->
<template>
  <div class="alert-list">
    <div
      v-for="alert in filteredAlerts"
      :key="alert.id"
      :class="['alert-card', `severity-${alert.severity.toLowerCase()}`]"
    >
      <h3>{{ alert.id }}</h3>
      <p>{{ alert.message }}</p>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const alerts = ref([])
const filter = ref('ALL')

const filteredAlerts = computed(() =>
  filter.value === 'ALL'
    ? alerts.value
    : alerts.value.filter(a => a.severity === filter.value)
)

const fetchAlerts = async () => {
  const res = await fetch('/api/alerts')
  alerts.value = await res.json()
}
</script>

<style scoped>
.alert-card {
  padding: 16px;
  border-radius: 8px;
}
.severity-p0 { background: #2d1f1f; border: 1px solid #e74c3c; }
.severity-p1 { background: #2d2a1f; border: 1px solid #f39c12; }
</style>
```

### 0.4 npm / 包管理基础

```bash
# 创建 React 项目（Vite）
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install
npm run dev

# 安装依赖
npm install react-router-dom       # 路由
npm install axios                  # HTTP 客户端
npm install zustand                # 状态管理
npm install @tanstack/react-query  # 服务端状态

# TypeScript 类型
npm install -D @types/react @types/node

# 构建
npm run build                     # 生产构建
npm run preview                   # 本地预览生产构建
```

### 0.5 常见面试基础问题

**Q: 什么是虚拟 DOM？**
- 虚拟 DOM 是真实 DOM 的 JavaScript 对象表示（React/Vue 维护）
- 数据变化时，先更新虚拟 DOM，再通过 Diff 算法算出最小变更，最后批量更新真实 DOM
- 避免频繁操作真实 DOM 导致的性能问题和重排重绘

**Q: React 和 Vue 的核心区别？**
- **React**：使用 JSX（JavaScript 语法扩展），单向数据流，社区大，生态丰富
- **Vue**：使用模板语法 + 响应式系统，入门简单，上手快，两者都能完成相同任务

**Q: npm install 和 npm ci 的区别？**
- `npm install`：根据 `package.json` 安装，可修改 `package-lock.json`
- `npm ci`：完全按 `package-lock.json` 安装，CI/CD 中使用，确保一致性，更快

---

## 1. React 虚拟 DOM 与 Diff 算法

### 核心原理

React 的虚拟 DOM 是一棵 JavaScript 对象树，对真实 DOM 的每次修改都先作用在这棵内存中的树上，再通过协调（Reconciliation）算法计算出最小变更集（Patch），批量提交给浏览器。

Diff 算法基于两个前提假设进行优化：
- **不同类型的节点产生不同的树**：`<div>` 变成 `<span>` 会直接卸载旧树
- **同层节点可以通过 key 区分**：不跨层级移动节点，只在同层比较

### key 的作用机制

key 帮助 React 识别哪些元素是稳定的。正确使用 key 可以让 Diff 算法在同层比较时复用已有 DOM 节点，只做属性更新而非重新挂载。

```tsx
// ✅ 正确：使用唯一稳定的 ID
{items.map(item => (
  <li key={item.id}>{item.name}</li>
))}

// ❌ 错误：使用数组下标（列表重排时会错误复用节点）
{items.map((item, index) => (
  <li key={index}>{item.name}</li>
))}

// ❌ 错误：使用随机 key（每次渲染都认为是新节点，无法复用）
<li key={Math.random()}>{item.name}</li>
```

### 为什么不能使用随机 key

随机 key 在每次渲染时都会生成全新的值，React 无法将上一次渲染的节点与下一次对应起来，等同于告诉 React"每个节点都是新来的"。这会导致：
1. 组件状态完全丢失（useState、useRef 等状态不会保留）
2. DOM 节点被完全卸载后重新挂载，性能严重退化
3. 动画状态、表单输入值等用户数据丢失

在生产环境中，可使用 `crypto.randomUUID()`、`nanoid()` 或后端返回的业务 ID 作为 key。

### Fiber 架构简析

React 16 引入 Fiber，将协调过程拆分为可中断的工作单元（Fiber Node）。每个 React 元素的 fiber 节点保存了 `child`、`sibling`、`return` 指针，形成链表结构，使得 React 可以在渲染中途让出主线程给高优先级任务（如用户输入），再恢复继续执行。

---

## 2. React Hooks 核心

### useState 与批量更新

```tsx
const [count, setCount] = useState(0);

// 函数式更新：基于前一个状态计算新状态
setCount(prev => prev + 1);  // ✅ 始终正确
setCount(count + 1);         // ⚠️ 闭包陷阱：在异步场景下可能读到过期值

// React 18+ 自动批量更新（Automatic Batching）
handleClick = () => {
  setCount(c => c + 1);      // 不触发重新渲染
  setName('Alice');          // 不触发重新渲染
  // 合并为一次重新渲染
};
```

### useEffect 的依赖数组

依赖数组决定了 effect 的执行时机和频率，是 React Hooks 最容易出错的地方：

```tsx
// 每次渲染后执行
useEffect(() => {
  document.title = `${count} times`;
});

// 仅在 count 变化时执行
useEffect(() => {
  fetchData(id).then(setData);
}, [id]);

// 仅在挂载时执行（清理副作用）
useEffect(() => {
  const subscription = eventBus.subscribe(handler);
  return () => subscription.unsubscribe();  // 清理函数
}, []);

// ❌ 常见错误：依赖了组件内部的函数，但函数未加入依赖数组
const handleSave = () => save(data);
useEffect(() => {
  // handleSave 每次渲染都是新引用，导致 effect 不断执行
  eventBus.on('save', handleSave);
}, []); // 缺少 handleSave！

// ✅ 修复方案1：将函数加入依赖（需用 useCallback 包裹）
const handleSave = useCallback(() => save(data), [data]);
useEffect(() => { eventBus.on('save', handleSave); }, [handleSave]);

// ✅ 修复方案2：在 effect 内部定义（不推荐大量使用）
useEffect(() => {
  const handleSave = () => save(data);
  eventBus.on('save', handleSave);
}, [data]);
```

### 闭包陷阱

在 `useEffect` / `setTimeout` / `eventListener` 中引用的 state 或 props，实际上捕获的是创建时所在渲染帧的值：

```tsx
const [count, setCount] = useState(0);
useEffect(() => {
  const timer = setInterval(() => {
    console.log(count);  // 永远是 0！因为捕获了初始化时的 count
    setCount(count + 1);  // 永远加到 1！
  }, 1000);
  return () => clearInterval(timer);
}, []);  // 空依赖，永不更新

// ✅ 正确写法：使用函数式更新
useEffect(() => {
  const timer = setInterval(() => {
    setCount(prev => prev + 1);  // 依赖 React 内部队列，避免闭包过期
  }, 1000);
  return () => clearInterval(timer);
}, []);
```

### useCallback 与 useMemo

两者都用于缓存引用/值，避免子组件因引用变化而重新渲染：

```tsx
// useCallback：缓存函数引用
const handleClick = useCallback((id: string) => {
  setSelectedId(id);
  analytics.track('select', { id });
}, []);  // 空数组：handleClick 引用永远不变

// useMemo：缓存计算结果
const sortedList = useMemo(() =>
  items.slice().sort((a, b) => a.price - b.price),
  [items]  // items 变化时才重新排序
);

// 典型场景：作为 React.memo 子组件的 props
const ListItem = React.memo(({ id, onSelect }: Props) => (
  <div onClick={() => onSelect(id)}>...</div>
));
// 如果 onSelect 不是 useCallback，每次父组件渲染 ListItem 都会重新渲染
```

---

## 3. React 状态管理

### 三层方案的选型逻辑

| 方案 | 适用场景 | 核心特点 |
|------|---------|---------|
| **useState** | 组件本地状态 | 最轻量，无额外开销 |
| **Context** | 跨层级透传（主题、Locale、用户信息） | 任何消费者都会重新渲染，需配合 memo |
| **Zustand** | 中等复杂度全局状态 | 极简 API，按需订阅避免不必要渲染 |
| **Redux Toolkit** | 大型多人协作项目、企业级状态追溯 | 不可变更新、时间旅行调试 |

### Context 的性能陷阱

Context 本质上是"值"的传播，而非"更新"的传播。当 Provider 的 value 变化时，所有消费该 Context 的组件都会触发重新渲染：

```tsx
// ❌ 每次渲染都创建新对象，导致所有消费者重新渲染
const ThemeContext = createContext({});
<ThemeContext.Provider value={{ color: 'blue', font: 'Arial' }}>

// ✅ 使用 useMemo 缓存 value
const value = useMemo(() => ({ color: 'blue', font: 'Arial' }), []);
<ThemeContext.Provider value={value}>

// ✅ 或拆分为多个细粒度 Context
const ColorContext = createContext('blue');
const FontContext = createContext('Arial');
```

### Zustand 示例

```tsx
import { create } from 'zustand';

interface AlertStore {
  alerts: Alert[];
  addAlert: (alert: Alert) => void;
  removeAlert: (id: string) => void;
}

const useAlertStore = create<AlertStore>((set) => ({
  alerts: [],
  addAlert: (alert) => set(state => ({ alerts: [...state.alerts, alert] })),
  removeAlert: (id) => set(state => ({
    alerts: state.alerts.filter(a => a.id !== id)
  })),
}));

// 组件中：只订阅 alerts，只在 alerts 变化时重新渲染
const AlertList = () => {
  const alerts = useAlertStore(s => s.alerts);  // selector 粒度控制
  return <>{alerts.map(a => <AlertItem key={a.id} alert={a} />)}</>;
};
```

### 单向数据流

React 的单向数据流是核心设计原则：数据从父组件通过 props 向下流动，子组件通过回调函数向上传递事件。Redux/Zustand 只是将这个模式扩展到全局——所有状态更新必须通过 dispatch（actions），reducer 产生新状态，UI 根据新状态重新渲染。永远不会出现"数据直接修改 UI"的情况。

---

## 4. React 性能优化

### React.memo vs useMemo vs useCallback

```tsx
// React.memo：高阶组件，包装子组件避免不必要的重渲染
const ExpensiveChart = React.memo(({ data, onRefresh }: ChartProps) => {
  // 仅在 data 或 onRefresh 引用变化时重新渲染
  return <CanvasChart data={data} />;
}, (prevProps, nextProps) => {
  // 自定义比较逻辑：返回 true 表示"不需要重新渲染"
  return prevProps.data.id === nextProps.data.id;
});

// useMemo：缓存计算结果，避免重复计算
const processed = useMemo(() =>
  transformData(rawData, options),
  [rawData, options]
);

// useCallback：缓存函数引用，避免子组件 props 引用变化
const handleSubmit = useCallback(async (formData: FormData) => {
  await submitToAPI(formData);
  resetForm();
}, [submitToAPI]);
```

### 代码分割：React.lazy + Suspense

```tsx
import { Suspense, lazy } from 'react';

// 路由级代码分割
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));

const App = () => (
  <Routes>
    <Route path="/dashboard" element={
      <Suspense fallback={<Spinner />}>
        <Dashboard />
      </Suspense>
    } />
  </Routes>
);

// 组件级分割（条件加载大型图表库）
const HeavyChart = lazy(() =>
  import('antd').then(({ Chart }) => ({ default: Chart }))
);
```

---

## 5. React + TypeScript 高级类型

### React.FC vs JSX.Element vs React.ReactNode

```tsx
// ❌ React.FC 的缺点：隐式包含 children，泛型类型推断不精确
type Props = { name: string };
const Header: React.FC<Props> = ({ name, children }) => { ... }

// ✅ 推荐：用函数声明 + 显式类型
type Props = { name: string; children?: React.ReactNode };
const Header = ({ name, children }: Props) => { ... }

// React.ReactNode ≈ JSX.Element | null | string | number | ReactPortal
// 适合 props 类型：children?: React.ReactNode

// JSX.Element：具体到 JSX 表达式，不含 null/文本/数字
```

### 事件处理类型

```tsx
// 常见事件类型速查
onClick: React.MouseEventHandler<HTMLButtonElement>
onChange: React.ChangeEventHandler<HTMLInputElement>
onSubmit: React.FormEventHandler<HTMLFormElement>
onKeyDown: React.KeyboardEventHandler<HTMLDivElement>
onScroll: React.UIEventHandler<HTMLDivElement>
onInput: React.FormEventHandler<HTMLInputElement>

// 获取事件目标的值（受控组件）
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  setValue(e.target.value);
};

// form 提交
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault();
};

// 获取鼠标位置
const handleMouseMove = (e: React.MouseEvent) => {
  console.log(e.clientX, e.clientY);
};
```

### ComponentPropsWithoutRef 与 forwardRef

```tsx
// 透传 props（不含 ref）
type ButtonBaseProps = React.ComponentPropsWithoutRef<'button'> & {
  variant: 'primary' | 'secondary';
};

// 使用 forwardRef 暴露 ref 给父组件
const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ label, error, ...props }, ref) => (
    <div>
      <label>{label}</label>
      <input ref={ref} {...props} />
      {error && <span>{error}</span>}
    </div>
  )
);
Input.displayName = 'Input';  // 调试友好
```

---

## 6. 大列表优化

### 虚拟滚动原理

虚拟滚动的核心思想：只渲染可视区域内的行，利用绝对定位或 transform 移动 DOM 节点，营造无限滚动的视觉效果。`react-window` 是最成熟的实现库：

```tsx
import { FixedSizeList as List } from 'react-window';

interface RowProps {
  index: number;
  style: React.CSSProperties;
  data: Alert[];  // 外部传入的完整数据集
}

const AlertRow = ({ index, style, data }: RowProps) => (
  <div style={style}>
    <AlertItem alert={data[index]} />
  </div>
);

const AlertList = ({ alerts }: { alerts: Alert[] }) => (
  <List
    height={600}
    itemCount={alerts.length}    // 列表总长度
    itemSize={72}                 // 每行固定高度（性能最优）
    width="100%"
    itemData={alerts}            // 通过 data prop 传递数据（避免闭包问题）
  >
    {AlertRow}
  </List>
);

// 变长行使用 VariableSizeList（每个 itemSize 不同，需缓存尺寸）
```

### 分页 + 懒加载组合策略

虚拟滚动适合一次性数据量很大的场景（数千行），但内存压力随数据总量线性增长。推荐混合策略：

```tsx
// 无限滚动：每次加载一页，虚拟滚动渲染
const [page, setPage] = useState(1);
const [alerts, setAlerts] = useState<Alert[]>([]);

const loadMore = async () => {
  const newData = await fetchAlerts({ page, pageSize: 50 });
  setAlerts(prev => [...prev, ...newData]);
};

// Intersection Observer 检测到底部时加载
useEffect(() => {
  const observer = new IntersectionObserver(([entry]) => {
    if (entry.isIntersecting && hasMore) loadMore();
  }, { rootMargin: '200px' });  // 提前 200px 触发
  if (sentinelRef.current) observer.observe(sentinelRef.current);
  return () => observer.disconnect();
}, [page, hasMore]);
```

### 内存泄漏排查

大列表最容易产生的内存问题：事件监听器未清理、订阅未取消、定时器未清除、闭包引用大数据集。

```tsx
useEffect(() => {
  let isMounted = true;
  const ws = new WebSocket(url);

  ws.onmessage = (event) => {
    if (isMounted) setData(JSON.parse(event.data));
  };

  return () => {
    isMounted = false;   // 阻止异步回调更新已卸载组件
    ws.close();          // 关闭 WebSocket
  };
}, []);
```

Chrome DevTools 的 Memory 面板配合 Heap Snapshot 快照对比，是定位此类泄漏的标准手段。组件卸载后控制台出现"Can't perform a React state update on an unmounted component"警告，应立即用 isMounted flag 或 AbortController 修复。

---

## 7. i18n 国际化

### react-i18next 核心配置

```tsx
// i18n.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import zhCN from './locales/zh-CN.json';
import enUS from './locales/en-US.json';

i18n.use(initReactI18next).init({
  resources: {
    'zh-CN': { translation: zhCN },
    'en-US': { translation: enUS },
  },
  lng: 'zh-CN',        // 默认语言
  fallbackLng: 'en-US',
  interpolation: { escapeValue: false },  // React 已经防注入
});

// 组件中使用
const { t, i18n } = useTranslation();

return (
  <div>
    <h1>{t('alert.pageTitle')}</h1>
    <button onClick={() =>
      i18n.changeLanguage(i18n.language === 'zh-CN' ? 'en-US' : 'zh-CN')
    }>
      {t('common.switchLang')}
    </button>
  </div>
);
```

### key 管理最佳实践

```json
// zh-CN.json — 按模块组织，避免重复和冲突
{
  "alert": {
    "pageTitle": "告警分析",
    "severity": {
      "critical": "严重",
      "high": "高危",
      "medium": "中危",
      "low": "低危"
    },
    "empty": "暂无告警数据",
    "count": "共 {{count}} 条告警"
  },
  "common": {
    "switchLang": "切换语言",
    "loading": "加载中...",
    "confirm": "确认",
    "cancel": "取消"
  }
}
```

翻译 key 的命名规范：`模块.子模块.具体含义`，使用命名空间（namespaces）分离业务模块，避免单一文件过大导致翻译更新冲突。

### 翻译更新同步策略

多人协作时，翻译文件更新容易出现 key 缺失或多余。建议：
- 使用 `i18next-parser` 自动扫描代码中的 `t('key')` 并生成/更新 JSON 文件
- 生产环境 key 缺失时 `t()` 返回 key 本身而非空字符串（便于快速定位未翻译项）
- 启用 `missingKeyHandler` 将缺失 key 记录到日志，发布后排查漏翻

### RTL 语言支持

阿拉伯语、希伯来语等 RTL（Right-to-Left）语言需要 HTML 属性和 CSS 同步翻转：

```tsx
// 设置 HTML dir 属性
document.documentElement.dir = locale === 'ar' ? 'rtl' : 'ltr';
document.documentElement.lang = locale;

// CSS 使用 logical properties
.container {
  padding-inline-start: 16px;  /* 代替 padding-left */
  margin-inline-end: 8px;      /* 代替 margin-right */
  text-align: start;          /* 代替 text-align: left */
}
```

---

## 8. Vue 3 Composition API

### setup 与响应式系统的初始化

```vue
<script setup lang="ts">
import { ref, reactive, computed, watch, watchEffect, onMounted } from 'vue';

// ref：基础类型的响应式容器（.value 访问）
const count = ref(0);
const name = ref<string>('');

// reactive：对象/数组的深度响应式（直接访问属性）
const state = reactive({
  alerts: [] as Alert[],
  filter: 'all',
});

// computed：派生响应式值（缓存特性）
const criticalAlerts = computed(() =>
  state.alerts.filter(a => a.severity === 'critical')
);

// watch：监听特定响应式源，懒执行
watch(() => state.filter, (newFilter) => {
  console.log('Filter changed to', newFilter);
}, { immediate: true });

// watchEffect：立即执行，依赖自动追踪（更简洁但不够精确）
watchEffect(() => {
  if (state.filter === 'critical') {
    analytics.track('critical_filter_view');
  }
});

// 生命周期钩子
onMounted(() => { fetchAlerts(); });
</script>
```

### ref vs reactive 的选择

- **ref**：适合原始类型和需要重新赋值整个对象的场景；模板中自动解包
- **reactive**：适合聚合相关状态为对象，减少模板中多个变量名

```vue
<!-- ref 在模板中自动解包，无需 .value -->
<template>
  <div>{{ count }}</div>  <!-- 直接是 0，不需要 {{ count.value }} -->
</template>
```

### Vue 3 组合式函数（Composables）

Vue 3 最有价值的设计：将可复用的逻辑封装为组合式函数，类似 React 自定义 Hooks：

```ts
// useAlertFilter.ts
import { ref, computed } from 'vue';

export function useAlertFilter(alerts: Ref<Alert[]>) {
  const filterSeverity = ref<Severity | 'all'>('all');
  const searchKeyword = ref('');

  const filtered = computed(() => {
    let result = alerts.value;
    if (filterSeverity.value !== 'all') {
      result = result.filter(a => a.severity === filterSeverity.value);
    }
    if (searchKeyword.value) {
      result = result.filter(a =>
        a.message.toLowerCase().includes(searchKeyword.value.toLowerCase())
      );
    }
    return result;
  });

  return { filterSeverity, searchKeyword, filtered };
}

// 组件中使用
const { filterSeverity, searchKeyword, filtered } = useAlertFilter(alerts);
```

---

## 9. Vue vs React 对比

### 响应式原理的本质差异

**React**：状态变化后调用 `setState`，触发当前组件及所有子组件的重新渲染，由虚拟 DOM + Diff 算法计算最小 DOM 更新。开发者对"什么时候重新渲染"没有直接控制权。

**Vue 3（Proxy）**：通过 JavaScript Proxy 拦截对象属性的读取和赋值，自动追踪依赖关系。当响应式数据变化时，精准通知所有依赖该数据的组件更新，粒度精确到属性级别。

```js
// Vue 3 Proxy 原理（简化版）
const observed = new Proxy(target, {
  get(obj, key) {
    track(obj, key);    // 建立"谁在读取"的依赖追踪
    return obj[key];
  },
  set(obj, key, value) {
    obj[key] = value;
    trigger(obj, key);  // 通知所有依赖方更新
    return true;
  }
});
```

### defineProperty vs Proxy

Vue 2 使用 `Object.defineProperty`，有以下局限：
- 必须遍历对象的每个 key 并手动设置 getter/setter
- 无法监听数组下标直接赋值（`arr[0] = x`）
- 无法新增/删除属性（需用 `Vue.set` / `Vue.delete`）

Vue 3 的 Proxy 解决了所有这些问题，是真正的深层响应式。

### 适用场景对比

| 维度 | React | Vue 3 |
|------|-------|-------|
| 学习曲线 | 稍陡（Hooks 体系庞大） | 平缓（Options/Composition 均可） |
| 团队规模 | 大型团队、分工明确 | 中小型团队、快速迭代 |
| 灵活性 | 高（需要自己做更多决策） | 适中（约定优先，减少选择成本） |
| 生态 | 更庞大（社区贡献多） | 官方工具链完整（Vite、Vue Router、Pinia 一体化） |
| TypeScript 支持 | 早期 TS 支持好 | 3.0+ 类型推导优秀 |

---

## 10. CI/CD 前端部署

### Docker 容器化

```dockerfile
# 多阶段构建：减小镜像体积
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# 生产镜像
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

```nginx
# nginx.conf：SPA 路由 fallback
server {
  listen 80;
  root /usr/share/nginx/html;
  index index.html;
  location / {
    try_files $uri $uri/ /index.html;  # 关键：所有路由回退到 index.html
  }
}
```

### K8s 部署配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alert-frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: alert-frontend
  template:
    metadata:
      labels:
        app: alert-frontend
    spec:
      containers:
        - name: frontend
          image: registry.example.com/alert-frontend:v1.2.3
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: alert-frontend-svc
spec:
  selector:
    app: alert-frontend
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

### 灰度发布与回滚

```bash
# Jenkins 蓝绿部署脚本（简化版）
kubectl rollout status deployment/alert-frontend
kubectl get pods -l app=alert-frontend

# 灰度切流 10% 流量到新版本
kubectl patch service alert-frontend \
  -p '{"spec":{"selector":{"app":"alert-frontend-canary"}}}'

# 验证无误后全量切换
kubectl rollout undo deployment/alert-frontend  # 一键回滚到上一版本

# 金丝雀发布（渐进式）
kubectl set image deployment/alert-frontend \
  frontend=registry.example.com/alert-frontend:v1.2.4 --record
```

核心回滚策略：
- 使用 `--record` 记录每次部署的镜像版本
- `kubectl rollout history deployment/<name>` 查看版本历史
- `kubectl rollout undo deployment/<name> --to-revision=N` 回滚到任意版本
- 生产环境建议使用 Argo Rollouts 或 Flagger 实现自动化金丝雀分析

---

## 面试 Q&A

**Q: React 中的 key 为什么不能用随机数？**
A: 随机 key 每次渲染都生成新值，React 无法匹配前后渲染的同一节点，导致 DOM 节点完全卸载重建、组件状态丢失、列表动画异常、性能严重退化。正确做法是用业务 ID（如 `item.id`）。

**Q: useEffect 依赖数组为空但 effect 里的变量还是旧值，怎么解决？**
A: 这是闭包陷阱。有三种解法：使用函数式更新 `setState(prev => prev + 1)`；将依赖变量加入数组并在函数前加 `useCallback`；使用 `useRef` 存储最新值 `const latestValue = useRef(value)` 绕过依赖数组。

**Q: React.memo、useMemo、useCallback 三个优化 hooks 的区别是什么？**
A: `React.memo` 控制组件是否重渲染；`useMemo` 缓存计算结果，避免重复执行昂贵计算；`useCallback` 缓存函数引用，避免子组件因 props 函数引用变化而重渲染。三者常配合使用，但不滥用——小型组件或低频更新场景下，过度优化反而增加代码复杂度。

**Q: Vue 3 的响应式系统和 React 有什么本质区别？**
A: Vue 3 通过 Proxy 精确追踪每个属性的访问依赖，数据变化时只更新依赖该数据的组件（细粒度更新）。React 依赖虚拟 DOM Diff，状态变化触发整棵子树重新渲染后再计算差异。Vue 在大量细粒度状态更新场景（如仪表盘实时数据）有优势；React 在需要手动控制更新边界时更灵活。

**Q: 虚拟滚动在大数据量下的性能瓶颈是什么？**
A: 虚拟滚动本身渲染行数恒定，但总数据量大时内存占用仍然线性增长。瓶颈包括：行高不固定导致的动态测量开销、滚动事件节流不当导致的帧率下降、DOM 节点复用时旧状态清理不彻底（建议在组件卸载时主动清空容器）。建议配合分页加载数据而非一次性加载全部。

**Q: i18n 翻译文件 key 管理混乱，怎么破？**
A: 使用 `i18next-parser` 自动扫描源码生成 key；命名空间按业务模块拆分（如 `alert.*`、`common.*`）；生产环境 `t()` 返回原始 key 以便发现漏翻项；使用命名插值 `{{count}}` 而非字符串拼接。

**Q: Docker 多阶段构建前端镜像，如何减少体积？**
A: 四个关键点：使用 `node:alpine` 而非完整镜像；`npm ci --only=production` 跳过 devDependencies；`.dockerignore` 排除 node_modules、.git、测试文件；最终镜像用 nginx:alpine 而非 Ubuntu 基底。一个典型前端生产镜像应控制在 30MB 以内。

**Q: CI/CD 中如何确保前端版本回滚的原子性？**
A: Docker 镜像不可变，版本回滚本质是切镜像版本。使用镜像 tag 作为版本号（如 `v1.2.3`），从不覆盖。K8s Deployment 的 `spec.replicas` 和 `strategy` 配置确保回滚原子执行。配合 Argo Rollouts 实现金丝雀灰度，自动化验证后再全量切换，回滚时间可控制在分钟级。
