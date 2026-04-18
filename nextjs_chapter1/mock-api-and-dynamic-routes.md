# Next.js Mock API 与动态路由命名详解

本文档详细介绍 Mock API 的作用、动态路由命名规则（`[param]`、`[[...param]]`）以及 route.ts 的使用方法。

---

## 目录

1. [Mock API 概述](#一mock-api-概述)
2. [Mock API 目录结构](#二mock-api-目录结构)
3. [mock/api 与 api 的区别](#三mockapi-与-api-的区别)
4. [为什么 Mock API 可以用 Node.js API](#四为什么-mock-api-可以用-nodejs-api)
5. [动态路由命名规则](#五动态路由命名规则)
6. [路由匹配优先级](#六路由匹配优先级)
7. [route.ts 文件详解](#七routets-文件详解)
8. [前端如何调用这些 API](#八前端如何调用这些-api)
9. [完整示例](#九完整示例)

---

## 一、Mock API 概述

### 1.1 什么是 Mock API

Mock API 是**模拟后端 API** 的系统，用于在前端开发时**无需真实后端服务**即可运行。

### 1.2 Mock API 的作用

| 场景 | 作用 |
|------|------|
| **本地开发** | 后端服务未启动时，前端也能运行和测试 |
| **静态演示网站** | 部署到 Vercel 等平台时不需要后端服务 |
| **E2E 测试** | Playwright 测试可以拦截真实 API，使用模拟数据 |
| **Demo 模式** | 展示功能给用户，无需完整后端环境 |
| **前端独立开发** | 前端开发人员可以不依赖后端进度 |

### 1.3 数据来源

```
真实 API 数据来源：
┌─────────────────┐
│   后端服务       │  ← Gateway API (8001)
│  (数据库/逻辑)   │  ← LangGraph Server (2024)
└─────────────────┘

Mock API 数据来源：
┌─────────────────┐
│   本地文件       │  ← public/demo/xxx.json
│  (硬编码数据)    │  ← route.ts 中直接返回
└─────────────────┘
```

---

## 二、Mock API 目录结构

### 2.1 完整目录树

```
frontend/src/app/mock/
│
└── api/                                    ← Mock API 根目录
    │
    ├── models/
    │   └── route.ts                        → URL: /mock/api/models
    │
    ├── skills/
    │   └── route.ts                        → URL: /mock/api/skills
    │
    ├── mcp/
    │   └── config/
    │       └── route.ts                    → URL: /mock/api/mcp/config
    │
    └── threads/
        │
        ├── search/
        │   └── route.ts                    → URL: /mock/api/threads/search
        │
        └── [thread_id]/                    ← 动态路由参数
            │
            ├── history/
            │   └── route.ts                → URL: /mock/api/threads/{id}/history
            │
            └── artifacts/
                └── [[...artifact_path]]/    ← 可选捕获所有路由
                    └── route.ts            → URL: /mock/api/threads/{id}/artifacts/{...}
```

### 2.2 各文件作用

| 文件 | URL | 作用 |
|------|-----|------|
| `models/route.ts` | `/mock/api/models` | 返回模拟的可用模型列表 |
| `skills/route.ts` | `/mock/api/skills` | 返回模拟的技能列表 |
| `mcp/config/route.ts` | `/mock/api/mcp/config` | 返回模拟的 MCP 服务器配置 |
| `threads/search/route.ts` | `/mock/api/threads/search` | 搜索会话（返回本地数据） |
| `threads/[thread_id]/history/route.ts` | `/mock/api/threads/{id}/history` | 获取指定会话的历史记录 |
| `threads/[thread_id]/artifacts/[[...artifact_path]]/route.ts` | `/mock/api/threads/{id}/artifacts/{path}` | 获取会话中的产物文件 |

### 2.3 Mock 数据存储位置

```
public/demo/
│
└── threads/
    │
    ├── demo-123/
    │   ├── thread.json         ← 会话元数据
    │   └── artifacts/
    │       └── output.txt      ← 产物文件
    │
    ├── demo-456/
    │   └── thread.json
    │
    └── ...
```

---

## 三、mock/api 与 api 的区别

### 3.1 核心差异对比

| 维度 | 真实 API (`app/api/`) | Mock API (`app/mock/api/`) |
|------|----------------------|---------------------------|
| **URL 路径** | `/api/xxx` | `/mock/api/xxx` |
| **运行位置** | Node.js 服务端 | Node.js 服务端 |
| **能否用 fs/path** | ✅ 可以 | ✅ 可以 |
| **数据来源** | 转发到后端服务（Gateway/LangGraph） | 本地 JSON 文件或硬编码 |
| **需要后端** | ✅ 需要启动后端服务 | ❌ 不需要后端 |
| **用途** | 生产环境、真实数据 | 开发、Demo、测试 |
| **响应速度** | 取决于后端服务 | 极快（本地文件） |

### 3.2 URL 对比

```
真实 API URL：
/api/models                    → 请求 Gateway API (8001)
/api/memory                    → 请求 Gateway API (8001)
/api/langgraph/threads         → 请求 LangGraph Server (2024)

Mock API URL：
/mock/api/models               → 返回硬编码数据
/mock/api/threads/demo/history → 返回本地 JSON 文件
```

### 3.3 代码对比

#### 真实 API（代理请求）

```typescript
// app/api/memory/[...path]/route.ts
const BACKEND_BASE_URL = "http://127.0.0.1:8001";

export async function GET(request, { params }) {
  // 代理请求到后端服务
  const response = await fetch(
    `${BACKEND_BASE_URL}/api/memory/${params.path.join("/")}`
  );
  return new Response(await response.arrayBuffer(), {
    status: response.status,
    headers: response.headers,
  });
}
```

#### Mock API（本地数据）

```typescript
// app/mock/api/threads/[thread_id]/history/route.ts
import fs from "fs";
import path from "path";

export async function POST(request, { params }) {
  const threadId = params.thread_id;
  
  // 直接读取本地 JSON 文件
  const jsonString = fs.readFileSync(
    path.resolve(process.cwd(), `public/demo/threads/${threadId}/thread.json`),
    "utf8"
  );
  
  return Response.json(JSON.parse(jsonString));
}
```

---

## 四、为什么 Mock API 可以用 Node.js API

### 4.1 关键概念

```
所有 route.ts 文件（包括 api/ 和 mock/api/）都在 Node.js 服务端运行！

不是在浏览器运行！
```

### 4.2 运行位置对比表

| 文件类型 | 运行位置 | 能否用 Node.js API |
|---------|---------|-------------------|
| `route.ts`（任何位置） | **Node.js 服务端** | ✅ 可以用 fs、path、process 等 |
| `page.tsx`（无 "use client"） | Node.js 服务端 | ✅ 可以用 fs（Server Component） |
| `layout.tsx`（无 "use client"） | Node.js 服务端 | ✅ 可以用 fs |
| `components/*.tsx`（有 "use client"） | 浏览器 | ❌ 不能用 fs |
| `hooks/*.ts`（被 Client 组件导入） | 浏览器 | ❌ 不能用 fs |

### 4.3 为什么设计成这样？

Next.js App Router 的设计：

```
┌─────────────────────────────────────────────────────┐
│                  Next.js 服务端                      │
│                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │  page.tsx   │  │ layout.tsx  │  │  route.ts   │ │
│  │ (Server)    │  │ (Server)    │  │ (API Route) │ │
│  │             │  │             │  │             │ │
│  │ 可用 fs ✅  │  │ 可用 fs ✅  │  │ 可用 fs ✅  │ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
│                                                     │
│         ↕ HTTP 请求/响应                            │
│                                                     │
└─────────────────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────────────────┐
│                    浏览器                            │
│                                                     │
│  ┌─────────────┐  ┌─────────────┐                  │
│  │ Component   │  │   hooks     │                  │
│  │ ("use client")│ │             │                  │
│  │             │  │             │                  │
│  │ 不能用 fs ❌│  │ 不能用 fs ❌│                  │
│  └─────────────┘  └─────────────┘                  │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 4.4 实际例子

```typescript
// app/mock/api/threads/[thread_id]/history/route.ts
import fs from "fs";        // ✅ Node.js 文件系统模块
import path from "path";    // ✅ Node.js 路径模块

export async function POST(request, { params }) {
  // process.cwd() — Node.js API，获取当前工作目录
  const cwd = process.cwd();
  
  // path.resolve() — Node.js API，解析路径
  const filePath = path.resolve(cwd, `public/demo/threads/${threadId}/thread.json`);
  
  // fs.readFileSync() — Node.js API，读取文件
  const content = fs.readFileSync(filePath, "utf8");
  
  // 所有这些 Node.js API 都可用！
  return Response.json(JSON.parse(content));
}
```

---

## 五、动态路由命名规则

### 5.1 三种路由类型总览

| 类型 | 文件夹命名 | 匹配规则 | 参数类型 | 示例 |
|------|-----------|---------|---------|------|
| **静态路由** | `folder` | 精确匹配固定字符串 | 无参数 | `threads/search` |
| **动态路由** | `[param]` | 匹配**单个**路径段 | `string` | `threads/[thread_id]` |
| **捕获所有路由** | `[...param]` | 匹配**至少一个**路径段 | `string[]` | `[...path]` |
| **可选捕获所有** | `[[...param]]` | 匹配**零个或多个**路径段 | `string[]` 或 `undefined` | `[[...artifact_path]]` |

### 5.2 `[thread_id]` — 动态路由（单层）

#### 5.2.1 概念

**匹配单个路径段**，用方括号包裹参数名。

#### 5.2.2 匹配规则

```
文件夹名：[thread_id]

匹配 URL：
  /api/threads/abc123         → thread_id = "abc123"
  /api/threads/demo-test      → thread_id = "demo-test"
  /api/threads/12345          → thread_id = "12345"
  /api/threads/user-session   → thread_id = "user-session"

不匹配：
  /api/threads                → ❌ 缺少路径段
  /api/threads/abc/def        → ❌ 路径段太多（只匹配一个）
```

#### 5.2.3 参数获取

```typescript
// route.ts
export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ thread_id: string }> }
) {
  const threadId = (await params).thread_id;
  // URL: /api/threads/abc123
  // threadId = "abc123"
  
  console.log(threadId);  // "abc123"
}
```

#### 5.2.4 实际使用场景

```
app/api/threads/[thread_id]/history/route.ts

匹配 URL：
  POST /api/threads/demo-123/history
  POST /api/threads/session-456/history
  
用途：获取指定 thread 的历史记录
```

### 5.3 `[[...artifact_path]]` — 可选捕获所有路由

#### 5.3.1 概念

**双括号表示可选**，**三个点表示捕获所有剩余路径**。

#### 5.3.2 与单括号的区别

```
单括号 [...path]：
  必须匹配至少一个路径段（不能为空）
  
双括号 [[...path]]：
  可以匹配零个或多个路径段（可以为空）
```

#### 5.3.3 匹配规则

```
文件夹名：[[...artifact_path]]

匹配 URL：
  /api/threads/abc/artifacts                     → artifact_path = undefined ✅
  /api/threads/abc/artifacts/file.txt            → artifact_path = ["file.txt"] ✅
  /api/threads/abc/artifacts/output.txt          → artifact_path = ["output.txt"] ✅
  /api/threads/abc/artifacts/mnt/output.txt      → artifact_path = ["mnt", "output.txt"] ✅
  /api/threads/abc/artifacts/folder/a/b/c.txt    → artifact_path = ["folder", "a", "b", "c.txt"] ✅

全部都能匹配！（零个、一个、多个路径段都可以）
```

#### 5.3.4 参数获取

```typescript
// route.ts
export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{
    thread_id: string;           // 来自外层的 [thread_id]
    artifact_path?: string[];    // 来自 [[...artifact_path]]（可选！）
  }> }
) {
  const { thread_id, artifact_path } = await params;
  
  // URL: /api/threads/abc/artifacts
  // thread_id = "abc"
  // artifact_path = undefined
  
  // URL: /api/threads/abc/artifacts/mnt/output.txt
  // thread_id = "abc"
  // artifact_path = ["mnt", "output.txt"]
  
  // 将数组转为路径字符串
  const filePath = artifact_path?.join("/") ?? "";
  // "mnt/output.txt" 或 ""（空字符串）
}
```

#### 5.3.5 实际使用场景

```
app/api/threads/[thread_id]/artifacts/[[...artifact_path]]/route.ts

匹配 URL：
  GET /api/threads/demo/artifacts                  ← 获取 artifacts 根目录信息
  GET /api/threads/demo/artifacts/file.txt         ← 获取单个文件
  GET /api/threads/demo/artifacts/mnt/output.txt   ← 获取嵌套路径的文件
  GET /api/threads/demo/artifacts/dir/subdir/file  ← 获取深层嵌套文件

用途：获取 thread 中任意路径的产物文件
```

### 5.4 对比三种动态路由

```
文件夹命名              匹配 URL                              参数值

[thread_id]            
                       /api/threads/abc                     thread_id = "abc"
                       /api/threads/xyz                     thread_id = "xyz"
                       
                       /api/threads                         ❌ 不匹配（必须有值）
                       /api/threads/abc/def                 ❌ 不匹配（只匹配一个）

[...path]
                       /api/files/a                        path = ["a"]
                       /api/files/a/b                      path = ["a", "b"]
                       /api/files/a/b/c/d                  path = ["a", "b", "c", "d"]
                       
                       /api/files                          ❌ 不匹配（至少一个）

[[...path]]
                       /api/artifacts                      path = undefined ✅
                       /api/artifacts/a                    path = ["a"] ✅
                       /api/artifacts/a/b/c                path = ["a", "b", "c"] ✅
                       
                       全部都能匹配！
```

### 5.5 嵌套动态路由

```
文件夹结构：
app/api/threads/[thread_id]/artifacts/[[...artifact_path]]/route.ts

URL 示例：
/api/threads/demo-123/artifacts/mnt/output.txt

参数解析：
├── threads/           ← 静态路径
├── [thread_id]/       ← 动态参数 → "demo-123"
├── artifacts/         ← 静态路径
└── [[...artifact_path]]/ ← 可选捕获所有 → ["mnt", "output.txt"]

最终 params：
{
  thread_id: "demo-123",
  artifact_path: ["mnt", "output.txt"]
}
```

---

## 六、路由匹配优先级

### 6.1 优先级规则

```
优先级从高到低：

1. 预定义静态路由（最高优先级）
   ↓
2. 动态路由 [param]
   ↓
3. 捕获所有路由 [...param]
   ↓
4. 可选捕获所有路由 [[...param]]（最低优先级）
```

### 6.2 具体示例

```
app/api/threads/
│
├── search/
│   └── route.ts              ← 静态路由（优先级最高）
│
└── [thread_id]/              ← 动态路由（优先级第二）
    ├── route.ts
    └── history/
        └── route.ts
```

**匹配结果**：

```
URL: /api/threads/search
     → 匹配 search/route.ts ✅（静态路由优先）

URL: /api/threads/demo-123
     → 不匹配 search（静态不匹配）
     → 匹配 [thread_id]/route.ts ✅（动态路由）

URL: /api/threads/demo-123/history
     → 匹配 [thread_id]/history/route.ts ✅
```

### 6.3 优先级图解

```
请求: GET /api/threads/search
                    │
                    ▼
        ┌───────────────────────┐
        │ 检查静态路由          │
        │ threads/search/route.ts │
        └───────────────────────┘
                    │
                    │ 匹配！
                    ▼
        ┌───────────────────────┐
        │ 执行 search/route.ts  │
        │ 中的 GET() 函数        │
        └───────────────────────┘


请求: GET /api/threads/abc123
                    │
                    ▼
        ┌───────────────────────┐
        │ 检查静态路由          │
        │ threads/search/       │
        └───────────────────────┘
                    │
                    │ 不匹配
                    ▼
        ┌───────────────────────┐
        │ 检查动态路由          │
        │ threads/[thread_id]/  │
        └───────────────────────┘
                    │
                    │ 匹配！thread_id = "abc123"
                    ▼
        ┌───────────────────────┐
        │ 执行 [thread_id]/route.ts │
        │ 中的 GET() 函数        │
        └───────────────────────┘
```

---

## 七、route.ts 文件详解

### 7.1 固定的函数名

Next.js App Router **强制要求**导出特定名称的函数：

```typescript
// route.ts 中可导出的函数名

export async function GET(request) { }     // 处理 HTTP GET 请求
export async function POST(request) { }    // 处理 HTTP POST 请求
export async function PUT(request) { }     // 处理 HTTP PUT 请求
export async function DELETE(request) { }  // 处理 HTTP DELETE 请求
export async function PATCH(request) { }   // 处理 HTTP PATCH 请求
export async function HEAD(request) { }    // 处理 HTTP HEAD 请求
export async function OPTIONS(request) { } // 处理 HTTP OPTIONS 请求
```

### 7.2 HTTP 方法对应表

| HTTP 方法 | 导出函数名 | REST 用途 |
|----------|-----------|----------|
| GET | `GET()` | 获取资源（只读） |
| POST | `POST()` | 创建资源 |
| PUT | `PUT()` | 全量更新资源 |
| PATCH | `PATCH()` | 部分更新资源 |
| DELETE | `DELETE()` | 删除资源 |

### 7.3 函数签名

#### 无动态参数

```typescript
export async function GET(request: NextRequest): Promise<Response> {
  // request 包含请求信息
  return Response.json({ data: "hello" });
}
```

#### 有动态参数

```typescript
export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ thread_id: string }> }
): Promise<Response> {
  const threadId = (await params).thread_id;
  return Response.json({ id: threadId });
}
```

### 7.4 request 参数详解

```typescript
export async function GET(request: NextRequest) {
  // ===== URL 信息 =====
  const url = request.nextUrl;
  const pathname = url.pathname;              // "/api/threads/abc"
  const searchParams = url.searchParams;      // URL 查询参数
  
  const page = searchParams.get("page");      // ?page=1 → "1"
  const limit = searchParams.get("limit");    // ?limit=10 → "10"
  
  // ===== Headers =====
  const headers = request.headers;
  const auth = headers.get("Authorization");  // 获取认证头
  const contentType = headers.get("Content-Type");
  
  // ===== Cookies =====
  const cookies = request.cookies;
  const sessionId = cookies.get("session")?.value;
  
  // ===== Body（POST/PUT/PATCH） =====
  const jsonBody = await request.json();      // JSON body
  const formData = await request.formData();  // FormData body
  const textBody = await request.text();      // 文本 body
  const arrayBuffer = await request.arrayBuffer(); // 二进制 body
  
  // ===== 其他 =====
  const method = request.method;              // "GET", "POST", etc.
  const geo = request.geo;                    // 地理位置（Vercel）
}
```

### 7.5 params 参数详解

```typescript
// app/api/threads/[thread_id]/artifacts/[[...artifact_path]]/route.ts

export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{
    thread_id: string;           // 单层动态路由 → string
    artifact_path?: string[];    // 捕获所有路由 → string[] 或 undefined
  }> }
) {
  // 必须 await params！
  const { thread_id, artifact_path } = await params;
  
  // URL: /api/threads/demo/artifacts/mnt/output.txt
  console.log(thread_id);      // "demo"（字符串）
  console.log(artifact_path);  // ["mnt", "output.txt"]（数组）
  
  // URL: /api/threads/demo/artifacts
  console.log(thread_id);      // "demo"
  console.log(artifact_path);  // undefined（可选捕获所有可为空）
  
  // 将数组转为路径字符串
  const filePath = artifact_path?.join("/") ?? "";
}
```

### 7.6 返回 Response 的方式

```typescript
// 返回 JSON
export async function GET() {
  return Response.json({ 
    success: true, 
    data: [...] 
  });
}

