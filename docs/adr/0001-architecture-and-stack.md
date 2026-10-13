# ADR-0001: Architecture shape and technology stack

- Status: **Proposed (revision 3)**, awaiting sign-off by Pawan
- Date: 2026-10-13
- History:
  - r1: TypeScript/Next.js draft.
  - r2: Astro + Vite SPA + NestJS.
  - r3: same frontends; the backend moves to Java/Spring and a native dealer app is added, because the delivery-time constraint was removed (discussion log D18–D21).
- Related: [architecture/01](../architecture/01-system-design.md), [engineering/06](../engineering/06-adversarial-security.md)

## Context
- Money-adjacent and PII-heavy, with an adversarial threat model (dealer-side incentive fraud is the top threat).
- Multiple, changing financiers, some exchanging files (SFTP or PGP), some via APIs.
- India data residency.
- Small volume today, designed for 10×.
- **Team decision (D18):** optimise for the best long-term system rather than the fastest delivery. The first hubs bridge the gap with the website plus a strict Excel template.

Selection criteria, in order: **security surface → correctness → long-term maintainability and handover → performance on low-end Android → delivery speed → run cost.**

## Decision

| Layer | Choice | Why |
|---|---|---|
| Public website | **Astro**, static, Hindi and English | No server attack surface. Near-zero JS by default gives the best performance on low-end phones and patchy networks. Built-in i18n routing |
| HO / hub / admin web | **React + Vite SPA** (TanStack Router and Query) | Authenticated app with no SEO need. No frontend server. Purpose-built for client-side apps |
| Dealer / field app | **React Native (Expo), Android**, distributed privately via **Managed Google Play** | Play Integrity (blocks rooted phones, emulators, tampered apps), mock-location detection, reliable scanning, an encrypted offline queue. These directly counter threat T1 |
| Backend | **Java (current LTS) + Spring Boot + Spring Modulith + Spring Security** | Mature transactions, module boundaries verified in tests, a built-in event publication registry (outbox), the most proven security framework in fintech, a calmer supply chain than npm, strong SFTP and PGP libraries for financier files, deep fintech talent in India |
| Data access | **PostgreSQL** (RDS Multi-AZ) with **row-level security**. **Flyway** migrations. **jOOQ** or Spring Data JDBC (no heavy ORM magic) | Explicit SQL for money-adjacent logic. RLS as a second authZ layer |
| Jobs | Spring Modulith events (outbox) + **db-scheduler** or JobRunr | No extra infrastructure. Exactly-once effects |
| API contract | **OpenAPI-first**. TypeScript clients and Zod schemas **generated** for web and mobile | Replaces "shared TS types". The contract is the reviewed source of truth |
| Identity | AWS Cognito (ap-south-1). Spring Security as the OAuth2 resource server | Managed, India-resident |
| Testing | JUnit 5, Testcontainers, **ArchUnit**, **jqwik** (property-based), **PIT** (mutation testing), REST Assured. Frontend: Vitest, Playwright. Mobile: Maestro | The best-in-class tooling for our testing standards |
| Infrastructure | AWS ap-south-1 (Mumbai), backups and DR in ap-south-2 (Hyderabad). ECS Fargate, CloudFront + WAF, S3, KMS, Secrets Manager, GuardDuty, Security Hub, CloudTrail. Terraform. GitHub Actions with OIDC | Unchanged from r2 |
| Analytics / ML (R3) | A separate **Python** service fed by the outbox | The right tool when that need arrives |

Exact versions are pinned at repository initialisation: the latest LTS or stable releases that are at least 2 weeks old.

## Options rejected, and why

- **Next.js (in any mode).**
  - Its strengths (per-request SSR, Server Components, server actions, middleware) are features we deliberately forbid. Business logic must live only in the backend.
  - Those features were where the 2025 critical CVEs landed (CVE-2025-29927, CVE-2025-55182).
  - Static-export mode disables most of the framework and adds friction (dynamic routes, no middleware, no headers).
  - It ships the React runtime on every public page.
  - **Reconsider if:** we need per-request personalised SEO pages, or the frontend team can only maintain Next.js.
- **Node/NestJS backend.** This was the r2 choice, made for speed with two people and one language. With the time constraint removed, Java/Spring wins on transactional maturity, enforced modularity, test tooling, and supply-chain risk. **Reconsider if** the team cannot review Java comfortably.
- **Python (Django or FastAPI) backend.** Weaker static typing for money-critical domain logic. Kept for analytics later.
- **Microservices.** Unnecessary at this scale.
- **Low-code CRM.** Weak for the battery lifecycle, reconciliation, and financier connectors.

## Consequences
- Three codebases (web, mobile, backend) in two languages. A monorepo with OpenAPI codegen keeps them in sync, and a contract change fails CI on any client that breaks.
- `docs/engineering/02-coding-standards.md` must be rewritten for Java and Spring after sign-off, and the TypeScript rules kept for web and mobile.
- The dealer app needs Play Console / Managed Google Play setup under the client's account.
- A week-1 spike: Cognito with Spring Security, phone OTP via DLT SMS, and the Play Integrity verification flow.
