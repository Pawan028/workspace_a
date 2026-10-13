# 05 — Development Workflow, Code Review, CI/CD, Release

## 1. Planning
- Run 2-week sprints. Each story has acceptance criteria written as Given/When/Then, with Hindi copy included where the story is user-facing.
- Definition of Ready: acceptance criteria agreed, designs or wireframes linked, partner dependencies identified, data fields mapped to the data inventory.
- A spike or ADR comes before building anything that touches money, a partner integration, or personal data in a new way.

## 2. Branching and commits
- Trunk-based development. `main` is always releasable. Short-lived branches (≤ 2 days) named `feat/…`, `fix/…`, `chore/…`, `docs/…`.
- Conventional Commits (`feat(battery): add refurbish transition`). Commits are small and focused.
- No direct pushes to `main`. Force-push to `main` is disabled. Signed commits are preferred.

## 3. Pull requests
- A PR should be at most about 400 changed lines of non-generated code. Larger changes are split.
- The PR template covers:
  - what and why;
  - a link to the story or ADR;
  - screenshots in **Hindi** for UI changes;
  - migration notes;
  - feature flag;
  - a test evidence checklist;
  - a privacy checklist (new PII? purpose? retention?);
  - a security checklist (authZ scoping? input validation?).
- Review rules:
  - One approval at minimum.
  - **Two approvals** for changes to `incentives`, `sales`, `recon`, `identity`, migrations, auth, or `integrations/*`.
  - The author never merges their own change without review.
- Reviewers check correctness, invariants, tests, authZ scope, PII and logging, and readability, in that order. Style is the linter's job.

## 4. CI pipeline (each PR)
1. Install with the lockfile, typecheck, lint (including boundary and PII rules), format check.
2. Unit, property, integration (Testcontainers), API, and contract tests, with coverage floors.
3. Build, Lighthouse CI on the public pages, axe.
4. SAST, dependency, secret, and IaC scans.
5. Migration dry-run against a copy of the schema.

Every step must pass. No `skip-ci` on `main`.

## 5. CD and release
- Merge to `main` triggers an automatic deploy to `dev`. A release-candidate tag deploys to `staging` and runs E2E, DAST, and UAT. A manual approval by the tech lead plus the client product owner then deploys to `prod`.
- Deploys are zero-downtime (rolling or blue/green). Migrations are backward-compatible with the previous app version.
- Feature flags decouple deploy from release. Rollouts go per state and per hub.
- Rollback: redeploy the previous image within 15 minutes. A data rollback goes through a forward-fix migration, never a manual DB edit.
- Release notes in plain English (and Hindi for the field team) for every production release.
- No production deploys on Fridays after 3 pm IST or on the day before a hub launch, except for hotfixes.

## 6. Definition of Done (each story)
- [ ] Acceptance criteria met and demoed
- [ ] Tests at the right layer, including the mandatory cases (doc 03 §4)
- [ ] AuthZ row scoping tested
- [ ] No PII in logs (redaction test passes)
- [ ] i18n complete (`hi` + `en`) and verified on a low-end device profile
- [ ] Accessibility checks pass
- [ ] Observability: metrics, logs, and an alert where it is a critical flow
- [ ] Docs: module README, ADR if needed, data inventory updated if PII changed
- [ ] Feature flag configured, release note drafted

## 7. Production access and data
- Engineers have **no standing production DB write access**. Data fixes go through reviewed scripts run by the pipeline, with audit.
- Production read access goes through a masked read replica or views. PII queries are logged.

## 8. Working with AI assistants
- AI-generated code follows every rule here and is reviewed like human code. The author owns it.
- Never paste production data, secrets, or customer PII into any AI tool or external service.
- `CLAUDE.md` at the repo root summarises these rules for assistants. Keep it in sync.

## 9. Communication cadence
- A daily stand-up (internal) and a weekly client demo with decision log updates.
- A risk register review every sprint (docs/discovery/02).
- Partner integration syncs are weekly until each integration is live.
