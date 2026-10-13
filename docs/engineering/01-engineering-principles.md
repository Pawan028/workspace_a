# 01 — Engineering Principles (non-negotiable)

These rules apply to every engineer and every AI assistant working on this repository. "MUST" means a PR that violates the rule is not merged. To change a rule, write an ADR.

## P1. Know which system owns the truth
- **MUST** respect the system-of-record matrix (docs/discovery/02 §B). Loan, KFS, repayment, and DPD data from the NBFC are **mirrors**. We never compute, edit, or "correct" them locally.
- **MUST NOT** route loan **repayments (EMIs)** through any Power Mart account, wallet, or virtual account. Power Mart receives only its own sale proceeds: the down payment, the financier's disbursal as seller, and dealer deposits (ADR-0002, ADR-0003).
- Financiers are plug-ins (registry + versioned schemes + connector port). No financier-specific logic outside `integrations/financiers/<name>`.
- Each mirrored record stores `source`, `external_ref`, `received_at`, and `source_version`.

## P2. Correctness over cleverness
- Domain state changes happen **only** through explicit state machines in the domain layer. Each transition is validated, persisted, and audit-logged in one DB transaction.
- Invariants live in the database as well as in code: `NOT NULL`, `UNIQUE` (battery serial, BPAN, dealer GSTIN), `CHECK` (amounts ≥ 0), and foreign keys. Code validation alone is never enough.
- Money is integer **paise** (`bigint`). No floats anywhere in the money path. All rounding is explicit and named (`roundHalfEvenToRupee`).
- Time is stored in UTC (`timestamptz`). Business dates (EMI day, incentive month) are computed in `Asia/Kolkata` through one shared helper.

## P3. Modular monolith with hard boundaries
- Modules are `identity`, `leads`, `customers`, `battery`, `channel`, `sales`, `loanfile`, `incentives`, `service`, `recon`, `reporting`, and `integrations/*`.
- A module exposes only its public service interface. **No cross-module imports of internals and no cross-schema joins in module code.** Reporting uses read models or views.
- Every external partner sits behind a **port** (interface) with swappable adapters: `api`, `file`, `manual`, `fake`. Domain code never imports a vendor SDK.

## P4. Integrations are hostile by default
- Every inbound webhook or file is:
  1. authenticated (signature or mTLS or IP allowlist);
  2. schema-validated;
  3. stored raw (encrypted) before processing;
  4. processed **idempotently** by `(source, event_id)`.
- Every outbound call has a timeout, retries with exponential backoff and jitter, a circuit breaker, and a correlation ID.
- Out-of-order events are handled with `source_version` / `occurred_at`. Stale updates never overwrite newer state.
- A daily reconciliation job runs per partner. Mismatches go to an exceptions queue with a human owner. They are never silently "fixed".
- Outbound events go through a **transactional outbox**. No dual writes.

## P5. Privacy and compliance by design
- Collect the minimum. Every personal-data field has a documented purpose, a lawful basis (consent ID), and a retention period in `docs/data/data-inventory.md`.
- Aadhaar is never stored in full. PAN, phone, and bank account numbers are masked in UI and logs by default. Unmasking is a permission and is logged.
- All data and backups stay in Indian regions. No third-party SaaS that processes PII outside India without an ADR plus legal sign-off.
- Phone permissions: never request contacts, call logs, SMS, or media access. Camera and location only at the point of use, with an explanation, and with consent recorded.
- No remote-immobilisation code path exists in the codebase until an ADR records the legal opinion and NBFC sign-off (see risk A3).

## P6. Built for the field
- Hindi-first. Every user-facing string is externalised, with no hard-coded copy. Devanagari is tested in every view.
- Performance budgets are enforced in CI (doc 03). Pages must work on a low-end Android over slow 3G.
- Ops and hub flows tolerate going offline: queue writes locally, sync with idempotency keys, and resolve conflicts with defined rules. Never lose a scan.
- Serial numbers are **scanned**, never typed. Manual entry is a fallback that requires a reason and a photo.

## P7. Configuration, not code, for business variables
- State, cluster, city, hub, dealer slabs, tenures, price, down payment, and incentive rules are **versioned configuration** with an effective-from date and maker-checker approval. A historical calculation always uses the config version that was in effect at the time.
- Feature flags gate releases per state and per hub.

## P8. Observable and operable
- Structured JSON logs with `correlation_id`, `actor_id`, `module`, and **no PII** (a redaction middleware is mandatory).
- Every critical flow emits metrics (lead created, sale, fitment, partner sync lag, recon mismatches) and has an alert.
- Log retention is ≥ 180 days in India (CERT-In). Server clocks are NTP-synced.
- Runbooks exist for partner outage, data breach (6-hour CERT-In reporting, DPDP Board notification), and restore from backup.

## P9. Simple first, extensible by design
- Build for the current release. Leave seams for known roadmap items: `region`, `lender_id`, and `product_type` on core entities, plus identity versioning for batteries. Do not build speculative features.
- Choose boring, proven technology. Adding any new runtime dependency or infrastructure component needs a short justification in the PR. Anything architectural needs an ADR.

## P10. Decisions are written down
- Architecture and policy decisions are recorded as ADRs in `docs/adr/` (template provided). Superseded ADRs are kept and marked as superseded.
- Assumptions are recorded as assumptions, with an owner and a date to verify. We do not build on unverified assumptions silently.
