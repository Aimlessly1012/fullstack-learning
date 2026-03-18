# 全栈开发学习规划（权限管理系统实战版）

## 🎯 目标项目

**RBAC 权限管理系统**
- 用户管理、角色管理、权限控制
- JWT 认证 + 中间件鉴权
- Next.js 管理后台 UI
- 部署上线（Vercel + Supabase）

> 做完这个项目 = 全栈核心技能全走通

---

## 📦 技术栈

| 层级 | 技术 |
|------|------|
| 前端框架 | Next.js 16 (App Router) |
| UI 组件 | shadcn/ui + Tailwind CSS |
| 后端 API | Next.js API Routes |
| 数据库 | PostgreSQL + Prisma 7 |
| 认证 | JWT + bcrypt |
| 部署 | Vercel + Supabase |

---

## 🗓️ 学习路线（按项目模块拆解）

### 阶段一：后端基础（Week 1-2）

**Day 01 ✅ 环境搭建**
- PostgreSQL 安装配置
- Prisma 7 初始化（prisma.config.ts）
- 注册 API（bcrypt 密码加密）

**Day 02 🔜 认证系统**
- 登录 API（bcrypt.compare 验证密码）
- JWT token 生成与验证
- Refresh Token 机制

**Day 03 🔜 RBAC 数据库设计**
- User / Role / Permission 表设计
- 多对多关系（用户-角色、角色-权限）
- Prisma migrate

**Day 04 🔜 鉴权中间件**
- JWT 验证中间件
- 角色权限校验
- API 路由保护

---

### 阶段二：核心 API（Week 3-4）

**Day 05 🔜 用户管理 API**
- 获取用户列表（分页）
- 创建/更新/删除用户
- 给用户分配角色

**Day 06 🔜 角色管理 API**
- 角色 CRUD
- 给角色分配权限

**Day 07 🔜 权限管理 API**
- 权限 CRUD
- 权限校验工具函数

**Day 08 🔜 API 规范化**
- 统一响应格式
- 错误处理
- Swagger 文档

---

### 阶段三：前端页面（Week 5-6）

**Day 09 🔜 UI 搭建**
- 安装 shadcn/ui
- 布局：侧边栏 + 顶栏
- 路由结构

**Day 10 🔜 登录页面**
- 登录表单
- JWT 存储（httpOnly Cookie）
- 登录后跳转

**Day 11 🔜 用户管理页面**
- 用户列表（表格 + 分页）
- 新增/编辑/删除弹窗

**Day 12 🔜 角色权限页面**
- 角色列表
- 权限分配界面（Checkbox Tree）

---

### 阶段四：部署上线（Week 7）

**Day 13 🔜 Supabase 配置**
- 迁移数据库到 Supabase（云 PostgreSQL）
- 环境变量配置

**Day 14 🔜 Vercel 部署**
- 推送到 GitHub
- Vercel 自动部署
- 域名配置

---

## 🗄️ 数据库 Schema 设计

```prisma
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  password  String
  name      String?
  avatar    String?
  status    Boolean  @default(true)
  roles     UserRole[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Role {
  id          String       @id @default(cuid())
  name        String       @unique
  description String?
  users       UserRole[]
  permissions RolePermission[]
  createdAt   DateTime     @default(now())
  updatedAt   DateTime     @updatedAt
}

model Permission {
  id          String           @id @default(cuid())
  name        String           @unique  // e.g. "user:create"
  description String?
  module      String           // e.g. "user", "role"
  action      String           // e.g. "create", "read", "update", "delete"
  roles       RolePermission[]
  createdAt   DateTime         @default(now())
}

// 用户-角色 多对多
model UserRole {
  userId    String
  roleId    String
  user      User   @relation(fields: [userId], references: [id])
  role      Role   @relation(fields: [roleId], references: [id])
  createdAt DateTime @default(now())
  @@id([userId, roleId])
}

// 角色-权限 多对多
model RolePermission {
  roleId       String
  permissionId String
  role         Role       @relation(fields: [roleId], references: [id])
  permission   Permission @relation(fields: [permissionId], references: [id])
  @@id([roleId, permissionId])
}
```

---

## 🛠️ 项目目录结构

```
fullstack-rbac/
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   │   └── login/page.tsx
│   │   ├── (dashboard)/
│   │   │   ├── users/page.tsx
│   │   │   ├── roles/page.tsx
│   │   │   └── permissions/page.tsx
│   │   └── api/
│   │       ├── auth/
│   │       │   ├── register/route.ts
│   │       │   ├── login/route.ts
│   │       │   └── logout/route.ts
│   │       ├── users/route.ts
│   │       ├── roles/route.ts
│   │       └── permissions/route.ts
│   ├── lib/
│   │   ├── prisma.ts        ← DB 客户端
│   │   ├── auth.ts          ← JWT 工具
│   │   └── permissions.ts   ← 权限校验工具
│   └── middleware.ts        ← 全局认证中间件
├── prisma/
│   └── schema.prisma
└── prisma.config.ts
```

---

## ⚡ AI Coding 时代的使用姿势

**用 Claude Code / Cursor 的正确方式：**
- ✅ 让 AI 生成样板代码（CRUD、Schema、类型定义）
- ✅ 让 AI 解释不熟悉的代码
- ✅ 让 AI 写测试用例
- ❌ 安全相关代码（JWT、密码、权限校验）要自己理解每一行

**你是架构师，AI 是高级程序员。**
你负责设计和决策，AI 负责执行和填充。