// 返回带状态码
export async function GET() {
  return new Response("Not found", { 
    status: 404,
    statusText: "Not Found"
  });
}

// 返回带 Headers
export async function GET() {
  return new Response("data", {
    status: 200,
    headers: {
      "Content-Type": "application/json",
      "Cache-Control": "no-cache",
      "X-Custom-Header": "value"
    }
  });
}

// 返回文件下载
export async function GET() {
  const fileContent = fs.readFileSync("file.txt");
  
  return new Response(fileContent, {
    status: 200,
    headers: {
      "Content-Type": "application/octet-stream",
      "Content-Disposition": "attachment; filename=file.txt"
    }
  });
}

// 代理请求并返回
export async function GET(request) {
  const backendResponse = await fetch("http://backend:8001/api/data");
  
  return new Response(await backendResponse.arrayBuffer(), {
    status: backendResponse.status,
    headers: backendResponse.headers
  });
}
```

### 7.7 未导出的方法返回 405

```typescript
// app/api/users/route.ts
export async function GET(request) {
  return Response.json({ users: [] });
}

// 只导出了 GET，没有 POST、DELETE 等
```

请求结果：

```
GET /api/users    → 200 OK，执行 GET() 函数
POST /api/users   → 405 Method Not Allowed
DELETE /api/users → 405 Method Not Allowed
PUT /api/users    → 405 Method Not Allowed
```

**Next.js 自动处理**：
- 有对应函数 → 执行
- 无对应函数 → 返回 `405 Method Not Allowed`

---

## 八、前端如何调用这些 API

### 8.1 URL 构建规则

**核心公式**：

```
API URL = 文件夹路径去掉 /app 和 /route.ts
```

```
文件夹结构                              URL
───────────────────────────────────────────────────────────────────

