# 04 - RBAC 数据库设计

## 📖 本课目标

- 深入理解 RBAC（基于角色的访问控制）权限模型
- 设计生产级的权限数据库结构（7 张表 + 1 个枚举）
- 掌握 Prisma Schema 自关联树形结构设计
- 实现完整的种子数据初始化（12 个权限 + 3 个角色）
- 理解权限类型的划分逻辑（menu/page/button/api）
- 配置 Prisma 7 的 seed 执行方式

---

## 🤔 为什么用 RBAC？

### 场景对比：直接给用户分权限 vs 角色管理

#### ❌ 直接给用户分权限的问题

```
公司有 100 个员工，每人都要设置一遍权限？

Peko  → [查看用户, 创建用户, 删除用户, 编辑用户, 查看角色, ...]  ← 15 个权限
Tom   → [查看用户, 创建用户, 删除用户, 编辑用户, 查看角色, ...]  ← 复制 15 个
Alice → [查看用户, 创建用户, 删除用户, 编辑用户, 查看角色, ...]  ← 再复制 15 个
...   → 重复 97 次
```

**问题来了：**
- 新员工入职要配置 15 个权限 → **太麻烦**
- 调整权限政策要改 100 个人 → **容易遗漏**
- 张三离职了权限没回收 → **安全隐患**

---

#### ✅ 使用 RBAC 的优势

```
用户(User) → 分配 → 角色(Role) → 拥有 → 权限(Permission)

Peko  → admin  角色 → [所有权限]
Tom   → editor 角色 → [查看+编辑，不能删除]
Alice → viewer 角色 → [只读]

新员工入职？分配 editor 角色，搞定！
调整权限？改 editor 角色，100 个编辑同步生效！
离职？移除角色，所有权限自动失效！
```

**关键优势：**
1. **批量管理**：一个角色管理一批人
2. **集中配置**：权限修改一次全员生效
3. **安全合规**：角色回收自动清空权限
4. **业务清晰**：角色名对应真实岗位（运营、编辑、财务）

---

## 🧠 核心概念大白话解释

### RBAC 三要素

```
┌─────────┐      ┌──────┐      ┌────────────┐
│  User   │─────→│ Role │─────→│ Permission │
│ (用户)   │  分配  │(角色)│  拥有  │   (权限)    │
└─────────┘      └──────┘      └────────────┘

用户：Peko（真实的人）
角色：admin（岗位抽象，如：总监、编辑、实习生）
权限：user:create（具体能做什么，如：创建用户）
```

### 权限命名规范

```
格式：module:action

user:create    → 用户模块的创建权限
user:delete    → 用户模块的删除权限
role:update    → 角色模块的更新权限
menu:dashboard → 菜单权限（显示仪表盘菜单）
```

**为什么用冒号分隔？**
- 方便代码解析：`const [module, action] = perm.split(':')`
- 便于权限分组展示（按模块折叠）
- 语义清晰：一眼看出是哪个模块的什么操作

---

### 四种权限类型

```typescript
enum PermissionType {
  menu,    // 菜单权限：控制左侧菜单是否显示
  page,    // 页面权限：控制能否访问某个页面
  button,  // 按钮权限：控制页面上的增删改按钮显示
  api      // 接口权限：后端 API 鉴权（最重要！）
}
```

#### 实际使用场景

```typescript
// 1. menu 菜单权限（前端渲染左侧导航栏时过滤）
if (hasPermission('menu:user-management')) {
  showMenuItem('用户管理')  // 有权限才显示菜单
}

// 2. page 页面权限（访问 /users 页面时检查）
if (!hasPermission('page:user-list')) {
  redirect('/403')  // 无权限跳转到禁止页
}

// 3. button 按钮权限（前端条件渲染）
{hasPermission('user:create') && <Button>新建用户</Button>}

// 4. api 接口权限（后端中间件验证）
if (!hasPermission('user:delete')) {
  return res.status(403).json({ error: '无权限删除用户' })
}
```

**为什么要分四种？**
- **菜单权限**：藏起来用户看不到的功能
- **页面权限**：即使 URL 输入也进不去
- **按钮权限**：看得到页面但没操作按钮
- **接口权限**：防止前端绕过直接调 API（最重要！）

> ⚠️ **重点：前端权限只是 UI 优化，后端接口权限才是真正的安全防线！**

---

## 💻 完整数据库 Schema

