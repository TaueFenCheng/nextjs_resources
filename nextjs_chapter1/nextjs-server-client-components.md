# Next.js App Router：Server Components 与 Client Components 完整规范

## 一、基本概念

### 1.1 默认规则

```
Next.js App Router 默认所有组件都是 Server Components
```

- **不需要声明** — 默认就是服务端组件
- **需要声明 `"use client"`** — 才能变成客户端组件

### 1.2 两种组件的本质区别

| 维度 | Server Component | Client Component |
|------|-----------------|------------------|
| **运行位置** | Node.js 服务端 | 浏览器（客户端） |
| **渲染时机** | 请求时（首屏）/ 缓存时 | 页面加载后（JS 执行） |
| **代码去向** | 不发送到浏览器 | 打包进 JS bundle |
| **HTML 生成** | 服务端生成完整 HTML | 客户端 JS 渲染 |

---

## 二、能力对比

### 2.1 Server Component 能做什么 ✅

```tsx
// ✅ 可以直接 async/await 获取数据
export async function ProductPage({ id }) {
  const product = await db.products.find(id);  // 直接访问数据库
  return <div>{product.name}</div>;
}

// ✅ 可以访问服务端资源
import { readFileSync } from 'fs';
export function ServerComponent() {
  const content = readFileSync('./data.json');  // 文件系统
  return <div>{content}</div>;
}

// ✅ 可以使用服务端环境变量（不带 NEXT_PUBLIC_）
export async function ServerComponent() {
  const secret = process.env.DATABASE_URL;  // 安全
  return <div>...</div>;
}

// ✅ SEO 友好，HTML 直接渲染
// ✅ 包体积小，代码不发送到客户端
```

### 2.2 Server Component 不能做什么 ❌

```tsx
// ❌ 不能使用 React hooks
import { useState, useEffect } from 'react';
export function ServerComponent() {
  const [count, setCount] = useState(0);  // 报错！
}

// ❌ 不能使用事件处理
export function ServerComponent() {
  return <button onClick={() => alert('hi')}>Click</button>;  // onClick 无效！
}

// ❌ 不能使用浏览器 API
export function ServerComponent() {
  const width = window.innerWidth;  // 报错！window 不存在
  localStorage.getItem('key');      // 报错！
  document.title = 'New Title';     // 报错！
}

// ❌ 不能使用仅在客户端的库
import { motion } from 'framer-motion';  // 可能报错！
```

### 2.3 Client Component 能做什么 ✅

```tsx
// ✅ 可以使用所有 React hooks
"use client";
import { useState, useEffect, useRef, useMemo } from 'react';
export function ClientComponent() {
  const [state, setState] = useState(0);
  useEffect(() => { ... }, []);
}

// ✅ 可以使用事件处理
"use client";
export function ClientComponent() {
  return <button onClick={(e) => console.log(e)}>Click</button>;
}

// ✅ 可以使用浏览器 API
"use client";
export function ClientComponent() {
  useEffect(() => {
    const width = window.innerWidth;
    localStorage.setItem('key', 'value');
    document.title = 'New Title';
  }, []);
}

// ✅ 可以使用动画库、图表库等
"use client";
import { motion } from 'framer-motion';
import { Chart } from 'react-chartjs-2';
```

### 2.4 Client Component 不能做什么 ❌

```tsx
// ❌ 不能直接 async/await（组件函数本身）
"use client";
export async function ClientComponent() {  // 报错！
  const data = await fetch('/api/data');
}

// ❌ 不能直接访问服务端资源
"use client";
import { readFileSync } from 'fs';  // 客户端没有 fs
export function ClientComponent() {
  readFileSync('./data.json');  // 报错！
}

// ❌ 不能访问非 NEXT_PUBLIC_ 环境变量
"use client";
export function ClientComponent() {
  const secret = process.env.DATABASE_URL;  // undefined（未暴露）
}
```

---

## 三、导入规则（关键！）

### 3.1 核心规则

```
Server Component 可以导入 Client Component ✅
Client Component 不能导入 Server Component ❌
```

