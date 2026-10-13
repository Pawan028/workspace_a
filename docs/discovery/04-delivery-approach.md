# 04 — Delivery Approach and Proposed Architecture

> Status: **PROPOSAL.** The final stack is recorded only after discovery answers arrive (ADR-0001). Dates assume a kickoff on or around 2026-10-20 and are re-baselined after the client meeting.

## 1. Phased releases

| Release | Target | Goal | In scope | Explicitly out |
|---|---|---|---|---|
| **R0: Launch-ready** | Before the first hub opens (Nov 2026, about 3–4 weeks) | Operate cleanly and capture correct data from day one | <ul><li>Public website v1: Hindi and English, offer, hub list, lead form, WhatsApp click-to-chat</li><li>Internal lead tracker</li><li>Battery serial register with scan, stock movement, and sale entry</li><li>Dealer onboarding form with document upload to Power Mart (not borrower KYC)</li><li>Excel import tool for October data</li><li>Basic daily KPI view</li><li>Auth plus RBAC</li></ul> | LSP and NBFC APIs (manual SOP instead), IoT, incentives automation |
| **R1: Channel** | Dec 2026 – Jan 2027 | Run hubs and dealers on the platform | <ul><li>Dealer portal: stock, sales, incentive statement (read)</li><li>Loan-file orchestration via the LSP adapter (API, file, or manual)</li><li>NBFC status mirror</li><li>Incentive engine with maker-checker</li><li>Reconciliation v1</li><li>WhatsApp templates</li><li>KPI dashboard (slide 24 top row)</li></ul> | IoT immobilisation, underwriting |
| **R2: Portfolio** | Feb – Mar 2027 | Lender-grade visibility | <ul><li>Repayment and DPD mirror, collection efficiency KPIs (slide 24 bottom row)</li><li>IoT telemetry ingest (health, location with consent, alerts)</li><li>Service and warranty tickets</li><li>Repossession, refurbish, and redeploy lifecycle</li><li>Bihar configuration</li></ul> | New states and products |
| **R3: Scale** | After Mar 2027 | Roadmap (slide 25) | <ul><li>Multi-state, multi-lender, new product types</li><li>Battery-health analytics</li><li>Consented alternate-data feeds to the NBFC</li><li>DPDP full-compliance audit before about May 2027</li></ul> | — |

Every release ships behind feature flags, with a demo to the client product owner and a written release note.

## 2. Architecture proposal (to be ratified in ADR-0001)

**Shape: a modular monolith.** It has one deployable backend with strictly separated domain modules and one Postgres database with a schema per module.
- *Why:* small team, small volume, high need for correctness and transactions, and fast iteration.
- Microservices would add distributed-transaction and ops cost with no benefit at about 6,000 loans.

```
+---------------- Clients ----------------+
| Public site (SSR, Hindi/EN, SEO)        |
| Ops/Hub web app (PWA, offline queue)    |
| Dealer portal (mobile web)              |
| Admin/HO console                        |
+--------------+--------------------------+
               | HTTPS (BFF / API)
+--------------v--------------------------+
| Backend: modular monolith               |
|  identity . leads . customers . battery |
|  channel(hub/dealer) . sales . loanfile |
|  incentives . service . recon . reports |
|  -- ports ----------------------------- |
|  LSP . NBFC . IoT . ERP . WhatsApp/SMS  |
|  payments(deposits) . storage . maps    |
+--+-----------+-----------+--------------+
   |           |           |
 Postgres   Object store  Job queue / outbox -> partner webhooks, analytics
 (India)    (India, KMS)  (retries, recon, notifications)
```

> **Superseded:** the stack table below was the first draft. The current recommendation is [ADR-0001](../adr/0001-architecture-and-stack.md): an Astro static site, a React SPA, a NestJS API, and Postgres with RLS on AWS Mumbai. The full design is in [architecture/01](../architecture/01-system-design.md).

**First-draft stack (historical):**

| Concern | Proposal | Why |
|---|---|---|
| Language | TypeScript end to end (strict) | One language for web and backend, shared types and validation schemas, wide hiring pool |
| Web | Next.js (App Router) for the public site and apps | SSR for SEO and slow networks, i18n, one framework |
| Backend | Node.js with NestJS (or Fastify plus a module convention) | Enforced module boundaries, DI, testability |
| DB | PostgreSQL 16+ | Transactions, constraints, JSONB for partner payloads, RLS if needed |
| ORM / migrations | Prisma or Drizzle; migrations versioned in the repo | Typed queries, reviewable migrations |
| Validation | Zod schemas shared between client and server | One contract source |
| Jobs | Postgres-backed queue (e.g. pg-boss) first | No extra infra at this volume |
| Hosting | AWS **ap-south-1 (Mumbai)** or Azure **Central India**. Client-owned account | Data localisation (RBI DL, CERT-In) |
| Files | S3 or Blob with SSE-KMS, private, signed URLs | KYC-grade document safety |
| Field mobile | Start as a PWA. Move to Expo / React Native only if offline capture, background location, or scanning demands it | Ship fast, avoid app-store friction |
| Observability | OpenTelemetry, structured JSON logs, error tracking, uptime checks | Debuggable from day one |
| IaC / CI | Terraform; GitHub Actions (or the client's choice) | Reproducible environments |

**Decisions deliberately deferred until discovery:** cloud vendor, mobile native vs PWA, KYC vendor (or none, if the LSP owns it), whether to integrate the ERP or run our own invoicing, and the IoT ingest design.

## 3. Core domain model (first cut)

`Region(State) -> Cluster -> City`, `Hub`, `Dealer(hub_id nullable)`, `User(role, scope)`,
`ReferencePartner`, `Lead`, `Customer`, `Vehicle(reg_no)`,
`Battery(serial, bpan?, model, state)`, `BatteryIdentity(history)`, `StockMovement`,
`Sale(battery, customer, channel, price_paise)`, `LoanApplication(mirror: lender_id, external_ref, status)`,
`RepaymentSnapshot(mirror)`, `IncentiveStatement`, `ServiceTicket`, `Consent`, `AuditEvent`, `OutboxEvent`.

**Battery lifecycle (draft):**
`MANUFACTURED -> IN_TRANSIT -> IN_STOCK(hub|dealer) -> RESERVED -> SOLD -> FITTED -> ACTIVE -> (SERVICE <-> ACTIVE) -> REPOSSESSED -> AT_OEM -> REFURBISHED(new identity) -> IN_STOCK ...  | SCRAPPED/RECYCLED`

## 4. Team shape (suggested)
- 1 tech lead / architect
- 2 full-stack engineers
- 1 frontend / UX engineer (Hindi UX)
- 1 QA (automation)
- 0.5 DevOps / SecOps
- A product owner on the client side

Scale up for R2 if the IoT and NBFC integrations prove heavy.
