# Hono Code Review Guide

Hono 审查重点：路由设计、中间件组合、请求验证（Zod）、错误处理、类型安全 RPC、认证安全、以及大型应用的模块拆分。

> **框架版本**：Hono v4+（适用于 Cloudflare Workers / Bun / Deno / Node.js / Vercel Edge 等所有运行时）

## 目录

- [路由设计](#路由设计)
- [Context 与请求响应](#context-与请求响应)
- [中间件](#中间件)
- [请求验证（Zod）](#请求验证zod)
- [错误处理](#错误处理)
- [类型安全 RPC](#类型安全-rpc)
- [认证与安全中间件](#认证与安全中间件)
- [应用拆分与模块化](#应用拆分与模块化)
- [Review Checklists](#review-checklists)

---

## 路由设计

### 路由注册顺序

```ts
// ❌ 通配符在前，精确路由永远不会被触发
app.get('*', (c) => c.text('catch all'))
app.get('/foo', (c) => c.text('foo'))  // 永远不会执行

// ✅ 精确路由在前，通配/兜底在后
app.get('/foo', (c) => c.text('foo'))
app.get('*', (c) => c.text('catch all'))
```

### 路径参数

```ts
// ✅ 单个参数
app.get('/user/:name', (c) => {
  const name = c.req.param('name')
  return c.text(`Hello, ${name}!`)
})

// ✅ 多个参数——推荐解构形式
app.get('/posts/:postId/comments/:commentId', (c) => {
  const { postId, commentId } = c.req.param()
  return c.json({ postId, commentId })
})

// ✅ 可选参数
app.get('/api/animal/:type?', (c) => {
  const type = c.req.param('type') ?? 'unknown'
  return c.text(`Animal type: ${type}`)
})

// ✅ 正则约束参数
app.get('/post/:date{[0-9]+}/:title{[a-z]+}', (c) => {
  const { date, title } = c.req.param()
  return c.json({ date, title })
})
```

### Query 参数

```ts
// ✅ 单个查询参数
app.get('/search', (c) => {
  const q = c.req.query('q')
  return c.json({ q })
})

// ✅ 同名多值参数（?tags=A&tags=B）
app.get('/filter', (c) => {
  const tags = c.req.queries('tags')  // string[]
  return c.json({ tags })
})

// ❌ 手动解析 URL 字符串
app.get('/search', (c) => {
  const url = new URL(c.req.url)
  const q = url.searchParams.get('q')  // 不必要的手动操作
  return c.json({ q })
})
```

---

## Context 与请求响应

### Context 变量（c.set / c.get）

```ts
// ❌ 不推荐：在 Context 上挂裸属性（无类型约束）
app.use(async (c, next) => {
  (c as any).user = { id: '123' }  // 类型不安全
  await next()
})

// ✅ 定义 Variables 类型，使用 c.set / c.get
type Variables = {
  user: { id: string; name: string }
}

const app = new Hono<{ Variables: Variables }>()

app.use(async (c, next) => {
  c.set('user', { id: '123', name: 'Alice' })
  await next()
})

app.get('/profile', (c) => {
  const user = c.get('user')  // 类型正确：{ id: string; name: string }
  return c.json(user)
})
```

### 响应类型

```ts
// ✅ JSON 响应（最常见）
app.get('/api/users', (c) => {
  return c.json({ users: [] })
})

// ✅ 带状态码
app.post('/api/users', (c) => {
  return c.json({ id: 1 }, 201)
})

// ✅ 文本响应
app.get('/ping', (c) => c.text('pong'))

// ✅ 重定向
app.get('/old', (c) => c.redirect('/new', 301))

// ❌ 直接 new Response — 丢失 Hono 的类型推断与 RPC 能力
app.get('/api', (c) => {
  return new Response(JSON.stringify({ ok: true }), {
    headers: { 'Content-Type': 'application/json' },
  })
})
```

### 请求体读取

```ts
// ✅ 读取 JSON body（搭配 validator 使用时不需要手动 await）
app.post('/api/posts', async (c) => {
  const body = await c.req.json<{ title: string; content: string }>()
  return c.json({ received: body })
})

// ✅ 读取 FormData
app.post('/upload', async (c) => {
  const body = await c.req.formData()
  const file = body.get('file')
  return c.text('ok')
})
```

---

## 中间件

### 中间件顺序

```ts
// ✅ 中间件必须 use 在对应路由之前
app.use('/api/*', authMiddleware)   // 先注册
app.get('/api/users', handler)      // 后注册，才会被 authMiddleware 保护

// ❌ 顺序错误——路由先于中间件注册，中间件不会执行
app.get('/api/users', handler)
app.use('/api/*', authMiddleware)   // 对上面的路由无效！
```

### 创建自定义中间件

```ts
import { createMiddleware } from 'hono/factory'

// ✅ 带参数的中间件工厂
const rateLimit = (limit: number) => {
  const requests = new Map<string, number>()

  return createMiddleware(async (c, next) => {
    const ip = c.req.header('CF-Connecting-IP') ?? 'unknown'
    const count = requests.get(ip) ?? 0

    if (count >= limit) {
      return c.json({ error: 'Rate limit exceeded' }, 429)
    }

    requests.set(ip, count + 1)
    await next()
  })
}

// ✅ 注入类型安全变量的认证中间件
type Variables = { user: { id: string; name: string } }

const auth = createMiddleware<{ Variables: Variables }>(async (c, next) => {
  const token = c.req.header('Authorization')
  if (!token) {
    return c.json({ error: 'Unauthorized' }, 401)
  }
  // 验证 token 并注入用户信息
  c.set('user', { id: '123', name: 'John' })
  await next()
})

const app = new Hono<{ Variables: Variables }>()
app.use('/api/*', rateLimit(100))
app.use('/protected/*', auth)
```

### 响应计时中间件示例

```ts
// ✅ 在 next() 之后修改响应头
const timing = createMiddleware(async (c, next) => {
  const start = Date.now()
  await next()
  c.res.headers.set('X-Response-Time', `${Date.now() - start}ms`)
})
```

### 动态配置内置中间件（Context-aware）

```ts
import { cors } from 'hono/cors'

// ✅ 从 c.env 读取配置，动态传入内置中间件
app.use('*', async (c, next) => {
  const middleware = cors({ origin: c.env.CORS_ORIGIN })
  return middleware(c, next)
})

// ❌ 硬编码 origin（在多环境部署时不灵活）
app.use('*', cors({ origin: 'https://example.com' }))
```

---

## 请求验证（Zod）

### 基础用法

```ts
import { zValidator } from '@hono/zod-validator'
import { z } from 'zod'

// ✅ 验证 JSON body
const createUserSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
})

app.post('/users', zValidator('json', createUserSchema), (c) => {
  const { name, email } = c.req.valid('json')  // 完全类型安全
  return c.json({ name, email }, 201)
})

// ✅ 验证 query 参数
app.get('/posts', zValidator('query', z.object({
  page: z.coerce.number().min(1).default(1),
  limit: z.coerce.number().max(100).default(20),
})), (c) => {
  const { page, limit } = c.req.valid('query')
  return c.json({ page, limit })
})

// ✅ 验证路径参数
app.get('/posts/:id', zValidator('param', z.object({
  id: z.coerce.number().int().positive(),
})), (c) => {
  const { id } = c.req.valid('param')
  return c.json({ id })
})
```

### 自定义验证错误响应

```ts
// ❌ 默认 400 响应格式不统一，不利于客户端解析
app.post('/users', zValidator('json', userSchema), handler)

// ✅ 自定义错误响应格式
app.post('/users',
  zValidator('json', userSchema, (result, c) => {
    if (!result.success) {
      return c.json({
        error: 'Validation failed',
        issues: result.error.issues,
      }, 422)
    }
  }),
  (c) => {
    const data = c.req.valid('json')
    return c.json(data, 201)
  }
)
```

### 同一路由多目标验证

```ts
// ✅ 同时验证 param + query + json
app.put('/posts/:id',
  zValidator('param', z.object({ id: z.coerce.number() })),
  zValidator('json', z.object({
    title: z.string().min(1),
    content: z.string(),
  })),
  (c) => {
    const { id } = c.req.valid('param')
    const { title, content } = c.req.valid('json')
    return c.json({ id, title, content })
  }
)
```

---

## 错误处理

### onError / notFound 全局处理

```ts
import { Hono } from 'hono'
import { HTTPException } from 'hono/http-exception'

const app = new Hono()

// ✅ 自定义 404
app.notFound((c) => {
  return c.json({
    error: 'Not Found',
    path: c.req.path,
  }, 404)
})

// ✅ 全局错误处理——区分 HTTPException 与未知错误
app.onError((err, c) => {
  if (err instanceof HTTPException) {
    return err.getResponse()  // 返回 HTTPException 内置的响应
  }
  console.error(err)
  return c.json({ error: 'Internal Server Error' }, 500)
})
```

### 抛出 HTTPException

```ts
// ✅ 在业务逻辑中抛出受控错误
app.get('/items/:id', async (c) => {
  const id = c.req.param('id')
  const item = await db.findItem(id)

  if (!item) {
    throw new HTTPException(404, { message: 'Item not found' })
  }

  return c.json(item)
})

// ✅ 需要自定义响应头时使用 res 选项
app.get('/secure', async (c) => {
  const token = c.req.header('Authorization')
  if (!token) {
    throw new HTTPException(401, {
      res: new Response('Unauthorized', {
        status: 401,
        headers: { 'WWW-Authenticate': 'Bearer realm="api"' },
      }),
    })
  }
  return c.text('ok')
})

// ✅ 链式错误追踪（保留原始 cause）
app.post('/login', async (c) => {
  try {
    await authorize(c)
  } catch (cause) {
    throw new HTTPException(401, { message: 'Invalid credentials', cause })
  }
  return c.redirect('/')
})

// ❌ 直接返回错误状态而不抛出——绕过全局错误处理器
app.get('/users/:id', async (c) => {
  const user = await getUser(c.req.param('id'))
  if (!user) return c.json({ error: 'not found' }, 404)  // 格式可能不统一
  return c.json(user)
})
```

---

## 类型安全 RPC

### 服务端定义（链式写法 + 导出类型）

```ts
// server.ts
import { Hono } from 'hono'
import { zValidator } from '@hono/zod-validator'
import { z } from 'zod'

// ✅ 必须使用链式写法，否则 RPC 类型推断失效
const app = new Hono()
  .get('/posts', (c) => {
    return c.json({ posts: [{ id: 1, title: 'Hello' }] })
  })
  .get('/posts/:id', (c) => {
    const id = c.req.param('id')
    return c.json({ id, title: `Post ${id}` })
  })
  .post('/posts',
    zValidator('json', z.object({ title: z.string(), body: z.string() })),
    (c) => {
      const { title, body } = c.req.valid('json')
      return c.json({ id: 3, title, body }, 201)
    }
  )

export type AppType = typeof app  // 导出类型供客户端使用
export default app
```

### 客户端调用

```ts
// client.ts
import { hc } from 'hono/client'
import type { InferRequestType, InferResponseType } from 'hono/client'
import type { AppType } from './server'

const client = hc<AppType>('http://localhost:8787')

// ✅ GET 请求
const res = await client.posts.$get()
if (res.ok) {
  const data = await res.json()
  console.log(data.posts)  // 类型正确：{ id: number; title: string }[]
}

// ✅ 带路径参数的 GET
const postRes = await client.posts[':id'].$get({ param: { id: '1' } })

// ✅ POST 请求（JSON body 有类型校验）
const createRes = await client.posts.$post({
  json: { title: 'New Post', body: 'Content' },
})

// ✅ 导出请求/响应类型（供前端组件复用）
type CreatePostRequest = InferRequestType<typeof client.posts.$post>['json']
type CreatePostResponse = InferResponseType<typeof client.posts.$post>
```

### 大型项目：hcWithType 优化 tsserver 性能

```ts
// client-factory.ts
import { app } from './app'
import { hc } from 'hono/client'

// ✅ 编译期计算类型，避免 tsserver 每次重复推断
export type Client = ReturnType<typeof hc<typeof app>>

export const hcWithType = (...args: Parameters<typeof hc>): Client =>
  hc<typeof app>(...args)
```

### 常见 RPC 陷阱

```ts
// ❌ 非链式写法导致 RPC 类型推断错误
const app = new Hono()
app.get('/posts', handlerA)   // 不链式
app.post('/posts', handlerB)  // 无法正确推断完整 AppType

// ✅ 链式写法
const app = new Hono()
  .get('/posts', handlerA)
  .post('/posts', handlerB)

export type AppType = typeof app
```

---

## 认证与安全中间件

### Bearer Token 认证

```ts
import { bearerAuth } from 'hono/bearer-auth'

// ✅ 静态 token 保护路由前缀
app.use('/api/*', bearerAuth({ token: 'secret-token' }))

// ✅ 动态 token 验证（查数据库或 JWT 解码）
app.use('/api/*', bearerAuth({
  verifyToken: async (token, c) => {
    const apiKey = await db.apiKeys.findUnique({ where: { key: token } })
    return apiKey !== null && apiKey.active
  },
}))

// ❌ 只保护了 POST，忘记保护写操作（安全遗漏）
app.post('/api/posts', bearerAuth({ token: 'secret' }), createPost)
// GET 没有认证，但可能包含敏感数据

// ✅ 在路由前缀统一保护所有方法
app.use('/api/*', bearerAuth({ token: 'secret' }))
```

### JWT 认证

```ts
import { jwt } from 'hono/jwt'

// ✅ 从环境变量获取密钥（不硬编码）
app.use('/api/*', async (c, next) => {
  const jwtMiddleware = jwt({ secret: c.env.JWT_SECRET })
  return jwtMiddleware(c, next)
})

// ❌ 硬编码密钥（安全反模式）
app.use('/api/*', jwt({ secret: 'hardcoded-secret' }))
```

### CORS

```ts
import { cors } from 'hono/cors'

// ✅ 明确指定允许的来源（不使用 * 通配）
app.use('/api/*', cors({
  origin: ['https://app.example.com', 'https://admin.example.com'],
  allowMethods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400,
}))

// ❌ 允许所有来源——生产环境安全风险
app.use('*', cors({ origin: '*' }))
```

### Secure Headers

```ts
import { secureHeaders } from 'hono/secure-headers'

// ✅ 自动设置安全相关响应头（CSP、HSTS 等）
app.use('*', secureHeaders())
```

---

## 应用拆分与模块化

### 基本路由分组

```ts
// books.ts
import { Hono } from 'hono'

const app = new Hono()

app.get('/', (c) => c.json({ books: [] }))
app.get('/:id', (c) => c.json({ id: c.req.param('id') }))
app.post('/', (c) => c.json({ message: 'Book created' }, 201))

export default app
```

```ts
// index.ts
import { Hono } from 'hono'
import books from './books'
import users from './users'

const app = new Hono()

app.route('/books', books)  // 挂载到 /books 前缀
app.route('/users', users)

export default app
```

### RPC 场景下的链式写法

```ts
// authors.ts — RPC 场景必须链式写法才能正确导出类型
const app = new Hono()
  .get('/', (c) => c.json({ authors: [] }))
  .post('/', (c) => c.json({ message: 'created' }, 201))
  .get('/:id', (c) => c.json({ id: c.req.param('id') }))

export default app
export type AppType = typeof app  // 导出供 hc() 使用
```

```ts
// index.ts
const routes = app.route('/authors', authors).route('/books', books)
export type RootAppType = typeof routes
```

### API 版本控制

```ts
// ✅ 使用 basePath 实现版本前缀
const v1 = new Hono().basePath('/api/v1')
v1.get('/status', (c) => c.json({ version: 'v1' }))

const v2 = new Hono().basePath('/api/v2')
v2.get('/status', (c) => c.json({ version: 'v2' }))

const app = new Hono()
app.route('/', v1)
app.route('/', v2)

export default app
```

---

## Review Checklists

### 路由

- [ ] 通配路由（`*`）在精确路由之后注册
- [ ] 中间件注册在被保护路由之前
- [ ] 路径参数使用 `c.req.param()` 而非手动解析 URL
- [ ] 多值 query 使用 `c.req.queries()` 而非 `c.req.query()`

### Context 与响应

- [ ] Context 变量通过泛型 `Variables` 类型约束，不使用 `any` 转型
- [ ] 响应使用 `c.json()` / `c.text()` / `c.redirect()`，避免裸 `new Response()`
- [ ] JSON 响应体包含一致的结构（成功/错误格式统一）
- [ ] 不在 handler 之外直接访问 `c.req.json()`（只能读一次）

### 中间件

- [ ] 自定义中间件使用 `createMiddleware` 工厂函数（有类型约束）
- [ ] 中间件必须 `await next()` 或 return Response（不遗漏）
- [ ] 带参数的中间件使用闭包工厂（如 `rateLimit(100)`）
- [ ] 内置中间件的密钥/配置从 `c.env` 取，不硬编码

### 请求验证

- [ ] 每个写操作（POST / PUT / PATCH）都有 `zValidator` 验证 body
- [ ] 路径参数中需要数字/枚举的使用 `z.coerce.number()` 或 `.enum()`
- [ ] 验证通过后使用 `c.req.valid()` 而非再次 `await c.req.json()`
- [ ] 自定义错误格式统一（状态码 422 + issues 字段）

### 错误处理

- [ ] 根应用定义了 `app.onError()` 全局错误处理器
- [ ] 根应用定义了 `app.notFound()` 自定义 404
- [ ] 受控错误通过 `throw new HTTPException(status, { message })` 抛出
- [ ] `onError` 中区分 `HTTPException` 与未知错误，防止信息泄露

### 类型安全 RPC

- [ ] 服务端路由使用链式写法（`.get().post()...`）
- [ ] 导出 `AppType = typeof app`（或 `typeof routes`）
- [ ] 客户端通过 `hc<AppType>()` 创建，不使用裸 fetch
- [ ] 大型项目使用 `hcWithType` 优化 tsserver 性能
- [ ] 不直接共享 runtime 代码至客户端（只 import `type`）

### 认证与安全

- [ ] 敏感路由组统一接 Bearer/JWT 中间件，不单独保护单个路由
- [ ] JWT 密钥从 `c.env` 或环境变量读取，不硬编码
- [ ] CORS 明确指定 `origin` 白名单，禁止 `origin: '*'` 用于生产
- [ ] 全局使用 `secureHeaders()` 中间件

### 应用结构

- [ ] 不同业务域拆分为独立 `Hono` 实例，通过 `app.route()` 组合
- [ ] 存在 API 版本时使用 `basePath()` 管理版本前缀
- [ ] RPC 场景的子模块使用链式写法并导出 `AppType`
- [ ] 全局中间件（日志、鉴权）在子应用挂载之前注册
