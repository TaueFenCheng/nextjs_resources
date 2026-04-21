---
title: React Core 模块代码组织模式
description: 分析 core/agents 模块的文件结构、职责分离和最佳实践
---

# React Core 模块代码组织模式

本文分析 DeerFlow 项目中 `core/` 模块的代码组织方式，以 `core/agents` 为例详解分层架构。

## 目录

1. [文件结构](#1-文件结构)
2. [分层架构](#2-分层架构)
3. [各层职责详解](#3-各层职责详解)
4. [使用方式](#4-使用方式)
5. [缓存管理策略](#5-缓存管理策略)
6. [组织优点](#6-组织优点)
7. [统一导出入口](#7-统一导出入口)
8. [与其他模块统一](#8-与其他模块统一)
9. [最佳实践](#9-最佳实践)

---

## 1. 文件结构

`core/agents` 模块的文件结构：

```
src/core/agents/
├── types.ts    ← 类型定义（接口、请求/响应类型）
├── api.ts      ← API 请求函数（纯函数，无 hooks）
├── hooks.ts    ← React Query hooks（封装 API + 缓存管理）
└── index.ts    ← 统一导出入口
```

### 文件职责表

| 文件 | 职责 | 依赖 |
|------|------|------|
| `types.ts` | 定义 TypeScript 类型 | 无 |
| `api.ts` | API 请求函数（纯函数） | `types.ts` |
| `hooks.ts` | React Query hooks | `api.ts`, `types.ts` |
| `index.ts` | 统一导出 | 所有文件 |

---

## 2. 分层架构

### 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                    core/agents 分层架构                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    组件层 (Components)                   │   │
│  │                                                         │   │
│  │  AgentGallery, AgentCard, NewAgentPage                  │   │
│  │                                                         │   │
│  │  import { useAgents, Agent } from "@/core/agents"       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           │ 导入                                │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Hooks 层 (hooks.ts)                   │   │
│  │                                                         │   │
│  │  useAgents()      → { agents, isLoading, error }        │   │
│  │  useAgent(name)   → { agent, isLoading, error }         │   │
│  │  useCreateAgent() → mutation + invalidate               │   │
│  │  useUpdateAgent() → mutation + invalidate               │   │
│  │  useDeleteAgent() → mutation + invalidate               │   │
│  │                                                         │   │
│  │  特点：                                                  │   │
│  │  - 使用 TanStack Query                                  │   │
│  │  - 自动缓存管理                                          │   │
│  │  - 返回适合组件使用的结构                                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           │ API 调用                            │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    API 层 (api.ts)                       │   │
│  │                                                         │   │
│  │  listAgents():       Promise<Agent[]>                   │   │
│  │  getAgent(name):     Promise<Agent>                     │   │
│  │  createAgent(req):   Promise<Agent>                     │   │
│  │  updateAgent(name, req): Promise<Agent>                 │   │
│  │  deleteAgent(name):  Promise<void>                      │   │
│  │  checkAgentName(name): Promise<{ available, name }>     │   │
│  │                                                         │   │
│  │  特点：                                                  │   │
│  │  - 纯函数，无 React 依赖                                 │   │
│  │  - 可在任何环境使用                                      │   │
│  │  - 自定义错误类                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           │ 类型约束                            │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Types 层 (types.ts)                   │   │
│  │                                                         │   │
│  │  interface Agent {                                      │   │
│  │    name: string;                                        │   │
│  │    description: string;                                 │   │
│  │    model: string | null;                                │   │
│  │    tool_groups: string[] | null;                        │   │
│  │    soul?: string | null;                                │   │
│  │  }                                                      │   │
│  │                                                         │   │
│  │  interface CreateAgentRequest { ... }                   │   │
│  │  interface UpdateAgentRequest { ... }                   │   │
│  │                                                         │   │
│  │  特点：                                                  │   │
│  │  - 统一类型定义                                          │   │
│  │  - 请求/响应类型分离                                     │   │
│  │  - TypeScript 类型安全                                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           │ 统一导出                            │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    导出入口 (index.ts)                   │   │
│  │                                                         │   │
│  │  export * from "./api";                                 │   │
│  │  export * from "./hooks";                               │   │
│  │  export * from "./types";                               │   │
│  │                                                         │   │
│  │  特点：                                                  │   │
│  │  - 单一导入入口                                          │   │
│  │  - 简化组件导入                                          │   │
│  │  - 统一 API 导出                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 数据流向

```
用户操作 → 组件 → hooks → api → 后端 API
                ↓
           React Query 缓存
                ↓
           状态更新 → 组件重渲染
```

---

## 3. 各层职责详解

### 3.1 Types 层 (types.ts)

定义所有 TypeScript 类型：

```tsx
// src/core/agents/types.ts

// 实体类型
export interface Agent {
  name: string;
  description: string;
  model: string | null;
  tool_groups: string[] | null;
  soul?: string | null;
}

// 创建请求类型
export interface CreateAgentRequest {
  name: string;
  description?: string;
  model?: string | null;
  tool_groups?: string[] | null;
  soul?: string;
}

// 更新请求类型
export interface UpdateAgentRequest {
  description?: string | null;
  model?: string | null;
  tool_groups?: string[] | null;
  soul?: string | null;
}
```

**职责：**

| 职责 | 说明 |
|------|------|
| 类型定义 | 定义实体、请求、响应类型 |
| 类型复用 | 被 API 和 hooks 层使用 |
| 类型安全 | 确保类型一致，避免错误 |
| 文档作用 | 类型即文档，说明数据结构 |

### 3.2 API 层 (api.ts)

纯函数 API 请求：

```tsx
// src/core/agents/api.ts
import { getBackendBaseURL } from "@/core/config";
import type { Agent, CreateAgentRequest, UpdateAgentRequest } from "./types";

// 自定义错误类
export class AgentNameCheckError extends Error {
  constructor(
    message: string,
    public readonly reason: "backend_unreachable" | "request_failed",
  ) {
    super(message);
    this.name = "AgentNameCheckError";
  }
}

// 列表查询
export async function listAgents(): Promise<Agent[]> {
  const res = await fetch(`${getBackendBaseURL()}/api/agents`);
  if (!res.ok) throw new Error(`Failed to load agents: ${res.statusText}`);
  const data = (await res.json()) as { agents: Agent[] };
  return data.agents;
}

// 单个查询
export async function getAgent(name: string): Promise<Agent> {
  const res = await fetch(`${getBackendBaseURL()}/api/agents/${name}`);
  if (!res.ok) throw new Error(`Agent '${name}' not found`);
  return res.json() as Promise<Agent>;
}

// 创建
export async function createAgent(request: CreateAgentRequest): Promise<Agent> {
  const res = await fetch(`${getBackendBaseURL()}/api/agents`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(request),
  });
  if (!res.ok) {
    const err = (await res.json().catch(() => ({}))) as { detail?: string };
    throw new Error(err.detail ?? `Failed to create agent: ${res.statusText}`);
  }
  return res.json() as Promise<Agent>;
}

// 更新
export async function updateAgent(
  name: string,
  request: UpdateAgentRequest,
): Promise<Agent> {
  const res = await fetch(`${getBackendBaseURL()}/api/agents/${name}`, {
    method: "PUT",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(request),
  });
  if (!res.ok) {
    const err = (await res.json().catch(() => ({}))) as { detail?: string };
    throw new Error(err.detail ?? `Failed to update agent: ${res.statusText}`);
  }
  return res.json() as Promise<Agent>;
}

// 删除
export async function deleteAgent(name: string): Promise<void> {
  const res = await fetch(`${getBackendBaseURL()}/api/agents/${name}`, {
    method: "DELETE",
  });
  if (!res.ok) throw new Error(`Failed to delete agent: ${res.statusText}`);
}

// 检查名称可用性（带特殊错误处理）
export async function checkAgentName(
  name: string,
): Promise<{ available: boolean; name: string }> {
  let res: Response;
  try {
    res = await fetch(
      `${getBackendBaseURL()}/api/agents/check?name=${encodeURIComponent(name)}`,
    );
  } catch {
    throw new AgentNameCheckError(
      "Could not reach the DeerFlow backend.",
      "backend_unreachable",
    );
  }

  if (!res.ok) {
    const err = (await res.json().catch(() => ({}))) as { detail?: string };
    if ([502, 503, 504].includes(res.status)) {
      throw new AgentNameCheckError(
        "Could not reach the DeerFlow backend.",
        "backend_unreachable",
      );
    }
    throw new AgentNameCheckError(
      err.detail ?? `Failed to check agent name: ${res.statusText}`,
      "request_failed",
    );
  }
  return res.json() as Promise<{ available: boolean; name: string }>;
}
```

**职责：**

| 职责 | 说明 |
|------|------|
| 网络请求 | 封装 fetch 调用 |
| 错误处理 | 统一错误抛出逻辑 |
| 无 React 依赖 | 可在任何环境使用 |
| 类型安全 | 使用 types.ts 类型 |

**使用场景：**

- Server Component 中直接调用
- Server Action 中调用
- 非 React 环境（如脚本）
- hooks 层调用

### 3.3 Hooks 层 (hooks.ts)

React Query hooks 封装：

```tsx
// src/core/agents/hooks.ts
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import {
  createAgent,
  deleteAgent,
  getAgent,
  listAgents,
  updateAgent,
} from "./api";
import type { CreateAgentRequest, UpdateAgentRequest } from "./types";

// 查询列表
export function useAgents() {
  const { data, isLoading, error } = useQuery({
    queryKey: ["agents"],
    queryFn: () => listAgents(),
  });
  return { agents: data ?? [], isLoading, error };
}

// 查询单个
export function useAgent(name: string | null | undefined) {
  const { data, isLoading, error } = useQuery({
    queryKey: ["agents", name],
    queryFn: () => getAgent(name!),
    enabled: !!name,  // 只有 name 存在时才查询
  });
  return { agent: data ?? null, isLoading, error };
}

// 创建 mutation
export function useCreateAgent() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (request: CreateAgentRequest) => createAgent(request),
    onSuccess: () => {
      // 创建成功后刷新列表
      void queryClient.invalidateQueries({ queryKey: ["agents"] });
    },
  });
}

// 更新 mutation
export function useUpdateAgent() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({
      name,
      request,
    }: {
      name: string;
      request: UpdateAgentRequest;
    }) => updateAgent(name, request),
    onSuccess: (_data, { name }) => {
      // 更新成功后刷新列表和单个
      void queryClient.invalidateQueries({ queryKey: ["agents"] });
      void queryClient.invalidateQueries({ queryKey: ["agents", name] });
    },
  });
}

// 删除 mutation
export function useDeleteAgent() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (name: string) => deleteAgent(name),
    onSuccess: () => {
      // 删除成功后刷新列表
      void queryClient.invalidateQueries({ queryKey: ["agents"] });
    },
  });
}
```

**职责：**

| 职责 | 说明 |
|------|------|
| React Query 封装 | 封装 useQuery/useMutation |
| 缓存管理 | 自动处理缓存失效 |
| 状态转换 | 返回适合组件使用的结构 |
| 条件查询 | 使用 enabled 控制查询时机 |

**返回结构对比：**

```tsx
// useQuery 原始返回
const { data, isLoading, error, refetch, isFetching } = useQuery(...);

// hooks 层封装返回（更简洁）
const { agents, isLoading, error } = useAgents();
const { agent, isLoading, error } = useAgent(name);
```

### 3.4 导出入口 (index.ts)

统一导出所有内容：

```tsx
// src/core/agents/index.ts
export * from "./api";
export * from "./hooks";
export * from "./types";
```

**职责：**

| 职责 | 说明 |
|------|------|
| 单一入口 | 所有导入从同一位置进入 |
| 简化导入 | `from "@/core/agents"` 代替多个导入 |
| 统一 API | 对外暴露统一接口 |

---

## 4. 使用方式

### 4.1 组件中使用 Hooks

```tsx
// 组件中使用 hooks（客户端组件）
"use client";

import { useAgents, useCreateAgent } from "@/core/agents";

export function AgentGallery() {
  // 查询列表
  const { agents, isLoading, error } = useAgents();

  // 创建 mutation
  const createMutation = useCreateAgent();

  async function handleCreate(name: string) {
    createMutation.mutate(
      { name, description: "" },
      {
        onSuccess: (agent) => {
          console.log("创建成功:", agent);
          // 自动刷新 agents 列表（hooks 内部处理）
        },
        onError: (error) => {
          console.error("创建失败:", error);
        },
      },
    );
  }

  if (isLoading) return <Spinner />;
  if (error) return <Error error={error} />;

  return (
    <div>
      {agents.map(agent => <AgentCard key={agent.name} agent={agent} />)}
    </div>
  );
}
```

### 4.2 Server Component 中使用 API

```tsx
// Server Component 中直接使用 API 函数
import { listAgents } from "@/core/agents/api";

export default async function AgentsPage() {
  // 直接调用 API，无 React Query
  const agents = await listAgents();

  return (
    <div>
      {agents.map(agent => <AgentCard key={agent.name} agent={agent} />)}
    </div>
  );
}
```

### 4.3 Server Action 中使用 API

```tsx
// Server Action 中使用
"use server";

import { createAgent, checkAgentName } from "@/core/agents/api";

export async function createAgentAction(formData: FormData) {
  const name = formData.get("name") as string;

  // 检查名称
  const checkResult = await checkAgentName(name);
  if (!checkResult.available) {
    return { error: "名称已存在" };
  }

  // 创建 agent
  const agent = await createAgent({ name });
  return { success: true, agent };
}
```

### 4.4 混合使用（复杂场景）

```tsx
// 复杂场景：hooks + 直接 API 调用
import { useAgents, checkAgentName, AgentNameCheckError } from "@/core/agents";

export function NewAgentPage() {
  const { agents } = useAgents();
  const [nameError, setNameError] = useState("");

  async function handleCheckName(name: string) {
    try {
      // 直接调用 API（不走 React Query）
      const result = await checkAgentName(name);
      if (!result.available) {
        setNameError("名称已存在");
      }
    } catch (err) {
      if (err instanceof AgentNameCheckError) {
        if (err.reason === "backend_unreachable") {
          setNameError("无法连接后端");
        } else {
          setNameError("检查失败");
        }
      }
    }
  }

  return <form>...</form>;
}
```

### 4.5 导入方式对比

```tsx
// ❌ 分别导入（繁琐）
import { useAgents } from "@/core/agents/hooks";
import { Agent } from "@/core/agents/types";
import { createAgent } from "@/core/agents/api";

// ✅ 统一导入（简洁）
import { useAgents, Agent, createAgent } from "@/core/agents";
```

---

## 5. 缓存管理策略

### 5.1 Query Key 设计

```tsx
// Query Key 层级设计
["agents"]           // 列表缓存
["agents", name]     // 单个 agent 缓存
["agents", name, "detail"]  // 更详细的缓存（如有）
```

### 5.2 缓存失效时机

```
┌─────────────────────────────────────────────────────────────────┐
│                     缓存失效时机                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  操作                  失效的 Query Key                          │
│  ───────────────────────────────────────────────────────────── │
│  createAgent()         ["agents"]                               │
│  updateAgent()         ["agents"], ["agents", name]             │
│  deleteAgent()         ["agents"]                               │
│                                                                 │
│  原因：                                                          │
│  - create/delete 会改变列表                                     │
│  - update 会改变列表和单个                                       │
│  - 失效后重新查询，保持数据一致性                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 缓存失效代码

```tsx
// 创建成功后失效列表缓存
export function useCreateAgent() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (request) => createAgent(request),
    onSuccess: () => {
      // 失效列表缓存，触发重新查询
      void queryClient.invalidateQueries({ queryKey: ["agents"] });
    },
  });
}

// 更新成功后失效列表 + 单个缓存
export function useUpdateAgent() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ name, request }) => updateAgent(name, request),
    onSuccess: (_data, { name }) => {
      // 失效列表缓存
      void queryClient.invalidateQueries({ queryKey: ["agents"] });
      // 失效单个缓存
      void queryClient.invalidateQueries({ queryKey: ["agents", name] });
    },
  });
}
```

### 5.4 手动刷新缓存

```tsx
// 组件中手动刷新
function AgentGallery() {
  const queryClient = useQueryClient();

  async function handleRefresh() {
    // 手动失效并重新获取
    await queryClient.invalidateQueries({ queryKey: ["agents"] });
  }

  // 或直接重新获取
  async function handleRefetch() {
    await queryClient.refetchQueries({ queryKey: ["agents"] });
  }
}
```

---

## 6. 组织优点

### 6.1 分层优点

| 层级 | 优点 |
|------|------|
| types.ts | 类型统一，文档作用，编译检查 |
| api.ts | 纯函数，无依赖，可复用，易测试 |
| hooks.ts | 缓存自动管理，组件简化，状态统一 |
| index.ts | 导入简洁，统一入口，易于重构 |

### 6.2 与其他方式对比

```tsx
// 方式一：全部写在组件中（混乱）
function AgentGallery() {
  const [agents, setAgents] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    setLoading(true);
    fetch("/api/agents")
      .then(res => res.json())
      .then(data => setAgents(data.agents))
      .finally(() => setLoading(false));
  }, []);

  async function handleCreate(name) {
    const res = await fetch("/api/agents", {
      method: "POST",
      body: JSON.stringify({ name }),
    });
    // 手动刷新列表
    setAgents([...agents, await res.json()]);
  }
}

