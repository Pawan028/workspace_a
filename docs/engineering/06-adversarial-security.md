# 06 — Adversarial Security Rules ("assume they know our rules")

> Doc 04 is the baseline (OWASP ASVS 5.0 Level 2, the OWASP Top 10 2025, and the regulatory controls). The baseline is public, so attackers know it too. **This doc is about beating someone who has read our docs, knows our stack, and is patient.** Every rule here is a MUST unless an ADR says otherwise.

## 0. Mindset
1. **Assume the attacker has our source code, docs, and stack list.** Security must not depend on secrecy of design. Only keys are secret.
2. **Assume breach.** Every layer is designed as if the layer in front of it has already failed (defense in depth). No single control protects money-triggering events or PII.
3. **Business logic is the real attack surface.** Scanners find XSS. They don't find "a dealer marks 9 fake sales on the 30th to hit the top incentive slab". We threat-model the business, not just the HTTP layer.
4. **Insiders and partners are part of the threat model.** Hub staff, dealers, FOS, our own admins, and compromised partner systems are all included.
5. **Fail closed.** Any error in authN, authZ, signature verification, or reconciliation means *deny or hold*, never *allow*.

## 1. What attackers want from this system (ranked)

| # | Attacker goal | Who | Impact |
|---|---|---|---|
| T1 | **Fake money-triggering events**: fake sale, approval, or disbursal to earn incentives or referral fees; ghost dealers | Dealers, referrers, insiders | Direct financial loss |
| T2 | **Steal PII** of borrowers and dealers (phone, address, loan status) and sell it to loan sharks or scammers | External attackers, insiders | Customer harm, DPDP penalties, client reputation |
| T3 | **Impersonate Power Mart or the financier** to customers ("pay your EMI to this UPI") | Scammers using leaked data | Customers lose money, trust destroyed |
| T4 | **Account takeover** of a dealer, hub, or admin account | External (OTP abuse, phishing, SIM swap) | Enables T1 and T2 |
| T5 | **Supply-chain or CI compromise**: malicious npm package, stolen deploy keys | External | Full compromise |
| T6 | **Destroy or ransom data** | External, disgruntled insider | Business stop |
| T7 | **Abuse public endpoints**: OTP or SMS bombing (costs money), lead-form spam, scraping, DoS on launch days | Bots, competitors | Cost and outage |
| T8 | **Deface the public site or inject SEO spam** | Opportunists | Reputation |

## 2. Attack → control matrix (beyond the baseline)

### Access control (OWASP A01: the most exploited class)
- **IDOR / BOLA.** UUIDs are *not* a control.
  - Every endpoint enforces policy-based row scope in the application layer.
  - **PostgreSQL row-level security (RLS)** is a second, independent layer. The API sets `app.actor_id` and `app.scope` per transaction, and RLS policies filter rows even if app code forgets.
  - CI runs an **auto-generated authZ matrix test**: every endpoint × every role × an own vs other's resource. A new endpoint without matrix coverage fails the build.
- **Mass assignment.** Zod schemas are `.strict()`, and DTOs are allowlisted. Fields like `status`, `scheme_id`, `amount`, `dealer_id`, and `role` are never accepted from the client where the server should derive them.
- **Privilege escalation.** Role changes need an admin with step-up MFA plus a second approver. Admins cannot raise their own role.
- **Admin console is not on the open internet.** It sits behind staff SSO + MFA + zero-trust access (e.g. AWS Verified Access or Cloudflare Access) and lives on a separate subdomain.

### Money-triggering events (T1)
- **Server derives all amounts** from versioned config: price, down payment, expected disbursal, incentive.
- **Race conditions.** Two concurrent sales of the same battery, or a double incentive payout, are prevented by a DB `UNIQUE` constraint on (battery, active assignment) and on (statement, dealer, month), together with `SELECT … FOR UPDATE` on state transitions and idempotency keys. We load-test races with parallel requests in CI.
- **Evidence chain for a sale:** an approved finance case, the payment-gateway-verified down payment, an **in-app camera** fitment photo, the serial scan, and the financier's disbursal plus a reconciled UTR.
  - The photo cannot be uploaded from the gallery.
  - The photo carries the device time and GPS, which must fall within X km of the hub or dealer.
  - We store a perceptual hash of each photo to detect reused photos.
- **Maker-checker** on: manual financier status updates (`APPROVED`, `DISBURSED`), incentive statements, refunds, dealer activation, scheme or config changes, and role changes. Maker ≠ checker is enforced by code, not by policy text.
- **Fraud analytics (rules first, R1):**
  - month-end slab gaming (sales clustered just over a slab boundary);
  - a referrer's phone or device matching the customer's;
  - one device used for many customers;
  - a fitment GPS far from the hub;
  - a sale-to-default pattern by dealer.
  Alerts go to HO and can auto-hold payouts.
- **Clawback hooks** exist in the incentive engine even before the client finalises the rules.

