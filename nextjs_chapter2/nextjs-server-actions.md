---
title: Next.js Server Actions
description: 详细介绍 Next.js Server Actions 的作用、用法和最佳实践
---

# Next.js Server Actions

本文详细介绍 Next.js 14+ 引入的 Server Actions 功能。

## 目录

1. [什么是 Server Actions](#1-什么是-server-actions)
2. [核心概念](#2-核心概念)
3. [基本用法](#3-基本用法)
4. [在组件中使用](#4-在组件中使用)
5. [常用工具函数](#5-常用工具函数)
6. [与 API Route 对比](#6-与-api-route-对比)
7. [渐进增强](#7-渐进增强)
8. [最佳实践](#8-最佳实践)
9. [常见问题](#9-常见问题)

---

## 1. 什么是 Server Actions

Server Actions 是 Next.js 14 引入的功能，允许在服务端直接定义和执行函数。

### 传统方式 vs Server Actions

**传统 API 方式：**

```
┌─────────────────────────────────────────────────────────────────┐
│                     传统 API 方式                                │
├─────────────────────────────────────────────────────────────────┤
│  Client Component                                               │
│       │                                                         │
│       │ fetch('/api/users', { method: 'POST', body: data })     │
│       ▼                                                         │
│  API Route (app/api/users/route.ts)                             │
│       │                                                         │
│       │ 数据库操作                                               │
│       ▼                                                         │
│  返回 JSON                                                      │
└─────────────────────────────────────────────────────────────────┘
```

需要：
1. 创建单独的 API 路由文件
2. 使用 fetch 发送请求
3. 手动处理请求/响应
4. 定义类型接口

**Server Actions 方式：**

```
┌─────────────────────────────────────────────────────────────────┐
│                     Server Actions 方式                          │
├─────────────────────────────────────────────────────────────────┤
│  Client Component                                               │
│       │                                                         │
│       │ await createUser(data)  ← 直接调用函数                   │
│       ▼                                                         │
│  Server Action (actions.ts)                                     │
│       │                                                         │
│       │ 服务端执行，可直接访问数据库                              │
│       ▼                                                         │
│  返回结果（自动类型推导）                                         │
└─────────────────────────────────────────────────────────────────┘
```

只需：
1. 定义一个函数
2. 直接调用即可

---

## 2. 核心概念

### 'use server' 指令

标记函数或文件在服务端执行：

```tsx
// 方式一：文件级别（整个文件都是 Server Actions）
'use server';

export async function createUser(formData: FormData) { ... }
export async function deleteUser(id: string) { ... }

// 方式二：函数级别（在 Server Component 内）
async function ServerComponent() {
  async function myAction() {
    'use server';  // 只标记这个函数
    // 服务端代码
  }

  return <form action={myAction}>...</form>;
}
```

### 执行流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    Server Actions 执行流程                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 用户在客户端触发操作（点击按钮、提交表单）                     │
│                         │                                       │
│                         ▼                                       │
│  2. Next.js 将请求发送到服务端                                   │
│                         │                                       │
│                         ▼                                       │
│  3. Server Action 函数在服务端执行                               │
│     - 可以访问数据库                                             │
│     - 可以访问文件系统                                           │
│     - 可以使用敏感信息（API Key）                                │
│                         │                                       │
│                         ▼                                       │
│  4. 返回结果到客户端                                             │
│     - 自动类型安全                                               │
│     - 自动 CSRF 保护                                             │
│                         │                                       │
│                         ▼                                       │
│  5. 客户端更新 UI                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 基本用法

### 创建 actions.ts

```tsx
// actions/users.ts
'use server';

import { db } from '@/lib/db';
import { revalidatePath, redirect } from 'next/cache';

// 创建用户
export async function createUser(formData: FormData) {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;

  // 验证数据
  if (!name || !email) {
    return { error: '请填写所有字段' };
  }

  // 直接访问数据库
  const user = await db.users.create({
    data: { name, email }
  });

  // 刷新用户列表页面缓存
  revalidatePath('/users');

  return { success: true, user };
}

// 删除用户
export async function deleteUser(id: string) {
  await db.users.delete({ where: { id } });

  // 刷新缓存
  revalidatePath('/users');

  return { success: true };
}

// 更新用户
export async function updateUser(id: string, formData: FormData) {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;

  const user = await db.users.update({
    where: { id },
    data: { name, email }
  });

  revalidateTag('users');

  return { success: true, user };
}

// 获取用户列表
export async function getUsers() {
  const users = await db.users.findMany();
  return users;
}
```

### 使用 FormData

```tsx
// actions/posts.ts
'use server';

import { db } from '@/lib/db';

export async function createPost(formData: FormData) {
  // 获取表单字段
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;
  const published = formData.get('published') === 'on';  // checkbox

  // 获取多选值
  const tags = formData.getAll('tags') as string[];

  // 获取文件
  const image = formData.get('image') as File;
  const imageBytes = await image.arrayBuffer();

  const post = await db.posts.create({
    data: {
      title,
      content,
      published,
      tags,
      image: Buffer.from(imageBytes),
    }
  });

  return post;
}
```

---

## 4. 在组件中使用

### Server Component 中使用

```tsx
// app/users/page.tsx (Server Component)
import { createUser, getUsers } from '@/actions/users';

// 使用 form action（无需 JavaScript）
export default async function UsersPage() {
  const users = await getUsers();  // 服务端获取数据

  return (
    <div>
      <h1>用户列表</h1>
      <ul>
        {users.map(user => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>

      <!-- 表单直接绑定 Server Action -->
      <form action={createUser}>
        <input name="name" type="text" required />
        <input name="email" type="email" required />
        <button type="submit">创建用户</button>
      </form>
    </div>
  );
}
```

### Client Component 中使用

```tsx
// app/users/create-page.tsx (Client Component)
'use client';

import { createUser } from '@/actions/users';
import { useFormState, useFormStatus } from 'react-dom';
import { useState } from 'react';

// 使用 useFormState 处理状态
function CreateForm() {
  const [state, formAction] = useFormState(createUser, null);

  return (
    <form action={formAction}>
      <input name="name" required />
      <input name="email" type="email" required />
      <SubmitButton />
      {state?.error && <p className="error">{state.error}</p>}
      {state?.success && <p className="success">创建成功！</p>}
    </form>
  );
}

// 使用 useFormStatus 获取提交状态
function SubmitButton() {
  const { pending } = useFormStatus();  // 获取表单提交状态

  return (
    <button type="submit" disabled={pending}>
      {pending ? '创建中...' : '创建用户'}
    </button>
  );
}

export default function CreateUserPage() {
  return (
    <div>
      <h1>创建用户</h1>
      <CreateForm />
    </div>
  );
}
```

### 手动调用 Server Action

```tsx
// app/users/delete-button.tsx
'use client';

import { deleteUser } from '@/actions/users';
import { useRouter } from 'next/navigation';

export function DeleteButton({ userId: string }) {
  const router = useRouter();
  const [loading, setLoading] = useState(false);

  async function handleDelete() {
    setLoading(true);
    try {
      const result = await deleteUser(userId);
      if (result.success) {
        router.refresh();  // 刷新当前页面
      }
    } catch (error) {
      console.error(error);
    }
    setLoading(false);
  }

  return (
    <button onClick={handleDelete} disabled={loading}>
      {loading ? '删除中...' : '删除'}
    </button>
  );
}
```

---

## 5. 常用工具函数

### revalidatePath

刷新指定路径的缓存：

```tsx
import { revalidatePath } from 'next/cache';

export async function createPost(formData: FormData) {
  await db.posts.create({ ... });

  // 刷新指定页面
  revalidatePath('/posts');       // 刷新 /posts 页面
  revalidatePath('/posts/[slug]'); // 刷新动态路由页面
  revalidatePath('/', 'layout');   // 刷新根布局
  revalidatePath('/', 'page');     // 刷新根页面
}
```

### revalidateTag

刷新特定标签的缓存：

```tsx
import { revalidateTag } from 'next/cache';

// fetch 时设置标签
async function getPosts() {
  const res = await fetch('https://api.example.com/posts', {
    next: { tags: ['posts'] }  // 设置缓存标签
  });
  return res.json();
}

// Server Action 中刷新
export async function createPost(formData: FormData) {
  await db.posts.create({ ... });

  revalidateTag('posts');  // 刷新所有带 'posts' 标签的请求
}
```

### redirect

操作后跳转页面：

```tsx
import { redirect } from 'next/navigation';

export async function createPost(formData: FormData) {
  const post = await db.posts.create({ ... });

  // 跳转到新文章页面
  redirect(`/posts/${post.slug}`);
}
```

### permanentRedirect

永久重定向：

```tsx
import { permanentRedirect } from 'next/navigation';

export async function updateUserSlug(id: string, newSlug: string) {
  await db.users.update({ where: { id }, data: { slug: newSlug } });

  // 永久重定向（返回 308 状态码）
  permanentRedirect(`/users/${newSlug}`);
}
```

### notFound

返回 404 页面：

```tsx
import { notFound } from 'next/navigation';

export async function getPost(slug: string) {
  const post = await db.posts.findUnique({ where: { slug } });

  if (!post) {
    notFound();  // 返回 not-found.tsx 页面
  }

  return post;
}
```

---

## 6. 与 API Route 对比

| 方面 | API Route | Server Actions |
|------|-----------|----------------|
| 文件位置 | `app/api/*/route.ts` | `actions.ts` 或组件内 |
| 定义方式 | HTTP 方法导出 | 函数定义 |
| 调用方式 | `fetch('/api/...')` | 直接调用函数 |
| 类型安全 | 需手动定义类型 | 自动类型推导 |
| 表单处理 | 需解析 JSON | FormData 直接支持 |
| CSRF 保护 | 需手动处理 | 自动内置 |
| 渐进增强 | 不支持 | 支持（无 JS 也能工作） |
| 适用场景 | REST API、第三方调用 | 表单提交、数据操作 |

### API Route 示例

```tsx
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { db } from '@/lib/db';

export async function POST(request: NextRequest) {
  const body = await request.json();
  const { name, email } = body;

  const user = await db.users.create({ data: { name, email } });

  return NextResponse.json(user);
}

// 客户端调用
const createUser = async (data: { name: string; email: string }) => {
  const res = await fetch('/api/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  return res.json();
};
```

### Server Action 示例

```tsx
// actions/users.ts
'use server';

import { db } from '@/lib/db';

export async function createUser(formData: FormData) {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;

  const user = await db.users.create({ data: { name, email } });

  return user;
}

// 客户端调用（直接函数调用）
import { createUser } from '@/actions/users';

const handleCreate = async (formData: FormData) => {
  const user = await createUser(formData);
  console.log(user);
};
```

---

## 7. 渐进增强

Server Actions 支持渐进增强，即使禁用 JavaScript 也能工作。

### 基础表单

```tsx
// actions/subscribe.ts
'use server';

import { db } from '@/lib/db';
import { redirect } from 'next/navigation';

export async function subscribe(formData: FormData) {
  const email = formData.get('email') as string;

  await db.subscriptions.create({ data: { email } });

  // 服务端跳转
  redirect('/thank-you');
}

// app/subscribe/page.tsx (Server Component)
import { subscribe } from '@/actions/subscribe';

export default function SubscribePage() {
  return (
    <form action={subscribe}>
      <input name="email" type="email" required />
      <button type="submit">订阅</button>
    </form>
  );
}
```

即使浏览器禁用 JavaScript：
1. 表单仍然可以提交
2. Server Action 在服务端执行
3. redirect 在服务端跳转

### 带验证的表单

```tsx
// actions/login.ts
'use server';

import { db } from '@/lib/db';
import { redirect } from 'next/navigation';
import { z } from 'zod';

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(6),
});

export async function login(formData: FormData): Promise<{ error?: string }> {
  const result = loginSchema.safeParse({
    email: formData.get('email'),
    password: formData.get('password'),
  });

  if (!result.success) {
    return { error: '邮箱或密码格式不正确' };
  }

  const user = await db.users.findUnique({
    where: { email: result.data.email }
  });

  if (!user || user.password !== result.data.password) {
    return { error: '邮箱或密码错误' };
  }

  redirect('/dashboard');
}

// app/login/page.tsx
import { login } from '@/actions/login';
import { useFormState } from 'react-dom';

function LoginForm() {
  const [state, formAction] = useFormState(login, null);

  return (
    <form action={formAction}>
      <input name="email" type="email" />
      <input name="password" type="password" />
      {state?.error && <p className="error">{state.error}</p>}
      <button type="submit">登录</button>
    </form>
  );
}

export default function LoginPage() {
  return (
    <div>
      <h1>登录</h1>
      <LoginForm />
    </div>
  );
}
```

---

## 8. 最佳实践

### 8.1 文件组织

```
app/
├── (dashboard)/
│   ├── users/
│   │   ├── page.tsx
│   │   └── actions.ts       ← 用户相关操作
│   ├── posts/
│   │   ├── page.tsx
│   │   └── actions.ts       ← 文章相关操作
│   └── layout.tsx
├── actions/
│   ├── auth.ts              ← 全局认证操作
│   ├── settings.ts          ← 全局设置操作
│   └── theme.ts             ← 全局主题操作
└── layout.tsx
```

### 8.2 输入验证

```tsx
// 使用 Zod 验证
import { z } from 'zod';

const createUserSchema = z.object({
  name: z.string().min(2).max(50),
  email: z.string().email(),
  age: z.number().int().positive().optional(),
});

export async function createUser(formData: FormData) {
  const result = createUserSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
    age: formData.get('age') ? parseInt(formData.get('age')) : undefined,
  });

  if (!result.success) {
    return {
      error: result.error.flatten().fieldErrors,
    };
  }

  const user = await db.users.create({ data: result.data });
  revalidatePath('/users');

  return { success: true, user };
}
```

### 8.3 错误处理

```tsx
export async function deleteUser(id: string) {
  try {
    await db.users.delete({ where: { id } });
    revalidatePath('/users');
    return { success: true };
  } catch (error) {
    if (error instanceof Error) {
      return { error: error.message };
    }
    return { error: '删除失败' };
  }
}

// 客户端处理
async function handleDelete(id: string) {
  const result = await deleteUser(id);

  if (result.error) {
    toast.error(result.error);
  } else {
    toast.success('删除成功');
  }
}
```

### 8.4 乐观更新

```tsx
'use client';

import { updateUser } from '@/actions/users';
import { useOptimistic } from 'react';

function UserCard({ user }) {
  const [optimisticUser, setOptimisticUser] = useOptimistic(
    user,
    (state, newData) => ({ ...state, ...newData })
  );

  async function handleUpdate(formData: FormData) {
    // 乐观更新
    setOptimisticUser({ name: formData.get('name') });

    // 实际更新
    const result = await updateUser(user.id, formData);

    if (!result.success) {
      // 失败时回滚（useOptimistic 自动处理）
      toast.error(result.error);
    }
  }

  return (
    <form action={handleUpdate}>
      <input name="name" defaultValue={optimisticUser.name} />
      <button>更新</button>
    </form>
  );
}
```

### 8.5 返回类型定义

```tsx
// actions/types.ts
export type ActionResult<T = void> = {
  success: boolean;
  data?: T;
  error?: string;
  errors?: Record<string, string[]>;
};

// actions/users.ts
import type { ActionResult } from './types';

export async function createUser(formData: FormData): Promise<ActionResult<User>> {
  // ...
}
```

---

## 9. 常见问题

### Q1: Server Actions 能被外部调用吗？

不能直接调用。Server Actions 设计用于内部调用，如果要提供 REST API，使用 API Route。

```tsx
// 内部调用 ✅
import { createUser } from '@/actions/users';
await createUser(formData);

// 外部调用 ❌（不支持）
fetch('https://your-site.com/actions/users', { ... });  // 无法工作

// 外部 API ✅
fetch('https://your-site.com/api/users', { method: 'POST', ... });
```

### Q2: 可以在 Client Component 中定义 Server Action 吗？

不行。Client Component 中的代码会发送到客户端，不能包含服务端逻辑。

```tsx
// ❌ 错误
'use client';

export async function myAction() {  // 会报错
  'use server';
  await db.users.create({ ... });
}

// ✅ 正确：从单独文件导入
'use client';

import { myAction } from '@/actions/users';  // 从 Server Action 文件导入
```

### Q3: 如何处理文件上传？

```tsx
'use server';

import { writeFile } from 'fs/promises';
import path from 'path';

export async function uploadFile(formData: FormData) {
  const file = formData.get('file') as File;

  if (!file) {
    return { error: '请选择文件' };
  }

  const bytes = await file.arrayBuffer();
  const buffer = Buffer.from(bytes);

  const filename = `${Date.now()}-${file.name}`;
  const filepath = path.join(process.cwd(), 'uploads', filename);

  await writeFile(filepath, buffer);

  return { success: true, filename };
}
```

### Q4: 如何添加加载状态？

```tsx
'use client';

import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button type="submit" disabled={pending}>
      {pending ? <Spinner /> : '提交'}
    </button>
  );
}

// 注意：SubmitButton 必须在 form 内部
function MyForm() {
  return (
    <form action={myAction}>
      <input name="title" />
      <SubmitButton />  {/* 必须在 form 内 */}
    </form>
  );
}
```

### Q5: 如何测试 Server Actions？

```tsx
// __tests__/actions/users.test.ts
import { createUser } from '@/actions/users';
import { db } from '@/lib/db';

// Mock 数据库
jest.mock('@/lib/db', () => ({
  users: {
    create: jest.fn(),
  },
}));

describe('createUser', () => {
  it('should create user successfully', async () => {
    const formData = new FormData();
    formData.set('name', 'Test User');
    formData.set('email', 'test@example.com');

    db.users.create.mockResolvedValue({ id: '1', name: 'Test User' });

    const result = await createUser(formData);

    expect(result.success).toBe(true);
    expect(result.user.name).toBe('Test User');
  });
});
```

---

## 总结

Server Actions 是 Next.js 14+ 的强大功能：

| 优势 | 说明 |
|------|------|
| 简洁 | 无需创建 API Route，直接定义函数 |
| 类型安全 | TypeScript 自动推导返回类型 |
| 安全性 | 自动 CSRF 保护，敏感代码留在服务端 |
| 渐进增强 | 无 JavaScript 也能工作 |
| 性能 | 减少客户端代码，优化包大小 |

适用场景：
- 表单提交
- 数据 CRUD 操作
- 需要渐进增强的场景
- 内部操作

不适用场景：
- 外部 REST API
- 需要复杂请求处理
- 第三方服务集成

---

## 参考资源

- [Next.js Server Actions 文档](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions)
- [React useFormStatus 文档](https://react.dev/reference/react-dom/hooks/useFormStatus)
- [React useFormState 文档](https://react.dev/reference/react-dom/hooks/useFormState)