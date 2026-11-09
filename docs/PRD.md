# PRD — Veriflow（Vite + React 邮箱登录，AWS Serverless）

## 1. 产品概述
- **一句话定位**：Veriflow 是一个部署在 AWS Serverless 上的登录优先小网站，用于演示完整的自建邮箱账号认证闭环。
- **目标用户**：需要注册/登录该网站的访客；查看项目代码与部署方式的技术人员（含面试评估者）。
- **核心价值**：用托管服务（API Gateway、Lambda、DynamoDB、SES、SQS、CloudFront、CDK）亲手实现并展示可解释的云上认证，月成本低于 $5（不含域名）。

## 2. 功能清单

| 功能 | 优先级 | 说明 |
|---|---|---|
| 注册（邮箱 + 密码）+ 发送 6 位邮箱验证码 | P0 | 写入用户、发码入队 |
| 邮箱验证（6 位码） | P0 | 未验证邮箱不能登录 |
| 登录（邮箱 + 密码） | P0 | 失败后统一文案；限流与锁定 |
| 退出登录 | P0 | 删除会话、清 Cookie |
| 刷新页面保持登录（7 天会话） | P0 | HttpOnly Cookie + `/api/auth/me` |
| 忘记密码 / 重置密码（6 位邮箱码） | P0 | 重置后旧会话失效 |
| 重发验证码页 `/resend-verification` | P0 | 统一提示，不暴露邮箱是否存在 |
| 可选邮箱码 MFA（默认关，设置页开关） | P1 | 登录多一步邮箱码 |
| 登录 / 注册 / 发码 / MFA 限流 | P0 | DynamoDB 共享状态 |
| 密码 5 次失败锁 15 分钟、MFA 10 次失败锁 15 分钟 | P0 | 账号级短锁 |
| AWS 部署（API Gateway、Lambda、DynamoDB、S3、CloudFront、SES、SQS、IAM、CDK） | P0 | 同源 `/api/*` |
| GitHub Actions：测试、构建、CDK 部署、前端发布 | P1 | OIDC 无长期密钥 |
| 基础告警（Lambda 错误、5xx、DLQ、DynamoDB 限流、SES 退信）+ 成本告警 | P1 | SNS 通知 |

## 3. 页面清单

| 页面 | 路由 | 结构与核心交互 |
|---|---|---|
| 登录 | `/login` | 邮箱、密码、显示/隐藏密码；登录、注册链接、忘记密码链接。提交 `POST /api/auth/login`；返回 `mfaRequired` 则携带 `mfaToken` 跳 `/mfa` |
| 注册 | `/register` | 邮箱、密码、确认密码；注册、登录链接。提交 `POST /api/auth/register`；跳 `/verify-email?email=` |
| 邮箱验证 | `/verify-email` | 邮箱（只读）、6 位码输入、验证、重发。提交 `POST /api/auth/verify-email` |
| 重发验证 | `/resend-verification` | 邮箱、重发；统一提示"如果存在，已发送" |
| 忘记密码 | `/forgot-password` | 邮箱、发送；统一提示 |
| 重置密码 | `/reset-password` | 邮箱、6 位码、新密码、确认。提交 `POST /api/auth/reset-password` |
| MFA 验证 | `/mfa` | 6 位码、验证、重发、回登录；需 `location.state.mfaToken` |
| 应用主页 | `/app` | 登录成功占位页；路由守卫保护 |
| 设置 | `/app/settings` | MFA 开关；开启/关闭流程需当前密码 + 邮箱码 |
| 未找到 | `*` | 404 |

## 4. 数据模型（用户 / 账号）

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `email` | String | 是 | 主键；trim + lowercase |
| `user_id` | String | 是 | `crypto.randomUUID()` |
| `password_hash` | String | 是 | bcryptjs cost 12 |
| `email_verified_at` | Number(epoch秒) | 否 | 未验证则不存在 |
| `mfa_enabled` | Boolean | 是 | 默认 false |
| `failed_login_count` | Number | 是 | 连续密码失败计数 |
| `failed_mfa_count` | Number | 是 | 连续 MFA 失败计数 |
| `locked_until` | Number(epoch秒) | 否 | 锁定到期；未锁定则不存在 |
| `session_version` | Number | 是 | 初始 1；重置密码 +1 |
| `created_at` | Number | 是 | epoch 秒 |
| `updated_at` | Number | 是 | epoch 秒 |

关联表（各自的访问模式见设计文档，不属于本表字段）：Sessions（`token_hash` PK）、VerificationCodes（`scope`+`sort_key` PK）、RateLimits（`bucket` PK）。

