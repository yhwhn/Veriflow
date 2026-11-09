# Veriflow — AWS 认证服务

Veriflow 是一个"登录优先"的认证服务，用于一个小型需登录的网站。它覆盖完整账号生命周期：邮箱+密码注册、邮箱验证、登录、可选邮箱码 MFA、密码重置、限流与短时限账号锁定。认证逻辑由项目自身实现，不依赖托管身份产品。

本仓库收录 Veriflow 的设计文档集：[PRD.md](docs/PRD.md)（产品需求与验收标准）、[design.md](docs/design.md)（系统架构、数据模型、接口契约、部署）、[TASKS.md](docs/TASKS.md)（任务拆分）、[ACCEPTANCE.md](docs/ACCEPTANCE.md)（验收清单）、[CHANGELOG.md](CHANGELOG.md)（变更记录）。

## 项目目标

做一个云/后端方向的个人作品集项目，展示真实的 AWS 架构与自建认证实现。项目刻意做小，但覆盖认证里容易做错的部分：验证码、会话 Cookie、MFA、滥用控制、异步邮件投递。

## 能力

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
   │  https://<cloudfront-domain>
   ▼
CloudFront（唯一入口；Viewer → CloudFront 为 HTTPS）
   ├── /*      → yhwhn.github.io（GitHub Pages，HTTPS Only 源站，静态站点）
   └── /api/*  → Elastic Beanstalk（单实例，HTTP Only 源站，CachingDisabled）
                     │
                     ▼
                  Express（Node.js 22，常驻进程）
                     ├── RDS PostgreSQL（users / sessions / verification_codes / used_mfa_tokens）
                     ├── 限流计数（单实例内存）
                     └── SQS：发信任务
                            │
                            ▼
                     EB 常驻 worker 进程 → SES → 用户邮箱

CloudFront Response Headers Policy：CSP / HSTS / X-Content-Type-Options
CloudFront Custom Error Pages：403/404 → /index.html（SPA fallback）
```

浏览器只访问 CloudFront 默认域名。前端是 GitHub Pages 上的静态站点，API 是 Elastic Beanstalk 上常驻的 Express 应用；两者经同一 CloudFront 分发按路径分流，浏览器看到单一源站，会话 Cookie 保持同源。GitHub Pages 侧不配置自定义域名，自定义域名（若使用）配在 CloudFront 的 Alternate Domain Name。

## 技术栈

| 层 | 选型 |
|---|---|
| 前端 | React + Vite + JavaScript、React Router、Tailwind CSS |
| 前端托管 | GitHub Pages（用户页 `yhwhn.github.io`） |
| 入口 / CDN | CloudFront（多源站、按路径分流、Response Headers Policy、Custom Error Pages） |
| API | Node.js 22、Express（原生，不经 serverless 适配层） |
| 计算 | AWS Elastic Beanstalk 单实例，托管 Node.js 平台 |
| 数据 | RDS PostgreSQL 单 AZ，原生 `pg` + 参数化 SQL |
| 密码哈希 | argon2id |
| 限流 | 单实例内存计数 |
| 邮件 | SES，经 SQS 与 EB 常驻 worker 进程异步投递 |
| 密钥 | EB 环境属性 / SSM Parameter Store，进程启动时加载 |
| 基础设施 | AWS CDK（TypeScript） |
| CI/CD | GitHub Actions（前端发布到 GitHub Pages）+ OIDC 对接 AWS |
| 可观测 | CloudWatch 日志、指标、告警 |
| 区域 | 默认 `ap-southeast-1` |

## 仓库结构

```text
.
├── README.md
├── CHANGELOG.md
├── agent.md
├── docs/
│   ├── PRD.md          # 产品需求 + Given/When/Then 验收标准
│   ├── design.md       # 系统架构、数据模型、接口契约、安全、部署、成本（英文）
│   ├── TASKS.md        # 43 个任务的六阶段执行计划（英文）
│   └── ACCEPTANCE.md   # 由 PRD 派生的勾选式验收清单
├── .gitignore
└── .gitattributes
```

## 文档

- [PRD](docs/PRD.md) — 产品概述、功能清单、页面清单、用户数据模型、业务规则、异常场景、25 条可验证验收标准、范围外清单。
- [系统设计](docs/design.md) — 目标与非目标、架构、PostgreSQL 数据模型、完整 API 契约、安全控制、CDK 部署、成本估算、风险（英文）。
- [任务计划](docs/TASKS.md) — 跨六个阶段的 43 个任务，每个映射到 PRD 功能点（英文）。
- [验收清单](docs/ACCEPTANCE.md) — 把 PRD 验收标准整理为可按模块勾选的检查表。
- [变更记录](CHANGELOG.md) — 架构与文档的决策、历史记录（英文）。
- [内容纪律](agent.md) — 代码 / UI / 文档的内容边界。

## 安全基线

设计在 MVP 中也保留一组不可妥协的控制：

- 密码用 argon2id 哈希。
- 验证码与 MFA 码以 HMAC-SHA256 摘要存储，绝不存明文。
- 短时效、单次使用的码，带尝试次数上限与重发冷却。
- 会话 Cookie 为 `HttpOnly`、`Secure`、`SameSite=Lax`，并带 `__Host-` 前缀；每次登录成功或 `session_version` 变更时重签发全新会话（会话固定防护）。
- 统一错误文案，不泄露账号是否存在。
- 限流计数由 API 单实例内存维护，锁定状态落在数据库；数据库访问全部使用参数化 SQL。
- 源码不含密钥；运行期密钥来自 EB 环境属性 / SSM Parameter Store，进程启动时加载。

## 成本

系统按小时固定计费：Elastic Beanstalk 单实例（EC2）、RDS PostgreSQL 单 AZ 实例与存储构成主要支出，CloudFront、SQS、SES、CloudWatch 在低流量下接近免费套餐。低流量下预计合计约 **$20–30/月**（不含域名，不含首年免费套餐抵扣），不设硬上限，配有成本告警用于可见性。详细估算见 [design.md](docs/design.md) §15。

## 本地开发

```bash
# 1. 启动 PostgreSQL
docker run -d --name veriflow-postgres -e POSTGRES_PASSWORD=dev -e POSTGRES_DB=veriflow -p 5432:5432 postgres:16

# 2. 安装依赖
npm install

# 3. 初始化 schema
npm run db:local

# 4. 启动 web + api
npm run dev
```

- `web`：Vite 在 `http://localhost:5173`，把 `/api` 代理到 `http://localhost:3001`。
- `api`：Express 在 `http://localhost:3001`，连接本地 PostgreSQL。
- `EMAIL_DRIVER=console`：验证码打印在 API 终端，不联系 SES 与 SQS。

## 部署

```bash
npm ci
npm run build
cd infra
npx cdk bootstrap
npx cdk deploy --all -c env=dev
```

CDK 创建 RDS PostgreSQL、Elastic Beanstalk 应用与环境、SQS 队列与 DLQ、CloudFront 分发、IAM 与 CloudWatch 告警。CloudFront 默认行为指向 GitHub Pages 源站，`/api/*` 行为指向 Elastic Beanstalk。前端构建产物由 GitHub Actions 发布到 GitHub Pages；GitHub Actions 通过 OIDC 获取临时 AWS 凭据，不使用长期密钥。浏览器入口使用 CloudFront 默认域名，版本一不配置自定义域名、ACM 证书或 Route 53。

## 许可证

尚未确定。暂未授予许可证。
