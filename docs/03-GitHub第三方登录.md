# 03 - GitHub 第三方登录（OAuth 2.0）

## 本节目标

- 理解 OAuth 2.0 授权流程
- 实现 GitHub 登录
- 理解多种登录方式的统一架构

---

## OAuth 2.0 流程

```
用户点击"GitHub登录"
  → GET /api/auth/github
  → 重定向到 GitHub 授权页
  → 用户点击授权
  → GitHub 回调 /api/auth/github/callback?code=xxx
  → 用 code 换 access_token
  → 用 access_token 获取用户信息
  → 查找或创建 User + Account
  → 生成 JWT，写 Cookie
  → 跳转首页
```

---

## 准备工作

### 1. 创建 GitHub OAuth App

1. 打开 https://github.com/settings/developers
2. 点 **New OAuth App**
3. 填写：
   - Homepage URL: `http://localhost:3000`
   - Callback URL: `http://localhost:3000/api/auth/github/callback`
4. 获取 `Client ID` 和 `Client Secret`

### 2. 配置 `.env`

```env
GITHUB_CLIENT_ID="你的 Client ID"
GITHUB_CLIENT_SECRET="你的 Client Secret"
NEXT_PUBLIC_APP_URL="http://localhost:3000"
```

---

## 代码实现

### 登录入口 `src/app/api/auth/github/route.ts`

```typescript
import { NextResponse } from 'next/server'

export async function GET() {
  const params = new URLSearchParams({
    client_id: process.env.GITHUB_CLIENT_ID!,
    redirect_uri: `${process.env.NEXT_PUBLIC_APP_URL}/api/auth/github/callback`,
    scope: 'user:email',
  })

  return NextResponse.redirect(
    `https://github.com/login/oauth/authorize?${params}`
  )
}
```

### 回调处理 `src/app/api/auth/github/callback/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server'
import jwt from 'jsonwebtoken'
import { prisma } from '@/lib/prisma'

export async function GET(req: NextRequest) {
  const code = req.nextUrl.searchParams.get('code')

  if (!code) {
    return NextResponse.redirect(
      `${process.env.NEXT_PUBLIC_APP_URL}/login?error=missing_code`
    )
  }

  try {
    // 1. 用 code 换 access_token
    const tokenRes = await fetch('https://github.com/login/oauth/access_token', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Accept: 'application/json',
      },
      body: JSON.stringify({
        client_id: process.env.GITHUB_CLIENT_ID,
        client_secret: process.env.GITHUB_CLIENT_SECRET,
        code,
      }),
    })
    const tokenData = await tokenRes.json()
    const accessToken = tokenData.access_token

    if (!accessToken) {
      return NextResponse.redirect(
        `${process.env.NEXT_PUBLIC_APP_URL}/login?error=token_failed`
      )
    }

    // 2. 获取 GitHub 用户信息
    const userRes = await fetch('https://api.github.com/user', {
      headers: { Authorization: `Bearer ${accessToken}` },
    })
    const githubUser = await userRes.json()

    // 3. 获取邮箱（GitHub 邮箱可能是私有的，单独请求）
    const emailRes = await fetch('https://api.github.com/user/emails', {
      headers: { Authorization: `Bearer ${accessToken}` },
    })
    const emails = await emailRes.json()
    const primaryEmail = emails.find(
      (e: any) => e.primary && e.verified
    )?.email

    // 4. 查找或创建用户（首次登录自动注册）
    let account = await prisma.account.findUnique({
      where: {
        provider_providerAccountId: {
          provider: 'github',
          providerAccountId: String(githubUser.id),
        }
      },
      include: { user: true }
    })

    let user = account?.user

    if (!account) {
      user = await prisma.user.create({
        data: {
          email: primaryEmail,
          name: githubUser.name || githubUser.login,
          avatar: githubUser.avatar_url,
          accounts: {
            create: {
              provider: 'github',
              providerAccountId: String(githubUser.id),
              accessToken,
            }
          }
        }
      })
    }

    // 5. 生成 JWT（与账号密码登录完全一致）
    const token = jwt.sign(
      { userId: user!.id, email: user!.email },
      process.env.JWT_SECRET!,
      { expiresIn: '7d' }
    )

    // 6. 写 Cookie，跳转首页
    const response = NextResponse.redirect(
      `${process.env.NEXT_PUBLIC_APP_URL}/`
    )
    response.cookies.set('token', token, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      maxAge: 60 * 60 * 24 * 7,
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

## 测试

浏览器访问：
```
http://localhost:3000/api/auth/github
```

流程：
1. 跳转 GitHub 授权页
2. 授权后回调
3. 跳转回首页

### 日志验证
```
GET /api/auth/github 307              ← 跳转 GitHub ✅
GET /api/auth/github/callback?code=xxx 307  ← 处理回调 ✅
GET / 200                             ← 跳转首页 ✅
```

### 数据库验证

Prisma Studio 查看：
- `User` 表：新增一条 GitHub 用户（name/avatar 来自 GitHub）
- `Account` 表：`provider: "github"`，`providerAccountId` 是 GitHub 用户 ID

---

## 架构总结

```
登录方式           provider            User 表
账号密码    →   credentials    →   email + password
GitHub     →   github         →   name + avatar（无密码）
钉钉(后续)  →   dingtalk       →   name + avatar（无密码）
```

三种方式最终都生成同一个 JWT，前端无感知。

---

## 本节收获

- ✅ OAuth 2.0 完整授权流程
- ✅ GitHub OAuth App 配置
- ✅ code → access_token → 用户信息 三步换取
- ✅ 首次登录自动注册（查找或创建）
- ✅ 统一 JWT 生成，多种登录方式无缝融合

## 下一步

- [ ] 04 - RBAC 数据库设计（Role / Permission）
- [ ] 鉴权中间件（保护需要登录的 API）
