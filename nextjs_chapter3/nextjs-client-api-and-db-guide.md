# Next.js 客户端调用与数据库连接详解

> 本文档详细解释 Next.js 中客户端 API 调用的最佳位置，以及 `db` 数据库客户端的实现方式
> 最后更新: 2026年4月19日

## 目录

1. [客户端 API 调用位置](#客户端-api-调用位置)
2. [数据库客户端 db 详解](#数据库客户端-db-详解)
3. [完整示例项目](#完整示例项目)
4. [对比总结](#对比总结)

---

## 客户端 API 调用位置

### 传统 API 模式的架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     传统 API 模式                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Client Component (客户端)                                      │
│       │                                                         │
│       │ fetch('/api/users')                                     │
│       ▼                                                         │
│  API Route (服务端)                                             │
│       │                                                         │
│       │ db.users.create()                                       │
│       ▼                                                         │
│  数据库                                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 文件组织方式

```
app/
├── api/
│   └── users/
│       └── route.ts              ← API Route（服务端）
│
├── users/
│   ├── page.tsx                  ← Server Component
│   ├── user-form.tsx             ← Client Component（调用 API）
│   └── api.ts                    ← API 封装层（可选）
│
└── lib/
    ├── db.ts                     ← 数据库客户端
    └── api/
        └── users.ts              ← 全局 API 层（可选）
```

---

### 方式一：直接在 Client Component 中调用

```tsx
// app/users/user-form.tsx
'use client';

import { useState } from 'react';

export function UserForm() {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  
  // 直接在组件中写 fetch
  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    setLoading(true);
    setError(null);
    
    const formData = new FormData(e.currentTarget);
    const data = {
      name: formData.get('name') as string,
      email: formData.get('email') as string,
    };
    
    try {
      const res = await fetch('/api/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      });
      
      if (!res.ok) {
        throw new Error('创建失败');
      }
      
      const user = await res.json();
      console.log('创建成功:', user);
      
      // 刷新页面或更新状态
      window.location.reload();
      
    } catch (err) {
      setError(err instanceof Error ? err.message : '未知错误');
    } finally {
      setLoading(false);
    }
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <input name="name" placeholder="姓名" required />
      <input name="email" type="email" placeholder="邮箱" required />
      
      {error && <p className="error">{error}</p>}
      
      <button type="submit" disabled={loading}>
        {loading ? '创建中...' : '创建用户'}
      </button>
    </form>
  );
}
```

**优点**：
- 简单直接，适合小型项目
- 代码集中在一处

**缺点**：
- 每个组件都要重复写 fetch 逻辑
- 错误处理分散
- 类型定义需要手动维护

---

### 方式二：封装成 API 层（推荐）

将 API 调用封装到单独的文件中，便于复用和维护。

```tsx
// lib/api/users.ts 或 app/users/api.ts

// 类型定义
export interface User {
  id: string;
  name: string;
  email: string;
  createdAt: string;
}

export interface CreateUserData {
  name: string;
  email: string;
}

export interface ApiError {
  message: string;
  status: number;
}

// 封装的 API 函数
export async function createUser(data: CreateUserData): Promise<User> {
  const res = await fetch('/api/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  
  if (!res.ok) {
    const error: ApiError = await res.json();
    throw new Error(error.message || '创建失败');
  }
  
  return res.json();
}

export async function getUsers(): Promise<User[]> {
  const res = await fetch('/api/users', {
    method: 'GET',
  });
  
  if (!res.ok) {
    throw new Error('获取用户列表失败');
  }
  
  return res.json();
}

export async function getUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  
  if (!res.ok) {
    throw new Error('获取用户详情失败');
  }
  
  return res.json();
}

export async function updateUser(id: string, data: Partial<CreateUserData>): Promise<User> {
  const res = await fetch(`/api/users/${id}`, {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  
  if (!res.ok) {
    throw new Error('更新用户失败');
  }
  
  return res.json();
}

export async function deleteUser(id: string): Promise<void> {
  const res = await fetch(`/api/users/${id}`, {
    method: 'DELETE',
  });
  
  if (!res.ok) {
    throw new Error('删除用户失败');
  }
}
```

```tsx
// app/users/user-form.tsx
'use client';

import { createUser } from '@/lib/api/users';  // 导入封装好的 API
import { useState } from 'react';

export function UserForm() {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  
  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    setLoading(true);
    setError(null);
    
    const formData = new FormData(e.currentTarget);
    
    try {
      // 直接调用封装好的函数，类型自动推导
      const user = await createUser({
        name: formData.get('name') as string,
        email: formData.get('email') as string,
      });
      
      console.log('创建成功:', user);
      window.location.reload();
      
    } catch (err) {
      setError(err instanceof Error ? err.message : '未知错误');
    } finally {
      setLoading(false);
    }
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <input name="name" placeholder="姓名" required />
      <input name="email" type="email" placeholder="邮箱" required />
      {error && <p className="error">{error}</p>}
      <button type="submit" disabled={loading}>
        {loading ? '创建中...' : '创建用户'}
      </button>
    </form>
  );
}
```

**优点**：
- 代码复用，多处可使用同一 API 函数
- 类型定义集中，自动推导
- 错误处理统一
- 便于测试和维护

---

### 方式三：Server Actions（最佳实践）

使用 Server Actions 后，**不需要 API 封装层**：

```tsx
// app/users/actions.ts（服务端）
'use server';

import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';

export async function createUser(formData: FormData) {
  const name = formData.get('name') as string;
  const email = formData.get('email') as string;
  
  const user = await db.users.create({
    data: { name, email },
  });
  
  revalidatePath('/users');
  
  return { success: true, user };
}
```

```tsx
// app/users/user-form.tsx（客户端）
'use client';

import { createUser } from './actions';  // 直接导入 Server Action
import { useFormStatus } from 'react-dom';

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? '创建中...' : '创建用户'}
    </button>
  );
}

export function UserForm() {
  return (
    // 表单直接绑定 Server Action
    <form action={createUser}>
      <input name="name" placeholder="姓名" required />
      <input name="email" type="email" placeholder="邮箱" required />
      <SubmitButton />
    </form>
  );
}
```

**对比**：

| 方式 | 文件数量 | 类型安全 | 代码复杂度 |
|------|----------|----------|------------|
| 直接 fetch | 少 | 无 | 高 |
| API 封装层 | 多 | 手动定义 | 中 |
| Server Actions | 最少 | 自动推导 | 低 |

---

## 数据库客户端 db 详解

### 什么是 db？

`import { db } from '@/lib/db'` 中的 `db` 是一个**数据库客户端实例**，它：

1. 连接真实数据库（PostgreSQL、MySQL、MongoDB 等）
2. 提供 API 来执行数据库操作
3. 通常由 ORM（对象关系映射）库提供

### db 是真实数据库吗？

```
┌─────────────────────────────────────────────────────────────────┐
│                     数据库连接架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Server Action / API Route                                      │
│       │                                                         │
│       │ db.users.create({ data: {...} })                        │
│       ▼                                                         │
│  db (ORM 客户端)                                                │
│       │                                                         │
│       │ 转换为 SQL: INSERT INTO users (...)                      │
│       ▼                                                         │
│  真实数据库（PostgreSQL / MySQL / MongoDB）                      │
│       │                                                         │
│       │ 执行 SQL，存储数据                                       │
│       ▼                                                         │
│  返回结果                                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**db 不是数据库本身**，而是：
- 连接数据库的客户端
- 将代码转换为 SQL/数据库操作
- 返回查询结果

---

### 最常用 ORM：Prisma

#### 安装和初始化

```bash
# 安装 Prisma
npm install prisma @prisma/client

# 初始化 Prisma
npx prisma init
```

#### 项目结构

```
project/
├── prisma/
│   └── schema.prisma         ← 数据库模型定义
│
├── lib/
│   └── db.ts                 ← db 客户端实例
│
├── .env                      ← 数据库连接字符串
│
└── app/
    └── users/
        └── actions.ts        ← 使用 db
```

#### 配置数据库连接

```env
# .env

# PostgreSQL
DATABASE_URL="postgresql://user:password@localhost:5432/mydb?schema=public"

# MySQL
# DATABASE_URL="mysql://user:password@localhost:3306/mydb"

# SQLite（本地开发）
# DATABASE_URL="file:./dev.db"

# MongoDB
# DATABASE_URL="mongodb://user:password@localhost:27017/mydb"
```

#### 定义数据库模型

```prisma
// prisma/schema.prisma

datasource db {
  provider = "postgresql"  // 或 mysql, sqlite, mongodb
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

// 用户模型
model User {
  id        String   @id @default(cuid())
  name      String
  email     String   @unique
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  // 关联：用户可以有多个文章
  posts     Post[]
}

// 文章模型
model Post {
  id        String   @id @default(cuid())
  title     String
  content   String?
  published Boolean  @default(false)
  authorId  String
  
  // 关联：文章属于一个用户
  author    User     @relation(fields: [authorId], references: [id])
  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

#### 创建 db 客户端

```tsx
// lib/db.ts

import { PrismaClient } from '@prisma/client';

// 全局变量，防止开发环境热重载时创建多个实例
const globalForPrisma = global as unknown as { db: PrismaClient };

export const db =
  globalForPrisma.db ||
  new PrismaClient({
    log: ['query', 'error', 'warn'],  // 可选：打印日志
  });

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.db = db;
}
```

#### 创建数据库表

```bash
# 创建迁移并应用到数据库
npx prisma migrate dev --name init

# 输出：
# Applying migration `20210419_init`
# The following migration(s) have been created and applied:
# migrations/
#   └─ 20210419_init/
#     └─ migration.sql
#
# Your database is now in sync with your Prisma schema.
```

生成的 SQL：

```sql
-- prisma/migrations/20210419_init/migration.sql

CREATE TABLE "users" (
    "id" TEXT NOT NULL,
    "name" TEXT NOT NULL,
    "email" TEXT NOT NULL,
    "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updatedAt" TIMESTAMP(3) NOT NULL,
    
    CONSTRAINT "users_pkey" PRIMARY KEY ("id")
);

CREATE UNIQUE INDEX "users_email_key" ON "users"("email");

CREATE TABLE "posts" (
    "id" TEXT NOT NULL,
    "title" TEXT NOT NULL,
    "content" TEXT,
    "published" BOOLEAN NOT NULL DEFAULT false,
    "authorId" TEXT NOT NULL,
    "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updatedAt" TIMESTAMP(3) NOT NULL,
    
    CONSTRAINT "posts_pkey" PRIMARY KEY ("id")
);

ALTER TABLE "posts" ADD CONSTRAINT "posts_authorId_fkey" 
    FOREIGN KEY ("authorId") REFERENCES "users"("id") 
    ON DELETE CASCADE ON UPDATE CASCADE;
```

---

### Prisma CRUD 操作示例

#### 创建数据

```tsx
// app/users/actions.ts
'use server';

import { db } from '@/lib/db';
import { revalidatePath } from 'next/cache';

export async function createUser(formData: FormData) {
  // 单条创建
  const user = await db.users.create({
    data: {
      name: formData.get('name') as string,
      email: formData.get('email') as string,
    },
  });
  
  revalidatePath('/users');
  return user;
}

// 批量创建
export async function createManyUsers(users: { name: string; email: string }[]) {
  const result = await db.users.createMany({
    data: users,
    skipDuplicates: true,  // 跳过重复
  });
  
  return result;
}
```

#### 查询数据

```tsx
// 查询所有
export async function getUsers() {
  const users = await db.users.findMany({
    orderBy: { createdAt: 'desc' },
    include: { posts: true },  // 关联查询
  });
  
  return users;
}

// 条件查询
export async function searchUsers(query: string) {
  const users = await db.users.findMany({
    where: {
      OR: [
        { name: { contains: query } },
        { email: { contains: query } },
      ],
    },
  });
  
  return users;
}

// 单条查询
export async function getUser(id: string) {
  const user = await db.users.findUnique({
    where: { id },
    include: { posts: true },
  });
  
  return user;
}

// 通过其他字段查询
export async function getUserByEmail(email: string) {
  const user = await db.users.findFirst({
    where: { email },
  });
  
  return user;
}

// 分页查询
export async function getUsersPaginated(page: number, pageSize: number = 10) {
  const users = await db.users.findMany({
    skip: (page - 1) * pageSize,
    take: pageSize,
    orderBy: { createdAt: 'desc' },
  });
  
  const total = await db.users.count();
  
  return {
    users,
    total,
    page,
    pageSize,
    totalPages: Math.ceil(total / pageSize),
  };
}
```

#### 更新数据

```tsx
export async function updateUser(id: string, data: { name?: string; email?: string }) {
  const user = await db.users.update({
    where: { id },
    data,
  });
  
  revalidatePath('/users');
  revalidatePath(`/users/${id}`);
  
  return user;
}

// 批量更新
export async function updateManyUsers(ids: string[], data: { published: boolean }) {
  const result = await db.users.updateMany({
    where: { id: { in: ids } },
    data,
  });
  
  revalidatePath('/users');
  
  return result;
}

// upsert（存在则更新，不存在则创建）
export async function upsertUser(email: string, name: string) {
  const user = await db.users.upsert({
    where: { email },
    update: { name },
    create: { email, name },
  });
  
  return user;
}
```

#### 删除数据

```tsx
export async function deleteUser(id: string) {
  await db.users.delete({
    where: { id },
  });
  
  revalidatePath('/users');
  
  return { success: true };
}

// 批量删除
export async function deleteManyUsers(ids: string[]) {
  const result = await db.users.deleteMany({
    where: { id: { in: ids } },
  });
  
  revalidatePath('/users');
  
  return result;
}
```

#### 事务操作

```tsx
export async function createPostWithUser(formData: FormData) {
  // 事务：多个操作要么全部成功，要么全部失败
  const result = await db.$transaction(async (tx) => {
    // 创建用户
    const user = await tx.users.create({
      data: {
        name: formData.get('name') as string,
        email: formData.get('email') as string,
      },
    });
    
    // 创建文章
    const post = await tx.posts.create({
      data: {
        title: formData.get('title') as string,
        content: formData.get('content') as string,
        authorId: user.id,
      },
    });
    
    return { user, post };
  });
  
  revalidatePath('/posts');
  
  return result;
}
```

---

### Prisma 支持的数据库

| 数据库 | provider | 说明 |
|--------|----------|------|
| PostgreSQL | `postgresql` | 最推荐，功能强大，支持全文搜索 |
| MySQL | `mysql` | 常用选择，稳定可靠 |
| SQLite | `sqlite` | 轻量级，适合本地开发和小项目 |
| MongoDB | `mongodb` | NoSQL 文档数据库 |
| CockroachDB | `cockroachdb` | 分布式数据库 |
| SQL Server | `sqlserver` | 企业级数据库 |

---

### 其他 ORM 选择

#### Drizzle ORM（轻量级）

```tsx
// lib/db.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from './schema';

const pool = new Pool({ 
  connectionString: process.env.DATABASE_URL 
});

export const db = drizzle(pool, { schema });
```

```tsx
// lib/schema.ts
import { pgTable, text, timestamp, boolean } from 'drizzle-orm/pg-core';
import { relations } from 'drizzle-orm';

export const users = pgTable('users', {
  id: text('id').primaryKey(),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
  createdAt: timestamp('created_at').defaultNow(),
});

export const posts = pgTable('posts', {
  id: text('id').primaryKey(),
  title: text('title').notNull(),
  content: text('content'),
  published: boolean('published').default(false),
  authorId: text('author_id').notNull(),
});

// 定义关联
export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}));
```

```tsx
// 使用 Drizzle
import { db } from '@/lib/db';
import { users } from '@/lib/schema';
import { eq } from 'drizzle-orm';

export async function getUsers() {
  return db.select().from(users);
}

export async function getUser(id: string) {
  return db.select().from(users).where(eq(users.id, id));
}

export async function createUser(data: { name: string; email: string }) {
  return db.insert(users).values(data).returning();
}
```

#### 原生 PostgreSQL（pg 库）

```tsx
// lib/db.ts
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

// 查询函数
export async function query(sql: string, params: any[] = []) {
  const client = await pool.connect();
  try {
    const result = await client.query(sql, params);
    return result.rows;
  } finally {
    client.release();
  }
}

// 事务
export async function transaction(callback: (client: any) => Promise<any>) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await callback(client);
    await client.query('COMMIT');
    return result;
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}
```

```tsx
// 使用原生 SQL
import { query } from '@/lib/db';

export async function getUsers() {
  return query('SELECT * FROM users ORDER BY created_at DESC');
}

export async function createUser(data: { name: string; email: string }) {
  return query(
    'INSERT INTO users (id, name, email) VALUES (gen_random_uuid(), $1, $2) RETURNING *',
    [data.name, data.email]
  );
}

export async function getUser(id: string) {
  const users = await query('SELECT * FROM users WHERE id = $1', [id]);
  return users[0];
}
```

---

### ORM 对比

| 特性 | Prisma | Drizzle | 原生 SQL |
|------|--------|---------|----------|
| 学习曲线 | 低 | 中 | 高 |
| 类型安全 | 自动 | 手动定义 | 无 |
| 迁移管理 | 自动 | 手动/工具 | 手动 |
| 查询语法 | 简洁 | SQL-like | SQL |
| 性能 | 好 | 更好 | 最佳 |
| 包大小 | 较大 | 很小 | 小 |
| 适合场景 | 大型项目 | 中型项目 | 性能敏感 |

---

## 完整示例项目

### 项目结构

```
app/
├── (auth)/
│   ├── login/
│   │   ├── page.tsx
│   │   └── actions.ts
│   └── register/
│   │   ├── page.tsx
│   │   └── actions.ts
│
├── (dashboard)/
│   ├── users/
│   │   ├── page.tsx              ← Server Component（获取数据）
│   │   ├── [id]/page.tsx         ← 用户详情
│   │   ├── actions.ts            ← Server Actions
│   │   ├── user-list.tsx         ← Client Component
│   │   └── user-form.tsx         ← Client Component
│   │
│   └── posts/
│   │   ├── page.tsx
│   │   ├── [slug]/page.tsx
│   │   ├── actions.ts
│   │   └── post-list.tsx
│   │
│   └── layout.tsx
│
├── lib/
│   ├── db.ts                     ← Prisma 客户端
│   ├── auth.ts                   ← 认证逻辑
│   └── utils.ts                  ← 工具函数
│
├── prisma/
│   └── schema.prisma             ← 数据库模型
│
└── .env                          ← 环境变量
```

### 数据库模型

```prisma
// prisma/schema.prisma

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id        String   @id @default(cuid())
  name      String
  email     String   @unique
  password  String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  posts     Post[]
}

model Post {
  id        String   @id @default(cuid())
  title     String
  slug      String   @unique
  content   String?
  published Boolean  @default(false)
  authorId  String
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

### db 客户端

```tsx
// lib/db.ts

import { PrismaClient } from '@prisma/client';

const globalForPrisma = global as unknown as { db: PrismaClient };

export const db =
  globalForPrisma.db ||
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' ? ['query', 'error', 'warn'] : ['error'],
  });

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.db = db;
}
```

### Server Actions

```tsx
// app/(dashboard)/users/actions.ts
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

  const user = await db.users.create({
    data: result.data,
  });

  revalidatePath('/users');
  
  return { success: true, user };
}

// 获取用户列表
export async function getUsers() {
  return db.users.findMany({
    orderBy: { createdAt: 'desc' },
    include: { posts: true },
  });
}

// 获取单个用户
export async function getUser(id: string) {
  return db.users.findUnique({
    where: { id },
    include: { posts: true },
  });
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
  await db.users.delete({
    where: { id },
  });

  revalidatePath('/users');
  
  return { success: true };
}
```

### Server Component 页面

```tsx
// app/(dashboard)/users/page.tsx

import { getUsers, createUser } from './actions';
import UserList from './user-list';
import UserForm from './user-form';

export default async function UsersPage() {
  // 服务端直接获取数据
  const users = await getUsers();

  return (
    <div className="p-8">
      <h1 className="text-2xl font-bold mb-6">用户管理</h1>
      
      {/* 创建表单 */}
      <UserForm />
      
      {/* 用户列表 */}
      <UserList users={users} />
    </div>
  );
}
```

### Client Component

```tsx
// app/(dashboard)/users/user-list.tsx
'use client';

import { deleteUser } from './actions';
import { useState } from 'react';

export default function UserList({ users }: { users: any[] }) {
  const [deleting, setDeleting] = useState<string | null>(null);

  async function handleDelete(id: string) {
    setDeleting(id);
    await deleteUser(id);
    setDeleting(null);
  }

  return (
    <ul className="space-y-2">
      {users.map(user => (
        <li key={user.id} className="p-4 border rounded flex justify-between">
          <div>
            <span className="font-medium">{user.name}</span>
            <span className="text-gray-500 ml-4">{user.email}</span>
          </div>
          <button
            onClick={() => handleDelete(user.id)}
            disabled={deleting === user.id}
            className="text-red-500"
          >
            {deleting === user.id ? '删除中...' : '删除'}
          </button>
        </li>
      ))}
    </ul>
  );
}
```

```tsx
// app/(dashboard)/users/user-form.tsx
'use client';

import { useFormStatus, useFormState } from 'react-dom';
import { createUser } from './actions';

function SubmitButton() {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending} className="bg-blue-500 text-white px-4 py-2 rounded">
      {pending ? '创建中...' : '创建用户'}
    </button>
  );
}