app/mock/api/models/route.ts           →  /mock/api/models

app/mock/api/skills/route.ts           →  /mock/api/skills

app/mock/api/mcp/config/route.ts       →  /mock/api/mcp/config

app/mock/api/threads/search/route.ts   →  /mock/api/threads/search

app/mock/api/threads/[thread_id]/
    history/route.ts                   →  /mock/api/threads/{实际值}/history

app/mock/api/threads/[thread_id]/
    artifacts/[[...artifact_path]]/
        route.ts                       →  /mock/api/threads/{实际值}/artifacts/{任意路径}
```

### 8.2 fetch 调用方式

#### 静态路由

```typescript
// 简单静态路由
const response = await fetch('/mock/api/models');
const data = await response.json();

// 带查询参数
const response = await fetch('/mock/api/models?version=v1');
```

#### 动态路由 `[thread_id]`

```typescript
const threadId = "demo-123";

// 用模板字符串拼接参数
const response = await fetch(`/mock/api/threads/${threadId}/history`, {
  method: "POST"
});
// URL: /mock/api/threads/demo-123/history
```

#### 可选捕获所有 `[[...artifact_path]]`

```typescript
const threadId = "demo-123";

// 无额外路径（artifact_path = undefined）
const response1 = await fetch(`/mock/api/threads/${threadId}/artifacts`);
// URL: /mock/api/threads/demo-123/artifacts

