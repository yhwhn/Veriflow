# TASKS — Veriflow Development Task List (Vite + React, AWS)

> Sources: `docs/PRD.md`, `docs/design.md`, and the locked project charter.
> Execution order: foundation → backend data layer/services → API → pages → security hardening → infrastructure/CI → E2E acceptance.
> Types: Page / Component / API / Logic. Complexity: S (≤0.5d) / M (1–2d) / L (spans multiple files and services).
> "PRD" points to a feature in `docs/PRD.md` (§2 features, §3 pages, §5 rules, §7 acceptance criteria AC).

## Phase 0: Foundation

| # | Task | Module | Type | Depends on | Complexity | PRD | Acceptance criteria |
|---|---|---|---|---|---|---|---|
| T01 | Workspace scaffold (npm workspaces) | Foundation | Logic | — | S | §1/§2 | Root `package.json` defines web/api/infra/packages sub-packages; each installs and runs independently; `.gitignore` covers node_modules/.env/dist; `.env.example` carries `VITE_APP_BASE_URL` for the dev origin (Vite injects only `VITE_`-prefixed variables); no `VITE_API_BASE_URL` is defined because the frontend calls relative `/api/*` |
| T02 | Shared zod schema package | Shared validation | Logic | T01 | S | §5/§7 | `packages/schemas` exports registration/login/code/reset/MFA zod schemas; web and api reference the same module; fields match §4/§5 |
| T03 | Vite + Tailwind project config | Foundation | Logic | T01 | S | §6/design§11 | `vite.config.js` (`/api` dev proxy to `localhost:3001`), `tailwind.config.js` (content scan, narrow-column tokens), `postcss.config.js`, `index.css` with `@tailwind`; `npm run build` emits `dist` for GitHub Pages |
| T04 | Frontend request wrapper `lib/api.js` | Frontend infra | Logic | T01 | M | §5/§7-AC9/16 | Wrap `fetch`: same-origin relative base, parse `{error:{code,message}}`, 401 triggers logout, read `Retry-After`; cover 401/429/423 parsing in unit behavior |
| T05 | Constants `lib/constants.js` | Frontend infra | Logic | T01 | S | §3 | Route paths and error-code constants centralized; pages reference instead of hardcoding |
| T06 | `AuthProvider` + `useAuth` | Frontend infra | Component | T04 | M | §3/§7-AC9/10 | On startup `GET /api/auth/me` sets `user/loading`; `logout()` calls the API and clears user; any 401 clears user and redirects to `/login` with `from`; hook exposes `user/refresh/logout` |
| T07 | `RequireAuth` route guard | Frontend infra | Component | T06 | S | §3 (/app guard) | `loading` shows skeleton; unauthenticated `Navigate /login state:{from}`; redirect only to in-site paths |
| T08 | Router and entry (router/App/main) | Frontend infra | Component | T06,T07 | S | §3 | `router.jsx` declares public/protected routes; `App.jsx` composes; `main.jsx` wraps `AuthProvider` |
| T09 | Base UI component set | Base components | Component | T03 | M | §3/§6 (a11y) | `AuthCard/TextField/PasswordField/Button/Alert/Spinner`; label binding, `role="alert"`, visible focus, correct autocomplete; Tailwind utility classes |
| T10 | NotFound page | Pages | Page | T08,T09 | S | §3 (*) | `*` route renders 404; unmatched routes land here |

## Phase 1: Backend data layer and services

