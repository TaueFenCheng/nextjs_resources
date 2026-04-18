# Next.js App Router：page.tsx 与 layout.tsx 完整指南

本文档详细介绍 Next.js App Router 中 `page.tsx` 和 `layout.tsx` 的作用、嵌套规则、渲染机制以及组件类型。

---

## 目录

1. [基本概念](#一基本概念)
2. [page.tsx 详解](#二pagetsx-详解)
3. [layout.tsx 详解](#三layouttsx-详解)
4. [布局嵌套机制](#四布局嵌套机制)
5. [为什么有的目录没有 layout.tsx](#五为什么有的目录没有-layouttsx)
6. [渲染机制详解](#六渲染机制详解)
7. [组件类型：Server Component vs Client Component](#七组件类型-server-component-vs-client-component)
8. [实际项目示例](#八实际项目示例)
9. [常见问题](#九常见问题)

---

## 一、基本概念

### 1.1 Next.js App Router 的文件命名约定

Next.js App Router 使用特殊的文件名来定义路由和布局：

| 文件名 | 作用 | 必需性 |
|--------|------|--------|
| `page.tsx` | 定义路由页面（UI） | 必需（路由才能访问） |
| `layout.tsx` | 定义布局（共享 UI） | 可选（不必需） |
| `loading.tsx` | 加载状态 UI | 可选 |
| `error.tsx` | 错误处理 UI | 可选 |
| `not-found.tsx` | 404 页面 | 可选 |
| `route.ts` | API 路由处理器 | API 端点必需 |

### 1.2 核心规则

```
┌─────────────────────────────────────────────────────────────────┐
│                        核心规则                                  │
│                                                                 │
│  1. page.tsx 是路由必需文件                                      │
│     - 没有 page.tsx，该路径无法访问                              │
│     - page.tsx 定义该路径的 UI 内容                              │
│                                                                 │
│  2. layout.tsx 是可选文件                                        │
│     - 没有 layout.tsx，使用父级的 layout                         │
│     - layout.tsx 定义共享的 UI 结构                              │
│                                                                 │
│  3. 布局会嵌套                                                   │
│     - 子 layout 包裹在父 layout 中                               │
│     - 形成嵌套的 UI 结构                                         │
│                                                                 │
│  4. 默认都是 Server Component                                    │
│     - 不需要 "use client" 声明                                   │
│     - 可以使用 async/await                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、page.tsx 详解

### 2.1 什么是 page.tsx

`page.tsx` 是 Next.js App Router 的**页面组件文件**，定义特定路由路径的 UI 内容。

### 2.2 基本规则

```
文件夹路径                        →    URL 路径

app/page.tsx                     →    /
app/about/page.tsx               →    /about
app/workspace/page.tsx           →    /workspace
app/workspace/chats/page.tsx     →    /workspace/chats
app/workspace/chats/[id]/page.tsx →    /workspace/chats/{id}

必须规则：
- 每个可访问的路由必须有 page.tsx
- 没有 page.tsx 的文件夹只是"路由段"，不能直接访问
```

### 2.3 page.tsx 的作用

| 功能 | 说明 |
|------|------|
| **定义页面 UI** | 该路由路径显示的内容 |
| **接收参数** | `params`（动态路由参数）、`searchParams`（查询参数） |
| **数据获取** | 可直接 `async/await` 获取数据（Server Component） |
| **SEO 元数据** | 可通过 `generateMetadata` 定义页面元数据 |

### 2.4 page.tsx 示例

#### Server Component（默认）

```tsx
// app/workspace/chats/[thread_id]/page.tsx
// 默认是 Server Component，不需要 "use client"

interface PageProps {
  params: Promise<{ thread_id: string }>;
  searchParams: Promise<{ mock?: string }>;
}

export default async function ChatPage({ params, searchParams }: PageProps) {
  // ✅ 可以使用 async/await
  const { thread_id } = await params;
  const { mock } = await searchParams;
  
  // ✅ 可以直接获取数据
  const thread = await fetchThread(thread_id);
  
  return (
    <div>
      <h1>Thread: {thread_id}</h1>
      <ThreadMessages messages={thread.messages} />
    </div>
  );
}

// 可选：定义页面元数据
export async function generateMetadata({ params }: PageProps) {
  const { thread_id } = await params;
  return {
    title: `Thread ${thread_id} - DeerFlow`,
  };
}
```

#### Client Component（需要声明）

```tsx
// app/settings/page.tsx
"use client";  // 必须声明才能使用 hooks

import { useState } from "react";

export default function SettingsPage() {
  // ✅ 可以使用 React hooks
  const [theme, setTheme] = useState("light");
  
  return (
    <div>
      <h1>Settings</h1>
      <select value={theme} onChange={(e) => setTheme(e.target.value)}>
        <option value="light">Light</option>
        <option value="dark">Dark</option>
      </select>
    </div>
  );
}
```

### 2.5 page.tsx 的参数

```tsx
// app/products/[category]/[id]/page.tsx

interface PageProps {
  // 动态路由参数（Next.js 15+ 是 Promise）
  params: Promise<{
    category: string;  // 来自 [category]
    id: string;        // 来自 [id]
  }>;
  
  // URL 查询参数（Next.js 15+ 是 Promise）
  searchParams: Promise<{
    page?: string;     // ?page=1
    sort?: string;     // ?sort=desc
  }>;
}

export default async function ProductPage({ params, searchParams }: PageProps) {
  const { category, id } = await params;
  const { page, sort } = await searchParams;
  
  // URL: /products/electronics/123?page=2&sort=desc
  // category = "electronics"
  // id = "123"
  // page = "2"
  // sort = "desc"
}
```

---

## 三、layout.tsx 详解

### 3.1 什么是 layout.tsx

`layout.tsx` 是 Next.js App Router 的**布局组件文件**，定义路由段及其子路由共享的 UI 结构。

### 3.2 基本规则

```
layout.tsx 的特点：

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. 可选文件                                                    │
│     - 没有 layout.tsx，使用父级的 layout                         │
│     - 可以在任何路由段定义                                       │
│                                                                 │
│  2. 共享 UI                                                     │
│     - 该路由段及其所有子路由共享                                 │
│     - 子路由切换时，layout 不会重新渲染                          │
│                                                                 │
│  3. 嵌套结构                                                    │
│     - 子 layout 嵌套在父 layout 中                               │
│     - 形成多层包装                                              │
│                                                                 │
│  4. 必须接收 children                                           │
│     - layout 必须接收并渲染 children prop                        │
│     - children 是子页面/子布局的内容                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 layout.tsx 的作用

| 功能 | 说明 |
|------|------|
| **共享 UI 结构** | 导航栏、侧边栏、页脚等 |
| **状态保持** | 子路由切换时，layout 保持状态不重新渲染 |
| **数据预加载** | 可在 layout 中获取共享数据 |
| **嵌套包装** | 形成多层 UI 结构 |

### 3.4 layout.tsx 示例

#### 根布局（Root Layout）

```tsx
// app/layout.tsx
// 必须存在，是所有页面的最外层包装

import "@/styles/globals.css";

export const metadata = {
  title: "DeerFlow",
  description: "AI Agent Framework",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <ThemeProvider>
          <I18nProvider>
            {children}  {/* 必须渲染 children */}
          </I18nProvider>
        </ThemeProvider>
      </body>
    </html>
  );
}
```

#### 子布局

```tsx
// app/workspace/layout.tsx
// workspace 及其子路由的共享布局

import { Sidebar } from "@/components/sidebar";
import { Header } from "@/components/header";

export default function WorkspaceLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <div className="flex">
      <Sidebar />  {/* 侧边栏：子路由切换时不重新渲染 */}
      <main className="flex-1">
        <Header />  {/* 头部：子路由切换时不重新渲染 */}
        {children}  {/* 子页面内容 */}
      </main>
    </div>
  );
}
```

### 3.5 layout.tsx 的关键特性：不重新渲染

```
用户在 workspace 内切换路由：

/workspace/chats/thread-1  →  /workspace/chats/thread-2

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  RootLayout (app/layout.tsx)                                   │
│  └─ 不重新渲染，保持状态                                         │
│                                                                 │
│    WorkspaceLayout (app/workspace/layout.tsx)                  │
│    └─ 不重新渲染，保持状态                                       │
│    └─ Sidebar、Header 保持不变                                  │
│                                                                 │
│      ChatPage (app/workspace/chats/[id]/page.tsx)              │
│      └─ 重新渲染，显示新的 thread 内容                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

优点：
- Sidebar/Header 状态保持（如展开状态）
- 避免不必要的重新渲染
- 提升用户体验和性能
```

---

## 四、布局嵌套机制

### 4.1 嵌套规则

```
Next.js App Router 的布局嵌套规则：

1. 每个 layout.tsx 包裹其所在路由段及所有子路由
2. 子 layout 嵌套在父 layout 中
3. 最终渲染：根 layout → ... → 子 layout → page
```

### 4.2 嵌套示意图

```
目录结构：
app/
│
├── layout.tsx              ← 根布局（Level 0）
│   │
│   ├── page.tsx            ← 首页 (/)
│   │
│   └── [lang]/             ← 多语言路由段
│   │   ├── layout.tsx      ← 语言布局（Level 1）
│   │   │   │
│   │   │   └── docs/
│   │   │   │   ├── layout.tsx  ← 文档布局（Level 2）
│   │   │   │   │   │
│   │   │   │   │   └── page.tsx  ← 文档页面
│   │   │   │   │
│   │   │
│   │   │
│   │   └── workspace/      ← 工作区路由段
│   │   │   ├── layout.tsx  ← 工作区布局（Level 2）
│   │   │   │   │
│   │   │   │   ├── page.tsx    ← 工作区入口
│   │   │   │   │
│   │   │   │   └── chats/
│   │   │   │   │   ├── layout.tsx  ← 会话布局（Level 3）
│   │   │   │   │   │   │
│   │   │   │   │   │   └── [id]/
│   │   │   │   │   │   │   └── page.tsx  ← 会话详情页
```

### 4.3 渲染嵌套顺序

访问 `/workspace/chats/demo-123` 时：

```
渲染顺序（从外到内）：

┌─────────────────────────────────────────────────────────────────┐
│  Level 0: RootLayout (app/layout.tsx)                           │
│  ├─ <html>、<body>                                              │
│  ├─ ThemeProvider、I18nProvider                                 │
│  └─ children                                                    │
│    │                                                            │
│    ▼                                                            │
│  Level 2: WorkspaceLayout (app/workspace/layout.tsx)            │
│  ├─ Sidebar、Header                                             │
│  └─ children                                                    │
│    │                                                            │
│    ▼                                                            │
│  Level 3: ChatsLayout (app/workspace/chats/layout.tsx)          │
│  ├─ 会话相关共享 UI                                             │
│  └─ children                                                    │
│    │                                                            │
│    ▼                                                            │
│  Page: ChatPage (app/workspace/chats/[id]/page.tsx)             │
│  ├─ 会话详情内容                                                │
│  └─ 消息列表、输入框等                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 实际渲染结果

```tsx
// 最终渲染的组件树：

<RootLayout>
  <html lang="en">
    <body>
      <ThemeProvider>
        <I18nProvider>
          <WorkspaceLayout>
            <Sidebar />
            <main>
              <Header />
              <ChatsLayout>
                {/* 会话共享 UI */}
                <ChatPage>
                  {/* 会话详情内容 */}
                </ChatPage>
              </ChatsLayout>
            </main>
          </WorkspaceLayout>
        </I18nProvider>
      </ThemeProvider>
    </body>
  </html>
</RootLayout>
```

---

## 五、为什么有的目录没有 layout.tsx

### 5.1 没有 layout.tsx 的原因

```
┌─────────────────────────────────────────────────────────────────┐
│                     原因分析                                     │
│                                                                 │
│  1. 不需要共享 UI                                               │
│     - 该路由段不需要特殊的布局结构                               │
│     - 直接使用父级的 layout 即可                                 │
│                                                                 │
│  2. 只作为路由段                                                │
│     - 文件夹只是为了组织路由结构                                 │
│     - 本身不渲染，只是路径的一部分                               │
│                                                                 │
│  3. 继承父 layout                                               │
│     - 没有子 layout，children 直接传给父 layout                  │
│     - 减少代码重复                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 没有 layout.tsx 时的渲染机制

#### 示例目录结构

```
app/
│
├── layout.tsx              ← 根布局（必需）
│   │
│   └── workspace/
│   │   ├── layout.tsx      ← 工作区布局（有）
│   │   │   │
│   │   │   ├── page.tsx    ← 工作区入口页
│   │   │   │
│   │   │   └── chats/
│   │   │   │   ├── layout.tsx      ← 会话布局（有）
│   │   │   │   │   │
│   │   │   │   │   └── [id]/
│   │   │   │   │   │   │
│   │   │   │   │   │   └── page.tsx  ← 会话详情页
│   │   │   │   │   │   │
│   │   │   │   │   │   └── agents/   ← 没有 layout.tsx！
│   │   │   │   │   │   │   └── page.tsx  ← Agent 页面
```

#### 有 layout.tsx 的渲染

访问 `/workspace/chats/demo-123`：

```
RootLayout
    │
    │ children
    │
    ▼
WorkspaceLayout
    │
    │ children
    │
    ▼
ChatsLayout
    │
    │ children
    │
    ▼
ChatPage  ← 最终页面内容
```

#### 没有 layout.tsx 的渲染

访问 `/workspace/chats/agents`（假设 agents 没有 layout.tsx）：

```
RootLayout
    │
    │ children
    │
    ▼
WorkspaceLayout
    │
    │ children
    │
    ▼
ChatsLayout
    │
    │ children（直接传给 ChatsLayout，没有中间层）
    │
    ▼
AgentsPage  ← 直接渲染，跳过 agents layout
```

### 5.3 渲染机制详解

```
渲染规则：

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  有 layout.tsx：                                                │
│                                                                 │
│  父 layout                                                      │
│      ↓ children                                                 │
│  子 layout                                                      │
│      ↓ children                                                 │
│  page                                                           │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  没有 layout.tsx：                                              │
│                                                                 │
│  父 layout                                                      │
│      ↓ children                                                 │
│  page（直接渲染，跳过中间 layout）                               │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  关键点：                                                        │
│  - children prop 会"穿透"没有 layout 的目录                      │
│  - 直接传给最近的父 layout                                       │
│  - 不需要占位 layout                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.4 实际项目示例

```tsx
// DeerFlow 项目的布局结构：

app/
│
├── layout.tsx              ← 有（根布局）
│   │
│   └── [lang]/             ← 没有 layout.tsx！
│   │   │   │
│   │   │   └── docs/
│   │   │   │   ├── layout.tsx  ← 有（文档布局）
│   │   │   │   │   │
│   │   │   │   │   └── page.tsx
│   │   │
│   │   └── workspace/
│   │   │   ├── layout.tsx  ← 有（工作区布局）
│   │   │   │   │
│   │   │   │   ├── page.tsx
│   │   │   │   │
│   │   │   │   └── chats/
│   │   │   │   │   │
│   │   │   │   │   └── [id]/
│   │   │   │   │   │   │
│   │   │   │   │   │   └── page.tsx  ← 没有 chats/layout.tsx！
│   │   │   │   │   │
│   │   │   │   │   └── agents/
│   │   │   │   │   │   └── [name]/
│   │   │   │   │   │   │   └── chats/
│   │   │   │   │   │   │   │   └── [id]/
│   │   │   │   │   │   │   │   │   └── page.tsx  ← 多层没有 layout！

// 访问 /workspace/chats/demo-123 的渲染：
RootLayout
    ↓ children
WorkspaceLayout（直接渲染 children）
    ↓ children
ChatPage（没有中间的 chats/layout）
```

---

## 六、渲染机制详解

### 6.1 渲染流程图

```
浏览器请求: /workspace/chats/demo-123
                    │
                    │ Next.js 路由匹配
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    路由段识别                                    │
│                                                                 │
│  路由段：                                                        │
│  ├─ /                → app/layout.tsx                          │
│  ├─ workspace        → app/workspace/layout.tsx                │
│  ├─ chats            → 无 layout.tsx（跳过）                    │
│  └─ [id]=demo-123    → app/workspace/chats/[id]/page.tsx       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                    │
                    │ 构建组件树
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    组件树构建                                    │
│                                                                 │
│  Step 1: 找到根 layout                                          │
│          app/layout.tsx                                         │
│                                                                 │
│  Step 2: 找到 workspace layout                                  │
│          app/workspace/layout.tsx                               │
│          嵌套在根 layout 的 children 中                          │
│                                                                 │
│  Step 3: chats 目录没有 layout                                  │
│          children 直接传给 workspace layout                      │
│          跳过这一层                                              │
│                                                                 │
│  Step 4: 找到 page                                              │
│          app/workspace/chats/[id]/page.tsx                      │
│          作为最终的 children                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                    │
                    │ 渲染
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    最终渲染结果                                  │
│                                                                 │
│  <RootLayout>                                                   │
│    <WorkspaceLayout>                                            │
│      <ChatPage />                                               │
│    </WorkspaceLayout>                                           │
│  </RootLayout>                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 children 的传递过程

```tsx
// 没有 layout.tsx 时，children 的传递：

// app/layout.tsx
export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {children}  // ← 这里接收的是什么？
      </body>
    </html>
  );
}

// 没有 [lang]/layout.tsx
// 没有 workspace/layout.tsx 时
// children 直接是 WorkspacePage 或 ChatPage

// 有 workspace/layout.tsx 时
export default function WorkspaceLayout({ children }) {
  return (
    <div>
      <Sidebar />
      {children}  // ← 这里接收的是 page.tsx 的内容
    </div>
  );
}

// 没有 chats/layout.tsx
// children 直接是 ChatPage
```

---

## 七、组件类型：Server Component vs Client Component

### 7.1 默认都是 Server Component

```
┌─────────────────────────────────────────────────────────────────┐
│                     默认规则                                     │
│                                                                 │
│  page.tsx 和 layout.tsx 默认都是 Server Component（RSC）        │
│                                                                 │
│  不需要声明：                                                    │
│  - 没有 "use client" 声明                                       │
│  - Next.js 自动识别为 Server Component                          │
│                                                                 │
│  Server Component 能做什么：                                    │
│  ✅ 使用 async/await                                            │
│  ✅ 直接访问数据库/文件系统                                      │
│  ✅ 使用服务端环境变量                                           │
│  ✅ SEO 友好（服务端渲染 HTML）                                  │
│                                                                 │
│  Server Component 不能做什么：                                  │
│  ❌ 使用 React hooks（useState、useEffect）                     │
│  ❌ 使用事件处理（onClick、onChange）                           │
│  ❌ 使用浏览器 API（window、localStorage）                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 page.tsx 和 layout.tsx 的类型选择

| 场景 | page.tsx 类型 | layout.tsx 类型 | 说明 |
|------|--------------|----------------|------|
| **静态内容展示** | Server Component | Server Component | 默认，无需声明 |
| **需要获取数据** | Server Component（async） | Server Component | 使用 async/await |
| **需要交互** | Client Component | Server Component | page 加 "use client" |
| **需要 hooks** | Client Component | Server Component | page 加 "use client" |
| **layout 需要交互** | - | Client Component | layout 加 "use client" |

### 7.3 Server Component 示例（默认）

```tsx
// app/workspace/page.tsx
// 默认是 Server Component，不需要 "use client"

import fs from "fs";
import path from "path";
import { redirect } from "next/navigation";

export default function WorkspacePage() {
  // ✅ 可以使用 fs（Node.js API）
  // ✅ 可以使用 redirect（服务端导航）
  
  if (process.env.STATIC_MODE === "true") {
    const firstThread = fs.readdirSync("public/demo/threads")[0];
    return redirect(`/workspace/chats/${firstThread}`);
  }
  
  return redirect("/workspace/chats/new");
}
```

```tsx
// app/layout.tsx
// 默认是 Server Component

export const metadata = {
  title: "DeerFlow",
  description: "AI Agent Framework",
};

export default function RootLayout({ children }) {
  // ✅ 可以定义 metadata（SEO）
  // ✅ 纯服务端渲染
  
  return (
    <html lang="en">
      <body>
        <ThemeProvider>  {/* Client Component 子组件 */}
          {children}
        </ThemeProvider>
      </body>
    </html>
  );
}
```

### 7.4 Client Component 示例

```tsx
// app/workspace/chats/[id]/page.tsx
// 需要 "use client" 才能用 hooks

"use client";

import { useState, useEffect } from "react";
import { useParams } from "next/navigation";

export default function ChatPage() {
  // ✅ 可以使用 hooks
  const [messages, setMessages] = useState([]);
  
  // ✅ 可以使用 useParams
  const params = useParams();
  const threadId = params.id;
  
  // ✅ 可以使用 useEffect
  useEffect(() => {
    fetch(`/api/threads/${threadId}`)
      .then(res => res.json())
      .then(data => setMessages(data.messages));
  }, [threadId]);
  
  return <div>{messages.map(...)}</div>;
}
```

### 7.5 layout.tsx 作为 Client Component

```tsx
// app/workspace/layout.tsx
// 声明为 Client Component

"use client";

import { useState } from "react";

export default function WorkspaceLayout({ children }) {
  // ✅ layout 也可以是 Client Component
  // ✅ 可以使用 hooks
  
  const [sidebarOpen, setSidebarOpen] = useState(true);
  
  return (
    <div className="flex">
      <aside className={sidebarOpen ? "w-64" : "w-0"}>
        {/* Sidebar */}
      </aside>
      <button onClick={() => setSidebarOpen(!sidebarOpen)}>
        Toggle Sidebar
      </button>
      <main>{children}</main>
    </div>
  );
}
```

### 7.6 Server + Client 组合模式

**最佳实践**：layout 保持 Server Component，需要交互的部分提取为 Client Component。

```tsx
// app/workspace/layout.tsx
// Server Component（默认）

import { Sidebar } from "@/components/sidebar";  // Client Component

export default function WorkspaceLayout({ children }) {
  // Server Component 可以导入 Client Component
  // 交互逻辑在 Client Component 中处理
  
  return (
    <div className="flex">
      <Sidebar />  {/* Client Component，处理交互 */}
      <main>{children}</main>
    </div>
  );
}

// components/sidebar.tsx
// Client Component

"use client";

import { useState } from "react";

export function Sidebar() {
  const [expanded, setExpanded] = useState(true);
  
  return (
    <aside>
      <button onClick={() => setExpanded(!expanded)}>
        Toggle
      </button>
      {/* ... */}
    </aside>
  );
}
```

### 7.7 类型总结表

| 文件 | 默认类型 | 可以改成 Client？ | 常见选择 |
|------|---------|------------------|---------|
| **layout.tsx** | Server Component | ✅ 可以（加 "use client"） | 通常保持 Server |
| **page.tsx** | Server Component | ✅ 可以（加 "use client"） | 根据需求选择 |

---

## 八、实际项目示例

### 8.1 DeerFlow 项目的布局结构

```
frontend/src/app/
│
├── layout.tsx                    ← Server Component
│   │                             根布局：ThemeProvider、I18nProvider
│   │
│   ├── page.tsx                  ← Server Component
│   │                             首页：静态展示
│   │
│   ├── [lang]/                   ← 没有 layout.tsx
│   │   │                         路由段：多语言参数
│   │   │
│   │   └── docs/
│   │   │   ├── layout.tsx        ← Server Component
│   │   │   │                     文档布局：Nextra 主题
│   │   │   │   │
│   │   │   │   └── [[...path]]/page.tsx  ← Server Component
│   │   │   │                     文档页面
│   │   │
│   │   └── workspace/
│   │   │   ├── layout.tsx        ← Server Component
│   │   │   │                     工作区布局：Sidebar、Header
│   │   │   │   │
│   │   │   │   ├── page.tsx      ← Server Component
│   │   │   │   │                 工作区入口：重定向
│   │   │   │   │
│   │   │   │   └── chats/
│   │   │   │   │   │             ← 没有 layout.tsx
│   │   │   │   │   │
│   │   │   │   │   └── [thread_id]/
│   │   │   │   │   │   └── page.tsx  ← Client Component
│   │   │   │   │   │             "use client"
│   │   │   │   │   │             会话详情：交互逻辑
│   │   │   │   │   │
│   │   │   │   └── agents/
│   │   │   │   │   │             ← 没有 layout.tsx
│   │   │   │   │   │
│   │   │   │   │   └── [agent_name]/
│   │   │   │   │   │   │
│   │   │   │   │   │   └── chats/
│   │   │   │   │   │   │   │
│   │   │   │   │   │   │   └── [thread_id]/
│   │   │   │   │   │   │   │   └── page.tsx  ← Client Component
│   │   │   │   │   │   │   │
│   │   │   │
│   │   └── blog/
│   │   │   ├── layout.tsx        ← Server Component
│   │   │   │                     博客布局
│   │   │   │   │
│   │   │   │   └── posts/
│   │   │   │   │   └── page.tsx  ← Server Component
│   │   │   │   │
│   │   │   │   └── [slug]/
│   │   │   │   │   └── page.tsx  ← Server Component
│
├── api/                          ← API Routes（不是组件）
│   ├── auth/
│   ├── memory/
│   └── threads/
│
└── mock/                         ← Mock API（不是组件）
```

### 8.2 渲染示例

访问 `/workspace/chats/demo-123?mock=true`：

```tsx
// 最终渲染的组件树：

// RootLayout (Server Component)
<RootLayout>
  <html lang="zh">
    <body>
      <ThemeProvider>
        <I18nProvider>
          
          // WorkspaceLayout (Server Component)
          <WorkspaceLayout>
            <Sidebar />
            <main>
              <Header />
              
              // 没有 chats/layout.tsx，直接渲染 page
              
              // ChatPage (Client Component)
              <ChatPage>
                {/* 交互逻辑：hooks、事件处理 */}
                <ThreadMessages />
                <InputBox />
                <Artifacts />
              </ChatPage>
              
            </main>
          </WorkspaceLayout>
          
        </I18nProvider>
      </ThemeProvider>
    </body>
  </html>
</RootLayout>
```

---

## 九、常见问题

### 9.1 必须有 layout.tsx 吗？

**答：不是。**

- `app/layout.tsx`（根布局）是**必需**的
- 其他路由段的 `layout.tsx` 是**可选**的
- 没有 `layout.tsx` 时，使用父级的 layout

### 9.2 必须有 page.tsx 吗？

**答：要让路由可访问，必须有。**

- 没有 `page.tsx`，该路径无法访问
- 只有 `layout.tsx` 没有 `page.tsx` → 404
- `route.ts` 和 `page.tsx` 不能同时存在（冲突）

### 9.3 page.tsx 和 layout.tsx 都可以是 async 吗？

**答：都可以。**

```tsx
// Server Component 默认可以是 async
export default async function Page() {
  const data = await fetchData();
  return <div>{data}</div>;
}

export default async function Layout({ children }) {
  const sharedData = await fetchSharedData();
  return <div data={sharedData}>{children}</div>;
}
```

### 9.4 layout 中的 children 是什么类型？

**答：React.ReactNode，是已渲染的 React 元素。**

```tsx
// children 是子组件的渲染结果
// 不是 Promise，不是函数，是 React 元素

export default function Layout({ children }: { children: React.ReactNode }) {
  return <div>{children}</div>;
}

// children 来自：
// - 子 layout 的渲染结果
// - 或直接是 page 的渲染结果（没有中间 layout 时）
```

### 9.5 layout 切换子路由时会重新渲染吗？

**答：不会。**

- 子路由切换时，layout 保持状态
- 只有 `page.tsx` 重新渲染
- 这是 layout 的核心优势

```
/workspace/chats/1  →  /workspace/chats/2

RootLayout:    不重新渲染
WorkspaceLayout: 不重新渲染（Sidebar 状态保持）
ChatPage:      重新渲染（显示新 thread）
```

### 9.6 可以在 layout 中获取 params 吗？

**答：可以。**

```tsx
// app/[lang]/layout.tsx

export default async function LangLayout({
  children,
  params,
}: {
  children: React.ReactNode;
  params: Promise<{ lang: string }>;
}) {
  const { lang } = await params;
  
  return (
    <div lang={lang}>
      {children}
    </div>
  );
}
```

### 9.7 page 和 layout 能同时用 "use client" 吗？

**答：可以，但不推荐。**

- 都用 "use client" → 都变成 Client Component
- 推荐：layout 保持 Server，交互逻辑提取到 Client Component 子组件

---

## 十、总结

### 10.1 核心概念表

| 文件 | 作用 | 必需性 | 默认类型 | 关键特性 |
|------|------|--------|---------|---------|
| **page.tsx** | 定义页面 UI | 路由必需 | Server Component | async/await、params |
| **layout.tsx** | 定义共享布局 | 可选（根 layout 必需） | Server Component | 不重新渲染、嵌套 |

### 10.2 一句话总结

> **page.tsx** = 路由页面内容，必需文件，定义该路径的 UI
>
> **layout.tsx** = 共享布局结构，可选文件，子路由切换时不重新渲染
>
> **没有 layout.tsx** = 使用父级 layout，children 直接传递
>
> **默认类型** = 都是 Server Component（RSC），可以 async/await
>
> **改成 Client Component** = 加 "use client"，可以使用 hooks 和事件处理

---

## 十一、参考资料

- [Next.js App Router Documentation](https://nextjs.org/docs/app)
- [Layouts and Pages](https://nextjs.org/docs/app/building-your-application/routing/layouts-and-pages)
- [Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components)
- [Client Components](https://nextjs.org/docs/app/building-your-application/rendering/client-components)