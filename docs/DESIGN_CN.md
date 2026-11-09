# DESIGN — Veriflow 前端设计（Vite + React，AWS Serverless）

本文档基于 `docs/PRD.md` 与已锁定的项目章程，描述前端的设计方案。后端、DynamoDB 数据模型、CDK 基础设施、SES/SQS 异步发信在源 `docs/design.md`（英文）中定义，本文只覆盖前端实现与前后端契约。

## 1. 技术栈

| 层 | 选型 | 说明 |
|---|---|---|
| 构建 | Vite | 开发服务器 + HMR + 生产打包；本地把 `/api` 代理到 `localhost:3001` |
| 视图 | React 18（JavaScript，不上 TypeScript） | 浏览器内 UI 运行时 |
| 路由 | React Router | 客户端路由 + `RequireAuth` 守卫 |
| 表单 | react-hook-form | 表单状态与受控；非 UI 组件库 |
| 校验 | zod | 运行时 schema；与后端共享同一份规则 |
| 样式 | Tailwind CSS | 原子化 CSS 框架，替代手写 styles.css |
| 状态管理 | React Context（`AuthProvider`） | 仅全局认证态（user / loading / 401 跳转）；表单局部态交给 react-hook-form |
| 请求 | 原生 `fetch` 封装于 `lib/api.js` | 统一错误解析、401 拦截 |

不引入：UI 组件库、Redux/Zustand、TypeScript（前端）、CSS 预处理器。

### 1.1 状态管理方案说明
全局只需要"是否已登录、当前用户、加载中、任意 401 回登录"这几项，且全部围绕一次 `GET /api/auth/me` 与一次登录结果，因此一个 Context（`AuthProvider`）足够，无需引入第三方状态库。表单内的字段态、校验、错误展示由 react-hook-form 在页面内局部管理，不进全局 Context。

### 1.2 zod 共享 schema
同一份 zod 规则同时服务前端表单与后端接口校验，避免两端规则漂移。前端与后端分属不同 workspace 包，因此抽出独立包 `packages/schemas`，由 `web` 与 `api` 共同依赖（见目录结构）。

## 2. 目录结构（完整文件夹树）

```text
login-app/
  package.json                 # npm workspaces 根：web / api / infra / packages/schemas
  README.md
  .env.example                 # VITE_APP_BASE_URL 等（Vite 仅注入 VITE_ 前缀变量）
  .gitignore
  packages/
    schemas/
      package.json
      src/index.js             # 导出的 zod schema（注册/登录/验证码/重置/MFA 等）
  web/
    index.html
    vite.config.js             # /api 代理到 localhost:3001；build 输出 dist
    tailwind.config.js
    postcss.config.js
    src/
      main.jsx                 # 挂载 <AuthProvider><App/></AuthProvider>
      App.jsx                  # 组合 Router
      router.jsx               # 路由表（公开/受保护）
      index.css                # @tailwind 指令入口（替代 styles.css）
      lib/
        api.js                 # fetch 封装：统一错误、401 拦截
        constants.js           # 路由路径常量、错误码常量
      auth/
        AuthProvider.jsx       # Context：user/loading/refresh/logout
        useAuth.js             # 消费 Context 的 hook
        RequireAuth.jsx        # 路由守卫组件
      components/
        AuthCard.jsx           # 居中卡片布局容器
        TextField.jsx          # 带 label/错误绑定的文本输入
        PasswordField.jsx      # 密码输入 + 显示/隐藏
        Button.jsx             # 提交按钮（loading/disabled）
        Alert.jsx              # role="alert" 错误/提示条
        Spinner.jsx            # 加载态
      pages/
        Login.jsx
        Register.jsx
        VerifyEmail.jsx
        ResendVerification.jsx
        ForgotPassword.jsx
        ResetPassword.jsx
        MfaVerify.jsx
        AppHome.jsx
        Settings.jsx
        NotFound.jsx
  api/
    package.json
    src/
      app.js                   # Express app（含 express.json({limit:'20kb'})）
      server.js                # 本地 listen（:3001）
      lambda.js                # serverless-http handler 导出
      config.js                # 读 env / Secrets
      validation/
        schemas.js             # re-export packages/schemas 或引用
      db/
        client.js              # DynamoDB DocumentClient
        users.js
        sessions.js
        codes.js
        rateLimits.js
      services/
        authService.js
        codeService.js
        emailService.js
        rateLimitService.js
        lockoutService.js
        tokenService.js
      middleware/
        auth.js
        rateLimit.js
        errors.js
      routes/
        auth.js
        mfa.js
        health.js
      emails/
        templates.js
    test/
  infra/
    bin/app.ts
    lib/
      data-stack.ts
      api-stack.ts
      web-stack.ts
      observability-stack.ts
    cdk.json
    package.json
  tests/
    e2e/
    load/
```