### Integrations and webhooks
- Inbound webhooks are processed in this order:
  1. IP allowlist (where the partner publishes ranges).
  2. HMAC signature over the **raw body**, with a constant-time compare.
  3. A timestamp window (≤ 5 min).
  4. Event-ID dedupe (anti-replay).
  5. Schema validation.
  6. **For money-relevant events (payment captured, disbursed), re-fetch the truth from the partner's API before acting.** A valid-looking webhook alone never moves money state.
- Webhook secrets are per partner and per environment, and rotated. The webhook endpoint path does not reveal which partner it serves.
- **SSRF.** The server never fetches a user-supplied URL. Outbound traffic goes through an **egress allowlist** (financier, payment gateway, WhatsApp, and SMS domains only).
- **Partner compromise.** Treat partner data as untrusted input: validate, size-limit, and never render partner-supplied HTML.

### Authentication (T4)
- **OTP:**
  - 6 digits, valid for 5 minutes, at most 5 attempts, and stored hashed.
  - Rate limits apply per phone, per IP, and per device fingerprint, with exponential backoff.
  - A challenge appears after N requests.
  - The response is identical for a known or unknown number (no enumeration).
  - Daily SMS spend cap with an alert (stops OTP bombing from costing money).
- **SIM-swap resistance.** Staff and admin accounts use **passkeys or TOTP, never SMS-only**. Dealers' high-risk actions (bank account change, payout details) need re-verification plus a cooling period plus a notification to the old contact.
- **Sessions:**
  - httpOnly, Secure, SameSite=Strict cookies.
  - Session ID rotated at login.
  - Idle timeout of 15 min for admin and 8 h for the field app, with an absolute lifetime limit.
  - Server-side revocation (logout everywhere, device lost).
- **No tokens in localStorage.** If we use JWTs, the algorithm is pinned, they are short-lived, and audience and issuer are checked.
- **Device binding** for dealer and field accounts. A new device triggers a notification plus approval.

### Framework-specific traps (we name them so nobody repeats them)
- **Never rely on edge or middleware for auth.** The 2025 Next.js middleware-bypass CVE (CVE-2025-29927) let attackers skip middleware with one header. Our authZ lives in the API service layer and in DB RLS.
- **No server-executed frontend.** The React Server Components RCE in Dec 2025 (CVE-2025-55182, CVSS 10, exploited in the wild) hit server-rendered React. Our public site is static and our app is a SPA, so **there is no frontend server to attack.** That is a deliberate stack decision (ADR-0001).
- **Patch SLA:**
  - critical internet-facing CVE: **24 h**;
  - high: 7 days;
  - medium: 30 days.
  We subscribe to security advisories for every direct dependency.

### Supply chain and CI/CD (OWASP A03, A08; T5)
- pnpm with a **committed lockfile**, `--frozen-lockfile` in CI, and **dependency lifecycle scripts blocked by default** (allowlist only).
- New package versions must age before adoption (minimum release age about 7 days, via Renovate or pnpm settings). This defends against worm-style npm compromises (e.g. the 2025 self-replicating npm worm campaigns).
- **Pin GitHub Actions by commit SHA.** No third-party action without review.
- Workflows from forks run without secrets. Deploy jobs need environment approval.
- **OIDC to AWS** with narrowly scoped roles. No long-lived cloud keys anywhere.
- An SBOM and build provenance are generated per release. Container images come from minimal bases, are scanned (Trivy), run as non-root, and use a read-only root filesystem.
- **Honeytokens:** canary AWS keys and DB credentials planted where an attacker would look (old config files, docs). Any use triggers an immediate alert.

### Data theft (T2)
- **Minimise:** no borrower KYC documents in our system at all. The financier holds them.
- Phone numbers are stored encrypted, plus an **HMAC (with a KMS-held key) for dedupe and search**, so a DB dump alone doesn't reveal phone numbers.
- Field-level encryption (KMS envelope) for bank account numbers and any document number.
- **Bulk-export control.** Exports need a permission, are row-limited, watermarked (user + time), logged, and alerted when large. API pagination caps stop scraping via the API.
- **Spreadsheet formula injection.** Escape cells starting with `= + - @` in every CSV or XLSX export. Imports accept `.xlsx` and `.csv` only, with no macros, size caps, and formula stripping.
- **File uploads:**
  - Allowlist JPEG, PNG, and PDF, checked by magic bytes. **No SVG.**
  - Images are **re-encoded** server-side, which strips EXIF after we extract the GPS and time we need.
  - PDFs are AV-scanned.
  - Size and page caps (stops zip and PDF bombs).
  - Files are served from a **separate cookieless domain** with `Content-Disposition: attachment`.
- **Production read access** goes through masked views. PII queries are logged and reviewed weekly.

### Impersonation and customer-facing fraud (T3)
- Email: SPF, DKIM, and **DMARC `p=reject`** on all client domains (including unused ones). A verified WhatsApp Business profile. DLT-registered SMS headers only.
- **Customer rule printed on the invoice, the battery label, and the site footer, and sent in every WhatsApp:** "Power Mart never asks you to pay EMI to a personal number or UPI ID. Pay only through your financier's official QR or app."
- Domain security:
  - registrar lock and MFA;
  - CAA records;
  - DNSSEC where supported;
  - a DNS inventory with a monthly check for **dangling CNAMEs** (subdomain takeover);
  - monitoring for look-alike domains.

