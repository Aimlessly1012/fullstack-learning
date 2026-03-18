# 全栈开发学习规划（前端 + Node.js 基础）

## 🎯 当前优势分析

已掌握：
- 前端三件套（HTML/CSS/JS）
- 框架（React/Vue）
- Node.js 基础（Express/API 开发）

这意味着你已经完成了 **60% 的全栈路径**，剩下的核心是：
**数据库、部署、架构设计、安全**

---

## 📦 需要补充的技能模块

### P0 必会（1-2个月内掌握）

#### 1. 数据库
- **PostgreSQL** — 关系型数据库，SQL 基础（SELECT/JOIN/事务）
- **Prisma ORM** — Node 生态最好用的 ORM，类型安全
- **Redis** — 缓存 + Session 存储基础

#### 2. 认证 & 安全
- JWT（生成、验证、刷新 token）
- OAuth 2.0（GitHub/Google 登录）
- 密码哈希（bcrypt）
- 常见安全漏洞：SQL 注入、XSS、CSRF

#### 3. API 设计规范
- RESTful 设计原则
- 错误处理规范（统一响应格式）
- API 版本管理
- 文档（Swagger/OpenAPI）

#### 4. Docker 基础
- Dockerfile 编写
- docker-compose 多服务编排
- 容器化 Node 应用

---

### P1 重要（2-3个月内掌握）

#### 5. 云部署
- **Vercel** — 前端/Next.js 极速部署
- **Railway / Render** — 后端服务部署
- **Supabase** — PostgreSQL + Auth + Storage 一体化
- CI/CD 基础（GitHub Actions）

#### 6. 全栈框架
- **Next.js** — App Router、Server Actions、API Routes
- tRPC（类型安全 API，前后端无缝对接）

#### 7. 文件 & 存储
- S3 / Cloudflare R2（文件上传）
- 图片处理基础（Sharp）

#### 8. 消息队列基础
- Bull/BullMQ（Node.js 任务队列）
- 异步任务处理模式

---

### P2 加分（AI 时代核心竞争力）

#### 9. AI 集成能力
- OpenAI / Anthropic API 调用
- 流式输出（Stream Response）
- Function Calling / Tool Use
- Prompt Engineering 最佳实践

#### 10. RAG & 向量数据库
- Embedding 基础概念
- Supabase pgvector / Pinecone
- 文档问答应用构建

#### 11. WebSocket & 实时通信
- Socket.io 基础
- Server-Sent Events（SSE）
- 实时功能（聊天、通知）

#### 12. 监控 & 可观测性
- Sentry（错误追踪）
- Vercel Analytics / Posthog
- 日志规范（pino/winston）

---

## 🗓️ 3个月学习路线

### 第1个月：打地基

**Week 1-2：数据库**
- PostgreSQL 安装 & SQL 基础（增删改查、JOIN、索引）
- 用 Prisma 连接数据库，做 CRUD API
- 实践：用 Express + Prisma + PostgreSQL 做一个 Todo API

**Week 3：认证系统**
- 实现注册/登录（JWT + bcrypt）
- Refresh Token 机制
- 实践：给 Todo API 加完整认证

**Week 4：Docker**
- Dockerfile 编写 Node 应用
- docker-compose 同时启动 Node + PostgreSQL + Redis
- 实践：容器化你的 Todo 项目

---

### 第2个月：做完整项目

**Week 5-6：Next.js 全栈**
- App Router、Server Components、Server Actions
- Next.js + Prisma + PostgreSQL 全栈项目
- 实践：做一个带认证的内容管理系统（博客/笔记应用）

**Week 7：部署上线**
- Vercel 部署 Next.js
- Supabase 托管数据库
- 配置环境变量、域名
- 实践：把第6周的项目部署上线，发布到网上

**Week 8：AI 功能集成**
- 集成 OpenAI API
- 实现流式输出
- 实践：给博客应用加 AI 写作助手功能

---

### 第3个月：提升 & 专项

**Week 9-10：选一个方向深入**
- 方向A：SaaS 产品（支付 Stripe、订阅、多租户）
- 方向B：AI 应用（RAG、向量搜索、AI Agent）
- 方向C：实时应用（WebSocket、协作编辑）

**Week 11：代码质量**
- TypeScript 严格模式
- 单元测试（Vitest）
- API 测试（Supertest）

**Week 12：总结 & 作品集**
- 完善 GitHub README
- 写技术博客总结
- 准备作品集展示

---

## 🛠️ 推荐实战项目清单

| 项目 | 核心技能 | 难度 |
|------|---------|------|
| Todo App（带认证） | Prisma + JWT + PostgreSQL | ⭐⭐ |
| 博客 CMS | Next.js + 富文本 + 图片上传 | ⭐⭐⭐ |
| AI 聊天应用 | OpenAI API + 流式输出 + 对话历史 | ⭐⭐⭐ |
| URL 短链服务 | Redis + 统计 + Docker 部署 | ⭐⭐⭐ |
| SaaS 模板 | Stripe 支付 + 订阅 + 多租户 | ⭐⭐⭐⭐ |
| RAG 知识库 | 向量数据库 + PDF 解析 + AI 问答 | ⭐⭐⭐⭐ |

---

## 🤖 AI Coding 时代的使用姿势

### 用 Cursor / Copilot 的正确方式
- ✅ 让 AI 生成样板代码（CRUD、Schema、类型定义）
- ✅ 让 AI 解释不熟悉的代码
- ✅ 让 AI 写测试用例
- ✅ 让 AI 重构代码
- ❌ 不要盲目复制 AI 代码，每行都要看懂
- ❌ 安全相关代码要自己 review

### 核心竞争力总结

> **AI 能写代码，但不能替你做决策**

1. **架构设计能力** — 数据库如何设计，API 如何划分，AI 不会自动告诉你
2. **需求拆解能力** — 把业务需求翻译成技术方案
3. **调试 & 排错能力** — 出了问题能快速定位，AI 有时会瞎猜
4. **安全意识** — 知道哪些地方容易出安全问题
5. **工程化思维** — 代码可维护、可扩展、可测试

**最终目标：你是架构师，AI 是高级程序员，你负责思考，它负责执行。**
