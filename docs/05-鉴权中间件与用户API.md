# 05 - 鉴权中间件与用户管理 API

## 本节目标

- 实现 Next.js 全局鉴权中间件
- 完成用户列表 API（分页 + 搜索 + 关联角色）

---

## 鉴权中间件

`src/middleware.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { jwtVerify } from 'jose'

const PUBLIC_ROUTES = [
  '/api/auth/login',
  '/api/auth/register',
  '/api/auth/github',
  '/api/auth/github/callback',
  '/login',
]

export async function middleware(req: NextRequest) {
  const { pathname } = req.nextUrl

  // 公开路由直接放行
  if (PUBLIC_ROUTES.some(route => pathname.startsWith(route))) {
    return NextResponse.next()
  }

  // 只拦截 /api 路由
  if (!pathname.startsWith('/api')) {
    return NextResponse.next()
  }

  const token = req.cookies.get('token')?.value

  if (!token) {
    return NextResponse.json({ error: '未登录' }, { status: 401 })
  }

  try {
    // jose 替代 jsonwebtoken（Edge Runtime 兼容）
    const secret = new TextEncoder().encode(process.env.JWT_SECRET!)
    const { payload } = await jwtVerify(token, secret)

    // 把用户信息注入请求头，供后续 API 使用
    const requestHeaders = new Headers(req.headers)
    requestHeaders.set('x-user-id', payload.userId as string)
    requestHeaders.set('x-user-email', payload.email as string)

    return NextResponse.next({ request: { headers: requestHeaders } })

  } catch (error) {
    return NextResponse.json({ error: 'token 无效或已过期' }, { status: 401 })
  }
}

export const config = {
  matcher: ['/api/:path*'],
}
```

> ⚠️ 为什么用 `jose` 而不是 `jsonwebtoken`？
> Next.js 中间件运行在 Edge Runtime，不支持 Node.js 原生模块，jose 是纯 Web API 实现。

```bash
pnpm add jose
```

### 测试

```bash
# 不带 token → 401
curl http://localhost:3000/api/users

# 登录获取 cookie
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@admin.com","password":"admin123"}' \
  -c /tmp/cookies.txt

# 带 cookie 访问 → 200
curl http://localhost:3000/api/users -b /tmp/cookies.txt
```

---

## 用户列表 API

`src/app/api/users/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'

export async function GET(req: NextRequest) {
  const { searchParams } = req.nextUrl
  const page = Number(searchParams.get('page') ?? 1)
  const pageSize = Number(searchParams.get('pageSize') ?? 10)
  const keyword = searchParams.get('keyword') ?? ''

  const where = keyword
    ? {
        OR: [
          { name: { contains: keyword } },
          { email: { contains: keyword } },
        ],
      }
    : {}

  const [total, users] = await Promise.all([
    prisma.user.count({ where }),
    prisma.user.findMany({
      where,
      skip: (page - 1) * pageSize,
      take: pageSize,
      select: {
        id: true,
        email: true,
        name: true,
        avatar: true,
        status: true,
        createdAt: true,
        department: { select: { id: true, name: true } },
        userRoles: {
          select: {
            role: { select: { id: true, name: true, description: true } }
          }
        }
      },
      orderBy: { createdAt: 'desc' },
    }),
  ])

  return NextResponse.json({
    data: users,
    total,
    page,
    pageSize,
    totalPages: Math.ceil(total / pageSize),
  })
}
```

### 测试

```bash
# 获取用户列表
curl http://localhost:3000/api/users -b /tmp/cookies.txt

# 搜索
curl "http://localhost:3000/api/users?keyword=admin" -b /tmp/cookies.txt

# 分页
curl "http://localhost:3000/api/users?page=1&pageSize=2" -b /tmp/cookies.txt
```

### 返回格式

```json
{
  "data": [
    {
      "id": "xxx",
      "email": "admin@admin.com",
      "name": "超级管理员",
      "avatar": null,
      "status": true,
      "createdAt": "2026-03-18T06:07:20.162Z",
      "department": null,
      "userRoles": [
        { "role": { "id": "xxx", "name": "admin", "description": "超级管理员" } }
      ]
    }
  ],
  "total": 4,
  "page": 1,
  "pageSize": 10,
  "totalPages": 1
}
```

---

## 本节收获

- ✅ Next.js 中间件（全局 JWT 验证）
- ✅ jose 替代 jsonwebtoken（Edge Runtime 兼容）
- ✅ 用户信息注入请求头（x-user-id）
- ✅ 用户列表 API（分页 + 搜索 + 关联角色/部门）

## 下一步

- [ ] 06 - 用户管理 API（创建/更新/删除/分配角色）
