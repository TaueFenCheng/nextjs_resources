---
title: TanStack Query (React Query) 核心用法
description: 详细介绍 useQuery、useMutation、useQueryClient 的使用方法
---

# TanStack Query (React Query) 核心用法

本文详细介绍 TanStack Query（原名 React Query）的三个核心 hooks：`useQuery`、`useMutation`、`useQueryClient`。

## 目录

1. [什么是 TanStack Query](#1-什么是-tanstack-query)
2. [useQuery - 数据查询](#2-usequery---数据查询)
3. [useMutation - 数据变更](#3-usemutation---数据变更)
4. [useQueryClient - 缓存管理](#4-usequeryclient---缓存管理)
5. [Query Key 设计](#5-query-key-设计)
6. [Query Key 深度理解](#6-query-key-深度理解)
7. [缓存失效策略](#7-缓存失效策略)
8. [完整示例](#8-完整示例)
9. [最佳实践](#9-最佳实践)

---

## 1. 什么是 TanStack Query

TanStack Query 是 React 的**异步数据管理库**，用于处理服务器状态的获取、缓存、更新和同步。

### 核心优势

| 优势 | 说明 |
|------|------|
| **自动缓存** | 查询结果自动缓存，减少重复请求 |
| **后台更新** | 自动在后台重新获取数据 |
| **状态管理** | 提供 loading、error、success 状态 |
| **乐观更新** | 支持乐观 UI 更新 |
| **失效管理** | 自动处理缓存失效 |

### 核心概念

```
┌─────────────────────────────────────────────────────────────────┐
│                    TanStack Query 核心概念                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Query（查询）                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  用于获取数据（GET）                                      │   │
│  │  useQuery({ queryKey, queryFn })                        │   │
│  │  自动缓存、自动后台更新                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Mutation（变更）                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  用于修改数据（POST/PUT/DELETE）                          │   │
│  │  useMutation({ mutationFn, onSuccess, onError })        │   │
│  │  手动触发、可处理缓存失效                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  QueryClient（客户端）                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  管理缓存和查询状态                                       │   │
│  │  useQueryClient() → QueryClient                         │   │
│  │  invalidateQueries, refetchQueries, setQueryData        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Query Key（查询键）                                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  缓存的唯一标识                                           │   │
│  │  ["agents"] 或 ["agents", "agent-name"]                 │   │
│  │  数组格式，支持层级结构                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. useQuery - 数据查询

### 2.1 基本用法

```tsx
import { useQuery } from "@tanstack/react-query";

function Component() {
  const { data, isLoading, error, refetch, isFetching } = useQuery({
    queryKey: ["agents"],        // 缓存键
    queryFn: () => listAgents(), // 数据获取函数
  });

  if (isLoading) return <Spinner />;
  if (error) return <Error error={error} />;
  
  return <div>{data.map(agent => ...)}</div>;
}
```

### 2.2 参数详解

```tsx
useQuery({
  // 必需参数
  queryKey: ["agents"],           // 缓存键（数组）
  queryFn: () => fetchAgents(),   // 数据获取函数

  // 可选参数
  enabled: true,                  // 是否启用查询（默认 true）
  staleTime: 0,                   // 数据新鲜时间（毫秒，默认 0）
  gcTime: 5 * 60 * 1000,          // 缓存保留时间（毫秒，默认 5 分钟）
  refetchOnWindowFocus: true,     // 窗口聚焦时是否重新获取（默认 true）
  refetchOnMount: true,           // 组件挂载时是否重新获取（默认 true）
  refetchInterval: false,         // 自动重新获取间隔（毫秒，默认 false）
  retry: 3,                       // 失败重试次数（默认 3）
  retryDelay: 1000,               // 重试延迟（毫秒）
  select: (data) => data.agents,  // 数据转换函数
  placeholderData: [],            // 占位数据（加载前显示）
});
```

### 2.3 返回值详解

```tsx
const result = useQuery({ queryKey: ["agents"], queryFn: listAgents });

// 返回值结构
result = {
  // 数据状态
  data: Agent[] | undefined,      // 查询结果数据
  dataUpdatedAt: number,          // 数据更新时间戳
  
  // 加载状态
  isLoading: boolean,             // 初次加载中（无缓存数据）
  isFetching: boolean,            // 正在获取（包括后台更新）
  isRefetching: boolean,          // 后台重新获取
  isStale: boolean,               // 数据是否过期
  isSuccess: boolean,             // 查询成功
  
  // 错误状态
  isError: boolean,               // 查询失败
  error: Error | null,            // 错误信息
  failureCount: number,           // 失败次数
  failureReason: Error | null,    // 失败原因
  
  // 手动操作
  refetch: () => Promise,         // 手动重新获取
  promise: Promise,               // 当前查询 Promise
  
  // 状态标记
  status: "idle" | "pending" | "success" | "error",
  fetchStatus: "idle" | "fetching" | "paused",
};
```

### 2.4 enabled 条件查询

```tsx
// enabled 控制查询是否执行
export function useAgent(name: string | null | undefined) {
  const { data, isLoading, error } = useQuery({
    queryKey: ["agents", name],
    queryFn: () => getAgent(name!),  // 使用 name!
    enabled: !!name,                 // 只有 name 存在时才查询
  });
  
  return { agent: data ?? null, isLoading, error };
}

// 使用示例
function AgentDetail({ agentName }: { agentName?: string }) {
  const { agent, isLoading } = useAgent(agentName);
  
  // agentName 为 undefined 时：
  // - enabled: false
  // - 不执行 queryFn
  // - isLoading: false
  // - data: undefined
  
  if (!agentName) return <div>请选择一个 Agent</div>;
  if (isLoading) return <Spinner />;
  return <div>{agent?.name}</div>;
}
```

### 2.5 staleTime 和 gcTime

```tsx
// staleTime - 数据新鲜时间
useQuery({
  queryKey: ["agents"],
  queryFn: listAgents,
  staleTime: 60 * 1000,  // 1 分钟内认为数据新鲜
  // 效果：
  // - 1 分钟内重新访问，不会重新获取
  // - 立即返回缓存数据
  // - 超过 1 分钟，会在后台重新获取
});

// gcTime - 缓存保留时间（垃圾回收时间）
useQuery({
  queryKey: ["agents"],
  queryFn: listAgents,
  gcTime: 10 * 60 * 1000,  // 10 分钟后清理缓存
  // 效果：
  // - 组件卸载后，缓存保留 10 分钟
  // - 10 分钟内重新挂载，使用缓存数据
  // - 超过 10 分钟，缓存被清理
});
```

### 2.6 select 数据转换

```tsx
// select 用于转换数据
useQuery({
  queryKey: ["agents"],
  queryFn: listAgents,
  select: (data) => {
    // 转换数据结构
    return {
      activeAgents: data.filter(a => a.active),
      inactiveAgents: data.filter(a => !a.active),
      total: data.length,
    };
  },
});

// 组件中直接使用转换后的数据
const { data } = useQuery(...);
// data = { activeAgents: [...], inactiveAgents: [...], total: 10 }
```

### 2.7 状态区分

```tsx
// isLoading vs isFetching
const { isLoading, isFetching } = useQuery(...);

// isLoading: true  → 初次加载，没有任何数据
// isFetching: true → 正在获取数据（包括后台更新）

// 区别：
// - isLoading 为 true 时，isFetching 也为 true
// - 后台更新时，isLoading 为 false，isFetching 为 true

// 示例
function Component() {
  const { isLoading, isFetching, data } = useQuery(...);

  // 初次加载
  if (isLoading) return <Spinner />;
  
  // 显示数据，同时显示更新指示器
  return (
    <div>
      {isFetching && <UpdatingIndicator />}
      {data.map(item => ...)}
    </div>
  );
}
```

---

## 3. useMutation - 数据变更

### 3.1 基本用法

```tsx
import { useMutation } from "@tanstack/react-query";

function Component() {
  const mutation = useMutation({
    mutationFn: (name: string) => createAgent({ name }),
  });

  // mutation 对象
  mutation = {
    mutate: (name, options) => void,  // 执行变更
    mutateAsync: (name) => Promise,   // 异步执行变更
    reset: () => void,                // 重置状态
    
    // 状态
    isPending: boolean,               // 执行中
    isError: boolean,                 // 失败
    isSuccess: boolean,               // 成功
    isIdle: boolean,                  // 未执行
    
    // 数据
    data: Agent | undefined,          // 成功结果
    error: Error | null,              // 错误信息
    
    // 状态标记
    status: "idle" | "pending" | "success" | "error",
  };

  async function handleCreate(name: string) {
    mutation.mutate(name, {
      onSuccess: (data) => {
        console.log("创建成功:", data);
      },
      onError: (error) => {
        console.error("创建失败:", error);
      },
    });
  }
}
```

### 3.2 mutationFn 参数类型

```tsx
// mutationFn 可以接收各种类型的参数

// 单个值
useMutation({
  mutationFn: (name: string) => deleteAgent(name),
});

// 对象参数
useMutation({
  mutationFn: (request: CreateAgentRequest) => createAgent(request),
});

// 多个参数（需要用对象包装）
useMutation({
  mutationFn: ({ name, request }: { 
    name: string; 
    request: UpdateAgentRequest;
  }) => updateAgent(name, request),
});

// 调用时传入对象
mutation.mutate({ name: "agent-1", request: { description: "新描述" } });
```

### 3.3 回调函数详解

```tsx
const mutation = useMutation({
  mutationFn: ({ name, request }) => updateAgent(name, request),
  
  // 成功回调
  onSuccess: (data, variables, context) => {
    // data: 返回的结果（Agent）
    // variables: mutate 传入的参数（{ name, request }）
    // context: onMutate 返回的上下文
    
    console.log("成功:", data);
    console.log("参数:", variables);
  },
  
  // 错误回调
  onError: (error, variables, context) => {
    // error: 错误对象
    // variables: mutate 传入的参数
    // context: onMutate 返回的上下文
    
    console.error("失败:", error);
    toast.error(error.message);
  },
  
  // 完成回调（无论成功或失败）
  onSettled: (data, error, variables, context) => {
    // 总是执行
    console.log("完成");
  },
  
  // 执行前回调（用于乐观更新）
  onMutate: (variables) => {
    // 返回的值会传递给 onSuccess、onError、onSettled
    return { previousAgents: queryClient.getQueryData(["agents"]) };
  },
});
```

### 3.4 回调参数图解

```
┌─────────────────────────────────────────────────────────────────┐
│                    Mutation 回调流程                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  mutate({ name, request })                                      │
│       │                                                         │
│       │  variables = { name, request }                          │
│       ▼                                                         │
│  onMutate(variables)                                            │
│       │                                                         │
│       │  return context = { previousAgents: [...] }             │
│       ▼                                                         │
│  mutationFn(variables)                                          │
│       │                                                         │
│       │  执行 API 请求                                           │
│       ▼                                                         │
│  ┌─────────────┐                                                │
│  │   成功？     │                                                │
│  └─────────────┘                                                │
│       │                                                         │
│   ┌───┴───┐                                                     │
│   │ Yes   │ No                                                  │
│   ▼       ▼                                                     │
│  onSuccess(data, variables, context)                            │
│  onError(error, variables, context)                             │
│       │       │                                                 │
│       └───┬───┘                                                 │
│           ▼                                                     │
│  onSettled(data, error, variables, context)                     │
│           │                                                     │
│           ▼                                                     │
│  完成                                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.5 invalidateQueries 缓存失效

```tsx
const queryClient = useQueryClient();

const mutation = useMutation({
  mutationFn: (request) => createAgent(request),
  
  onSuccess: (data, variables) => {
    // 失效查询缓存，触发重新获取
    void queryClient.invalidateQueries({ queryKey: ["agents"] });
    
    // 可以同时失效多个
    void queryClient.invalidateQueries({ queryKey: ["agents"] });
    void queryClient.invalidateQueries({ queryKey: ["stats"] });
  },
});
```

### 3.6 更新复杂参数示例

```tsx
// 更新 Agent 的 mutation
export function useUpdateAgent() {
  const queryClient = useQueryClient();
  
  return useMutation({
    // 接收对象参数
    mutationFn: ({
      name,
      request,
    }: {
      name: string;
      request: UpdateAgentRequest;
    }) => updateAgent(name, request),
    
    onSuccess: (_data, { name }) => {
      // 失效列表缓存
      void queryClient.invalidateQueries({ queryKey: ["agents"] });
      
      // 失效单个缓存（使用参数中的 name）
      void queryClient.invalidateQueries({ queryKey: ["agents", name] });
    },
  });
}

// 组件中使用
function AgentEditor({ agentName }: { agentName: string }) {
  const updateMutation = useUpdateAgent();
  
  function handleUpdate(description: string) {
    updateMutation.mutate(
      { 
        name: agentName, 
        request: { description } 
      },
      {
        onSuccess: () => {
          toast.success("更新成功");
        },
      },
    );
  }
}
```

### 3.7 mutate vs mutateAsync

```tsx
const mutation = useMutation({ mutationFn: createAgent });

// mutate - 不返回 Promise，适合回调处理
mutation.mutate({ name: "test" }, {
  onSuccess: (data) => console.log(data),
  onError: (error) => console.error(error),
});

// mutateAsync - 返回 Promise，适合 async/await
async function handleCreate() {
  try {
    const data = await mutation.mutateAsync({ name: "test" });
    console.log(data);
  } catch (error) {
    console.error(error);
  }
}
```

---

## 4. useQueryClient - 缓存管理

### 4.1 基本用法

```tsx
import { useQueryClient } from "@tanstack/react-query";

function Component() {
  const queryClient = useQueryClient();

  // queryClient 提供多种缓存管理方法
}
```

### 4.2 invalidateQueries - 失效缓存

```tsx
// 失效指定查询
void queryClient.invalidateQueries({ 
  queryKey: ["agents"] 
});

// 失效匹配的所有查询（使用部分 key）
void queryClient.invalidateQueries({ 
  queryKey: ["agents"] 
});
// 会失效: ["agents"], ["agents", "name"], ["agents", "name", "detail"]

// 精确失效（只失效完全匹配的）
void queryClient.invalidateQueries({ 
  queryKey: ["agents"],
  exact: true,  // 只失效 ["agents"]，不影响子级
});

// 失效所有查询
void queryClient.invalidateQueries();

// 使用 predicate 函数失效
void queryClient.invalidateQueries({
  predicate: (query) => query.queryKey[0] === "agents",
});

// 失效并立即重新获取
await queryClient.invalidateQueries({ 
  queryKey: ["agents"],
  refetchType: "active",  // 失效后立即重新获取活跃查询
});
```

### 4.3 refetchQueries - 重新获取

```tsx
// 重新获取指定查询
await queryClient.refetchQueries({ 
  queryKey: ["agents"] 
});

// 重新获取所有查询
await queryClient.refetchQueries();

// 重新获取活跃查询
await queryClient.refetchQueries({ 
  type: "active"  // 只重新获取当前正在使用的查询
});

// 重新获取过期的查询
await queryClient.refetchQueries({ 
  type: "stale"  // 只重新获取过期数据
});

// 强制重新获取（忽略 staleTime）
await queryClient.refetchQueries({
  queryKey: ["agents"],
  force: true,
});
```

### 4.4 setQueryData - 直接设置数据

```tsx
// 直接设置缓存数据（不触发请求）
queryClient.setQueryData(["agents"], [
  { name: "agent-1", description: "Agent 1" },
  { name: "agent-2", description: "Agent 2" },
]);

// 更新数据（使用函数）
queryClient.setQueryData(["agents"], (oldData) => {
  if (!oldData) return [];
  return [...oldData, { name: "agent-3", description: "Agent 3" }];
});

// 更新单个 Agent
queryClient.setQueryData(["agents", "agent-1"], (oldData) => {
  if (!oldData) return oldData;
  return { ...oldData, description: "新描述" };
});
```

### 4.5 getQueryData - 获取缓存数据

```tsx
// 获取缓存数据（不触发请求）
const agents = queryClient.getQueryData(["agents"]);

// 获取单个缓存
const agent = queryClient.getQueryData(["agents", "agent-1"]);

// 用于乐观更新回滚
const mutation = useMutation({
  mutationFn: deleteAgent,
  onMutate: (name) => {
    // 保存旧数据用于回滚
    const previousAgents = queryClient.getQueryData(["agents"]);
    return { previousAgents };
  },
  onError: (err, name, context) => {
    // 回滚到旧数据
    queryClient.setQueryData(["agents"], context.previousAgents);
  },
});
```

### 4.6 removeQueries - 删除缓存

```tsx
// 删除指定查询缓存
queryClient.removeQueries({ queryKey: ["agents"] });

// 删除匹配的查询
queryClient.removeQueries({ 
  queryKey: ["agents"],
  exact: true,
});

// 删除所有缓存
queryClient.removeQueries();
```

### 4.7 cancelQueries - 取消查询

```tsx
// 取消正在进行的查询
await queryClient.cancelQueries({ queryKey: ["agents"] });

// 用于清理
useEffect(() => {
  return () => {
    queryClient.cancelQueries({ queryKey: ["agents"] });
  };
}, []);
```

### 4.8 clear - 清空所有缓存

```tsx
// 清空所有缓存
queryClient.clear();
```

---

## 5. Query Key 设计

### 5.1 基本规则

```tsx
// Query Key 是数组格式
queryKey: ["agents"]           // 简单
queryKey: ["agents", name]     // 带参数
queryKey: ["agents", name, "detail"]  // 层级

// 规则：
// 1. 使用数组格式
// 2. 第一个元素是资源名称
// 3. 后续元素是参数或子级
```

### 5.2 层级设计

```
┌─────────────────────────────────────────────────────────────────┐
│                    Query Key 层级设计                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ["agents"]                                                     │
│     │                                                           │
│     ├── ["agents", "agent-1"]                                   │
│     │       │                                                   │
│     │       ├── ["agents", "agent-1", "config"]                │
│     │       └── ["agents", "agent-1", "history"]               │
│     │                                                           │
│     ├── ["agents", "agent-2"]                                   │
│     │                                                           │
│     └── ["agents", "agent-3"]                                   │
│                                                                 │
│  失效 ["agents"] 会影响所有子级                                  │
│  失效 ["agents", "agent-1"] 只影响 agent-1 相关                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 常见设计模式

```tsx
// 列表
queryKey: ["agents"]

// 单个资源
queryKey: ["agents", agentName]

// 分页列表
queryKey: ["agents", { page: 1, limit: 10 }]

// 过滤列表
queryKey: ["agents", { status: "active", sortBy: "name" }]

// 用户相关
queryKey: ["users", userId, "agents"]

// 详情子级
queryKey: ["agents", agentName, "config"]
queryKey: ["agents", agentName, "history"]
queryKey: ["agents", agentName, "tools"]
```

### 5.4 Query Key Factory

```tsx
// 推荐：使用工厂函数统一管理
const agentKeys = {
  all: ["agents"] as const,
  lists: () => [...agentKeys.all, "list"] as const,
  list: (filters: Record<string, unknown>) => 
    [...agentKeys.lists(), filters] as const,
  details: () => [...agentKeys.all, "detail"] as const,
  detail: (name: string) => 
    [...agentKeys.details(), name] as const,
  config: (name: string) => 
    [...agentKeys.detail(name), "config"] as const,
};

// 使用
useQuery({ queryKey: agentKeys.lists() });
useQuery({ queryKey: agentKeys.detail("agent-1") });
useQuery({ queryKey: agentKeys.config("agent-1") });

// 失效
queryClient.invalidateQueries({ queryKey: agentKeys.all });  // 失效所有
queryClient.invalidateQueries({ queryKey: agentKeys.detail("agent-1") });
```

---

## 6. Query Key 深度理解

### 6.1 queryKey 是什么

`queryKey` 是缓存的**唯一标识符**，类似数据库的主键。

```tsx
// queryKey 决定了：
// 1. 缓存存储位置
// 2. 缓存是否需要更新
// 3. 不同查询是否共享缓存
```

### 6.2 缓存存储结构

```
┌─────────────────────────────────────────────────────────────────┐
│                    queryKey = 缓存标识符                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  缓存存储结构（类似 Map）                                         │
│                                                                 │
│  Cache = {                                                      │
│    ["agents"]           → { data: [...], updatedAt: ... }       │
│    ["agents", "agent-1"] → { data: {...}, updatedAt: ... }      │
│    ["agents", "agent-2"] → { data: {...}, updatedAt: ... }      │
│    ["mcpConfig"]        → { data: {...}, updatedAt: ... }       │
│  }                                                              │
│                                                                 │
│  每个不同的 queryKey 对应独立的缓存条目                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 动态 queryKey 的理解

```tsx
function Component({ agentName }: { agentName: string }) {
  const { data } = useQuery({
    queryKey: ["agent", agentName],  // agentName 是动态变量
    queryFn: () => getAgent(agentName),
  });
}
```

**关键理解：queryKey 的值在每次渲染时计算**

```
┌─────────────────────────────────────────────────────────────────┐
│                    动态 queryKey 工作机制                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  渲染 1: agentName = "agent-1"                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  queryKey = ["agent", "agent-1"]                        │   │
│  │                                                         │   │
│  │  Cache 检查:                                            │   │
│  │  - 缓存中是否有 ["agent", "agent-1"]?                   │   │
│  │  - 没有 → 执行 queryFn，获取 agent-1 数据               │   │
│  │  - 缓存结果到 ["agent", "agent-1"]                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  渲染 2: agentName = "agent-2"（agentName 变化）                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  queryKey = ["agent", "agent-2"]                        │   │
│  │                                                         │   │
│  │  Cache 检查:                                            │   │
│  │  - 缓存中是否有 ["agent", "agent-2"]?                   │   │
│  │  - 没有 → 执行 queryFn，获取 agent-2 数据               │   │
│  │  - 缓存结果到 ["agent", "agent-2"]                       │   │
│  │                                                         │   │
│  │  注意: ["agent", "agent-1"] 的缓存仍然存在              │   │
│  │       只是这次用的是不同的 key                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  渲染 3: agentName = "agent-1"（切换回来）                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  queryKey = ["agent", "agent-1"]                        │   │
│  │                                                         │   │
│  │  Cache 检查:                                            │   │
│  │  - 缓存中是否有 ["agent", "agent-1"]?                   │   │
│  │  - 有 → 使用缓存，不重新获取（除非数据过期）             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.4 动态 queryKey 的实际效果

```tsx
// 组件切换不同 agent
function AgentDetail() {
  const [selectedAgent, setSelectedAgent] = useState("agent-1");

  const { data, isLoading } = useQuery({
    queryKey: ["agent", selectedAgent],
    queryFn: () => getAgent(selectedAgent),
  });

  return (
    <div>
      {/* 选择不同的 agent */}
      <button onClick={() => setSelectedAgent("agent-1")}>Agent 1</button>
      <button onClick={() => setSelectedAgent("agent-2")}>Agent 2</button>
      <button onClick={() => setSelectedAgent("agent-3")}>Agent 3</button>

      {/* 显示当前 agent */}
      {isLoading ? <Spinner /> : <div>{data?.name}</div>}
    </div>
  );
}

// 执行流程：
// 点击 "Agent 1" → queryKey: ["agent", "agent-1"] → 获取数据 → 缓存
// 点击 "Agent 2" → queryKey: ["agent", "agent-2"] → 获取数据 → 缓存
// 点击 "Agent 3" → queryKey: ["agent", "agent-3"] → 获取数据 → 缓存
// 点击 "Agent 1" → queryKey: ["agent", "agent-1"] → 使用缓存（不重新获取）
```

### 6.5 queryKey 变化时的处理

```tsx
// queryKey 变化 = 新的查询
// React Query 会：

// 1. 检查新 key 是否有缓存
// 2. 如果有缓存且未过期 → 使用缓存
// 3. 如果没有缓存或已过期 → 执行 queryFn
// 4. 旧 key 的缓存保留在内存中
```

### 6.6 多参数动态 queryKey

```tsx
// 多个动态参数
useQuery({
  queryKey: ["agents", { status: activeStatus, sortBy: sortField }],
  queryFn: () => listAgents({ status: activeStatus, sortBy: sortField }),
});

// 不同参数组合 = 不同缓存
// ["agents", { status: "active", sortBy: "name" }] → 缓存 A
// ["agents", { status: "inactive", sortBy: "name" }] → 缓存 B
// ["agents", { status: "active", sortBy: "date" }] → 缓存 C

// 参数变化 → 检查对应缓存是否存在
```

### 6.7 queryKey 层级关系

```
┌─────────────────────────────────────────────────────────────────┐
│                    queryKey 层级关系                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ["agents"]                     ← 父级                          │
│     │                                                           │
│     ├── ["agents", "agent-1"]   ← 子级                          │
│     │     │                                                     │
│     │     ├── ["agents", "agent-1", "config"]  ← 更深层         │
│     │     └── ["agents", "agent-1", "history"]                 │
│     │                                                           │
│     ├── ["agents", "agent-2"]                                   │
│     │                                                           │
│     └── ["agents", "agent-3"]                                   │
│                                                                 │
│  失效规则:                                                       │
│  - invalidate ["agents"] → 失效所有子级                          │
│  - invalidate ["agents", "agent-1"] → 只失效 agent-1 相关       │
│  - 使用 exact: true → 只失效精确匹配的 key                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.8 invalidateQueries 与动态 queryKey

```tsx
// 失效所有 agents 相关缓存
void queryClient.invalidateQueries({ queryKey: ["agents"] });
// 会失效: ["agents"], ["agents", "agent-1"], ["agents", "agent-2"] 等

// 失效特定 agent
void queryClient.invalidateQueries({ queryKey: ["agents", "agent-1"] });
// 只失效: ["agents", "agent-1"] 及其子级

// 精确失效
void queryClient.invalidateQueries({ 
  queryKey: ["agents"], 
  exact: true 
});
// 只失效: ["agents"]，不影响子级
```

### 6.9 动态 queryKey 使用示例

```tsx
// 页面分页
function AgentsList() {
  const [page, setPage] = useState(1);

  const { data } = useQuery({
    queryKey: ["agents", "list", { page }],
    queryFn: () => listAgents({ page }),
  });

  // page 变化 → queryKey 变化 → 获取新页数据
  // 每个 page 有独立缓存
}

// 搜索过滤
function SearchAgents() {
  const [searchTerm, setSearchTerm] = useState("");

  const { data } = useQuery({
    queryKey: ["agents", "search", searchTerm],
    queryFn: () => searchAgents(searchTerm),
    enabled: searchTerm.length >= 2,  // 至少输入 2 个字符才查询
  });

  // searchTerm 变化 → queryKey 变化 → 获取搜索结果
  // 不同搜索词有独立缓存
}

// 选中详情
function AgentPage({ agentId }: { agentId: string }) {
  // 列表缓存
  const { data: agents } = useQuery({
    queryKey: ["agents"],
    queryFn: listAgents,
  });

  // 单个缓存
  const { data: agent } = useQuery({
    queryKey: ["agents", agentId],
    queryFn: () => getAgent(agentId),
    enabled: !!agentId,
  });

  // agentId 变化 → 获取新 agent
  // 列表和单个是独立缓存
}
```

### 6.10 queryKey 总结表

| 概念 | 说明 |
|------|------|
| queryKey 是缓存地址 | 不同 key = 不同缓存位置 |
| 动态 queryKey | 每次渲染根据变量值计算 |
| key 变化 = 新查询 | 变化后检查新 key 的缓存 |
| 旧 key 缓存保留 | 切换回来时可使用缓存 |
| 层级失效 | 失效父级会影响子级 |
| exact: true | 只失效精确匹配的 key |

---

## 7. 缓存失效策略

### 7.1 常见失效场景

```tsx
// 创建 → 失效列表
useMutation({
  mutationFn: createAgent,
  onSuccess: () => {
    void queryClient.invalidateQueries({ queryKey: ["agents"] });
  },
});

// 更新 → 失效列表 + 单个
useMutation({
  mutationFn: ({ name, request }) => updateAgent(name, request),
  onSuccess: (_data, { name }) => {
    void queryClient.invalidateQueries({ queryKey: ["agents"] });
    void queryClient.invalidateQueries({ queryKey: ["agents", name] });
  },
});

// 删除 → 失效列表 + 删除单个缓存
useMutation({
  mutationFn: deleteAgent,
  onSuccess: (_data, name) => {
    void queryClient.invalidateQueries({ queryKey: ["agents"] });
    queryClient.removeQueries({ queryKey: ["agents", name] });
  },
});
```

### 7.2 失效策略表

| 操作 | 失效策略 | 说明 |
|------|---------|------|
| 创建 | 失效列表 | 新增数据，列表需要更新 |
| 更新 | 失效列表 + 单个 | 数据变化，列表和详情都需要更新 |
| 删除 | 失效列表 + 删除单个 | 数据删除，列表更新，删除详情缓存 |
| 批量操作 | 失效列表 | 影响多个，统一更新列表 |

### 7.3 乐观更新

```tsx
// 乐观更新：先更新 UI，后提交数据
const mutation = useMutation({
  mutationFn: updateAgent,
  
  // 提交前：乐观更新缓存
  onMutate: ({ name, request }) => {
    // 取消正在进行的查询，避免冲突
    void queryClient.cancelQueries({ queryKey: ["agents", name] });
    
    // 保存旧数据用于回滚
    const previousAgent = queryClient.getQueryData(["agents", name]);
    
    // 乐观更新
    queryClient.setQueryData(["agents", name], (old) => {
      if (!old) return old;
      return { ...old, ...request };
    });
    
    return { previousAgent };
  },
  
  // 成功：失效缓存（获取最新数据）
  onSuccess: (_data, { name }) => {
    void queryClient.invalidateQueries({ queryKey: ["agents", name] });
  },
  
  // 失败：回滚到旧数据
  onError: (err, { name }, context) => {
    queryClient.setQueryData(["agents", name], context.previousAgent);
  },
  
  // 完成：无论如何都重新获取
  onSettled: (_data, _err, { name }) => {
    void queryClient.invalidateQueries({ queryKey: ["agents", name] });
  },
});
```

---

## 8. 完整示例

### 8.1 标准 CRUD hooks

```tsx
// hooks.ts
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { listAgents, getAgent, createAgent, updateAgent, deleteAgent } from "./api";
import type { CreateAgentRequest, UpdateAgentRequest } from "./types";

// 查询列表
export function useAgents() {
  const { data, isLoading, error } = useQuery({
    queryKey: ["agents"],
    queryFn: () => listAgents(),
    staleTime: 60 * 1000,  // 1 分钟新鲜
  });
  return { agents: data ?? [], isLoading, error };
}

// 查询单个（条件查询）
export function useAgent(name: string | null | undefined) {
  const { data, isLoading, error } = useQuery({
    queryKey: ["agents", name],
    queryFn: () => getAgent(name!),
    enabled: !!name,  // 只有 name 存在才查询
  });
  return { agent: data ?? null, isLoading, error };
}

// 创建
export function useCreateAgent() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (request: CreateAgentRequest) => createAgent(request),
    onSuccess: () => {
      void queryClient.invalidateQueries({ queryKey: ["agents"] });
    },
  });
}

// 更新
export function useUpdateAgent() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ name, request }: { name: string; request: UpdateAgentRequest }) => 
      updateAgent(name, request),
    onSuccess: (_data, { name }) => {
      void queryClient.invalidateQueries({ queryKey: ["agents"] });
      void queryClient.invalidateQueries({ queryKey: ["agents", name] });
    },
  });
}

// 删除
export function useDeleteAgent() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (name: string) => deleteAgent(name),
    onSuccess: (_data, name) => {
      void queryClient.invalidateQueries({ queryKey: ["agents"] });
      queryClient.removeQueries({ queryKey: ["agents", name] });
    },
  });
}
```

### 8.2 组件使用

```tsx
// AgentGallery.tsx
"use client";

import { useAgents, useCreateAgent, useDeleteAgent } from "@/core/agents";
import { toast } from "sonner";

export function AgentGallery() {
  const { agents, isLoading, error } = useAgents();
  const createMutation = useCreateAgent();
  const deleteMutation = useDeleteAgent();

  async function handleCreate(name: string) {
    createMutation.mutate(
      { name, description: "" },
      {
        onSuccess: (agent) => {
          toast.success(`Agent ${agent.name} 创建成功`);
        },
        onError: (error) => {
          toast.error(error.message);
        },
      },
    );
  }

  async function handleDelete(name: string) {
    deleteMutation.mutate(name, {
      onSuccess: () => {
        toast.success("删除成功");
      },
    });
  }

  if (isLoading) return <Spinner />;
  if (error) return <Error error={error} />;

  return (
    <div>
      <button onClick={() => handleCreate("new-agent")}>创建</button>
      {agents.map(agent => (
        <div key={agent.name}>
          {agent.name}
          <button onClick={() => handleDelete(agent.name)}>删除</button>
        </div>
      ))}
    </div>
  );
}
```

---

## 9. 最佳实践

### 9.1 Query Key 管理

```tsx
// ✅ 使用工厂函数
const keys = {
  all: ["agents"],
  list: () => [...keys.all, "list"],
  detail: (id: string) => [...keys.all, "detail", id],
};

// ❌ 散乱定义
queryKey: ["agents"]
queryKey: ["agentList"]
queryKey: ["getAgent"]
```

### 9.2 条件查询

```tsx
// ✅ 使用 enabled
useQuery({
  queryKey: ["agent", name],
  queryFn: () => getAgent(name),
  enabled: !!name,
});

// ❌ 手动控制
if (name) {
  useQuery({ ... });
}
```

### 9.3 缓存失效

```tsx
// ✅ 使用 invalidateQueries
onSuccess: () => {
  void queryClient.invalidateQueries({ queryKey: ["agents"] });
};

// ❌ 手动重新获取
onSuccess: () => {
  await queryClient.refetchQueries({ queryKey: ["agents"] });
};
```

### 9.4 错误处理

```tsx
// ✅ 使用回调
mutation.mutate(data, {
  onError: (error) => {
    toast.error(error.message);
  },
});

// ❌ 忽略错误处理
mutation.mutate(data);
```

---

## 总结

### 核心 hooks

| Hook | 用途 | 触发方式 |
|------|------|---------|
| `useQuery` | 数据查询 | 自动 |
| `useMutation` | 数据变更 | 手动 |
| `useQueryClient` | 缓存管理 | 手动 |

### 关键概念

| 概念 | 说明 |
|------|------|
| `queryKey` | 缓存唯一标识 |
| `queryFn` | 数据获取函数 |
| `mutationFn` | 数据变更函数 |
| `enabled` | 条件查询控制 |
| `staleTime` | 数据新鲜时间 |
| `invalidateQueries` | 失效缓存 |

---

## 参考资源

- [TanStack Query 官方文档](https://tanstack.com/query)
- [useQuery API](https://tanstack.com/query/latest/docs/framework/react/reference/useQuery)
- [useMutation API](https://tanstack.com/query/latest/docs/framework/react/reference/useMutation)
- [QueryClient API](https://tanstack.com/query/latest/docs/framework/react/reference/QueryClient)