```tsx
// ✅ 正确：Server Component 导入 Client Component
// app/layout.tsx（Server）
import { Navbar } from '@/components/navbar';  // Client Component
export default function Layout({ children }) {
  return <Navbar>{children}</Navbar>;
}

// ❌ 错误：Client Component 导入 Server Component
// components/navbar.tsx（Client）
"use client";
import { ServerDataFetcher } from './server-data';  // Server Component
// 报错："You're importing a component that needs 'fs'..."
```

### 3.2 为什么有这个限制？

```
Server Component 的代码不会发送到客户端
Client Component 在浏览器运行，无法执行服务端代码
```

如果 Client Component 导入 Server Component：
- Server Component 的服务端代码（如 `fs.readFile`）会被打包进客户端 bundle
- 浏览器无法执行 `fs.readFile` → 报错

### 3.3 组件树的"边界"

```
Server Component 层
    │
    │ "use client" 边界
    │
Client Component 层
    │
    │ 所有子组件默认都在客户端运行
    │
Client Component 子组件（无需再声明 "use client"）
```

一旦跨越 `"use client"` 边界，下面的所有子组件默认都在客户端运行：

```tsx
// page.tsx（Server）
import { InteractivePanel } from './interactive-panel';

export default function Page() {
  return <InteractivePanel />;
}

// interactive-panel.tsx（Client，有 "use client"）
"use client";
import { ChildA } from './child-a';  // ChildA 也是 Client，无需声明
import { ChildB } from './child-b';  // ChildB 也是 Client，无需声明

export function InteractivePanel() {
  return (
    <div>
      <ChildA />  {/* 客户端运行 */}
      <ChildB />  {/* 客户端运行 */}
    </div>
  );
}

// child-a.tsx（无需 "use client"，被 Client 组件导入）
export function ChildA() {
  const [count, setCount] = useState(0);  // ✅ 可以用 hooks！
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

---

## 四、数据传递方式

### 4.1 Server → Client 传递数据

Server Component 通过 **props** 将数据传给 Client Component：

```tsx
// Server Component
export async function ProductPage({ id }) {
  const product = await db.products.find(id);  // 服务端获取

  // ✅ 通过 props 传递给 Client Component
  return <ProductDetails product={product} />;
}

// Client Component
"use client";
export function ProductDetails({ product }) {
  const [expanded, setExpanded] = useState(false);  // 客户端交互
  return (
    <div>
      <h1>{product.name}</h1>
      <button onClick={() => setExpanded(!expanded)}>
        {expanded ? '收起' : '展开'}
      </button>
    </div>
  );
}
```

### 4.2 传递数据的限制

**传递的 props 必须是可序列化的**（能转成 JSON）：

```tsx
// ✅ 可序列化的数据
<ProductDetails
  product={{ id: 1, name: 'Phone' }}  // 对象 ✅
  items={[1, 2, 3]}                    // 数组 ✅
  count={42}                           // 数字 ✅
  enabled={true}                       // 布尔 ✅
/>

// ❌ 不可序列化的数据
<ProductDetails
  onClick={() => alert('hi')}          // 函数 ❌（但 Client 可定义）
  date={new Date()}                    // Date 对象 ❌（传字符串）
  element={<div>...</div>}             // React 元素 ❌
/>
```

### 4.3 Client → Server 获取数据

Client Component **不能直接访问服务端数据**，需要通过 API：

```tsx
// Client Component
"use client";
import { useEffect, useState } from 'react';