// 单层路径
const response2 = await fetch(`/mock/api/threads/${threadId}/artifacts/file.txt`);
// URL: /mock/api/threads/demo-123/artifacts/file.txt

// 多层路径
const response3 = await fetch(`/mock/api/threads/${threadId}/artifacts/mnt/output.txt`);
// URL: /mock/api/threads/demo-123/artifacts/mnt/output.txt

// 深层嵌套
const response4 = await fetch(`/mock/api/threads/${threadId}/artifacts/dir/sub/file.txt`);
// URL: /mock/api/threads/demo-123/artifacts/dir/sub/file.txt
```

### 8.3 不同 HTTP 方法

```typescript
// GET 请求 → 调用 GET() 函数
const response = await fetch('/mock/api/models');
// 默认 method: 'GET'

// POST 请求 → 调用 POST() 函数
const response = await fetch('/mock/api/threads/demo/history', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ data: '...' })
});

// DELETE 请求 → 调用 DELETE() 函数
const response = await fetch('/mock/api/memory/facts/123', {
  method: 'DELETE'
});

// PATCH 请求 → 调用 PATCH() 函数
const response = await fetch('/mock/api/memory/facts/123', {
  method: 'PATCH',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ content: 'new content' })
});
```

### 8.4 在 React 组件中使用

```tsx
"use client";

