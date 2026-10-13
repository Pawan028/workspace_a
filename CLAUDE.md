# CLN Power Mart — Rules for assistants and engineers

Merchant platform for CLN Energy's lithium e-rickshaw batteries (UP and Bihar). Hubs and dealers sell. **Financiers (plug-ins, one or more per product) do all lending.** Power Mart is the seller only.

Start here: `README.md`. Decisions: `docs/discovery/05-discussion-log.md` and `docs/adr/`. Design: `docs/architecture/`. Rules: `docs/engineering/01..06`.

## Hard rules (never violate; to change one, write an ADR)
1. **Money boundary.** Power Mart receives only its own sale proceeds: the down payment, the financier's disbursal as seller, and dealer deposits. **Never EMIs or repayments.** Loan and repayment data is a read-only mirror from the financier.
2. **No borrower KYC in our system.** Financiers run KYC, credit, KFS, mandates, and collections in their own journeys. Share customer data with a financier only after consent naming that financier.
3. **Financiers are plug-ins** (registry + versioned schemes + connector port: manual, link, file, api). No financier-specific code outside `integrations/financiers/<name>`. Changing a financier never touches existing loans.
4. **Payments are hosted checkout or links only.** The server computes amounts. Success = signed webhook **+** server re-fetch. Never trust a redirect.
5. **Money is integer paise.** Time is stored in UTC; business dates are computed in Asia/Kolkata via the helper. Inject `Clock`.
6. **State changes go only through domain state machines.** Each is one transaction plus an audit event and an outbox event. Idempotency key on every mutation.
7. **Authorization is enforced in the service layer AND by Postgres RLS.** Never in edge or middleware alone. Every new endpoint gets authZ-matrix tests.
8. **Money-triggering statuses** (approved, disbursed, incentive, refund) need evidence + maker-checker (maker ≠ checker in code) + source re-verification.
9. **PII:**
   - Never store a full Aadhaar.
   - Encrypt phone numbers and keep an HMAC for lookup.
   - Mask PII in UI and logs, and never log PII.
   - No contacts, call-log, SMS, or media permissions.
   - Data stays in India only.
10. **No remote battery immobilisation code** until an accepted ADR records the legal opinion and financier sign-off.
11. **Stack (ADR-0001 r3, PROPOSED, check sign-off before coding):**
    - Astro static site;
    - React + Vite SPA (HO, hub, admin);
    - React Native (Expo) Android dealer app;
    - Java LTS + Spring Boot + Spring Modulith API;
    - PostgreSQL (RLS) + Flyway;
    - OpenAPI-first with generated TS clients;
    - Cognito;
    - AWS ap-south-1.

    No server-rendered frontend runtime (no Next.js).
12. **Supply chain:** pnpm frozen lockfile, dependency scripts blocked, packages aged at least 7 days, Actions pinned by SHA, no long-lived cloud keys.
13. **Hindi-first i18n** with no hard-coded strings. Every bug fix has a failing-first test. No production data or secrets in lower environments, fixtures, prompts, or logs.
14. **No building on assumptions.** [CONFIRM] items are asked, not guessed. The client is non-technical: we translate rules into simple choices.
