# Next.js 15+ params 异步变化详解

本文档详细介绍 Next.js 15 中 `params` 从同步对象变成 `Promise` 的原因、使用方式和注意事项。

---

## 目录

1. [变化概述](#一变化概述)
2. [新旧对比](#二新旧对比)
3. [为什么改成 Promise](#三为什么改成-promise)
4. [使用方式](#四使用方式)
5. [Client Components 中的处理](#五client-components-中的处理)
6. [其他异步参数](#六其他异步参数)
7. [迁移指南](#七迁移指南)
8. [常见问题](#八常见问题)

---

## 一、变化概述

### 1.1 核心变化

**Next.js 15+ 将动态路由参数 `params` 从同步对象改为 `Promise`。**

```
Next.js 14：params 是普通对象（同步访问）
Next.js 15：params 是 Promise（必须 await）
```

### 1.2 影响范围

| 受影响的文件 | 示例 |
|-------------|------|
| **route.ts**（API Routes） | `app/api/threads/[id]/route.ts` |
| **page.tsx**（页面） | `app/workspace/[id]/page.tsx` |
| **layout.tsx**（布局） | `app/[lang]/layout.tsx` |
| **generateMetadata** | SEO 元数据生成函数 |

---

## 二、新旧对比

### 2.1 API Routes（route.ts）

#### Next.js 14 及之前（同步）

```typescript
// ❌ 旧版本写法（Next.js 15 会报错）
import type { NextRequest } from "next/server";

export async function GET(
  request: NextRequest,
  { params }: { params: { thread_id: string } }  // 同步对象
) {
  // 直接访问，无需 await
  const threadId = params.thread_id;
  
  return Response.json({ id: threadId });
}
```

#### Next.js 15+（异步）

```typescript
// ✅ 新版本写法（Next.js 15 必须这样写）
import type { NextRequest } from "next/server";

export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ thread_id: string }> }  // Promise！
) {
  // 必须先 await params
  const { thread_id } = await params;
  
  return Response.json({ id: thread_id });
}
```

### 2.2 Pages（page.tsx）

#### Next.js 14 及之前

```tsx
// ❌ 旧版本写法
export default function Page({ params }: { params: { id: string } }) {
  return <div>ID: {params.id}</div>;  // 直接访问
}
```

#### Next.js 15+

```tsx
// ✅ 新版本写法（Server Component）
export default async function Page({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;  // 必须 await
  
  return <div>ID: {id}</div>;
}
```

### 2.3 实际项目示例

```typescript
// app/api/memory/[...path]/route.ts（Next.js 15+）
import type { NextRequest } from "next/server";

export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ path: string[] }> }  // Promise<string[]>
) {
  // await params 后才能访问 .path
  const fullPath = (await params).path.join("/");
  
  return proxyRequest(request, `/api/memory/${fullPath}`);
}

export async function POST(
  request: NextRequest,
  { params }: { params: Promise<{ path: string[] }> }
) {
  const fullPath = (await params).path.join("/");
  
  return proxyRequest(request, `/api/memory/${fullPath}`);
}
```

---

## 三、为什么改成 Promise

### 3.1 支持部分预渲染（Partial Prerendering, PPR）

Next.js 15 引入了 **PPR** 功能，允许页面同时包含静态和动态内容：

```
传统 SSR（服务端渲染）：
┌─────────────────────────────────────────────┐
│                                             │
│   用户请求                                   │
│       │                                     │
│       ▼                                     │
│   服务端渲染整个页面                          │
│       │                                     │
│       ▼                                     │
│   返回完整 HTML                              │
│       │                                     │
│       ▼                                     │
│   用户看到页面                               │
│                                             │
│   ⏱ 用户等待时间长                           │
│                                             │
└─────────────────────────────────────────────┘


PPR（部分预渲染）：
┌─────────────────────────────────────────────┐
│                                             │
│   用户请求                                   │
│       │                                     │
│       ▼                                     │
│   立即返回静态部分 HTML                       │
│       │                                     │
│       ▼                                     │
│   用户立即看到页面骨架                        │
│       │                                     │
│       ├────── 同时 ──────┐                  │
│       │                  │                  │
│       ▼                  ▼                  │
│   静态内容已显示    动态内容异步加载           │
│       │                  │                  │
│       │                  ▼                  │
│       │           动态内容填充               │
│       │                  │                  │
│       └──────────────────┘                  │
│                                             │
│   ⏱ 用户等待时间短，体验更好                  │
│                                             │
└─────────────────────────────────────────────┘
```

### 3.2 params 在 PPR 中的作用

```
页面渲染流程：

┌────────────────────────────────────────────────────────┐
│                      Page 组件                          │
│                                                        │
│  ┌──────────────────────────────────────────────────┐ │
│  │              静态部分（立即渲染）                   │ │
│  │                                                  │ │
│  │  • Header                                        │ │
│  │  • Navigation                                    │ │
│  │  • Footer                                        │ │
│  │  • 不依赖 params 的内容                           │ │
│  │                                                  │ │
│  │  这些部分不需要 await params                      │ │
│  │  可以立即返回 HTML                                │ │
│  └──────────────────────────────────────────────────┘ │
│                                                        │
│  ┌──────────────────────────────────────────────────┐ │
│  │              动态部分（异步渲染）                   │ │
│  │                                                  │ │
│  │  • 依赖 params 的内容                             │ │
│  │  • 需要根据 ID 获取的数据                          │ │
│  │                                                  │ │
│  │  const { id } = await params;  ← 异步等待        │ │
│  │  const data = await fetchData(id);               │ │
│  │                                                  │ │
│  │  这部分在 params 解析后渲染                        │ │
│  └──────────────────────────────────────────────────┘ │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### 3.3 性能对比

| 方式 | 渲染过程 | 用户体验 |
|------|---------|---------|
| **同步 params（旧）** | 等待 params 解析 → 渲染整个页面 → 返回 | 用户等待时间长 |
| **异步 params（新）** | 静态部分立即返回 → 动态部分异步加载 | 用户立即看到内容 |

### 3.4 代码示例：PPR 的效果

```tsx
// app/products/[id]/page.tsx
export default async function ProductPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  // 静态部分：不依赖 params，可以立即渲染
  const staticContent = (
    <div>
      <Header />  {/* 立即渲染 */}
      <Navigation />  {/* 立即渲染 */}
      <div className="skeleton">Loading product...</div>  {/* 占位符 */}
    </div>
  );
  
  // 动态部分：依赖 params，需要 await
  const { id } = await params;  // 异步等待
  const product = await getProduct(id);  // 根据 ID 获取数据
  
  // 最终返回
  return (
    <div>
      <Header />
      <Navigation />
      <ProductDetails product={product} />  {/* 动态填充 */}
    </div>
  );
}
```

---

## 四、使用方式

### 4.1 基本用法

```typescript
// 步骤 1：声明类型为 Promise
{ params }: { params: Promise<{ id: string }> }

// 步骤 2：await params 获取实际值
const { id } = await params;

// 步骤 3：使用参数值
console.log(id);
```

### 4.2 API Routes 完整示例

```typescript
// app/api/threads/[thread_id]/route.ts
import type { NextRequest } from "next/server";

export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ thread_id: string }> }
) {
  // ✅ 正确：await params
  const { thread_id } = await params;
  
  // ❌ 错误：直接访问 params.thread_id（会报错）
  // const thread_id = params.thread_id;
  
  const thread = await getThread(thread_id);
  
  return Response.json(thread);
}

export async function DELETE(
  request: NextRequest,
  { params }: { params: Promise<{ thread_id: string }> }
) {
  const { thread_id } = await params;
  
  await deleteThread(thread_id);
  
  return Response.json({ success: true });
}
```

### 4.3 捕获所有路由示例

```typescript
// app/api/files/[...path]/route.ts
import type { NextRequest } from "next/server";

export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ path: string[] }> }  // 注意是数组
) {
  const { path } = await params;
  
  // path 是数组，如 ["folder", "subfolder", "file.txt"]
  const fullPath = path.join("/");
  
  return Response.json({ path: fullPath });
}
```

### 4.4 可选捕获所有路由示例

```typescript
// app/api/artifacts/[[...paths]]/route.ts
import type { NextRequest } from "next/server";

export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ paths?: string[] }> }  // 可选数组
) {
  const { paths } = await params;
  
  // paths 可能是 undefined（空路径）
  const fullPath = paths?.join("/") ?? "";
  
  if (!fullPath) {
    return Response.json({ message: "Root artifacts" });
  }
  
  return Response.json({ path: fullPath });
}
```

### 4.5 Page 组件示例

```tsx
// app/workspace/[thread_id]/page.tsx