import { useState, useEffect } from "react";

export function ThreadHistory({ threadId }) {
  const [history, setHistory] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    // 动态 URL 拼接
    fetch(`/mock/api/threads/${threadId}/history`, {
      method: "POST"
    })
      .then(res => res.json())
      .then(data => {
        setHistory(data);
        setLoading(false);
      });
  }, [threadId]);
  
  if (loading) return <div>Loading...</div>;
  
  return <div>{JSON.stringify(history)}</div>;
}

// 使用方式
<ThreadHistory threadId="demo-123" />
```

---

## 九、完整示例

### 9.1 Mock API 完整代码示例

#### models/route.ts（硬编码数据）

```typescript
// app/mock/api/models/route.ts

export function GET() {
  // 直接返回硬编码数据，不需要请求后端
  return Response.json({
    models: [
      {
        id: "gpt-5",
        name: "gpt-5",
        display_name: "GPT-5",
        supports_thinking: true,
      },
      {
        id: "deepseek-v3",
        name: "deepseek-v3",
        display_name: "DeepSeek v3",
        supports_thinking: true,
      },
    ],
  });
}
```

#### threads/[thread_id]/history/route.ts（读取本地文件）

```typescript
// app/mock/api/threads/[thread_id]/history/route.ts
import fs from "fs";
import path from "path";
import type { NextRequest } from "next/server";

