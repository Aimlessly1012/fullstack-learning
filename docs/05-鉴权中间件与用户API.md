# 05 - 鉴权中间件与用户 API

## 📖 本课目标

- 深入理解 Next.js Middleware 工作原理
- 掌握 Edge Runtime 和 Node.js Runtime 的区别
- 实现全局 JWT 鉴权中间件（白名单设计）
- 理解为什么用 jose 而不是 jsonwebtoken
- 实现用户列表 API（分页、搜索、关联查询）
- 掌握 Prisma select 安全字段过滤
- 学习中间件注入请求头的技巧（x-user-id、x-user-email）

---

## 🤔 为什么这样设计？

### 技术选型理由

#### 1. 为什么要用中间件做鉴权？

**场景对比：**

```typescript
// ❌ 不用中间件：每个 API 都要复制鉴权代码
export async function GET(req: NextRequest) {
  const token = req.cookies.get('token')?.value
  if (!token) return NextResponse.json({ error: '未登录' }, { status: 401 })
  
  try {
    const decoded = jwt.verify(token, secret)
    // ... 业务逻辑
  } catch {
    return NextResponse.json({ error: 'token 无效' }, { status: 401 })
  }
}
// 每个 API 都要写这一堆 → 重复代码太多！

// ✅ 用中间件：一处验证，全局生效
// middleware.ts 统一拦截所有 /api/* 请求
// API 只需专注业务逻辑，拿到的请求已经是验证过的
```

**优势：**
- **代码复用**：鉴权逻辑只写一次
- **集中管理**：白名单、黑名单统一配置
- **性能优化**：中间件运行在 Edge Runtime，比 Node.js 快
- **安全性**：统一拦截，不会漏掉某个 API

---

#### 2. 为什么用 jose 而不是 jsonwebtoken？

```typescript
// ❌ jsonwebtoken 在 Edge Runtime 会报错
import jwt from 'jsonwebtoken'  // Error: crypto.createPrivateKey is not a function

// ✅ jose 是纯 Web API 实现，兼容 Edge Runtime
import { jwtVerify } from 'jose'  // ✅ 正常工作
```

**技术背景：**

| 特性 | jsonwebtoken | jose |
|------|-------------|------|
| 依赖 | Node.js 原生 crypto 模块 | 纯 Web Crypto API |
| 运行环境 | ❌ 只能在 Node.js Runtime | ✅ Edge Runtime + Node.js |
| 性能 | 普通 | 更快（边缘计算） |
| 包大小 | 较大 | 较小 |

**什么是 Edge Runtime？**
```
传统 Node.js Runtime：
客户端 → 服务器机房（可能在北京）→ 执行代码 → 返回
延迟：可能 200ms+

Edge Runtime：
客户端 → 最近的边缘节点（全球分布）→ 执行代码 → 返回
延迟：可能只要 20ms

Next.js Middleware 默认运行在 Edge Runtime，全球部署，超低延迟！
```

**限制：**
- ❌ 不能用 Node.js 原生模块（fs、crypto、path 等）
- ❌ 不能连接数据库（Prisma 不可用）
- ✅ 只能用 Web 标准 API（fetch、Headers、Request、Response）

---

#### 3. 为什么注入请求头而不是传参？

```typescript
// ❌ 方案 1：中间件返回用户信息（不可行）
// Edge Runtime 不能修改请求 body，只能改 headers

// ❌ 方案 2：每个 API 重新验证 token
export async function GET(req: NextRequest) {
  const token = req.cookies.get('token')?.value
  const user = jwt.verify(token, secret)  // 重复验证，浪费性能
}

// ✅ 方案 3：中间件注入请求头，API 直接读取
// middleware.ts
requestHeaders.set('x-user-id', payload.userId)

// route.ts
const userId = req.headers.get('x-user-id')  // 直接拿到，无需再验证
```

**优势：**
- **性能优化**：token 只验证一次
- **代码简洁**：API 不需要关心鉴权细节
- **类型安全**：可以扩展 NextRequest 类型

---

## 🧠 核心概念大白话解释

### Next.js Middleware 执行流程