### 整体设计思路

```
7 张表设计：

核心业务表：
- User        (用户表)
- Department  (部门表，树形结构)
- Account     (第三方登录关联表)

权限相关：
- Role        (角色表)
- Permission  (权限表，树形结构)

多对多关联：
- UserRole       (用户-角色关联表)
- RolePermission (角色-权限关联表)
```

---

### `prisma/schema.prisma`

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ========================================
// 部门表（树形结构，支持多级部门）
// ========================================
model Department {
  id        String       @id @default(cuid())
  name      String       // 部门名称
  parentId  String?      // 父部门 ID（null 表示顶级部门）
  
  // 自关联：parent 是上级部门，children 是下级部门列表
  parent    Department?  @relation("DeptTree", fields: [parentId], references: [id])
  children  Department[] @relation("DeptTree")
  
  users     User[]       // 一个部门有多个员工
  
  createdAt DateTime     @default(now())
  updatedAt DateTime     @updatedAt
  
  @@map("departments")  // 数据库表名
}

// ========================================
// 用户表（核心业务主体）
// ========================================
model User {
  id           String      @id @default(cuid())
  email        String?     @unique        // 邮箱（可空，支持第三方登录没邮箱的情况）
  password     String?                    // 密码哈希值（第三方登录时为空）
  name         String?                    // 用户昵称
  avatar       String?                    // 头像 URL
  status       Boolean     @default(true) // 账号状态（true=启用 false=禁用）
  departmentId String?                    // 所属部门
  
  department   Department? @relation(fields: [departmentId], references: [id])
  accounts     Account[]   // 一个用户可绑定多个第三方账号（GitHub、Google 等）
  userRoles    UserRole[]  // 一个用户可以有多个角色
  
  createdAt    DateTime    @default(now())
  updatedAt    DateTime    @updatedAt
  
  @@map("users")
}

// ========================================
// 第三方账号关联表（支持多种登录方式）
// ========================================
model Account {
  id                String   @id @default(cuid())
  userId            String                    // 关联的用户 ID
  provider          String                    // 登录提供商：credentials | github | google
  providerAccountId String                    // 第三方平台的用户 ID
  accessToken       String?                   // OAuth 访问令牌（可选）
  
  user              User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  createdAt         DateTime @default(now())
  
  // 确保同一个第三方账号只能绑定一次
  @@unique([provider, providerAccountId])
  @@map("accounts")
}

// ========================================
// 角色表（如：admin、editor、viewer）
// ========================================
model Role {
  id              String           @id @default(cuid())
  name            String           @unique  // 角色唯一标识：admin、editor
  description     String?                   // 角色描述：超级管理员、普通编辑
  
  userRoles       UserRole[]                // 哪些用户拥有此角色
  rolePermissions RolePermission[]          // 此角色拥有哪些权限
  
  createdAt       DateTime         @default(now())
  updatedAt       DateTime         @updatedAt
  
  @@map("roles")
}

// ========================================
// 权限表（树形结构，支持菜单层级）
// ========================================
model Permission {
  id          String           @id @default(cuid())
  name        String           @unique  // 权限唯一标识：user:create | menu:dashboard
  description String?                   // 权限描述：创建用户 | 仪表盘菜单
  type        PermissionType            // 权限类型：menu | page | button | api
  module      String                    // 所属模块：user | role | dept
  action      String                    // 操作类型：create | read | update | delete
  path        String?                   // 路径（菜单路径 | API 路径 | 按钮标识）
  icon        String?                   // 菜单图标（仅 type=menu 时使用）
  sort        Int              @default(0) // 排序号（菜单显示顺序）
  parentId    String?                   // 父权限 ID（菜单树形结构）
  
  // 自关联：parent 是上级菜单，children 是子菜单列表
  parent          Permission?      @relation("PermTree", fields: [parentId], references: [id])
  children        Permission[]     @relation("PermTree")
  
  rolePermissions RolePermission[] // 哪些角色拥有此权限
  
  createdAt       DateTime         @default(now())
  
  @@map("permissions")
}

// ========================================
// 权限类型枚举
// ========================================
enum PermissionType {
  menu    // 菜单权限：控制左侧导航栏显示
  page    // 页面权限：控制能否访问页面
  button  // 按钮权限：控制页面内操作按钮
  api     // 接口权限：后端 API 鉴权（最重要）
}

