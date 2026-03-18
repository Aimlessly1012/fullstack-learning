# 02 - 登录认证与 JWT

## 本节目标

- 设计支持多种登录方式的数据库结构
- 实现账号密码登录 API
- 理解 JWT 生成与 httpOnly Cookie 存储

---

## 数据库变更：新增 Account 表

### 为什么要单独一张 Account 表？

原来 User 表只有 `email + password`，只能支持账号密码登录。
想支持钉钉、微信等第三方登录，需要把"登录方式"抽离出来。

```
User (用户主体)
 └── Account (登录方式，可以有多个)
      ├── provider: "credentials"  ← 账号密码
      ├── provider: "dingtalk"     ← 钉钉（后续添加）
      └── provider: "wechat"       ← 微信（后续添加）
```

### 更新后的 `prisma/schema.prisma`

```prisma
model User {
  id        String    @id @default(cuid())
  email     String?   @unique   // 改为可选（第三方登录可能没邮箱）
  password  String?             // 改为可选（第三方登录没密码）
  name      String?
  avatar    String?
  status    Boolean   @default(true)
  accounts  Account[]
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
}

model Account {
  id                String   @id @default(cuid())
  userId            String
  provider          String   // "credentials" | "dingtalk"
  providerAccountId String   // 第三方平台的用户 ID
  accessToken       String?
  user              User     @relation(fields: [userId], references: [id])
  createdAt         DateTime @default(now())

  @@unique([provider, providerAccountId])  // 同平台同账号唯一
}
```

```bash
pnpm prisma migrate dev --name add-account-table
pnpm prisma generate
```

---

## 注册 API（兼容新 Schema）

`src/app/api/auth/register/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server'
import bcrypt from 'bcryptjs'
import { prisma } from '@/lib/prisma'

export async function POST(req: NextRequest) {
  try {
    const { email, password, name } = await req.json()

    if (!email || !password) {
      return NextResponse.json({ error: '邮箱和密码必填' }, { status: 400 })
    }

    const existing = await prisma.user.findUnique({ where: { email } })
    if (existing) {
      return NextResponse.json({ error: '该邮箱已注册' }, { status: 409 })
    }

    const hashedPassword = await bcrypt.hash(password, 10)

    // 创建 User 同时创建 Account 记录
    const user = await prisma.user.create({
      data: {
        email,
        password: hashedPassword,
        name,
        accounts: {
          create: {
            provider: 'credentials',
            providerAccountId: email,
          }
        }
      },
      select: { id: true, email: true, name: true, createdAt: true }
    })

    return NextResponse.json({ user }, { status: 201 })
  } catch (error) {
    return NextResponse.json({ error: '服务器错误' }, { status: 500 })
  }
}
```

---

## 登录 API

`src/app/api/auth/login/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server'
import bcrypt from 'bcryptjs'
import jwt from 'jsonwebtoken'
import { prisma } from '@/lib/prisma'

export async function POST(req: NextRequest) {
  try {
    const { email, password } = await req.json()

    if (!email || !password) {
      return NextResponse.json({ error: '邮箱和密码必填' }, { status: 400 })
    }

    const user = await prisma.user.findUnique({
      where: { email },
      include: {
        accounts: { where: { provider: 'credentials' } }
      }
    })

    // 故意不区分"用户不存在"和"密码错误"，防止枚举攻击
    if (!user || !user.password || user.accounts.length === 0) {
      return NextResponse.json({ error: '邮箱或密码错误' }, { status: 401 })
    }

    if (!user.status) {
      return NextResponse.json({ error: '账号已被禁用' }, { status: 403 })
    }

    const isValid = await bcrypt.compare(password, user.password)
    if (!isValid) {
      return NextResponse.json({ error: '邮箱或密码错误' }, { status: 401 })
    }

    // 生成 JWT
    const token = jwt.sign(
      { userId: user.id, email: user.email },
      process.env.JWT_SECRET!,
      { expiresIn: '7d' }
    )

    const response = NextResponse.json({
      user: { id: user.id, email: user.email, name: user.name, avatar: user.avatar }
    })

    // httpOnly Cookie（比 localStorage 安全）
    response.cookies.set('token', token, {
      httpOnly: true,   // JS 无法读取，防 XSS
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',  // 防 CSRF
      maxAge: 60 * 60 * 24 * 7  // 7天
    })

    return response
  } catch (error) {
    return NextResponse.json({ error: '服务器错误' }, { status: 500 })
  }
}
```

---

## 测试

```bash
# 注册
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"peko@test.com","password":"123456","name":"Peko"}'

# 登录（查看 Cookie）
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"peko@test.com","password":"123456"}' \
  -v 2>&1 | grep -i "set-cookie"
```

### 预期返回

```json
{"user":{"id":"xxx","email":"peko@test.com","name":"Peko","avatar":null}}
```

Cookie：
```
set-cookie: token=eyJhbGci...; HttpOnly; SameSite=lax; Max-Age=604800
```

---

## 本节收获

- ✅ Account 表设计（支持多种登录方式）
- ✅ 登录 API（bcrypt 验证 + JWT 生成）
- ✅ httpOnly Cookie（安全存储 token）
- ✅ 防枚举攻击（不区分用户不存在/密码错误）
- ✅ 账号禁用检查

## 下一步

- [ ] 03 - RBAC 数据库设计（User / Role / Permission）
- [ ] 鉴权中间件（验证 JWT，保护 API）
