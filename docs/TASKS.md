# TASKS — Veriflow 开发任务清单（Vite + React，AWS Serverless）

> 来源：`docs/PRD.md`、`docs/DESIGN_CN.md`、已锁定项目章程。
> 执行顺序：基础结构 → 后端数据层/服务 → 接口 → 页面 → 安全加固 → 基础设施/CI → E2E 验收。
> 类型：页面 / 组件 / 接口 / 逻辑。复杂度：S（≤0.5d） / M（1–2d） / L（跨多文件多服务）。
> "关联PRD" 指向 `docs/PRD.md` 的功能点（§2 功能清单、§3 页面、§5 规则、§7 验收标准 AC）。

## 阶段 0：基础结构

| # | 任务名 | 所属模块 | 类型 | 依赖 | 复杂度 | 关联 PRD | 验收标准 |
|---|---|---|---|---|---|---|---|
| T01 | 工作区脚手架（npm workspaces） | 基础结构 | 逻辑 | — | S | §1/§2 | 根 `package.json` 含 web/api/infra/packages 四子包；各子包可独立 install/运行；`.gitignore` 覆盖 node_modules/.env/dist；`.env.example` 含 `VITE_APP_BASE_URL` 等（Vite 仅注入 `VITE_` 前缀变量） |
| T02 | 共享 zod schema 包 | 共享校验 | 逻辑 | T01 | S | §5/§7 | `packages/schemas` 导出注册/登录/验证码/重置/MFA 的 zod schema；web 与 api 引用同一份；字段与 §4/§5 一致 |
| T03 | Vite + Tailwind 工程配置 | 基础结构 | 逻辑 | T01 | S | §6/DESIGN§6 | `vite.config.js`（`/api` 代理 `localhost:3001`）、`tailwind.config.js`（content 扫描、窄栏令牌）、`postcss.config.js`、`index.css` 含 `@tailwind`；`npm run build` 产出 `dist` |
| T04 | 前端请求封装 `lib/api.js` | 前端基础设施 | 逻辑 | T01 | M | §5/§7-AC9/16 | 封装 `fetch`：同源基址、解析 `{error:{code,message}}`、401 触发登出、读取 `Retry-After`；覆盖 401/429/423 解析的单元行为 |
| T05 | 常量 `lib/constants.js` | 前端基础设施 | 逻辑 | T01 | S | §3 | 路由路径与错误码常量集中定义；页面引用而非硬编码 |
| T06 | `AuthProvider` + `useAuth` | 前端基础设施 | 组件 | T04 | M | §3/§7-AC9/10 | 启动 `GET /api/auth/me` 设 `user/loading`；`logout()` 调接口清 user；任意 401 清 user 跳 `/login` 带 `from`；hook 暴露 `user/refresh/logout` |
| T07 | `RequireAuth` 路由守卫 | 前端基础设施 | 组件 | T06 | S | §3（/app 守卫） | `loading` 显示骨架；未登录 `Navigate /login state:{from}`；回跳仅站内路径 |
| T08 | 路由与入口（router/App/main） | 前端基础设施 | 组件 | T06,T07 | S | §3 | `router.jsx` 声明公开/受保护路由；`App.jsx` 组合；`main.jsx` 包裹 `AuthProvider` |
| T09 | 基础 UI 组件集 | 基础组件 | 组件 | T03 | M | §3/§6（可访问性） | `AuthCard/TextField/PasswordField/Button/Alert/Spinner`；label 绑定、`role="alert"`、focus 可见、autocomplete 正确；Tailwind 原子类 |
| T10 | NotFound 页 | 页面 | 页面 | T08,T09 | S | §3（*） | `*` 路由渲染 404；未匹配路由跳转 |

## 阶段 1：后端数据层与服务