// ========================================
// 用户-角色 多对多关联表
// ========================================
model UserRole {
  userId    String
  roleId    String
  
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  role      Role     @relation(fields: [roleId], references: [id], onDelete: Cascade)
  
  createdAt DateTime @default(now())
  
  // 联合主键：同一个用户同一个角色只能关联一次
  @@id([userId, roleId])
  @@map("user_roles")
}

// ========================================
// 角色-权限 多对多关联表
// ========================================
model RolePermission {
  roleId       String
  permissionId String
  
  role         Role       @relation(fields: [roleId], references: [id], onDelete: Cascade)
  permission   Permission @relation(fields: [permissionId], references: [id], onDelete: Cascade)
  
  // 联合主键：同一个角色同一个权限只能关联一次
  @@id([roleId, permissionId])
  @@map("role_permissions")
}
```

---

### 表设计理由详解

#### 1. User 表设计要点

```prisma
model User {
  email    String?  @unique   // 为什么可空？支持第三方登录没邮箱
  password String?             // 为什么可空？GitHub 登录不需要密码
  status   Boolean  @default(true)  // 软删除：禁用而不是物理删除
}
```

**为什么 email 和 password 都可空？**
- 场景：用户用 GitHub 登录，GitHub API 没返回邮箱
- 解决：email 设为 nullable，后续可以让用户补填
- 密码同理：第三方登录不需要密码

**为什么用 status 而不是 deleted_at？**
- `status = false` 代表账号禁用，随时可恢复
- 真正删除用 `DELETE` 语句，不留数据

---

#### 2. Account 表设计要点

```prisma
model Account {
  provider          String  // credentials | github | google
  providerAccountId String  // 第三方平台用户 ID
  
  @@unique([provider, providerAccountId])  // 防止重复绑定
}
```

**为什么需要 Account 表？**
- 一个用户可以绑定多个登录方式：
  ```
  User: Peko
    ├─ Account 1: provider=credentials, providerAccountId=peko@test.com
    ├─ Account 2: provider=github,      providerAccountId=12345678
    └─ Account 3: provider=google,      providerAccountId=abcdef123
  ```
- 下次用 GitHub 登录时，通过 `providerAccountId` 找到对应的 User

**为什么要 unique 约束？**
- 防止同一个 GitHub 账号绑定到多个用户
- 保证 `provider + providerAccountId` 全局唯一

---

#### 3. Department 自关联树形结构

```prisma
model Department {
  parentId  String?
  parent    Department?  @relation("DeptTree", fields: [parentId], references: [id])
  children  Department[] @relation("DeptTree")
}
```

**树形结构示例：**
```
技术部 (parentId = null)
  ├─ 前端组 (parentId = 技术部.id)
  │   ├─ React 小组
  │   └─ Vue 小组
  └─ 后端组 (parentId = 技术部.id)
      ├─ Java 小组
      └─ Node.js 小组
```

**查询示例：**
```typescript
// 查询某部门的所有下级部门
const dept = await prisma.department.findUnique({
  where: { id: '技术部id' },
  include: {
    children: true  // 返回前端组、后端组
  }
})

// 查询某部门的上级部门
const frontendDept = await prisma.department.findUnique({
  where: { id: '前端组id' },
  include: {
    parent: true  // 返回技术部
  }
})
```

---

#### 4. Permission 自关联树形结构

```prisma
model Permission {
  parentId  String?
  parent    Permission?  @relation("PermTree", fields: [parentId], references: [id])
  children  Permission[] @relation("PermTree")
}
```

**菜单树形结构示例：**
```
系统管理 (type=menu, parentId=null)
  ├─ 用户管理 (type=menu, parentId=系统管理.id)
  │   ├─ 查看用户 (type=button, parentId=用户管理.id)
  │   ├─ 创建用户 (type=button)
  │   └─ 删除用户 (type=button)
  └─ 角色管理 (type=menu)
      ├─ 查看角色 (type=button)
      └─ 编辑角色 (type=button)
```

**前端渲染菜单代码：**
```typescript
// 递归渲染树形菜单
function renderMenu(permissions: Permission[]) {
  return permissions
    .filter(p => p.type === 'menu' && !p.parentId)  // 只取顶级菜单
    .map(menu => (
      <Menu.Item key={menu.id} icon={menu.icon}>
        {menu.description}
        {menu.children && renderMenu(menu.children)}  // 递归渲染子菜单
      </Menu.Item>
    ))
}
```

---

#### 5. UserRole 和 RolePermission 设计要点

```prisma
// 用户-角色 多对多
model UserRole {
  userId String
  roleId String
  @@id([userId, roleId])  // 联合主键
}