// 方式二：分层组织（清晰）
function AgentGallery() {
  const { agents, isLoading } = useAgents();
  const createMutation = useCreateAgent();

  // 所有逻辑在 hooks 中处理
}
```

### 6.3 优点总结

| 优点 | 说明 |
|------|------|
| **职责分离** | types/api/hooks 各司其职 |
| **可复用** | API 函数可在多环境使用 |
| **类型安全** | TypeScript 类型统一管理 |
| **缓存自动化** | hooks 自动处理缓存失效 |
| **易测试** | API 纯函数易于单元测试 |
| **易维护** | 修改一处，影响可控 |
| **导入简洁** | 单一入口，统一导入 |

---

## 7. 统一导出入口

### 7.1 index.ts 作用

```tsx
// src/core/agents/index.ts
export * from "./api";    // 导出 API 函数
export * from "./hooks";  // 导出 hooks
export * from "./types";  // 导出类型
```

### 7.2 导入对比

```tsx
// ❌ 分别导入（繁琐）
import { useAgents } from "@/core/agents/hooks";
import { useAgent } from "@/core/agents/hooks";
import { Agent } from "@/core/agents/types";
import { CreateAgentRequest } from "@/core/agents/types";
import { createAgent } from "@/core/agents/api";
import { AgentNameCheckError } from "@/core/agents/api";

