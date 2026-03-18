# 07 - 角色与权限管理 API

## 📖 本课目标

- 实现角色的 CRUD（创建、读取、更新、删除）API
- 实现权限列表查询，支持树形结构返回
- 掌握 Prisma 中的关联数据统计（`_count`）
- 掌握角色-权限关联的嵌套写入与更新
- 理解「幂等设计」在权限更新中的应用
- 实现递归函数构建权限树

---

## 🤔 为什么这样选

### 为什么角色和权限要分开管理？

现实业务中，角色和权限是「多对多」关系：

- 一个角色可以拥有多个权限（比如「管理员」可以有「用户管理」「订单管理」「系统设置」等权限）
- 一个权限可以被多个角色共享（比如「查看订单」权限可能同时授予「销售员」「财务」「管理员」等角色）

如果把权限写死在角色字段里（比如 `role.permissions = 'user:create,user:delete'`），会出现：

- ❌ 权限变更时需要修改所有相关角色
- ❌ 无法灵活组合权限
- ❌ 难以查询「哪些角色拥有某个权限」

**正确的做法是通过中间表（`role_permissions`）建立多对多关系**，这样：

- ✅ 权限可复用
- ✅ 角色权限可动态调整
- ✅ 查询灵活（可以查角色的权限，也可以查权限的角色）

### 为什么更新权限用 `deleteMany + create`，不用 `upsert`？

假设当前角色拥有权限 `[A, B, C]`，现在要改成 `[A, D]`：

**方案 1：逐个 upsert（❌ 不推荐）**
```typescript
// 需要判断 B、C 是否该删除，D 是否该新增，逻辑复杂
for (const permId of newPermissions) {
  await prisma.rolePermission.upsert({ where: { ... }, create: { ... }, update: { ... } })
}
// 还要删除不在新列表中的 B、C...
```

**方案 2：先全删再全写（✅ 推荐）**
```typescript
rolePermissions: {
  deleteMany: {},  // 清空旧权限
  create: permissionIds.map(id => ({ permissionId: id }))  // 重新写入新权限
}
```

优势：
- ✅ **幂等性**：无论执行多少次，结果一致
- ✅ **逻辑简单**：不需要计算增量，直接覆盖
- ✅ **性能好**：Prisma 底层会优化为批量操作

这是 Prisma 官方推荐的「关联数据批量更新」模式。

### 为什么权限要用树形结构？

真实业务中的权限往往是层级化的：

```
用户管理
 ├─ 查看用户
 ├─ 创建用户
 ├─ 编辑用户
 └─ 删除用户
订单管理
 ├─ 查看订单
 ├─ 创建订单
 └─ 取消订单
```

扁平化存储会导致：
- ❌ 前端展示不直观（需要手动分组）
- ❌ 难以表达父子关系（比如「用户管理」模块下的所有权限）
- ❌ 批量授权不方便（比如「勾选用户管理，自动选中所有子权限」）

**用树形结构返回**：
- ✅ 前端可以直接渲染成折叠菜单
- ✅ 可以实现「全选父节点 = 全选子节点」
- ✅ 数据结构清晰，层级关系明确

---

## 💻 完整可运行代码

### 路由结构

```
GET    /api/roles              ← 获取角色列表（含权限数量、用户数量）
POST   /api/roles              ← 创建新角色（可同时分配权限）
GET    /api/roles/:id          ← 获取单个角色详情
PATCH  /api/roles/:id          ← 更新角色信息（可重新分配权限）
DELETE /api/roles/:id          ← 删除角色（级联删除关联数据）

GET    /api/permissions        ← 获取权限列表（树形结构）
```

### `src/app/api/roles/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'

/**
 * GET /api/roles
 * 获取所有角色列表，包含：
 * - 角色基本信息
 * - 拥有该角色的用户数量（_count）
 * - 该角色拥有的所有权限（rolePermissions）
 */