## 3. 组件拆分方案

| 组件 / 文件 | 职责 |
|---|---|
| `main.jsx` | 入口：渲染 `<AuthProvider><App/></AuthProvider>` |
| `App.jsx` | 组合 `router.jsx` 的路由 |
| `router.jsx` | 声明路由表；公开路由（`/login` 等）与受保护路由（`/app`、`/app/settings`） |
| `auth/AuthProvider.jsx` | 持有 `{ user, loading }`；挂载时 `GET /api/auth/me`；暴露 `refresh()`、`logout()`；任意 API 401 触发登出并跳 `/login`（带 `from`） |
| `auth/useAuth.js` | 读取 AuthContext 的 hook，页面通过它取 user、调用 logout |
| `auth/RequireAuth.jsx` | 守卫：`loading` 显示骨架；未登录 `<Navigate to="/login" state={{from}}>`；登录后回跳 `from`（仅站内路径） |
| `lib/api.js` | 封装 `fetch`：注入同源 `/api` 基址、解析 `{error:{code,message}}`、捕获 401 交给 AuthProvider、解析 `Retry-After` |
| `lib/constants.js` | 路由路径、错误码字符串常量，避免散落硬编码 |
| `components/AuthCard.jsx` | 窄栏居中卡片容器（最大宽度、内边距、背景） |
| `components/TextField.jsx` | 通用文本输入：`label` 绑定、`aria-invalid`、`error` 文案、`autocomplete` |
| `components/PasswordField.jsx` | 密码输入 + 显示/隐藏切换、强度无关 |
| `components/Button.jsx` | 提交按钮：`loading` 时 disabled + Spinner |
| `components/Alert.jsx` | 错误/提示条：`role="alert"`，统一视觉 |
| `components/Spinner.jsx` | 加载指示 |
| `pages/Login.jsx` | 邮箱+密码表单（react-hook-form + zod）；提交 `POST /api/auth/login`；`mfaRequired` 时 `navigate('/mfa',{state:{mfaToken}})` |
| `pages/Register.jsx` | 邮箱+密码+确认；提交注册；跳 `/verify-email?email=` |
| `pages/VerifyEmail.jsx` | 6 位码输入；提交验证；含重发 |
| `pages/ResendVerification.jsx` | 邮箱输入重发；统一提示 |
| `pages/ForgotPassword.jsx` | 邮箱输入；统一提示 |
| `pages/ResetPassword.jsx` | 邮箱+码+新密码；提交重置 |
| `pages/MfaVerify.jsx` | 6 位码；提交 `POST /api/auth/mfa/verify`，从 `location.state.mfaToken` 取 token；刷新丢失则回 `/login` |
| `pages/AppHome.jsx` | 登录成功占位页 |
| `pages/Settings.jsx` | MFA 开关；开启/关闭流程（当前密码 + 邮箱码） |
| `pages/NotFound.jsx` | 404 |

## 4. 数据流说明

### 4.1 全局认证态（`AuthProvider`）
- 应用启动：`loading=true`，调用 `GET /api/auth/me`。
  - `200 { user }` → 设 `user`，`loading=false`。
  - `401` → `user=null`，`loading=false`（未登录）。
- `login(email, password)`：提交登录；返回 `{ user }` 则更新 `user`；返回 `{ mfaRequired, mfaToken }` 则不更新 `user`，由页面持有 `mfaToken` 跳 `/mfa`。
- `logout()`：调用 `POST /api/auth/logout` → 清 `user`。
- 任意通过 `lib/api.js` 的请求收到 `401`：`AuthProvider` 清 `user` 并跳 `/login`，携带当前路径作 `from`。
- 登录成功后的会话重签发是**后端行为**（PRD §5/AC24）：后端在登录成功与 `session_version` 变更时签发全新 `token_hash` 并作废旧 token；前端只需在响应后 `refresh()` 取 `user`，不沿用登录前的任何 cookie。

### 4.2 `mfaToken` 流转（关键）
- `mfaToken` 在登录接口返回后，**只存于 React 路由 `location.state` 与页面内存**，不进 Context、不进 localStorage/sessionStorage。
- `/mfa` 页面依赖 `location.state.mfaToken`；用户刷新 `/mfa` 会丢失该 state → 页面检测不到 `mfaToken` 即回 `/login`。
- MFA 验证成功后，后端设会话 Cookie，前端 `refresh()` 取 `user`，进入 `/app`。

