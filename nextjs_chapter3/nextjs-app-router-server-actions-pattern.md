# Next.js App Router + Server Actions 模式完整指南

> 本文档详细介绍 Next.js App Router 配合 Server Actions 的最佳实践模式
> 最后更新: 2026年4月19日

## 目录

1. [核心理念](#核心理念)
2. [与传统 API 模式对比](#与传统-api-模式对比)
3. [完整示例：用户管理系统](#完整示例用户管理系统)
4. [核心优势](#核心优势)
5. [实际项目结构](#实际项目结构)
6. [常用工具函数](#常用工具函数)
7. [最佳实践](#最佳实践)
8. [适用场景](#适用场景)

---

## 核心理念

**App Router + Server Actions 模式**的核心是：在服务端组件中直接定义和调用数据操作函数，**无需创建 API Route**。

这意味着：
- Server Components 直接获取数据（无需 fetch）
- Client Components 直接调用 Server Action（无需 API）
- 数据操作代码集中在一个 `actions.ts` 文件中
- 类型安全自动推导，无需手动定义接口

---

## 与传统 API 模式对比

### 传统 API 模式

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

### App Router + Server Actions 模式

```
┌─────────────────────────────────────────────────────────────────┐
│                     Server Actions 方式                          │
├─────────────────────────────────────────────────────────────────┤
│  Server Component                                               │
│       │                                                         │
│       │ const users = await getUsers()  ← 直接调用函数           │
│       ▼                                                         │
│  Server Action (actions.ts)                                     │
│       │                                                         │
│       │ 服务端执行，可直接访问数据库                              │
│       ▼                                                         │
│  返回结果（自动类型推导）                                         │
└─────────────────────────────────────────────────────────────────┘
```

只需：
1. 定义一个函数（带 `'use server'` 指令）
2. 直接调用即可

---

## 完整示例：用户管理系统

### 1. 定义 Server Actions

```tsx
// app/users/actions.ts
'use server';

import { db } from '@/lib/db';
import { revalidatePath, redirect } from 'next/cache';
import { z } from 'zod';

// 验证 schema
const userSchema = z.object({
  name: z.string().min(2, '姓名至少2个字符'),
  email: z.string().email('邮箱格式不正确'),
});

// 创建用户
export async function createUser(formData: FormData) {
  // 验证输入
  const result = userSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
  });

  if (!result.success) {
    return { 
      error: '验证失败', 
      errors: result.error.flatten().fieldErrors 
    };
  }

  // 直接访问数据库
  const user = await db.users.create({ 
    data: result.data 
  });
  
  // 刷新页面缓存（自动更新 UI）
  revalidatePath('/users');
  
  // 可选：跳转到用户详情页
  // redirect(`/users/${user.id}`);
  
  return { success: true, user };
}

// 获取用户列表（Server Component 直接调用）
export async function getUsers() {
  return db.users.findMany({ 
    orderBy: { createdAt: 'desc' } 
  });
}

// 获取单个用户
export async function getUser(id: string) {
  const user = await db.users.findUnique({ where: { id } });
  
  if (!user) {
    return null;
  }
  
  return user;
}

// 更新用户
export async function updateUser(id: string, formData: FormData) {
  const result = userSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
  });

  if (!result.success) {
    return { error: '验证失败' };
  }

  const user = await db.users.update({
    where: { id },
    data: result.data,
  });
  
  revalidatePath('/users');
  revalidatePath(`/users/${id}`);
  
  return { success: true, user };
}

// 删除用户
export async function deleteUser(id: string) {
  await db.users.delete({ where: { id } });
  
  revalidatePath('/users');
  
  return { success: true };
}
```

### 2. Server Component 页面（直接获取数据）

```tsx
// app/users/page.tsx
// 注意：这是 Server Component（默认），无需 'use server'

import { getUsers, createUser } from './actions';
import UserList from './user-list';

// 服务端直接获取数据，无需 API 调用
export default async function UsersPage() {
  const users = await getUsers();  // 直接在服务端执行
  
  return (
    <div className="p-8">
      <h1 className="text-2xl font-bold mb-6">用户管理</h1>
      
      {/* 表单直接绑定 Server Action */}
      <form action={createUser} className="mb-8 space-y-4">
        <div>
          <label htmlFor="name">姓名</label>
          <input 
            id="name"
            name="name" 
            placeholder="请输入姓名" 
            className="border p-2 rounded w-full"
            required 
          />
        </div>
        <div>
          <label htmlFor="email">邮箱</label>
          <input 
            id="email"
            name="email" 
            type="email" 
            placeholder="请输入邮箱"
            className="border p-2 rounded w-full"
            required 
          />
        </div>
        <button 
          type="submit"
          className="bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600"
        >
          创建用户
        </button>
      </form>
      
      {/* 传递数据给 Client Component */}
      <UserList users={users} />
    </div>
  );
}
```

### 3. Client Component（交互部分）

```tsx
// app/users/user-list.tsx
'use client';

import { deleteUser } from './actions';
import { useState } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
}

export default function UserList({ users }: { users: User[] }) {
  const [deleting, setDeleting] = useState<string | null>(null);
  
  async function handleDelete(id: string) {
    setDeleting(id);
    const result = await deleteUser(id);  // 直接调用 Server Action
    
    if (result.success) {
      // revalidatePath 已在 Server Action 中调用，页面会自动刷新
    }
    
    setDeleting(null);
  }
  
  return (
    <div>
      <h2 className="text-xl font-semibold mb-4">用户列表</h2>
      
      {users.length === 0 ? (
        <p className="text-gray-500">暂无用户</p>
      ) : (
        <ul className="space-y-2">
          {users.map(user => (
            <li 
              key={user.id} 
              className="flex items-center justify-between p-4 border rounded hover:bg-gray-50"
            >
              <div>
                <span className="font-medium">{user.name}</span>
                <span className="text-gray-500 ml-4">{user.email}</span>
              </div>
              <button 
                onClick={() => handleDelete(user.id)}
                disabled={deleting === user.id}
                className="text-red-500 hover:text-red-700 disabled:opacity-50 disabled:cursor-not-allowed"
              >
                {deleting === user.id ? '删除中...' : '删除'}
              </button>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

### 4. 用户详情页（动态路由）

```tsx
// app/users/[id]/page.tsx
import { getUser } from '../actions';
import { notFound } from 'next/navigation';

export default async function UserDetailPage({
  params,
}: {
  params: { id: string };
}) {
  const user = await getUser(params.id);
  
  if (!user) {
    notFound();  // 返回 404 页面
  }
  
  return (
    <div className="p-8">
      <h1 className="text-2xl font-bold">{user.name}</h1>
      <p className="text-gray-500">{user.email}</p>
    </div>
  );
}
```

### 5. 使用 useFormState 处理表单状态

```tsx
// app/users/create-form.tsx
'use client';

import { useFormState, useFormStatus } from 'react-dom';
import { createUser } from './actions';

// 初始状态
const initialState = {
  error: null,
  errors: null,
  success: false,
};

// 提交按钮组件（必须在 form 内部）
function SubmitButton() {
  const { pending } = useFormStatus();
  
  return (
    <button 
      type="submit" 
      disabled={pending}
      className="bg-blue-500 text-white px-4 py-2 rounded disabled:opacity-50"
    >
      {pending ? '创建中...' : '创建用户'}
    </button>
  );
}

// 表单组件
export default function CreateForm() {
  const [state, formAction] = useFormState(createUser, initialState);
  
  return (
    <form action={formAction} className="space-y-4">
      <div>
        <input 
          name="name" 
          placeholder="姓名"
          className="border p-2 rounded w-full"
        />
        {state.errors?.name && (
          <p className="text-red-500 text-sm mt-1">
            {state.errors.name[0]}
          </p>
        )}
      </div>
      
      <div>
        <input 
          name="email" 
          type="email"
          placeholder="邮箱"
          className="border p-2 rounded w-full"
        />
        {state.errors?.email && (
          <p className="text-red-500 text-sm mt-1">
            {state.errors.email[0]}
          </p>
        )}
      </div>
      
      <SubmitButton />
      
      {state.error && (
        <p className="text-red-500">{state.error}</p>
      )}
      
      {state.success && (
        <p className="text-green-500">创建成功！</p>
      )}
    </form>
  );
}
```

---

## 核心优势

| 特性 | API Route | Server Actions |
|------|-----------|----------------|
| 文件数量 | 需要 `app/api/*/route.ts` | 一个 `actions.ts` |
| 调用方式 | `fetch('/api/users')` | `await createUser(formData)` |
| 类型安全 | 需手动定义接口 | **自动类型推导** |
| CSRF 保护 | 需手动处理 | **自动内置** |
| 无 JavaScript | 不工作 | **渐进增强，仍可用** |
| 请求体处理 | 需解析 JSON | FormData 直接支持 |
| 代码复杂度 | 高（请求/响应处理） | 低（直接函数调用） |

### 代码对比

**API Route 方式：**

```tsx
// app/api/users/route.ts
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

**Server Actions 方式：**

```tsx
// app/users/actions.ts
'use server';

export async function createUser(formData: FormData) {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;
  
  const user = await db.users.create({ data: { name, email } });
  
  return user;
}

// 客户端调用（直接函数调用）
import { createUser } from './actions';

const user = await createUser(formData);  // 简洁！
```

---

## 实际项目结构

```
app/
├── (auth)/                     ← 路由组（不影响 URL）
│   ├── login/
│   │   ├── page.tsx            ← Server Component
│   │   ├── actions.ts          ← 登录相关 Server Actions
│   │   └── login-form.tsx      ← Client Component
│   ├── register/
│   │   ├── page.tsx
│   │   ├── actions.ts
│   │   └── register-form.tsx
│   └── layout.tsx
│
├── (dashboard)/                ← 路由组
│   ├── users/
│   │   ├── page.tsx            ← Server Component，直接获取数据
│   │   ├── [id]/page.tsx       ← 动态路由，用户详情
│   │   ├── actions.ts          ← 用户 CRUD 操作
│   │   ├── user-list.tsx       ← Client Component
│   │   └── user-form.tsx       ← Client Component
│   │
│   ├── posts/
│   │   ├── page.tsx
│   │   ├── [slug]/page.tsx
│   │   ├── actions.ts          ← 文章 CRUD 操作
│   │   ├── post-list.tsx
│   │   └── post-form.tsx
│   │
│   └── layout.tsx              ← Dashboard 布局
│
├── actions/                    ← 全局 Server Actions
│   ├── auth.ts                 ← 认证相关（登录、登出）
│   ├── theme.ts                ← 主题切换
│   └── settings.ts             ← 全局设置
│
├── api/                        ← 仅用于外部 API（可选）
│   └── webhooks/
│   │   └── stripe/route.ts     ← Stripe webhook
│   └── oauth/
│   │   └── callback/route.ts   ← OAuth 回调
│
├── lib/
│   ├── db.ts                   ← 数据库连接
│   ├── auth.ts                 ← 认证逻辑
│   └── utils.ts                ← 工具函数
│
└── layout.tsx                  ← 根布局
```

### 目录组织原则

1. **每个功能模块有自己的 `actions.ts`**
   - 用户模块 → `app/(dashboard)/users/actions.ts`
   - 文章模块 → `app/(dashboard)/posts/actions.ts`

2. **全局操作放在 `app/actions/`**
   - 认证、主题、设置等

3. **API Route 只用于外部接口**
   - Webhooks、OAuth 回调、第三方 API

---

## 常用工具函数

### revalidatePath - 刷新页面缓存

```tsx
import { revalidatePath } from 'next/cache';

export async function createPost(formData: FormData) {
  await db.posts.create({ ... });
  
  revalidatePath('/posts');         // 刷新 /posts 页面
  revalidatePath('/posts/[slug]');  // 刷新动态路由
  revalidatePath('/', 'layout');    // 刷新根布局
}
```

### revalidateTag - 按标签刷新

```tsx
import { revalidateTag } from 'next/cache';

// fetch 时设置标签
async function getPosts() {
  const res = await fetch('https://api.example.com/posts', {
    next: { tags: ['posts'] }
  });
  return res.json();
}

// Server Action 中刷新
export async function createPost(formData: FormData) {
  await db.posts.create({ ... });
  revalidateTag('posts');  // 刷新所有带 'posts' 标签的请求
}
```

### redirect - 重定向

```tsx
import { redirect } from 'next/navigation';

export async function createPost(formData: FormData) {
  const post = await db.posts.create({ ... });
  redirect(`/posts/${post.slug}`);  // 跳转到新文章
}
```

### notFound - 返回 404

```tsx
import { notFound } from 'next/navigation';

export async function getUser(id: string) {
  const user = await db.users.findUnique({ where: { id } });
  
  if (!user) {
    notFound();  // 返回 not-found.tsx 页面
  }
  
  return user;
}
```

---

## 最佳实践

### 1. 输入验证（使用 Zod）

```tsx
import { z } from 'zod';

const userSchema = z.object({
  name: z.string().min(2, '姓名至少2个字符'),
  email: z.string().email('邮箱格式不正确'),
  age: z.number().int().positive().optional(),
});

export async function createUser(formData: FormData) {
  const result = userSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
    age: formData.get('age') ? parseInt(formData.get('age')) : undefined,
  });

  if (!result.success) {
    return {
      error: '验证失败',
      errors: result.error.flatten().fieldErrors,
    };
  }

  const user = await db.users.create({ data: result.data });
  return { success: true, user };
}
```

### 2. 统一返回类型

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

export async function createUser(
  formData: FormData
): Promise<ActionResult<User>> {
  // ...
}
```

### 3. 错误处理

```tsx
export async function deleteUser(id: string): Promise<ActionResult> {
  try {
    await db.users.delete({ where: { id } });
    revalidatePath('/users');
    return { success: true };
  } catch (error) {
    if (error instanceof Error) {
      return { error: error.message };
    }
    return { error: '删除失败，请稍后重试' };
  }
}
```

### 4. 乐观更新

```tsx
'use client';

import { useOptimistic } from 'react';
import { updateUser } from './actions';

function UserCard({ user }: { user: User }) {
  const [optimisticUser, updateOptimisticUser] = useOptimistic(
    user,
    (state, newData) => ({ ...state, ...newData })
  );

  async function handleUpdate(formData: FormData) {
    // 乐观更新（立即显示）
    updateOptimisticUser({ name: formData.get('name') });
    
    // 实际更新
    const result = await updateUser(user.id, formData);
    
    if (!result.success) {
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

### 5. 加载状态

```tsx
'use client';

import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus();
  
  return (
    <button type="submit" disabled={pending}>
      {pending ? '提交中...' : '提交'}
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

---

## 适用场景

### ✅ 适用场景

| 场景 | 说明 |
|------|------|
| 表单提交 | 创建、更新、删除数据 |
| 数据 CRUD | 内部数据操作 |
| 用户认证 | 登录、登出、注册 |
| 需要渐进增强 | 无 JavaScript 也能工作 |
| 内部操作 | 不需要外部调用的场景 |

### ❌ 不适用场景

| 场景 | 说明 | 替代方案 |
|------|------|----------|
| 外部 REST API | 需要第三方调用 | API Route |
| Webhooks | 第三方服务回调 | API Route |
| OAuth 回调 | 认证服务回调 | API Route |
| 公开 API | 移动端或其他客户端 | API Route |
| 复杂请求处理 | 自定义头部、缓存控制 | API Route |

### 混合使用

实际项目中，**Server Actions 和 API Route 可以共存**：

```
app/
├── (dashboard)/
│   ├── users/
│   │   ├── actions.ts          ← Server Actions（内部操作）
│   │   └── page.tsx
│
├── api/
│   ├── webhooks/
│   │   └ stripe/route.ts       ← API Route（外部回调）
│   └── users/
│   │   └ route.ts              ← API Route（公开 API）
```

---

## 总结

### App Router + Server Actions 模式

| 特点 | 说明 |
|------|------|
| **无需 API Route** | 数据操作直接定义函数 |
| **直接调用** | Server Component 直接获取数据 |
| **类型安全** | TypeScript 自动推导返回类型 |
| **自动 CSRF** | 内置安全保护 |
| **渐进增强** | 禁用 JavaScript 也能工作 |
| **简洁** | 减少代码复杂度和文件数量 |

### 核心流程

```
1. Server Component → await getUsers() → 直接获取数据
2. Client Component → await createUser(formData) → 直接调用 Server Action
3. Server Action → 数据库操作 → revalidatePath → 自动更新 UI
```

### 最佳实践

1. 每个功能模块一个 `actions.ts`
2. 使用 Zod 验证输入
3. 统一返回类型 `ActionResult`
4. Server Component 直接获取数据
5. Client Component 用于交互
6. API Route 仅用于外部接口

---

## 参考资源

- [Next.js Server Actions 官方文档](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions)
- [React useFormStatus 文档](https://react.dev/reference/react-dom/hooks/useFormStatus)
- [React useFormState 文档](https://react.dev/reference/react-dom/hooks/useFormState)
- [Zod 验证库](https://zod.dev/)