export async function GET() {
  try {
    const roles = await prisma.role.findMany({
      select: {
        id: true,
        name: true,
        description: true,
        createdAt: true,
        updatedAt: true,
        // 统计有多少用户拥有该角色
        _count: { 
          select: { userRoles: true } 
        },
        // 查询该角色拥有的所有权限
        rolePermissions: {
          select: {
            permission: {
              select: {
                id: true,
                name: true,
                type: true,
                module: true,
                action: true
              }
            }
          }
        }
      },
      orderBy: { createdAt: 'asc' }
    })

    return NextResponse.json({ data: roles })
  } catch (error) {
    console.error('获取角色列表失败:', error)
    return NextResponse.json(
      { error: '服务器内部错误' },
      { status: 500 }
    )
  }
}

/**
 * POST /api/roles
 * 创建新角色，支持同时分配权限
 * 请求体：
 * {
 *   "name": "测试员",              // 必填，角色名称（唯一）
 *   "description": "测试人员角色", // 选填，角色描述
 *   "permissionIds": ["id1", "id2"] // 选填，权限ID数组
 * }
 */
export async function POST(req: NextRequest) {
  try {
    const { name, description, permissionIds } = await req.json()

    // 1. 校验必填字段
    if (!name || typeof name !== 'string' || name.trim() === '') {
      return NextResponse.json(
        { error: '角色名称不能为空' },
        { status: 400 }
      )
    }

    // 2. 检查角色名是否已存在
    const existing = await prisma.role.findUnique({
      where: { name: name.trim() }
    })

    if (existing) {
      return NextResponse.json(
        { error: '角色名称已存在，请换一个名称' },
        { status: 409 } // 409 Conflict 表示资源冲突
      )
    }

    // 3. 创建角色，同时建立权限关联
    const role = await prisma.role.create({
      data: {
        name: name.trim(),
        description: description?.trim() || null,
        // 如果传了 permissionIds，就同时创建 rolePermissions 关联记录
        rolePermissions: permissionIds?.length
          ? {
              create: permissionIds.map((permissionId: string) => ({
                permissionId
              }))
            }
          : undefined, // 没有权限就不写入关联
      },
      // 返回创建好的角色及其权限
      select: {
        id: true,
        name: true,
        description: true,
        createdAt: true,
        rolePermissions: {
          select: {
            permission: {
              select: {
                id: true,
                name: true,
                type: true
              }
            }
          }
        }
      }
    })

    return NextResponse.json({ data: role }, { status: 201 })
  } catch (error) {
    console.error('创建角色失败:', error)
    return NextResponse.json(
      { error: '服务器内部错误' },
      { status: 500 }
    )
  }
}
```

**核心知识点：**

1. **`_count` 统计关联数量**
   ```typescript
   _count: { select: { userRoles: true } }
   // 会返回 _count: { userRoles: 5 }，表示有 5 个用户拥有该角色
   ```

2. **嵌套创建关联数据**
   ```typescript
   rolePermissions: {
     create: permissionIds.map(id => ({ permissionId: id }))
   }
   // 在创建角色的同时，批量创建 rolePermissions 中间表记录
   ```

3. **HTTP 状态码规范**
   - `200 OK`：成功获取数据
   - `201 Created`：成功创建资源
   - `400 Bad Request`：请求参数错误
   - `409 Conflict`：资源冲突（比如名称重复）
   - `500 Internal Server Error`：服务器内部错误

---

### `src/app/api/roles/[id]/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'

/**
 * GET /api/roles/:id
 * 获取指定角色的详细信息
 */
export async function GET(
  _req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    const { id } = await params

    const role = await prisma.role.findUnique({
      where: { id },
      select: {
        id: true,
        name: true,
        description: true,
        createdAt: true,
        updatedAt: true,
        // 统计用户数
        _count: { select: { userRoles: true } },
        // 查询权限列表
        rolePermissions: {
          select: {
            permission: {
              select: {
                id: true,
                name: true,
                description: true,
                type: true,
                module: true,
                action: true,
                path: true
              }
            }
          }
        }
      }
    })

    if (!role) {
      return NextResponse.json(
        { error: '角色不存在' },
        { status: 404 }
      )
    }

    return NextResponse.json({ data: role })
  } catch (error) {
    console.error('获取角色详情失败:', error)
    return NextResponse.json(
      { error: '服务器内部错误' },
      { status: 500 }
    )
  }
}