| # | Task | Module | Type | Depends on | Complexity | PRD | Acceptance criteria |
|---|---|---|---|---|---|---|---|
| T11 | PostgreSQL client and schema access | Backend data layer | Logic | T01 | M | §4/§5 | `pg` client with parameterized SQL only (no string concatenation); tables `users`/`sessions`/`verification_codes`/`used_mfa_tokens`; **every email input passes `normalizeEmail()` (trim+lowercase) before use as key (PRD §6)**; reads check `expires_at` for lazy expiry |
| T12 | Local schema init script | Backend data layer | Logic | T11 | S | §1 (local) | `db:local` initializes the PostgreSQL schema against `docker run postgres:16`; no DynamoDB Local |
| T13 | Password hashing service | Backend service | Logic | T11 | S | §5 (argon2id)/§7-AC5 | argon2id async hash/verify; min 8 / max 72 bytes; dummy hash comparison for nonexistent email |
| T14 | Verification code service | Backend service | Logic | T11 | M | §5 (code rules)/§7-AC3/13 | 6-digit random, HMAC-SHA256 storage, 10-minute TTL, 5 attempts, 60-second cooldown, single use, atomic consume via conditional update |
| T15 | Rate-limit service | Backend service | Logic | T11 | M | §5 (rate-limit matrix)/§7-AC14 | Single-instance in-memory counters per action key/action+IP/window; each action's key/limit/window per PRD §5; over-limit returns 429 + `Retry-After` |
| T16 | Lockout service | Backend service | Logic | T11 | S | §5 (lockout)/§7-AC6 | Password 5 / MFA 10 failures write `locked_until`; login success and reset clear counters; 423 does not distinguish password/MFA |
| T17 | `mfaToken` service | Backend service | Logic | T11 | S | §5 (MFA)/§7-AC7/8 | HMAC sign/verify; 10-minute expiry; validate `purpose` and `session_version` with `timingSafeEqual`; single-use tracked in `used_mfa_tokens` |
| T18 | Email service + SQS adapter | Backend service | Logic | T11 | M | §2 (email)/§5 | `emailService` enqueues to SQS; local `console` adapter; the EB resident worker process consumes SQS and calls SES; enqueue failure yields 503; plaintext body |

## Phase 2: API

| # | Task | Module | Type | Depends on | Complexity | PRD | Acceptance criteria |
|---|---|---|---|---|---|---|---|
| T19 | Express app + server skeleton | API | Logic | T11–T18 | M | §1 | `app.js` with `express.json({limit:'20kb'})` and same-origin `Origin` validation (**dev config must whitelist the browser origin so `:5173`→`:3001` writes are not 403, PRD §6**); `server.js` listens locally; native Express, no `serverless-http` |
| T20 | Unified error middleware | API | Logic | T19 | S | §5 (uniform wording) | `errors.js` emits `{error:{code,message}}`; does not leak account existence; no stack/table name/internal ID |
| T21 | Auth endpoints (register/verify/resend/login/logout/me/forgot/reset) | API | API | T13–T16,T18,T20 | L | §2/§3/§7-AC1..11/AC24/AC25 | `routes/auth.js` implements all endpoints per design §8 contract; status codes match §7 (201/409/403/401/423/429/503); **login success and `session_version` change issue a brand-new `token_hash` session and revoke the old token (AC24); already-verified account `/resend-verification` still returns 200 but sends no code, does not change state, does not reveal verification status (AC25)** |
| T22 | Health check | API | API | T19 | S | §7 | `GET /api/health` returns `200 {status:"ok"}` |
| T23 | MFA settings + mfa/verify/resend endpoints | API | API | T14,T16,T17,T20 | M | §2 (MFA P1)/§7-AC7/8 | `routes/mfa.js`; requires session cookie; enable=current password+code, disable=password+code; `mfa/verify` and `mfa/resend` use `mfaToken` |
| T24 | Session auth middleware | API | Logic | T11 | S | §5 (session) | Read `__Host-app_session` cookie → SHA-256 lookup in `sessions`; check `session_version`; expired/mismatched deletes and returns 401 |

## Phase 3: Pages

| # | Task | Module | Type | Depends on | Complexity | PRD | Acceptance criteria |
|---|---|---|---|---|---|---|---|
| T25 | Register page | Pages | Page | T08,T09,T02,T21 | M | §3 register/§7-AC1/2 | react-hook-form+zod; on success go to `/verify-email?email=`; 409 shows "email already registered" |
| T26 | Email verification page | Pages | Page | T08,T09,T02,T21 | M | §3 verify/§7-AC3 | 6-digit code form + resend; 400 uniform hint, no remaining-attempts display |
| T27 | Resend verification page | Pages | Page | T08,T09,T21 | S | §3 resend/§2 P0 | Submit email to resend; uniform "if it exists, a code was sent" |
| T28 | Login page | Pages | Page | T08,T09,T02,T21,T24 | M | §3 login/§7-AC4/5/6/7 | Email+password; `mfaRequired` → `/mfa` with `state.mfaToken`; 401/403/423/429 wording and `Retry-After` countdown |
| T29 | MFA verify page | Pages | Page | T08,T09,T02,T23,T24 | M | §3 MFA/§7-AC8 | Take `location.state.mfaToken`; lost on refresh → back to `/login`; on success `refresh()` → `/app` |
| T30 | Forgot password page | Pages | Page | T08,T09,T02,T21 | S | §3 forgot/§2 P0 | Submit email; uniform hint |
| T31 | Reset password page | Pages | Page | T08,T09,T02,T21 | M | §3 reset/§7-AC11 | Email+code+new password; on success hint to go to login |
| T32 | App home page | Pages | Page | T08,T09,T06 | S | §3 /app | Post-login placeholder; `RequireAuth` protected |
| T33 | Settings page (MFA toggle) | Pages | Page | T08,T09,T02,T23,T06 | M | §3 settings/§2 MFA P1 | Toggle flow: current password → receive code → confirm; enable/disable both follow §5 |

