# 06 - 用户管理完整 API

## 本节目标

- 实现用户 CRUD 完整接口
- 掌握关联数据的操作（角色分配、部门关联）
- 处理删除时的外键约束问题

---

## 路由结构

```
GET    /api/users          ← 用户列表（分页+搜索）
POST   /api/users          ← 创建用户
GET    /api/users/:id      ← 获取单个用户
PATCH  /api/users/:id      ← 更新用户
DELETE /api/users/:id      ← 删除用户
```

---

## `src/app/api/users/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server'
import bcrypt from 'bcryptjs'
import { prisma } from '@/lib/prisma'

// GET /api/users - 用户列表
export async function GET(req: NextRequest) {
  const { searchParams } = req.nextUrl
  const page = Number(searchParams.get('page') ?? 1)
  const pageSize = Number(searchParams.get('pageSize') ?? 10)
  const keyword = searchParams.get('keyword') ?? ''

  const where = keyword
    ? { OR: [{ name: { contains: keyword } }, { email: { contains: keyword } }] }
    : {}

  const [total, users] = await Promise.all([
    prisma.user.count({ where }),
    prisma.user.findMany({
      where,
      skip: (page - 1) * pageSize,
      take: pageSize,
      select: {
        id: true, email: true, name: true,
        avatar: true, status: true, createdAt: true,
        department: { select: { id: true, name: true } },
        userRoles: { select: { role: { select: { id: true, name: true, description: true } } } }
      },
      orderBy: { createdAt: 'desc' },
    }),
  ])

  return NextResponse.json({ data: users, total, page, pageSize, totalPages: Math.ceil(total / pageSize) })
}

// POST /api/users - 创建用户
export async function POST(req: NextRequest) {
  try {
    const { email, password, name, roleIds, departmentId } = await req.json()

    if (!email || !password) {
      return NextResponse.json({ error: '邮箱和密码必填' }, { status: 400 })
    }

    const existing = await prisma.user.findUnique({ where: { email } })
    if (existing) {
      return NextResponse.json({ error: '邮箱已存在' }, { status: 409 })
    }

    const hashedPassword = await bcrypt.hash(password, 10)

    const user = await prisma.user.create({
      data: {
        email, password: hashedPassword, name,
        departmentId: departmentId || null,
        accounts: { create: { provider: 'credentials', providerAccountId: email } },
        userRoles: roleIds?.length
          ? { create: roleIds.map((roleId: string) => ({ roleId })) }
          : undefined,
      },
      select: {
        id: true, email: true, name: true, status: true, createdAt: true,
        userRoles: { select: { role: { select: { id: true, name: true } } } }
      }
    })

    return NextResponse.json({ data: user }, { status: 201 })
  } catch (error) {
    return NextResponse.json({ error: '服务器错误' }, { status: 500 })
  }
}
```

---

## `src/app/api/users/[id]/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'

// GET /api/users/:id
export async function GET(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params

  const user = await prisma.user.findUnique({
    where: { id },
    select: {
      id: true, email: true, name: true,
      avatar: true, status: true, createdAt: true,
      department: { select: { id: true, name: true } },
      userRoles: { select: { role: { select: { id: true, name: true } } } }
    }
  })

  if (!user) return NextResponse.json({ error: '用户不存在' }, { status: 404 })
  return NextResponse.json({ data: user })
}

// PATCH /api/users/:id
export async function PATCH(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    const { id } = await params
    const { name, status, departmentId, roleIds } = await req.json()

    const user = await prisma.user.update({
      where: { id },
      data: {
        ...(name !== undefined && { name }),
        ...(status !== undefined && { status }),
        ...(departmentId !== undefined && { departmentId }),
        ...(roleIds && {
          userRoles: {
            deleteMany: {},  // 先清空旧角色
            create: roleIds.map((roleId: string) => ({ roleId }))  // 再写入新角色
          }
        }),
      },
      select: {
        id: true, email: true, name: true,
        status: true, updatedAt: true,
        userRoles: { select: { role: { select: { id: true, name: true } } } }
      }
    })

    return NextResponse.json({ data: user })
  } catch (error) {
    return NextResponse.json({ error: '服务器错误' }, { status: 500 })
  }
}

// DELETE /api/users/:id
export async function DELETE(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    const { id } = await params

    // ⚠️ 必须先删关联数据，否则违反外键约束
    await prisma.account.deleteMany({ where: { userId: id } })
    await prisma.userRole.deleteMany({ where: { userId: id } })
    await prisma.user.delete({ where: { id } })

    return NextResponse.json({ message: '删除成功' })
  } catch (error) {
    return NextResponse.json({ error: '服务器错误' }, { status: 500 })
  }
}
```

---

## 测试命令

```bash
# 创建用户（带角色）
curl -X POST http://localhost:3000/api/users \
  -H "Content-Type: application/json" \
  -b /tmp/cookies.txt \
  -d '{"email":"tom@test.com","password":"123456","name":"Tom"}'

# 获取单个
curl http://localhost:3000/api/users/<id> -b /tmp/cookies.txt

# 更新（禁用 + 改名）
curl -X PATCH http://localhost:3000/api/users/<id> \
  -H "Content-Type: application/json" \
  -b /tmp/cookies.txt \
  -d '{"name":"Tom Updated","status":false}'

# 删除
curl -X DELETE http://localhost:3000/api/users/<id> -b /tmp/cookies.txt
```

---

## 关键点总结

| 问题 | 解决方案 |
|------|---------|
| 删除用户报外键错误 | 先删 Account + UserRole，再删 User |
| 更新角色 | deleteMany 清空旧角色 + create 写入新角色 |
| 可选字段更新 | 用 `...(field !== undefined && { field })` 展开 |

---

## 本节收获

- ✅ 用户 CRUD 完整接口
- ✅ 关联数据操作（角色分配）
- ✅ 外键约束处理（删除顺序）
- ✅ 可选字段更新技巧

## 下一步

- [ ] 07 - 角色与权限管理 API