/**
 * PATCH /api/roles/:id
 * 更新角色信息，支持重新分配权限
 * 请求体：
 * {
 *   "name": "新角色名",           // 选填
 *   "description": "新描述",      // 选填
 *   "permissionIds": ["id1", "id2"] // 选填，传了就会覆盖旧权限
 * }
 */
export async function PATCH(
  req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    const { id } = await params
    const { name, description, permissionIds } = await req.json()

    // 1. 检查角色是否存在
    const existing = await prisma.role.findUnique({ where: { id } })
    if (!existing) {
      return NextResponse.json(
        { error: '角色不存在' },
        { status: 404 }
      )
    }

    // 2. 如果修改了角色名，检查新名称是否重复
    if (name && name !== existing.name) {
      const duplicate = await prisma.role.findUnique({
        where: { name: name.trim() }
      })
      if (duplicate) {
        return NextResponse.json(
          { error: '角色名称已存在' },
          { status: 409 }
        )
      }
    }

    // 3. 更新角色
    const role = await prisma.role.update({
      where: { id },
      data: {
        // 只更新传入的字段
        ...(name && { name: name.trim() }),
        ...(description !== undefined && { description: description?.trim() || null }),
        // 如果传了 permissionIds，就重新分配权限
        ...(permissionIds && {
          rolePermissions: {
            deleteMany: {},  // 先删除所有旧权限关联
            create: permissionIds.map((permissionId: string) => ({
              permissionId
            }))  // 再创建新的权限关联
          }
        }),
      },
      select: {
        id: true,
        name: true,
        description: true,
        updatedAt: true,
        rolePermissions: {
          select: {
            permission: {
              select: {
                id: true,
                name: true,
                type: true,
                module: true
              }
            }
          }
        }
      }
    })

    return NextResponse.json({ data: role })
  } catch (error) {
    console.error('更新角色失败:', error)
    return NextResponse.json(
      { error: '服务器内部错误' },
      { status: 500 }
    )
  }
}

/**
 * DELETE /api/roles/:id
 * 删除角色，级联删除关联数据
 * 删除顺序：rolePermissions -> userRoles -> role
 */
export async function DELETE(
  _req: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  try {
    const { id } = await params

    // 检查角色是否存在
    const existing = await prisma.role.findUnique({ where: { id } })
    if (!existing) {
      return NextResponse.json(
        { error: '角色不存在' },
        { status: 404 }
      )
    }

    // 级联删除顺序很重要！
    // 1. 先删除角色-权限关联
    await prisma.rolePermission.deleteMany({
      where: { roleId: id }
    })

    // 2. 再删除用户-角色关联
    await prisma.userRole.deleteMany({
      where: { roleId: id }
    })

    // 3. 最后删除角色本身
    await prisma.role.delete({
      where: { id }
    })

    return NextResponse.json({
      message: '角色删除成功'
    })
  } catch (error) {
    console.error('删除角色失败:', error)
    return NextResponse.json(
      { error: '服务器内部错误' },
      { status: 500 }
    )
  }
}
```

**核心知识点：**

1. **幂等更新设计**
   ```typescript
   rolePermissions: {
     deleteMany: {},  // 清空所有旧权限
     create: newIds.map(id => ({ permissionId: id }))  // 重新写入新权限
   }
   ```
   无论执行多少次，结果都是「角色拥有指定的这些权限」，不会重复也不会遗漏。

2. **级联删除顺序**
   必须先删除「引用者」，再删除「被引用者」：
   ```
   rolePermissions (引用 roleId) → 删除
   userRoles (引用 roleId) → 删除
   role (被引用) → 删除
   ```
   如果直接删除 `role`，Prisma 会报错「外键约束冲突」。

3. **可选字段更新**
   ```typescript
   data: {
     ...(name && { name }),  // 只有传了 name 才更新
     ...(description !== undefined && { description })  // 允许传空字符串清空描述
   }
   ```

---

### `src/app/api/permissions/route.ts`

```typescript
import { NextResponse } from 'next/server'
import { prisma } from '@/lib/prisma'

