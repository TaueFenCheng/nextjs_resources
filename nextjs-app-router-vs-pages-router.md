---
title: Next.js App Router vs Pages Router
description: 详细对比 Next.js 的 App Router 和 Pages Router 路由系统
---

# Next.js App Router vs Pages Router

本文详细对比 Next.js 的两种路由系统：Pages Router（旧版）和 App Router（新版）。

## 目录

1. [基本概念](#1-基本概念)
2. [路由文件命名对比](#2-路由文件命名对比)
3. [核心差异详解](#3-核心差异详解)
4. [特殊文件](#4-特殊文件)
5. [组件模型](#5-组件模型)
6. [数据流对比](#6-数据流对比)
7. [API 路由对比](#7-api-路由对比)
8. [路由组与并行路由](#8-路由组与并行路由)
9. [迁移建议](#9-迁移建议)
10. [总结对比表](#10-总结对比表)

---

## 1. 基本概念

### Pages Router（旧版）

- 基于 `pages/` 目录的文件系统路由
- 2016 年 Next.js 发布时就存在
- 所有页面默认是服务端渲染（SSR）
- 当前处于维护模式，不再添加新功能

### App Router（新版）

- 基于 `app/` 目录的文件系统路由
- 2023 年 Next.js 13 正式发布
- 引入 React Server Components
- 活跃开发中，是 Next.js 的未来方向

### 目录结构对比

```
Pages Router 结构:
pages/
├── index.tsx          → /
├── about.tsx          → /about
├── blog/
│   ├── index.tsx      → /blog
│   └── [slug].tsx     → /blog/:slug
└── api/
    └── hello.ts       → /api/hello

App Router 结构:
app/
├── page.tsx           → /
├── about/
│   └── page.tsx       → /about
├── blog/
│   ├── page.tsx       → /blog
│   └── [slug]/
│       └── page.tsx   → /blog/:slug
└── api/
    └── hello/
        └── route.ts   → /api/hello
```

---

## 2. 路由文件命名对比

| 功能 | Pages Router | App Router |
|------|--------------|------------|
| 页面 | `pages/index.tsx` | `app/page.tsx` |
| 动态路由 | `pages/blog/[slug].tsx` | `app/blog/[slug]/page.tsx` |
| 嵌套路由 | 需要手动组合 | 文件夹层级即路由层级 |
| API 路由 | `pages/api/hello.ts` | `app/api/hello/route.ts` |
| 布局 | `_app.tsx` / `_document.tsx` | `layout.tsx` |
| 404 页面 | `pages/404.tsx` | `app/not-found.tsx` |
| 错误页面 | `pages/_error.tsx` | `app/error.tsx` |
| 加载状态 | 手动实现 | `loading.tsx` |

---

## 3. 核心差异详解

### 3.1 渲染模式

#### Pages Router

```
┌─────────────────────────────────────────────────────────────────┐
│                        Pages Router                              │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐                                                │
│  │   Client    │  所有组件默认是客户端组件                        │
│  │  Component  │  使用 getServerSideProps / getStaticProps       │
│  │   (默认)    │  在服务端获取数据后传输给客户端                    │
│  └─────────────┘                                                │
└─────────────────────────────────────────────────────────────────┘
```

- 所有组件在客户端执行
- 数据通过 `getServerSideProps` 或 `getStaticProps` 在服务端获取
- 数据必须序列化后传输到客户端
- 整个页面需要等待所有数据加载完成

#### App Router

```
┌─────────────────────────────────────────────────────────────────┐
│                        App Router                                │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐     ┌─────────────┐                           │
│  │   Server    │     │   Client    │                           │
│  │  Component  │────▶│  Component  │                           │
│  │   (默认)    │     │ ("use client")│                         │
│  └─────────────┘     └─────────────┘                           │
│                                                                 │
│  Server Component 在服务端渲染，只传输 HTML 到客户端               │
│  Client Component 在浏览器中执行 JavaScript                      │
└─────────────────────────────────────────────────────────────────┘
```

- Server Component 默认，在服务端渲染
- 只传输 HTML 到客户端，减少 JavaScript 包大小
- 支持流式渲染（Streaming）
- 数据获取可以并行进行

### 3.2 数据获取方式

#### Pages Router

```tsx
// pages/blog/[slug].tsx
import { GetServerSideProps, GetStaticPaths, GetStaticProps } from 'next';

// SSR - 每次请求时获取数据
export const getServerSideProps: GetServerSideProps = async (context) => {
  const { slug } = context.params;
  const post = await fetchPost(slug);

  return {
    props: { post },  // 序列化后传给客户端
  };
};

// SSG - 构建时获取数据
export const getStaticProps: GetStaticProps = async (context) => {
  const { slug } = context.params;
  const post = await fetchPost(slug);

  return {
    props: { post },
    revalidate: 60,  // ISR: 60秒后重新生成
  };
};

// 静态路径生成
export const getStaticPaths: GetStaticPaths = async () => {
  const posts = await fetchPosts();

  return {
    paths: posts.map(p => ({ params: { slug: p.slug } })),
    fallback: 'blocking',
  };
};

export default function BlogPost({ post }) {
  return <article>{post.title}</article>;
}
```

#### App Router

```tsx
// app/blog/[slug]/page.tsx

// 直接使用 async/await，无需特殊函数
async function BlogPost({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await fetchPost(slug);  // 自动缓存

  return <article>{post.title}</article>;
}

export default BlogPost;

// 生成静态参数（替代 getStaticPaths）
export async function generateStaticParams() {
  const posts = await fetchPosts();
  return posts.map(p => ({ slug: p.slug }));
}

// 配置选项
export const dynamic = 'force-static';  // 强制静态
export const revalidate = 60;            // ISR 重新验证时间
```

**数据获取配置对比**

| Pages Router | App Router |
|--------------|------------|
| `getServerSideProps` | async 组件函数 + `export const dynamic = 'force-dynamic'` |
| `getStaticProps` | async 组件函数（默认缓存） |
| `getStaticPaths` | `generateStaticParams()` |
| `revalidate` | `export const revalidate = 60` |

### 3.3 布局系统

#### Pages Router - 单一布局

```tsx
// pages/_app.tsx - 全局布局（只有一个）
import type { AppProps } from 'next/app';
import Layout from '@/components/Layout';

export default function App({ Component, pageProps }: AppProps) {
  return (
    <Layout>
      <Component {...pageProps} />
    </Layout>
  );
}
```

**限制：**
- 无法实现嵌套布局
- 每个页面需要手动包装布局组件
- 切换页面时整个布局会重新渲染

#### App Router - 嵌套布局

```tsx
// app/layout.tsx - 根布局
export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <Header />
        {children}
        <Footer />
      </body>
    </html>
  );
}

// app/dashboard/layout.tsx - Dashboard 布局
export default function DashboardLayout({ children }) {
  return (
    <div className="dashboard">
      <Sidebar />
      <main>{children}</main>
    </div>
  );
}

// app/dashboard/settings/layout.tsx - 设置页面布局
export default function SettingsLayout({ children }) {
  return (
    <div className="settings">
      <SettingsNav />
      {children}
    </div>
  );
}
```

**布局嵌套结构：**

```
┌─────────────────────────────────────────────────┐
│                  Root Layout                     │
│  ┌───────────────────────────────────────────┐  │
│  │              Header                        │  │
│  └───────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────┐  │
│  │           Dashboard Layout                 │  │
│  │  ┌─────────┐  ┌────────────────────────┐  │  │
│  │  │ Sidebar │  │   Settings Layout      │  │  │
│  │  │         │  │  ┌──────────────────┐  │  │  │
│  │  │         │  │  │ SettingsNav      │  │  │  │
│  │  │         │  │  ├──────────────────┤  │  │  │
│  │  │         │  │  │    Page          │  │  │  │
│  │  │         │  │  │    Content       │  │  │  │
│  │  │         │  │  └──────────────────┘  │  │  │
│  │  │         │  └────────────────────────┘  │  │
│  │  └─────────┘                              │  │
│  └───────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────┐  │
│  │              Footer                        │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

**优势：**
- 布局之间可以嵌套
- 切换子页面时父布局保持状态
- Sidebar 导航状态不会丢失
- 减少不必要的重新渲染

---

## 4. 特殊文件

| 文件 | Pages Router | App Router | 说明 |
|------|-------------|------------|------|
| 布局 | `_app.tsx` | `layout.tsx` | App Router 支持嵌套 |
| 文档 | `_document.tsx` | 根 `layout.tsx` | HTML 结构定义 |
| 错误 | `_error.tsx` | `error.tsx` | 错误边界处理 |
| 404 | `404.tsx` | `not-found.tsx` | 未找到页面 |
| 加载 | 手动实现 | `loading.tsx` | 自动 Suspense |
| 模板 | 无 | `template.tsx` | 类似布局但每次重新挂载 |
| 默认 | 无 | `default.tsx` | 并行路由默认页面 |

### App Router 特殊文件详解

#### layout.tsx

定义共享布局，嵌套时保持状态：

```tsx
export default function Layout({ children }) {
  return <div className="container">{children}</div>;
}
```

#### template.tsx

类似布局，但导航时会重新挂载：

```tsx
export default function Template({ children }) {
  return <div className="animate-fade-in">{children}</div>;
}
```

#### loading.tsx

自动创建 Suspense 边界：

```tsx
export default function Loading() {
  return <Spinner />;
}
```

#### error.tsx

React Error Boundary：

```tsx
'use client';

export default function Error({ error, reset }) {
  return (
    <div>
      <h2>出错了！</h2>
      <button onClick={reset}>重试</button>
    </div>
  );
}
```

#### not-found.tsx

处理 404：

```tsx
import Link from 'next/link';

export default function NotFound() {
  return (
    <div>
      <h2>页面未找到</h2>
      <Link href="/">返回首页</Link>
    </div>
  );
}
```

---

## 5. 组件模型

### Server Component（默认）

```
┌─────────────────────────────────────────────────────────┐
│                  Server Component                        │
├─────────────────────────────────────────────────────────┤
│ ✅ 直接访问数据库、文件系统                               │
│ ✅ 使用敏感信息（API Key）                                │
│ ✅ 减少客户端 JavaScript 包大小                           │
│ ✅ async/await 直接获取数据                               │
│ ❌ 不能使用 useState、useEffect、事件处理                 │
│ ❌ 不能使用浏览器 API（window、document）                 │
└─────────────────────────────────────────────────────────┘
```

### Client Component（"use client"）

```
┌─────────────────────────────────────────────────────────┐
│                  Client Component                        │
├─────────────────────────────────────────────────────────┤
│ ✅ 使用 useState、useEffect 等 hooks                      │
│ ✅ 使用事件处理 (onClick, onSubmit)                       │
│ ✅ 使用浏览器 API                                         │
│ ✅ 使用 Context API                                       │
│ ❌ 不能直接访问数据库                                     │
│ ❌ 不能使用敏感信息（会被发送到客户端）                    │
└─────────────────────────────────────────────────────────┘
```

### 组合规则

```
Server Component → 可以导入 → Client Component  ✅
Client Component → 不能导入 → Server Component  ❌
```

### 示例：正确组合

```tsx
// ✅ 正确：Server Component 导入 Client Component
// app/page.tsx (Server Component)
import LikeButton from './LikeButton';

export default async function Page() {
  const post = await fetchPost();

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
      <LikeButton postId={post.id} />
    </article>
  );
}

// app/LikeButton.tsx (Client Component)
'use client';

import { useState } from 'react';

export default function LikeButton({ postId }: { postId: string }) {
  const [liked, setLiked] = useState(false);

  return (
    <button onClick={() => setLiked(!liked)}>
      {liked ? '❤️' : '🤍'}
    </button>
  );
}
```

### 示例：错误用法

```tsx
// ❌ 错误：Client Component 导入 Server Component
// app/ClientPage.tsx
'use client';

import ServerChild from './ServerChild';  // ❌ 不能这样做！

export default function ClientPage() {
  return <ServerChild />;
}

// ✅ 正确方式：通过 children prop 传递
// app/page.tsx
import ClientWrapper from './ClientWrapper';
import ServerChild from './ServerChild';

export default function Page() {
  return (
    <ClientWrapper>
      <ServerChild />  {/* 作为 children 传递 ✅ */}
    </ClientWrapper>
  );
}
```

---

## 6. 数据流对比

### Pages Router 数据流

```
┌──────────────────────────────────────────────────────────────┐
│                      Pages Router 数据流                      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  浏览器请求 ──▶ Next.js Server                               │
│                    │                                         │
│                    ▼                                         │
│         getServerSideProps() 执行                            │
│                    │                                         │
│                    ▼                                         │
│         数据序列化为 JSON                                     │
│                    │                                         │
│                    ▼                                         │
│         传输到客户端 (props)                                  │
│                    │                                         │
│                    ▼                                         │
│         客户端 hydration                                      │
│                    │                                         │
│                    ▼                                         │
│         React 组件渲染                                        │
│                                                              │
│  特点: 所有数据在页面加载前就要准备好                           │
│       数据必须可序列化 (JSON)                                  │
│       客户端需要等待所有数据                                   │
└──────────────────────────────────────────────────────────────┘
```

### App Router 数据流

```
┌──────────────────────────────────────────────────────────────┐
│                       App Router 数据流                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  浏览器请求 ──▶ Next.js Server                               │
│                    │                                         │
│                    ▼                                         │
│         Server Component 树渲染                              │
│         （并行获取数据）                                       │
│                    │                                         │
│         ┌─────────┴─────────┐                               │
│         ▼                   ▼                                │
│  Server Component A   Server Component B                     │
│  async fetchA()       async fetchB()                         │
│         │                   │                               │
│         └─────────┬─────────┘                               │
│                   ▼                                          │
│         生成 HTML + Client Component 指令                     │
│                   │                                          │
│                   ▼                                          │
│         流式传输 HTML 到客户端 (Streaming)                    │
│                   │                                          │
│                   ▼                                          │
│         客户端快速显示 HTML                                   │
│                   │                                          │
│                   ▼                                          │
│         加载 Client Component JS                              │
│                   │                                          │
│                   ▼                                          │
│         Hydration（只针对 Client Component）                  │
│                                                              │
│  特点: Server Component 数据不传到客户端                       │
│       支持流式渲染 (Streaming SSR)                             │
│       数据获取可以并行                                         │
│       用户可以更快看到内容                                     │
└──────────────────────────────────────────────────────────────┘
```

### 流式渲染优势

```tsx
// app/page.tsx
import { Suspense } from 'react';

async function SlowComponent() {
  await new Promise(r => setTimeout(r, 3000));
  return <div>慢加载完成</div>;
}

async function FastComponent() {
  return <div>快速内容</div>;
}

export default function Page() {
  return (
    <div>
      <FastComponent />  {/* 立即显示 */}
      <Suspense fallback={<Spinner />}>
        <SlowComponent />  {/* 3秒后显示 */}
      </Suspense>
    </div>
  );
}
```

---

## 7. API 路由对比

### Pages Router API

```ts
// pages/api/users/[id].ts
import type { NextApiRequest, NextApiResponse } from 'next';

export default function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  const { id } = req.query;

  if (req.method === 'GET') {
    res.status(200).json({ id, name: 'User ' + id });
  } else if (req.method === 'POST') {
    res.status(201).json({ message: 'Created' });
  } else {
    res.status(405).end();  // Method Not Allowed
  }
}
```

### App Router API

```ts
// app/api/users/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server';

// 每个 HTTP 方法是独立导出
export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  return NextResponse.json({ id, name: 'User ' + id });
}

export async function POST(request: NextRequest) {
  const body = await request.json();
  return NextResponse.json({ message: 'Created' }, { status: 201 });
}

// 支持 HEAD, PUT, PATCH, DELETE, OPTIONS
export async function DELETE(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  return NextResponse.json({ deleted: id });
}
```

### App Router API 优势

#### 流式响应

```ts
export async function GET() {
  const encoder = new TextEncoder();
  const stream = new ReadableStream({
    async start(controller) {
      for (let i = 0; i < 10; i++) {
        controller.enqueue(encoder.encode(`data: ${i}\n\n`));
        await new Promise(r => setTimeout(r, 1000));
      }
      controller.close();
    },
  });

  return new Response(stream, {
    headers: { 'Content-Type': 'text/event-stream' },
  });
}
```

#### 使用 Web 标准

```ts
// FormData 处理
export async function POST(request: Request) {
  const formData = await request.formData();
  const file = formData.get('file') as File;
  const bytes = await file.arrayBuffer();

  return new Response('Uploaded', { status: 201 });
}

// Headers 处理
export async function GET(request: Request) {
  const auth = request.headers.get('Authorization');

  if (!auth) {
    return new Response('Unauthorized', { status: 401 });
  }

  return NextResponse.json({ data: 'secret' });
}
```

---

## 8. 路由组与并行路由

### 路由组 (Route Groups)

使用 `(groupName)` 格式，不影响 URL：

```
app/
├── (marketing)/          ← 路由组，不影响 URL
│   ├── layout.tsx        ← 共享布局
│   ├── about/
│   │   └── page.tsx      → /about
│   └── contact/
│       └── page.tsx      → /contact
├── (shop)/
│   ├── layout.tsx        ← 不同的布局
│   ├── products/
│   │   └── page.tsx      → /products
│   └── cart/
│       └── page.tsx      → /cart
```

**用途：**
- 组织路由不影响 URL 结构
- 不同路由组可以有不同的布局
- 方便模块化管理

### 并行路由 (Parallel Routes)

使用 `@slotName` 格式，同时渲染多个区域：

```
app/
├── @team/               ← 并行路由槽
│   └── members/
│       └── page.tsx
├── @analytics/          ← 并行路由槽
│   └── page.tsx
└── layout.tsx
```

```tsx
// app/layout.tsx
export default function Layout({
  children,
  team,
  analytics,
}: {
  children: React.ReactNode;
  team: React.ReactNode;
  analytics: React.ReactNode;
}) {
  return (
    <div>
      <main>{children}</main>
      <aside>{team}</aside>
      <aside>{analytics}</aside>
    </div>
  );
}
```

**用途：**
- 同时渲染多个独立区域
- 每个区域可以独立加载
- 实现复杂的布局结构

### 拦截路由 (Intercepting Routes)

使用 `(..)` 格式拦截路由：

```
app/
├── @modal/
│   └── (.)photo/        ← 拦截同层级的 /photo
│   │   └── page.tsx
│   └── (..)photo/       ← 拦截上一层的 /photo
│   │   └── page.tsx
│   └── (..)(..)photo/   ← 拦截上两层的 /photo
│   │   └── page.tsx
│   └── (...)photo/      ← 拦截根层的 /photo
│       └── page.tsx
├── photo/
│   └── [id]/
│       └── page.tsx     ← 实际的 /photo/:id 页面
└── layout.tsx
```

**用途：**
- 在当前页面打开模态框
- 刷新时显示完整页面
- Instagram/Facebook 风格的图片查看

---

## 9. 迁移建议

### 何时选择 Pages Router

- 已有项目，迁移成本高
- 需要使用某些 Pages Router 特有功能
- 团队对 App Router 不熟悉，想保持稳定
- 快速原型开发

### 何时选择 App Router

- 新项目
- 需要 Server Components 优化性能
- 需要嵌套布局保持状态
- 需要流式渲染提升用户体验
- 需要更好的 SEO 支持
- 项目规模较大，需要模块化路由

### 迁移步骤

```bash
# 1. 升级 Next.js 到 13+
npm install next@latest react@latest react-dom@latest

# 2. 创建 app 目录
mkdir app

# 3. 创建根布局
# app/layout.tsx
```

```tsx
// app/layout.tsx
export default function RootLayout({ children }) {
  return (
    <html lang="zh">
      <body>{children}</body>
    </html>
  );
}
```

```bash
# 4. 逐步迁移页面
# pages/index.tsx → app/page.tsx

# 5. 修改数据获取
# getServerSideProps → async 组件函数

# 6. 更新 API 路由
# pages/api/* → app/api/*/route.ts

# 7. 保持 pages 和 app 共存（渐进迁移）
# Next.js 支持同时存在两种路由
```

### 数据获取迁移对照

| Pages Router | App Router |
|--------------|------------|
| `getServerSideProps` | `async function Page()` + `dynamic = 'force-dynamic'` |
| `getStaticProps` | `async function Page()` + 默认缓存 |
| `getStaticPaths` | `generateStaticParams()` |
| `getInitialProps` | 不推荐使用 |

---

## 10. 总结对比表

| 特性 | Pages Router | App Router |
|------|--------------|------------|
| 目录 | `pages/` | `app/` |
| 页面文件 | `*.tsx` | `page.tsx` |
| 默认渲染 | 客户端组件 | 服务端组件 |
| 数据获取 | getServerSideProps, getStaticProps | async/await |
| 布局 | 单一 _app.tsx | 嵌套 layout.tsx |
| API 路由 | handler 函数 | HTTP 方法导出 |
| 流式渲染 | 需手动配置 | 原生支持 |
| 组件类型 | 全是客户端组件 | Server/Client 组件 |
| 包大小 | 较大 | 较小 |
| 并行路由 | 不支持 | 支持 |
| 路由组 | 不支持 | 支持 |
| 拦截路由 | 不支持 | 支持 |
| 模板 | 不支持 | 支持 |
| Suspense | 需手动实现 | loading.tsx 自动 |
| 错误边界 | 需手动实现 | error.tsx 自动 |
| params | 同步对象 | Promise |
| searchParams | 同步对象 | Promise |
| 学习曲线 | 较低 | 较高 |
| 文档完善度 | 完善 | 完善 |
| 维护状态 | 维护模式 | 活跃开发 |
| 未来方向 | 逐步淘汰 | 主要方向 |

---

## 最佳实践建议

1. **新项目**：直接使用 App Router
2. **大型项目**：优先 App Router，享受性能优势
3. **迁移项目**：渐进迁移，保持 pages 和 app 共存
4. **Server Component**：尽可能使用，减少客户端包大小
5. **Client Component**：只在需要交互时使用
6. **布局**：利用嵌套布局保持状态
7. **流式渲染**：使用 Suspense 提升用户体验

---

## 参考资源

- [Next.js App Router 文档](https://nextjs.org/docs/app)
- [Next.js Pages Router 文档](https://nextjs.org/docs/pages)
- [React Server Components](https://react.dev/blog/2023/03/22/react-labs-what-we-have-been-working-on-march-2023#react-server-components)