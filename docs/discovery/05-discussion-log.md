# 05 — Team Discussion and Decision Log

> A running record of what we (the two-person dev team) discussed and decided. Newest entries at the bottom. Each decision has a status: **Agreed** (team), **Proposed** (needs client or partner confirmation), or **Superseded**.

## 2026-10-13 — Session 1: Deck review and first positions
- Reviewed the client deck (26 slides). Our understanding is in [01-project-understanding](01-project-understanding.md), risks in [02](02-complexity-and-risk-register.md), and the delivery plan in [04](04-delivery-approach.md).
- Research surfaced regulatory points that affect features in the deck:
  - remote battery on/off;
  - "mobile data" underwriting;
  - repayment money flow;
  - recovery-agent rules (from 1 Jan 2027);
  - DPDP (core duties from about May 2027).

## 2026-10-13 — Session 2: Developer talk (scope and model)

| # | Decision | Status |
|---|---|---|
| D1 | **The client is non-technical and doesn't know RBI or tech rules.** We own all technical and regulatory thinking. We present simple business choices, never regulation lectures. Our job is to make the idea real without the client ever hitting a regulatory or technical wall. | Agreed |
| D2 | **Power Mart is the merchant (seller) only.** Financiers (NBFCs, banks, lending platforms) do all regulated lending work: KYC, bureau checks, underwriting, KFS, e-sign, mandate or QR, disbursal, collections, recovery. Power Mart never collects borrower KYC documents. | Agreed |
| D3 | **We build:** the website, leads and referrals, hubs and dealers, battery stock by serial, sales, the down payment, the finance-application hand-off and status tracking, fitment, service, incentives, reconciliation, and dashboards. | Agreed |
| D4 | **No online checkout** for a Rs 75k battery in the near term. The flow is enquire → book a visit → buy at the hub or dealer. An online booking token is a possible later feature. | Agreed |
| D5 | **No remote battery immobilisation built by us.** Telemetry and monitoring only. Any lock feature is the financier's legal decision, made later with a written opinion. | Agreed |
| D6 | **Phased delivery:**<ul><li>R0 (launch-ready, about 4–5 weeks)</li><li>R1 (Dec–Jan, channel and finance)</li><li>R2 (Feb–Mar, portfolio, service, IoT)</li><li>R3 (scale)</li></ul> | Agreed (dates re-baselined after the client meeting) |
| D7 | **Team split:**<ul><li>**Pawan:** client communication and decisions, financier and partner coordination, real-device and field testing, account ownership and access.</li><li>**Claude:** most code, tests, docs.</li><li>**Together:** architecture, code review, release decisions.</li></ul> | Agreed |

## 2026-10-13 — Session 3: The client's first meeting outcome, and financiers
- **Client input:** each product has its own financier, products may have different financiers, and financiers may change or new ones may be added.

| # | Decision | Status |
|---|---|---|
| D8 | **The financier is a plug-in, not a hard-wired integration.** We build a financier registry, versioned finance schemes (product × financier × region × channel), a standard status language, and connectors at four levels: Manual, Link, File, API. See [architecture/02-financier-integration](../architecture/02-financier-integration.md) and [ADR-0002](../adr/0002-financier-plug-in.md). | Agreed |
| D9 | **The client chooses financiers; we never choose a lender for them.** We may *suggest* a lending platform or an EV financier only for a product that has no financier. | Agreed |
| D10 | **Manual-first.** Every new financier goes live on day one in Manual mode: staff use the financier's app and record the reference in our system. Sales never wait for an integration. | Agreed |
| D11 | **Consent per named financier.** Customer data goes to a financier only after the customer consents to that specific financier. A rejection followed by a try with the next financier needs fresh consent. | Agreed |
| D12 | **Changing a financier never touches existing loans.** Old loans stay with the original financier to closure. Schemes are versioned with effective dates. | Agreed |

## 2026-10-13 — Session 4: Money, gateways, security

