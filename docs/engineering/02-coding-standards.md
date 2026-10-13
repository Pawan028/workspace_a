# 02 — Coding Standards

> Written for the proposed TypeScript stack (ADR-0001, pending). If the stack changes, the **intent** of each rule stays and its tooling section is rewritten.

## 1. Language and compiler
- TypeScript `strict: true`, plus `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, and `noImplicitOverride`.
- `any` is banned. `unknown` plus narrowing at boundaries is allowed. `// @ts-expect-error` needs a linked issue.
- Use ESM and Node LTS only. Pin the Node version in `.nvmrc` and `engines`.

## 2. Formatting and linting (enforced in pre-commit and CI)
- Prettier is the formatter; formatting is not debated in review.
- ESLint uses typescript-eslint (type-checked), import ordering, `no-floating-promises`, `eqeqeq`, `no-console` (use the logger), and boundary rules (`eslint-plugin-boundaries` or similar) so modules cannot import each other's internals.
- Custom lint rules:
  - No `number` type for anything named `*Amount|*Price|*Paise`. Use the `Paise` branded type.
  - No `new Date()` in domain code. Inject a `Clock`.
  - No raw `fetch` or `axios` outside `integrations/*`.
  - No string literal that matches a 12-digit Aadhaar pattern, a PAN pattern, or a phone number in non-test code or logs.

## 3. Structure
```
apps/
  web/            # public site + ops + dealer + admin (route groups), or split per audience
  api/            # backend modular monolith
packages/
  domain-types/   # shared Zod schemas & branded types (Paise, BatterySerial, PhoneE164)
  ui/             # design system (Hindi-ready components)
  config/         # eslint/tsconfig presets
modules (inside api/src/modules/<name>/):
  domain/         # entities, value objects, state machines — no framework imports
  application/    # use cases (commands/queries), transactions, ports
  infrastructure/ # repositories, adapters
  interface/      # HTTP controllers, DTOs, webhook handlers
  index.ts        # the ONLY public export of the module
```

## 4. Naming
- Files use `kebab-case.ts`. Types and classes use `PascalCase`. Functions and variables use `camelCase`. Constants use `SCREAMING_SNAKE`. DB tables and columns use `snake_case`, with plural table names.
- Use domain language from the deck consistently: `Hub`, `Dealer`, `ReferencePartner`, `Fitment`, `DownPayment`, `IncentiveSlab`, `Lender`. Glossary: `docs/glossary.md`.
- Booleans start with `is/has/can`. Money values end with `Paise`. Timestamps end with `At`. Business dates end with `Date`.

## 5. Domain modelling rules
- Use value objects for: `Paise`, `BatterySerial`, `Bpan`, `PhoneE164`, `Gstin`, `Pan(masked)`, `VehicleRegNo`, `PinCode`. Each one validates in its constructor.
- State machines are declared as data (states, events, guards) and unit-tested exhaustively. Hand-rolled `if (status === ...)` chains in controllers are forbidden.
- Commands return a result type (`Ok | Err<DomainError>`). Domain errors are typed and carry a stable `code` (used for i18n messages). Exceptions are only for programmer errors and infrastructure failures.
- Every write use case:
  1. takes an `idempotencyKey`;
  2. runs in one transaction;
  3. writes an `audit_event`;
  4. enqueues `outbox_event`s.

## 6. API design
- REST and JSON, versioned (`/v1`). Resource names are nouns. OpenAPI is generated from Zod schemas and committed. Breaking changes need a new version.
- Mutations accept the `Idempotency-Key` header.
- Pagination is cursor-based. Filters are explicit and allow-listed.
- Errors follow RFC 9457 problem+json with a stable `code`. Stack traces and internal IDs never leak.
- AuthZ is checked in the application layer by **policy functions** (`can(actor, action, resource)`) with row scoping. A dealer sees only its own records; a hub user sees only their hub. Checking only at the controller is not enough.

## 7. Database
- Every schema change is a migration in the repo. Migrations are reviewed like code and are **forward-only**. Destructive changes follow expand → migrate → contract across releases.
- Every table has `id` (UUIDv7), `created_at`, `updated_at`, and `created_by` where applicable. Soft delete only where retention law needs it. Otherwise follow the retention jobs.
- PII columns are tagged in the data inventory. Highly sensitive fields (bank account numbers, document numbers) use application-level encryption with KMS envelope keys.
- No business logic in triggers, except audit and immutability guards.
- Every query on a hot path has an index. `EXPLAIN` is attached to the PR for new heavy queries.

## 8. Frontend
- Server components or SSR for public pages. Client JS only where interaction requires it.
- Every string goes through i18n (`hi` is the default, `en` is secondary). Never concatenate translated fragments; use ICU messages. Money is formatted with `Intl.NumberFormat('hi-IN' | 'en-IN')` using lakh/crore grouping.
- Forms use shared Zod schemas, show inline errors in Hindi, and hold large tap targets (≥ 48px). Every icon has a text label.
- Accessibility target: WCAG 2.2 AA. Use semantic HTML, label every input, and make focus visible.
- Images are optimised and lazy-loaded. No layout shift. Fonts are subset for Devanagari.

## 9. Error handling and logging
- Use one logger (pino or similar) with redaction paths configured centrally (phone, PAN, Aadhaar, bank, address, name in partner payloads).
- Log at the boundary, not at every layer. Each log line has a correlation ID.
- User-facing errors are actionable, in Hindi and English, and never expose internals.

## 10. Dependencies
- A new dependency needs a justification in the PR: maintenance status, licence (no AGPL in shipped code without approval), size, and security history.
- The lockfile is committed. Renovate or Dependabot runs weekly. Critical CVEs are patched within 72 hours.

## 11. Comments and docs
- Code says *what*; comments say *why*, and only where it is non-obvious (a regulatory reason, a partner quirk, a business rule source such as "slide 16 slab table, config v3").
- Every module has a short `README.md` covering its purpose, public API, state machine diagram, and owned tables.
- Every partner adapter documents the partner's quirks and the contact person.