// Server Component
export default async function ThreadPage({
  params,
}: {
  params: Promise<{ thread_id: string }>;
}) {
  const { thread_id } = await params;
  
  // 可以用 thread_id 获取数据
  const thread = await fetchThread(thread_id);
  
  return (
    <div>
      <h1>Thread: {thread_id}</h1>
      <ThreadContent thread={thread} />
    </div>
  );
}
```

### 4.6 Layout 组件示例

```tsx
// app/[lang]/layout.tsx

export default async function LangLayout({
  params,
  children,
}: {
  params: Promise<{ lang: string }>;
  children: React.ReactNode;
}) {
  const { lang } = await params;
  
  return (
    <div lang={lang}>
      <Header lang={lang} />
      {children}
    </div>
  );
}
```

---

## 五、Client Components 中的处理

### 5.1 问题

Client Components 不能直接使用 `await`（组件函数本身不能是 async）：

```tsx
// ❌ 错误：Client Component 不能是 async
"use client";

export default async function ClientPage({ params }) {  // 报错！
  const { id } = await params;
}
```

### 5.2 解决方案：使用 React 19 的 `use()` API

```tsx
// ✅ 正确：使用 React 19 的 use() API
"use client";

import { use } from "react";

export default function ClientPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  // use() 可以在 Client Component 中"解开" Promise
  const { id } = use(params);
  
  return <div>ID: {id}</div>;
}
```

### 5.3 use() API 说明

```tsx
"use client";