/**
 * Permission 类型定义
 */
interface Permission {
  id: string
  name: string
  description: string | null
  type: string
  module: string
  action: string | null
  path: string | null
  icon: string | null
  sort: number
  parentId: string | null
  children?: Permission[] // 递归子节点
}

/**
 * GET /api/permissions
 * 获取所有权限，返回树形结构
 * 
 * 扁平数据示例：
 * [
 *   { id: '1', name: '用户管理', parentId: null },
 *   { id: '2', name: '查看用户', parentId: '1' },
 *   { id: '3', name: '创建用户', parentId: '1' }
 * ]
 * 
 * 树形数据示例：
 * [
 *   {
 *     id: '1', name: '用户管理', parentId: null,
 *     children: [
 *       { id: '2', name: '查看用户', parentId: '1' },
 *       { id: '3', name: '创建用户', parentId: '1' }
 *     ]
 *   }
 * ]
 */
export async function GET() {
  try {
    // 1. 查询所有权限（扁平数据）
    const permissions = await prisma.permission.findMany({
      orderBy: [
        { module: 'asc' },  // 先按模块排序
        { sort: 'asc' }     // 再按 sort 字段排序
      ],
      select: {
        id: true,
        name: true,
        description: true,
        type: true,
        module: true,
        action: true,
        path: true,
        icon: true,
        sort: true,
        parentId: true,
      }
    })

    // 2. 转换为树形结构
    const tree = buildTree(permissions as Permission[])

    return NextResponse.json({ data: tree })
  } catch (error) {
    console.error('获取权限列表失败:', error)
    return NextResponse.json(
      { error: '服务器内部错误' },
      { status: 500 }
    )
  }
}

/**
 * 递归构建树形结构
 * @param items 扁平的权限列表
 * @param parentId 当前节点的父 ID（null 表示根节点）
 * @returns 树形结构数组
 * 
 * 工作原理：
 * 1. 找到所有 parentId === 当前 parentId 的节点（第一次调用时 parentId=null，找到所有根节点）
 * 2. 对每个节点，递归调用 buildTree(items, node.id) 找子节点
 * 3. 把子节点挂到 node.children 上
 * 4. 返回当前层级的所有节点
 * 
 * 示例执行过程：
 * buildTree([A, B, C, D], null)
 *   → 找到 A (parentId=null)
 *     → buildTree([A, B, C, D], A.id)
 *       → 找到 B (parentId=A.id)
 *         → buildTree([A, B, C, D], B.id) → []
 *       → 找到 C (parentId=A.id)
 *         → buildTree([A, B, C, D], C.id) → []
 *       → return [B, C]
 *     → A.children = [B, C]
 *   → return [A]
 */
