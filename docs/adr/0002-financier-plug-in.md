# ADR-0002: Financiers as configurable plug-ins (multi-financier, per product)

- Status: **Accepted (team)**. The client's admin UX will be validated in the prototype.
- Date: 2026-10-13
- Related: [architecture/02-financier-integration](../architecture/02-financier-integration.md), discussion log D8–D13

## Context
The client says each product has its own financier, different products may use different financiers, and financiers will change or be added. Financiers' tech maturity is unknown: some may offer only an app or a portal, others files or APIs. Power Mart must stay a merchant, outside the regulated lending perimeter.

## Options considered
1. **Hard-code one integration per financier.** Each new financier means a project, and changes risk breaking others.
2. **Rely on a single third-party lending gateway.** No plug-and-play gateway exists that covers the client's existing financiers for this segment, and it would force financiers onto someone else's platform.
3. **Our own financier plug-in layer** (chosen): a registry, versioned schemes, a standard status language, and a connector port with Manual, Link, File, and API adapters.

## Decision
Option 3. The rules:
- The client chooses financiers.
- Every financier starts in **Manual** mode on day one.
- Customer data is shared only with **consent naming that financier**.
- Rejections waterfall **sequentially**, with fresh consent.
- Changing a financier never touches existing loans.
- Loan and repayment data is a read-only mirror.
- Power Mart receives only sale proceeds (down payment, disbursal as seller), never repayments.

## Consequences
- About 1 extra week in R1 to build the generic layer. After that, a new financier costs hours (Manual) or 1–3 weeks (API), with no regression risk to the others.
- Manual mode needs maker-checker plus evidence, to stop fake statuses (engineering/06, T1).
- If a financier sits behind a lending platform, that platform is simply another adapter.
