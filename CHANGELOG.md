# Changelog

All notable project-level changes and decisions for **Veriflow** are recorded in this file.

This file is the single place that records architecture "before → after" changes, decisions, and rationale. Technical documentation bodies are written as pure current-state descriptions and do not carry history, plans, or decision records; those belong here, in issues, and in commit messages.

This project follows [Keep a Changelog](https://keepachangelog.com/) and [Semantic Versioning](https://semver.org/).

## [Unreleased] - 2026-11-09

### Changed

- **Architecture reposition:** the technology stack was repositioned from a serverless stack (API Gateway HTTP API + Lambda + DynamoDB + S3/CloudFront + SES/SQS + CDK) to a single-instance server stack: CloudFront + Elastic Beanstalk + RDS PostgreSQL.
- **Topology A confirmed:** one CloudFront distribution with multiple origins and path-based routing.
  - Default `*` cache behavior → GitHub Pages user site `yhwhn.github.io` as a custom origin, HTTPS Only, static caching.
  - `/api/*` cache behavior → Elastic Beanstalk single-instance origin, HTTP Only (no ALB, no TLS on EB), `CachingDisabled`, forward `Cookie` / `Origin`.
  - The browser's only entry point is the CloudFront default domain; no custom domain, ACM certificate, or Route 53 is used.
  - Same-origin request model retained: no CORS; POST `Origin` validation with a dev allowlist; the `__Host-app_session` cookie with `HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=604800`.
- **Compute:** Elastic Beanstalk single-instance Node.js platform running native Express. No `serverless-http`, no Lambda, no API Gateway.
- **Database:** RDS PostgreSQL single-AZ, accessed with native `pg` + SQL, parameterized queries only. Tables: `users`, `sessions`, `verification_codes`, `used_mfa_tokens`. Expired rows are removed by read-time lazy deletion (every read checks `expires_at`).
- **Rate limiting:** single-instance in-memory counters using a fixed window. A roughly 2× boundary burst is a known characteristic of this approach.
- **Password hashing:** argon2id. Password length of min 8 / max 72 bytes, validated by byte length. No bcryptjs / cost 12.
- **Email:** SQS feeding an EB-resident long-running worker process that calls SES `SendEmail` with plaintext. Enqueue failure returns `503 EMAIL_QUEUE_UNAVAILABLE`; enqueue success with a later SES failure still returns 201/200; 3 retries then a DLQ. Local development uses `EMAIL_DRIVER=console`.
- **Secrets:** loaded at process start from EB environment properties / SSM Parameter Store. No Secrets Manager rotation and no dual-key window.
- **Infrastructure as code:** CDK TypeScript in `infra/` only, region `ap-southeast-1`. GitHub Actions with OIDC publishes the frontend to GitHub Pages. Local development uses `docker run postgres:16` + `npm run dev`.
- **Cost model:** fixed hourly billing of approximately $20–30/month (EB single instance + RDS single-AZ, no ALB). The `<$5/month` cap is removed and no new hard cap is set. Cost alarm visibility is configured. "Pay-per-request" is retained only as a user preference/comment and is not a project goal or advantage.
- **`npm run db:local`:** now initializes the PostgreSQL schema locally (previously it initialized the DynamoDB Local schema).
- **Project name:** stays **Veriflow**; the word "Serverless" is dropped from titles (for example, `Veriflow — AWS Authentication Service`).
- **Documentation language rule (locked):** technical docs / architecture / API specs / code comments → ENGLISH; product requirements / business logic / test cases / end-user operation docs → CHINESE. Per file: `README.md`, `docs/PRD.md`, `docs/ACCEPTANCE.md` → Chinese; `docs/design.md`, ADRs, `CHANGELOG.md` → English; `docs/TASKS.md` → English.
- **Documentation set restructure:** `docs/DESIGN_CN.md` (Chinese design doc) is merged into the English `docs/design.md` and deleted; `docs/TASKS.md` is rewritten in English; `docs/PRD.md`, `docs/ACCEPTANCE.md`, `README.md` are rewritten as Chinese current-state docs; `docs/design.md` is rewritten as an English current-state description with decision/history sections removed. `agent.md` is unchanged.

### Removed

- The `<$5/month` cost cap (no new hard cap set).
- "Pay-per-request" as a stated goal or advantage.
- DynamoDB, and with it DynamoDB Local, GSI, Scan, and the TTL service.
- Lambda.
- API Gateway.
- `serverless-http`.
- S3 private bucket + OAC for frontend hosting.
- Secrets Manager rotation and the dual-key window.
- ACM / Route 53 / custom domain.
- X-Ray.
- The `docs/adr/0001-serverless-authentication.md` file and its Option D (serverless) decision; no replacement ADR is created.
- The `docs/DESIGN_CN.md` file (merged into `docs/design.md`).

### Added

- `CHANGELOG.md` itself, as the record of project-level changes and decisions.
- CloudFront Response Headers Policy supplying CSP / HSTS / `X-Content-Type-Options: nosniff` (AC16).
- CloudFront custom error pages mapping 403/404 → `/index.html` with 200 (SPA fallback).
- The `used_mfa_tokens` table.
- Read-time lazy deletion of expired rows (every read checks `expires_at`).
- A `used_mfa_tokens`-based single-use `mfaToken`.
- A single-instance in-memory rate limiter.
- SSM Parameter Store / EB environment properties as the secrets source.
- `docker run postgres:16` for local development.
- GitHub Pages frontend publishing via GitHub Actions.

### Unchanged

Business behavior and all user-observable outcomes remain equivalent. No new features were added.

- AC1–AC25 observable behavior.
- The full API contract: base `/api`, the endpoint set, `{error:{code,message}}` error shape, `Retry-After` on 429/423, error codes, and the `user` object `{id,email,mfaEnabled}`.
- Authentication implemented by hand, with no Cognito (this goal still holds).
- Frontend pages, routes, components, `AuthProvider` React Context, react-hook-form + zod shared `packages/schemas`, Tailwind CSS.
- 6-digit codes, 10-minute validity, 5 attempts, 60-second cooldown, single use, HMAC storage.
- 7-day session, `session_version`, session fixation defense (AC24), already-verified resend (AC25), `normalizeEmail()`, 72-byte password validation, the rate-limit matrix, lockout rules, uniform error wording, log redaction, the 20KB body limit, and the `__Host-app_session` cookie.
- WAF remains out of scope (not enabled).
- Only the `dev` environment exists; no prod suite.

### Supersedes

- `docs/adr/0001-serverless-authentication.md` (2026-11-08, Status: Proposed) — the serverless Option D decision (API Gateway HTTP API + Lambda + DynamoDB + S3/CloudFront + SES/SQS + CDK, with hard constraints `<$5/month`, no Cognito, no VPC/NAT/RDS/load balancer). This file was deleted in this change. No replacement ADR (no ADR 0002) is created; the supersession is recorded here instead.

## Working agreement

- **Language rule:** technical docs / architecture / API specs / code comments → ENGLISH; product requirements / business logic / test cases / end-user operation docs → CHINESE.
- **Authorization rule:** if anything is unclear, list the questions and wait for confirmation; do not guess the user's intent and do not add features on your own. Writing code, editing docs, and committing each require step-by-step explicit authorization. After a rewrite, stop and wait for review; commit and push happen only on a separate explicit instruction. An earlier overreach (writing `login-app/` code without authorization) was reverted and the repo returned to the design stage.