function buildTree(items: Permission[], parentId: string | null = null): Permission[] {
  return items
    .filter(item => item.parentId === parentId)  // 找到当前层级的节点
    .map(item => ({
      ...item,
      children: buildTree(items, item.id)  // 递归找子节点
    }))
}
```

**核心知识点：**

1. **递归函数的执行逻辑**
   ```typescript
   function buildTree(items, parentId) {
     // 1. filter: 找到 parentId 匹配的节点
     // 2. map: 对每个节点递归调用 buildTree
     // 3. 返回当前层级的节点数组（每个节点都有 children 字段）
   }
   ```
   递归的「出口」是：当 `filter` 找不到子节点时，返回空数组 `[]`。

2. **为什么要在后端转换成树形结构？**
   - ✅ 前端可以直接渲染，不需要额外处理
   - ✅ 减少前端计算量（树形转换需要多次遍历）
   - ✅ API 语义清晰（「获取权限树」比「获取权限列表然后自己转树」更直观）

3. **`orderBy` 排序技巧**
   ```typescript
   orderBy: [
     { module: 'asc' },  // 先按模块排序
     { sort: 'asc' }     // 模块相同时按 sort 排序
   ]
   ```
   数组顺序就是排序优先级。

---

## ⚠️ 避坑指南

### 坑 1：删除角色时报「外键约束错误」

**错误信息：**
```
Foreign key constraint failed on the field: `roleId`
```

**原因：**
直接删除 `role`，但还有 `rolePermissions` 或 `userRoles` 引用着它。

**解决方案：**
按顺序删除：
```typescript
await prisma.rolePermission.deleteMany({ where: { roleId } })
await prisma.userRole.deleteMany({ where: { roleId } })
await prisma.role.delete({ where: { id: roleId } })
```

**更优雅的方案（推荐）：**
在 Prisma schema 中设置级联删除：
```prisma
model Role {
  id              String            @id @default(cuid())
  rolePermissions RolePermission[]  @relation(onDelete: Cascade) // 删除角色时自动删除关联权限
  userRoles       UserRole[]        @relation(onDelete: Cascade) // 删除角色时自动删除用户角色
}
```

---

### 坑 2：更新权限时遗漏或重复

**错误做法：**
```typescript
// 想法：遍历新权限，逐个 upsert
for (const permId of newPermissions) {
  await prisma.rolePermission.upsert({
    where: { roleId_permissionId: { roleId, permissionId: permId } },
    create: { roleId, permissionId: permId },
    update: {}
  })
}
// 问题：没有删除「不在新列表中的旧权限」，会遗漏删除
```

**正确做法：**
```typescript
rolePermissions: {
  deleteMany: {},  // 先全删
  create: newIds.map(id => ({ permissionId: id }))  // 再全写
}
```

**为什么推荐「全删全写」？**
- ✅ 逻辑简单，不需要计算增量
- ✅ 幂等性：执行多次结果一致
- ✅ Prisma 底层会优化成批量操作，性能不差

---

### 坑 3：`_count` 返回的是对象，不是数字

**错误用法：**
```typescript
const roles = await prisma.role.findMany({
  select: { _count: { select: { userRoles: true } } }
})

console.log(roles[0]._count)  // { userRoles: 5 }
console.log(roles[0]._count.userRoles)  // 5 ✅
```

**注意：**
`_count` 返回的是一个对象，需要通过 `._count.userRoles` 访问数量。

**前端使用：**
```typescript
// 显示「该角色有 5 个用户」
<span>{role._count.userRoles} 个用户</span>
```

---

### 坑 4：递归函数栈溢出

**场景：**
权限层级过深（比如 10 层嵌套），递归调用 `buildTree` 可能导致栈溢出。

**解决方案 1：限制层级深度**
```typescript
function buildTree(items: Permission[], parentId: string | null = null, depth = 0): Permission[] {
  if (depth > 10) return []  // 最多支持 10 层
  return items
    .filter(item => item.parentId === parentId)
    .map(item => ({
      ...item,
      children: buildTree(items, item.id, depth + 1)
    }))
}
```

**解决方案 2：改用迭代（适用于海量数据）**
```typescript
function buildTreeIterative(items: Permission[]): Permission[] {
  const map = new Map<string, Permission & { children: Permission[] }>()
  const roots: Permission[] = []

  // 1. 建立 id -> node 的映射
  items.forEach(item => {
    map.set(item.id, { ...item, children: [] })
  })

  // 2. 遍历所有节点，挂到父节点的 children 上
  items.forEach(item => {
    const node = map.get(item.id)!
    if (item.parentId === null) {
      roots.push(node)  // 根节点
    } else {
      const parent = map.get(item.parentId)
      if (parent) {
        parent.children.push(node)
      }
    }
  })

  return roots
}
```

对于一般业务（权限层级 ≤ 3 层），递归方案足够用。

---

### 坑 5：角色名称重复检查漏洞

**场景：**
用户提交 `"Admin"` 和 `" Admin "` (前后有空格)，数据库会认为是两个不同的角色。

**解决方案：**
```typescript
// 创建前 trim
const name = req.body.name.trim()