### Web hardening (browser side)
- A strict **CSP** with nonces and no `unsafe-inline` or `unsafe-eval`, plus `frame-ancestors 'none'`, `object-src 'none'`, and `base-uri 'none'`. Trusted Types where supported.
- Headers: HSTS (preload), `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, a `Permissions-Policy` that denies everything except camera and geolocation on the fitment screen, COOP and CORP.
- No `dangerouslySetInnerHTML`. User or partner text is always rendered as text.
- PWA offline store:
  - holds only the minimum (pending scans and forms);
  - is encrypted with a WebCrypto non-extractable key;
  - expires (TTL);
  - is wiped on logout and remote revoke.
- Subresource Integrity on any third-party script (ideally there are none).

### Availability and destruction (T6, T7)
- CloudFront + WAF managed rules, plus rate-based rules per path (OTP, lead form, login, webhooks). Shield Standard.
- Autoscaling with **max caps and billing alarms** (no denial-of-wallet).
- Load test before each hub launch wave.
- **Immutable backups** in a separate AWS account with vault lock and a cross-region copy (Hyderabad). Even a fully compromised prod account cannot delete them. A restore drill every month.
- Prod DB deletion protection. Terraform `prevent_destroy` on stateful resources.

### Error handling and logging (OWASP A09, A10)
- Exceptions inside authN, authZ, or signature code ⇒ deny. Generic error to the client, full detail internally (without PII).
- **Security event catalog**, each with an alert:
  - login anomalies;
  - an authZ deny spike;
  - OTP abuse;
  - webhook signature failures;
  - config, role, or scheme changes;
  - bulk exports;
  - honeytoken use;
  - new admin device;
  - reconciliation exceptions above a threshold.
- Audit logs are append-only and shipped to the log-archive account (Object Lock). App admins cannot alter them.

## 3. Kill switches (pre-built, tested, one click by HO or us)
- Pause all incentive and referral payouts.
- Pause a dealer, hub, or financier (no new sales or offers).
- Force logout of all sessions for a role or user.
- Rotate all webhook secrets and API keys (scripted).
- Put the public lead form into "WhatsApp-only" mode under bot attack.

## 4. Verification program (how we prove it, continuously)

| Activity | When |
|---|---|
| **Threat model** (STRIDE + abuse stories, i.e. "As a dishonest dealer I want to…") for every epic. Abuse stories become automated tests | Before building each epic |
| AuthZ matrix tests, race-condition tests, webhook replay and forgery tests, upload attack fixtures (polyglot, oversized, SVG, EXIF) | Every PR (CI) |
| SAST (CodeQL or Semgrep with custom rules for our patterns), secrets scan, dependency, container, and IaC scans | Every PR |
| DAST (ZAP authenticated scan) on staging | Every release candidate |
| Internal red-team day: we attack our own business logic with the T1–T8 goals | Before each release (R0–R3) |
| **External VAPT** by a CERT-In-empanelled auditor, with the report shared with financiers | Before go-live, after major changes, and yearly |
| `/.well-known/security.txt` and a responsible-disclosure page. Bug bounty later | From go-live |
| Restore drill, incident tabletop (including the 6-h CERT-In reporting path) | Monthly / every 6 months |

## 5. Security review checklist (on every PR that touches an endpoint, money, PII, or auth)
- [ ] Who can call this? Is row scope enforced in the service **and** by RLS? Is the authZ matrix updated?
- [ ] Are any client-supplied values trusted that the server should derive?
- [ ] Is it race-safe (constraint, lock, idempotency)?
- [ ] For a money-relevant state change: is evidence required, is it maker-checker, is it re-verified with the source?
- [ ] Is new PII collected? Purpose, consent, retention, masking, encryption? Is the data inventory updated?
- [ ] Do errors fail closed? Is there no PII in logs?
- [ ] Any new dependency (justified, aged, scanned)?
- [ ] Which abuse story does this test cover?

## References
- [OWASP Top 10:2025](https://owasp.org/Top10/2025/)
- OWASP ASVS 5.0 (May 2025): [Security Compass](https://www.securitycompass.com/blog/what-is-owasp-asvs/), [Appknox](https://www.appknox.com/cyber-security-jargons/owasp-asvs)
- CVE-2025-29927 (Next.js middleware bypass): [Snyk](https://snyk.io/blog/cve-2025-29927-authorization-bypass-in-next-js-middleware/), [Fastly](https://fastly.com/blog/cve-2025-29927-authorization-bypass-in-next-js)
- CVE-2025-55182 (React Server Components RCE): [Arctic Wolf](https://arcticwolf.com/resources/blog-uk/cve-2025-55182-critical-remote-code-execution-vulnerability-found-in-react-server-components/), [Kudelski Security](https://kudelskisecurity.com/research/critical-security-vulnerability-in-react-server-components)