## 5. 业务规则

- **字段校验**：邮箱格式校验、trim+lowercase；密码最短 8 **字节**、最长 72 **字节**，按 `Buffer.byteLength` 计字节数（非 UTF-16 码元 `length`）；不 trim、不限特殊字符；确认密码须一致。
- **验证码**：6 位数字、10 分钟有效、最多 5 次尝试、60 秒重发、单次使用；只存 HMAC-SHA256，不存明文、不返回客户端。**尝试次数按"码"计，不按"账号窗口"计；每次重发会 mint 一条 `attempts=0` 的新码并令旧码失效，因此重发会重置该码预算——重发限流（而非"5 次"）才是真实的试错边界。**
- **会话**：7 天；HttpOnly + Secure + SameSite=Lax；库只存 token 的 SHA-256；不做滑动续期；重置密码 `session_version += 1` 使旧会话失效。**会话固定防护：登录成功与 `session_version` 变更时必须签发全新 `token_hash` 的会话 Cookie，不复用登录前已有的任何会话（含匿名预置 cookie），旧 token 服务端作废。**
- **MFA**：可选、默认关；只做邮箱码；`mfaToken` HMAC 签名、10 分钟；仅放 React state，不落 localStorage。**`mfaToken` 单次使用：`mfa/verify` 成功后该 token 作废，不可重放。同一账号可并存多个未过期 `mfaToken`（每次登录 challenge 都签发新 token），设计不提供单 token 撤销，依赖 10 分钟过期与 `session_version` 失效。**
- **限流**：登录 IP 20/15 分钟；注册 IP 5/1 小时；重发注册码 邮箱 3/15 分钟；验证邮箱 邮箱 10/15 分钟、IP 30/15 分钟；忘记密码 邮箱 3/15 分钟、IP 10/1 小时；重置密码提交 IP 10/15 分钟；MFA 验证 IP 30/15 分钟；MFA 重发 账号 3/15 分钟；MFA 设置发码 账号 5/15 分钟。**限流为固定窗口（`window_start = floor(now/window)`）；窗口交界处的突发会分别计入前后两窗口，瞬时可通过约 2 倍限额。此为已知取舍；若需严格上限须改滑动窗口/令牌桶。**
- **锁定**：密码连续失败 5 次 → `locked_until = now+15min`；MFA 连续失败 10 次 → 同；登录成功清零；重置密码清零。**重置密码失败不计入账号锁定**：`reset-password` 的码试错既不增 `failed_login_count` 也不增 `failed_mfa_count`，账号永不因重置试错被锁，仅受重置接口 IP 限流（10/15 分钟）约束。此为已知取舍（6 位码 + 10 分钟过期已拉长爆破窗口）；若要求重置失败也锁账号，须将其并入 `failed_login_count` 或独立计数并接同一把锁。
- **重发验证码（已验证账号）**：若账号 `email_verified_at` 已存在，提交 `/resend-verification` 仍返回 `200`，但**不发送新码、不修改验证状态、不通过文案或差异响应暴露账号是否已验证**（统一提示"如果存在，已发送验证码"）。
- **错误文案**：统一、简短；不暴露"账号是否存在"；登录失败不区分"密码错"与"邮箱不存在"；锁定不区分密码/MFA。
- **安全底线**：bcrypt 哈希；验证码 HMAC；Cookie 属性；日志不记密码/码/Cookie/完整 token；IP 哈希、UA 截断；DynamoDB 只用 GetItem/Query/PutItem/UpdateItem/DeleteItem/TransactWriteItems，禁止 `Scan`、无 GSI；IAM 最小权限、S3 阻止公开访问、密钥存 Secrets Manager。

## 6. 异常场景