// 角色-权限 多对多
model RolePermission {
  roleId       String
  permissionId String
  @@id([roleId, permissionId])
}
```

**为什么用联合主键？**
- 防止重复关联：同一个用户不能重复添加同一个角色
- 查询优化：`(userId, roleId)` 天然索引，查询快

**级联删除逻辑：**
```prisma
onDelete: Cascade  // 用户删除时，自动删除 UserRole 记录
```
- 删除用户 → 自动清空该用户的所有角色关联
- 删除角色 → 自动清空该角色下的所有用户关联
- 保证数据一致性，不会有孤儿数据

---

## ⚠️ 避坑指南

### 坑 1：忘记运行 `prisma generate`

```bash
❌ 错误现象：
import { PrismaClient } from '@prisma/client'
// 报错：Cannot find module '@prisma/client'

✅ 解决：
pnpm prisma generate  # 生成 TypeScript 类型代码
```

**原因：**
- `prisma migrate` 只同步数据库表结构
- `prisma generate` 才生成 TS 类型和 Prisma Client 代码

---

### 坑 2：修改 schema 后忘记迁移

```bash
❌ 错误现象：
修改了 schema.prisma，代码运行报错：
PrismaClientValidationError: Unknown field `newField`

✅ 解决：
pnpm prisma migrate dev --name add-new-field
pnpm prisma generate
```

**正确流程：**
1. 修改 `schema.prisma`
2. 运行 `migrate dev` 生成迁移文件并同步数据库
3. 运行 `generate` 重新生成类型代码
4. 重启开发服务器

---

### 坑 3：自关联字段命名冲突

```prisma
❌ 错误写法：
model Department {
  parent   Department?  @relation(fields: [parentId], references: [id])
  children Department[] @relation(fields: [parentId], references: [id])  // ❌ 重复关系
}

✅ 正确写法：
model Department {
  parent   Department?  @relation("DeptTree", fields: [parentId], references: [id])
  children Department[] @relation("DeptTree")  // ✅ 同一个关系名
}
```

**原因：**
- 自关联必须用 `@relation("名称")` 指定关系名
- `parent` 和 `children` 共享同一个关系名

---

### 坑 4：枚举值修改后迁移失败

```bash
❌ 错误现象：
修改 enum PermissionType 后运行 migrate dev 报错：
Error: Enum value cannot be changed because existing data uses it

✅ 解决：
# 方案 1：开发环境直接重置
pnpm prisma migrate reset  # 会清空数据！

# 方案 2：生产环境手动迁移
pnpm prisma migrate dev --create-only
# 编辑生成的 migration.sql，添加数据转换逻辑
pnpm prisma migrate deploy
```

---

### 坑 5：seed 数据重复执行报错

```bash
❌ 错误现象：
pnpm prisma db seed
报错：Unique constraint failed on the fields: (`email`)

✅ 解决：seed 脚本加幂等性检查
const existing = await prisma.user.findUnique({ where: { email } })
if (!existing) {
  await prisma.user.create({ data: { email, ... } })
}
```

---

## 💻 Seed 初始化数据

### Prisma 7 配置方式

**Prisma 6 及以前：**
```json
// package.json
{
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  }
}
```

**Prisma 7 新方式：**
```typescript
// prisma.config.ts
import { defineConfig, env } from "prisma/config"

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: {
    path: "prisma/migrations",
    seed: "ts-node --compiler-options {\"module\":\"CommonJS\"} prisma/seed.ts",
  },
  datasource: {
    url: env("DATABASE_URL"),
  },
})
```

**为什么要指定 `module:CommonJS`？**
- ts-node 默认用 ESM 模式
- Prisma Client 目前只支持 CommonJS
- 不加参数会报 `require() of ES Module` 错误

---

### `prisma/seed.ts` 完整代码

```typescript
import { PrismaClient, PermissionType } from '@prisma/client'
import bcrypt from 'bcryptjs'

const prisma = new PrismaClient()

