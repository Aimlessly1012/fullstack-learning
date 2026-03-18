# 03 - GitHub 第三方登录（OAuth 2.0）

## 📖 本课目标

- 理解 OAuth 2.0 授权流程
- 实现 GitHub 第三方登录
- 掌握 Account 表多登录方式统一管理
- 了解 CSRF 防护（state 参数）

---

## 🤔 为什么要第三方登录？

用户最怕注册！每次填邮箱、设密码、验证邮件……流失率极高。

第三方登录好处：
- **用户体验好**：一键授权，不用记密码
- **安全性高**：密码由 GitHub 管，我们不存敏感信息
- **信任感强**：用户已经信任 GitHub

---

## 🧠 OAuth 2.0 是什么？

OAuth 2.0 是一个**授权协议**，让第三方应用可以在用户授权下访问其资源，而不需要知道用户的密码。

**类比理解：**
> 你住酒店，前台给你一张房卡（token）。
> 房卡只能开你的房间，不能开其他房间，也不是你的钥匙。
> 这就是 OAuth —— 有限授权，不共享密码。

---

## 🔄 OAuth 2.0 完整流程

```
用户点击"GitHub 登录"
        ↓
1. 我们的应用 → 跳转到 GitHub 授权页
   https://github.com/login/oauth/authorize
   ?client_id=xxx&redirect_uri=xxx&state=随机字符串

        ↓ 用户点击"同意授权"

2. GitHub → 回调我们的接口，带上 code 和 state
   http://localhost:3000/api/auth/github/callback
   ?code=xxxx&state=xxx

        ↓ 服务端用 code 换 token

3. 我们的服务器 → 请求 GitHub API
   POST https://github.com/login/oauth/access_token
   body: { client_id, client_secret, code }

        ↓ GitHub 返回 access_token

4. 我们用 access_token 获取用户信息
   GET https://api.github.com/user
   Authorization: Bearer access_token

        ↓ 拿到用户信息，生成我们自己的 JWT

5. 设置 httpOnly Cookie → 跳转到首页
```

---

## 为什么不用 NextAuth？

NextAuth 是一个成熟的认证库，确实好用。但这门课**手动实现**的理由：

1. **学到原理**：NextAuth 把所有细节封装了，你不知道发生了什么
2. **灵活性更高**：自己实现可以随时定制逻辑
3. **面试必考**：OAuth 流程是面试高频题，用了 NextAuth 你说不清楚
4. **项目需要时再用**：理解原理后，用 NextAuth 才知道它在做什么

---

## 第一步：创建 GitHub OAuth App

1. 打开 https://github.com/settings/developers
2. 点击 **"New OAuth App"**
3. 填写：
   - **Application name**: `fullstack-todo-dev`
   - **Homepage URL**: `http://localhost:3000`
   - **Authorization callback URL**: `http://localhost:3000/api/auth/github/callback`
4. 点击 **"Register application"**
5. 复制 **Client ID** 和生成 **Client Secret**

---

## 第二步：配置环境变量

`.env` 添加：

```env
GITHUB_CLIENT_ID="你的 Client ID"
GITHUB_CLIENT_SECRET="你的 Client Secret"
NEXT_PUBLIC_APP_URL="http://localhost:3000"
```

---

## 第三步：发起授权请求

`src/app/api/auth/github/route.ts`

```typescript
import { NextResponse } from 'next/server'

export async function GET() {
  const clientId = process.env.GITHUB_CLIENT_ID!
  const redirectUri = `${process.env.NEXT_PUBLIC_APP_URL}/api/auth/github/callback`

  // state 参数：随机字符串，防止 CSRF 攻击
  // CSRF 攻击：攻击者伪造一个回调链接，诱导用户点击，绑定攻击者的账号
  // 有了 state，我们可以验证回调是否来自我们发起的授权请求
  const state = Math.random().toString(36).substring(7)

  const params = new URLSearchParams({
    client_id: clientId,
    redirect_uri: redirectUri,
    scope: 'user:email',  // 申请读取用户邮箱权限
    state,
  })

  const githubAuthUrl = `https://github.com/login/oauth/authorize?${params}`

  return NextResponse.redirect(githubAuthUrl)
}
```

---

## 第四步：处理回调

`src/app/api/auth/github/callback/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { SignJWT } from 'jose'
import { prisma } from '@/lib/prisma'

const JWT_SECRET = new TextEncoder().encode(process.env.JWT_SECRET!)