// 检查重复时也要 trim
const existing = await prisma.role.findUnique({
  where: { name }
})
```

**更严格的方案（推荐）：**
在数据库层面设置唯一约束 + 自动 trim：
```prisma
model Role {
  id   String @id @default(cuid())
  name String @unique  // 数据库层面保证唯一
}
```

在应用层 trim：
```typescript
name: name.trim().toLowerCase()  // 统一转小写，避免大小写导致的重复
```

---

## 🧠 核心概念大白话解释

### 1. `_count` 是什么？

`_count` 是 Prisma 提供的「关联计数」功能，用来统计某个关联关系有多少条记录。

**例子：**
```typescript
const role = await prisma.role.findUnique({
  where: { id: 'xxx' },
  select: {
    _count: {
      select: {
        userRoles: true,       // 有多少用户拥有该角色
        rolePermissions: true  // 该角色有多少个权限
      }
    }
  }
})

console.log(role._count)
// { userRoles: 5, rolePermissions: 12 }
```

**替代方案（不推荐）：**
```typescript
// 手动查询统计
const userCount = await prisma.userRole.count({ where: { roleId: 'xxx' } })
const permCount = await prisma.rolePermission.count({ where: { roleId: 'xxx' } })
```

用 `_count` 的好处：
- ✅ 一次查询搞定，不需要多次请求
- ✅ Prisma 底层会优化 SQL（用 `COUNT(*)` 子查询）
- ✅ 代码更简洁

---

### 2. 嵌套写入 vs 分步写入

**场景：**
创建角色时同时分配权限。

**方式 1：嵌套写入（推荐）**
```typescript
await prisma.role.create({
  data: {
    name: '测试员',
    rolePermissions: {
      create: [
        { permissionId: 'perm1' },
        { permissionId: 'perm2' }
      ]
    }
  }
})
```

**方式 2：分步写入**
```typescript
const role = await prisma.role.create({
  data: { name: '测试员' }
})

await prisma.rolePermission.createMany({
  data: [
    { roleId: role.id, permissionId: 'perm1' },
    { roleId: role.id, permissionId: 'perm2' }
  ]
})
```

**为什么推荐嵌套写入？**
- ✅ 原子性：一次事务完成，要么全成功，要么全失败
- ✅ 代码简洁：不需要拿到 `role.id` 再写第二次
- ✅ 性能更好：减少数据库往返次数

---

### 3. 幂等性设计

**什么是幂等性？**
无论执行多少次，结果都一样。

**例子 1：非幂等操作**
```typescript
// 每次执行都会增加 1
await prisma.user.update({
  where: { id: 'xxx' },
  data: { loginCount: { increment: 1 } }
})
```

**例子 2：幂等操作**
```typescript
// 无论执行多少次，loginCount 都是 10
await prisma.user.update({
  where: { id: 'xxx' },
  data: { loginCount: 10 }
})
```

**在权限更新中的应用：**
```typescript
// 幂等：无论执行多少次，角色都拥有 [A, D] 这两个权限
rolePermissions: {
  deleteMany: {},
  create: ['A', 'D'].map(id => ({ permissionId: id }))
}
```

---

### 4. 递归函数的本质

递归 = 自己调用自己 + 终止条件

**buildTree 的递归过程：**
```
调用栈：
buildTree(所有节点, null)           // 找根节点
  ├─ buildTree(所有节点, 根节点1.id)  // 找根节点1的子节点
  │    ├─ buildTree(所有节点, 子节点1.id)  // 找子节点1的子节点
  │    │    └─ return []  // 没有子节点了，返回空数组（终止）
  │    └─ buildTree(所有节点, 子节点2.id)
  │         └─ return []
  └─ buildTree(所有节点, 根节点2.id)
       └─ ...
```

**终止条件：**
```typescript
items.filter(item => item.parentId === parentId)
// 当找不到子节点时，filter 返回 []，map 也返回 []，递归结束
```

---

## 测试 API

### 1. 获取角色列表

```bash
curl http://localhost:3000/api/roles \
  -H "Cookie: session=你的session值"