import { use } from "react";

// use() 可以处理 Promise、Context 等
const value = use(promise);  // 解开 Promise
const contextValue = use(context);  // 获取 Context 值

// 示例：在 Client Component 中使用 params
function ClientComponent({
  params,
}: {
  params: Promise<{ thread_id: string }>;
}) {
  const { thread_id } = use(params);  // 相当于 await
  
  // 然后可以正常使用 thread_id
  const [thread, setThread] = useState(null);
  
  useEffect(() => {
    fetch(`/api/threads/${thread_id}`)
      .then(res => res.json())
      .then(setThread);
  }, [thread_id]);
  
  return <div>{thread?.title}</div>;
}
```

### 5.4 Server vs Client 对比

```tsx
// Server Component：直接 await
export default async function ServerPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;  // async/await
  return <div>{id}</div>;
}

// Client Component：使用 use()
"use client";

import { use } from "react";

export default function ClientPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = use(params);  // use() API
  return <div>{id}</div>;
}
```

---

## 六、其他异步参数

### 6.1 searchParams 也变成 Promise

除了 `params`，URL 查询参数 `searchParams` 也变成了 Promise：

```tsx
// app/search/page.tsx
export default async function SearchPage({
  params,
  searchParams,
}: {
  params: Promise<{ category: string }>;
  searchParams: Promise<{ q: string; page: string }>;  // 也是 Promise
}) {
  const { category } = await params;
  const { q, page } = await searchParams;  // 必须 await
  
  const results = await searchProducts(category, q, parseInt(page ?? "1"));
  
  return <SearchResults results={results} />;
}
```

### 6.2 完整参数列表

| 参数 | 旧类型 | 新类型 | 用途 |
|------|--------|--------|------|
| `params` | `{ id: string }` | `Promise<{ id: string }>` | 动态路由参数 |
| `searchParams` | `{ page: string }` | `Promise<{ page: string }>` | URL 查询参数 |

### 6.3 实际示例

```tsx
// URL: /products/electronics?q=phone&page=2

export default async function ProductListPage({
  params,
  searchParams,
}: {
  params: Promise<{ category: string }>;
  searchParams: Promise<{ q?: string; page?: string }>;
}) {
  const { category } = await params;  // "electronics"
  const { q, page } = await searchParams;  // q="phone", page="2"
  
  console.log(category);  // "electronics"
  console.log(q);         // "phone"
  console.log(page);      // "2"
  
  // ...
}
```

---

## 七、迁移指南

### 7.1 从 Next.js 14 迁移到 15

#### 步骤 1：更新类型声明

```typescript
// 旧代码
{ params }: { params: { id: string } }

// 新代码
{ params }: { params: Promise<{ id: string }> }
```

#### 步骤 2：添加 await

```typescript
// 旧代码
const id = params.id;

// 新代码
const { id } = await params;
```

#### 步骤 3：更新所有使用 params 的地方

```typescript
// route.ts
export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ thread_id: string }> }  // ← 更新类型
) {
  const { thread_id } = await params;  // ← 添加 await
  // ...
}

// page.tsx
export default async function Page({
  params,
}: {
  params: Promise<{ id: string }>  // ← 更新类型
}) {
  const { id } = await params;  // ← 添加 await
  // ...
}
```

### 7.2 批量迁移示例

#### API Routes

```typescript
// 迁移前（Next.js 14）
export async function GET(
  request,
  { params }: { params: { path: string[] } }
) {
  return proxyRequest(request, `/api/memory/${params.path.join("/")}`);
}