| # | Decision | Status |
|---|---|---|
| D13 | **Correction to the earlier wording "we never touch money".** Precisely:<ul><li>Power Mart **does** receive its own **sale proceeds**: the down payment, the financier's disbursal paid to the seller for the battery (the standard in-store finance model, net of subvention or charges), and dealer security deposits.</li><li>Power Mart **never** receives, holds, or passes through **loan repayments (EMIs)**. Those go only to the financier's own collection account or QR.</li></ul> | Agreed |
| D14 | **Two kinds of "gateway".**<ul><li>A **payment gateway** collects Power Mart's own money.</li><li>A **lending gateway** connects to lenders. No plug-and-play one exists for this use case, so our financier plug-in system *is* the lending gateway.</li></ul> | Agreed |
| D15 | **Payment gateway: hosted checkout and payment links only.** Card and UPI data never touch our servers. Shortlist: **Cashfree** or **Razorpay**; confirm each is on the RBI list of authorised payment aggregators, and compare rates. See [ADR-0003](../adr/0003-payment-gateway-hosted-checkout.md). | Proposed (client picks the vendor, based on bank relationship and rates) |
| D16 | **Security posture is adversarial.** We assume the attacker knows our stack and our rules. The attacker's main targets are fake money-triggering events (incentive or disbursal fraud), PII theft, account takeover, and impersonating Power Mart to customers. See [engineering/06-adversarial-security](../engineering/06-adversarial-security.md). | Agreed |
| D17 | **Tech stack:**<ul><li>a static public site (Astro);</li><li>a React SPA (single-page app) PWA for ops, dealer and admin users, with no frontend server;</li><li>a NestJS (Fastify) API;</li><li>PostgreSQL with row-level security;</li><li>AWS Mumbai, with backups in Hyderabad;</li><li>Cognito for identity.</li></ul>See [ADR-0001](../adr/0001-architecture-and-stack.md). | Proposed (team sign-off pending) |

## 2026-10-13 — Session 5: Dealer role and stack challenge

| # | Decision | Status |
|---|---|---|
| D18 | **Optimise for the best long-term stack, not the fastest delivery** (Pawan). The first hubs run on the website plus a strict Excel template (imported later), and the client is told this upfront. | Agreed |
| D19 | **Next.js rejected, on the merits.** Claude had briefly agreed to Next.js static export without defending the original reasoning; that agreement is withdrawn. Reasons are in ADR-0001 r3: no frontend server by design, Next's strengths are the features we forbid, static export friction, JS weight on low-end phones. Public site: Astro. Web app: React + Vite SPA. | Proposed |
| D20 | **Backend: Java (LTS) + Spring Boot + Spring Modulith**, replacing NestJS. The decision changed because the time constraint was removed, not because of pushback. OpenAPI-first with generated TS clients. Python only for later analytics. | Proposed (awaiting Pawan's sign-off) |
| D21 | **Dealer and field app: native React Native (Expo) Android** via Managed Google Play, for Play Integrity, mock-GPS detection, scanning, and an encrypted offline queue (anti-fraud T1). HO/hub/admin stays a web SPA. | Proposed |
| D22 | **The platform is dealer-centric.** Dealers run the full sale, including the finance step. Manual financier mode is a day-one fallback only; the target is Link or API. Manual approvals and disbursals get an HO check, because dealers have an incentive motive. | Agreed |
| D23 | **Dealer × financier empanelment.** A dealer can offer a financier's scheme only when that financier has empanelled the dealer (status + dealer code tracked). | Agreed |

## Open items (owner → due)
- [ ] Pawan: sign off on ADR-0001 r3 (Java/Spring backend, native dealer app)
- [ ] Client: are sales mostly through dealers, or also direct at hubs? This sets R0 priority
- [ ] Client: list of products × current financier × how each financier works today (app, portal, paper) → next meeting
- [ ] Client: who collects the down payment (Power Mart or the financier), and who receives the disbursal for dealer sales (Power Mart or the dealer)
- [ ] Client: incentive slab rules (retrospective or marginal, the counting event, clawback)
- [ ] Client: domain, brand assets, WhatsApp Business account, SMS DLT registration, payment gateway account, cloud account in their name
- [ ] Pawan: get a contact person at each financier, and ask whether they have an API or file exchange and what their data-security vendor checklist requires
- [ ] Claude: clickable Hindi prototype of the hub sales flow plus the financier and scheme admin screens, for the next client meeting
- [ ] Team: sign off on ADR-0001, then initialise the repository and CI