export default function UserForm() {
  const [state, formAction] = useFormState(createUser, null);

  return (
    <form action={formAction} className="mb-8 space-y-4">
      <input name="name" placeholder="姓名" className="border p-2 rounded" required />
      <input name="email" type="email" placeholder="邮箱" className="border p-2 rounded" required />
      
      {state?.error && <p className="text-red-500">{state.error}</p>}
      
      <SubmitButton />
    </form>
  );
}
```

---

## 对比总结

### 客户端调用方式对比

| 方式 | 位置 | 调用方法 | 类型安全 |
|------|------|----------|----------|
| 直接 fetch | Client Component | `fetch('/api/users')` | 无 |
| API 封装层 | `lib/api/*.ts` | `await createUser(data)` | 手动定义 |
| Server Actions | `actions.ts` | `await createUser(formData)` | 自动推导 |

### 数据库客户端对比

| ORM | 类型安全 | 迁移 | 学习难度 |
|-----|----------|------|----------|
| Prisma | 自动 | 自动 | 低 |
| Drizzle | 手动 | 工具 | 中 |
| 原生 SQL | 无 | 手动 | 高 |

### 最佳实践

1. **Server Actions > API Route**：内部操作用 Server Actions
2. **Prisma 推荐**：类型安全 + 自动迁移 + 简洁语法
3. **Server Component 获取数据**：直接调用，无额外 HTTP
4. **Client Component 交互**：导入 Server Action，直接调用

---

## 参考资源

- [Prisma 官方文档](https://www.prisma.io/docs)
- [Drizzle ORM 文档](https://orm.drizzle.team/docs/overview)
- [Next.js Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions)
- [pg (PostgreSQL) 文档](https://node-postgres.com/)