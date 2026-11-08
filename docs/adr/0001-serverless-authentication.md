# ADR 0001: Use a serverless, self-managed authentication stack

- Status: Proposed
- Date: 2026-11-08
- Deciders: project author
- Related: [Design document](../design.md)

## Context

The project is a small, login-first website with a public portfolio purpose. It must demonstrate practical cloud/backend work, not only application features. The account lifecycle includes registration, email verification, sign-in, optional email-code MFA, password reset, rate limiting, and short account lockouts.

The design has four hard constraints:

1. The monthly bill must stay under **$5** at low traffic, excluding the domain.
2. Authentication logic should be implemented and explainable, not delegated to a managed identity product.
3. The system must use real AWS managed services, not a single EC2 host presented as "cloud".
4. The first version must be deliverable by one developer without a VPC, NAT gateway, RDS instance, or load balancer.

An earlier iteration used Express and SQLite on one host. It met the product requirements but had a fixed server bill, a single point of failure, and little cloud-native surface.

## Options considered

| Option | Summary | Why it was not selected |
|---|---|---|
| A. Cognito + custom UI | Cognito User Pool with a custom React front end | Moves the most interesting work (passwords, sessions, verification, lockout behavior) into a managed black box; does not match the learning goal. |
| B. Cognito Hosted UI | Cognito end to end | Too little application code; the project would look thin. |
| C. EC2 + Express + SQLite | Self-managed monolith on one instance | Fixed monthly cost, single point of failure, and mostly traditional operations; fails the cloud-native and budget goals. |
| D. Serverless, self-managed | API Gateway HTTP API + Lambda + DynamoDB + S3/CloudFront + SES/SQS + CDK | Meets the budget, cloud-native, and learning goals; adds cold starts and DynamoDB modeling work. |

## Decision

Use **Option D**:

- **Frontend:** React + Vite, hosted on a private S3 bucket behind CloudFront.
- **API:** Express running on Lambda through `serverless-http`, exposed by API Gateway HTTP API.
- **Data:** DynamoDB on-demand with four tables: `Users`, `Sessions`, `VerificationCodes`, and `RateLimits`.
- **Email:** SES, fed asynchronously through SQS and a worker Lambda.
- **Infrastructure:** AWS CDK in TypeScript, deployed through GitHub Actions with OIDC.
- **Identity:** self-managed authentication; no Cognito.

The frontend and API share one CloudFront origin. The browser calls `/api/*`, and CloudFront forwards those requests to API Gateway. This keeps session cookies first-party and avoids CORS.

## Consequences

### Positive

- No idle compute or database bill; cost scales with actual use.
- Every layer is an AWS managed service that can be explained in an interview.
- The four-table DynamoDB model is small enough to reason about and does not need a GSI in version 1.
- No VPC, NAT, RDS, or load balancer is required for the first version.

### Negative

- Lambda cold starts can make login slower than a long-running server.
- DynamoDB requires access-pattern-first modeling; there is no ad-hoc join.
- Local development needs DynamoDB Local and a longer debugging path.
- Email delivery is asynchronous; failures must be handled through SQS, a DLQ, and SES feedback.
- The team owns password hashing, session management, verification codes, rate limiting, and lockout logic.

### Follow-up decisions

- Whether to add a one-line public landing page or keep a pure login gate.
- Domain name, AWS region, SES production access, and environment count.
- Whether WAF is worth its fixed monthly cost.
- Whether a future version should add an audit table, a GSI, or a separate MFA service.

These are tracked in the design document's open questions rather than decided here.