export async function POST(
  request: NextRequest,
  { params }: { params: Promise<{ thread_id: string }> }
) {
  const threadId = (await params).thread_id;
  
  try {
    // 从本地文件读取数据
    const filePath = path.resolve(
      process.cwd(),
      `public/demo/threads/${threadId}/thread.json`
    );
    
    // 检查文件是否存在
    if (!fs.existsSync(filePath)) {
      return new Response("Thread not found", { status: 404 });
    }
    
    // 读取文件内容
    const jsonString = fs.readFileSync(filePath, "utf8");
    const threadData = JSON.parse(jsonString);
    
    return Response.json(threadData);
  } catch (error) {
    return new Response("Error reading thread", { status: 500 });
  }
}
```

#### artifacts/[[...artifact_path]]/route.ts（处理任意路径）

```typescript
// app/mock/api/threads/[thread_id]/artifacts/[[...artifact_path]]/route.ts
import fs from "fs";
import path from "path";
import type { NextRequest } from "next/server";

export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{
    thread_id: string;
    artifact_path?: string[];
  }> }
) {
  const { thread_id, artifact_path } = await params;
  
  // 将数组转为路径字符串
  let filePath = artifact_path?.join("/") ?? "";
  
  // 如果没有路径，返回目录列表
  if (!filePath) {
    const dirPath = path.resolve(
      process.cwd(),
      `public/demo/threads/${thread_id}`
    );
    
    if (fs.existsSync(dirPath)) {
      const files = fs.readdirSync(dirPath);
      return Response.json({ files });
    }
    
    return new Response("Directory not found", { status: 404 });
  }
  
  // 处理虚拟路径映射
  if (filePath.startsWith("mnt/")) {
    filePath = filePath.replace("mnt/", "");
  }
  
  // 构建完整文件路径
  const fullPath = path.resolve(
    process.cwd(),
    `public/demo/threads/${thread_id}/${filePath}`
  );
  
  // 安全检查：防止路径遍历攻击
  const safeBase = path.resolve(process.cwd(), `public/demo/threads/${thread_id}`);
  if (!fullPath.startsWith(safeBase)) {
    return new Response("Forbidden", { status: 403 });
  }
  
  // 检查文件是否存在
  if (!fs.existsSync(fullPath)) {
    return new Response("File not found", { status: 404 });
  }
  
  // 读取并返回文件
  const content = fs.readFileSync(fullPath);
  
  // 根据文件扩展名设置 Content-Type
  const ext = path.extname(fullPath);
  const contentType = getContentType(ext);
  
  return new Response(content, {
    status: 200,
    headers: {
      "Content-Type": contentType,
      "Content-Disposition": `attachment; filename="${path.basename(fullPath)}"`
    }
  });
}