// 迁移后（Next.js 15）
export async function GET(
  request,
  { params }: { params: Promise<{ path: string[] }> }
) {
  return proxyRequest(request, `/api/memory/${(await params).path.join("/")}`);
}
```

#### Pages

```tsx
// 迁移前（Next.js 14）
export default function ThreadPage({ params }: { params: { id: string } }) {
  return <Thread id={params.id} />;
}

// 迁移后（Next.js 15）
export default async function ThreadPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  return <Thread id={id} />;
}
```

### 7.3 使用 Codemod 自动迁移

Next.js 提供了自动迁移工具：

```bash
# 使用 Next.js codemod 自动迁移
npx @next/codemod@latest next-15-params .

# 这会自动将 params 从同步改为 Promise
```

---

## 八、常见问题

### 8.1 不使用 await 会怎样？

```typescript
// ❌ 错误示例
export async function GET(
  request,
  { params }: { params: Promise<{ id: string }> }
) {
  // 不使用 await，直接访问 params.id
  const id = params.id;  // undefined！Promise 没有 id 属性
  
  // 这会导致：
  // 1. TypeScript 类型错误
  // 2. 运行时 id 为 undefined
  // 3. API 返回错误数据
}
```

**后果**：
- TypeScript 报错：`Property 'id' does not exist on type 'Promise<...>'`
- 运行时 `params.id` 为 `undefined`
- API 返回错误数据或报错

### 8.2 await 后 params 还能用吗？

```typescript
// ✅ 正确：await 一次后，可以多次使用
export async function GET(request, { params }) {
  const { id } = await params;  // await 一次
  
  // await 后得到的是普通对象，可以正常使用
  console.log(id);  // ✅ 正常工作
  
  // 不需要再次 await
  fetchData(id);    // ✅ 正常工作
  deleteData(id);   // ✅ 正常工作
}
```

### 8.3 可以 await params.path 而不是整个 params 吗？

```typescript
// ❌ 错误：不能只 await 部分属性
export async function GET(request, { params }) {
  const path = await params.path;  // 报错！params 没有 .path 属性
}

// ✅ 正确：await 整个 params
export async function GET(request, { params }) {
  const { path } = await params;  // 先 await params，再解构
  const fullPath = path.join("/");
}
```

### 8.4 在同步函数中如何使用？

```typescript
// ❌ 错误：同步函数不能使用 await
export function GET(request, { params }) {
  const { id } = await params;  // 报错！同步函数不能用 await
}

// ✅ 正确：必须用 async 函数
export async function GET(request, { params }) {
  const { id } = await params;  // async 函数可以用 await
}
```

### 8.5 generateMetadata 中的 params

```typescript
// generateMetadata 中也需要 await
export async function generateMetadata({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;  // 必须 await
  
  const product = await getProduct(id);
  
  return {
    title: product.name,
    description: product.description,
  };
}
```

---

## 九、总结

### 9.1 核心变化表

| 维度 | Next.js 14 | Next.js 15+ |
|------|-----------|-------------|
| **params 类型** | `{ id: string }` | `Promise<{ id: string }>` |
| **获取方式** | `params.id` | `await params` 后解构 |
| **searchParams 类型** | `{ q: string }` | `Promise<{ q: string }>` |
| **Client Component** | 直接访问 | 使用 `use(params)` |

### 9.2 使用规则

```
┌─────────────────────────────────────────────────────┐
│                  使用规则                             │
│                                                     │
│  1. 类型声明：Promise<{ id: string }>               │
│                                                     │
│  2. Server Component：                              │
│     const { id } = await params;                   │
│                                                     │
│  3. Client Component：                              │
│     const { id } = use(params);                    │
│                                                     │
│  4. API Route：                                     │
│     const { id } = await params;                   │
│                                                     │
│  5. 必须先 await 整个 params，再访问属性             │
│                                                     │
│  6. 不能直接访问 params.xxx（会报错）               │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 9.3 一句话总结

> **Next.js 15+ 将 `params` 改为 `Promise`**，目的是支持**部分预渲染（PPR）**，提升页面加载性能。使用时必须先 `await params` 才能访问参数值。Client Components 使用 React 19 的 `use()` API 来处理。

---

## 十、参考资料

- [Next.js 15 Release Notes](https://nextjs.org/blog/next-15)
- [Next.js App Router Documentation](https://nextjs.org/docs/app/api-reference/functions)
- [React 19 use() API](https://react.dev/reference/react/use)
- [Partial Prerendering (PPR)](https://nextjs.org/docs/app/building-your-application/rendering/partial-prerendering)