```
用户请求 /api/users
    ↓
[1] 进入 middleware.ts（在 Edge Runtime 执行）
    ↓
[2] 检查路径是否在白名单
    ├─ 是 → 直接放行（跳到步骤 4）
    └─ 否 → 继续
    ↓
[3] 验证 JWT token
    ├─ 无效 → 返回 401 错误（终止请求）
    └─ 有效 → 注入 x-user-id 到请求头
    ↓
[4] 请求到达 /api/users/route.ts
    ↓
[5] API 从请求头读取 x-user-id
    ↓
[6] 查询数据库返回数据
```

**关键点：**
- Middleware 在 API 执行**之前**运行
- Middleware 可以修改请求（添加 headers）或直接返回响应（中断请求）
- 通过验证的请求才会到达真正的 API 代码

---

### JWT Token 结构

```
JWT Token 格式：Header.Payload.Signature

示例：
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJ4eHgiLCJlbWFpbCI6InBla29AdGVzdC5jb20iLCJpYXQiOjE2NzAwMDAwMDB9.xyz123

解析后：
Header:  { alg: "HS256", typ: "JWT" }
Payload: { userId: "xxx", email: "peko@test.com", iat: 1670000000 }
Signature: (用密钥签名，防止篡改)
```

**Payload 设计建议：**
```typescript
// ❌ 不要放敏感信息（JWT 是 Base64 编码，不是加密！）
{ userId: "xxx", password: "123456" }  // ❌ 密码泄露

// ✅ 只放必要的身份标识
{ userId: "xxx", email: "peko@test.com", iat: 1670000000, exp: 1670086400 }

// iat: issued at（签发时间）
// exp: expiration（过期时间）
```

---

### 白名单设计策略

```typescript
// 哪些路由需要鉴权？
const PUBLIC_ROUTES = [
  '/api/auth/login',       // 登录接口（未登录也能访问）
  '/api/auth/register',    // 注册接口
  '/api/auth/github',      // GitHub OAuth 入口
  '/api/auth/github/callback',  // OAuth 回调
  '/login',                // 登录页面
]

// 拦截逻辑
if (PUBLIC_ROUTES.some(route => pathname.startsWith(route))) {
  return NextResponse.next()  // 白名单直接放行
}

if (!pathname.startsWith('/api')) {
  return NextResponse.next()  // 非 API 路由不拦截（页面路由）
}

// 到这里说明是非白名单的 API 请求，需要验证 token
```

**为什么用 `startsWith` 而不是精确匹配？**
```typescript
// 精确匹配的问题
pathname === '/api/auth/github'  // ✅ /api/auth/github
pathname === '/api/auth/github'  // ❌ /api/auth/github/callback（没匹配上！）

// startsWith 支持前缀匹配
pathname.startsWith('/api/auth/github')  // ✅ /api/auth/github
pathname.startsWith('/api/auth/github')  // ✅ /api/auth/github/callback
```

---

## 💻 完整代码实现

### `src/middleware.ts`（鉴权中间件）

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { jwtVerify } from 'jose'

/**
 * 公开路由白名单（不需要鉴权的路径）
 */
const PUBLIC_ROUTES = [
  '/api/auth/login',            // 登录接口
  '/api/auth/register',         // 注册接口
  '/api/auth/github',           // GitHub OAuth 入口
  '/api/auth/github/callback',  // GitHub OAuth 回调
  '/login',                     // 登录页面
  '/register',                  // 注册页面
]

/**
 * Next.js 全局中间件
 * 运行在 Edge Runtime，拦截所有请求
 */
export async function middleware(req: NextRequest) {
  const { pathname } = req.nextUrl

  // ========================================
  // 1. 白名单检查：公开路由直接放行
  // ========================================
  if (PUBLIC_ROUTES.some(route => pathname.startsWith(route))) {
    return NextResponse.next()
  }

  // ========================================
  // 2. 非 API 路由不拦截（页面路由交给前端处理）
  // ========================================
  if (!pathname.startsWith('/api')) {
    return NextResponse.next()
  }

  // ========================================
  // 3. 验证 JWT Token
  // ========================================
  const token = req.cookies.get('token')?.value

  // 3.1 未携带 token → 401 未授权
  if (!token) {
    return NextResponse.json(
      { error: '未登录，请先登录' },
      { status: 401 }
    )
  }

  try {
    // 3.2 验证 token 签名和有效期
    const secret = new TextEncoder().encode(process.env.JWT_SECRET!)
    const { payload } = await jwtVerify(token, secret)

    // ========================================
    // 4. 注入用户信息到请求头
    // ========================================
    const requestHeaders = new Headers(req.headers)
    requestHeaders.set('x-user-id', payload.userId as string)
    requestHeaders.set('x-user-email', payload.email as string)

    // 4.1 返回修改后的请求（继续执行后续 API）
    return NextResponse.next({
      request: {
        headers: requestHeaders,
      },
    })

  } catch (error) {
    // 3.3 token 无效或已过期 → 401
    return NextResponse.json(
      { error: 'token 无效或已过期，请重新登录' },
      { status: 401 }
    )
  }
}