```

**预期响应：**
```json
{
  "data": [
    {
      "id": "role_1",
      "name": "管理员",
      "description": "拥有所有权限",
      "createdAt": "2024-01-01T00:00:00.000Z",
      "updatedAt": "2024-01-01T00:00:00.000Z",
      "_count": { "userRoles": 3 },
      "rolePermissions": [
        {
          "permission": {
            "id": "perm_1",
            "name": "查看用户",
            "type": "menu",
            "module": "用户管理",
            "action": "view"
          }
        }
      ]
    }
  ]
}
```

---

### 2. 创建角色（同时分配权限）

```bash
curl -X POST http://localhost:3000/api/roles \
  -H "Content-Type: application/json" \
  -H "Cookie: session=你的session值" \
  -d '{
    "name": "测试员",
    "description": "负责测试工作",
    "permissionIds": ["perm_1", "perm_2"]
  }'
```

**预期响应：**
```json
{
  "data": {
    "id": "role_new",
    "name": "测试员",
    "description": "负责测试工作",
    "createdAt": "2024-01-01T00:00:00.000Z",
    "rolePermissions": [
      { "permission": { "id": "perm_1", "name": "查看用户", "type": "menu" } },
      { "permission": { "id": "perm_2", "name": "创建用户", "type": "button" } }
    ]
  }
}
```

---

### 3. 更新角色权限

```bash
curl -X PATCH http://localhost:3000/api/roles/role_new \
  -H "Content-Type: application/json" \
  -H "Cookie: session=你的session值" \
  -d '{
    "permissionIds": ["perm_3"]
  }'
```

**预期响应：**
```json
{
  "data": {
    "id": "role_new",
    "name": "测试员",
    "description": "负责测试工作",
    "updatedAt": "2024-01-01T01:00:00.000Z",
    "rolePermissions": [
      { "permission": { "id": "perm_3", "name": "删除用户", "type": "button" } }
    ]
  }
}
```

**注意：**旧权限 `perm_1` 和 `perm_2` 已被删除，只剩下 `perm_3`。

---

### 4. 获取权限列表（树形结构）

```bash
curl http://localhost:3000/api/permissions \
  -H "Cookie: session=你的session值"
```

**预期响应：**
```json
{
  "data": [
    {
      "id": "perm_module_1",
      "name": "用户管理",
      "type": "menu",
      "module": "用户管理",
      "parentId": null,
      "children": [
        {
          "id": "perm_1",
          "name": "查看用户",
          "type": "menu",
          "parentId": "perm_module_1",
          "children": []
        },
        {
          "id": "perm_2",
          "name": "创建用户",
          "type": "button",
          "parentId": "perm_module_1",
          "children": []
        }
      ]
    },
    {
      "id": "perm_module_2",
      "name": "订单管理",
      "type": "menu",
      "module": "订单管理",
      "parentId": null,
      "children": [
        {
          "id": "perm_5",
          "name": "查看订单",
          "type": "menu",
          "parentId": "perm_module_2",
          "children": []
        }
      ]
    }
  ]
}
```

---

### 5. 删除角色

```bash
curl -X DELETE http://localhost:3000/api/roles/role_new \
  -H "Cookie: session=你的session值"
```

**预期响应：**
```json
{
  "message": "角色删除成功"
}
```

---

## ✅ 本课收获

1. ✅ 掌握了 Prisma 的 `_count` 关联统计
2. ✅ 理解了多对多关系的嵌套写入（`create`）
3. ✅ 学会了「全删全写」的幂等更新模式（`deleteMany + create`）
4. ✅ 实现了递归函数 `buildTree` 构建树形结构
5. ✅ 掌握了级联删除的正确顺序
6. ✅ 避开了 5 个常见的坑

---

## 下一步

- [ ] 08 - 管理后台 UI 搭建（shadcn/ui + 路由分组 + 侧边栏）

**提示：**
下一课我们会用 shadcn/ui 搭建管理后台界面，包括：
- 登录页面
- 侧边栏导航
- 用户管理列表
- 角色管理列表

到时候会用到本课实现的所有 API。
