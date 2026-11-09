# ACCEPTANCE — 验收检查清单

> 来源：`docs/PRD.md` §6 异常场景、§7 验收标准（Given/When/Then）、§8 Scope Out。
> 本清单供交付前逐条勾选；每条都对应 PRD 中可验证的验收标准（标注 AC 编号）或异常场景（标注 EX 编号）。

## 一、注册与邮箱验证

- [ ] 注册成功并发码：未注册合法邮箱 + 合规密码 → `201`，`verification_codes` 写入 `verify_email` 记录，SQS 出现发信消息，用户 `email_verified_at` 不存在（AC1）
- [ ] 重发验证码：注册后 `/resend-verification` 可重发，统一提示"如果存在，已发送"，不暴露邮箱是否存在（AC1 / EX: 重发）
- [ ] 已验证账号重发：账号已验证时请求 `/resend-verification` → `200`，不发新码、不改 `email_verified_at`、不暴露是否已验证（AC25）
- [ ] 重复注册：已注册邮箱再次提交 → `409 EMAIL_ALREADY_REGISTERED`（AC2 / EX: 邮箱已注册）
- [ ] 邮箱验证成功：正确 6 位码（未过期/未超 5 次/未消费）→ `200`，`email_verified_at` 写入，码标记 `consumed_at`（AC3）
- [ ] 验证码错误：错误/过期/已消费码 → `400 INVALID_CODE`，不返回剩余次数（AC3 / EX: 验证码错误过期已消费）
- [ ] 60 秒重发冷却：60 秒内重复发码 → `429 CODE_COOLDOWN`（EX: 60 秒重复发码）
- [ ] 验证码不落明文：读 `verification_codes` 仅见 `code_hash`(HMAC)，无原始 6 位码字段（AC13）
- [ ] 并发注册：同一邮箱并发注册仅一个成功（数据库唯一约束 / 条件更新）(EX: 并发注册)
- [ ] 密码字节长度边界：72 字节多字节密码可注册登录，73 字节被拒；校验按字节长度而非 `.length`（AC19）
- [ ] 重发重置验证码预算：旧码试满 5 次后重发生成 `attempts=0` 新码，旧码失效；"5 次"非账号级上限（AC21 / EX: 重发重置预算）

## 二、登录与会话

- [ ] 邮箱未验证不能登录：已注册未验证邮箱 + 正确密码 → `403 EMAIL_NOT_VERIFIED`（AC4）
- [ ] 登录失败统一文案：不存在邮箱 或 密码错误 → 均 `401 INVALID_CREDENTIALS`，响应不含"邮箱不存在"（AC5 / EX: 错误文案）
- [ ] 登录限流：同一 IP 15 分钟内连续失败达 20 次 → 第 21 次 `429 RATE_LIMITED` + `Retry-After`（AC6）
- [ ] 账号锁定：同账号密码连续失败 5 次 → `423 ACCOUNT_LOCKED` + `Retry-After`；15 分钟内任何凭证均 423（AC6 / EX: 锁定）
- [ ] 正常登录（无 MFA）：正确邮箱+密码 → `200 { user }` 且 Set-Cookie `__Host-app_session`（AC5 / AC12）
- [ ] 会话保持：刷新页面时 `GET /api/auth/me` → 已登录 `200 {user}`，未登录 `401` 且前端跳 `/login`（AC9）
- [ ] 退出登录：`POST /api/auth/logout` → `204`，`sessions` 项删除，Cookie 清除（AC10）
- [ ] 会话固定防护：登录前已持有任意 `__Host-app_session`（含匿名）→ 登录成功后 Set-Cookie 为新 `token_hash`，旧 token 服务端作废（AC24）
- [ ] 任意 401 回登录：前端任意 `/api/*` 收 `401` → 清 user 并跳 `/login` 带 `from`（AC9 / PRD§3）
- [ ] 锁定 DoS 缓解：短锁 + IP 限流生效（接受残余风险，验证措施存在）(EX: 锁定 DoS)

## 三、密码找回与重置

- [ ] 忘记密码：提交邮箱 → `200` 统一提示"如果存在，已发送"；无论邮箱是否存在（PRD§3）
- [ ] 重置密码：邮箱+正确码+新密码 → `200`，旧会话失效（AC11 / EX: 找回重置）
- [ ] 重置使旧会话失效：另一设备重置成功后，旧会话下次请求 → `401` 且会话被删除（AC11）
- [ ] 重置失败码：错误/过期码 → `400 INVALID_CODE`（EX: 验证码错误）

## 四、MFA（邮箱码，默认关）

- [ ] 开启后登录：用户 `mfa_enabled=true` 正确密码登录 → `200 { mfaRequired:true, mfaToken }`，不设会话 Cookie（AC7）
- [ ] MFA 验证通过：有效 `mfaToken` + 正确码 → `200 { user }` + Set-Cookie 会话（AC8）
- [ ] MFA 重发：凭 `mfaToken` 重发 → `200`，不返回新码（PRD§3 / design.md §8）
- [ ] MFA 失败锁定：连续失败 10 次 → `423 ACCOUNT_LOCKED`（PRD§5 锁定规则）
- [ ] `/mfa` 刷新丢失：刷新 `/mfa` 丢失 `location.state.mfaToken` → 回 `/login`（EX: mfa 刷新丢失）
- [ ] MFA 设置开关：设置页开启/关闭需当前密码 + 邮箱码；确认后 `mfa_enabled` 变更（PRD§3 设置页）
- [ ] mfaToken 单次使用：已成功的 `mfaToken` 再次 `mfa/verify` → `401`，不可重放（AC20 / EX: mfaToken 重放）