function getContentType(ext: string): string {
  const types: Record<string, string> = {
    ".txt": "text/plain",
    ".json": "application/json",
    ".html": "text/html",
    ".css": "text/css",
    ".js": "application/javascript",
    ".png": "image/png",
    ".jpg": "image/jpeg",
    ".pdf": "application/pdf",
  };
  return types[ext] || "application/octet-stream";
}
```

### 9.2 完整 URL 匹配表

```
route.ts 文件位置                                          匹配的 URL

app/mock/api/models/route.ts                               GET /mock/api/models

app/mock/api/skills/route.ts                               GET /mock/api/skills

app/mock/api/mcp/config/route.ts                           GET /mock/api/mcp/config

app/mock/api/threads/search/route.ts                       POST /mock/api/threads/search

app/mock/api/threads/[thread_id]/history/route.ts          POST /mock/api/threads/abc/history
                                                           POST /mock/api/threads/demo-123/history
                                                           POST /mock/api/threads/any-id/history

app/mock/api/threads/[thread_id]/artifacts/
    [[...artifact_path]]/route.ts                          GET /mock/api/threads/abc/artifacts
                                                           GET /mock/api/threads/abc/artifacts/file.txt
                                                           GET /mock/api/threads/abc/artifacts/mnt/output.txt
                                                           GET /mock/api/threads/abc/artifacts/dir/sub/file.txt
