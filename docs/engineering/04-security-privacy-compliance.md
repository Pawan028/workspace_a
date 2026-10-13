# 04 — Security, Privacy and Regulatory Engineering Rules

> These rules turn the regulations into engineering controls. They do **not** replace legal advice. Every item marked ⚖ needs confirmation from the NBFC's compliance team or counsel.

## 1. Regulatory map → controls

| Source | Requirement (as we understand it) | Our control |
|---|---|---|
| RBI Digital Lending Directions 2025 ⚖ | Repayment goes directly to the RE. No LSP pass-through | No borrower-money flows in our system (P1). The QR is NBFC-issued |
| | Data stored only in India. If processed abroad, delete it there and bring it back within 24 hours | India-region cloud only. Vendor check for every PII processor |
| | No access to phone files, media, contacts, call logs, or telephony. Camera, mic, and location one-time for KYC with consent | No such permissions requested. Consent recorded per access |
| | Explicit consent at every stage. KFS before disbursal. Cooling-off period | Consent ledger. KFS displayed from the NBFC only. No pre-ticked boxes |
| | Minimal data held by LSP / tech partners | Prefer the LSP or NBFC owning KYC. We store references |
| RBI Co-Lending Directions 2025 ⚖ | Blended rate, single borrower interface, disclosures | Pricing displayed from the lender's data only |
| RBI recovery / device-lock amendments (in force 1 Jan 2027) ⚖ | Device restriction barred as a recovery tool except for financed devices, with notice, a graduated process, and essential functions protected. Recovery-agent conduct rules | Immobilisation not built pending legal opinion. If Power Mart contacts overdue customers: contact windows, logging, recording retention |
| DPDP Act 2023 + Rules 2025 (core duties from about May 2027) | Notice, consent, purpose limitation, security safeguards, breach notification, erasure | Consent ledger, data inventory, retention jobs, breach runbook, grievance flow |
| Aadhaar Act / UIDAI | No storage of full Aadhaar except in a compliant vault | Mask to the last 4. Store DigiLocker or offline e-KYC references |
| CERT-In Directions 2022 | Report incidents within 6 hours. 180-day logs in India. NTP sync | Logging, retention, incident runbook, clock sync |
| TRAI TCCCPR | SMS via DLT-registered headers and templates | DLT registration is a client action item. Templates stored as config |
| GST | Correct invoices and e-invoicing where applicable | ERP integration or a reviewed invoicing module ⚖ |
| MoRTH BPAN (draft) | 21-character battery ID with lifecycle data | Identity table ready. Activate when notified |

## 2. Identity and access
- SSO / OIDC for staff. Phone OTP plus device binding for dealers and partners. MFA (TOTP) is mandatory for head office and admin roles.
- RBAC roles: `customer`, `reference_partner`, `dealer_owner`, `dealer_staff`, `hub_exec`, `hub_manager`, `fos`, `asm`, `zonal_head`, `ho_ops`, `ho_crm`, `ho_finance`, `admin`, `lender_viewer`, `auditor`. Each role is combined with **row scope** (hub, dealer, region).
- Least privilege. Joiner, mover, and leaver changes are applied the same day. Access is reviewed quarterly.
- Break-glass access is time-boxed, needs a reason, and alerts on use.

## 3. Data protection
- TLS 1.2+ everywhere, with HSTS. Encryption at rest is KMS-managed. Bank account and ID document numbers get field-level encryption.
- KYC and dealer documents go in a private bucket, accessed only through short-lived signed URLs. They are virus-scanned on upload, with type and size enforced.
- Secrets live in a cloud secret manager only. Never in the repo, `.env` files in shared drives, or CI logs. Secret scanning runs pre-commit and in CI.
- Backups are encrypted and stored in India. A **restore is tested every month** and recorded. RPO ≤ 15 minutes, RTO ≤ 4 hours (to be agreed).

## 4. Application security
- OWASP ASVS Level 2 is the baseline. The OWASP Top 10 and API Top 10 are checklist items in PR review for any endpoint change.
- SAST (Semgrep or CodeQL), dependency scanning, container scanning, and IaC scanning (tfsec or Checkov) run in CI. Critical findings block merge.
- DAST (ZAP baseline) runs against staging on each release candidate.
- Rate limiting on OTP, login, lead forms, and public APIs. Bot protection on public forms, with privacy-preserving options preferred.
- Input is validated at every boundary (Zod). Output is encoded. Parameterised SQL only.
- File uploads are checked for MIME by content (magic bytes) and size, and image EXIF is stripped.

## 5. Consent and privacy UX
- Consent is granular (marketing, data sharing with the lender, telemetry and location), versioned, and timestamped, with the notice text stored by version, and can be withdrawn.
- Notices are in Hindi and English, in plain language.
- Battery telemetry and location: consent is captured at sale. Location is used only for the stated purposes (service, theft, portfolio monitoring as agreed with the NBFC). Access to customer location is permissioned and logged.

## 6. Incident response
- Severity matrix, an on-call rota, and a 6-hour CERT-In reporting path with a named owner. DPDP Board and affected-user notification follows the Rules.
- Run a tabletop exercise before go-live, then every 6 months.

## 7. Partner due diligence pack (we maintain it; the NBFC will ask for it)
Architecture diagram, data-flow diagram, data inventory, hosting and region evidence, VAPT report, access-control policy, backup and DR evidence, incident response plan, list of sub-processors.