/**
 * 配置中间件匹配规则
 * matcher: 只拦截 /api/* 路径
 */
export const config = {
  matcher: ['/api/:path*'],  // 匹配所有 /api 开头的路由
}
```

---

### 环境变量配置

**`.env.local`**
```bash
# JWT 密钥（生产环境必须用强随机字符串）
JWT_SECRET=your-super-secret-key-at-least-32-characters-long

# 数据库连接
DATABASE_URL=postgresql://user:password@localhost:5432/mydb
```

**生成安全的 JWT 密钥：**
```bash
# 方法 1：Node.js 生成
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"

# 方法 2：OpenSSL 生成
openssl rand -hex 32

# 输出示例：
# a3f8e9d4c2b1a5f7e6d8c9b2a4f6e8d7c5b3a1f9e7d6c8b4a2f5e9d3c7b1a6f8
```

---

### 安装依赖

```bash
pnpm add jose  # JWT 验证库（Edge Runtime 兼容）
```

---

## 💻 用户列表 API 实现

### `src/app/api/users/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'

/**
 * GET /api/users
 * 获取用户列表（分页 + 搜索 + 关联查询）
 */
export async function GET(req: NextRequest) {
  try {
    // ========================================
    // 1. 从请求头获取当前用户信息（中间件注入）
    // ========================================
    const currentUserId = req.headers.get('x-user-id')
    const currentUserEmail = req.headers.get('x-user-email')
    
    console.log(`[用户列表] 当前用户: ${currentUserEmail} (${currentUserId})`)

    // ========================================
    // 2. 解析查询参数
    // ========================================
    const { searchParams } = req.nextUrl
    
    // 分页参数（默认第 1 页，每页 10 条）
    const page = Number(searchParams.get('page') ?? 1)
    const pageSize = Number(searchParams.get('pageSize') ?? 10)
    
    // 搜索关键词（支持邮箱和姓名模糊搜索）
    const keyword = searchParams.get('keyword')?.trim() ?? ''

    // ========================================
    // 3. 构建查询条件
    // ========================================
    const where = keyword
      ? {
          OR: [
            { name: { contains: keyword, mode: 'insensitive' } },   // 姓名模糊搜索（忽略大小写）
            { email: { contains: keyword, mode: 'insensitive' } },  // 邮箱模糊搜索
          ],
        }
      : {}

    // ========================================
    // 4. 并行查询总数和列表数据（性能优化）
    // ========================================
    const [total, users] = await Promise.all([
      // 查询符合条件的总记录数
      prisma.user.count({ where }),
      
      // 查询分页数据
      prisma.user.findMany({
        where,
        skip: (page - 1) * pageSize,  // 跳过前面的记录
        take: pageSize,               // 取当前页数据
        
        // ========================================
        // 5. select 安全字段过滤（不返回密码）
        // ========================================
        select: {
          id: true,
          email: true,
          name: true,
          avatar: true,
          status: true,
          createdAt: true,
          updatedAt: true,
          
          // 关联查询部门信息
          department: {
            select: {
              id: true,
              name: true,
            }
          },
          
          // 关联查询角色信息（通过 UserRole 中间表）
          userRoles: {
            select: {
              role: {
                select: {
                  id: true,
                  name: true,
                  description: true,
                }
              }
            }
          },
          
          // ❌ 不返回 password 字段！
        },
        
        // 按创建时间倒序排列（最新的在前）
        orderBy: { createdAt: 'desc' },
      }),
    ])

    // ========================================
    // 6. 格式化返回数据
    // ========================================
    return NextResponse.json({
      data: users,
      pagination: {
        total,              // 总记录数
        page,               // 当前页码
        pageSize,           // 每页条数
        totalPages: Math.ceil(total / pageSize),  // 总页数
      },
      keyword,  // 回显搜索关键词
    })

  } catch (error) {
    console.error('[用户列表] 查询失败:', error)
    return NextResponse.json(
      { error: '服务器错误，请稍后重试' },
      { status: 500 }
    )
  }
}
```

---

### Prisma Client 初始化

**`src/lib/prisma.ts`**
```typescript
import { PrismaClient } from '@prisma/client'

