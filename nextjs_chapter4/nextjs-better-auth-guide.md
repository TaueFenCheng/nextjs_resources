---
title: Next.js Better-Auth 认证模块
description: 详细介绍 Better-Auth 在 Next.js 中的配置和使用
---

# Next.js Better-Auth 认证模块

本文详细介绍 Better-Auth 在 Next.js 项目中的配置和使用方式。

## 目录

1. [什么是 Better-Auth](#1-什么是-better-auth)
2. [模块结构](#2-模块结构)
3. [API 路由配置](#3-api-路由配置)
4. [认证配置](#4-认证配置)
5. [客户端使用](#5-客户端使用)
6. [服务端使用](#6-服务端使用)
7. [完整工作流程](#7-完整工作流程)
8. [扩展配置](#8-扩展配置)
9. [常见问题](#9-常见问题)
10. [authClient 方法详解](#10-authclient-方法详解)
11. [自定义账号登录](#11-自定义账号登录)
12. [authClient 方法汇总表](#12-authclient-方法汇总表)

---

## 1. 什么是 Better-Auth

Better-Auth 是一个现代化的 TypeScript 认证库，专为现代 Web 应用设计。

### 特点

| 特点 | 说明 |
|------|------|
| TypeScript | 完整类型支持，自动推导 Session 类型 |
| 多框架支持 | Next.js、Nuxt、SvelteKit、Remix 等 |
| 多认证方式 | 邮箱密码、OAuth、Magic Link、Two Factor 等 |
| 多数据库 | SQLite、PostgreSQL、MySQL、MongoDB 等 |
| 零配置 | 默认配置即可使用 |
| 类型安全 | 服务端和客户端共享类型 |

### 与其他方案对比

| 方案 | Better-Auth | NextAuth.js | Clerk |
|------|-------------|-------------|-------|
| 类型安全 | ✅ 完整 | 部分 | ✅ |
| 自托管 | ✅ | ✅ | ❌ (SaaS) |
| 配置复杂度 | 低 | 中 | 低 |
| 数据库支持 | 多种 | 有限 | 自管理 |
| 价格 | 免费 | 免费 | 付费套餐 |

---

## 2. 模块结构

Better-Auth 在 Next.js 项目中的标准目录结构：

```
src/
├── app/
│   └── api/
│       └── auth/
│           └── [...all]/      ← catch-all 路由
│               └── route.ts   ← API 入口
├── server/
│   └── better-auth/
│       ├── config.ts    ← 认证配置（服务端）
│       ├── client.ts    ← React 客户端 hooks
│       ├── server.ts    ← 服务端获取 session
│       └── index.ts     ← 导出入口
└── components/
    └── auth/
        ├── login-form.tsx
        └── user-menu.tsx
```

### 文件职责

| 文件 | 职责 |
|------|------|
| `app/api/auth/[...all]/route.ts` | 拦截所有认证 API 请求 |
| `config.ts` | 配置认证策略、数据库、插件 |
| `client.ts` | 提供客户端 React hooks |
| `server.ts` | 提供服务端 session 获取方法 |
| `index.ts` | 统一导出入口 |

---

## 3. API 路由配置

### catch-all 路由

```tsx
// src/app/api/auth/[...all]/route.ts
import { toNextJsHandler } from "better-auth/next-js";
import { auth } from "@/server/better-auth";

// 导出 GET 和 POST 方法
export const { GET, POST } = toNextJsHandler(auth.handler);
```

**关键点：**

- `[...all]` 是 Next.js 的 **catch-all 路由**，匹配所有 `/api/auth/*` 下的路径
- `toNextJsHandler` 将 better-auth handler 转换为 Next.js API Route 格式
- 只需导出 GET 和 POST，better-auth 内部处理所有认证逻辑

### 路由映射

```
请求路径                              →  Better-Auth 处理

/api/auth/sign-in/email              →  邮箱登录
/api/auth/sign-up/email              →  邮箱注册
/api/auth/sign-out                   →  登出
/api/auth/session                    →  获取当前 session
/api/auth/verify-email               →  验证邮箱
/api/auth/reset-password             →  重置密码
/api/auth/update-password            →  更新密码
/api/auth/callback/github            →  GitHub OAuth 回调
/api/auth/callback/google            →  Google OAuth 回调
/api/auth/sign-in/github             →  GitHub OAuth 登录入口
/api/auth/...                        →  所有其他认证路由
```

### 为什么使用 catch-all

1. **简洁**：一个文件处理所有认证路由
2. **自动更新**：better-auth 新增功能自动可用
3. **维护方便**：无需手动添加每个路由

---

## 4. 认证配置

### 基础配置

```tsx
// src/server/better-auth/config.ts
import { betterAuth } from "better-auth";

export const auth = betterAuth({
  // 基础 URL（生产环境需要配置）
  baseURL: process.env.BETTER_AUTH_URL || "http://localhost:3000",

  // 邮箱密码登录
  emailAndPassword: {
    enabled: true,
  },

  // 数据库配置（可选，默认使用 SQLite）
  database: {
    provider: "sqlite",
    url: "./auth.db",
  },
});

// 导出类型供客户端使用
export type Session = typeof auth.$Infer.Session;
```

### 完整配置示例

```tsx
// src/server/better-auth/config.ts
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { nextCookies } from "better-auth/next-js";
import { prisma } from "@/lib/prisma";

export const auth = betterAuth({
  // 基础配置
  baseURL: process.env.NEXT_PUBLIC_APP_URL,
  secret: process.env.BETTER_AUTH_SECRET,  // 加密密钥

  // 数据库适配器
  database: prismaAdapter(prisma, {
    provider: "postgresql",
  }),

  // 邮箱密码登录
  emailAndPassword: {
    enabled: true,
    requireEmailVerification: true,  // 要求邮箱验证
  },

  // OAuth providers
  socialProviders: {
    github: {
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    },
    google: {
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    },
  },

  // 插件
  plugins: [
    nextCookies(),  // Next.js cookie 支持
  ],

  // 用户表额外字段
  user: {
    additionalFields: {
      role: {
        type: "string",
        required: false,
        defaultValue: "user",
      },
      avatar: {
        type: "string",
        required: false,
      },
    },
  },

  // Session 配置
  session: {
    expiresIn: 60 * 60 * 24 * 7,  // 7 天
    updateAge: 60 * 60 * 24,      // 每天更新
    cookieCache: {
      enabled: true,
      maxAge: 60 * 5,  // 5 分钟缓存
    },
  },

  // 高级配置
  advanced: {
    generateId: false,  // 使用默认 ID 生成
  },
});

export type Session = typeof auth.$Infer.Session;
export type User = typeof auth.$Infer.User;
```

### 数据库适配器

Better-Auth 支持多种数据库：

```tsx
// SQLite（默认，无需配置）
database: {
  provider: "sqlite",
  url: "./auth.db",
}

// PostgreSQL (使用 Drizzle)
import { drizzleAdapter } from "better-auth/adapters/drizzle";
database: drizzleAdapter(db, { provider: "postgresql" })

// PostgreSQL (使用 Prisma)
import { prismaAdapter } from "better-auth/adapters/prisma";
database: prismaAdapter(prisma, { provider: "postgresql" })

// MySQL
database: {
  provider: "mysql",
  url: "mysql://user:pass@localhost/db",
}

// MongoDB
import { mongodbAdapter } from "better-auth/adapters/mongodb";
database: mongodbAdapter(mongoClient.db())
```

### 数据库表结构

Better-Auth 自动创建以下表：

```sql
-- 用户表
CREATE TABLE user (
  id TEXT PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  emailVerified BOOLEAN,
  name TEXT,
  image TEXT,
  createdAt TIMESTAMP,
  updatedAt TIMESTAMP
);

-- 账户表（存储 OAuth 等）
CREATE TABLE account (
  id TEXT PRIMARY KEY,
  userId TEXT NOT NULL,
  accountType TEXT NOT NULL,
  provider TEXT NOT NULL,
  providerAccountId TEXT,
  refreshToken TEXT,
  accessToken TEXT,
  expiresAt TIMESTAMP,
  FOREIGN KEY (userId) REFERENCES user(id)
);

-- Session 表
CREATE TABLE session (
  id TEXT PRIMARY KEY,
  userId TEXT NOT NULL,
  expiresAt TIMESTAMP,
  token TEXT UNIQUE,
  ipAddress TEXT,
  userAgent TEXT,
  FOREIGN KEY (userId) REFERENCES user(id)
);

-- 验证表（邮箱验证等）
CREATE TABLE verification (
  id TEXT PRIMARY KEY,
  identifier TEXT NOT NULL,
  value TEXT NOT NULL,
  expiresAt TIMESTAMP,
);
```

---

## 5. 客户端使用

### 创建客户端实例

```tsx
// src/server/better-auth/client.ts
import { createAuthClient } from "better-auth/react";

// 基础配置
export const authClient = createAuthClient();

// 或者自定义配置
export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_APP_URL,
});

// 导出类型
export type Session = typeof authClient.$Infer.Session;
```

### React Hooks

Better-Auth 提供以下 React hooks：

```tsx
'use client';

import { authClient } from '@/server/better-auth/client';

// useSession - 获取当前 session
const { data: session, isPending, error } = authClient.useSession();

// signIn - 登录方法
const { signIn } = authClient;

// signUp - 注册方法
const { signUp } = authClient;

// signOut - 登出方法
const { signOut } = authClient;

// useUser - 仅获取用户信息
const { user } = authClient.useUser();
```

### 登录组件示例

```tsx
// components/auth/login-form.tsx
'use client';

import { authClient } from '@/server/better-auth/client';
import { useState } from 'react';

export function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);
    setError('');

    const result = await authClient.signIn.email({
      email,
      password,
    });

    setLoading(false);

    if (result.error) {
      setError(result.error.message);
    } else {
      // 登录成功，可以跳转
      window.location.href = '/dashboard';
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={email}
        onChange={e => setEmail(e.target.value)}
        placeholder="邮箱"
        required
      />
      <input
        type="password"
        value={password}
        onChange={e => setPassword(e.target.value)}
        placeholder="密码"
        required
      />
      {error && <p className="error">{error}</p>}
      <button disabled={loading}>
        {loading ? '登录中...' : '登录'}
      </button>
    </form>
  );
}
```

### 注册组件示例

```tsx
// components/auth/register-form.tsx
'use client';

import { authClient } from '@/server/better-auth/client';
import { useState } from 'react';

export function RegisterForm() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();

    const result = await authClient.signUp.email({
      email,
      password,
      name,  // 可选
    });

    if (result.error) {
      console.error(result.error);
    } else {
      console.log('注册成功:', result.data);
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        type="text"
        value={name}
        onChange={e => setName(e.target.value)}
        placeholder="姓名"
      />
      <input
        type="email"
        value={email}
        onChange={e => setEmail(e.target.value)}
        placeholder="邮箱"
        required
      />
      <input
        type="password"
        value={password}
        onChange={e => setPassword(e.target.value)}
        placeholder="密码"
        required
        minLength={8}
      />
      <button>注册</button>
    </form>
  );
}
```

### OAuth 登录示例

```tsx
// components/auth/oauth-buttons.tsx
'use client';

import { authClient } from '@/server/better-auth/client';

export function OAuthButtons() {
  async function signInWithGitHub() {
    await authClient.signIn.social({
      provider: 'github',
      callbackURL: '/dashboard',  // 登录后跳转
    });
  }

  async function signInWithGoogle() {
    await authClient.signIn.social({
      provider: 'google',
      callbackURL: '/dashboard',
    });
  }

  return (
    <div>
      <button onClick={signInWithGitHub}>
        使用 GitHub 登录
      </button>
      <button onClick={signInWithGoogle}>
        使用 Google 登录
      </button>
    </div>
  );
}
```

### 用户菜单示例

```tsx
// components/auth/user-menu.tsx
'use client';

import { authClient } from '@/server/better-auth/client';

export function UserMenu() {
  const { data: session, isPending } = authClient.useSession();

  if (isPending) {
    return <div>加载中...</div>;
  }

  if (!session) {
    return <a href="/login">登录</a>;
  }

  return (
    <div className="user-menu">
      <img
        src={session.user.image || '/default-avatar.png'}
        alt={session.user.name}
      />
      <span>{session.user.name || session.user.email}</span>
      <button onClick={() => authClient.signOut()}>
        登出
      </button>
    </div>
  );
}
```

---

## 6. 服务端使用

### 创建服务端获取方法

```tsx
// src/server/better-auth/server.ts
import { headers } from "next/headers";
import { cache } from "react";
import { auth } from ".";

/**
 * 获取当前 session
 * 使用 React cache 在同一请求内缓存结果
 */
export const getSession = cache(async () =>
  auth.api.getSession({
    headers: await headers(),
  }),
);
```

**关键点：**

- `headers()` 获取当前请求头（包含 cookies）
- `cache()` 确保同一请求内只调用一次
- 返回 session 或 null

### Server Component 中使用

```tsx
// app/dashboard/page.tsx (Server Component)
import { getSession } from '@/server/better-auth/server';
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
  const session = await getSession();

  // 未登录跳转到登录页
  if (!session) {
    redirect('/login');
  }

  return (
    <div>
      <h1>仪表板</h1>
      <p>欢迎, {session.user.name || session.user.email}</p>
      {/* 只有登录用户才能看到的内容 */}
    </div>
  );
}
```

### Server Action 中使用

```tsx
// actions/user.ts
'use server';

import { getSession } from '@/server/better-auth/server';
import { db } from '@/lib/db';

export async function updateUserProfile(data: { name: string }) {
  const session = await getSession();

  if (!session) {
    return { error: '未登录' };
  }

  const user = await db.users.update({
    where: { id: session.user.id },
    data,
  });

  return { success: true, user };
}
```

### Middleware 中保护路由

```tsx
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // 需要登录的路径
  const protectedPaths = ['/dashboard', '/settings', '/profile'];

  if (protectedPaths.some(path => pathname.startsWith(path))) {
    // 检查 session cookie
    const sessionToken = request.cookies.get('better-auth.session_token');

    if (!sessionToken) {
      return NextResponse.redirect(new URL('/login', request.url));
    }
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/settings/:path*', '/profile/:path*'],
};
```

---

## 7. 完整工作流程

### 登录流程

```
┌─────────────────────────────────────────────────────────────────┐
│                       用户登录流程                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 用户输入邮箱密码                                              │
│       │                                                         │
│       │  authClient.signIn.email({ email, password })           │
│       ▼                                                         │
│  2. 客户端发送 POST 请求                                          │
│       │                                                         │
│       │  POST /api/auth/sign-in/email                           │
│       │  Body: { email, password }                              │
│       ▼                                                         │
│  3. Next.js API Route 接收                                       │
│       │                                                         │
│       │  app/api/auth/[...all]/route.ts                         │
│       │  toNextJsHandler(auth.handler)                          │
│       ▼                                                         │
│  4. Better-Auth Handler 处理                                     │
│       │                                                         │
│       │  ├─ 验证邮箱格式                                         │
│       │  ├─ 查询数据库是否存在用户                                │
│       │  ├─ 验证密码 hash                                        │
│       │  ├─ 创建 session 记录                                    │
│       │  └─ 设置 cookie                                          │
│       ▼                                                         │
│  5. 返回响应                                                     │
│       │                                                         │
│       │  Response: {                                            │
│       │    user: { id, email, name, ... },                      │
│       │    session: { id, expiresAt, ... }                      │
│       │  }                                                       │
│       │  Cookies: better-auth.session_token=xxx                 │
│       ▼                                                         │
│  6. 客户端接收响应                                                │
│       │                                                         │
│       │  authClient.useSession() 自动更新                        │
│       │  页面跳转到 dashboard                                     │
│       ▼                                                         │
│  7. 用户已登录状态                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### OAuth 登录流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      OAuth 登录流程                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 用户点击 "使用 GitHub 登录"                                    │
│       │                                                         │
│       │  authClient.signIn.social({ provider: 'github' })       │
│       ▼                                                         │
│  2. 客户端请求 OAuth 入口                                         │
│       │                                                         │
│       │  GET /api/auth/sign-in/github                           │
│       ▼                                                         │
│  3. Better-Auth 返回 OAuth URL                                   │
│       │                                                         │
│       │  Response: { url: "https://github.com/login/oauth/..." }│
│       ▼                                                         │
│  4. 浏览器跳转到 GitHub                                           │
│       │                                                         │
│       │  用户在 GitHub 授权                                      │
│       ▼                                                         │
│  5. GitHub 回调                                                  │
│       │                                                         │
│       │  GET /api/auth/callback/github?code=xxx                 │
│       ▼                                                         │
│  6. Better-Auth 处理回调                                         │
│       │                                                         │
│       │  ├─ 使用 code 获取 access_token                          │
│       │  ├─ 获取 GitHub 用户信息                                 │
│       │  ├─ 创建/更新用户记录                                    │
│       │  ├─ 创建 session                                         │
│       │  └─ 设置 cookie                                          │
│       ▼                                                         │
│  7. 跳转到 callbackURL                                           │
│       │                                                         │
│       │  redirect('/dashboard')                                 │
│       ▼                                                         │
│  8. 用户已登录                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Session 验证流程

```
┌─────────────────────────────────────────────────────────────────┐
│                     Session 验证流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 用户访问需要登录的页面                                         │
│       │                                                         │
│       │  GET /dashboard                                         │
│       │  Cookies: better-auth.session_token=xxx                 │
│       ▼                                                         │
│  2. Server Component 调用 getSession()                           │
│       │                                                         │
│       │  const session = await getSession()                     │
│       ▼                                                         │
│  3. Better-Auth 验证 session                                     │
│       │                                                         │
│       │  ├─ 从 cookie 获取 session token                        │
│       │  ├─ 查询数据库验证 session                               │
│       │  ├─ 检查是否过期                                         │
│       │  └─ 返回 session 数据                                    │
│       ▼                                                         │
│  4. 根据结果处理                                                  │
│       │                                                         │
│       │  session 存在 → 渲染页面                                 │
│       │  session 不存在 → redirect('/login')                    │
│       ▼                                                         │
│  5. 页面渲染                                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. 扩展配置

### 添加 OAuth Provider

```tsx
// config.ts
import { betterAuth } from "better-auth";

export const auth = betterAuth({
  socialProviders: {
    // GitHub
    github: {
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    },
    // Google
    google: {
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    },
    // Discord
    discord: {
      clientId: process.env.DISCORD_CLIENT_ID!,
      clientSecret: process.env.DISCORD_CLIENT_SECRET!,
    },
    // Twitter
    twitter: {
      clientId: process.env.TWITTER_CLIENT_ID!,
      clientSecret: process.env.TWITTER_CLIENT_SECRET!,
    },
  },
});
```

### 添加插件

```tsx
// config.ts
import { betterAuth } from "better-auth";
import { twoFactor } from "better-auth/plugins/two-factor";
import { username } from "better-auth/plugins/username";
import { admin } from "better-auth/plugins/admin";
import { nextCookies } from "better-auth/next-js";

export const auth = betterAuth({
  plugins: [
    // Next.js cookie 支持
    nextCookies(),

    // 用户名登录（替代邮箱）
    username(),

    // 两步验证
    twoFactor({
      issuer: "Your App Name",
    }),

    // 管理员功能
    admin({
      acRoles: ["admin", "user"],
    }),
  ],
});
```

### 用户额外字段

```tsx
// config.ts
export const auth = betterAuth({
  user: {
    additionalFields: {
      // 用户角色
      role: {
        type: "string",
        required: false,
        defaultValue: "user",
        input: false,  // 注册时不可输入
      },
      // 头像
      avatar: {
        type: "string",
        required: false,
      },
      // 电话
      phone: {
        type: "string",
        required: false,
      },
    },
  },
});

// 使用时
const session = await getSession();
console.log(session.user.role);  // "admin" | "user"
```

### 自定义 Session 存储

```tsx
// config.ts
export const auth = betterAuth({
  session: {
    // Session 有效期
    expiresIn: 60 * 60 * 24 * 7,  // 7 天

    // 更新间隔（滑动过期）
    updateAge: 60 * 60 * 24,  // 每天更新

    // Cookie 缓存（减少数据库查询）
    cookieCache: {
      enabled: true,
      maxAge: 60 * 5,  // 5 分钟
    },

    // IP 混淆（隐私保护）
    ipAddress: {
      ipAddress: true,
      userAgent: true,
    },
  },
});
```

### 邮箱验证

```tsx
// config.ts
export const auth = betterAuth({
  emailAndPassword: {
    enabled: true,
    requireEmailVerification: true,  // 强制验证邮箱
  },

  // 验证邮件配置
  emailVerification: {
    sendVerificationEmail: async ({ user, url }) => {
      // 发送验证邮件
      await sendEmail({
        to: user.email,
        subject: "验证您的邮箱",
        body: `点击链接验证: ${url}`,
      });
    },
  },
});
```

---

## 9. 常见问题

### Q1: 如何获取环境变量？

```bash
# .env.local
BETTER_AUTH_SECRET=your-secret-key-at-least-32-characters
NEXT_PUBLIC_APP_URL=http://localhost:3000

# OAuth providers
GITHUB_CLIENT_ID=your-github-client-id
GITHUB_CLIENT_SECRET=your-github-client-secret
```

### Q2: 如何生成 BETTER_AUTH_SECRET？

```bash
# 使用 openssl
openssl rand -base64 32

# 或 Node.js
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

### Q3: 生产环境需要什么配置？

```tsx
// config.ts
export const auth = betterAuth({
  // 必须配置生产环境 URL
  baseURL: process.env.NEXT_PUBLIC_APP_URL,

  // 必须设置 secret
  secret: process.env.BETTER_AUTH_SECRET,

  // 数据库（不能用 SQLite 文件）
  database: prismaAdapter(prisma, {
    provider: "postgresql",
  }),

  // 安全配置
  session: {
    cookieCache: {
      enabled: true,
      maxAge: 60 * 5,
    },
  },

  // Trusted Origins（防止 CSRF）
  trustedOrigins: [
    "https://yourdomain.com",
    "https://api.yourdomain.com",
  ],
});
```

### Q4: 如何自定义登录页面跳转？

```tsx
// 登录成功后跳转
await authClient.signIn.email({
  email,
  password,
  callbackURL: "/dashboard",  // 指定跳转页面
});

// 登出后跳转
await authClient.signOut({
  callbackURL: "/login",
});
```

### Q5: 如何保护 API Route？

```tsx
// app/api/protected/route.ts
import { auth } from "@/server/better-auth";
import { NextRequest, NextResponse } from "next/server";

export async function GET(request: NextRequest) {
  // 获取 session
  const session = await auth.api.getSession({
    headers: request.headers,
  });

  if (!session) {
    return NextResponse.json(
      { error: "未授权" },
      { status: 401 }
    );
  }

  // 返回受保护的数据
  return NextResponse.json({
    message: "受保护的 API",
    user: session.user,
  });
}
```

### Q6: 如何检查用户角色？

```tsx
// server.ts
import { auth } from "./config";

export async function requireAdmin() {
  const session = await getSession();

  if (!session) {
    throw new Error("未登录");
  }

  // 需要配置 admin 插件
  const isAdmin = await auth.api.userHasRole({
    userId: session.user.id,
    role: "admin",
  });

  if (!isAdmin) {
    throw new Error("需要管理员权限");
  }

  return session;
}
```

### Q7: 如何在同一请求内多次使用 session？

使用 React `cache`：

```tsx
// server.ts
import { cache } from "react";

export const getSession = cache(async () => ...);

// 在多个组件中使用，只查询一次
async function Page() {
  const session1 = await getSession();
  const session2 = await getSession();  // 返回缓存结果
}
```

---

## 总结

Better-Auth 是现代化的 TypeScript 认证方案，关键配置：

| 配置项 | 说明 |
|------|------|
| `[...all]/route.ts` | catch-all 拦截所有认证请求 |
| `config.ts` | 认证策略、数据库、插件配置 |
| `client.ts` | React hooks（useSession, signIn, signOut） |
| `server.ts` | 服务端 getSession + React cache |

核心优势：
- 类型安全（服务端/客户端共享类型）
- 零配置（默认 SQLite 即可使用）
- 多认证方式（邮箱密码、OAuth、Two Factor）
- 多数据库支持
- Next.js 原生集成

---

## 10. authClient 方法详解

### 10.1 创建客户端实例

```tsx
// src/server/better-auth/client.ts
import { createAuthClient } from "better-auth/react";

// 基础配置
export const authClient = createAuthClient();

// 自定义配置
export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_APP_URL,  // API 基础路径
  basePath: "/api/auth",                     // 认证路径
  disableDefaultFetchPlugins: false,         // 是否禁用默认插件

  // 插件配置（需要与服务端插件对应）
  plugins: [
    usernameClient(),  // 用户名登录插件
  ],

  // 生命周期钩子
  fetchOptions: {
    onSuccess: (ctx) => console.log("成功", ctx),
    onError: (ctx) => console.log("失败", ctx),
    onRequest: (ctx) => console.log("请求", ctx),
    onResponse: (ctx) => console.log("响应", ctx),
  },
});

// 类型导出
export type Session = typeof authClient.$Infer.Session;
export type User = typeof authClient.$Infer.User;
```

### 10.2 默认 API 路径

所有方法默认调用 `/api/auth/*` 下的接口：

```
baseURL: "/api/auth"（默认）

├── /sign-in/email      → POST  邮箱登录
├── /sign-in/username   → POST  用户名登录（需插件）
├── /sign-in/social     → POST  OAuth 登录
├── /sign-up/email      → POST  邮箱注册
├── /sign-out           → POST  登出
├── /session            → GET   获取 session
├── /update-user        → POST  更新用户
├── /change-password    → POST  修改密码
├── /delete-user        → POST  删除账户
└── ... 更多接口
```

### 10.3 邮箱登录 `signIn.email()`

```tsx
// POST /api/auth/sign-in/email
const result = await authClient.signIn.email({
  email: "user@example.com",
  password: "password123",
  callbackURL: "/dashboard",  // 登录成功跳转（可选）
  rememberMe: true,           // 记住登录状态（可选）
});

// 返回值结构
result = {
  data: {
    user: {
      id: "xxx",
      email: "user@example.com",
      name: "John",
      image: "https://...",
      emailVerified: true,
      createdAt: "2024-01-01",
      updatedAt: "2024-01-01",
    },
    session: {
      id: "xxx",
      userId: "xxx",
      expiresAt: "2024-01-08",
      token: "session-token",
      ipAddress: "1.2.3.4",
      userAgent: "Chrome/...",
    },
  },
  error: null | { message: string, code: string },
};
```

### 10.4 邮箱注册 `signUp.email()`

```tsx
// POST /api/auth/sign-up/email
const result = await authClient.signUp.email({
  email: "user@example.com",
  password: "password123",
  name: "John Doe",           // 可选
  image: "https://...",       // 可选
  callbackURL: "/welcome",    // 注册成功跳转（可选）
});

// 返回值结构
result = {
  data: {
    user: { id, email, name, ... },
    session: { id, userId, ... },
    token: "verification-token",  // 需邮箱验证时返回
  },
  error: null | { message, code },
};
```

### 10.5 OAuth 登录 `signIn.social()`

```tsx
// POST /api/auth/sign-in/social
const result = await authClient.signIn.social({
  provider: "github",         // github | google | discord | twitter | apple 等
  callbackURL: "/dashboard",  // 登录成功跳转
  errorCallbackURL: "/login", // 登录失败跳转（可选）
  newUserCallbackURL: "/onboarding", // 新用户跳转（可选）
  scopes: ["user", "repo"],   // 自定义 scope（可选）
  disableRedirect: false,     // 是否禁用自动跳转
});

// disableRedirect: true 时返回 URL
result = {
  data: {
    url: "https://github.com/login/oauth/authorize?...",
    redirect: true,
  },
  error: null,
};

// 使用 idToken 直接登录（无需跳转）
const result = await authClient.signIn.social({
  provider: "google",
  idToken: {
    token: "google-id-token",
    nonce: "optional-nonce",
  },
});
```

### 10.6 登出 `signOut()`

```tsx
// POST /api/auth/sign-out
await authClient.signOut({
  callbackURL: "/login",  // 登出后跳转（可选）
  fetchOptions: {
    onSuccess: () => {
      console.log("已登出");
      // 清理本地状态
    },
  },
});
```

### 10.7 Session 管理

#### 获取 Session `useSession()`

```tsx
'use client';

import { authClient } from '@/server/better-auth/client';

function Component() {
  const { data, isPending, error, refetch } = authClient.useSession();

  // data 结构
  data = {
    user: {
      id: "xxx",
      email: "user@example.com",
      name: "John",
      image: "https://...",
      emailVerified: true,
      createdAt: Date,
      updatedAt: Date,
    },
    session: {
      id: "xxx",
      userId: "xxx",
      expiresAt: Date,
      token: "xxx",
      ipAddress: "1.2.3.4",
      userAgent: "Chrome/...",
    },
  } | null;

  if (isPending) return <Spinner />;
  if (error) return <Error error={error} />;
  if (!data) return <LoginButton />;
  return <UserMenu user={data.user} />;
}
```

#### 查看所有登录设备 `listSessions()`

```tsx
// GET /api/auth/list-sessions
const { data: sessions } = await authClient.listSessions();

// 返回结构
sessions = [
  {
    id: "session-1",
    userId: "user-id",
    expiresAt: "2024-01-08",
    ipAddress: "192.168.1.1",
    userAgent: "Chrome/Windows",
    createdAt: "2024-01-01",
  },
  {
    id: "session-2",
    ipAddress: "10.0.0.1",
    userAgent: "Safari/Mac",
  },
];
```

#### 撤销 Session

```tsx
// 撤销所有其他登录
await authClient.revokeOtherSessions();

// 撤销所有登录（包括当前）
await authClient.revokeSessions();

// 撤销指定 session
await authClient.revokeSession({
  token: "session-token",
});
```

### 10.8 用户管理

#### 更新用户信息 `updateUser()`

```tsx
// POST /api/auth/update-user
const result = await authClient.updateUser({
  name: "New Name",
  image: "https://new-avatar.png",
  // 其他自定义字段
});

result = {
  data: {
    user: { id, email, name: "New Name", ... },
    success: true,
  },
  error: null,
};
```

#### 修改邮箱 `changeEmail()`

```tsx
// POST /api/auth/change-email
const result = await authClient.changeEmail({
  newEmail: "new@example.com",
  callbackURL: "/verified",  // 验证后跳转
});
```

#### 删除账户 `deleteUser()`

```tsx
// POST /api/auth/delete-user
await authClient.deleteUser({
  callbackURL: "/",             // 删除后跳转
  password: "current-password", // 需确认密码（可选）
});
```

### 10.9 密码管理

#### 修改密码 `changePassword()`

```tsx
// POST /api/auth/change-password
const result = await authClient.changePassword({
  newPassword: "newPassword123",
  currentPassword: "oldPassword123",    // 当前密码
  revokeOtherSessions: true,            // 撤销其他登录
});
```

#### 重置密码流程

```tsx
// 步骤 1：请求重置
await authClient.requestPasswordReset({
  email: "user@example.com",
  redirectTo: "/reset-password",  // 重置页面 URL
});

// 步骤 2：执行重置（用户收到邮件后）
await authClient.resetPassword({
  token: "reset-token-from-url",
  newPassword: "newPassword123",
});
```

### 10.10 邮箱验证

```tsx
// 发送验证邮件
await authClient.sendVerificationEmail({
  email: "user@example.com",
  callbackURL: "/verified",
});

// 验证邮箱
const result = await authClient.verifyEmail({
  token: "verification-token-from-email",
  callbackURL: "/dashboard",
});
```

### 10.11 账户管理

```tsx
// 查看关联账户
const { data: accounts } = await authClient.listUserAccounts();

// 关联社交账户
await authClient.linkSocialAccount({
  provider: "github",
  callbackURL: "/settings",
});

// 解绑账户
await authClient.unlinkAccount({
  providerId: "github",
  accountId: "provider-account-id",
});
```

### 10.12 内部属性

```tsx
const client = authClient;

// 直接调用 API
const response = await client.$fetch("/custom-endpoint", {
  method: "POST",
  body: { data: "value" },
});

// 状态管理
client.$store.atoms.session;      // session 状态
client.$store.notify("$sessionSignal");  // 手动刷新 session
client.$store.listen("$sessionSignal", callback);  // 监听变化

// 类型推导
type Session = typeof authClient.$Infer.Session;
type User = typeof authClient.$Infer.User;
```

---

## 11. 自定义账号登录

Better-Auth 默认使用邮箱登录，但可以通过插件支持**用户名登录**。

### 11.1 启用用户名登录

```tsx
// src/server/better-auth/config.ts
import { betterAuth } from "better-auth";
import { username } from "better-auth/plugins/username";

export const auth = betterAuth({
  // 保留邮箱密码登录
  emailAndPassword: {
    enabled: true,
  },

  // 添加用户名插件
  plugins: [
    username({
      // 用户名最小长度
      minUsernameLength: 3,

      // 用户名最大长度
      maxUsernameLength: 30,

      // 自定义用户名验证器
      usernameValidator: async (username) => {
        // 默认: /^[a-zA-Z0-9_.]+$/
        return /^[a-zA-Z0-9_]+$/.test(username);
      },

      // 用户名规范化（默认小写）
      usernameNormalization: (username) => username.toLowerCase(),
    }),
  ],

  // 用户表添加 username 字段
  user: {
    additionalFields: {
      username: {
        type: "string",
        unique: true,
        required: false,
      },
    },
  },
});
```

### 11.2 客户端配置

```tsx
// src/server/better-auth/client.ts
import { createAuthClient } from "better-auth/react";
import { usernameClient } from "better-auth/plugins/username/client";

export const authClient = createAuthClient({
  plugins: [usernameClient()],
});
```

### 11.3 用户名注册

```tsx
// 方式一：仅用户名
await authClient.signUp.email({
  username: "johndoe",      // 用户名
  password: "password123",
  name: "John Doe",         // 可选
});

// 方式二：用户名 + 邮箱
await authClient.signUp.email({
  email: "user@example.com",
  username: "johndoe",
  password: "password123",
});
```

### 11.4 用户名登录

```tsx
// POST /api/auth/sign-in/username
const result = await authClient.signIn.username({
  username: "johndoe",
  password: "password123",
  callbackURL: "/dashboard",
  rememberMe: true,
});

// 返回值与邮箱登录相同
result = {
  data: {
    user: { id, username: "johndoe", email, ... },
    session: { ... },
  },
  error: null,
};
```

### 11.5 检查用户名可用性

```tsx
// POST /api/auth/is-username-available
const { data } = await authClient.isUsernameAvailable({
  username: "johndoe",
});

console.log(data.available);  // true 或 false
```

### 11.6 用户名规则配置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `minUsernameLength` | 3 | 最小长度 |
| `maxUsernameLength` | 30 | 最大长度 |
| `usernameValidator` | `/^[a-zA-Z0-9_.]+$/` | 格式验证 |
| `usernameNormalization` | `toLowerCase()` | 规范化处理 |

### 11.7 自定义用户名验证

```tsx
// 只允许字母和数字，不允许特殊字符
username({
  usernameValidator: async (username) => {
    return /^[a-zA-Z0-9]+$/.test(username);
  },
})

// 允许中文用户名
username({
  usernameValidator: async (username) => {
    return /^[\u4e00-\u9fa5a-zA-Z0-9]+$/.test(username);
  },
  minUsernameLength: 2,
  maxUsernameLength: 20,
})
```

### 11.8 完整登录页面示例（用户名版）

```tsx
// app/login/page.tsx
'use client';

import { authClient } from '@/server/better-auth/client';
import { useState } from 'react';

export default function LoginPage() {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');

  async function handleLogin(e: React.FormEvent) {
    e.preventDefault();
    setError('');

    const result = await authClient.signIn.username({
      username,
      password,
      callbackURL: '/dashboard',
    });

    if (result.error) {
      setError(result.error.message);
    }
  }

  return (
    <form onSubmit={handleLogin}>
      <input
        type="text"
        value={username}
        onChange={e => setUsername(e.target.value)}
        placeholder="用户名"
        required
      />
      <input
        type="password"
        value={password}
        onChange={e => setPassword(e.target.value)}
        placeholder="密码"
        required
      />
      {error && <p className="error">{error}</p>}
      <button>登录</button>
    </form>
  );
}
```

---

## 12. authClient 方法汇总表

### 认证方法

| 方法 | HTTP | 路径 | 说明 |
|------|------|------|------|
| `signIn.email()` | POST | `/sign-in/email` | 邮箱登录 |
| `signIn.username()` | POST | `/sign-in/username` | 用户名登录（插件） |
| `signIn.social()` | POST | `/sign-in/social` | OAuth 登录 |
| `signUp.email()` | POST | `/sign-up/email` | 邮箱注册 |
| `signOut()` | POST | `/sign-out` | 登出 |

### Session 方法

| 方法 | HTTP | 路径 | 说明 |
|------|------|------|------|
| `useSession()` | GET | `/session` | React hook |
| `getSession()` | GET | `/session` | 获取 session |
| `listSessions()` | GET | `/list-sessions` | 查看所有登录 |
| `revokeSessions()` | POST | `/revoke-sessions` | 撤销所有登录 |
| `revokeOtherSessions()` | POST | `/revoke-other-sessions` | 撤销其他登录 |
| `revokeSession()` | POST | `/revoke-session` | 撤销指定登录 |

### 用户方法

| 方法 | HTTP | 路径 | 说明 |
|------|------|------|------|
| `updateUser()` | POST | `/update-user` | 更新信息 |
| `changeEmail()` | POST | `/change-email` | 修改邮箱 |
| `changePassword()` | POST | `/change-password` | 修改密码 |
| `setPassword()` | POST | `/set-password` | 设置密码 |
| `deleteUser()` | POST | `/delete-user` | 删除账户 |

### 验证方法

| 方法 | HTTP | 路径 | 说明 |
|------|------|------|------|
| `sendVerificationEmail()` | POST | `/send-verification-email` | 发送验证邮件 |
| `verifyEmail()` | POST | `/verify-email` | 验证邮箱 |
| `requestPasswordReset()` | POST | `/request-password-reset` | 请求重置密码 |
| `resetPassword()` | POST | `/reset-password` | 重置密码 |

### 账户方法

| 方法 | HTTP | 路径 | 说明 |
|------|------|------|------|
| `listUserAccounts()` | GET | `/list-user-accounts` | 查看关联账户 |
| `linkSocialAccount()` | POST | `/link-social-account` | 关联社交账户 |
| `unlinkAccount()` | POST | `/unlink-account` | 解绑账户 |
| `getAccessToken()` | GET | `/access-token` | 获取 OAuth token |

### 用户名插件方法

| 方法 | HTTP | 路径 | 说明 |
|------|------|------|------|
| `signIn.username()` | POST | `/sign-in/username` | 用户名登录 |
| `isUsernameAvailable()` | POST | `/is-username-available` | 检查用户名 |

---

## 13. 错误处理

### 错误结构

```tsx
result = {
  data: null,
  error: {
    message: "Invalid email or password",
    code: "INVALID_EMAIL_OR_PASSWORD",
    status: 401,
  },
};
```

### 常见错误码

| 错误码 | 说明 |
|--------|------|
| `INVALID_EMAIL_OR_PASSWORD` | 邮箱或密码错误 |
| `USER_ALREADY_EXISTS` | 用户已存在 |
| `EMAIL_NOT_VERIFIED` | 邮箱未验证 |
| `USERNAME_TOO_SHORT` | 用户名太短 |
| `USERNAME_TOO_LONG` | 用户名太长 |
| `INVALID_USERNAME` | 用户名格式错误 |
| `USERNAME_IS_ALREADY_TAKEN` | 用户名已被占用 |
| `PASSWORD_TOO_SHORT` | 密码太短 |
| `PASSWORD_TOO_LONG` | 密码太长 |
| `UNAUTHORIZED` | 未授权 |
| `FORBIDDEN` | 禁止访问 |

### 错误处理示例

```tsx
const result = await authClient.signIn.email({
  email: "user@example.com",
  password: "wrong-password",
});

if (result.error) {
  switch (result.error.code) {
    case "INVALID_EMAIL_OR_PASSWORD":
      toast.error("邮箱或密码错误");
      break;
    case "EMAIL_NOT_VERIFIED":
      toast.error("请先验证邮箱");
      // 可以重新发送验证邮件
      await authClient.sendVerificationEmail({ email });
      break;
    default:
      toast.error(result.error.message);
  }
}
```

---

## 参考资源

- [Better-Auth 官方文档](https://better-auth.com)
- [Better-Auth GitHub](https://github.com/better-auth/better-auth)
- [Better-Auth Next.js 指南](https://better-auth.com/docs/integrations/next)
- [Better-Auth 数据库适配器](https://better-auth.com/docs/adapters)
- [Username 插件文档](https://better-auth.com/docs/plugins/username)
- [Two Factor 插件文档](https://better-auth.com/docs/plugins/two-factor)
- [Admin 插件文档](https://better-auth.com/docs/plugins/admin)