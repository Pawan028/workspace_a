---
name: project-cln-power-mart
description: "CLN Power Mart client project — e-rickshaw battery merchant platform with multi-financier plug-ins; status, key decisions, where docs live"
metadata:
  node_type: memory
  type: project
  originSessionId: b27a5a67-d1bd-4d88-85eb-8c35849499ea
  modified: 2026-10-13T08:36:07.099Z
---

The client is CLN Power Mart, the retail arm of CLN Energy (Noida, BSE SME-listed). It sells a Rs 75k LFP e-rickshaw battery with a Rs 15k down payment and financier loans. Plan: 20 hubs and 200 dealers in UP and Bihar by Mar 2027. The first hubs open Nov 2026. Source deck: Downloads/CLN_Power_Mart_Deck_v2.pptx.

Team: Pawan plus Claude, two people.
- Pawan: client, financiers, field testing, accounts.
- Claude: most code, tests, docs.
- Together: architecture, review, releases.

Key decisions (2026-10-13), full log in docs/discovery/05-discussion-log.md:
- The client is non-technical. We own the tech and regulatory thinking.
- Power Mart is **merchant only**. Financiers do KYC and lending.
- Power Mart receives only sale proceeds (down payment, disbursal as seller), never EMIs.
- The client said **each product has its own financier, and financiers can change**. We designed financier plug-ins: a registry, versioned schemes, a standard status language, and Manual/Link/File/API connectors. Every financier starts Manual-first, with consent per named financier and a sequential waterfall on rejection (ADR-0002, docs/architecture/02).
- Payments use Cashfree or Razorpay hosted checkout only (ADR-0003).
- No remote battery immobilisation.
- Stack (ADR-0001 r3, proposed, awaiting Pawan's sign-off): Astro static site, React+Vite SPA for HO and admin, a React Native (Expo) Android dealer app (Play Integrity), Java + Spring Boot + Spring Modulith backend, Postgres with RLS, OpenAPI-first, Cognito, AWS Mumbai. Next.js is rejected on the merits. Pawan chose "best stack, even if it takes longer" (D18).
- The platform is dealer-centric. Dealer × financier empanelment is tracked (D22–D23).
- Adversarial security rules are in docs/engineering/06.
- Releases: R0 launch-ready (about 4–5 weeks from mid-Oct), R1 Dec–Jan, R2 Feb–Mar.

**Why:** The deck is a business pitch from a non-technical client. Regulatory traps (RBI DL 2025, the device-lock rules from 1 Jan 2027, DPDP from about May 2027) must never block sales.

**How to apply:**
- Read CLAUDE.md and the discussion log before any work.
- The open items at the bottom of the log were pending as of 2026-10-13. Ask whether they're answered rather than assuming.

See [[user-working-style]].