```

### 9.3 前端调用示例

```typescript
// core/artifacts/utils.ts
import { getBackendBaseURL } from "../config";

export function urlOfArtifact({
  filepath,
  threadId,
  download = false,
  isMock = false,
}: {
  filepath: string;
  threadId: string;
  download?: boolean;
  isMock?: boolean;
}) {
  const base = isMock ? `${getBackendBaseURL()}/mock/api` : `${getBackendBaseURL()}/api`;
  
  // 构建动态 URL
  return `${base}/threads/${threadId}/artifacts${filepath}${download ? "?download=true" : ""}`;
}

// 使用示例
const url = urlOfArtifact({
  threadId: "demo-123",
  filepath: "/mnt/output.txt",
  isMock: true
});
// 结果："/mock/api/threads/demo-123/artifacts/mnt/output.txt"
```

---

## 十、总结

### 10.1 Mock API 核心要点

| 要点 | 说明 |
|------|------|
| **作用** | 模拟后端 API，无需真实后端服务 |
| **数据来源** | 本地 JSON 文件或硬编码 |
| **运行位置** | Node.js 服务端（可用 fs、path） |
| **URL 格式** | `/mock/api/xxx` |
| **用途** | 开发、Demo、测试 |

### 10.2 动态路由命名要点

| 命名 | 匹配规则 | 参数类型 |
|------|---------|---------|
| `[param]` | 单个路径段 | `string` |
| `[...param]` | 至少一个路径段 | `string[]` |
| `[[...param]]` | 零个或多个路径段 | `string[]` 或 `undefined` |

### 10.3 route.ts 要点

| 要点 | 说明 |
|------|------|
| **函数名** | GET、POST、PUT、DELETE、PATCH（固定） |
| **运行位置** | Node.js 服务端 |
| **参数** | `request`（请求信息） + `params`（动态路由参数） |
| **返回** | `Response` 对象 |

### 10.4 一句话总结

> **Mock API** = 前端独立开发时的模拟后端，数据来自本地，运行在服务端
>
> **`[param]`** = 匹配单个路径段（如 `abc123`）
>
> **`[[...param]]`** = 匹配任意路径（如 `a/b/c.txt` 或空）
>
> **route.ts** = API 处理器，函数名对应 HTTP 方法，运行在 Node.js 可用 fs/path
>
> **URL 规则** = 文件夹路径去掉 `/app` 和 `/route.ts` = API URL