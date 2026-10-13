# 03 — Testing Standards

## 1. Philosophy
We test **behaviour and invariants**, not implementation. The riskiest code gets the most tests. In this project that means money, state machines, partner integrations, authorization scoping, and reconciliation.

## 2. The test pyramid for this project

| Layer | What | Tooling (proposed) | Gate |
|---|---|---|---|
| Unit | Domain: value objects, state machines, incentive slabs, tenure tables, date helpers | Vitest | Required in CI |
| Property-based | State machine (no illegal transition reachable), money arithmetic (no loss across split and round), dedupe rules | fast-check | Required for `domain/` of money and state modules |
| Integration | Use cases against a **real Postgres** (Testcontainers): transactions, constraints, idempotency, audit and outbox writes | Vitest + Testcontainers | Required |
| Contract | Each partner adapter against recorded and schema-validated payloads. Provider contracts where the partner allows it | Pact or JSON-schema fixtures | Required per adapter |
| API | HTTP layer: authN, authZ **row scoping**, validation errors, problem+json | Supertest | Required |
| E2E | Critical journeys in a browser, in Hindi | Playwright | Required on main before release |
| Non-functional | Performance budgets, accessibility, security scans | Lighthouse CI, axe, ZAP, SAST | Release gate |

## 3. Coverage rules
- Coverage is a signal, not a goal. Floors (CI fails below them): `domain/` **95% branches**, `application/` 85%, overall 75%.
- **Mutation testing** (Stryker) runs nightly on the `incentives`, `sales`, `battery`, and `recon` domain modules. A mutation score below 80% is a P2 issue.
- A bug fix **MUST** come with a test that fails before the fix.

## 4. Mandatory test cases (the minimum set; extend it, never drop it)
**Money and incentives**
- Golden tables for every slab boundary (4/5, 8/9 batteries a month), both the retrospective and marginal variants (until the client decides), month boundaries in IST, and clawback.
- Tenure table: 12/15/18 months with principal per month and per day exactly as on slide 11. The display must never show an EMI without the lender's KFS data.

**Battery lifecycle**
- Every legal transition succeeds and every illegal one is rejected.
- A serial cannot be in two places at once. A duplicate scan is idempotent.
- Refurbishment creates a new identity linked to its lineage.

**Integrations**
- The same webhook delivered twice gives one effect.
- Out-of-order events: an older `source_version` is ignored.
- Partner timeout or 5xx: retry, then circuit opens, then an alert. No partial domain state.
- Malformed payload: rejected and stored for triage, with no crash.
- Reconciliation detects each mismatch type (sale without disbursal, disbursal without fitment, fitment without sale).

**Security and authorization**
- A dealer A user cannot read dealer B's leads, customers, or stock (API test per resource).
- PII redaction: log snapshot tests assert that no phone, PAN, or Aadhaar pattern appears.
- Masked fields stay masked without the unmask permission. Each unmask is audit-logged.

**Field and offline**
- An offline-queued sale syncs exactly once after reconnecting. Conflict rules apply (stock already sold elsewhere leads to a clear user message).
- Hindi rendering: visual snapshot of key screens in `hi`, with long-string overflow checks.

**E2E critical journeys**
1. Lead (website) → hub follow-up → sale → loan file reference → down payment recorded → fitment with serial scan.
2. Dealer onboarding → deposit recorded → stock allocated → dealer sale → incentive statement.
3. Repossession → OEM → refurbish → back into stock (R2).

## 5. Test data
- Use synthetic data only, from factories with Indian-realistic names (Devanagari and Latin), PIN codes, and GSTIN formats with valid checksums.
- **Production data never enters lower environments.** If a production issue needs reproducing, anonymise first through an approved script.
- Partner fakes (`adapters/fake`) are maintained alongside the real adapters and used in E2E.

## 6. Performance and quality budgets (CI-enforced)
- Public site, mobile profile (Moto G-class device, slow 4G): LCP ≤ 2.5 s, INP ≤ 200 ms, CLS ≤ 0.1, initial JS ≤ 170 KB gzipped.
- Ops PWA: interactive in ≤ 4 s on slow 3G after the first load. Scan-to-save in ≤ 1 s with the network online.
- API: p95 ≤ 300 ms for reads and ≤ 600 ms for writes at 10× the expected peak.
- Accessibility: zero axe "serious" or "critical" violations on key pages.

## 7. Environments and test flow
- `local` (Docker Compose) → `dev` (each merge) → `staging` (release candidate, partner sandboxes, production-like config) → `prod`.
- Staging is the only environment connected to partner **sandboxes**. Production keys exist only in production.
- UAT with the client product owner happens on staging, with a signed UAT checklist per release.
- Before go-live: a **VAPT** by a CERT-In-empanelled auditor, with critical and high findings closed and the report shared with the NBFC.

## 8. Flaky tests
A flaky test is quarantined within 24 hours with an owner and an issue, then fixed or deleted within 5 working days. We never just re-run until green.
