# Architecture 01 — System Design

> Stack rationale: [ADR-0001](../adr/0001-architecture-and-stack.md). Financier design: [02-financier-integration](02-financier-integration.md). Security: [engineering/06](../engineering/06-adversarial-security.md).

## 1. Context (who talks to the system)

```
        Customers (web, WhatsApp)       Reference partners      Investors/public
                 │                              │                      │
                 ▼                              ▼                      ▼
   ┌──────────────────────────── CLN Power Mart Platform ───────────────────────────┐
   │ Public site · Ops app (hub/FOS/HO) · Dealer portal · Admin · API · Workers       │
   └───────┬──────────┬───────────┬───────────┬───────────┬───────────┬─────────────┘
           │          │           │           │           │           │
     Financiers   Payment     WhatsApp /   IoT/BMS     CLN ERP      Maps
     (N, plug-in) gateway     SMS (DLT)    platform    (stock,      (hub
     KYC, loans,  (down pmt,                           invoice,     locator)
     repayments   deposits)                            GST)
```

## 2. Containers (what we deploy)

| Container | Tech | Exposure | Notes |
|---|---|---|---|
| **Public site** | Astro (static build), React islands only where needed | CloudFront + S3, public | **No server at all.** Pages are pre-built in Hindi and English. The lead form posts to the API |
| **App** (ops, dealer, admin) | React + Vite SPA, installable PWA | CloudFront + S3 (static); admin routes need staff SSO | No frontend server. Every operation goes through the API |
| **API** | NestJS on Fastify (Node LTS) | ALB behind CloudFront + WAF | The only component holding business logic and DB access |
| **Worker** | Same codebase, worker entrypoint | Private | Jobs: outbox dispatch, partner sync, file imports, reconciliation, notifications, retention |
| **Webhook ingress** | Same API, separate route group + WAF rules | Public, but signature-checked | Financier, payment gateway, WhatsApp, IoT callbacks |
| **Database** | PostgreSQL (RDS, Multi-AZ) | Private subnet | A schema per module. Row-level security as a second authZ layer |
| **Cache / limits** | Valkey (ElastiCache) | Private | Rate limiting, OTP throttles, short-lived caches. **Never the source of truth** |
| **Object storage** | S3 (SSE-KMS), private buckets | Signed URLs only, served from a separate domain | Dealer docs, fitment photos, import files, partner raw payloads |
| **Identity** | AWS Cognito (ap-south-1) | Hosted endpoints | Staff: passkeys or TOTP MFA (SSO federation if the client has Google or Microsoft). Dealers and partners: phone OTP |

## 3. Backend modules (modular monolith)