## Phase 4: Security hardening and details

| # | Task | Module | Type | Depends on | Complexity | PRD | Acceptance criteria |
|---|---|---|---|---|---|---|---|
| T34 | Cookie `__Host-` prefix | Security | Logic | T21,T24 | S | §5/§7-AC12 | Set-Cookie named `__Host-app_session`; `Secure` in production; HTTPS only |
| T35 | CloudFront response headers policy | Security | Logic | T40 | M | §7-AC16 | Response Headers Policy adds CSP/HSTS/`X-Content-Type-Options: nosniff`; attached to the default cache behavior |
| T36 | 20KB request body limit | Security | Logic | T19 | S | §7-AC17 | `express.json({limit:'20kb'})`; oversize rejected before business logic |
| T37 | Log redaction | Security | Logic | T19,T21 | S | §5/§7-AC18 | Structured logs omit plaintext password/code/cookie/full mfaToken/Authorization; IP hashed, UA truncated |

## Phase 5: Infrastructure and delivery

| # | Task | Module | Type | Depends on | Complexity | PRD | Acceptance criteria |
|---|---|---|---|---|---|---|---|
| T38 | DataStack (CDK) | Infrastructure | Logic | T11 | M | §2/§4 | RDS PostgreSQL single-AZ instance; schema migration applied; deletion protection (dev) |
| T39 | ApiStack (CDK) | Infrastructure | Logic | T21,T23,T38,T18 | L | §2/§5 | Elastic Beanstalk single-instance Node.js platform + SQS+DLQ + resident email worker + EB environment properties / SSM Parameter Store + IAM (least privilege) + CloudWatch logs/alarms |
| T40 | WebStack (CDK) | Infrastructure | Logic | T03 | M | §2/§7-AC15 | One CloudFront distribution with two origins: default `*` → `yhwhn.github.io` custom origin HTTPS Only + static caching; `/api/*` → EB origin HTTP Only + `CachingDisabled` + forward Cookie/Origin; custom error pages 403/404→`/index.html` 200; no custom domain/ACM/Route 53 |
| T41 | ObservabilityStack (CDK) | Infrastructure | Logic | T39,T40 | M | §2 (alarms)/§6 | CloudWatch alarms (EB app errors/5xx/DLQ/RDS metrics/SES bounce) + SNS + cost alarm visibility |
| T42 | GitHub Actions | CI/CD | Logic | T38–T41 | M | §2 P1/§7 | `ci.yml` (lint/test/build/cdk synth) + `deploy.yml` (OIDC assume role, `cdk deploy`) + frontend publish to GitHub Pages user site; OIDC, no long-lived keys |
| T43 | End-to-end acceptance script | Testing | Logic | T25–T33,T39 | L | §7 all AC | Playwright covers register→verify→login→MFA→reset→401 redirect; assert AC1..25 pass individually; include session fixation (AC24), already-verified resend (AC25), byte-length boundary (AC19), mfaToken single use (AC20), reset not locking (AC23) |

---

### Dependency map (critical path)
- Foundation: `T01 → T02/T03/T04/T05 → T06 → T07 → T08 → T09/T10`
- Backend: `T01 → T11 → T12–T18 → T19 → T20/T21/T22/T23/T24`
- Pages: `T08+T09 + (T21 or T23) → T25–T33`
- Hardening: `T19 → T36/T37`; `T21/T24 → T34`; `T40 → T35`
- Delivery: `T11/T21/T23/T18 → T38/T39 → T40 → T41 → T42`; `T25–T33 + T39 → T43`

### Out of scope (see PRD §8)
Third-party login / Passkey / TOTP / SMS / QR login / enterprise SSO / multi-region / WAF / CAPTCHA / device fingerprint / risk engine / admin console / email templates / MFA recovery codes / blog content / sliding sessions / public landing page.