- 邮箱已被注册（注册 409）。
- 验证码错误 / 过期 / 已消费（400）。
- 60 秒内重复发码（429）。
- 登录达限流（429 + Retry-After）；账号锁定（423 + Retry-After）。
- 邮箱未验证登录（403）。
- 发信队列不可用（503）；入队成功但 SES 失败（用户未收信，可重发）。
- 网络/服务端 5xx（前端统一错误提示，任意 401 回登录）。
- `/mfa` 刷新丢失 `mfaToken` → 回 `/login`。
- 重置密码后旧会话下次请求被拒并删除。
- TTL 延迟导致过期项仍可被读到（读必查 `expires_at`）。
- 同一验证码并发消费、同一邮箱并发注册（条件写保证仅一个成功）。
- 锁定 DoS：攻击者可锁他人账号（短锁 + IP 限流缓解）。
- MFA 邮箱丢失：无恢复码，需管理员在控制台/DynamoDB 处理。
- `mfaToken` 重放与并存：成功 `mfa/verify` 后 token 应作废；同账号可并存多个未过期 `mfaToken`，旧 token 在 10 分钟窗口内仍可被使用，无单 token 撤销。
- 重发验证码重置尝试预算：每次重发 mint 新码并令旧码失效，旧的 5 次尝试预算被重置；"5 次"不是账号级上限，重发限流才是真实边界。
- 边界突发（固定窗口限流）：窗口交界处的突发可瞬时通过约 2 倍限额（前后两窗口各计一次）。
- 密码字节长度：多字节字符（中文/emoji）下，按 `length` 计会误判 72 上限；须按字节长度校验；bcrypt 静默截断到 72 字节。
- 重置密码失败不走账号锁：重置码试错只受 IP 限流约束，账号永不因重置试错被锁。
- 会话固定：若登录成功复用登录前已有的会话 Cookie（含匿名预置 token），攻击者可诱导受害者沿用旧 token。防护为登录成功/提权时重签发全新会话（见 §5 会话段）。
- 重发验证码（已验证账号）：已验证账号请求 `/resend-verification` 的行为未定义则实现会各写各的；按 §5 规则应返回 200 但不再发码、不改状态、不泄露验证状态。
- SES 沙盒静默失败（保留）：SES 沙盒仅投递"已验证收件人"，未验证收件人注册/找回会接口返回成功却永收不到邮件，账号不可达；属上线前置（须申请 SES 生产访问），作为已知风险保留。
- 开发期 Origin 同源误杀（保留）：前端 `:5173` 调 API `:3001` 时，若 `Origin` 校验未对开发浏览器 origin 白名单放行，所有写接口会被 403；属开发配置约束，作为已知风险保留。
- 邮箱归一化不一致（保留）：仅注册做 trim+lowercase 而其它入口未做，会导致同一邮箱被当成不同账号（身份分裂）；应统一在请求最前端收口 `normalizeEmail()`，作为已知风险保留。

## 7. 验收标准（Given/When/Then，可验证）

1. **注册成功并发码**
   - Given 未注册的合法邮箱与合规密码
   - When 提交 `/api/auth/register`
   - Then 返回 `201`，`VerificationCodes` 写入一条 `purpose=verify_email` 记录，SQS 出现一条发信消息，用户 `email_verified_at` 不存在

2. **重复注册**
   - Given 已注册邮箱
   - When 再次提交 `/api/auth/register`
   - Then 返回 `409 EMAIL_ALREADY_REGISTERED`

3. **邮箱验证**
   - Given 正确的 6 位码（未过期、未超 5 次尝试、未消费）
   - When 提交 `/api/auth/verify-email`
   - Then 返回 `200`，用户 `email_verified_at` 被写入，码被标记 `consumed_at`

4. **未验证邮箱不能登录**
   - Given 已注册但未验证邮箱
   - When 提交正确的 `/api/auth/login`
   - Then 返回 `403 EMAIL_NOT_VERIFIED`

5. **登录失败统一文案**
   - Given 不存在的邮箱 或 密码错误的已存在邮箱
   - When 提交 `/api/auth/login`
   - Then 均返回 `401 INVALID_CREDENTIALS`，响应体不含"邮箱不存在"

6. **登录限流与锁定**
   - Given 同一 IP 连续 20 次登录失败
   - When 第 21 次在 15 分钟内提交
   - Then 返回 `429 RATE_LIMITED` 且带 `Retry-After`
   - Given 同一账号密码连续失败 5 次
   - When 第 6 次提交（含正确密码）
   - Then 返回 `423 ACCOUNT_LOCKED` 且带 `Retry-After`，15 分钟内任何凭证均 423

7. **MFA 开启后的登录**
   - Given 用户 `mfa_enabled=true`
   - When 提交正确密码登录
   - Then 返回 `200 { mfaRequired: true, mfaToken }`，**不**设置会话 Cookie

8. **MFA 验证通过**
   - Given 有效的 `mfaToken` 与正确邮箱码
   - When 提交 `/api/auth/mfa/verify`
   - Then 返回 `200 { user }` 并设置会话 Cookie

9. **会话保持**
   - Given 已登录用户刷新页面
   - When 应用启动调用 `GET /api/auth/me`
   - Then 返回 `200 { user }`；未登录返回 `401` 且前端跳 `/login`

10. **退出登录**
    - Given 已登录用户
    - When 提交 `POST /api/auth/logout`
    - Then 返回 `204`，`Sessions` 对应项被删除，Cookie 被清除