async function main() {
  console.log('🌱 开始初始化种子数据...\n')

  // ========================================
  // 1. 创建 12 个权限（4 个菜单 + 8 个按钮）
  // ========================================
  console.log('📝 创建权限...')
  
  const permissions = [
    // 菜单权限（树形结构）
    { name: 'menu:system',    description: '系统管理', type: PermissionType.menu, module: 'system', action: 'read', path: '/system', icon: 'SettingOutlined', sort: 1, parentId: null },
    { name: 'menu:user',      description: '用户管理', type: PermissionType.menu, module: 'user',   action: 'read', path: '/system/users', icon: 'UserOutlined', sort: 2, parentId: null },
    { name: 'menu:role',      description: '角色管理', type: PermissionType.menu, module: 'role',   action: 'read', path: '/system/roles', icon: 'TeamOutlined', sort: 3, parentId: null },
    { name: 'menu:permission', description: '权限管理', type: PermissionType.menu, module: 'permission', action: 'read', path: '/system/permissions', icon: 'SafetyOutlined', sort: 4, parentId: null },
    
    // 用户管理按钮权限
    { name: 'user:create', description: '创建用户', type: PermissionType.button, module: 'user', action: 'create', path: 'btn-user-create', parentId: null },
    { name: 'user:read',   description: '查看用户', type: PermissionType.button, module: 'user', action: 'read',   path: 'btn-user-read',   parentId: null },
    { name: 'user:update', description: '编辑用户', type: PermissionType.button, module: 'user', action: 'update', path: 'btn-user-update', parentId: null },
    { name: 'user:delete', description: '删除用户', type: PermissionType.button, module: 'user', action: 'delete', path: 'btn-user-delete', parentId: null },
    
    // 角色管理按钮权限
    { name: 'role:create', description: '创建角色', type: PermissionType.button, module: 'role', action: 'create', path: 'btn-role-create', parentId: null },
    { name: 'role:read',   description: '查看角色', type: PermissionType.button, module: 'role', action: 'read',   path: 'btn-role-read',   parentId: null },
    { name: 'role:update', description: '编辑角色', type: PermissionType.button, module: 'role', action: 'update', path: 'btn-role-update', parentId: null },
    { name: 'role:delete', description: '删除角色', type: PermissionType.button, module: 'role', action: 'delete', path: 'btn-role-delete', parentId: null },
  ]
  
  const createdPermissions: Record<string, string> = {}
  for (const perm of permissions) {
    const existing = await prisma.permission.findUnique({ where: { name: perm.name } })
    if (!existing) {
      const created = await prisma.permission.create({ data: perm })
      createdPermissions[perm.name] = created.id
      console.log(`  ✅ ${perm.description} (${perm.type})`)
    } else {
      createdPermissions[perm.name] = existing.id
      console.log(`  ⏭️  ${perm.description} 已存在`)
    }
  }
  
  // ========================================
  // 2. 创建 3 个角色
  // ========================================
  console.log('\n👥 创建角色...')
  
  const roles = [
    { name: 'admin',  description: '超级管理员' },
    { name: 'editor', description: '普通编辑' },
    { name: 'viewer', description: '访客（只读）' },
  ]
  
  const createdRoles: Record<string, string> = {}
  for (const role of roles) {
    const existing = await prisma.role.findUnique({ where: { name: role.name } })
    if (!existing) {
      const created = await prisma.role.create({ data: role })
      createdRoles[role.name] = created.id
      console.log(`  ✅ ${role.description}`)
    } else {
      createdRoles[role.name] = existing.id
      console.log(`  ⏭️  ${role.description} 已存在`)
    }
  }
  
  // ========================================
  // 3. 分配权限给角色
  // ========================================
  console.log('\n🔑 分配权限...')
  
  // admin 角色：所有权限
  const allPermissionIds = Object.values(createdPermissions)
  await prisma.rolePermission.deleteMany({ where: { roleId: createdRoles.admin } })
  await prisma.rolePermission.createMany({
    data: allPermissionIds.map(pid => ({ roleId: createdRoles.admin, permissionId: pid })),
    skipDuplicates: true,
  })
  console.log(`  ✅ admin → 所有权限 (${allPermissionIds.length} 个)`)
  
  // editor 角色：查看+编辑（不能删除）
  const editorPermissions = [
    'menu:system', 'menu:user', 'menu:role',
    'user:read', 'user:update', 'user:create',
    'role:read', 'role:update',
  ].map(name => createdPermissions[name])
  await prisma.rolePermission.deleteMany({ where: { roleId: createdRoles.editor } })
  await prisma.rolePermission.createMany({
    data: editorPermissions.map(pid => ({ roleId: createdRoles.editor, permissionId: pid })),
    skipDuplicates: true,
  })
  console.log(`  ✅ editor → 查看+编辑权限 (${editorPermissions.length} 个)`)
  
  // viewer 角色：只读
  const viewerPermissions = [
    'menu:system', 'menu:user', 'menu:role',
    'user:read', 'role:read',
  ].map(name => createdPermissions[name])
  await prisma.rolePermission.deleteMany({ where: { roleId: createdRoles.viewer } })
  await prisma.rolePermission.createMany({
    data: viewerPermissions.map(pid => ({ roleId: createdRoles.viewer, permissionId: pid })),
    skipDuplicates: true,
  })
  console.log(`  ✅ viewer → 只读权限 (${viewerPermissions.length} 个)`)
  
  // ========================================
  // 4. 创建管理员账号
  // ========================================
  console.log('\n👤 创建管理员账号...')
  
  const adminEmail = 'admin@admin.com'
  const adminPassword = await bcrypt.hash('admin123', 10)
  
  let adminUser = await prisma.user.findUnique({ where: { email: adminEmail } })
  if (!adminUser) {
    adminUser = await prisma.user.create({
      data: {
        email: adminEmail,
        password: adminPassword,
        name: '超级管理员',
        status: true,
        accounts: {
          create: {
            provider: 'credentials',
            providerAccountId: adminEmail,
          }
        }
      }
    })
    console.log(`  ✅ 创建管理员：${adminEmail} / admin123`)
  } else {
    console.log(`  ⏭️  管理员已存在`)
  }
  
  // ========================================
  // 5. 分配 admin 角色给管理员
  // ========================================
  const existingUserRole = await prisma.userRole.findUnique({
    where: {
      userId_roleId: {
        userId: adminUser.id,
        roleId: createdRoles.admin,
      }
    }
  })
  
  if (!existingUserRole) {
    await prisma.userRole.create({
      data: {
        userId: adminUser.id,
        roleId: createdRoles.admin,
      }
    })
    console.log(`  ✅ 分配 admin 角色`)
  } else {
    console.log(`  ⏭️  已拥有 admin 角色`)
  }
  
  console.log('\n✨ 种子数据初始化完成！\n')
  console.log('📊 统计：')
  console.log(`  - 权限：${permissions.length} 个`)
  console.log(`  - 角色：${roles.length} 个`)
  console.log(`  - 管理员：1 个 (${adminEmail})`)
}