| # | 任务名 | 所属模块 | 类型 | 依赖 | 复杂度 | 关联 PRD | 验收标准 |
|---|---|---|---|---|---|---|---|
| T11 | DynamoDB 客户端与表访问 | 后端数据层 | 逻辑 | T01 | M | §4/§5（DynamoDB） | `client.js` + users/sessions/codes/rateLimits.js；仅用 GetItem/Query/PutItem/UpdateItem/DeleteItem/TransactWriteItems；无 `Scan`、无 GSI；`ConsistentRead`；**所有 email 入参先经 `normalizeEmail()`（trim+lowercase）再作主键/限流键，防身份分裂（PRD §6）** |
| T12 | 本地建表脚本 | 后端数据层 | 逻辑 | T11 | S | §1（本地） | `db:local` 在 DynamoDB Local 建 4 表 + TTL 属性 |
| T13 | 密码哈希服务 | 后端服务 | 逻辑 | T11 | S | §5（bcrypt cost12）/§7-AC5 | bcryptjs cost12 异步 hash/compare；min8/max72bytes；邮箱不存在做假哈希比较 |
| T14 | 验证码服务 | 后端服务 | 逻辑 | T11 | M | §5（码规则）/§7-AC3/13 | 6 位随机、HMAC-SHA256 存储、10 分钟、5 次尝试、60 秒冷却、单次消费、条件写防并发消费 |
| T15 | 限流服务 | 后端服务 | 逻辑 | T11 | M | §5（限流矩阵）/§7-AC14 | DynamoDB 原子 `ADD` 计数；各动作键/限额/窗口；邮箱+IP 双键；超限返 429 + `Retry-After` |
| T16 | 锁定服务 | 后端服务 | 逻辑 | T11 | S | §5（锁定）/§7-AC6 | 密码 5 / MFA 10 失败写 `locked_until`；登录成功与重置清零；423 不区分密码/MFA |
| T17 | `mfaToken` 服务 | 后端服务 | 逻辑 | T11 | S | §5（MFA）/§7-AC7/8 | HMAC 签名/验签；10 分钟；校验 `purpose` 与 `session_version`；`timingSafeEqual` |
| T18 | 邮件服务 + SQS 适配器 | 后端服务 | 逻辑 | T11 | M | §2（发信）/§5 | `emailService` 入队 SQS；本地 `console` adapter；SES SDK 仅 worker 调用；纯文本 |

## 阶段 2：接口

| # | 任务名 | 所属模块 | 类型 | 依赖 | 复杂度 | 关联 PRD | 验收标准 |
|---|---|---|---|---|---|---|---|
| T19 | Express app + server + lambda 骨架 | 接口 | 逻辑 | T11–T18 | M | §1 | `app.js` 含 `express.json({limit:'20kb'})`、同源 `Origin` 校验（**开发环境须将浏览器 origin 纳入白名单，避免 `:5173`→`:3001` 写接口被 403 自锁，PRD §6**）；`server.js` 本地监听；`lambda.js` 经 `serverless-http` 导出 handler |
| T20 | 统一错误中间件 | 接口 | 逻辑 | T19 | S | §5（统一文案） | `errors.js` 输出 `{error:{code,message}}`；不泄露账号是否存在；不返回堆栈/表名/内部 ID |
| T21 | 认证接口（注册/验证/重发/登录/登出/me/忘记/重置） | 接口 | 接口 | T13–T16,T18,T20 | L | §2/§3/§7-AC1..11/AC24/AC25 | `routes/auth.js` 实现全部端点；与 DESIGN§5.1 契约一致；返回码对齐 §7（201/409/403/401/423/429/503 等）；**登录成功与 `session_version` 变更须重签发全新 `token_hash` 会话并作废旧 token（AC24）；已验证账号 `/resend-verification` 仍 200 但不发码/不改状态/不泄露验证态（AC25）** |
| T22 | 健康检查 | 接口 | 接口 | T19 | S | §7 | `GET /api/health` 返回 `200 {status:"ok"}` |
| T23 | MFA 设置 + mfa/verify/resend 接口 | 接口 | 接口 | T14,T16,T17,T20 | M | §2（MFA P1）/§7-AC7/8 | `routes/mfa.js`；需会话 Cookie；开启=当前密码+码、关闭=密码+码；`mfa/verify` 与 `mfa/resend` 用 `mfaToken` |
| T24 | 会话鉴权中间件 | 接口 | 逻辑 | T11 | S | §5（会话） | 读 `__Host-app_session` Cookie → SHA-256 查 Sessions；`session_version` 一致；过期/不一致删并 401 |

## 阶段 3：页面

| # | 任务名 | 所属模块 | 类型 | 依赖 | 复杂度 | 关联 PRD | 验收标准 |
|---|---|---|---|---|---|---|---|
| T25 | 注册页 | 页面 | 页面 | T08,T09,T02,T21 | M | §3 注册/§7-AC1/2 | react-hook-form+zod；提交后跳 `/verify-email?email=`；409 提示"邮箱已注册" |
| T26 | 邮箱验证页 | 页面 | 页面 | T08,T09,T02,T21 | M | §3 验证/§7-AC3 | 6 位码表单 + 重发；400 统一提示且不显剩余次数 |
| T27 | 重发验证码页 | 页面 | 页面 | T08,T09,T21 | S | §3 重发/§2 P0 | 邮箱提交重发；统一提示"如果存在，已发送" |
| T28 | 登录页 | 页面 | 页面 | T08,T09,T02,T21,T24 | M | §3 登录/§7-AC4/5/6/7 | 邮箱+密码；`mfaRequired` 跳 `/mfa` 带 `state.mfaToken`；401/403/423/429 文案与 `Retry-After` 倒计时 |
| T29 | MFA 验证页 | 页面 | 页面 | T08,T09,T02,T23,T24 | M | §3 MFA/§7-AC8 | 取 `location.state.mfaToken`；刷新丢失即回 `/login`；成功后 `refresh()` 进 `/app` |
| T30 | 忘记密码页 | 页面 | 页面 | T08,T09,T02,T21 | S | §3 忘记/§2 P0 | 邮箱提交；统一提示 |
| T31 | 重置密码页 | 页面 | 页面 | T08,T09,T02,T21 | M | §3 重置/§7-AC11 | 邮箱+码+新密码；成功后提示去登录 |
| T32 | 应用主页 | 页面 | 页面 | T08,T09,T06 | S | §3 /app | 登录成功占位；`RequireAuth` 保护 |
| T33 | 设置页（MFA 开关） | 页面 | 页面 | T08,T09,T02,T23,T06 | M | §3 设置/§2 MFA P1 | 开关流程：当前密码→收码→确认；开启/关闭均走 §5 规则 |

