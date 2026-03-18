# 07 - 角色与权限管理 API

## 本节目标

- 实现角色 CRUD 接口
- 实现权限列表（树形结构）
- 掌握角色-权限关联操作

---

## 路由结构

```
GET    /api/roles              ← 角色列表（含权限+用户数）
POST   /api/roles              ← 创建角色
GET    /api/roles/:id          ← 获取单个角色
PATCH  /api/roles/:id          ← 更新角色（含权限重新分配）
DELETE /api/roles/:id          ← 删除角色

GET    /api/permissions        ← 权限列表（树形结构）
```

---

## `src/app/api/roles/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'

export async function GET() {
  const roles = await prisma.role.findMany({
    select: {
      id: true, name: true, description: true, createdAt: true,
      _count: { select: { userRoles: true } },  // 统计用户数
      rolePermissions: {
        select: { permission: { select: { id: true, name: true, type: true } } }
      }
    },
    orderBy: { createdAt: 'asc' }
  })
  return NextResponse.json({ data: roles })
}

export async function POST(req: NextRequest) {
  try {
    const { name, description, permissionIds } = await req.json()

    if (!name) return NextResponse.json({ error: '角色名称必填' }, { status: 400 })

    const existing = await prisma.role.findUnique({ where: { name } })
    if (existing) return NextResponse.json({ error: '角色名已存在' }, { status: 409 })

    const role = await prisma.role.create({
      data: {
        name, description,
        rolePermissions: permissionIds?.length
          ? { create: permissionIds.map((permissionId: string) => ({ permissionId })) }
          : undefined,
      },
      select: {
        id: true, name: true, description: true, createdAt: true,
        rolePermissions: { select: { permission: { select: { id: true, name: true } } } }
      }
    })

    return NextResponse.json({ data: role }, { status: 201 })
  } catch (error) {
    return NextResponse.json({ error: '服务器错误' }, { status: 500 })
  }
}
```

---

## `src/app/api/roles/[id]/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'

export async function GET(
  _req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params
  const role = await prisma.role.findUnique({
    where: { id },
    select: {
      id: true, name: true, description: true, createdAt: true,
      _count: { select: { userRoles: true } },
      rolePermissions: { select: { permission: true } }
    }
  })
  if (!role) return NextResponse.json({ error: '角色不存在' }, { status: 404 })
  return NextResponse.json({ data: role })
}

export async function PATCH(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    const { id } = await params
    const { name, description, permissionIds } = await req.json()

    const role = await prisma.role.update({
      where: { id },
      data: {
        ...(name && { name }),
        ...(description !== undefined && { description }),
        ...(permissionIds && {
          rolePermissions: {
            deleteMany: {},  // 清空旧权限
            create: permissionIds.map((permissionId: string) => ({ permissionId }))
          }
        }),
      },
      select: {
        id: true, name: true, description: true, updatedAt: true,
        rolePermissions: { select: { permission: { select: { id: true, name: true, type: true } } } }
      }
    })

    return NextResponse.json({ data: role })
  } catch (error) {
    return NextResponse.json({ error: '服务器错误' }, { status: 500 })
  }
}

export async function DELETE(
  _req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    const { id } = await params
    await prisma.rolePermission.deleteMany({ where: { roleId: id } })
    await prisma.userRole.deleteMany({ where: { roleId: id } })
    await prisma.role.delete({ where: { id } })
    return NextResponse.json({ message: '删除成功' })
  } catch (error) {
    return NextResponse.json({ error: '服务器错误' }, { status: 500 })
  }
}
```

---

## `src/app/api/permissions/route.ts`

```typescript
import { NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'

export async function GET() {
  const permissions = await prisma.permission.findMany({
    orderBy: [{ module: 'asc' }, { sort: 'asc' }],
    select: {
      id: true, name: true, description: true,
      type: true, module: true, action: true,
      path: true, icon: true, sort: true, parentId: true,
    }
  })

  return NextResponse.json({ data: buildTree(permissions) })
}

function buildTree(items: any[], parentId: string | null = null): any[] {
  return items
    .filter(item => item.parentId === parentId)
    .map(item => ({ ...item, children: buildTree(items, item.id) }))
}
```

---

## 测试

```bash
# 角色列表
curl http://localhost:3000/api/roles -b /tmp/cookies.txt

# 权限列表（树形）
curl http://localhost:3000/api/permissions -b /tmp/cookies.txt

# 创建角色（带权限）
curl -X POST http://localhost:3000/api/roles \
  -H "Content-Type: application/json" \
  -b /tmp/cookies.txt \
  -d '{"name":"tester","description":"测试员","permissionIds":["权限id1","权限id2"]}'

# 更新角色权限
curl -X PATCH http://localhost:3000/api/roles/<id> \
  -H "Content-Type: application/json" \
  -b /tmp/cookies.txt \
  -d '{"permissionIds":["权限id1"]}'
```

---

## 本节收获

- ✅ 角色 CRUD 完整接口
- ✅ 权限列表树形结构（buildTree 递归）
- ✅ 角色权限重新分配（deleteMany + create）
- ✅ `_count` 统计关联数量

## 下一步

- [ ] 08 - 管理后台 UI 搭建（shadcn/ui + 布局）