main()
  .catch((e) => {
    console.error('❌ 初始化失败:', e)
    process.exit(1)
  })
  .finally(async () => {
    await prisma.$disconnect()
  })
```

**安装依赖：**
```bash
pnpm add -D ts-node @types/node
pnpm add bcryptjs
pnpm add -D @types/bcryptjs
```

**执行 seed：**
```bash
pnpm prisma db seed
```

**预期输出：**
```
🌱 开始初始化种子数据...

📝 创建权限...
  ✅ 系统管理 (menu)
  ✅ 用户管理 (menu)
  ✅ 角色管理 (menu)
  ✅ 权限管理 (menu)
  ✅ 创建用户 (button)
  ✅ 查看用户 (button)
  ✅ 编辑用户 (button)
  ✅ 删除用户 (button)
  ✅ 创建角色 (button)
  ✅ 查看角色 (button)
  ✅ 编辑角色 (button)
  ✅ 删除角色 (button)

👥 创建角色...
  ✅ 超级管理员
  ✅ 普通编辑
  ✅ 访客（只读）

🔑 分配权限...
  ✅ admin → 所有权限 (12 个)
  ✅ editor → 查看+编辑权限 (8 个)
  ✅ viewer → 只读权限 (5 个)

👤 创建管理员账号...
  ✅ 创建管理员：admin@admin.com / admin123
  ✅ 分配 admin 角色

✨ 种子数据初始化完成！

📊 统计：
  - 权限：12 个
  - 角色：3 个
  - 管理员：1 个 (admin@admin.com)
```

---

## 🧠 Seed 数据设计逻辑

### 权限设计思路

```typescript
// 4 个菜单权限（树形结构的顶级节点）
menu:system      → 系统管理菜单
menu:user        → 用户管理菜单
menu:role        → 角色管理菜单
menu:permission  → 权限管理菜单