/**
 * Prisma Client 单例模式
 * 避免开发环境热更新时重复创建连接
 */
const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' ? ['query', 'error', 'warn'] : ['error'],
  })

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma
}
```

**为什么要用单例模式？**
```typescript
// ❌ 不用单例：每次热更新都创建新连接
const prisma = new PrismaClient()  // 开发环境可能创建几十个连接！

// ✅ 用单例：全局只有一个实例
globalForPrisma.prisma ?? new PrismaClient()  // 复用连接，节省资源
```

---

## ⚠️ 避坑指南

### 坑 1：Edge Runtime 不能用 Prisma

```typescript
// ❌ 错误写法：在 middleware.ts 查询数据库
import { prisma } from '@/lib/prisma'

export async function middleware(req: NextRequest) {
  const user = await prisma.user.findUnique({ where: { id: '...' } })
  // Error: Prisma requires Node.js runtime
}

// ✅ 正确做法：中间件只验证 JWT，不查数据库
export async function middleware(req: NextRequest) {
  const { payload } = await jwtVerify(token, secret)  // ✅ 只验证 token
  requestHeaders.set('x-user-id', payload.userId)
  // 数据库查询放到 API 里做
}
```

**原因：**
- Prisma 依赖 Node.js 原生模块（`fs`、`crypto`）
- Edge Runtime 只支持 Web 标准 API
- 解决：中间件只做轻量级验证，数据库查询放到 API 层

---

### 坑 2：忘记配置 matcher 导致循环拦截

```typescript
// ❌ 没有配置 matcher，拦截所有请求
export async function middleware(req: NextRequest) {
  // 拦截 /login 页面 → 检查 token → 失败 → 跳转 /login → 再次拦截 → 无限循环！
}

// ✅ 只拦截 API 路由
export const config = {
  matcher: ['/api/:path*'],  // 只拦截 /api/* 路径
}
```

---

### 坑 3：select 忘记过滤密码字段

```typescript
// ❌ 返回了密码哈希值（安全隐患）
const users = await prisma.user.findMany()
// 返回：[{ id, email, password: "$2a$10$...", ... }]

// ✅ 明确列出返回字段
const users = await prisma.user.findMany({
  select: {
    id: true,
    email: true,
    name: true,
    // password: false,  // 不需要显式写 false，不写就不返回
  }
})
```

**最佳实践：**
- 永远用 `select` 明确指定返回字段
- 不要用 `findMany()` 不带 select（容易泄露敏感字段）

---

### 坑 4：关联查询层级过深导致性能问题

```typescript
// ❌ 查询层级过深（N+1 问题）
const users = await prisma.user.findMany({
  include: {
    userRoles: {
      include: {
        role: {
          include: {
            rolePermissions: {
              include: {
                permission: true  // 嵌套 4 层！
              }
            }
          }
        }
      }
    }
  }
})
// 查询 100 个用户可能触发几百次 SQL 查询！

// ✅ 只查需要的字段和层级
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    userRoles: {
      select: {
        role: {
          select: { id: true, name: true }  // 只取 2 层，只要 id 和 name
        }
      }
    }
  }
})
```

---

### 坑 5：JWT 密钥泄露到代码里

```typescript
// ❌ 密钥硬编码（Git 泄露风险）
const secret = new TextEncoder().encode('my-secret-key')

// ✅ 从环境变量读取
const secret = new TextEncoder().encode(process.env.JWT_SECRET!)

// .env.local（不要提交到 Git）
JWT_SECRET=a3f8e9d4c2b1a5f7e6d8c9b2a4f6e8d7c5b3a1f9e7d6c8b4a2f5e9d3c7b1a6f8
```

**安全检查清单：**
- ✅ `.gitignore` 包含 `.env.local`
- ✅ 生产环境用不同的 JWT_SECRET
- ✅ 密钥长度至少 32 字符

---

## 💻 测试 API

### 测试流程

```bash
# 1. 登录获取 token（cookie 会自动保存到文件）
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@admin.com","password":"admin123"}' \
  -c /tmp/cookies.txt \
  -v