11. **找回密码使旧会话失效**
    - Given 已登录用户 A，另一设备发起忘记密码→重置成功
    - When 用户 A 下一次请求携带旧会话 Cookie
    - Then 返回 `401`，旧会话被删除

12. **Cookie 属性**
    - Given 生产环境登录成功
    - When 检查 `Set-Cookie`
    - Then 含 `HttpOnly`、`Secure`、`SameSite=Lax`、`Path=/`、`Max-Age=604800`，且名为 `__Host-app_session`（带前缀）

13. **验证码不落明文**
    - Given 任意发码动作完成
    - When 直接读取 `VerificationCodes` 表项
    - Then `code_hash` 为 HMAC 值，表中不存在原始 6 位码字段

14. **限流状态跨实例有效**
    - Given 至少 10 个并发登录请求命中同一限流键
    - When 计数达到限额
    - Then 超限请求全部返回 `429`，且总计数不超过限额 + 容差（DynamoDB 原子 `ADD`）

15. **静态资源与 API 同源**
    - Given 通过 CloudFront 域名访问
    - When 浏览器加载前端并发起 `/api/*` 请求
    - Then 请求经同一 CloudFront 分发到 API Gateway，Cookie 不跨站

16. **安全响应头**
    - Given 通过 CloudFront 访问任意页面
    - When 检查响应头
    - Then 含 `Content-Security-Policy`、`Strict-Transport-Security`、`X-Content-Type-Options: nosniff`

17. **请求体大小限制**
    - Given 提交超过 20KB 的 JSON 登录请求体
    - When 到达 API
    - Then 被 413/拒收，不进入业务处理

18. **日志不含敏感信息**
    - Given 一次完整登录与发码流程
    - When 检查 CloudWatch 结构化日志
    - Then 日志中不存在明文密码、验证码、Cookie、完整 `mfaToken`、`Authorization`

19. **密码字节长度边界**
    - Given 一个恰为 72 字节的多字节密码（如含中文/emoji）与 73 字节密码
    - When 提交注册
    - Then 72 字节成功 `201`；73 字节被拒（`VALIDATION_ERROR`），校验按 `Buffer.byteLength` 而非 `.length`

20. **mfaToken 单次使用**
    - Given 一个已用于 `mfa/verify` 成功的 `mfaToken`
    - When 用同一 `mfaToken` 再次提交 `mfa/verify`
    - Then 返回 `401`（不可重放）；仅未使用过的有效 token 可通过

21. **重发重置验证码预算**
    - Given 某码已试错 5 次（已达上限）
    - When 触发重发验证码（冷却满足）生成新码
    - Then 新码 `attempts=0`、可重新尝试；旧码被忽略；重置预算靠重发限流约束

22. **固定窗口边界突发**
    - Given 在限流窗口交界前后各发起接近限额的请求
    - When 统计窗口交界 1 秒内的总通过量
    - Then 不超过该动作限额的 2 倍（固定窗口已知取舍，非严格上限）

23. **重置密码失败不锁账号**
    - Given 对同账号反复提交错误重置码直至超过 5 次
    - When 随后用正确密码登录该账号
    - Then 账号未被 `locked_until` 阻塞（重置失败不计入 `failed_login_count`/`failed_mfa_count`，仅受重置接口 IP 限流约束）

24. **登录成功后重签发会话（防会话固定）**
    - Given 登录前浏览器已持有任意 `app_session` Cookie（含匿名预置）
    - When 提交正确凭证登录成功
    - Then 响应 Set-Cookie 为新 `token_hash`，旧会话在服务端被作废；后续请求仅新 token 有效

25. **已验证账号请求重发验证码**
    - Given 账号 `email_verified_at` 已存在
    - When 提交 `/resend-verification`
    - Then 返回 `200`，但不再发送新验证码、不修改 `email_verified_at`、不通过文案/差异响应暴露"账号是否已验证"

## 8. 本期不做的功能（Scope Out）
- 第三方登录（Google/GitHub 等 OAuth）、Passkey、TOTP、短信验证码
- 扫码登录、企业 SSO / SAML
- 多区域、高可用、灾备演练
- WAF、图文/滑块验证码、设备指纹、完整风控体系
- 管理员后台、用户管理、审计日志查询页
- 自定义邮件模板系统、邮件打开率
- MFA 恢复码（邮箱丢失仅管理员在控制台处理）
- 博客 / 项目介绍内容（应用主页为占位）
- 滑动续期会话、"记住我"独立勾选框
- 公开 Landing 首页（`/` 先纯门跳 `/app`）