## 五、限流与锁定

- [ ] 限流计数有效：≥10 并发命中同一限流键，达限额后超限请求全 `429`，总计数不超限额 + 容差（API 单实例内存计数）（AC14 / EX: 并发消费）
- [ ] 限流矩阵覆盖：登录/注册/发码/验证/MFA/重置各自键与限额按 PRD§5 生效（PRD§5 限流矩阵）
- [ ] 固定窗口边界突发：窗口交界 1 秒内总通过量不超过限额 2 倍（固定窗口已知取舍）（AC22 / EX: 边界突发）
- [ ] 重置失败不锁账号：对同账号反复错误重置码超过 5 次，随后正确密码登录仍不被 `locked_until` 阻塞（AC23 / EX: 重置不走锁）

## 六、邮件与异步发信

- [ ] 发信队列不可用：入队失败 → `503 EMAIL_QUEUE_UNAVAILABLE`，用户可稍后重发（EX: 队列不可用）
- [ ] 入队成功但 SES 失败：API 仍返回成功，用户未收信可重发，DLQ 告警通知（EX: SES 失败）
- [ ] worker 异步发信：SQS 消息由 EB 常驻 worker 进程消费，调 SES `SendEmail` 纯文本；三次重试后进 DLQ（design.md §10）
- [ ] SES 沙盒/退信：上线前验证域名+DKIM+生产访问；退信率受监控告警（EX: SES 沙盒退信）
- [ ] SES 沙盒静默失败（已知风险）：未验证收件人注册/找回接口返回成功却收不到邮件；验收确认已申请 SES 生产访问（保留风险，EX）

## 七、安全与基础设施

- [ ] Cookie 属性：`Set-Cookie` 含 `HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=604800`，名 `__Host-app_session`（AC12）
- [ ] 静态与 API 同源：经 CloudFront 默认域名访问，`*` 转发 GitHub Pages、`/api/*` 转发 Elastic Beanstalk，Cookie 不跨站（AC15）
- [ ] 安全响应头：经 CloudFront Response Headers Policy，任意页面响应含 `Content-Security-Policy`、`Strict-Transport-Security`、`X-Content-Type-Options: nosniff`（AC16）
- [ ] SPA fallback：CloudFront custom error pages 将 403/404 回 `/index.html` 返回 200，刷新子路由不 404（AC15）
- [ ] 请求体大小：>20KB JSON 登录请求体被拒绝，不进业务（AC17）
- [ ] 日志脱敏：CloudWatch 日志无明文密码/验证码/Cookie/完整 mfaToken/Authorization；IP 哈希、UA 截断（AC18 / PRD§5）
- [ ] 参数化 SQL：数据库访问全部参数化，无字符串拼接（PRD§5 安全底线）
- [ ] 密钥管理：密钥来自 EB 环境属性 / SSM Parameter Store，进程启动时加载，不在仓库（PRD§5）
- [ ] 读时惰性删除：所有读查询核对 `expires_at`，过期行不被误用（EX: 读时惰性删除）

## 八、前端异常与交互

- [ ] 网络/5xx：前端统一错误提示，任意 `401` 回登录（EX: 网络 5xx）
- [ ] 验证码统一提示：只显示统一错误，不显剩余次数（PRD§5 错误文案）
- [ ] 路由守卫：`/app`、`/app/settings` 未登录跳 `/login`；登录后回跳 `from`（仅站内路径）(PRD§3)
- [ ] `/` 纯门：未登录 `/` 跳 `/app` → 再跳 `/login`（锁定决策：先纯门，Landing 待定）

## 九、整体检查

- [ ] 路由：公开（/login、/register、/verify-email、/resend-verification、/forgot-password、/reset-password、/mfa）与受保护（/app、/app/settings）划分正确，`*` → 404
- [ ] 样式：Tailwind CSS 生效，浅色窄栏、单一低饱和强调色、无渐变/玻璃拟态
- [ ] 响应式：移动端单列、按钮全宽
- [ ] 可访问性：输入绑定 `label`、错误 `role="alert"`、焦点可见、按钮 loading/disabled、`autocomplete` 正确
- [ ] 状态管理：仅一个 `AuthProvider` 管全局认证态；表单态由 react-hook-form 局部管理；不引 Redux/Zustand
- [ ] 校验共享：前端与后端引用 `packages/schemas` 同一份 zod 规则
- [ ] mfaToken 不落存储：仅 React state / `location.state`，不进 localStorage/sessionStorage
- [ ] 开发期 Origin 误杀（已知风险）：本地 `:5173` 前端调 `:3001` API 时，开发配置须对浏览器 origin 放行同源校验，否则写接口被 403（保留风险，EX）
- [ ] 邮箱归一化（已知风险）：所有入口统一 `normalizeEmail()`（trim+lowercase）后再作主键/限流键，防止身份分裂（保留风险，EX）
- [ ] 范围外未混入：无第三方登录/Passkey/TOTP/短信/扫码/SSO/多区域/WAF/图文验证码/设备指纹/管理后台/邮件模板/MFA 恢复码/博客内容/滑动续期/公开 Landing（PRD§8）
- [ ] 成本：成本为按小时固定计费（EB 单实例 + RDS 单 AZ 约 $20–30/月），无硬上限；成本告警可见性已配（PRD§1/§2）
- [ ] 本地启动：`docker run postgres:16` → `npm install` → `npm run db:local` → `npm run dev` 全流程可跑（TASKS T01/T12）