# 2. 携带 cookie 访问用户列表（200 成功）
curl http://localhost:3000/api/users \
  -b /tmp/cookies.txt

# 3. 不带 cookie 访问（401 未授权）
curl http://localhost:3000/api/users

# 4. 搜索用户
curl "http://localhost:3000/api/users?keyword=admin" \
  -b /tmp/cookies.txt

# 5. 分页查询
curl "http://localhost:3000/api/users?page=1&pageSize=5" \
  -b /tmp/cookies.txt

# 6. 组合查询（搜索 + 分页）
curl "http://localhost:3000/api/users?keyword=test&page=2&pageSize=10" \
  -b /tmp/cookies.txt
```

---

### 返回数据格式

```json
{
  "data": [
    {
      "id": "cm5x8y9z0000001...",
      "email": "admin@admin.com",
      "name": "超级管理员",
      "avatar": null,
      "status": true,
      "createdAt": "2026-03-18T06:07:20.162Z",
      "updatedAt": "2026-03-18T06:07:20.162Z",
      "department": null,
      "userRoles": [
        {
          "role": {
            "id": "cm5x8y9z0000002...",
            "name": "admin",
            "description": "超级管理员"
          }
        }
      ]
    }
  ],
  "pagination": {
    "total": 4,
    "page": 1,
    "pageSize": 10,
    "totalPages": 1
  },
  "keyword": ""
}
```

---

## ✅ 本课收获

### 知识点清单

- ✅ 理解 Next.js Middleware 工作原理（请求拦截、修改、放行）
- ✅ 掌握 Edge Runtime 和 Node.js Runtime 的区别
- ✅ 理解为什么 Edge Runtime 不能用 Node.js 原生模块
- ✅ 掌握 jose 库的使用（jwtVerify）
- ✅ 理解白名单设计策略（公开路由 vs 需鉴权路由）
- ✅ 掌握中间件注入请求头的技巧（x-user-id）
- ✅ 实现用户列表 API（分页、搜索、关联查询）
- ✅ 掌握 Prisma select 安全字段过滤（不返回密码）
- ✅ 理解 Promise.all 并行查询优化
- ✅ 掌握 Prisma Client 单例模式

### 可复用代码

- ✅ 完整的鉴权中间件（可直接用于生产）
- ✅ 用户列表 API（分页 + 搜索 + 关联查询）
- ✅ Prisma Client 单例初始化
- ✅ 环境变量配置模板

---

## 🚀 下一步

在下一课 **06-用户管理完整API** 中，我们将：

1. 实现用户 CRUD 完整接口（POST/GET/PATCH/DELETE）
2. 理解 RESTful API 设计规范
3. 掌握为什么 PATCH 不用 PUT（部分更新 vs 全量更新）
4. 实现创建用户时同时创建 Account 记录
5. 掌握更新用户角色的幂等性设计（deleteMany + recreate）
6. 处理删除用户的级联删除顺序（先删关联再删主表）
7. 学习错误处理规范

**预习建议：**
- 了解 HTTP 方法语义（GET/POST/PUT/PATCH/DELETE）
- 思考：为什么删除用户要先删 Account 和 UserRole？
- 阅读 Prisma 事务文档

**动手作业：**
1. 尝试不带 token 访问 `/api/users`，验证 401 错误
2. 修改白名单添加 `/api/public`，测试是否生效
3. 在用户列表 API 添加状态筛选（`?status=true` 只查启用用户）
4. 思考：如果要实现「只能看自己部门的用户」，应该怎么改？

---

## 📚 扩展阅读

- [Next.js Middleware 官方文档](https://nextjs.org/docs/app/building-your-application/routing/middleware)
- [Edge Runtime vs Node.js Runtime](https://nextjs.org/docs/app/building-your-application/rendering/edge-and-nodejs-runtimes)
- [jose 库官方文档](https://github.com/panva/jose)
- [JWT 标准规范 RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519)

---

**🎉 恭喜！你已经掌握了生产级的鉴权中间件和用户列表 API！**