export function ProductList() {
  const [products, setProducts] = useState([]);

  useEffect(() => {
    // ✅ 通过 API Route 获取数据
    fetch('/api/products')
      .then(res => res.json())
      .then(data => setProducts(data));
  }, []);

  return <div>{products.map(p => <div key={p.id}>{p.name}</div>)}</div>;
}
```

---

## 五、使用场景总结

### 5.1 什么时候用 Server Component

| 场景 | 原因 |
|------|------|
| **获取后端数据** | 可直接 async/await，访问数据库 |
| **SEO 重要页面** | HTML 直接渲染，搜索引擎可见 |
| **静态内容展示** | 无交互需求，减少客户端 JS |
| **保护敏感数据** | 环境变量、API Key 不暴露给客户端 |
| **大型列表/表格** | 服务端渲染，减少首屏加载时间 |
| **页面布局** | 不需要 hooks，只是组合子组件 |

```tsx
// ✅ Server Component 示例
export async function BlogPage() {
  const posts = await db.posts.findAll();  // 服务端获取
  return (
    <main>
      <Header />
      <PostList posts={posts} />  {/* posts 传给 Client */}
    </main>
  );
}
```

### 5.2 什么时候用 Client Component

| 场景 | 原因 |
|------|------|
| **交互逻辑** | onClick、onChange、onSubmit |
| **React hooks** | useState、useEffect、useRef、useMemo |
| **浏览器 API** | window、localStorage、navigator |
| **动画效果** | framer-motion、CSS transitions |
| **第三方交互库** | 图表、拖拽、富文本编辑器 |
| **实时数据轮询** | setInterval、WebSocket |
| **表单验证** | 客户端即时反馈 |

```tsx
// ✅ Client Component 示例
"use client";

import { useState, useEffect, useCallback, useMemo } from 'react';
import { useRouter } from 'next/navigation';

export function CommandPalette() {
  const router = useRouter();
  const [open, setOpen] = useState(false);       // ✅ useState
  const [isMac, setIsMac] = useState(false);

  useEffect(() => {
    setIsMac(navigator.userAgent.includes('Mac'));  // ✅ 浏览器 API
  }, []);

  const handleNewChat = useCallback(() => {
    router.push('/workspace/chats/new');  // ✅ 路由跳转
    setOpen(false);
  }, [router]);

  return (
    <CommandDialog open={open} onOpenChange={setOpen}>
      <CommandItem onSelect={handleNewChat}>  {/* ✅ 事件处理 */}
        New Chat
      </CommandItem>
    </CommandDialog>
  );
}
```

---

## 六、最佳实践

### 6.1 组件拆分策略

将**交互部分**提取为 Client Component，**静态部分**保持 Server Component：

```tsx
// ❌ 不好：整个页面是 Client Component
"use client";
export function ProductPage({ id }) {
  const [product, setProduct] = useState(null);
  const [expanded, setExpanded] = useState(false);

  useEffect(() => {
    fetch(`/api/products/${id}`).then(res => res.json()).then(setProduct);
  }, [id]);

  return (
    <div>
      <h1>{product?.name}</h1>
      <button onClick={() => setExpanded(!expanded)}>展开</button>
    </div>
  );
}

// ✅ 好：Server 获取数据，Client 处理交互
// page.tsx（Server）
export async function ProductPage({ id }) {
  const product = await db.products.find(id);
  return <ProductDetails product={product} />;
}

// product-details.tsx（Client）
"use client";
export function ProductDetails({ product }) {
  const [expanded, setExpanded] = useState(false);
  return (
    <div>
      <h1>{product.name}</h1>
      <button onClick={() => setExpanded(!expanded)}>展开</button>
    </div>
  );
}
```

### 6.2 "use client" 放在哪里

```tsx
// ✅ 放在文件最顶部，第一行
"use client";

import { useState } from 'react';
import { Something } from './something';

export function MyComponent() {
  // ...
}

// ❌ 不要放在 import 之后
import { useState } from 'react';
"use client";  // 错误位置！
```

### 6.3 减少 Client Component 数量

```tsx
// ❌ 不好：整个大组件都是 Client
"use client";
export function LargeComponent() {
  const [selected, setSelected] = useState(null);
  return (
    <div>
      <Header />        {/* 无交互，不需要 Client */}
      <StaticContent /> {/* 无交互，不需要 Client */}
      <InteractiveList onSelect={setSelected} />  {/* 只有这个需要交互 */}
    </div>
  );
}

// ✅ 好：只把交互部分提取为 Client
// page.tsx（Server）
export function LargeComponent() {
  return (
    <div>
      <Header />              {/* Server */}
      <StaticContent />       {/* Server */}
      <InteractiveList />     {/* Client */}
    </div>
  );
}

// interactive-list.tsx（Client）
"use client";
export function InteractiveList() {
  const [selected, setSelected] = useState(null);
  // ...
}
```

### 6.4 第三方库的处理

很多 UI 库内部已经声明 `"use client"`：

```tsx
// Shadcn UI 组件内部已有 "use client"
// 你可以安全地在 Server Component 中使用
import { Button } from '@/components/ui/button';  // ✅ Button 是 Client