// ✅ 统一导入（简洁）
import {
  useAgents,
  useAgent,
  Agent,
  CreateAgentRequest,
  createAgent,
  AgentNameCheckError,
} from "@/core/agents";
```

### 7.3 重构友好

```tsx
// 如果需要重构模块结构，只需修改 index.ts
// src/core/agents/index.ts
export * from "./queries";  // 重命名 hooks.ts → queries.ts
export * from "./services"; // 重命名 api.ts → services.ts
export * from "./types";

// 组件导入无需修改
import { useAgents } from "@/core/agents";  // 仍然有效
```

---

## 8. 与其他模块统一

整个 `core/` 目录遵循相同的组织模式：

```
src/core/
├── agents/     → types.ts, api.ts, hooks.ts, index.ts
├── memory/     → types.ts, api.ts, hooks.ts, index.ts
├── mcp/        → types.ts, api.ts, hooks.ts, index.ts
├── models/     → types.ts, api.ts, hooks.ts, index.ts
├── skills/     → type.ts,  api.ts, hooks.ts, index.ts
├── uploads/    → api.ts,   hooks.ts, index.ts（无 types）
├── threads/    → types.ts, hooks.ts, index.ts（无 api，复杂逻辑）
├── artifacts/  → hooks.ts, loader.ts, index.ts（无 api/types）
├── settings/   → hooks.ts, local.ts, store.ts, index.ts（本地状态）
├── i18n/       → context.tsx, hooks.ts, index.ts, locales/（国际化）
└── utils/      → datetime.ts, files.tsx, json.ts（工具函数）
```

### 模块类型对比

| 模块类型 | 文件组成 | 说明 |
|---------|---------|------|
| API 模块 | types + api + hooks + index | 标准 CRUD 模块 |
| 复杂模块 | types + hooks + utils + index | 复杂业务逻辑 |
| 本地模块 | hooks + store + index | 本地状态管理 |
| 工具模块 | 多个独立文件 | 纯工具函数 |
| Context 模块 | context + hooks + index | React Context |

---

## 9. 最佳实践

### 9.1 新建模块模板

```tsx
// 1. 创建目录
mkdir src/core/my-module

