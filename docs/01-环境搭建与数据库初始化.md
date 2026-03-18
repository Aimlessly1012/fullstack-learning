# Day 01 - PostgreSQL + Prisma 环境搭建

**日期：** 2026-03-18
**目标：** 搭建 Next.js + Prisma + PostgreSQL 全栈环境，完成第一个注册 API

---

## 环境准备

### 安装 PostgreSQL
```bash
brew install postgresql@16
brew services start postgresql@16

# 验证是否成功
psql postgres   # 进入后输入 \q 退出
```

### 创建数据库
```bash
createdb todo_db
```

---

## 创建项目

```bash
pnpm create next-app@latest fullstack-todo --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"
cd fullstack-todo

# 安装 Prisma
pnpm add @prisma/client @prisma/adapter-pg
pnpm add -D prisma
pnpm add dotenv bcryptjs jsonwebtoken
pnpm add -D @types/bcryptjs @types/jsonwebtoken

# 初始化 Prisma
pnpm prisma init
```

---

## Prisma 7 配置（重要！）

> ⚠️ Prisma 7 与旧版不同，schema 文件里不能写 url，需要单独的 config 文件

### `prisma/schema.prisma`
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  # 注意：Prisma 7 不在这里写 url！
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  password  String
  name      String?
  todos     Todo[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Todo {
  id        String   @id @default(cuid())
  title     String
  done      Boolean  @default(false)
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

### `prisma.config.ts`（根目录）
```typescript
import "dotenv/config"
import { defineConfig, env } from "prisma/config"

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: {
    path: "prisma/migrations",
  },
  datasource: {
    url: env("DATABASE_URL"),
  },
})
```

### `.env`
```env
DATABASE_URL="postgresql://你的用户名@localhost:5432/todo_db?schema=public"
JWT_SECRET="your-super-secret-key-change-this"
```

---

## 生成数据库表

```bash
pnpm prisma migrate dev --name init
pnpm prisma generate

# 可视化查看数据库
pnpm prisma studio
```

---

## Prisma Client 单例

`src/lib/prisma.ts`

```typescript
import { PrismaClient } from '@prisma/client'
import { PrismaPg } from '@prisma/adapter-pg'

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined
}

function createPrismaClient() {
  const adapter = new PrismaPg({
    connectionString: process.env.DATABASE_URL!,
  })
  return new PrismaClient({ adapter })
}

export const prisma = globalForPrisma.prisma ?? createPrismaClient()

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma
}
```

> 💡 为什么用单例？Next.js 开发模式热重载会反复执行模块，单例避免创建大量数据库连接。

---

## 注册 API

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

    // 密码加密，绝对不能存明文！
    const hashedPassword = await bcrypt.hash(password, 10)

    const user = await prisma.user.create({
      data: { email, password: hashedPassword, name },
      select: { id: true, email: true, name: true, createdAt: true }
      // select 只返回安全字段，不返回 password
    })

    return NextResponse.json({ user }, { status: 201 })
  } catch (error) {
    return NextResponse.json({ error: '服务器错误' }, { status: 500 })
  }
}
```

### 测试
```bash
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"123456","name":"Peko"}'
```

### 预期返回
```json
{
  "user": {
    "id": "cmmvhjzdl000082r24nii6yqq",
    "email": "test@test.com",
    "name": "Peko",
    "createdAt": "2026-03-18T03:32:38.745Z"
  }
}
```

---

## 今日收获

- ✅ PostgreSQL 安装和启动
- ✅ Prisma 7 新版配置方式（prisma.config.ts）
- ✅ 数据库 Schema 设计（User + Todo 关联）
- ✅ 密码加密（bcrypt）
- ✅ 第一个 REST API（注册接口）
- ✅ select 过滤敏感字段

## 下一步

- [ ] 登录 API（JWT token 生成）
- [ ] 认证中间件
- [ ] Todo CRUD API
