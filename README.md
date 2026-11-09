# Veriflow — AWS Serverless 认证服务

一个"登录优先"的认证服务，用于一个小型需登录的网站。它在 AWS 上以无固定服务器账单的方式覆盖完整账号生命周期：邮箱+密码注册、邮箱验证、登录、可选邮箱码 MFA、密码重置、限流与短时限账号锁定。

> **状态：设计阶段。** 尚未编写应用代码。实现契约位于 `docs/` 文档集：[design.md](docs/design.md)（后端架构）、[PRD.md](docs/PRD.md)（产品需求与验收标准）、[DESIGN_CN.md](docs/DESIGN_CN.md)（前端设计）、[TASKS.md](docs/TASKS.md)（任务拆分）、[ACCEPTANCE.md](docs/ACCEPTANCE.md)（验收清单）。

## 项目目标

目标是做一个云/后端方向的个人作品集项目，展示真实的 AWS 设计取舍，而不是把一个 CRUD demo 搬上 EC2。项目刻意做小，但覆盖了认证里容易做错的部分：验证码、会话 Cookie、MFA、滥用控制、异步邮件投递。

## 计划能力

- 邮箱+密码注册；用 6 位邮箱验证码验证邮箱。
- 登录与退出；刷新页面保持登录。
- 可选邮箱码 MFA，默认关闭，可在设置页开启。
- 忘记密码 / 重置密码，复用同一套邮箱验证码机制。
- 登录、注册、发码、密码重置、MFA 的限流。
- 密码或 MFA 连续失败后的短时账号锁定。
- 一个最小受保护 `/app` 区域与一个公开的 `/resend-verification` 页。

## 架构

```text
Browser
   │  https://app.example.com
   ▼
CloudFront (ACM, HTTPS, 自定义域名)
   ├── /*      → S3 (React build, 私有桶 + OAC)
   └── /api/*  → API Gateway HTTP API
                    │
                    ▼
                 Lambda (Express via serverless-http)
                    ├── DynamoDB: Users / Sessions / VerificationCodes / RateLimits
                    ├── SQS: 发信任务
                    └── CloudWatch / X-Ray
                           │
                           ▼
                    Lambda (email worker) → SES → 用户邮箱
```

React 站点与 API 共用一个 CloudFront 分发，浏览器看到单一源站，会话 Cookie 保持简单。API 是经 `serverless-http` 打包到 Lambda 的 Express 应用；DynamoDB 存放用户、会话、验证码、限流计数。SES 投递通过 SQS 解耦，使缓慢或失败的发信不阻塞 HTTP 请求。

## 技术栈

| 层 | 选型 |
|---|---|
| 前端 | React + Vite + JavaScript、React Router、Tailwind CSS |
| API | Node.js 22、Express、`serverless-http` |
| 计算 | API Gateway HTTP API 后的 AWS Lambda |
| 数据 | DynamoDB，四张 On-Demand 表 |
| 邮件 | SES，由 SQS 队列与 worker Lambda 驱动 |
| 基础设施 | AWS CDK（TypeScript） |
| CI/CD | GitHub Actions 通过 OIDC 对接 AWS |
| 可观测 | CloudWatch 日志、指标、告警；X-Ray 追踪 |
| 区域 | 默认 `ap-southeast-1`；全球边缘证书用 `us-east-1` |

## 仓库结构

```text
.
├── README.md
├── docs/
│   ├── design.md          # 后端架构、流程、数据模型、API、安全、部署
│   ├── PRD.md             # 产品需求 + Given/When/Then 验收标准
│   ├── DESIGN_CN.md       # 前端设计：技术栈、组件拆分、数据流、API 契约、样式
│   ├── TASKS.md           # 43 个任务的六阶段执行计划
│   ├── ACCEPTANCE.md      # 由 PRD 派生的勾选式验收清单
│   └── adr/
│       └── 0001-serverless-authentication.md
├── .gitignore
└── (应用与基础设施代码将在设计批准后补充)
```

## 文档

- [PRD](docs/PRD.md) — 产品概述、功能清单、页面清单、用户数据模型、业务规则、异常场景、25 条可验证验收标准、范围外清单。
- [前端设计](docs/DESIGN_CN.md) — 技术栈（React Context + react-hook-form + zod + Tailwind CSS）、完整目录树、组件职责、数据流、完整 API 契约、开发配置约束。
- [后端设计](docs/design.md) — 目标、非目标、架构、DynamoDB 模型、API 契约、安全控制、CDK 部署、成本估算、风险。
- [任务计划](docs/TASKS.md) — 跨六个阶段的 43 个任务，每个映射到 PRD 功能点。
- [验收清单](docs/ACCEPTANCE.md) — 把 PRD 验收标准整理为可按模块勾选的检查表。
- [ADR 0001](docs/adr/0001-serverless-authentication.md) — 项目为何采用 Serverless 自建认证栈。

## 安全基线

设计在 MVP 中也保留一组不可妥协的控制：

- 密码用 bcrypt 哈希，cost 12。
- 验证码与 MFA 码以 HMAC-SHA256 摘要存储，绝不存明文。
- 短时效、单次使用的码，带尝试次数上限与重发冷却。
- 会话 Cookie 为 `HttpOnly`、`Secure`、`SameSite=Lax`，并带 `__Host-` 前缀；每次登录成功或 `session_version` 变更时重签发全新会话（会话固定防护）。
- 统一错误文案，不泄露账号是否存在。
- 限流与锁定在 DynamoDB 中强制，而非仅进程内存。
- 源码不含密钥；运行期密钥来自 Secrets Manager。

## 成本目标

设计面向个人项目：API Gateway、Lambda、DynamoDB On-Demand、S3、CloudFront、SES、SQS、CloudWatch 在低流量下预计合计约 **$1–3/月**，硬预算上限低于 **$5/月**。域名注册费用另计。

## 后续步骤

- 审批设计并制定实现计划。
- 构建认证 API 与 DynamoDB 访问层。
- 构建 React 账号流程。
- 补充 CDK 基础设施与 GitHub Actions 部署。
- 部署到开发用 AWS 账号并运行验收测试。

## 许可证

尚未确定。暂未授予许可证。