### 4.3 表单态
- 各页面的字段值、校验错误、提交中态由 react-hook-form 在页面内管理；校验规则来自 `packages/schemas` 的 zod schema（通过 `@hookform/resolvers/zod` 接入）。
- 前端校验只做格式/长度/一致性；服务端做权威校验，前端错误以服务端返回码为准展示。

### 4.4 错误展示
- 服务端返回 `{ error: { code, message } }`；前端 `Alert` 组件显示 `message`（统一文案，不暴露账号是否存在）。
- `429`/`423` 读取 `Retry-After` 秒数，按钮倒计时禁用。

## 5. 接口约定

基址：`/api`。所有请求 `Content-Type: application/json`。写接口服务端校验 `Origin` 同源。成功 `200/201/202/204`；错误体统一为 `{ "error": { "code": "<CODE>", "message": "<文案>" } }`；`429`/`423` 带 `Retry-After` 响应头。

### 5.1 认证

**POST /api/auth/register**
- 入参：`{ "email": string, "password": string }`
- 成功 `201`：`{ "message": "验证码已发送" }`
- 错误：`400 VALIDATION_ERROR`、`409 EMAIL_ALREADY_REGISTERED`、`429 RATE_LIMITED`、`503 EMAIL_QUEUE_UNAVAILABLE`

**POST /api/auth/verify-email**
- 入参：`{ "email": string, "code": string(6位) }`
- 成功 `200`：`{ "message": "邮箱已验证" }`
- 错误：`400 INVALID_CODE`（不返回剩余次数）、`429 RATE_LIMITED`

**POST /api/auth/resend-verification**
- 入参：`{ "email": string }`
- 成功 `200`：`{ "message": "如果存在，已发送验证码" }`（不暴露邮箱是否存在）
- 错误：`400 VALIDATION_ERROR`、`429 CODE_COOLDOWN` / `RATE_LIMITED`

**POST /api/auth/login**
- 入参：`{ "email": string, "password": string }`
- 成功 `200`（无 MFA）：`{ "user": { "id": string, "email": string, "mfaEnabled": boolean } }`，Set-Cookie `__Host-app_session`
- 成功 `200`（有 MFA）：`{ "mfaRequired": true, "mfaToken": string }`（不设会话 Cookie）
- 错误：`401 INVALID_CREDENTIALS`、`403 EMAIL_NOT_VERIFIED`、`423 ACCOUNT_LOCKED`、`429 RATE_LIMITED`

**POST /api/auth/mfa/verify**
- 入参：`{ "mfaToken": string, "code": string(6位) }`
- 成功 `200`：`{ "user": { "id": string, "email": string, "mfaEnabled": boolean } }`，Set-Cookie `__Host-app_session`
- 错误：`401 INVALID_CODE`、`423 ACCOUNT_LOCKED`、`429 RATE_LIMITED`

**POST /api/auth/mfa/resend**
- 入参：`{ "mfaToken": string }`
- 成功 `200`：`{ "message": "已发送" }`（不返回新码）
- 错误：`401`、`429 RATE_LIMITED`

**POST /api/auth/logout**
- 入参：无（依赖会话 Cookie）
- 成功 `204`：清除 Cookie
- 错误：`401`

**GET /api/auth/me**
- 入参：无（依赖会话 Cookie）
- 成功 `200`：`{ "user": { "id": string, "email": string, "mfaEnabled": boolean } }`
- 错误：`401`

**POST /api/auth/forgot-password**
- 入参：`{ "email": string }`
- 成功 `200`：`{ "message": "如果存在，已发送重置邮件" }`（无论邮箱是否存在）
- 错误：`429 RATE_LIMITED`

**POST /api/auth/reset-password**
- 入参：`{ "email": string, "code": string(6位), "newPassword": string }`
- 成功 `200`：`{ "message": "密码已重置" }`（旧会话失效）
- 错误：`400 INVALID_CODE` / `VALIDATION_ERROR`、`429 RATE_LIMITED`

### 5.2 MFA 设置（需会话 Cookie）

**POST /api/auth/mfa/enable**
- 入参：`{ "password": string }`
- 成功 `202`：`{ "message": "验证码已发送" }`
- 错误：`401`、`409 MFA_ALREADY_ENABLED`、`429 RATE_LIMITED`