## 阶段 4：安全加固与细节

| # | 任务名 | 所属模块 | 类型 | 依赖 | 复杂度 | 关联 PRD | 验收标准 |
|---|---|---|---|---|---|---|---|
| T34 | Cookie `__Host-` 前缀 | 安全加固 | 逻辑 | T21,T24 | S | §5/§7-AC12 | Set-Cookie 名为 `__Host-app_session`；生产带 `Secure`；仅 HTTPS |
| T35 | CloudFront 安全响应头策略 | 安全加固 | 逻辑 | T40 | M | §7-AC16 | `Response Headers Policy` 加 CSP/HSTS/`X-Content-Type-Options: nosniff`；attach WebStack |
| T36 | 请求体 20KB 限制 | 安全加固 | 逻辑 | T19 | S | §7-AC17 | `express.json({limit:'20kb'})`；超限拒收不进业务 |
| T37 | 日志脱敏 | 安全加固 | 逻辑 | T19,T21 | S | §5/§7-AC18 | 结构化日志不打密码/码/Cookie/完整 mfaToken/Authorization；IP 哈希、UA 截断 |

## 阶段 5：基础设施与交付

| # | 任务名 | 所属模块 | 类型 | 依赖 | 复杂度 | 关联 PRD | 验收标准 |
|---|---|---|---|---|---|---|---|
| T38 | DataStack（CDK） | 基础设施 | 逻辑 | T11 | M | §2/§4 | 4 张 DynamoDB 表 On-Demand + TTL；Users PITR；删除保护（dev） |
| T39 | ApiStack（CDK） | 基础设施 | 逻辑 | T21,T23,T38,T18 | L | §2/§5 | HTTP API + Lambda + SQS+DLQ + email worker + Secrets + IAM（最小权限、无 Scan）+ 日志/X-Ray + 告警 |
| T40 | WebStack（CDK） | 基础设施 | 逻辑 | T03 | M | §2/§7-AC15 | S3 私有+OAC + CloudFront + ACM(us-east-1) + 同源 `/api` 转发 + SPA fallback |
| T41 | ObservabilityStack（CDK） | 基础设施 | 逻辑 | T39,T40 | M | §2（告警）/§6 | CloudWatch alarms（Lambda 错误/5xx/DLQ/DynamoDB 限流/SES 退信）+ SNS + Budget $5 |
| T42 | GitHub Actions（OIDC） | CI/CD | 逻辑 | T38–T41 | M | §2 P1/§7 | `ci.yml`（lint/test/build/cdk synth）+ `deploy.yml`（OIDC assume role、`cdk deploy`、s3 sync + 失效） |
| T43 | 端到端验收脚本 | 测试 | 逻辑 | T25–T33,T39 | L | §7 全部 AC | Playwright 覆盖注册→验证→登录→MFA→重置→401 跳转；断言 AC1..25 可逐项通过；含会话固定（AC24）、已验证重发（AC25）、字节长度边界（AC19）、mfaToken 单次使用（AC20）、重置不锁（AC23） |

---

### 依赖关系速览（关键路径）
- 基础：`T01 → T02/T03/T04/T05 → T06 → T07 → T08 → T09/T10`
- 后端：`T01 → T11 → T12–T18 → T19 → T20/T21/T22/T23/T24`
- 页面：`T08+T09 + (T21 或 T23) → T25–T33`
- 加固：`T19 → T36/T37`；`T21/T24 → T34`；`T40 → T35`
- 交付：`T11/T21/T23/T18 → T38/T39 → T40 → T41 → T42`；`T25–T33 + T39 → T43`

### 范围外（不在本清单，见 PRD §8）
第三方登录 / Passkey / TOTP / 短信 / 扫码 / 企业 SSO / 多区域 / WAF / 图文验证码 / 设备指纹 / 风控 / 管理后台 / 邮件模板 / MFA 恢复码 / 博客内容 / 滑动续期 / 公开 Landing。
