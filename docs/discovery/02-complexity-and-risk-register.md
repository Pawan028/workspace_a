# 02 — Complexity and Risk Register

> How to read this: each item says why it is hard, what it does to the design, and how we mitigate it. Severity is H, M, or L, judged for the **Mar 2027 horizon**.
> Prepared: 2026-10-13. Review every sprint, add new risks, and close with evidence.

> **Update (2026-10-13):** decisions D2, D8–D15 (merchant-only model, financier plug-ins, hosted payment gateway) **lower the severity of A1, A2 and B1–B2 from H to M**. Power Mart does not collect borrower KYC, and every financier starts in Manual mode. A2 now reads: Power Mart receives only sale proceeds (down payment, disbursal as seller), never repayments.

## A. Regulatory perimeter (Severity: H)

| # | Complexity | Design consequence | Mitigation |
|---|---|---|---|
| A1 | **Power Mart's role in lending is undefined.** Collecting KYC or loan documents on the NBFC's behalf, or earning a sourcing fee, makes Power Mart (or its tech) an LSP or DLA under the RBI Digital Lending Directions 2025. | Data minimisation, storage in India, consent at every stage, a listed DLA, an RE–LSP agreement, and NBFC due diligence of **our** code and infra (expect a VAPT request). | Preferred: the LSP's own app or SDK captures KYC and we keep only a reference ID and status. If we must capture KYC, design to LSP-grade controls from day one (doc 04 of engineering). |
| A2 | **Money flow.** Repayments must go directly into the RE's account with no pass-through. Disbursal goes to the borrower's account or, for a specific end-use, directly to the end-beneficiary (the seller). | The platform never holds borrower money. The daily QR is the NBFC's QR. We only **read** repayment status. | Our only own money flows are dealer security deposits and Power Mart's own sales receipts, held in a separate PG merchant account. |
| A3 | **Remote immobilisation of the battery** (deck: "remote on/off"). The RBI device-lock bar takes effect 1 Jan 2027. EV coverage is unclear. Cutting power in motion is a safety hazard. Reputational risk. | Must never be a default or automated action. | Telemetry ships; the immobilise command is not built until (1) a written NBFC and legal opinion exists, (2) a firmware interlock allows action only when the vehicle is stationary and at power-up, (3) there is advance notice to the borrower, (4) there is a fast reversal SLA, and (5) every action is fully audit-logged and attributed to a named NBFC officer. |
| A4 | **Recovery conduct** (from 1 Jan 2027). If Power Mart staff follow up on overdues, they fall under recovery-agent norms: certification, calling hours, recording retention. | The CRM must enforce contact windows, log every contact, and store recordings with retention. | Business decision: collections stay NBFC-only (deck slide 14) unless the client opts in. |
| A5 | **Co-lending** (2025 Directions, effective 1 Jan 2026). A blended rate, an ex-ante agreement, and irrevocable partner commitment. | KFS, APR, and the EMI schedule come from the NBFC. We display, we do not compute. | The single source of truth for pricing is the NBFC API. Show "rate as per lender" until it is integrated. |
| A6 | **DPDP Act and Rules 2025.** Core consent, notice, breach, and security duties apply from about May 2027. Consent-manager provisions apply from about Nov 2026. | Consent ledger, purpose limitation, retention and erasure jobs, breach runbook. | Build to DPDP from sprint 1. Retrofitting consent is the expensive path. |
| A7 | **Aadhaar handling.** Storing full Aadhaar numbers or images without a UIDAI-compliant vault is prohibited. | Never store the full number. Mask to the last 4. Prefer DigiLocker or offline e-KYC references. | Lint rule plus a DB check constraint against 12-digit patterns in free-text columns (see coding standards). |
| A8 | **CERT-In directions (2022).** Report incidents within 6 hours, keep logs for 180 days in India, sync clocks to NTP. | Logging and retention design, plus an incident runbook. | Infra checklist item; verified at go-live. |
| A9 | **Battery identity regulation.** The MoRTH Battery Pack Aadhaar (BPAN) draft proposes a 21-character ID, with a new BPAN on repurposing. | The battery entity must support multiple identifiers and lineage (refurbished packs get a new ID). | Model `battery_identity` as a separate, versioned table now. |
| A10 | **GST and TDS.** Dealer sales (B2B, e-invoicing at CLN's turnover), incentives (credit notes?), referral payouts (TDS u/s 194H?). | The incentive and payout engine must produce tax-correct documents, or hand off to the ERP. | Client's finance team decides. We integrate with the ERP rather than re-implement tax. |

## B. Partner integration risk (Severity: H)

| # | Complexity | Mitigation |
|---|---|---|
| B1 | LSP and NBFC are **not named**. Small NBFCs often have no APIs, only Excel or SFTP drops and email. | Build an **integration adapter port** with three adapters: API, file (SFTP/CSV), and manual upload with maker-checker. The domain code never depends on a specific partner. |
| B2 | Multiple NBFCs later (co-lending, then more partners). | Lender is a dimension on every loan record from day one. No single-lender assumptions. |
| B3 | IoT/BMS vendor unknown. Telemetry may sit in a vendor cloud. | Ingest only what we need (SoC, SoH, cycles, last location, alerts) via a webhook or pull adapter. Don't build a time-series platform before it's needed. |
| B4 | Partner status events arrive late, duplicated, or out of order. | Idempotent handlers, event versioning, daily reconciliation jobs with an exceptions queue. |
| B5 | CLN ERP unknown (stock, invoices, warranty). | Same adapter pattern. Decide which system is the master for each data element (see the matrix below). |

**System-of-record matrix (draft, to be confirmed in the meeting):**

| Data | Master | Others hold |
|---|---|---|
| Lead, referral, customer profile (non-KYC) | Power Mart | — |
| KYC documents | LSP or NBFC | reference ID only |
| Loan sanction, KFS, schedule, repayments, DPD | NBFC | read-only mirror |
| Battery serial, BPAN, lifecycle state, location | Power Mart (lifecycle), OEM (manufacture) | — |
| Stock and invoices, GST | CLN ERP _[CONFIRM]_ | mirror |
| Telemetry | IoT platform | derived summaries |
| Dealer, hub, incentives | Power Mart | ERP for payout accounting |

## C. Domain correctness (Severity: H)

- **C1 — One battery, many parties.** The battery's serial number is the thread joining OEM dispatch, hub stock, dealer stock, the sale, the loan, fitment, telemetry, service, repossession, refurbishment, and redeployment. A battery swap, mismatch, or duplicate breaks the reconciliation of money and assets. **Mitigation:**
  - an explicit battery state machine;
  - scan-based capture (QR or barcode), with no typing of serials;
  - a fitment photo with the serial visible;
  - a unique constraint plus an audit trail.
- **C2 — Reconciliation.** Sales vs disbursals vs fitments vs incentives vs collections must tie out daily. **Mitigation:** a reconciliation module as a first-class feature, not a report.
- **C3 — Explicit state machines** for Lead, LoanApplication (mirror), Battery, Dealer, Hub, ServiceTicket, IncentiveStatement. Illegal transitions are rejected in the domain layer.
- **C4 — Money.** Integer paise, INR only, explicit rounding rules, golden tests for slabs and tenures.
- **C5 — Time.** Daily repayments imply "day" = IST calendar day. Store UTC, compute business dates in Asia/Kolkata.

## D. Field reality (Severity: M–H)

- **D1 — Connectivity** in tier-3 towns is patchy. Hub and FOS flows must tolerate offline use: queue, retry, and conflict rules.
- **D2 — Devices.** Low-end Android and shared phones. Performance budget enforced in CI (doc 03 of engineering).
- **D3 — Language and literacy.** Hindi-first UI with icon-supported actions, voice and IVR, or WhatsApp for customers. Devanagari names in data; no ASCII assumptions.
- **D4 — Trust and cash culture.** Customers and dealers will call the hub rather than use a portal. WhatsApp-first notifications on the Business API, with approved templates. SMS needs DLT registration.

## E. Fraud and abuse (Severity: M)

| Vector | Control |
|---|---|
| Fake or duplicate leads to farm referral payouts | Dedupe on phone and vehicle registration, payout only after fitment plus N EMIs paid _[CONFIRM]_ |
| Dealer–customer collusion or ghost sales for slab incentives | Count sales on disbursal + fitment photo + IoT activation. Clawback rule |
| Battery swap or serial tampering | Serial scan at every handover, IoT device bound to pack serial, alert on mismatch |
| Identity fraud and first-payment default | Owned by NBFC underwriting. We surface signals (duplicate phone, device, address) |
| Insider data leakage | RBAC plus row-level scoping (a dealer sees only its own customers), PII access logging |

## F. Delivery risk (Severity: H)

- **F1 — Timeline.** Today is 13 Oct 2026. UP Phase 1 (10 cities, 10 dealers) is **this month**. The first 4 hubs open in **Nov 2026**. A full platform will **not** be ready for the first hubs. **Mitigation:** a phased release plan (doc 04). Release 0 covers the minimum to operate cleanly and capture data correctly from day one, with SOP-backed manual steps where integrations aren't ready.
- **F2 — Scope creep from the roadmap** (alternate-data underwriting, inverter and vehicle finance). **Mitigation:** design extension points (product type, lender, state as dimensions) but build only what the current phase needs.
- **F3 — Data captured before the platform exists** (Excel in Oct and Nov). **Mitigation:** give the client a **strict Excel template now**, matching our schema, with an import tool in Release 0.
- **F4 — Key-person and partner dependency.** Every partner integration has a named owner on both sides and a sandbox date in the plan.

## G. Future-proofing levers (cheap now, expensive later)

- `state`/`region`, `lender_id`, and `product_type` on core entities.
- Multi-language content model (hi, en, then bho/mai if needed).
- Battery identity versioning (BPAN, refurbishment lineage).
- Event log (outbox) on all state changes. This enables analytics, partner webhooks, and audit without re-engineering.
