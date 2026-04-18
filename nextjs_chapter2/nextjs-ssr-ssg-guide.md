# Next.js SSR 与 SSG 完整指南

本文档详细介绍 Next.js 中 SSR（服务端渲染）和 SSG（静态站点生成）的区别、使用方式以及如何搭建一个纯静态 SSG 项目。

---

## 目录

1. [核心概念对比](#一核心概念对比)
2. [渲染时机详解](#二渲染时机详解)
3. [Next.js App Router 中的实现](#三nextjs-app-router-中的实现)
4. [ISR 增量静态再生](#四isr-增量静态再生)
5. [搭建 SSG 项目完整步骤](#五搭建-ssg-项目完整步骤)
6. [完整项目示例](#六完整项目示例)
7. [SSG 的限制](#七ssg-的限制)
8. [选择指南](#八选择指南)
9. [常见问题](#九常见问题)

---

## 一、核心概念对比

### 1.1 定义

| 渲染方式 | 定义 |
|---------|------|
| **SSR (Server-Side Rendering)** | 每次用户请求时，服务器实时渲染 HTML 并返回 |
| **SSG (Static Site Generation)** | 构建时预先渲染所有页面，生成静态 HTML 文件 |
| **ISR (Incremental Static Regeneration)** | SSG + 定时更新，静态页面按需重新生成 |

### 1.2 核心差异对比表

| 特性 | SSR | SSG | ISR |
|------|-----|-----|-----|
| **渲染时机** | 每次请求时 | 构建时 | 构建时 + 定时更新 |
| **数据新鲜度** | 实时最新 | 构建时固定 | 按配置间隔更新 |
| **响应速度** | 较慢（需等待渲染） | 极快（直接返回静态文件） | 快（静态 + 命中时更新） |
| **服务器负载** | 高（每次请求都渲染） | 极低（只返回文件） | 低（按需更新） |
| **服务器要求** | 需要 Node.js 服务器 | 可部署到 CDN | 需要服务器（轻量） |
| **部署成本** | 高 | 低（静态托管） | 中 |
| **SEO** | ✅ 友好 | ✅ 友好 | ✅ 友好 |

### 1.3 适用场景

| 渲染方式 | 适用场景 |
|---------|---------|
| **SSR** | 用户仪表盘、实时数据、个性化内容、电商购物车 |
| **SSG** | 博客、文档网站、营销页面、静态展示网站 |
| **ISR** | 内容网站、新闻站点、产品页面（偶尔更新） |

---

## 二、渲染时机详解

### 2.1 SSR 渲染流程

```
用户发起请求
    │
    │ 例如：GET /dashboard
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Node.js 服务器                               │
│                                                                 │
│  Step 1: 接收请求                                               │
│          解析 URL、参数、cookies                                 │
│                                                                 │
│  Step 2: 获取数据                                               │
│          fetch('/api/user-data')                                │
│          await database.query()                                 │
│          ── 实时获取最新数据                                      │
│                                                                 │
│  Step 3: 渲染组件                                               │
│          执行 React 组件代码                                     │
│          生成 HTML                                              │
│          ── 每次请求都重新渲染                                    │
│                                                                 │
│  Step 4: 返回 HTML                                              │
│          发送完整 HTML 给浏览器                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
    │
    │ 返回渲染后的 HTML
    │
    ▼
浏览器显示页面

时间消耗：
├─ 网络请求
├─ 服务端处理（数据获取 + 渲染）
├─ HTML 传输
└─ 总计：几百毫秒到几秒
```

### 2.2 SSG 渲染流程

```
构建阶段（pnpm build）：

┌─────────────────────────────────────────────────────────────────┐
│                     构建时预渲染                                  │
│                                                                 │
│  Step 1: 分析路由                                               │
│          扫描 app/ 目录                                          │
│          识别所有页面和动态路由                                   │
│                                                                 │
│  Step 2: 执行 generateStaticParams                              │
│          获取所有动态路径参数                                     │
│          例如：['post-1', 'post-2', 'post-3']                   │
│                                                                 │
│  Step 3: 预渲染每个页面                                         │
│          执行组件代码                                            │
│          获取数据（此时是构建时，非实时）                          │
│          生成静态 HTML                                           │
│                                                                 │
│  Step 4: 输出到 out/ 目录                                       │
│          out/index.html                                         │
│          out/blog/post-1/index.html                             │
│          out/blog/post-2/index.html                             │
│          ...                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

用户请求阶段：

用户发起请求
    │
    │ 例如：GET /blog/post-1
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│                     CDN / 静态服务器                             │
│                                                                 │
│  直接返回静态文件：                                              │
│  out/blog/post-1/index.html                                     │
│                                                                 │
│  无需渲染！无需数据获取！                                         │
│  直接读取磁盘文件返回                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
    │
    │ 立即返回 HTML
    │
    ▼
浏览器显示页面

时间消耗：
├─ 网络请求（极快，CDN 边缘节点）
├─ 无服务端处理
├─ HTML 传输（已缓存）
└─ 总计：几十毫秒
```

### 2.3 ISR 渲染流程

```
构建阶段（与 SSG 相同）：

预渲染所有页面 → 输出静态 HTML

用户请求阶段：

用户请求 /blog/post-1
    │
    ▼
检查缓存：
    │
    ├─ 缓存存在且未过期？
    │   │
    │   └─ 是 → 直接返回静态 HTML（极快）
    │   │
    │   └─ 否（超过 revalidate 时间）？
    │       │
    │       ├─ 先返回旧的静态 HTML（用户立即看到）
    │       │
    │       ├─ 后台触发重新生成
    │       │   ├─ 获取最新数据
    │       │   ├─ 重新渲染
    │       │   └─ 更新缓存
    │       │
    │       └─ 下次请求返回新的 HTML
    │
    ▼
浏览器显示页面

配置示例：
export const revalidate = 3600;  // 每小时更新一次
```

---

## 三、Next.js App Router 中的实现

### 3.1 SSR（默认行为）

Next.js App Router 默认使用 SSR：

```tsx
// app/dashboard/page.tsx
// 默认是 SSR（Server Component）

export default async function DashboardPage() {
  // ✅ 每次请求时执行
  const userData = await fetchCurrentUser();    // 实时获取
  const notifications = await fetchNotifications();  // 实时获取
  
  return (
    <div>
      <h1>Welcome, {userData.name}</h1>
      <NotificationsList data={notifications} />
    </div>
  );
}

// 每次用户访问：
// 1. 服务器接收请求
// 2. 执行 fetchCurrentUser()、fetchNotifications()
// 3. 渲染组件生成 HTML
// 4. 返回给浏览器
```

**特点**：
- 默认行为，无需额外配置
- 数据始终是最新的
- 适合动态内容

### 3.2 SSG（静态生成）

#### 方式一：generateStaticParams

```tsx
// app/blog/[slug]/page.tsx

interface PageProps {
  params: Promise<{ slug: string }>;
}

// 🔑 关键：定义静态路径
export async function generateStaticParams() {
  // 构建时执行，获取所有文章 slug
  const posts = await getAllPosts();  // 从 CMS、数据库、文件系统
  
  return posts.map(post => ({
    slug: post.slug,
  }));
  
  // 返回结果：
  // [
  //   { slug: 'hello-world' },
  //   { slug: 'nextjs-guide' },
  //   { slug: 'typescript-tips' }
  // ]
  
  // 构建时会生成：
  // /blog/hello-world
  // /blog/nextjs-guide
  // /blog/typescript-tips
}

export default async function BlogPost({ params }: PageProps) {
  const { slug } = await params;
  
  // 构建时执行（不是请求时）
  const post = await getPostBySlug(slug);
  
  return (
    <article>
      <h1>{post.title}</h1>
      <time>{post.date}</time>
      <div>{post.content}</div>
    </article>
  );
}

// 获取文章数据（构建时调用）
async function getAllPosts() {
  // 从本地 Markdown 文件读取
  const files = fs.readdirSync('content/posts');
  return files.map(file => ({
    slug: file.replace('.md', ''),
  }));
}
```

#### 方式二：静态导出

```typescript
// next.config.ts

import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "export",  // 🔑 关键配置
  images: {
    unoptimized: true,  // 静态导出不支持图片优化
  },
  trailingSlash: true,  // URL 统一带 /
};

export default nextConfig;
```

**`output: 'export'` 的效果**：

```
pnpm build 输出：

out/
├── index.html              ← 首页
├── blog/
│   ├── index.html          ← 博客列表
│   │
│   ├── hello-world/
│   │   └── index.html      ← 文章详情
│   │
│   ├── nextjs-guide/
│   │   └── index.html
│   │
│   └── typescript-tips/
│   │       └── index.html
│   │
│   └── tags/
│   │   ├── nextjs/
│   │   │   └── index.html
│   │   │
│   │   └── typescript/
│   │       └── index.html
│   │
├── about/
│   └── index.html
│   │
└── _next/
    └── static/             ← JS、CSS 等静态资源
```

### 3.3 静态页面（无动态数据）

```tsx
// app/about/page.tsx
// 纯静态页面，无需数据获取

export default function AboutPage() {
  return (
    <div>
      <h1>About Us</h1>
      <p>This is a static page with hardcoded content.</p>
    </div>
  );
}

// 构建时生成静态 HTML
// 无需 generateStaticParams（无动态参数）
```

---

## 四、ISR 增量静态再生

### 4.1 基本配置

```tsx
// app/blog/[slug]/page.tsx

// 🔑 关键：设置重新验证时间
export const revalidate = 3600;  // 3600 秒 = 1 小时

export async function generateStaticParams() {
  const posts = await getAllPosts();
  return posts.map(post => ({ slug: post.slug }));
}

export default async function BlogPost({ params }: PageProps) {
  const { slug } = await params;
  const post = await getPostBySlug(slug);
  
  return <article>{post.content}</article>;
}
```

### 4.2 ISR 工作机制

```
时间线示例（revalidate = 3600）：

10:00 - 用户 A 访问 /blog/post-1
        │
        ├─ 缓存存在（构建时生成）
        ├─ 返回静态 HTML
        └─ 缓存时间标记：10:00
        
11:30 - 用户 B 访问 /blog/post-1
        │
        ├─ 缓存存在
        ├─ 检查：11:30 - 10:00 = 90 分钟 < 60 分钟？
        ├─ 否，但不超过 revalidate
        ├─ 返回静态 HTML（仍有效）
        
12:01 - 用户 C 访问 /blog/post-1
        │
        ├─ 缓存存在
        ├─ 检查：12:01 - 10:00 = 120 分钟 > 60 分钟？
        ├─ 是，缓存过期！
        ├─ 立即返回旧的静态 HTML（用户看到内容）
        ├─ 同时触发后台重新生成
        │   ├─ fetch 最新数据
        │   ├─ 重新渲染
        │   ├─ 更新缓存
        │   └─ 新缓存时间标记：12:01
        
12:05 - 用户 D 访问 /blog/post-1
        │
        ├─ 缓存已更新（12:01）
        ├─ 返回新的静态 HTML
```

### 4.3 按需重新验证

```tsx
// API Route 触发重新验证
// app/api/revalidate/route.ts

import { revalidatePath, revalidateTag } from "next/cache";

export async function POST(request: Request) {
  const { path, tag } = await request.json();
  
  // 重新验证特定路径
  revalidatePath(`/blog/${path}`);
  
  // 或重新验证特定标签
  revalidateTag("blog-posts");
  
  return Response.json({ revalidated: true });
}

// 博客文章页面使用标签
export default async function BlogPost({ params }) {
  const post = await fetch(`/api/posts/${params.slug}`, {
    next: { tags: ["blog-posts"] }  // 🔑 关键：使用标签
  });
  
  // ...
}
```

---

## 五、搭建 SSG 项目完整步骤

### 5.1 创建项目

```bash
# 创建 Next.js 项目
pnpm create next-app@latest my-ssg-blog

# 配置选择：
# ✔ TypeScript: Yes
# ✔ ESLint: Yes
# ✔ Tailwind CSS: Yes
# ✔ `src/` directory: Yes
# ✔ App Router: Yes
# ✔ Turbopack for `dev`: Yes
# ✔ Customize import alias: No

# 进入项目
cd my-ssg-blog
```

### 5.2 配置静态导出

```typescript
// next.config.ts

import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  // 🔑 关键配置：静态导出
  output: "export",
  
  // 静态导出的必要配置
  images: {
    unoptimized: true,  // 静态导出不支持 Next.js 图片优化
  },
  
  // 可选：URL 统一带尾部斜杠
  trailingSlash: true,
  
  // 可选：自定义输出目录
  // distDir: "dist",
};

export default nextConfig;
```

### 5.3 创建内容目录

```bash
# 创建 Markdown 文章目录
mkdir -p content/posts

# 创建示例文章
touch content/posts/hello-world.md
touch content/posts/nextjs-ssg.md
touch content/posts/typescript-tips.md
```

### 5.4 编写文章内容

```markdown
# content/posts/hello-world.md

---
title: Hello World
date: 2024-01-15
tags: [intro, beginner]
---

This is my first blog post using Next.js SSG.

## Introduction

Static Site Generation is amazing!

- Fast loading
- SEO friendly
- Low cost deployment
```

### 5.5 创建文章解析工具

```typescript
// lib/posts.ts

import fs from "fs";
import path from "path";
import matter from "gray-matter";  // 需要：pnpm add gray-matter

const postsDirectory = path.join(process.cwd(), "content/posts");

// 获取所有文章元数据
export function getAllPosts() {
  const fileNames = fs.readdirSync(postsDirectory);
  
  const posts = fileNames.map(fileName => {
    const slug = fileName.replace(".md", "");
    const fullPath = path.join(postsDirectory, fileName);
    const fileContents = fs.readFileSync(fullPath, "utf8");
    const { data } = matter(fileContents);  // 解析 frontmatter
    
    return {
      slug,
      title: data.title,
      date: data.date,
      tags: data.tags,
    };
  });
  
  // 按日期排序
  return posts.sort((a, b) => 
    new Date(b.date).getTime() - new Date(a.date).getTime()
  );
}

// 获取单篇文章内容
export function getPostBySlug(slug: string) {
  const fullPath = path.join(postsDirectory, `${slug}.md`);
  const fileContents = fs.readFileSync(fullPath, "utf8");
  const { data, content } = matter(fileContents);
  
  return {
    slug,
    title: data.title,
    date: data.date,
    tags: data.tags,
    content,
  };
}

// 获取所有标签
export function getAllTags() {
  const posts = getAllPosts();
  const tagsSet = new Set<string>();
  
  posts.forEach(post => {
    post.tags?.forEach(tag => tagsSet.add(tag));
  });
  
  return Array.from(tagsSet);
}
```

### 5.6 创建博客列表页

```tsx
// app/blog/page.tsx

import Link from "next/link";
import { getAllPosts } from "@/lib/posts";

export default function BlogListPage() {
  const posts = getAllPosts();
  
  return (
    <div className="max-w-2xl mx-auto py-8">
      <h1 className="text-3xl font-bold mb-8">Blog</h1>
      
      <ul className="space-y-4">
        {posts.map(post => (
          <li key={post.slug}>
            <Link 
              href={`/blog/${post.slug}`}
              className="block p-4 border rounded hover:bg-gray-50"
            >
              <h2 className="text-xl font-semibold">{post.title}</h2>
              <time className="text-gray-500">{post.date}</time>
              {post.tags && (
                <div className="flex gap-2 mt-2">
                  {post.tags.map(tag => (
                    <span 
                      key={tag}
                      className="text-xs bg-gray-200 px-2 py-1 rounded"
                    >
                      {tag}
                    </span>
                  ))}
                </div>
              )}
            </Link>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### 5.7 创建文章详情页

```tsx
// app/blog/[slug]/page.tsx

import { notFound } from "next/navigation";
import { getAllPosts, getPostBySlug } from "@/lib/posts";

interface PageProps {
  params: Promise<{ slug: string }>;
}

// 🔑 关键：定义静态路径
export async function generateStaticParams() {
  const posts = getAllPosts();
  
  return posts.map(post => ({
    slug: post.slug,
  }));
}

// 生成元数据（SEO）
export async function generateMetadata({ params }: PageProps) {
  const { slug } = await params;
  const post = getPostBySlug(slug);
  
  return {
    title: post.title,
    description: post.content.slice(0, 160),
  };
}

export default async function BlogPostPage({ params }: PageProps) {
  const { slug } = await params;
  
  // 获取文章内容
  const post = getPostBySlug(slug);
  
  if (!post) {
    notFound();
  }
  
  return (
    <article className="max-w-2xl mx-auto py-8">
      <header className="mb-8">
        <h1 className="text-3xl font-bold">{post.title}</h1>
        <time className="text-gray-500">{post.date}</time>
      </header>
      
      {/* 文章内容 */}
      <div className="prose">
        {post.content}
      </div>
      
      {/* 标签 */}
      {post.tags && (
        <div className="mt-8 flex gap-2">
          {post.tags.map(tag => (
            <span 
              key={tag}
              className="text-sm bg-gray-200 px-3 py-1 rounded"
            >
              {tag}
            </span>
          ))}
        </div>
      )}
    </article>
  );
}
```

### 5.8 创建标签页面

```tsx
// app/blog/tags/[tag]/page.tsx

import Link from "next/link";
import { getAllPosts, getAllTags } from "@/lib/posts";

interface PageProps {
  params: Promise<{ tag: string }>;
}

export async function generateStaticParams() {
  const tags = getAllTags();
  
  return tags.map(tag => ({
    tag,
  }));
}

export default async function TagPage({ params }: PageProps) {
  const { tag } = await params;
  const allPosts = getAllPosts();
  
  // 过滤该标签的文章
  const posts = allPosts.filter(post => 
    post.tags?.includes(tag)
  );
  
  return (
    <div className="max-w-2xl mx-auto py-8">
      <h1 className="text-3xl font-bold mb-8">
        Posts tagged with "{tag}"
      </h1>
      
      <ul className="space-y-4">
        {posts.map(post => (
          <li key={post.slug}>
            <Link href={`/blog/${post.slug}`}>
              <h2>{post.title}</h2>
              <time>{post.date}</time>
            </Link>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### 5.9 构建和预览

```bash
# 安装依赖
pnpm add gray-matter

# 构建
pnpm build

# 输出：
# ✓ Creating an optimized production build
# ✓ Compiled successfully
# ✓ Linting and checking validity of types
# ✓ Collecting page data
# ✓ Generating static pages (10/10)
# ✓ Collecting build traces
# ✓ Exporting (10/10)
# ✓ Finalizing page optimization

# 预览
pnpm start
# 或直接打开 out/index.html
```

### 5.10 部署

```bash
# 部署到 Vercel（自动识别静态导出）
vercel --prod

# 部署到 GitHub Pages
# 1. 修改 next.config.ts
#    basePath: '/my-repo-name',
# 2. 构建
#    pnpm build
# 3. 将 out/ 目录推送到 gh-pages 分支

# 部署到 Netlify
# 1. 连接 Git 仓库
# 2. Build command: pnpm build
# 3. Publish directory: out
```

---

## 六、完整项目示例

### 6.1 目录结构

```
my-ssg-blog/
│
├── next.config.ts           ← output: 'export'
├── package.json
├── tsconfig.json
│
├── src/
│   ├── app/
│   │   ├── layout.tsx       ← 根布局
│   │   ├── page.tsx         ← 首页
│   │   │
│   │   ├── blog/
│   │   │   ├── page.tsx     ← 文章列表
│   │   │   │
│   │   │   └── [slug]/
│   │   │   │   └── page.tsx ← 文章详情
│   │   │   │       │
│   │   │   │       └── generateStaticParams()
│   │   │   │
│   │   │   └── tags/
│   │   │   │   └── [tag]/
│   │   │   │   │   └── page.tsx ← 标签页面
│   │   │   │   │       │
│   │   │   │   │       └── generateStaticParams()
│   │   │   │
│   │   └── about/
│   │   │   └── page.tsx     ← 关于页面
│   │   │
│   │   └── contact/
│   │   │   └── page.tsx     ← 联系页面
│   │   │
│   │   └── 404.tsx          ← 404 页面
│   │
│   ├── lib/
│   │   └── posts.ts         ← 文章解析工具
│   │
│   └── components/
│   │   ├── Header.tsx       ← 导航栏
│   │   ├── Footer.tsx       ← 页脚
│   │   └── PostCard.tsx     ← 文章卡片
│   │
│   └── styles/
│   │   └── globals.css      ← 全局样式
│
├── content/
│   └── posts/
│   │   ├── hello-world.md
│   │   ├── nextjs-ssg.md
│   │   └── typescript-tips.md
│   │
│   └── pages/
│   │   ├── about.md
│   │   └── contact.md
│
├── public/
│   ├── images/
│   │   ├── avatar.png
│   │   └── cover.jpg
│   │
│   └── favicon.ico
│
└── out/                     ← 构建输出（静态文件）
    ├── index.html
    ├── blog/
    │   ├── index.html
    │   ├── hello-world/
    │   │   └── index.html
    │   ├── nextjs-ssg/
    │   │   └── index.html
    │   ├── tags/
    │   │   ├── intro/
    │   │   │   └── index.html
    │   │   └── nextjs/
    │   │       └── index.html
    ├── about/
    │   └── index.html
    └── 404.html
```

### 6.2 根布局示例

```tsx
// src/app/layout.tsx

import "@/styles/globals.css";
import { Header } from "@/components/Header";
import { Footer } from "@/components/Footer";

export const metadata = {
  title: {
    default: "My SSG Blog",
    template: "%s | My SSG Blog",
  },
  description: "A statically generated blog using Next.js",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <Header />
        <main className="min-h-screen">
          {children}
        </main>
        <Footer />
      </body>
    </html>
  );
}
```

### 6.3 首页示例

```tsx
// src/app/page.tsx

import Link from "next/link";
import { getAllPosts } from "@/lib/posts";

export default function HomePage() {
  const recentPosts = getAllPosts().slice(0, 5);
  
  return (
    <div className="max-w-2xl mx-auto py-12">
      <h1 className="text-4xl font-bold mb-4">
        Welcome to My Blog
      </h1>
      
      <p className="text-gray-600 mb-8">
        A statically generated blog built with Next.js
      </p>
      
      <section>
        <h2 className="text-2xl font-semibold mb-4">Recent Posts</h2>
        
        <ul className="space-y-4">
          {recentPosts.map(post => (
            <li key={post.slug}>
              <Link 
                href={`/blog/${post.slug}`}
                className="block p-4 border rounded hover:bg-gray-50"
              >
                <h3 className="font-semibold">{post.title}</h3>
                <time className="text-sm text-gray-500">
                  {post.date}
                </time>
              </Link>
            </li>
          ))}
        </ul>
        
        <Link 
          href="/blog"
          className="inline-block mt-6 text-blue-600 hover:underline"
        >
          View all posts →
        </Link>
      </section>
    </div>
  );
}
```

---

## 七、SSG 的限制

### 7.1 不支持的功能

使用 `output: 'export'` 时，以下功能不可用：

| 功能 | 原因 | 替代方案 |
|------|------|---------|
| **API Routes** | 需要服务器运行 | 使用外部 API 或第三方服务 |
| **Server Actions** | 需要服务器处理 | 使用 Client Component + 外部 API |
| **动态路由（无 generateStaticParams）** | 无法预渲染未知路径 | 必须定义所有路径 |
| **ISR（revalidate）** | 需要服务器重新生成 | 手动重新构建部署 |
| **Headers/Cookies 动态读取** | 需要请求上下文 | 使用客户端 localStorage |
| **Next.js 图片优化** | 需要服务器处理 | 使用 `unoptimized: true` 或外部 CDN |
| **Middleware** | 需要服务器处理 | 在 CDN 或客户端处理 |
| **Rewrites/Redirects（动态）** | 需要服务器处理 | 在 CDN 配置 |

### 7.2 动态路由限制

```tsx
// ❌ 错误：未定义 generateStaticParams
// app/blog/[slug]/page.tsx

export default function BlogPost({ params }) {
  return <div>{params.slug}</div>;
}

// 用户访问 /blog/new-post
// → 构建时没有预渲染这个路径
// → 404 错误！
```

```tsx
// ✅ 正确：定义所有路径
export async function generateStaticParams() {
  const posts = await getAllPosts();  // 返回所有已知文章
  
  // 还可以添加 fallback
  return posts.map(post => ({ slug: post.slug }));
}

// 或者设置 dynamicParams = true 允许按需生成（需要服务器）
// export const dynamicParams = true;
```

### 7.3 图片处理限制

```typescript
// next.config.ts
const nextConfig: NextConfig = {
  output: "export",
  images: {
    unoptimized: true,  // 必须配置
  },
};

// 或者使用外部图片服务
// <Image src="https://cdn.example.com/image.jpg" />
```

---

## 八、选择指南

### 8.1 渲染方式选择决策树

```
需要什么？
    │
    ├─ 实时用户数据？
    │   ├─ 是 → SSR
    │   │   例如：仪表盘、购物车、个性化推荐
    │   │
    │   └─ 否 → 继续判断
    │
    ├─ 内容需要频繁更新？
    │   ├─ 是（几分钟/几小时）→ ISR
    │   │   例如：新闻网站、产品页面
    │   │
    │   └─ 否（几天/几周）→ SSG
    │       例如：博客、文档、营销页面
    │
    ├─ 是否需要服务器？
    │   ├─ 必须无服务器 → 纯静态导出（output: 'export'）
    │   │   例如：GitHub Pages、纯静态托管
    │   │
    │   └─ 可以有服务器 → ISR 或 SSR
    │
    └─ 混合需求？
        ├─ 静态内容 + 动态区域 → 混合使用
        │   例如：博客首页（静态） + 评论（客户端）
        │
        └─ 部分页面 SSR + 部分 SSG → 路径级别配置
```

### 8.2 混合使用示例

```tsx
// 静态博客页面
// app/blog/[slug]/page.tsx

export const revalidate = 3600;  // ISR

export default async function BlogPost({ params }) {
  const post = await getPost(params.slug);
  
  return (
    <article>
      {/* 静态内容 */}
      <h1>{post.title}</h1>
      <div>{post.content}</div>
      
      {/* 动态区域 */}
      <Comments postId={params.slug} />  {/* 流式渲染 */}
      <LiveViews />  {/* 客户端组件 */}
    </article>
  );
}

// 流式渲染动态区域
import { Suspense } from "react";

export default async function BlogPost({ params }) {
  return (
    <article>
      <h1>{post.title}</h1>
      <Suspense fallback={<CommentsSkeleton />}>
        <Comments postId={params.slug} />
      </Suspense>
    </article>
  );
}
```

---

## 九、常见问题

### 9.1 generateStaticParams 什么时候执行？

**答：构建时（pnpm build）执行一次。**

```
构建流程：

pnpm build
    │
    ▼
分析路由结构
    │
    │ 发现 [slug]/page.tsx
    │
    ▼
执行 generateStaticParams()
    │
    │ 返回 [{ slug: 'a' }, { slug: 'b' }]
    │
    ▼
预渲染每个路径
    │
    │ /blog/a → 生成 index.html
    │ /blog/b → 生成 index.html
    │
    ▼
输出静态文件
```

### 9.2 用户访问未定义的路径会怎样？

**答：404 错误（纯静态导出）。**

```tsx
// generateStaticParams 返回 [{ slug: 'a' }, { slug: 'b' }]

// 用户访问 /blog/c
// → 构建时没有生成
// → 404 Not Found
```

**解决方案**：

```tsx
// 方案 1：添加 fallback 页面
// app/blog/[slug]/not-found.tsx

export default function NotFound() {
  return <div>Post not found</div>;
}

// 方案 2：使用 ISR（需要服务器）
export const dynamicParams = true;  // 允许按需生成
export const revalidate = 3600;

// 方案 3：客户端获取未渲染的页面
"use client";
export default function BlogPost() {
  const { slug } = useParams();
  const [post, setPost] = useState(null);
  
  useEffect(() => {
    fetch(`/api/posts/${slug}`).then(...);  // 客户端获取
  }, [slug]);
}
```

### 9.3 SSG 能用 Client Component 吗？

**答：可以，但需要注意。**

```tsx
// app/blog/[slug]/page.tsx

import { useState } from "react";

export default function BlogPost({ params }) {
  // ❌ 错误：Server Component 不能用 hooks
  const [likes, setLikes] = useState(0);  // 报错！
}

// ✅ 正确：拆分为 Client Component
// app/blog/[slug]/page.tsx (Server Component)
import { LikeButton } from "@/components/LikeButton";

export default async function BlogPost({ params }) {
  const post = await getPost(params.slug);  // Server Component
  
  return (
    <article>
      <div>{post.content}</div>
      <LikeButton postId={params.slug} />  {/* Client Component */}
    </article>
  );
}

// components/LikeButton.tsx (Client Component)
"use client";

import { useState } from "react";

export function LikeButton({ postId }) {
  const [likes, setLikes] = useState(0);
  
  return (
    <button onClick={() => setLikes(likes + 1)}>
      Like ({likes})
    </button>
  );
}
```

### 9.4 如何获取构建时的环境变量？

```tsx
// 构建时可用的环境变量
process.env.NODE_ENV            // 'production'
process.env.NEXT_PUBLIC_XXX     // 公开的环境变量

// 构建时不可用的环境变量（需要服务器）
process.env.DATABASE_URL        // ❌ 构建时无法访问
process.env.API_SECRET          // ❌ 构建时无法访问

// 解决方案：硬编码或使用公开变量
const API_URL = process.env.NEXT_PUBLIC_API_URL || "https://api.example.com";
```

### 9.5 如何更新静态内容？

**答：重新构建并部署。**

```bash
# 修改内容
vim content/posts/new-article.md

# 重新构建
pnpm build

# 部署新的静态文件
vercel --prod
# 或
git push  # 自动触发 CI/CD
```

---

## 十、总结

### 10.1 渲染方式对比表

| 特性 | SSR | SSG | ISR | 静态导出 |
|------|-----|-----|-----|---------|
| **渲染时机** | 请求时 | 构建时 | 构建+定时 | 构建时 |
| **数据新鲜度** | 最新 | 固定 | 定时更新 | 固定 |
| **服务器要求** | Node.js | 无 | Node.js（轻量） | 无 |
| **部署位置** | 服务器 | 服务器/CDN | 服务器 | CDN/静态托管 |
| **响应速度** | 中 | 极快 | 快 | 极快 |
| **适用场景** | 动态内容 | 静态内容 | 偶尔更新 | 纯静态 |

### 10.2 一句话总结

> **SSR** = 每次请求实时渲染，数据最新，需要服务器
> 
> **SSG** = 构建时预渲染，极快响应，可部署到 CDN
> 
> **ISR** = SSG + 定时更新，兼顾速度和新鲜度
> 
> **静态导出** = 纯静态文件，无需服务器，适合博客/文档

---

## 十一、参考资料

- [Next.js Rendering Documentation](https://nextjs.org/docs/app/building-your-application/rendering)
- [Static Exports](https://nextjs.org/docs/app/building-your-application/deploying/static-exports)
- [generateStaticParams](https://nextjs.org/docs/app/api-reference/functions/generate-static-params)
- [Incremental Static Regeneration](https://nextjs.org/docs/app/building-your-application/data-fetching/incremental-static-regeneration)