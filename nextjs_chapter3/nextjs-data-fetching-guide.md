---
title: Next.js App Router 数据获取指南
description: 总结 Next.js 官方文档中关于数据获取的核心内容
---

# Next.js App Router 数据获取指南

本文总结 Next.js App Router 中数据获取的核心概念、方法和最佳实践。

---

## 目录

1. [核心概念](#一核心概念)
2. [Server Components 中获取数据](#二server-components-中获取数据)
3. [扩展的 fetch API](#三扩展的-fetch-api)
4. [静态与动态数据获取](#四静态与动态数据获取)
5. [并行与顺序数据获取](#五并行与顺序数据获取)
6. [缓存与重新验证](#六缓存与重新验证)
7. [Client Components 数据获取](#七client-components-数据获取)
8. [Loading UI 与 Streaming](#八loading-ui-与-streaming)
9. [最佳实践](#九最佳实践)
10. [常见问题](#十常见问题)

---

## 一、核心概念

### 1.1 App Router 的数据获取革命

Next.js App Router 基于 React Server Components，带来了全新的数据获取方式：

```
┌─────────────────────────────────────────────────────────────────┐
│                  传统方式 (Pages Router)                         │
├─────────────────────────────────────────────────────────────────┤
│  Client Component                                               │
│       │                                                         │
│       │ useEffect(() => { fetch(...) })                        │
│       ▼                                                         │
│  客户端发起请求                                                  │
│       │                                                         │
│       ▼                                                         │
│  等待响应 + 状态管理 (loading, error)                            │
│       │                                                         │
│       ▼                                                         │
│  渲染数据                                                       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  新方式 (App Router)                             │
├─────────────────────────────────────────────────────────────────┤
│  Server Component (async)                                       │
│       │                                                         │
│       │ await fetch(...)                                        │
│       ▼                                                         │
│  服务端直接获取数据                                               │
│       │                                                         │
│       ▼                                                         │
│  渲染完成后发送 HTML 到客户端                                     │
│       │                                                         │
│       ▼                                                         │
│  客户端直接展示，无需 useEffect                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 主要优势

| 优势 | 说明 |
|------|------|
| **无需 useEffect** | Server Components 直接 async/await |
| **无需额外库** | 不必依赖 SWR、React Query 进行初始数据获取 |
| **更安全** | 服务端可安全访问后端资源（数据库、API密钥） |
| **更好的性能** | 数据获取和渲染在同一环境，减少网络往返 |
| **更小的 JS bundle** | 不需要在客户端发送数据获取逻辑 |
| **自动请求去重** | Next.js 自动对相同请求进行去重 |

### 1.3 获取数据的场景

| 场景 | 推荐方式 |
|------|---------|
| **初始页面数据** | Server Component + async/await |
| **用户交互触发** | Client Component + Server Actions 或 API Route |
| **实时更新数据** | Client Component + SWR/React Query |
| **表单提交** | Server Actions |

---

## 二、Server Components 中获取数据

### 2.1 基本用法

Server Components 可以直接使用 `async/await` 获取数据：

```tsx
// app/page.tsx (Server Component)

async function getData() {
  const res = await fetch('https://api.example.com/posts');

  if (!res.ok) {
    // 处理错误
    throw new Error('Failed to fetch data');
  }

  return res.json();
}

export default async function Page() {
  const data = await getData();

  return (
    <main>
      <h1>{data.title}</h1>
      <p>{data.content}</p>
    </main>
  );
}
```

### 2.2 组件必须是 async

```tsx
// ✅ 正确：async 函数
export default async function Page() {
  const data = await fetch('/api/data');
  return <div>{data}</div>;
}

// ❌ 错误：非 async 函数无法使用 await
export default function Page() {
  const data = await fetch('/api/data'); // 报错！
  return <div>{data}</div>;
}
```

### 2.3 TypeScript 类型定义

```tsx
// 定义数据类型
interface Post {
  id: number;
  title: string;
  content: string;
}

// 获取函数带类型
async function getPosts(): Promise<Post[]> {
  const res = await fetch('https://api.example.com/posts');
  return res.json();
}

// 组件中使用
export default async function PostsPage() {
  const posts: Post[] = await getPosts();

  return (
    <ul>
      {posts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### 2.4 嵌套组件数据获取

```tsx
// app/posts/page.tsx
async function getPosts() {
  const res = await fetch('https://api.example.com/posts');
  return res.json();
}

export default async function PostsPage() {
  const posts = await getPosts();

  return (
    <div>
      <h1>Posts</h1>
      {posts.map((post) => (
        <PostItem key={post.id} post={post} />
      ))}
    </div>
  );
}

// 子组件可以直接接收数据作为 props
function PostItem({ post }: { post: Post }) {
  return (
    <article>
      <h2>{post.title}</h2>
      <p>{post.content}</p>
    </article>
  );
}
```

---

## 三、扩展的 fetch API

### 3.1 Next.js 扩展的 fetch

Next.js 扩展了原生 `fetch`，添加了 `next` 选项：

```tsx
// 原生 fetch
fetch('https://api.example.com/data');

// Next.js 扩展 fetch
fetch('https://api.example.com/data', {
  next: {
    revalidate: 60,  // 每 60 秒重新验证
    tags: ['posts'], // 用于按标签重新验证
  }
});
```

### 3.2 扩展选项说明

| 选项 | 类型 | 说明 |
|------|------|------|
| `next.revalidate` | `number \| false` | 重新验证间隔（秒），false 表示永不更新 |
| `next.tags` | `string[]` | 缓存标签，用于按标签批量重新验证 |
| `cache` | `'force-cache' \| 'no-store'` | 缓存策略 |

### 3.3 自动请求去重

Next.js 自动对相同 URL 的请求进行去重：

```tsx
// 这两个请求会被自动合并为一个
async function ComponentA() {
  const data = await fetch('https://api.example.com/data');
  // ...
}

async function ComponentB() {
  const data = await fetch('https://api.example.com/data');
  // ...
}

// 在同一个渲染周期内，只会发起一次实际请求
```

请求去重示意图：

```
┌─────────────────────────────────────────────────────────────────┐
│                     请求去重机制                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Component A  ──┐                                               │
│                 │                                               │
│                 │   fetch('https://api.example.com/data')       │
│                 │         │                                     │
│  Component B  ──┤         ▼                                     │
│                 │   ┌───────────────┐                           │
│                 │   │  去重检查      │                           │
│                 │   │  相同 URL?    │                           │
│                 │   └───────┬───────┘                           │
│                 │           │                                   │
│                 │           ▼                                   │
│                 │   ┌───────────────┐                           │
│                 │   │  发起一次请求  │                           │
│                 │   └───────┬───────┘                           │
│                 │           │                                   │
│                 ├───────────┼───────────┐                       │
│                 │           │           │                       │
│                 ▼           ▼           ▼                       │
│            Component A   Component B   共享结果                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 四、静态与动态数据获取

### 4.1 静态数据获取

默认情况下，`fetch` 会缓存数据，构建时生成静态 HTML：

```tsx
// 静态数据获取（默认行为）
async function Page() {
  // 构建时获取并缓存
  const data = await fetch('https://api.example.com/data');

  return <div>{data}</div>;
}

// 明确指定缓存
async function Page() {
  const data = await fetch('https://api.example.com/data', {
    cache: 'force-cache'  // 强制缓存
  });

  return <div>{data}</div>;
}
```

### 4.2 动态数据获取

使用 `no-store` 禁用缓存，每次请求都重新获取：

```tsx
// 动态数据获取
async function Page() {
  const data = await fetch('https://api.example.com/data', {
    cache: 'no-store'  // 每次请求都重新获取
  });

  return <div>{data}</div>;
}
```

### 4.3 对比表

| 特性 | 静态数据 (`force-cache`) | 动态数据 (`no-store`) |
|------|------------------------|----------------------|
| **获取时机** | 构建时 | 每次请求时 |
| **数据新鲜度** | 固定（构建时） | 最新 |
| **响应速度** | 极快（静态文件） | 较慢（需等待） |
| **适用场景** | 博客、文档、固定内容 | 用户仪表盘、实时数据 |

### 4.4 使用动态函数会强制动态渲染

以下函数会使页面变成动态渲染：

```tsx
// 使用 cookies() → 动态渲染
import { cookies } from 'next/headers';

export default async function Page() {
  const cookieStore = await cookies();
  const theme = cookieStore.get('theme');

  // 这个页面会变成动态渲染
  return <div>Theme: {theme?.value}</div>;
}

// 使用 headers() → 动态渲染
import { headers } from 'next/headers';

export default async function Page() {
  const headersList = await headers();
  const userAgent = headersList.get('user-agent');

  // 这个页面会变成动态渲染
  return <div>UA: {userAgent}</div>;
}
```

---

## 五、并行与顺序数据获取

### 5.1 顺序数据获取（Waterfall）

当一个请求依赖另一个请求的结果时：

```tsx
// 顺序获取（瀑布流）
async function Page() {
  // 第一个请求
  const user = await fetch('https://api.example.com/user/1');

  // 第二个请求依赖第一个的结果
  const posts = await fetch(`https://api.example.com/posts?userId=${user.id}`);

  return (
    <div>
      <h1>{user.name}</h1>
      <PostsList posts={posts} />
    </div>
  );
}
```

瀑布流示意图：

```
时间线：
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  │                                                              │
│  │  fetch user ─────────────────────┐                          │
│  │                                  │                          │
│  │                                  ▼                          │
│  │                            user ready                       │
│  │                                  │                          │
│  │                                  │                          │
│  │  fetch posts (依赖 user.id) ────┐│                          │
│  │                                 ││                          │
│  │                                 ▼│                          │
│  │                           posts ready                       │
│  │                                  │                          │
│  │                                  ▼                          │
│  │                            渲染完成                          │
│                                                                 │
│  ⏱ 总耗时 = fetch user + fetch posts                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 并行数据获取

独立请求应该并行执行：

```tsx
// 并行获取
async function Page() {
  // 使用 Promise.all 同时发起多个请求
  const [users, posts, comments] = await Promise.all([
    fetch('https://api.example.com/users'),
    fetch('https://api.example.com/posts'),
    fetch('https://api.example.com/comments'),
  ]);

  return (
    <div>
      <UsersList users={users} />
      <PostsList posts={posts} />
      <CommentsList comments={comments} />
    </div>
  );
}
```

并行获取示意图：

```
时间线：
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  │                                                              │
│  │  fetch users  ───┐                                           │
│  │                   │                                           │
│  │  fetch posts  ───┼─── 同时发起                                │
│  │                   │                                           │
│  │  fetch comments ─┘                                           │
│  │                   │                                           │
│  │                   ▼                                           │
│  │            全部完成 (等待最慢的)                               │
│  │                   │                                           │
│  │                   ▼                                           │
│  │              渲染完成                                          │
│                                                                 │
│  ⏱ 总耗时 = max(fetch users, fetch posts, fetch comments)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 预加载模式

创建预加载函数优化数据获取：

```tsx
// utils/data.ts

// 预加载函数 - 立即发起请求但不等待结果
let usersPromise: Promise<User[]> | null = null;

export function preloadUsers() {
  usersPromise = fetch('https://api.example.com/users').then(r => r.json());
}

export async function getUsers() {
  if (!usersPromise) {
    preloadUsers();
  }
  return usersPromise!;
}

// app/layout.tsx - 在布局中预加载
import { preloadUsers } from '@/utils/data';

export default function Layout({ children }) {
  preloadUsers(); // 立即发起请求
  return <div>{children}</div>;
}

// app/users/page.tsx - 使用预加载的数据
import { getUsers } from '@/utils/data';

export default async function UsersPage() {
  const users = await getUsers(); // 直接获取已发起的请求结果
  return <UsersList users={users} />;
}
```

---

## 六、缓存与重新验证

### 6.1 使用 revalidate 定时更新

```tsx
// 每 60 秒重新验证数据
async function Page() {
  const data = await fetch('https://api.example.com/data', {
    next: {
      revalidate: 60  // ISR - 增量静态再生
    }
  });

  return <div>{data}</div>;
}
```

重新验证时间建议：

| revalidate 值 | 适用场景 |
|--------------|---------|
| `false` 或 `3600` (1小时) | 博客文章、文档 |
| `60` (1分钟) | 新闻、产品列表 |
| `10` (10秒) | 实时排行榜、股票价格 |
| `0` | 实时数据（等同于 `no-store`） |

### 6.2 使用 tags 按标签重新验证

```tsx
// 定义缓存标签
async function getPosts() {
  const res = await fetch('https://api.example.com/posts', {
    next: {
      tags: ['posts'],  // 标签名称
    }
  });
  return res.json();
}

// Server Action 中按标签重新验证
import { revalidateTag } from 'next/cache';

export async function refreshPosts() {
  revalidateTag('posts'); // 重新验证所有带 'posts' 标签的请求
}

// API Route 中按标签重新验证
import { revalidateTag } from 'next/cache';

export async function POST() {
  revalidateTag('posts');
  return Response.json({ revalidated: true });
}
```

### 6.3 按路径重新验证

```tsx
import { revalidatePath } from 'next/cache';

// 重新验证特定路径
export async function refreshPage() {
  revalidatePath('/posts'); // 重新验证 /posts 路径
}

// 重新验证所有路径
export async function refreshAll() {
  revalidatePath('/', 'layout'); // 重新验证根布局下的所有页面
}
```

### 6.4 缓存策略对比

| 策略 | 代码 | 数据更新时机 |
|------|------|-------------|
| **静态（永不更新）** | `cache: 'force-cache'` 或 `revalidate: false` | 永不 |
| **定时更新（ISR）** | `revalidate: 60` | 每 60 秒 |
| **按需更新** | `revalidateTag()` / `revalidatePath()` | 手动触发 |
| **动态（每次获取）** | `cache: 'no-store'` 或 `revalidate: 0` | 每次请求 |

---

## 七、Client Components 数据获取

### 7.1 Server Components 的限制

Client Components 不能直接使用 `async/await`：

```tsx
// ❌ 错误：Client Component 不能是 async
'use client';

export default async function ClientPage() {
  const data = await fetch('/api/data'); // 报错！
  return <div>{data}</div>;
}
```

### 7.2 解决方案

#### 方案一：在父级 Server Component 获取数据

```tsx
// app/page.tsx (Server Component)
async function getData() {
  const res = await fetch('https://api.example.com/data');
  return res.json();
}

export default async function Page() {
  const data = await getData();

  // 将数据传递给 Client Component
  return <ClientComponent data={data} />;
}

// components/ClientComponent.tsx
'use client';

export function ClientComponent({ data }: { data: Data }) {
  // 在 Client Component 中使用数据
  return <div>{data.title}</div>;
}
```

#### 方案二：使用 useEffect

```tsx
// components/ClientComponent.tsx
'use client';

import { useEffect, useState } from 'react';

export function ClientComponent() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/data')
      .then(res => res.json())
      .then(data => {
        setData(data);
        setLoading(false);
      });
  }, []);

  if (loading) return <div>Loading...</div>;
  return <div>{data.title}</div>;
}
```

#### 方案三：使用 SWR 或 React Query

```tsx
// 使用 SWR
'use client';

import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then(res => res.json());

export function ClientComponent() {
  const { data, error, isLoading } = useSWR('/api/data', fetcher);

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error</div>;

  return <div>{data.title}</div>;
}

// 使用 React Query
'use client';

import { useQuery } from '@tanstack/react-query';

export function ClientComponent() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['data'],
    queryFn: () => fetch('/api/data').then(res => res.json()),
  });

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error</div>;

  return <div>{data.title}</div>;
}
```

### 7.3 推荐策略

```
┌─────────────────────────────────────────────────────────────────┐
│                     数据获取策略                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  场景 1：初始页面数据                                             │
│  ────────────────────                                           │
│  → Server Component + async/await                               │
│  → 无需客户端状态管理                                             │
│                                                                 │
│  场景 2：用户交互触发（按钮点击）                                  │
│  ────────────────────                                           │
│  → Server Actions                                               │
│  → 无需手动 fetch                                                │
│                                                                 │
│  场景 3：实时更新、轮询                                           │
│  ────────────────────                                           │
│  → Client Component + SWR/React Query                           │
│  → 自动刷新、缓存管理                                             │
│                                                                 │
│  场景 4：混合场景                                                 │
│  ────────────────────                                           │
│  → Server Component 获取初始数据                                 │
│  → 传递给 Client Component                                      │
│  → Client Component 用 SWR 处理实时更新                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 八、Loading UI 与 Streaming

### 8.1 使用 loading.tsx

创建 `loading.tsx` 文件提供加载状态：

```tsx
// app/posts/loading.tsx
export default function Loading() {
  return <div>Loading posts...</div>;
}

// app/posts/page.tsx
async function getPosts() {
  const res = await fetch('https://api.example.com/posts');
  return res.json();
}

export default async function PostsPage() {
  const posts = await getPosts(); // 加载期间显示 loading.tsx

  return (
    <ul>
      {posts.map(post => <li key={post.id}>{post.title}</li>)}
    </ul>
  );
}
```

### 8.2 使用 Suspense 进行 Streaming

```tsx
// app/page.tsx
import { Suspense } from 'react';

async function SlowComponent() {
  await new Promise(resolve => setTimeout(resolve, 2000));
  return <div>慢加载的内容</div>;
}

async function FastComponent() {
  return <div>快加载的内容</div>;
}

export default function Page() {
  return (
    <div>
      {/* 快速加载的内容立即显示 */}
      <FastComponent />

      {/* 慢加载的内容流式传输 */}
      <Suspense fallback={<div>Loading...</div>}>
        <SlowComponent />
      </Suspense>
    </div>
  );
}
```

### 8.3 Streaming 工作原理

```
┌─────────────────────────────────────────────────────────────────┐
│                     Streaming 流式渲染                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  用户请求                                                        │
│      │                                                          │
│      ▼                                                          │
│  ┌────────────────────────────────────┐                         │
│  │  发送初始 HTML（Shell）              │                         │
│  │  • FastComponent 内容               │                         │
│  │  • Suspense fallback               │                         │
│  │  • Loading UI                      │                         │
│  └────────────────────────────────────┘                         │
│      │                                                          │
│      │  用户立即看到部分内容                                        │
│      │                                                          │
│      ▼                                                          │
│  ┌────────────────────────────────────┐                         │
│  │  SlowComponent 数据获取完成         │                         │
│  │  • 流式发送真实内容                 │                         │
│  │  • 替换 Suspense fallback          │                         │
│  └────────────────────────────────────┘                         │
│      │                                                          │
│      ▼                                                          │
│  用户看到完整内容                                                 │
│                                                                 │
│  优势：                                                          │
│  • 用户不必等待所有数据加载完成                                    │
│  • 感知加载速度更快                                               │
│  • SEO 友好（最终 HTML 完整）                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 九、最佳实践

### 9.1 选择合适的获取方式

```tsx
// ✅ 推荐：Server Component 直接获取
export default async function Page() {
  const data = await fetch('/api/data');
  return <div>{data}</div>;
}

// ❌ 不推荐：Server Component 使用 useEffect
export default function Page() {
  // useEffect 只能在 Client Component 中使用
  useEffect(() => {...}, []); // 错误！
}
```

### 9.2 合理使用缓存

```tsx
// 静态内容 - 使用缓存
export default async function BlogPage() {
  const posts = await fetch('https://api.example.com/posts', {
    cache: 'force-cache'  // 构建时缓存
  });
  return <PostsList posts={posts} />;
}

// 动态内容 - 不使用缓存
export default async function DashboardPage() {
  const userData = await fetch('https://api.example.com/user', {
    cache: 'no-store'  // 每次请求获取最新数据
  });
  return <Dashboard data={userData} />;
}

// 定时更新 - 使用 revalidate
export default async function NewsPage() {
  const news = await fetch('https://api.example.com/news', {
    next: { revalidate: 60 }  // 每 60 秒更新
  });
  return <NewsList news={news} />;
}
```

### 9.3 并行获取独立数据

```tsx
// ✅ 推荐：并行获取
export default async function Page() {
  const [users, posts] = await Promise.all([
    getUsers(),
    getPosts(),
  ]);
  return <Dashboard users={users} posts={posts} />;
}

// ❌ 不推荐：顺序获取独立数据
export default async function Page() {
  const users = await getUsers();     // 等待
  const posts = await getPosts();     // 再等待
  return <Dashboard users={users} posts={posts} />;
}
```

### 9.4 错误处理

```tsx
// 推荐：在数据获取函数中处理错误
async function getData() {
  const res = await fetch('https://api.example.com/data');

  if (!res.ok) {
    throw new Error('Failed to fetch data');
  }

  return res.json();
}

// 使用 error.tsx 捕获错误
export default async function Page() {
  const data = await getData();
  return <div>{data}</div>;
}

// app/posts/error.tsx
'use client';

export default function Error({
  error,
  reset,
}: {
  error: Error;
  reset: () => void;
}) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

### 9.5 组织数据获取代码

```tsx
// 推荐：将数据获取函数放在单独文件中
// lib/data.ts
export async function getUsers() {
  const res = await fetch('https://api.example.com/users');
  return res.json();
}

export async function getPosts() {
  const res = await fetch('https://api.example.com/posts');
  return res.json();
}

export async function getComments() {
  const res = await fetch('https://api.example.com/comments');
  return res.json();
}

// app/page.tsx
import { getUsers, getPosts, getComments } from '@/lib/data';

export default async function Page() {
  const [users, posts, comments] = await Promise.all([
    getUsers(),
    getPosts(),
    getComments(),
  ]);

  return <Dashboard users={users} posts={posts} comments={comments} />;
}
```

---

## 十、常见问题

### 10.1 为什么我的数据不更新？

**问题**：使用了默认缓存，数据被静态缓存了。

**解决方案**：

```tsx
// 方案 1：禁用缓存
fetch('https://api.example.com/data', {
  cache: 'no-store'
});

// 方案 2：设置定时更新
fetch('https://api.example.com/data', {
  next: { revalidate: 60 }
});

// 方案 3：手动触发重新验证
revalidatePath('/page');
revalidateTag('data');
```

### 10.2 Client Component 如何获取数据？

**方案**：

```tsx
// 方案 1：父级 Server Component 传递数据
export default async function Page() {
  const data = await getData();
  return <ClientComponent data={data} />;
}

// 方案 2：使用 SWR
'use client';
import useSWR from 'swr';

export function ClientComponent() {
  const { data } = useSWR('/api/data', fetcher);
  return <div>{data}</div>;
}

// 方案 3：使用 Server Actions
'use client';
import { refreshData } from './actions';

export function ClientComponent() {
  const handleClick = async () => {
    await refreshData();
  };
}
```

### 10.3 fetch 请求是否在服务端执行？

**是的**，Server Components 中的 fetch 在服务端执行：

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Server Component                                               │
│       │                                                         │
│       │ await fetch('https://api.example.com/data')             │
│       │                                                         │
│       ▼                                                         │
│  ┌────────────────────────────────────┐                         │
│  │  Next.js Server (Node.js)          │                         │
│  │  • fetch 在服务端执行               │                         │
│  │  • 不会暴露 API 密钥给客户端        │                         │
│  │  • 可直接访问数据库                 │                         │
│  └────────────────────────────────────┘                         │
│       │                                                         │
│       │ 请求外部 API                                              │
│       ▼                                                         │
│  External API                                                   │
│       │                                                         │
│       ▼                                                         │
│  返回数据 → 渲染 HTML → 发送给客户端                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 10.4 如何处理请求超时？

```tsx
// 使用 AbortController 设置超时
async function getData() {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), 5000);

  try {
    const res = await fetch('https://api.example.com/data', {
      signal: controller.signal,
    });
    clearTimeout(timeoutId);
    return res.json();
  } catch (error) {
    if (error.name === 'AbortError') {
      throw new Error('Request timeout');
    }
    throw error;
  }
}
```

---

## 参考资料

- [Data Fetching | Next.js](https://nextjs.org/docs/app/building-your-application/data-fetching)
- [Fetching | Next.js](https://nextjs.org/docs/app/building-your-application/data-fetching/fetching)
- [Server Components | Next.js](https://nextjs.org/docs/app/building-your-application/rendering/server-components)
- [Loading UI and Streaming | Next.js](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming)