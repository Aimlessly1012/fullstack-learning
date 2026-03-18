# 06 - 用户管理完整 API（CRUD）

## 📖 本课目标

- 掌握 RESTful API 设计规范
- 实现用户 CRUD（创建、读取、更新、删除）
- 理解 PATCH vs PUT 的区别
- 掌握级联删除的正确顺序
- 理解幂等性设计

---

## 🧠 RESTful API 设计规范

REST 不是协议，是一种**设计风格**。核心原则：

```
资源用名词，操作用 HTTP 方法

GET    /api/users          ← 获取用户列表
POST   /api/users          ← 创建用户
GET    /api/users/:id      ← 获取单个用户
PATCH  /api/users/:id      ← 部分更新用户
DELETE /api/users/:id      ← 删除用户
```

**HTTP 方法语义：**
| 方法 | 含义 | 幂等？ |
|------|------|--------|
| GET | 读取 | ✅ |
| POST | 创建 | ❌ |
| PUT | 全量替换 | ✅ |
| PATCH | 部分更新 | ✅（通常） |
| DELETE | 删除 | ✅ |

---

## 🤔 为什么用 PATCH 不用 PUT？

**PUT** = 全量替换，你必须把所有字段都传过来：
```json
// PUT /api/users/123 — 必须传所有字段
{ "email": "peko@test.com", "name": "Peko", "status": true, "departmentId": null }
// 漏传任何一个字段，那个字段就会被清空！
```

**PATCH** = 部分更新，只传你想改的字段：
```json
// PATCH /api/users/123 — 只传要改的
{ "name": "Peko New Name" }
// email、status 等保持不变
```

用户管理场景下，往往只需要改名字或状态，用 PATCH 更合适。

---

## `src/app/api/users/route.ts` — 列表 + 创建

```typescript
import { NextRequest, NextResponse } from 'next/server'
import bcrypt from 'bcryptjs'
import { prisma } from '@/lib/prisma'

// GET /api/users — 用户列表（分页 + 搜索）
export async function GET(req: NextRequest) {
  const { searchParams } = new URL(req.url)
  const page = Number(searchParams.get('page') ?? 1)
  const pageSize = Number(searchParams.get('pageSize') ?? 10)
  const keyword = searchParams.get('keyword') ?? ''

  // 构建搜索条件
  const where = keyword
    ? {
        OR: [
          { name: { contains: keyword, mode: 'insensitive' as const } },
          { email: { contains: keyword, mode: 'insensitive' as const } },
        ],
      }
    : {}

  // 并行查询：数据 + 总数（性能优化）
  const [users, total] = await Promise.all([
    prisma.user.findMany({
      where,
      skip: (page - 1) * pageSize,
      take: pageSize,
      orderBy: { createdAt: 'desc' },
      select: {
        id: true,
        email: true,
        name: true,
        avatar: true,
        status: true,
        createdAt: true,
        department: { select: { id: true, name: true } },
        userRoles: { select: { role: { select: { id: true, name: true } } } },
        // ⚠️ 注意：绝对不返回 password 字段！
      },
    }),
    prisma.user.count({ where }),
  ])

  return NextResponse.json({ data: users, total, page, pageSize })
}

// POST /api/users — 创建用户
export async function POST(req: NextRequest) {
  try {
    const { email, password, name, roleIds } = await req.json()

    // 参数校验
    if (!email || !password) {
      return NextResponse.json({ error: '邮箱和密码必填' }, { status: 400 })
    }
    if (password.length < 6) {
      return NextResponse.json({ error: '密码至少 6 位' }, { status: 400 })
    }

    // 检查邮箱是否已存在
    const existing = await prisma.user.findUnique({ where: { email } })
    if (existing) {
      return NextResponse.json({ error: '该邮箱已注册' }, { status: 409 })
    }

    // 密码加密
    const hashedPassword = await bcrypt.hash(password, 10)

    // 创建用户 + Account 记录（credentials 登录方式）
    // 为什么同时创建 Account？
    // 因为我们用 Account 表统一管理登录方式
    // 密码登录也是一种"登录方式"，provider = "credentials"
    const user = await prisma.user.create({
      data: {
        email,
        name,
        password: hashedPassword,
        // 同步创建 Account 记录
        accounts: {
          create: {
            provider: 'credentials',
            providerAccountId: email,  // credentials 用邮箱作为唯一标识
          },
        },
        // 如果传了角色 ID，同步分配角色
        userRoles: roleIds?.length
          ? { create: roleIds.map((roleId: string) => ({ roleId })) }
          : undefined,
      },
      select: {
        id: true,
        email: true,
        name: true,
        status: true,
        createdAt: true,
        userRoles: { select: { role: { select: { id: true, name: true } } } },
      },
    })

    return NextResponse.json({ data: user }, { status: 201 })
  } catch (error) {
    console.error('Create user error:', error)
    return NextResponse.json({ error: '服务器错误' }, { status: 500 })
  }
}
```

---

## `src/app/api/users/[id]/route.ts` — 单个用户操作

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'

// GET /api/users/:id — 获取单个用户
export async function GET(
  _req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params

  const user = await prisma.user.findUnique({
    where: { id },
    select: {
      id: true,
      email: true,
      name: true,
      avatar: true,
      status: true,
      createdAt: true,
      updatedAt: true,
      department: { select: { id: true, name: true } },
      userRoles: { select: { role: { select: { id: true, name: true, description: true } } } },
      accounts: { select: { provider: true, createdAt: true } },
    },
  })

  if (!user) {
    return NextResponse.json({ error: '用户不存在' }, { status: 404 })
  }

  return NextResponse.json({ data: user })
}

// PATCH /api/users/:id — 部分更新用户
export async function PATCH(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    const { id } = await params
    const { name, avatar, status, departmentId, roleIds } = await req.json()

    // 检查用户是否存在
    const existing = await prisma.user.findUnique({ where: { id } })
    if (!existing) {
      return NextResponse.json({ error: '用户不存在' }, { status: 404 })
    }

    const user = await prisma.user.update({
      where: { id },
      data: {
        // 只更新传了的字段（undefined 的字段 Prisma 会忽略）
        ...(name !== undefined && { name }),
        ...(avatar !== undefined && { avatar }),
        ...(status !== undefined && { status }),
        ...(departmentId !== undefined && { departmentId }),

        // 角色重新分配
        // 🧠 幂等性设计：先全删再重建
        // 为什么不用 upsert？因为需要处理"删除旧角色"的情况
        // deleteMany: {} = 删除该用户所有角色
        // create: [...] = 重新创建新角色列表
        // 这样无论调用多少次，结果都一样 = 幂等
        ...(roleIds !== undefined && {
          userRoles: {
            deleteMany: {},
            create: roleIds.map((roleId: string) => ({ roleId })),
          },
        }),
      },
      select: {
        id: true,
        email: true,
        name: true,
        avatar: true,
        status: true,
        updatedAt: true,
        department: { select: { id: true, name: true } },
        userRoles: { select: { role: { select: { id: true, name: true } } } },
      },
    })

    return NextResponse.json({ data: user })
  } catch (error) {
    console.error('Update user error:', error)
    return NextResponse.json({ error: '服务器错误' }, { status: 500 })
  }
}

// DELETE /api/users/:id — 删除用户
export async function DELETE(
  _req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    const { id } = await params

    // 检查用户是否存在
    const existing = await prisma.user.findUnique({ where: { id } })
    if (!existing) {
      return NextResponse.json({ error: '用户不存在' }, { status: 404 })
    }

    // ⚠️ 级联删除顺序很重要！
    // 必须先删"子表"（引用 userId 的表），再删"主表"（User）
    // 否则违反外键约束，报错！
    //
    // 顺序：
    // 1. Account（关联 userId）
    // 2. UserRole（关联 userId）
    // 3. User（主表）
    await prisma.account.deleteMany({ where: { userId: id } })
    await prisma.userRole.deleteMany({ where: { userId: id } })
    await prisma.user.delete({ where: { id } })

    return NextResponse.json({ message: '删除成功' })
  } catch (error) {
    console.error('Delete user error:', error)
    return NextResponse.json({ error: '服务器错误' }, { status: 500 })
  }
}
```

---

## 测试所有接口

```bash
# 先登录获取 cookie
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -c /tmp/cookies.txt \
  -d '{"email":"admin@admin.com","password":"admin123"}'

# 1. 获取用户列表
curl http://localhost:3000/api/users -b /tmp/cookies.txt

# 2. 搜索用户
curl "http://localhost:3000/api/users?keyword=admin" -b /tmp/cookies.txt

# 3. 创建用户
curl -X POST http://localhost:3000/api/users \
  -H "Content-Type: application/json" \
  -b /tmp/cookies.txt \
  -d '{"email":"tom@test.com","password":"123456","name":"Tom"}'

# 4. 获取单个用户（把 ID 替换成真实 ID）
curl http://localhost:3000/api/users/用户ID -b /tmp/cookies.txt

# 5. 更新用户
curl -X PATCH http://localhost:3000/api/users/用户ID \
  -H "Content-Type: application/json" \
  -b /tmp/cookies.txt \
  -d '{"name":"Tom Updated","status":false}'

# 6. 删除用户
curl -X DELETE http://localhost:3000/api/users/用户ID -b /tmp/cookies.txt
```

---

## 错误处理规范

统一的错误响应格式：

```typescript
// ✅ 好的错误处理
return NextResponse.json({ error: '用户不存在' }, { status: 404 })

// ❌ 避免：直接暴露内部错误
return NextResponse.json({ error: error.message }, { status: 500 })
// 可能泄露数据库结构、堆栈信息等敏感内容

// ✅ 服务器错误统一返回通用信息
} catch (error) {
  console.error('具体错误记录到日志:', error)  // 服务端日志
  return NextResponse.json({ error: '服务器错误' }, { status: 500 })  // 前端只看到通用信息
}
```

**常用 HTTP 状态码：**
| 码 | 含义 | 使用场景 |
|----|------|---------|
| 200 | OK | 成功获取/更新 |
| 201 | Created | 成功创建资源 |
| 400 | Bad Request | 参数错误 |
| 401 | Unauthorized | 未登录 |
| 403 | Forbidden | 无权限 |
| 404 | Not Found | 资源不存在 |
| 409 | Conflict | 冲突（如邮箱已存在） |
| 500 | Server Error | 服务器内部错误 |

---

## ⚠️ 避坑指南

### 1. 永远不要返回 password 字段
使用 `select` 明确指定要返回的字段，密码字段绝对不能出现在响应中。

### 2. 级联删除顺序
删除有外键关联的记录时，必须先删引用主表的子表，否则报外键约束错误。

### 3. 幂等性设计
角色分配使用 `deleteMany` + `create` 模式，而不是逐个 `upsert`，确保任何时候调用结果一致。

### 4. params 是 Promise
Next.js 15+ 中 `params` 是异步的，必须 `await params` 才能拿到值。

```typescript
// ❌ 旧写法（Next.js 14）
{ params }: { params: { id: string } }
const { id } = params

// ✅ 新写法（Next.js 15+）
{ params }: { params: Promise<{ id: string }> }
const { id } = await params
```

---

## ✅ 本课收获

- ✅ RESTful API 设计规范（资源命名 + HTTP 方法语义）
- ✅ PATCH vs PUT（部分更新 vs 全量替换）
- ✅ 用户 CRUD 完整实现
- ✅ 幂等性设计（角色分配 deleteMany + create）
- ✅ 级联删除正确顺序
- ✅ 错误处理规范和 HTTP 状态码

## 下一步

- [ ] 07 - 角色与权限管理 API