// 2. 创建文件
touch src/core/my-module/types.ts
touch src/core/my-module/api.ts
touch src/core/my-module/hooks.ts
touch src/core/my-module/index.ts

// 3. types.ts
export interface MyEntity {
  id: string;
  name: string;
}

export interface CreateMyEntityRequest {
  name: string;
}

// 4. api.ts
import { getBackendBaseURL } from "@/core/config";
import type { MyEntity, CreateMyEntityRequest } from "./types";

export async function listMyEntities(): Promise<MyEntity[]> {
  const res = await fetch(`${getBackendBaseURL()}/api/my-entities`);
  if (!res.ok) throw new Error(`Failed: ${res.statusText}`);
  return res.json();
}

export async function createMyEntity(request: CreateMyEntityRequest): Promise<MyEntity> {
  const res = await fetch(`${getBackendBaseURL()}/api/my-entities`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(request),
  });
  if (!res.ok) throw new Error(`Failed: ${res.statusText}`);
  return res.json();
}

// 5. hooks.ts
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { listMyEntities, createMyEntity } from "./api";

export function useMyEntities() {
  const { data, isLoading, error } = useQuery({
    queryKey: ["my-entities"],
    queryFn: () => listMyEntities(),
  });
  return { entities: data ?? [], isLoading, error };
}