| Module | Owns | Key state machine |
|---|---|---|
| `identity` | users, roles, scopes, devices | User: invited → active → suspended → removed |
| `catalog` | products, prices (versioned) | — |
| `channel` | regions, clusters, cities, hubs, dealers, deposits | Dealer: applied → verified → deposit_paid → active ⇄ paused → exited |
| `leads` | leads, referrals, reference partners | Lead: new → contacted → visit_booked → quoted → converted / lost |
| `customers` | customer profile (non-KYC), vehicle, consents | — |
| `inventory` | batteries, identities (serial, BPAN), stock movements | Battery lifecycle (see delivery doc 04 §3) |
| `sales` | quotes, sales, invoices (ref to ERP), fitment | Sale: quoted → financing → approved → paid_down → fitted → completed / cancelled |
| `finance` | financiers, schemes, cases, DOs, disbursals, loan mirror | FinanceCase (architecture 02 §3) |
| `payments` | payment intents, PG events, refunds (Power Mart's own money only) | Payment: created → pending → captured / failed → refunded |
| `incentives` | slabs (config), statements, payouts | Statement: draft → computed → checked → approved → paid |
| `service` | tickets, warranty, repossession → refurbish flow (R2) | Ticket: open → diagnosing → fixed / swapped / sent_to_oem → closed |
| `telemetry` | IoT summaries (SoC, SoH, last location with consent, alerts) (R2) | — |
| `recon` | daily reconciliation runs, exceptions | Exception: open → investigating → resolved |
| `reporting` | read models, KPI dashboards (slide 24) | — |
| `notifications` | WhatsApp, SMS, email templates (DLT IDs), delivery log | — |
| `audit` | append-only audit events | — |

**Cross-cutting:**
- Transactional outbox, which feeds the worker and from there partners and read models.
- Idempotency-key store.
- Feature flags (per state and hub).
- Config versioning with maker-checker.

## 4. Key flows
1. **Lead.** A website form (bot-protected) goes through phone OTP verification and dedupe, is assigned to the nearest hub by pincode, and triggers a WhatsApp acknowledgement. The hub works the lead.
2. **Sale with finance.** See architecture 02 §4.
3. **Dealer onboarding.** Application, then GSTIN, PAN, and bank verification (using a verification API, or manually with evidence), then the agreement, the security deposit via hosted payment link, approval (maker-checker), and finally activation and stock allocation.
4. **Stock.** OEM dispatch (ERP feed or import) creates batteries `IN_TRANSIT`. A scan at the hub moves them `IN_STOCK`, a transfer scan moves them to a dealer, and a fitment scan marks them `FITTED`.
5. **Month end.** Incentive computation from the versioned slab config and *reconciled* sales, then maker-checker, then the payout file, then statements to dealers on WhatsApp and in the portal.
6. **Daily reconciliation.** Sales vs DOs vs disbursals vs fitments vs payment-gateway settlements vs stock. Mismatches go to the exceptions queue.

## 5. Deployment (AWS, India only)

```
AWS Organization (client-owned)
├── prod account (ap-south-1 Mumbai)
│    VPC: public (ALB/NAT) · private-app (ECS Fargate: api, worker) · private-data (RDS, Valkey)
│    CloudFront + WAF (public site, app, api) · S3 (static, docs) · KMS · Secrets Manager
│    GuardDuty · Security Hub · Config · CloudTrail → log-archive account
├── staging account (mirror of prod, smaller, partner sandboxes only)
├── log-archive account (CloudTrail, VPC flow, app audit logs; S3 Object Lock)
└── backup account (AWS Backup vault lock; cross-region copy → ap-south-2 Hyderabad)
```
- CI/CD uses GitHub Actions with OIDC into AWS (no long-lived keys), Terraform for all infrastructure, and per-environment approvals.
- **Rough order-of-magnitude cost** for prod plus staging at this scale: about USD 400–900 a month. This is an estimate to validate with the AWS pricing calculator once the design is final.

## 6. Non-functional targets

| Area | Target |
|---|---|
| Availability (API, app) | 99.9% monthly. Planned maintenance outside 8 am – 9 pm IST |
| RPO / RTO | ≤ 15 min (RDS point-in-time recovery) / ≤ 4 h (restore runbook, drilled monthly) |
| Public site performance | LCP ≤ 2.5 s on a mid-range Android over slow 4G. Almost no JS by default |
| App performance | Interactive in ≤ 4 s on slow 3G after the first load. Scan-to-save in ≤ 1 s online. Works offline with a sync queue |
| API latency | p95 ≤ 300 ms reads, ≤ 600 ms writes at 10× expected peak |
| Scale headroom | Designed for 10× the Mar 2027 plan (about 60k loans, 2k users, 60k devices) without re-architecture |
| Data residency | All data, backups, and logs in India (Mumbai + Hyderabad) |

## 7. Scaling path (only when its trigger fires)
- More states or products: config only (region, product type, and lender are already dimensions).
- IoT volume beyond what Postgres comfortably handles: add TimescaleDB or the vendor's time-series store behind the telemetry port.
- Heavy reporting load: a read replica, then a warehouse (e.g. Redshift Serverless or ClickHouse) fed by the outbox.
- A team per module: extract that module to a service. Its boundaries already exist.