// 用户模块 4 个按钮权限（CRUD）
user:create  → 新建用户按钮
user:read    → 查看用户详情
user:update  → 编辑用户按钮
user:delete  → 删除用户按钮

// 角色模块 4 个按钮权限
role:create  → 新建角色按钮
role:read    → 查看角色详情
role:update  → 编辑角色按钮
role:delete  → 删除角色按钮
```

### 角色权限分配逻辑

| 角色 | 权限范围 | 使用场景 |
|------|---------|---------|
| **admin** | 所有 12 个权限 | 超级管理员，全部操作权限 |
| **editor** | 8 个权限（查看+编辑+创建） | 普通运营，能看能改但不能删 |
| **viewer** | 5 个权限（只读） | 访客/实习生，只能看不能改 |

**设计原则：**
- **最小权限原则**：默认给最少权限，需要再加
- **菜单权限在前**：没菜单权限连入口都看不到
- **按钮权限在后**：有菜单才需要按钮权限

---

## 💻 数据库操作命令速查

```bash
# 1. 同步 schema 到数据库（生成迁移文件）
pnpm prisma migrate dev --name init-rbac

# 2. 生成 TypeScript 类型代码
pnpm prisma generate

# 3. 执行 seed 初始化数据
pnpm prisma db seed

# 4. 可视化查看数据库（浏览器打开）
pnpm prisma studio

# 5. 重置数据库（清空所有数据并重新迁移）
pnpm prisma migrate reset  # 会自动执行 seed

# 6. 查看当前数据库状态
pnpm prisma migrate status

# 7. 部署迁移到生产环境（不生成新迁移）
pnpm prisma migrate deploy
```

---

## ✅ 本课收获

### 知识点清单

- ✅ 理解 RBAC 权限模型（用户→角色→权限）
- ✅ 掌握为什么用 RBAC 而不是直接给用户分权限
- ✅ 设计 7 张表的完整权限数据库结构
- ✅ 理解 4 种权限类型（menu/page/button/api）的区别和使用场景
- ✅ 掌握 Prisma 自关联树形结构（Department 树、Permission 树）
- ✅ 理解多对多关联表设计（UserRole、RolePermission）
- ✅ 掌握联合主键和级联删除
- ✅ 配置 Prisma 7 的 seed 执行方式（prisma.config.ts）
- ✅ 编写幂等性 seed 脚本（可重复执行不报错）
- ✅ 理解权限命名规范（module:action）

### 可复用代码

- ✅ 完整的 RBAC Schema（可直接用于生产项目）
- ✅ Seed 初始化脚本（12 个权限 + 3 个角色 + 管理员账号）
- ✅ 自关联树形结构查询示例
- ✅ Prisma 配置文件（prisma.config.ts）

---

## 🚀 下一步

在下一课 **05-鉴权中间件与用户API** 中，我们将：

1. 实现 Next.js 全局鉴权中间件（JWT 验证）
2. 理解 Edge Runtime 和为什么要用 jose 而不是 jsonwebtoken
3. 实现用户列表 API（分页、搜索、关联查询）
4. 掌握 Prisma select 安全字段过滤
5. 学习中间件注入请求头（x-user-id）的技巧

**预习建议：**
- 了解 JWT 令牌结构（Header.Payload.Signature）
- 阅读 Next.js Middleware 文档
- 思考：为什么前端权限控制不够，必须有后端接口权限？

**动手作业：**
1. 运行 `pnpm prisma studio` 查看初始化的数据
2. 尝试手动添加一个新权限 `dept:create`
3. 创建一个新角色 `hr`，分配部门管理权限
4. 思考：如果要支持「数据权限」（只能看自己部门的数据），应该怎么设计？

---

## 📚 扩展阅读

- [Prisma Schema 官方文档](https://www.prisma.io/docs/concepts/components/prisma-schema)
- [RBAC 权限模型详解](https://en.wikipedia.org/wiki/Role-based_access_control)
- [为什么 RBAC 是企业应用的标配](https://www.okta.com/identity-101/role-based-access-control-rbac/)
- [Prisma 自关联关系](https://www.prisma.io/docs/concepts/components/prisma-schema/relations/self-relations)

---

**🎉 恭喜！你已经掌握了生产级的 RBAC 数据库设计！**