**POST /api/auth/mfa/enable/confirm**
- 入参：`{ "code": string(6位) }`
- 成功 `200`：`{ "message": "MFA 已开启" }`
- 错误：`400 INVALID_CODE`、`401`、`429 RATE_LIMITED`

**POST /api/auth/mfa/disable**
- 入参：`{ "password": string }`
- 成功 `202`：`{ "message": "验证码已发送" }`
- 错误：`401`、`409 MFA_NOT_ENABLED`、`429 RATE_LIMITED`

**POST /api/auth/mfa/disable/confirm**
- 入参：`{ "password": string, "code": string(6位) }`
- 成功 `200`：`{ "message": "MFA 已关闭" }`
- 错误：`400 INVALID_CODE`、`401`、`429 RATE_LIMITED`

### 5.3 健康检查

**GET /api/health**
- 成功 `200`：`{ "status": "ok" }`

### 5.4 `user` 对象字段
`{ "id": string, "email": string, "mfaEnabled": boolean }`（`id` = `user_id`；`mfaEnabled` = `mfa_enabled`）

## 6. 样式方案（Tailwind CSS）

- 使用 Tailwind CSS 替代手写 `styles.css`；入口 `web/src/index.css` 含 `@tailwind base/components/utilities`。
- `tailwind.config.js` 配置 `content` 扫描 `web/index.html` 与 `web/src/**/*.{js,jsx}`，定义设计令牌（颜色、间距、最大宽度 640px 窄栏）。
- 视觉基调：浅色、近黑文字、浅灰背景、单一低饱和强调色；不做渐变/玻璃拟态。响应式：移动端单列、按钮全宽。
- 可访问性：所有输入绑定 `label`；错误用 `role="alert"`；焦点可见；按钮 `loading` 时 `disabled`；正确设置 `autocomplete`（邮箱 `email`、密码 `current-password`/`new-password`）。
- 安全响应头（CloudFront Response Headers Policy 提供，非前端代码）：`Content-Security-Policy`、`Strict-Transport-Security`、`X-Content-Type-Options: nosniff`。

---

## 7. 开发配置约束（已知风险与实现约定）

以下为 PRD §6 标注的"保留"已知风险，落到实现层的硬性约定，避免开发期踩坑：

- **邮箱归一化收口**：所有入口（register / login / forgot / reset / resend / verify）在进入任何 DynamoDB 主键或限流键之前，必须统一经 `normalizeEmail()`（trim + lowercase）处理，且仅按字节/字符一致比较；不能在任一入口单独遗漏，否则同一邮箱会被当成不同账号（身份分裂）。建议在请求最前端（middleware 或 service 入口）统一收口。
- **开发期 `Origin` 同源校验白名单**：生产环境对所有写接口校验 `Origin` 与同源；但本地开发是前端 `:5173`、API `:3001` 双端口，浏览器 `Origin` 为 `5173`。`Origin` 中间件必须识别开发环境（由 `APP_ENV`/CDK context 注入），将浏览器 origin（`VITE_APP_BASE_URL` 对应值）纳入同源白名单，否则所有写接口会被 403 自锁。Vite 代理是 server→server 转发，代理后的 `Origin` 头需按此规则处理。
- **SES 沙盒静默失败**：SES 沙盒仅投递"已验证收件人"，未验证收件人注册/找回会接口返回成功却永收不到邮件，账号不可达。此为上线前置（须验证发件域名/DKIM 并申请 SES 生产访问），实现层需在 runbook/验收中确认生产访问已批，而不应把它当普通 200 成功对待。
- **密码字节长度**：前端 zod 与后端校验均按 `Buffer.byteLength` 计字节数（非 JS `.length`），最短 8 字节、最长 72 字节，避免多字节字符（中文/emoji）误判 72 上限。
- **已验证账号重发**：`/resend-verification` 在 `email_verified_at` 已存在时仍返回 `200`，但不再发码、不改验证状态、不泄露是否已验证（PRD §5/AC25）。

> 兼容性提醒：
> 1. 本文件命名 `DESIGN_CN.md` 而非 `DESIGN.md`，因为仓库已有 `docs/design.md`（小写，英文）。Windows 大小写不敏感，`DESIGN.md` 会覆盖它。
> 2. 源 `docs/design.md` 第 3 节仍写"纯 CSS；不引入 UI 组件库"。本次确认改用 Tailwind CSS，需同步更新该处表述以保持一致。
> 3. 新增 `packages/schemas` 是为落实已确认的"zod 前后端共享 schema"决策；它是该决策的实现手段，不是新增功能。
