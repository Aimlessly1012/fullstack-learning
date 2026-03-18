# 04 - RBAC 数据库设计

## 本节目标

- 理解 RBAC 权限模型
- 设计完整的权限数据库结构
- 初始化种子数据（角色/权限/管理员）

---

## RBAC 是什么

Role-Based Access Control（基于角色的访问控制）

```
用户(User) → 分配 → 角色(Role) → 拥有 → 权限(Permission)

Peko  → admin  角色 → 所有权限
Tom   → editor 角色 → 查看+编辑
Alice → viewer 角色 → 只读
```

---

## 权限类型设计

```
PermissionType
├── menu    菜单权限（哪些菜单可见）
├── page    页面权限（哪些页面可访问）
├── button  按钮权限（增/删/改/导出等操作）
└── api     接口权限（后端 API 保护）
```

---

## 完整 Schema

`prisma/schema.prisma`

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
}

model Department {
  id        String       @id @default(cuid())
  name      String
  parentId  String?
  parent    Department?  @relation("DeptTree", fields: [parentId], references: [id])
  children  Department[] @relation("DeptTree")
  users     User[]
  createdAt DateTime     @default(now())
  updatedAt DateTime     @updatedAt
}

model User {
  id           String      @id @default(cuid())
  email        String?     @unique
  password     String?
  name         String?
  avatar       String?
  status       Boolean     @default(true)
  departmentId String?
  department   Department? @relation(fields: [departmentId], references: [id])
  accounts     Account[]
  userRoles    UserRole[]
  createdAt    DateTime    @default(now())
  updatedAt    DateTime    @updatedAt
}

model Account {
  id                String   @id @default(cuid())
  userId            String
  provider          String
  providerAccountId String
  accessToken       String?
  user              User     @relation(fields: [userId], references: [id])
  createdAt         DateTime @default(now())

  @@unique([provider, providerAccountId])
}

model Role {
  id              String           @id @default(cuid())
  name            String           @unique
  description     String?
  userRoles       UserRole[]
  rolePermissions RolePermission[]
  createdAt       DateTime         @default(now())
  updatedAt       DateTime         @updatedAt
}

model Permission {
  id              String           @id @default(cuid())
  name            String           @unique  // "user:create" | "menu:dashboard"
  description     String?
  type            PermissionType   // menu | page | button | api
  module          String           // "user" | "role" | "dept"
  action          String           // "create" | "read" | "update" | "delete"
  path            String?          // 菜单路径 / API路径 / 按钮标识
  icon            String?          // 菜单图标
  sort            Int              @default(0)
  parentId        String?          // 菜单树形结构
  parent          Permission?      @relation("PermTree", fields: [parentId], references: [id])
  children        Permission[]     @relation("PermTree")
  rolePermissions RolePermission[]
  createdAt       DateTime         @default(now())
}

enum PermissionType {
  menu
  page
  button
  api
}

// 用户-角色 多对多
model UserRole {
  userId    String
  roleId    String
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  role      Role     @relation(fields: [roleId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now())

  @@id([userId, roleId])
}

// 角色-权限 多对多
model RolePermission {
  roleId       String
  permissionId String
  role         Role       @relation(fields: [roleId], references: [id], onDelete: Cascade)
  permission   Permission @relation(fields: [permissionId], references: [id], onDelete: Cascade)

  @@id([roleId, permissionId])
}
```

```bash
pnpm prisma migrate dev --name add-full-rbac
pnpm prisma generate
```

---

## Seed 初始化数据

`prisma/seed.ts` — 自动创建默认角色、权限、管理员账号

### 配置 `prisma.config.ts`

```typescript
import "dotenv/config"
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

```bash
pnpm add -D ts-node
pnpm prisma db seed
```

### 初始化结果

```
✅ 12 个权限（菜单权限 4 个 + 用户/角色按钮权限 8 个）
✅ 3 个角色：admin / editor / viewer
✅ 角色权限绑定
✅ 管理员账号：admin@admin.com / admin123
```

---

## 命令速查

| 命令 | 作用 |
|------|------|
| `prisma migrate dev --name xxx` | 同步 schema 到数据库（建/改表） |
| `prisma generate` | 生成 TypeScript 类型代码 |
| `prisma db seed` | 执行种子数据脚本 |
| `prisma studio` | 可视化查看数据库 |

---

## 本节收获

- ✅ RBAC 权限模型设计
- ✅ 4 种权限类型（menu/page/button/api）
- ✅ 部门树形结构（自关联）
- ✅ 权限树形结构（自关联）
- ✅ 多对多关联表（UserRole / RolePermission）
- ✅ Seed 初始化数据

## 下一步

- [ ] 05 - 鉴权中间件（JWT 验证 + 权限校验）