export function useCreateMyEntity() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (request) => createMyEntity(request),
    onSuccess: () => {
      void queryClient.invalidateQueries({ queryKey: ["my-entities"] });
    },
  });
}

// 6. index.ts
export * from "./api";
export * from "./hooks";
export * from "./types";
```

### 9.2 Query Key 命名规范

```tsx
// ✅ 好的命名
["agents"]              // 列表
["agents", name]        // 单个
["agents", name, "config"]  // 详细配置

// ❌ 不好的命名
["agentList"]           // 不一致
["getAgent"]            // 函数名，不是资源名
["agents", name, "data", "detail"]  // 过长
```

### 9.3 错误处理规范

```tsx
// 自定义错误类
export class MyModuleError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly reason?: string,
  ) {
    super(message);
    this.name = "MyModuleError";
  }
}

// 组件中使用
try {
  await someOperation();
} catch (err) {
  if (err instanceof MyModuleError) {
    if (err.code === "NOT_FOUND") {
      toast.error("资源未找到");
    } else if (err.reason === "network") {
      toast.error("网络错误");
    }
  }
}
```

### 9.4 条件查询规范

```tsx
// 使用 enabled 控制查询时机
export function useAgent(name: string | null | undefined) {
  const { data, isLoading, error } = useQuery({
    queryKey: ["agents", name],
    queryFn: () => getAgent(name!),
    enabled: !!name,  // name 存在时才查询
  });
  return { agent: data ?? null, isLoading, error };
}

// 使用 staleTime 控制刷新频率
export function useAgents() {
  return useQuery({
    queryKey: ["agents"],
    queryFn: () => listAgents(),
    staleTime: 60 * 1000,  // 1 分钟内不重新查询
  });
}
```

---

## 总结

### 核心原则

| 原则 | 实践 |
|------|------|
| **分层设计** | types → api → hooks → index |
| **职责分离** | 每层只做一件事 |
| **类型安全** | TypeScript 类型统一定义 |
| **可复用** | API 函数无 React 依赖 |
| **缓存自动化** | hooks 自动管理 React Query |
| **统一入口** | index.ts 统一导出 |

### 适用场景

- 标准 CRUD 模块
- 需要缓存管理的数据
- 需要在多环境使用的 API
- 复杂业务逻辑需要分层

---

## 参考资源

- [TanStack Query 文档](https://tanstack.com/query)
- [React 最佳实践](https://react.dev/learn)
- [TypeScript 类型设计](https://www.typescriptlang.org/docs/handbook/2/types-from-types.html)