export function ServerPage() {
  return <Button>Click</Button>;  // ✅ 正常工作
}
```

但如果第三方库没有声明，你需要包装：

```tsx
// motion-wrapper.tsx
"use client";
import { motion } from 'framer-motion';
export { motion };

// page.tsx（Server）
import { motion } from './motion-wrapper';
export function Page() {
  return <motion.div animate={{ x: 100 }}>Hello</motion.div>;
}
```

---

## 七、常见陷阱

### 7.1 在 Server Component 中使用 onClick

```tsx
// ❌ 错误：Server Component 中的 onClick 无效
export function ServerButton() {
  return <button onClick={() => alert('hi')}>Click</button>;
  // onClick 被忽略，点击无反应
}

// ✅ 正确：提取为 Client Component
// interactive-button.tsx
"use client";
export function InteractiveButton() {
  return <button onClick={() => alert('hi')}>Click</button>;
}
```

### 7.2 在 Server Component 中使用 useState

```tsx
// ❌ 错误
export function Counter() {
  const [count, setCount] = useState(0);  // 报错！
  return <div>{count}</div>;
}

// ✅ 正确
"use client";
export function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

### 7.3 混用 async 函数和 Client Component

```tsx
// ❌ 错误：Client Component 不能是 async 函数
"use client";
export async function ClientComponent() {  // 报错！
  const data = await fetch('/api/data');
}

// ✅ 正确：用 useEffect 获取数据
"use client";
export function ClientComponent() {
  const [data, setData] = useState(null);
  useEffect(() => {
    fetch('/api/data').then(res => res.json()).then(setData);
  }, []);
}
```

### 7.4 在 Server Component 中传递函数 props

```tsx
// ❌ 错误：函数不能序列化
export function ServerPage() {
  const handleClick = () => alert('hi');
  return <ClientButton onClick={handleClick} />;
}

// ✅ 正确：在 Client Component 中定义函数
// client-button.tsx
"use client";
export function ClientButton() {
  const handleClick = () => alert('hi');  // 在 Client 中定义
  return <button onClick={handleClick}>Click</button>;
}
```

### 7.5 子组件忘记声明但被 Client 导入

```tsx
// child.tsx（没有 "use client"）
import { useState } from 'react';
export function Child() {
  const [count, setCount] = useState(0);  // 用了 hooks
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

// parent.tsx（Client）
"use client";
import { Child } from './child';
export function Parent() {
  return <Child />;  // ✅ 能工作（因为 Parent 是 Client）
}

// page.tsx（Server）
import { Child } from './child';
export function Page() {
  return <Child />;  // ❌ 报错！Child 用了 hooks 但没声明
}
```

**最佳实践**：组件自身用了 hooks，就应该自身声明 `"use client"`。

---

## 八、判断流程图

```
需要组件？
    │
    ├─ 需要交互（onClick/onChange）？ ──→ "use client"
    │
    ├─ 需要 hooks（useState/useEffect）？ ──→ "use client"
    │
    ├─ 需要浏览器 API（window/localStorage）？ ──→ "use client"
    │
    ├─ 需要动画库/图表库？ ──→ "use client"
    │
    ├─ 需要获取服务端数据？ ──→ Server Component（async）
    │
    ├─ 需要访问数据库/文件系统？ ──→ Server Component
    │
    ├─ SEO 重要？ ──→ Server Component
    │
    ├─ 静态展示内容？ ──→ Server Component
    │
    └─ 只组合其他组件？ ──→ Server Component
```

---

## 九、一句话总结

> **Server Component**：服务端运行，可访问数据库/文件系统，SEO友好，不能用 hooks 和事件处理
>
> **Client Component**：客户端运行，可用 hooks 和事件处理，不能直接访问服务端资源，需要通过 API
>
> **核心规则**：Server 可以导入 Client ✅，Client 不能导入 Server ❌
>
> **最佳实践**：尽量用 Server Component（默认），只在需要交互时提取为 Client Component