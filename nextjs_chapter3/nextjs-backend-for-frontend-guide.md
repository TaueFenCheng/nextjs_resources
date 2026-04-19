# Next.js Backend for Frontend (BFF) 完整指南

> 本文档总结自 Next.js 官方文档: https://nextjs.org/docs/app/guides/backend-for-frontend
> 最后更新: 2026年4月15日

## 目录

1. [概述](#概述)
2. [公共端点](#公共端点)
3. [内容类型](#内容类型)
4. [内容协商](#内容协商)
5. [消费请求体](#消费请求体)
6. [数据操作](#数据操作)
7. [代理到后端](#代理到后端)
8. [NextRequest 和 NextResponse](#nextrequest-和-nextresponse)
9. [Webhooks 和回调 URL](#webhooks-和回调-url)
10. [重定向](#重定向)
11. [Proxy](#proxy)
12. [安全](#安全)
13. [库模式](#库模式)
14. [注意事项](#注意事项)

---

## 概述

Next.js 支持 "Backend for Frontend" (BFF) 模式。这允许你创建公共端点来处理 HTTP 请求并返回任何内容类型——不仅仅是 HTML。你还可以访问数据源并执行副作用，如更新远程数据。

### 创建项目

如果你正在开始一个新项目，使用 `create-next-app` 带 `--api` 标志会自动在新的项目的 `app/` 目录中包含一个示例 `route.ts`，演示如何创建 API 端点：

```bash
pnpm create next-app --api
# 或
npm create next-app --api
# 或
yarn create next-app --api
# 或
bun create next-app --api
```

### 重要说明

Next.js 后端功能不是一个完整的后端替代品。它们作为一个 API 层：

- **公开可达**：任何人都可以访问
- **处理任何 HTTP 请求**：支持所有 HTTP 方法
- **可返回任何内容类型**：JSON、XML、HTML、图片等

### 实现方式

要实现 BFF 模式，可以使用：

- **Route Handlers** (`route.ts` 或 `route.js`)
- **Proxy** (`proxy.ts` 或 `proxy.js`)
- 在 Pages Router 中使用 **API Routes**

---

## 公共端点

Route Handlers 是公共 HTTP 端点。任何客户端都可以访问它们。

### 创建 Route Handler

使用 `route.ts` 或 `route.js` 文件约定创建 Route Handler：

```typescript
// app/api/route.ts
export function GET(request: Request) {}
```

这处理发送到 `/api` 的 GET 请求。

### 错误处理

使用 `try/catch` 块处理可能抛出异常的操作：

```typescript
// app/api/route.ts
import { submit } from '@/lib/submit'

export async function POST(request: Request) {
  try {
    await submit(request)
    return new Response(null, { status: 204 })
  } catch (reason) {
    const message =
      reason instanceof Error ? reason.message : 'Unexpected error'

    return new Response(message, { status: 500 })
  }
}
```

**重要**：避免在发送给客户端的错误消息中暴露敏感信息。

要限制访问，需要实现身份验证和授权。参见 Authentication 指南。

---

## 内容类型

Route Handlers 允许你提供非 UI 响应，包括 JSON、XML、图片、文件和纯文本。

### Next.js 文件约定

Next.js 使用文件约定来创建常见端点：

| 文件约定 | 用途 |
|---------|------|
| `sitemap.xml` | 网站地图 |
| `opengraph-image.jpg`, `twitter-image` | 社交媒体图片 |
| `favicon`, `app icon`, `apple-icon` | 图标文件 |
| `manifest.json` | Web 应用清单 |
| `robots.txt` | SEO 爬虫规则 |

### 自定义端点

你还可以定义自定义端点，例如：

- `llms.txt` - 为 AI 语言模型提供的内容
- `rss.xml` - RSS 订阅源
- `.well-known` - 标准化元数据

### RSS 订阅源示例

```typescript
// app/rss.xml/route.ts
export async function GET(request: Request) {
  const rssResponse = await fetch(/* rss endpoint */)
  const rssData = await rssResponse.json()

  const rssFeed = `<?xml version="1.0" encoding="UTF-8" ?>
<rss version="2.0">
<channel>
 <title>${rssData.title}</title>
 <description>${rssData.description}</description>
 <link>${rssData.link}</link>
 <copyright>${rssData.copyright}</copyright>
 ${rssData.items.map((item) => {
   return `<item>
    <title>${item.title}</title>
    <description>${item.description}</description>
    <link>${item.link}</link>
    <pubDate>${item.publishDate}</pubDate>
    <guid isPermaLink="false">${item.guid}</guid>
 </item>`
 })}
</channel>
</rss>`

  const headers = new Headers({ 'content-type': 'application/xml' })

  return new Response(rssFeed, { headers })
}
```

**重要**：清理用于生成标记的任何输入，防止 XSS 攻击。

---

## 内容协商

你可以使用带有头部匹配的重写（rewrites）来根据请求的 `Accept` 头部从同一 URL 提供不同的内容类型。这被称为内容协商。

### 示例：文档站点

一个文档站点可能向浏览器提供 HTML 页面，向 AI 代理提供原始 Markdown，使用相同的 `/docs/…` URL。

### 步骤 1：配置重写

配置匹配 `Accept` 头部的重写：

```javascript
// next.config.js
module.exports = {
  async rewrites() {
    return [
      {
        source: '/docs/:slug*',
        destination: '/docs/md/:slug*',
        has: [
          {
            type: 'header',
            key: 'accept',
            value: '(.*)text/markdown(.*)',
          },
        ],
      },
    ]
  },
}
```

当请求 `/docs/getting-started` 包含 `Accept: text/markdown` 时，重写将其路由到 `/docs/md/getting-started`。该路径的 Route Handler 返回 Markdown 响应。不发送 `text/markdown` 的客户端继续接收正常的 HTML 页面。

### 步骤 2：创建 Route Handler

```typescript
// app/docs/md/[...slug]/route.ts
import { getDocsMd, generateDocsStaticParams } from '@/lib/docs'

export async function generateStaticParams() {
  return generateDocsStaticParams()
}

export async function GET(_: Request, ctx: RouteContext<'/docs/md/[...slug]'>) {
  const { slug } = await ctx.params
  const mdDoc = await getDocsMd({ slug })

  if (mdDoc == null) {
    return new Response(null, { status: 404 })
  }

  return new Response(mdDoc, {
    headers: {
      'Content-Type': 'text/markdown; charset=utf-8',
      Vary: 'Accept',
    },
  })
}
```

**重要概念**：

- `Vary: Accept` 响应头部告诉缓存响应体依赖于 `Accept` 请求头部。没有它，共享缓存可能向浏览器提供缓存的 Markdown 响应（反之亦然）。大多数托管提供商已在缓存键中包含 `Accept` 头部，但显式设置 `Vary` 确保所有 CDN 和代理缓存的正确行为。
- `generateStaticParams` 让你在构建时预渲染 Markdown 变体，可以从边缘提供而无需每次请求都访问源服务器。

### 步骤 3：测试

```bash
# 返回 Markdown
curl -H "Accept: text/markdown" https://example.com/docs/getting-started

# 返回正常的 HTML 页面
curl https://example.com/docs/getting-started
```

**注意**：`/docs/md/...` 路由在没有重写的情况下仍然可以直接访问。如果你想限制它只通过重写提供，使用 `proxy` 来阻止不包含预期 `Accept` 头部的直接请求。

对于更高级的协商逻辑，可以使用 `proxy` 而不是重写来获得更多灵活性。

---

## 消费请求体

使用 `Request` 实例方法如 `.json()`, `.formData()`, 或 `.text()` 来访问请求体。

**注意**：GET 和 HEAD 请求不携带请求体。

### 示例：返回请求体

```typescript
// app/api/echo-body/route.ts
export async function POST(request: Request) {
  const res = await request.json()
  return Response.json({ res })
}
```

### 示例：发送邮件

```typescript
// app/api/send-email/route.ts
import { sendMail, validateInputs } from '@/lib/email-transporter'

export async function POST(request: Request) {
  const formData = await request.formData()
  const email = formData.get('email')
  const contents = formData.get('contents')

  try {
    await validateInputs({ email, contents })
    const info = await sendMail({ email, contents })

    return Response.json({ messageId: info.messageId })
  } catch (reason) {
    const message =
      reason instanceof Error ? reason.message : 'Unexpected exception'

    return new Response(message, { status: 500 })
  }
}
```

**重要**：在将数据传递给其他系统之前进行验证。

### 克隆请求

你只能读取请求体一次。如果需要再次读取，克隆请求：

```typescript
// app/api/clone/route.ts
export async function POST(request: Request) {
  try {
    const clonedRequest = request.clone()

    await request.body()
    await clonedRequest.body()
    await request.body() // 抛出错误！

    return new Response(null, { status: 204 })
  } catch {
    return new Response(null, { status: 500 })
  }
}
```

---

## 数据操作

Route Handlers 可以转换、过滤和聚合来自一个或多个数据源的数据。这保持了前端逻辑简洁，并避免暴露内部系统。

你还可以将繁重的计算卸载到服务器，减少客户端电池和数据使用。

### 示例：天气数据处理

```typescript
import { parseWeatherData } from '@/lib/weather'

export async function POST(request: Request) {
  const body = await request.json()
  const searchParams = new URLSearchParams({ lat: body.lat, lng: body.lng })

  try {
    const weatherResponse = await fetch(`${weatherEndpoint}?${searchParams}`)

    if (!weatherResponse.ok) {
      /* handle error */
    }

    const weatherData = await weatherResponse.text()
    const payload = parseWeatherData.asJSON(weatherData)

    return new Response(payload, { status: 200 })
  } catch (reason) {
    const message =
      reason instanceof Error ? reason.message : 'Unexpected exception'

    return new Response(message, { status: 500 })
  }
}
```

**重要**：此示例使用 POST 来避免将地理位置数据放在 URL 中。GET 请求可能被缓存或记录，这可能暴露敏感信息。

---

## 代理到后端

你可以使用 Route Handler 作为到另一个后端的代理。在转发请求之前添加验证逻辑。

### 动态代理示例

```typescript
// app/api/[...slug]/route.ts
import { isValidRequest } from '@/lib/utils'

export async function POST(request: Request, { params }) {
  const clonedRequest = request.clone()
  const isValid = await isValidRequest(clonedRequest)

  if (!isValid) {
    return new Response(null, { status: 400, statusText: 'Bad Request' })
  }

  const { slug } = await params
  const pathname = slug.join('/')
  const proxyURL = new URL(pathname, 'https://nextjs.org')
  const proxyRequest = new Request(proxyURL, request)

  try {
    return fetch(proxyRequest)
  } catch (reason) {
    const message =
      reason instanceof Error ? reason.message : 'Unexpected exception'

    return new Response(message, { status: 500 })
  }
}
```

### 其他方式

你也可以使用：

- **Proxy** (`proxy.ts`)
- **Rewrites**（在 `next.config.js` 中配置）

---

## NextRequest 和 NextResponse

Next.js 扩展了 `Request` 和 `Response` Web API，提供了简化常见操作的方法。这些扩展在 Route Handlers 和 Proxy 中都可用。

### 功能概述

两者都提供了读取和操作 cookies 的方法：

- **NextRequest**：包含 `nextUrl` 属性，暴露来自传入请求的解析值，例如更容易访问请求路径名和搜索参数。
- **NextResponse**：提供辅助方法如 `next()`, `json()`, `redirect()`, 和 `rewrite()`。

你可以将 `NextRequest` 传递给任何期望 `Request` 的函数。同样，你可以在期望 `Response` 的地方返回 `NextResponse`。

### 示例

```typescript
// app/echo-pathname/route.ts
import { type NextRequest, NextResponse } from 'next/server'

export async function GET(request: NextRequest) {
  const nextUrl = request.nextUrl

  if (nextUrl.searchParams.get('redirect')) {
    return NextResponse.redirect(new URL('/', request.url))
  }

  if (nextUrl.searchParams.get('rewrite')) {
    return NextResponse.rewrite(new URL('/', request.url))
  }

  return NextResponse.json({ pathname: nextUrl.pathname })
}
```

了解更多关于 `NextRequest` 和 `NextResponse`。

---

## Webhooks 和回调 URL

使用 Route Handlers 接收来自第三方应用程序的事件通知。

### Webhook 示例

例如，当 CMS 中的内容更改时重新验证路由：

1. 配置 CMS 在更改时调用特定端点：

```typescript
// app/webhook/route.ts
import { type NextRequest, NextResponse } from 'next/server'

export async function GET(request: NextRequest) {
  const token = request.nextUrl.searchParams.get('token')

  if (token !== process.env.REVALIDATE_SECRET_TOKEN) {
    return NextResponse.json({ success: false }, { status: 401 })
  }

  const tag = request.nextUrl.searchParams.get('tag')

  if (!tag) {
    return NextResponse.json({ success: false }, { status: 400 })
  }

  revalidateTag(tag)
  return NextResponse.json({ success: true })
}
```

### 回调 URL 示例

当用户完成第三方流程后，第三方将他们发送到回调 URL。使用 Route Handler 验证响应并决定将用户重定向到哪里：

```typescript
// app/auth/callback/route.ts
import { type NextRequest, NextResponse } from 'next/server'

export async function GET(request: NextRequest) {
  const token = request.nextUrl.searchParams.get('session_token')
  const redirectUrl = request.nextUrl.searchParams.get('redirect_url')

  const response = NextResponse.redirect(new URL(redirectUrl, request.url))

  response.cookies.set({
    value: token,
    name: '_token',
    path: '/',
    secure: true,
    httpOnly: true,
    expires: undefined, // session cookie
  })

  return response
}
```

---

## 重定向

在 Route Handler 中使用 `redirect` 函数：

```typescript
// app/api/route.ts
import { redirect } from 'next/navigation'

export async function GET(request: Request) {
  redirect('https://nextjs.org/')
}
```

了解更多关于 `redirect` 和 `permanentRedirect`。

---

## Proxy

每个项目只允许一个 `proxy` 文件。使用 `config.matcher` 来针对特定路径。

### 认证代理示例

使用 `proxy` 在请求到达路由路径之前生成响应：

```typescript
// proxy.ts
import { isAuthenticated } from '@lib/auth'

export const config = {
  matcher: '/api/:function*',
}

export function proxy(request: Request) {
  if (!isAuthenticated(request)) {
    return Response.json(
      { success: false, message: 'authentication failed' },
      { status: 401 }
    )
  }
}
```

### 重写代理

你也可以使用 `proxy` 来重写请求：

```typescript
// proxy.ts
import { NextResponse } from 'next/server'

export function proxy(request: Request) {
  if (request.nextUrl.pathname === '/proxy-this-path') {
    const rewriteUrl = new URL('https://nextjs.org')
    return NextResponse.rewrite(rewriteUrl)
  }
}
```

### 重定向代理

`proxy` 还可以产生重定向响应：

```typescript
// proxy.ts
import { NextResponse } from 'next/server'

export function proxy(request: Request) {
  if (request.nextUrl.pathname === '/v1/docs') {
    request.nextUrl.pathname = '/v2/docs'
    return NextResponse.redirect(request.nextUrl)
  }
}
```

了解更多关于 `proxy`。

---

## 安全

### 处理头部

要谨慎处理头部的去向，避免直接将传入请求头部传递到传出响应。

**上游请求头部**：

在 `proxy` 中，`NextResponse.next({ request: { headers } })` 修改你的服务器接收的头部，不会暴露给客户端。

**响应头部**：

以下方式将头部发送回客户端：

- `new Response(..., { headers })`
- `NextResponse.json(..., { headers })`
- `NextResponse.next({ headers })`
- `response.headers.set(...)`

如果敏感值被添加到这些头部，它们将对客户端可见。

了解更多关于 Proxy 中的 NextResponse 头部。

### 速率限制

你可以在 Next.js 后端实现速率限制。除了代码检查之外，启用托管提供商提供的任何速率限制功能。

```typescript
// app/resource/route.ts
import { NextResponse } from 'next/server'
import { checkRateLimit } from '@/lib/rate-limit'

export async function POST(request: Request) {
  const { rateLimited } = await checkRateLimit(request)

  if (rateLimited) {
    return NextResponse.json({ error: 'Rate limit exceeded' }, { status: 429 })
  }

  return new Response(null, { status: 204 })
}
```

### 验证请求体

**永远不要信任传入的请求数据**：

- 验证内容类型和大小
- 在使用前对 XSS 进行清理
- 使用超时防止滥用并保护服务器资源
- 将用户生成的静态资产存储在专用服务中
- 尽可能从浏览器上传，并将返回的 URI 存储在数据库中以减少请求大小

### 访问受保护资源

- 在授予访问权限之前始终验证凭据
- 不要单独依赖 `proxy` 进行身份验证和授权
- 从响应和后端日志中删除敏感或不必要的数据
- 定期轮换凭据和 API 密钥

### 预检请求

预检请求使用 `OPTIONS` 方法询问服务器是否根据来源、方法和头部允许请求。

如果 `OPTIONS` 未定义，Next.js 会自动添加它，并根据其他定义的方法设置 `Allow` 头部。

---

## 库模式

社区库通常使用工厂模式来创建 Route Handlers。

### Route Handler 工厂

```typescript
// app/api/[...path]/route.ts
import { createHandler } from 'third-party-library'

const handler = createHandler({
  /* library-specific options */
})

export const GET = handler
// 或 export { handler as POST }
```

这创建了 GET 和 POST 请求的共享处理器。库根据请求中的方法和路径名自定义行为。

### Proxy 工厂

库还可以提供 `proxy` 工厂：

```typescript
// proxy.ts
import { createMiddleware } from 'third-party-library'

export default createMiddleware()
```

**注意**：第三方库可能仍然将 `proxy` 称为 `middleware`。

---

## 注意事项

### Server Components

**在 Server Components 中直接从数据源获取数据，而不是通过 Route Handlers。**

对于在构建时预渲染的 Server Components，使用 Route Handlers 会导致构建步骤失败。因为在构建时没有服务器监听这些请求。

对于按需渲染的 Server Components，从 Route Handlers 获取数据较慢，由于处理器和渲染过程之间的额外 HTTP 循环。

服务器端获取请求使用绝对 URL。这意味着 HTTP 循环，到外部服务器。在开发过程中，你自己的开发服务器充当外部服务器。在构建时没有服务器，在运行时，服务器可通过你的公共域访问。

### 客户端数据获取

Server Components 满足大多数数据获取需求。然而，客户端数据获取可能是必要的：

- **依赖于仅客户端 Web API 的数据**：
  - 地理位置API
  - 存储 API
  - 音频 API
  - 文件 API
- **频繁轮询的数据**

对于这些情况，使用社区库如 `swr` 或 `react-query`。

### Server Actions

Server Actions 让你从客户端运行服务器端代码。它们的主要目的是从前端客户端变更数据。

**Server Actions 是排队的**。使用它们进行数据获取会引入顺序执行。

### export mode

`export` mode 输出没有运行时服务器的静态站点。

需要 Next.js 运行时的功能不被支持，因为此模式产生静态站点，没有运行时服务器。

在 `export` mode 中，只支持 **GET Route Handlers**，结合 `dynamic` 路由段配置设置为 `'force-static'`。

这可用于生成静态 HTML、JSON、TXT 或其他文件：

```typescript
// app/hello-world/route.ts
export const dynamic = 'force-static'

export function GET() {
  return new Response('Hello World', { status: 200 })
}
```

### 部署环境

某些主机将 Route Handlers 部署为 lambda 函数。这意味着：

- **Route Handlers 不能在请求之间共享数据**
- **环境可能不支持写入文件系统**
- **长时间运行的处理器可能因超时而被终止**
- **WebSockets 无法工作**，因为连接在超时后关闭，或在响应生成后关闭

---

## API 参考

了解更多关于相关 API：

| API | 说明 |
|-----|------|
| [route.js](https://nextjs.org/docs/app/api-reference/file-system-conventions/route) | `route.js` 特殊文件的 API 参考 |
| [proxy.js](https://nextjs.org/docs/app/api-reference/file-system-conventions/proxy) | `proxy.js` 文件的 API 参考 |
| [rewrites](https://nextjs.org/docs/app/api-reference/config/next-config-js/rewrites) | 在 Next.js 应用中添加重写 |

---

## 更多示例

查看更多使用 Route Handlers 和 `proxy` API 参考的示例，包括：

- Cookies 操作
- Headers 处理
- Streaming
- Proxy 负匹配
- 其他有用的代码片段

---

## 总结

Next.js Backend for Frontend (BFF) 模式提供了强大的 API 层功能：

1. **公共端点**：通过 `route.ts` 创建可公开访问的 HTTP 端点
2. **灵活的内容类型**：支持 JSON、XML、图片、文件等多种响应格式
3. **内容协商**：根据请求头部动态提供不同内容
4. **代理功能**：可以作为其他后端服务的代理层
5. **安全特性**：速率限制、请求验证、头部管理等
6. **Webhook 支持**：接收第三方事件通知和回调

**最佳实践**：

- 直接在 Server Components 中获取数据，避免通过 Route Handlers 的额外 HTTP 循环
- 验证所有传入数据，防止 XSS 和其他安全漏洞
- 使用 POST 方法处理敏感数据，避免 URL 泄露
- 实现适当的错误处理，不暴露敏感信息
- 考虑部署环境的限制（如 lambda 函数的超时、无文件系统写入等）