export async function GET(req: NextRequest) {
  const { searchParams } = new URL(req.url)
  const code = searchParams.get('code')
  const state = searchParams.get('state')

  // 参数校验
  if (!code) {
    return NextResponse.redirect(
      `${process.env.NEXT_PUBLIC_APP_URL}/login?error=no_code`
    )
  }

  try {
    // Step 1: 用 code 换 access_token
    const tokenRes = await fetch(
      'https://github.com/login/oauth/access_token',
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Accept: 'application/json',  // 让 GitHub 返回 JSON 格式
        },
        body: JSON.stringify({
          client_id: process.env.GITHUB_CLIENT_ID,
          client_secret: process.env.GITHUB_CLIENT_SECRET,
          code,
        }),
      }
    )

    const tokenData = await tokenRes.json()

    if (tokenData.error) {
      // code 过期或已使用（code 只能用一次！）
      return NextResponse.redirect(
        `${process.env.NEXT_PUBLIC_APP_URL}/login?error=invalid_code`
      )
    }

    const accessToken = tokenData.access_token

    // Step 2: 用 access_token 获取用户信息
    const githubUserRes = await fetch('https://api.github.com/user', {
      headers: {
        Authorization: `Bearer ${accessToken}`,
        Accept: 'application/vnd.github.v3+json',
      },
    })

    const githubUser = await githubUserRes.json()

    // Step 3: 获取用户邮箱（有些用户没公开邮箱，需要单独请求）
    let email = githubUser.email
    if (!email) {
      const emailsRes = await fetch('https://api.github.com/user/emails', {
        headers: { Authorization: `Bearer ${accessToken}` },
      })
      const emails = await emailsRes.json()
      // 找主邮箱
      const primaryEmail = emails.find((e: any) => e.primary && e.verified)
      email = primaryEmail?.email
    }

    // Step 4: 查找或创建用户
    // 先查 Account 表，看这个 GitHub 账号是否已绑定
    const existingAccount = await prisma.account.findUnique({
      where: {
        provider_providerAccountId: {
          provider: 'github',
          providerAccountId: String(githubUser.id),
        },
      },
      include: { user: true },
    })

    let userId: string

    if (existingAccount) {
      // 已绑定过，直接用已有用户
      userId = existingAccount.userId

      // 更新 access_token（token 可能刷新了）
      await prisma.account.update({
        where: { id: existingAccount.id },
        data: { accessToken },
      })
    } else {
      // 新用户：创建 User + Account
      // 先看邮箱是否已存在（可能用密码注册过）
      let user = email
        ? await prisma.user.findUnique({ where: { email } })
        : null

      if (!user) {
        // 全新用户，创建
        user = await prisma.user.create({
          data: {
            email,
            name: githubUser.name || githubUser.login,
            avatar: githubUser.avatar_url,
          },
        })
      }

      // 创建 Account 记录，绑定 GitHub
      await prisma.account.create({
        data: {
          userId: user.id,
          provider: 'github',
          providerAccountId: String(githubUser.id),
          accessToken,
        },
      })

      userId = user.id
    }

    // Step 5: 生成我们自己的 JWT
    const user = await prisma.user.findUnique({
      where: { id: userId },
      select: { id: true, email: true, name: true },
    })

    const token = await new SignJWT({
      sub: user!.id,
      email: user!.email,
    })
      .setProtectedHeader({ alg: 'HS256' })
      .setExpirationTime('7d')
      .sign(JWT_SECRET)

    // Step 6: 设置 cookie，跳转首页
    const response = NextResponse.redirect(
      `${process.env.NEXT_PUBLIC_APP_URL}/dashboard`
    )

    response.cookies.set('token', token, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      maxAge: 60 * 60 * 24 * 7,  // 7 天
      path: '/',
    })

    return response
  } catch (error) {
    console.error('GitHub OAuth error:', error)
    return NextResponse.redirect(
      `${process.env.NEXT_PUBLIC_APP_URL}/login?error=server_error`
    )
  }
}
```

---

## Account 表设计理由

```prisma
model Account {
  id                String   @id @default(cuid())
  userId            String
  provider          String           // "credentials" | "github" | "dingtalk"
  providerAccountId String           // 第三方平台的用户 ID
  accessToken       String?          // 第三方 token（可能过期）
  user              User     @relation(fields: [userId], references: [id])
  createdAt         DateTime @default(now())

  @@unique([provider, providerAccountId])  // 同一平台同一用户唯一
}
```

**为什么需要 Account 表？**

一个用户可能有多种登录方式：
```
User (peko@gmail.com)
  ├── Account (provider: "credentials")    ← 密码登录
  ├── Account (provider: "github")         ← GitHub 登录
  └── Account (provider: "dingtalk")       ← 钉钉登录（未来扩展）
```

所有登录方式最终都指向同一个 User，权限、数据共享。

---

## ⚠️ 避坑指南

### 1. code 只能用一次
GitHub 的 code 使用一次后就失效。如果用户刷新 callback 页面，会报错。解决：收到 code 后立即跳转，不要停留在 callback URL。

### 2. 邮箱可能为 null
GitHub 用户可以不公开邮箱。必须额外请求 `/user/emails` 接口，找 `primary: true && verified: true` 的邮箱。

### 3. Client Secret 绝对不能泄露
Client Secret 放在服务端环境变量（`.env`），绝不能放在前端代码或 `NEXT_PUBLIC_` 前缀的变量中。

### 4. scope 按需申请
只申请需要的权限，用户信任感更高。我们只需要 `user:email`，不要申请不必要的 `repo` 等权限。

### 5. state 参数
生产环境应该把 state 存 session 或 Redis，回调时验证一致性，防 CSRF。本课简化处理，生产环境务必完善。

---

## 测试

```bash
# 浏览器打开（不能用 curl，需要浏览器处理重定向）
open http://localhost:3000/api/auth/github

# 应该跳转到 GitHub 授权页，授权后跳回 /dashboard
```

---

## ✅ 本课收获

- ✅ OAuth 2.0 四步授权流程（redirect → code → token → 用户信息）
- ✅ GitHub OAuth App 创建和配置
- ✅ Account 表多登录方式统一管理
- ✅ code 换 token，token 换用户信息
- ✅ 新用户创建 vs 已有用户绑定的逻辑
- ✅ state 参数防 CSRF 原理

## 下一步

- [ ] 04 - RBAC 数据库设计（用户、角色、权限三层模型）
